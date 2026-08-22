# anupamsinha.github.io

Personal tech blog — practical Java/Spring Boot guides, architecture patterns, AI engineering, and developer references.

**Live at**: [https://anupamsinha.github.io](https://anupamsinha.github.io)

---

## Posts (40)

### Java & Spring Boot Fundamentals
| Post | Code |
|------|------|
| Java 18 to Java 21 Migration Guide | — |
| Beat the Basics — Spring Boot from Zero to Confident | [spring-boot-basics](https://github.com/AnupamSinha/spring-boot-basics) |
| Spring Boot with Virtual Threads (Project Loom) | [spring-boot-virtual-threads](https://github.com/AnupamSinha/spring-boot-virtual-threads) |
| Spring Boot with GraalVM Native Images | [spring-boot-graalvm-native](https://github.com/AnupamSinha/spring-boot-graalvm-native) |
| Spring Boot Performance Tuning Guide | [spring-boot-performance-tuning](https://github.com/AnupamSinha/spring-boot-performance-tuning) |

### Spring AI
| Post | Code |
|------|------|
| Building a RAG Application with Spring AI + PGVector | [spring-ai-in-action](https://github.com/AnupamSinha/spring-ai-in-action) |
| Spring AI Function Calling — Making LLMs Do Things | [spring-ai-function](https://github.com/AnupamSinha/spring-ai-function) |
| Spring AI Tool Calling + Conversation Memory | [spring-ai-function](https://github.com/AnupamSinha/spring-ai-function) (memory branch) |
| Spring AI + MCP — Exposing Tools as a Server | [spring-ai-mcp](https://github.com/AnupamSinha/spring-ai-mcp) |
| Spring AI Agentic Patterns + Streaming | [spring-ai-agents](https://github.com/AnupamSinha/spring-ai-agents) |

### Architecture & Design
| Post | Code |
|------|------|
| Spring Modulith — Modular Monoliths | — |
| Design Patterns in Spring Boot (Strategy, Observer, Template Method) | — |
| Hexagonal Architecture (Ports & Adapters) | [spring-boot-hexagonal](https://github.com/AnupamSinha/spring-boot-hexagonal) |
| CQRS + Event Sourcing | [spring-boot-cqrs-eventsourcing](https://github.com/AnupamSinha/spring-boot-cqrs-eventsourcing) |
| Outbox Pattern with Kafka | [spring-boot-outbox-pattern](https://github.com/AnupamSinha/spring-boot-outbox-pattern) |
| Spring Boot Multi-Tenancy | [spring-boot-multi-tenancy](https://github.com/AnupamSinha/spring-boot-multi-tenancy) |
| API Versioning Strategies | — |

### Data & Integration
| Post | Code |
|------|------|
| Spring Data JPA — Beyond the Basics | [spring-data-jpa-advanced](https://github.com/AnupamSinha/spring-data-jpa-advanced) |
| Spring Boot + MongoDB | [spring-boot-mongodb](https://github.com/AnupamSinha/spring-boot-mongodb) |
| Spring Boot + GraphQL | [spring-boot-graphql](https://github.com/AnupamSinha/spring-boot-graphql) |
| Spring Boot + gRPC | [spring-boot-grpc](https://github.com/AnupamSinha/spring-boot-grpc) |
| Event-Driven Microservices with Kafka | [spring-boot-kafka-microservices](https://github.com/AnupamSinha/spring-boot-kafka-microservices) |
| Caching Strategies (Redis + Caffeine) | [spring-boot-caching](https://github.com/AnupamSinha/spring-boot-caching) |
| Spring Batch — Processing Millions of Records | [spring-batch-demo](https://github.com/AnupamSinha/spring-batch-demo) |
| Spring Boot + WebSocket Real-Time | [spring-boot-websocket](https://github.com/AnupamSinha/spring-boot-websocket) |
| Spring Boot + Flyway Database Migrations | — |

### Security
| Post | Code |
|------|------|
| Spring Security 6 + OAuth2/OIDC with Keycloak | [spring-security-oauth2-demo](https://github.com/AnupamSinha/spring-security-oauth2-demo) |

### Testing
| Post | Code |
|------|------|
| Spring Boot + Testcontainers — Integration Testing | [spring-boot-testcontainers](https://github.com/AnupamSinha/spring-boot-testcontainers) |

### DevOps & Cloud
| Post | Code |
|------|------|
| Spring Cloud Gateway + Distributed Tracing | — |
| Spring Boot Observability (Prometheus, Loki, Tempo, Grafana) | [spring-boot-observability](https://github.com/AnupamSinha/spring-boot-observability) |
| Spring Boot + Docker/Kubernetes Deployment | [spring-boot-k8s-deploy](https://github.com/AnupamSinha/spring-boot-k8s-deploy) |
| GitHub Actions CI/CD for Spring Boot | — |
| Spring Boot on AWS (Lambda, ECS, RDS) | — |
| Migrating from Spring Cloud Netflix to Spring Cloud 2024 | — |
| Feature Flags with Togglz | [spring-boot-feature-flags](https://github.com/AnupamSinha/spring-boot-feature-flags) |
| Service Mesh vs Helm in OpenShift | — |

### Developer Experience
| Post | Code |
|------|------|
| Spring Boot Developer Productivity | — |
| Building a CLI with Spring Shell | [spring-shell-cli](https://github.com/AnupamSinha/spring-shell-cli) |
| Setting Up a Blog with Jekyll Chirpy | — |

---

## Tech Stack

| Component | Technology |
|-----------|-----------|
| Static site generator | Jekyll |
| Theme | [Chirpy](https://github.com/cotes2020/jekyll-theme-chirpy) |
| Hosting | GitHub Pages |
| CI/CD | GitHub Actions |
| Diagrams | Mermaid (rendered client-side) |

---

## Features

- Dark/light mode toggle
- Mermaid diagram support in posts
- Syntax-highlighted code blocks with line numbers
- Categories, tags, and archives
- SEO optimized (sitemap, meta descriptions, Open Graph)
- Table of contents on each post

---

## Writing a New Post

```bash
# Create a file in _posts/ with this naming:
YYYY-MM-DD-title-of-post.md
```

Front matter:

```yaml
---
title: "Your Post Title"
date: 2026-08-22
categories: [Java, Spring]
tags: [spring-boot, java-21]
description: "One-line SEO description."
mermaid: true  # if using diagrams
---
```

---

## Local Development

```bash
bundle install
bundle exec jekyll serve
```

Site available at `http://localhost:4000`.

---

## Build and Deploy

Automatic via GitHub Actions on push to `main`. Workflow: `.github/workflows/pages-deploy.yml`.

---

## License

Content and code examples are MIT licensed.
