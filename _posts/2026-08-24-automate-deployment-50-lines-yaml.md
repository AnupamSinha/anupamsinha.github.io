---
title: "How I Automate My Entire Deployment Pipeline with 50 Lines of YAML"
date: 2026-08-24
categories: [DevOps, Cloud]
tags: [github-actions, devops, kubernetes, cicd, spring-boot]
description: "A minimal but complete CI/CD pipeline in GitHub Actions — build, test, Docker image, push to registry, deploy to Kubernetes — no over-engineering required"
mermaid: true
---
I've seen CI/CD pipelines that are 500+ lines of YAML spread across 8 files with conditional matrices, reusable workflows, and custom composite actions. For a team of 3 deploying one service.

Let me show you the opposite: a complete, production-grade pipeline in roughly 50 lines of YAML. It builds your Spring Boot app, runs tests, creates a Docker image, pushes it to a container registry, and deploys to Kubernetes. No over-engineering. No abstraction layers you'll never extend. Just what works.

## The Complete Pipeline

Here's the entire file. I'll explain every section after.

```yaml
name: Build and Deploy

on:
  push:
    branches: [main]
  pull_request:
    branches: [main]

env:
  REGISTRY: ghcr.io
  IMAGE_NAME: ${{ github.repository }}

jobs:
  build-test-deploy:
    runs-on: ubuntu-latest
    permissions:
      contents: read
      packages: write

    steps:
      - uses: actions/checkout@v4

      - uses: actions/setup-java@v4
        with:
          java-version: '21'
          distribution: 'temurin'
          cache: 'maven'

      - name: Build and Test
        run: ./mvnw verify -B -ntp

      - name: Build Docker Image
        if: github.event_name == 'push' && github.ref == 'refs/heads/main'
        run: |
          docker build -t ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}:${{ github.sha }} \
                       -t ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}:latest .

      - name: Push to Registry
        if: github.event_name == 'push' && github.ref == 'refs/heads/main'
        run: |
          echo "${{ secrets.GITHUB_TOKEN }}" | docker login ${{ env.REGISTRY }} -u $ --password-stdin
          docker push ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}:${{ github.sha }}
          docker push ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}:latest

      - name: Deploy to Kubernetes
        if: github.event_name == 'push' && github.ref == 'refs/heads/main'
        run: |
          echo "${{ secrets.KUBE_CONFIG }}" | base64 -d > kubeconfig
          export KUBECONFIG=kubeconfig
          kubectl set image deployment/order-service \
            order-service=${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}:${{ github.sha }} \
            -n production
          kubectl rollout status deployment/order-service -n production --timeout=300s
          rm kubeconfig
```

That's it. ~50 lines. Let me break down every decision.

## Section-by-Section Breakdown

### Trigger Configuration

```yaml
on:
  push:
    branches: [main]
  pull_request:
    branches: [main]
```

Two triggers serve different purposes:

- **pull_request** — Runs build and tests only. Validates the code before merge. The `if` conditions on later steps prevent Docker build and deploy from running on PRs.
- **push to main** — Full pipeline. After merge, build, test, containerize, and deploy.

This is intentional. PRs should be fast — build and test only. Deployment happens only after code reaches main.

### Environment Variables

```yaml
env:
  REGISTRY: ghcr.io
  IMAGE_NAME: ${{ github.repository }}
```

I use GitHub Container Registry (ghcr.io) because:

- It's free for public repos and included in GitHub Pro
- Authentication uses the same `GITHUB_TOKEN` — no extra secrets to manage
- The image name matches the repo name automatically

If you prefer Docker Hub or AWS ECR, just change the `REGISTRY` value and add appropriate credentials.

### Java Setup with Caching

```yaml
- uses: actions/setup-java@v4
  with:
    java-version: '21'
    distribution: 'temurin'
    cache: 'maven'
```

The `cache: 'maven'` parameter is critical. Without it, every build downloads your entire `.m2/repository` — that's often 500MB+ for a real project. With caching, subsequent builds start 2-3 minutes faster.

I use Temurin because it's the community successor to AdoptOpenJDK with the most predictable update cadence.

### Build and Test

```yaml
- name: Build and Test
  run: ./mvnw verify -B -ntp
```

Three flags that matter:

- **verify** — Runs compile, test, and integration-test phases. Not just `test`. If you have integration tests bound to the `verify` phase (which you should), they run here.
- **-B** — Batch mode. No interactive prompts, no download progress bars polluting your logs.
- **-ntp** — No transfer progress. Eliminates the "Downloading artifact..." noise.

Why `./mvnw` instead of `mvn`? The Maven Wrapper ensures the CI uses the exact same Maven version your team uses locally. No "works on my machine" versioning issues.

### Docker Build

```yaml
- name: Build Docker Image
  if: github.event_name == 'push' && github.ref == 'refs/heads/main'
  run: |
    docker build -t ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}:${{ github.sha }} \
                 -t ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}:latest .
```

Two tags per image:

- **SHA tag** (`abc123def`) — Immutable reference to this exact build. Used for deployments and rollbacks.
- **latest tag** — Convenience pointer. Useful for local development `docker pull`. Never use `latest` in production deployments.

The `if` condition ensures Docker only builds on pushes to main. PRs just validate the code compiles and tests pass.

### The Dockerfile

The pipeline references a `Dockerfile`. Here's the one I use for Spring Boot:

```dockerfile
FROM eclipse-temurin:21-jre-alpine AS runtime

WORKDIR /app

COPY target/*.jar app.jar

RUN addgroup -S appgroup && adduser -S appuser -G appgroup
USER appuser

EXPOSE 8080

HEALTHCHECK --interval=30s --timeout=3s --retries=3 \
  CMD wget --no-verbose --tries=1 --spider http://localhost:8080/actuator/health || exit 1

ENTRYPOINT ["java", "-XX:+UseContainerSupport", "-XX:MaxRAMPercentage=75.0", "-jar", "app.jar"]
```

Key decisions:

- **JRE Alpine** — Not the full JDK. Cuts image size from ~400MB to ~180MB.
- **Non-root user** — Security baseline. Never run containers as root.
- **UseContainerSupport** — Ensures the JVM respects container memory limits.
- **MaxRAMPercentage=75** — Leaves 25% for non-heap memory (thread stacks, native memory, metaspace).
- **HEALTHCHECK** — Kubernetes uses its own probes, but this provides Docker-level health checking for local and Docker Compose use.

### Push to Registry

```yaml
- name: Push to Registry
  if: github.event_name == 'push' && github.ref == 'refs/heads/main'
  run: |
    echo "${{ secrets.GITHUB_TOKEN }}" | docker login ${{ env.REGISTRY }} -u $ --password-stdin
    docker push ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}:${{ github.sha }}
    docker push ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}:latest
```

`GITHUB_TOKEN` is automatically available — no manual secret creation needed. The `permissions: packages: write` at the job level grants push access to ghcr.io.

### Deploy to Kubernetes

```yaml
- name: Deploy to Kubernetes
  if: github.event_name == 'push' && github.ref == 'refs/heads/main'
  run: |
    echo "${{ secrets.KUBE_CONFIG }}" | base64 -d > kubeconfig
    export KUBECONFIG=kubeconfig
    kubectl set image deployment/order-service \
      order-service=${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}:${{ github.sha }} \
      -n production
    kubectl rollout status deployment/order-service -n production --timeout=300s
    rm kubeconfig
```

This is the simplest deployment strategy that actually works:

1. **Decode kubeconfig** — Stored as a base64-encoded secret in GitHub
2. **Update the image** — `kubectl set image` triggers a rolling update
3. **Wait for rollout** — `rollout status` blocks until all pods are updated or the 5-minute timeout expires
4. **Cleanup** — Remove the kubeconfig file (security hygiene)

The rolling update means zero downtime by default — Kubernetes spins up new pods, health-checks them, then terminates old pods.

## Setting Up the Kubernetes Secret

To get the `KUBE_CONFIG` secret, run locally:

```bash
# Generate a base64-encoded kubeconfig
cat ~/.kube/config | base64 | pbcopy  # macOS
cat ~/.kube/config | base64 -w 0      # Linux (copy the output)
```

Then add it as a repository secret in GitHub: Settings → Secrets and variables → Actions → New repository secret → Name: `KUBE_CONFIG`.

**Important:** In production, don't use your personal kubeconfig. Create a service account with minimal permissions:

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: github-deployer
  namespace: production
---
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: deployer-role
  namespace: production
rules:
  - apiGroups: ["apps"]
    resources: ["deployments"]
    verbs: ["get", "patch", "update"]
  - apiGroups: ["apps"]
    resources: ["deployments/status"]
    verbs: ["get"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: deployer-binding
  namespace: production
subjects:
  - kind: ServiceAccount
    name: github-deployer
    namespace: production
roleRef:
  kind: Role
  name: deployer-role
  apiGroup: rbac.authorization.k8s.io
```

This service account can only update deployments in the `production` namespace. Nothing else.

## Extending the Pipeline

The base pipeline handles 80% of cases. Here are common extensions:

### Add Code Quality Checks

```yaml
      - name: SonarQube Analysis
        if: github.event_name == 'pull_request'
        env:
          SONAR_TOKEN: ${{ secrets.SONAR_TOKEN }}
        run: ./mvnw sonar:sonar -Dsonar.host.url=https://sonarcloud.io -B -ntp
```

### Add Slack Notification on Failure

```yaml
      - name: Notify on Failure
        if: failure()
        run: |
          curl -X POST ${{ secrets.SLACK_WEBHOOK }} \
            -H 'Content-type: application/json' \
            -d '{"text":"Deployment failed for ${{ github.repository }} - ${{ github.sha }}"}'
```

### Add Database Migration Before Deploy

```yaml
      - name: Run Flyway Migrations
        if: github.event_name == 'push' && github.ref == 'refs/heads/main'
        run: |
          echo "${{ secrets.KUBE_CONFIG }}" | base64 -d > kubeconfig
          export KUBECONFIG=kubeconfig
          kubectl run flyway-migrate --rm -i --restart=Never \
            --image=${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}:${{ github.sha }} \
            -n production \
            -- java -cp app.jar org.flywaydb.commandline.Main migrate
```

### Add Rollback on Failed Health Check

```yaml
      - name: Deploy to Kubernetes
        if: github.event_name == 'push' && github.ref == 'refs/heads/main'
        run: |
          echo "${{ secrets.KUBE_CONFIG }}" | base64 -d > kubeconfig
          export KUBECONFIG=kubeconfig
          kubectl set image deployment/order-service \
            order-service=${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}:${{ github.sha }} \
            -n production
          if ! kubectl rollout status deployment/order-service -n production --timeout=300s; then
            echo "Rollout failed! Rolling back..."
            kubectl rollout undo deployment/order-service -n production
            exit 1
          fi
          rm kubeconfig
```

## Why One Job, Not Multiple?

You might ask: shouldn't build, test, Docker, and deploy be separate jobs? In theory, yes — parallel execution, cleaner separation. In practice:

- **Separate jobs re-download dependencies** — Even with caching, each job spins up a fresh runner. Your Maven cache needs to be restored each time.
- **Artifact passing adds overhead** — Passing the JAR between jobs via `actions/upload-artifact` and `actions/download-artifact` adds 30-60 seconds.
- **Sequential anyway** — These steps are inherently sequential. You can't deploy before building.

For a single-service repo, one job with conditional steps is faster and simpler. Save multi-job pipelines for monorepos where you need parallel builds of different services.

## Pipeline Execution Times

From my real projects, approximate times:

**PR check (build + test only)**
- Checkout — 3s
- Java setup (cached) — 8s
- Maven build with tests — 45-90s (depends on test count)
- **Total — ~1-2 minutes**

**Full deploy (build + test + Docker + push + deploy)**
- Checkout — 3s
- Java setup (cached) — 8s
- Maven build with tests — 45-90s
- Docker build — 15-30s
- Docker push — 10-20s
- Kubernetes deploy + rollout — 30-60s
- **Total — ~2-4 minutes**

Under 4 minutes from push to production. That's fast enough to not break your flow.

## Common Mistakes to Avoid

- **Using `latest` tag for deployments** — Kubernetes won't restart pods if the tag hasn't changed. Always use SHA or version tags.
- **Not setting a rollout timeout** — Without `--timeout`, `rollout status` waits forever if pods are stuck in CrashLoopBackOff.
- **Skipping the `if` conditions** — Without them, PRs try to push Docker images and deploy. They'll fail with permission errors and confuse everyone.
- **Hardcoding secrets in YAML** — I've seen kubeconfigs committed in pipeline files. Always use GitHub Secrets.
- **Running `mvn install` instead of `mvn verify`** — `install` pushes to your local repo, which is meaningless in CI. `verify` does everything you need without the useless install step.

## For Gradle Users

The Maven parts translate directly:

```yaml
      - uses: actions/setup-java@v4
        with:
          java-version: '21'
          distribution: 'temurin'
          cache: 'gradle'

      - name: Build and Test
        run: ./gradlew build --no-daemon
```

Same concept, different tool. Everything else stays the same.

## Final Thoughts

The best CI/CD pipeline is one your whole team understands. If a new developer joins and can't figure out what the pipeline does in 5 minutes, it's too complex.

Start with ~50 lines. Add complexity only when you have a specific problem to solve — not because a Medium article told you to add matrix builds and reusable workflows on day one. You'll know when you need those things. Until then, keep it simple.
