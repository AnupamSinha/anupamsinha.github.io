---
title: "The Ultimate Docker Compose for Java Developers (15 Services, One File)"
date: 2026-08-24
categories: [DevOps, Cloud]
tags: [docker, java, spring-boot, devops, docker-compose]
description: "Spin up PostgreSQL, Redis, Kafka, Elasticsearch, MongoDB, Keycloak, Prometheus, Grafana, Loki, Tempo, Zipkin, pgAdmin, Redis Commander, Kafka UI, and MinIO with one command"
mermaid: true
---
Every Java project I start ends up needing at least 5-6 infrastructure services running locally. Database, cache, message broker, search engine — the list grows fast. And every time, I waste 30 minutes setting things up from scratch, googling port numbers, and debugging connectivity issues.

So I built one Docker Compose file that I copy into every project. It has 15 services covering everything a modern Spring Boot application needs — from databases to observability. This is the file I've refined over 3 years and dozens of projects. Let me walk you through every service, explain the configuration choices, and hand you the complete working file.

## Prerequisites

- Docker Desktop 4.x+ (or Docker Engine with Compose V2)
- At least 16GB RAM allocated to Docker (these services consume ~8-10GB under load)
- Docker Compose V2 (the `docker compose` command, not the legacy `docker-compose`)

## The Complete Docker Compose File

```yaml
version: '3.9'

services:
  # ============================================
  # DATABASES
  # ============================================
  postgres:
    image: postgres:16-alpine
    container_name: dev-postgres
    environment:
      POSTGRES_DB: appdb
      POSTGRES_USER: appuser
      POSTGRES_PASSWORD: apppass
      POSTGRES_MULTIPLE_DATABASES: keycloak
    ports:
      - "5432:5432"
    volumes:
      - postgres_data:/var/lib/postgresql/data
      - ./docker/postgres/init-multiple-dbs.sh:/docker-entrypoint-initdb.d/init-multiple-dbs.sh
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U appuser -d appdb"]
      interval: 10s
      timeout: 5s
      retries: 5
    networks:
      - backend

  mongodb:
    image: mongo:7
    container_name: dev-mongodb
    environment:
      MONGO_INITDB_ROOT_USERNAME: mongouser
      MONGO_INITDB_ROOT_PASSWORD: mongopass
      MONGO_INITDB_DATABASE: appdb
    ports:
      - "27017:27017"
    volumes:
      - mongodb_data:/data/db
    healthcheck:
      test: echo 'db.runCommand("ping").ok' | mongosh localhost:27017/test --quiet
      interval: 10s
      timeout: 5s
      retries: 5
    networks:
      - backend

  # ============================================
  # CACHING
  # ============================================
  redis:
    image: redis:7-alpine
    container_name: dev-redis
    command: redis-server --requirepass redispass --maxmemory 256mb --maxmemory-policy allkeys-lru
    ports:
      - "6379:6379"
    volumes:
      - redis_data:/data
    healthcheck:
      test: ["CMD", "redis-cli", "-a", "redispass", "ping"]
      interval: 10s
      timeout: 5s
      retries: 5
    networks:
      - backend

  # ============================================
  # MESSAGING
  # ============================================
  kafka:
    image: confluentinc/cp-kafka:7.6.0
    container_name: dev-kafka
    environment:
      KAFKA_NODE_ID: 1
      KAFKA_LISTENER_SECURITY_PROTOCOL_MAP: CONTROLLER:PLAINTEXT,PLAINTEXT:PLAINTEXT,PLAINTEXT_HOST:PLAINTEXT
      KAFKA_ADVERTISED_LISTENERS: PLAINTEXT://kafka:29092,PLAINTEXT_HOST://localhost:9092
      KAFKA_LISTENERS: PLAINTEXT://kafka:29092,CONTROLLER://kafka:29093,PLAINTEXT_HOST://0.0.0.0:9092
      KAFKA_CONTROLLER_LISTENER_NAMES: CONTROLLER
      KAFKA_CONTROLLER_QUORUM_VOTERS: 1@kafka:29093
      KAFKA_PROCESS_ROLES: broker,controller
      KAFKA_GROUP_INITIAL_REBALANCE_DELAY_MS: 0
      KAFKA_OFFSETS_TOPIC_REPLICATION_FACTOR: 1
      KAFKA_TRANSACTION_STATE_LOG_REPLICATION_FACTOR: 1
      CLUSTER_ID: MkU3OEVBNTcwNTJENDM2Qk
    ports:
      - "9092:9092"
    volumes:
      - kafka_data:/var/lib/kafka/data
    healthcheck:
      test: kafka-topics --bootstrap-server kafka:29092 --list
      interval: 30s
      timeout: 10s
      retries: 5
    networks:
      - backend

  # ============================================
  # SEARCH
  # ============================================
  elasticsearch:
    image: docker.elastic.co/elasticsearch/elasticsearch:8.13.0
    container_name: dev-elasticsearch
    environment:
      - discovery.type=single-node
      - xpack.security.enabled=false
      - xpack.security.http.ssl.enabled=false
      - ES_JAVA_OPTS=-Xms512m -Xmx512m
    ports:
      - "9200:9200"
    volumes:
      - elasticsearch_data:/usr/share/elasticsearch/data
    healthcheck:
      test: curl -s http://localhost:9200/_cluster/health | grep -vq '"status":"red"'
      interval: 20s
      timeout: 10s
      retries: 5
    networks:
      - backend

  # ============================================
  # OBJECT STORAGE
  # ============================================
  minio:
    image: minio/minio:latest
    container_name: dev-minio
    command: server /data --console-address ":9001"
    environment:
      MINIO_ROOT_USER: minioadmin
      MINIO_ROOT_PASSWORD: minioadmin
    ports:
      - "9000:9000"
      - "9001:9001"
    volumes:
      - minio_data:/data
    healthcheck:
      test: ["CMD", "mc", "ready", "local"]
      interval: 10s
      timeout: 5s
      retries: 5
    networks:
      - backend

  # ============================================
  # IDENTITY & ACCESS MANAGEMENT
  # ============================================
  keycloak:
    image: quay.io/keycloak/keycloak:24.0
    container_name: dev-keycloak
    command: start-dev
    environment:
      KC_DB: postgres
      KC_DB_URL: jdbc:postgresql://postgres:5432/keycloak
      KC_DB_USERNAME: appuser
      KC_DB_PASSWORD: apppass
      KEYCLOAK_ADMIN: admin
      KEYCLOAK_ADMIN_PASSWORD: admin
      KC_HEALTH_ENABLED: true
    ports:
      - "8180:8080"
    depends_on:
      postgres:
        condition: service_healthy
    healthcheck:
      test: ["CMD-SHELL", "exec 3<>/dev/tcp/localhost/8080 && echo -e 'GET /health/ready HTTP/1.1\\r\\nHost: localhost\\r\\nConnection: close\\r\\n\\r\\n' >&3 && cat <&3 | grep -q '200'"]
      interval: 30s
      timeout: 10s
      retries: 5
      start_period: 60s
    networks:
      - backend

  # ============================================
  # OBSERVABILITY - METRICS
  # ============================================
  prometheus:
    image: prom/prometheus:v2.51.0
    container_name: dev-prometheus
    command:
      - '--config.file=/etc/prometheus/prometheus.yml'
      - '--storage.tsdb.path=/prometheus'
      - '--storage.tsdb.retention.time=7d'
      - '--web.enable-lifecycle'
    ports:
      - "9090:9090"
    volumes:
      - ./docker/prometheus/prometheus.yml:/etc/prometheus/prometheus.yml
      - ./docker/prometheus/alert-rules.yml:/etc/prometheus/alert-rules.yml
      - prometheus_data:/prometheus
    healthcheck:
      test: wget --no-verbose --tries=1 --spider http://localhost:9090/-/healthy
      interval: 15s
      timeout: 5s
      retries: 5
    networks:
      - backend

  grafana:
    image: grafana/grafana:10.4.0
    container_name: dev-grafana
    environment:
      GF_SECURITY_ADMIN_USER: admin
      GF_SECURITY_ADMIN_PASSWORD: admin
      GF_USERS_ALLOW_SIGN_UP: false
    ports:
      - "3000:3000"
    volumes:
      - ./docker/grafana/provisioning:/etc/grafana/provisioning
      - ./docker/grafana/dashboards:/var/lib/grafana/dashboards
      - grafana_data:/var/lib/grafana
    depends_on:
      prometheus:
        condition: service_healthy
    healthcheck:
      test: wget --no-verbose --tries=1 --spider http://localhost:3000/api/health
      interval: 15s
      timeout: 5s
      retries: 5
    networks:
      - backend

  # ============================================
  # OBSERVABILITY - LOGGING
  # ============================================
  loki:
    image: grafana/loki:2.9.5
    container_name: dev-loki
    command: -config.file=/etc/loki/loki-config.yml
    ports:
      - "3100:3100"
    volumes:
      - ./docker/loki/loki-config.yml:/etc/loki/loki-config.yml
      - loki_data:/loki
    healthcheck:
      test: wget --no-verbose --tries=1 --spider http://localhost:3100/ready
      interval: 15s
      timeout: 5s
      retries: 5
    networks:
      - backend

  # ============================================
  # OBSERVABILITY - TRACING
  # ============================================
  tempo:
    image: grafana/tempo:2.4.1
    container_name: dev-tempo
    command: -config.file=/etc/tempo/tempo-config.yml
    ports:
      - "3200:3200"
      - "4317:4317"
      - "4318:4318"
    volumes:
      - ./docker/tempo/tempo-config.yml:/etc/tempo/tempo-config.yml
      - tempo_data:/tmp/tempo
    healthcheck:
      test: wget --no-verbose --tries=1 --spider http://localhost:3200/ready
      interval: 15s
      timeout: 5s
      retries: 5
    networks:
      - backend

  zipkin:
    image: openzipkin/zipkin:3
    container_name: dev-zipkin
    ports:
      - "9411:9411"
    environment:
      STORAGE_TYPE: mem
    healthcheck:
      test: wget --no-verbose --tries=1 --spider http://localhost:9411/health
      interval: 15s
      timeout: 5s
      retries: 5
    networks:
      - backend

  # ============================================
  # ADMIN UIs
  # ============================================
  pgadmin:
    image: dpage/pgadmin4:8
    container_name: dev-pgadmin
    environment:
      PGADMIN_DEFAULT_EMAIL: admin@local.dev
      PGADMIN_DEFAULT_PASSWORD: admin
      PGADMIN_CONFIG_SERVER_MODE: "False"
    ports:
      - "5050:80"
    depends_on:
      postgres:
        condition: service_healthy
    volumes:
      - pgadmin_data:/var/lib/pgadmin
    networks:
      - backend

  redis-commander:
    image: rediscommander/redis-commander:latest
    container_name: dev-redis-commander
    environment:
      REDIS_HOSTS: local:redis:6379:0:redispass
    ports:
      - "8081:8081"
    depends_on:
      redis:
        condition: service_healthy
    networks:
      - backend

  kafka-ui:
    image: provectuslabs/kafka-ui:v0.7.2
    container_name: dev-kafka-ui
    environment:
      KAFKA_CLUSTERS_0_NAME: local
      KAFKA_CLUSTERS_0_BOOTSTRAPSERVERS: kafka:29092
    ports:
      - "8082:8080"
    depends_on:
      kafka:
        condition: service_healthy
    networks:
      - backend

# ============================================
# NETWORKS
# ============================================
networks:
  backend:
    driver: bridge

# ============================================
# VOLUMES
# ============================================
volumes:
  postgres_data:
  mongodb_data:
  redis_data:
  kafka_data:
  elasticsearch_data:
  minio_data:
  prometheus_data:
  grafana_data:
  loki_data:
  tempo_data:
  pgadmin_data:
```

## Breaking Down Each Service

### PostgreSQL — The Primary Database

I use PostgreSQL 16 Alpine for the smaller image size. Key configuration choices:

- **Health check with pg_isready** — Other services that depend on Postgres won't start until it's actually accepting connections, not just when the container is running.
- **Named volume** — Data persists between `docker compose down` and `docker compose up`. Only `docker compose down -v` wipes it.
- **Multiple databases** — The init script creates a separate database for Keycloak. Your app database stays clean.

The init script (`docker/postgres/init-multiple-dbs.sh`):

```bash
#!/bin/bash
set -e

for db in $(echo $POSTGRES_MULTIPLE_DATABASES | tr ',' ' '); do
    psql -v ON_ERROR_STOP=1 --username "$POSTGRES_USER" <<-EOSQL
        CREATE DATABASE $db;
        GRANT ALL PRIVILEGES ON DATABASE $db TO $POSTGRES_USER;
EOSQL
done
```

### Redis — Caching and Session Store

- **Password protected** — Even locally, it's good practice. Your Spring Boot config already has the password.
- **Memory limit with LRU eviction** — 256MB cap prevents Redis from eating all available memory. LRU eviction means least-recently-used keys get dropped when full.
- **Alpine image** — 30MB vs 130MB for the full image.

Spring Boot connection config:

```yaml
spring:
  data:
    redis:
      host: localhost
      port: 6379
      password: redispass
```

### Kafka — Event Streaming (KRaft Mode)

This is Kafka 7.6 running in KRaft mode — **no ZooKeeper required**. This is the modern setup:

- **Single node** — KRaft allows Kafka to act as both broker and controller.
- **Two listeners** — `PLAINTEXT` for container-to-container communication (other Docker services connect to `kafka:29092`), `PLAINTEXT_HOST` for your Spring Boot app running on the host (connects to `localhost:9092`).
- **Replication factor of 1** — It's a single node. Don't replicate what you can't replicate.

Spring Boot Kafka config:

```yaml
spring:
  kafka:
    bootstrap-servers: localhost:9092
    consumer:
      group-id: my-app
      auto-offset-reset: earliest
    producer:
      key-serializer: org.apache.kafka.common.serialization.StringSerializer
      value-serializer: org.springframework.kafka.support.serializer.JsonSerializer
```

### Elasticsearch — Full-Text Search

- **Security disabled** — For local development only. In production, always enable security.
- **512MB heap** — Elasticsearch is memory-hungry. 512MB is the minimum for local use.
- **Single-node discovery** — No cluster formation overhead.

### MinIO — S3-Compatible Object Storage

MinIO gives you a local S3 API without needing an AWS account. Perfect for testing file uploads, presigned URLs, and bucket operations.

- **Port 9000** — S3 API endpoint
- **Port 9001** — Web console (great for browsing uploaded files)

Spring Boot config with AWS SDK:

```yaml
aws:
  s3:
    endpoint: http://localhost:9000
    access-key: minioadmin
    secret-key: minioadmin
    region: us-east-1
    bucket: my-app-files
```

### Keycloak — Identity Provider

Keycloak in dev mode provides a full OAuth2/OIDC identity provider:

- **Port 8180** — Avoids conflict with your Spring Boot app on 8080.
- **PostgreSQL backend** — Uses the same Postgres instance (separate database).
- **start-dev** — No HTTPS required, in-memory caching, live reload of themes.
- **start_period** — Keycloak takes 30-60 seconds to start. The health check waits before declaring it unhealthy.

### Observability Stack (Prometheus + Grafana + Loki + Tempo)

This is the full Grafana LGTM stack:

- **Prometheus** — Scrapes `/actuator/prometheus` from your Spring Boot app
- **Grafana** — Dashboards for metrics, logs, and traces in one place
- **Loki** — Log aggregation (like a lightweight ELK)
- **Tempo** — Distributed tracing backend (receives OTLP traces)

Prometheus config (`docker/prometheus/prometheus.yml`):

```yaml
global:
  scrape_interval: 15s
  evaluation_interval: 15s

rule_files:
  - alert-rules.yml

scrape_configs:
  - job_name: 'spring-boot-app'
    metrics_path: '/actuator/prometheus'
    scrape_interval: 5s
    static_configs:
      - targets: ['host.docker.internal:8080']
        labels:
          application: 'my-app'
```

### Zipkin — Alternative Tracing UI

I include both Tempo and Zipkin because Zipkin's UI is simpler for quick trace debugging, while Tempo integrates better with Grafana dashboards.

### Admin UIs

- **pgAdmin** — Visual database management at `localhost:5050`
- **Redis Commander** — Browse Redis keys at `localhost:8081`
- **Kafka UI** — Topics, consumers, messages at `localhost:8082`

## Port Reference

Quick reference for all exposed ports:

**PostgreSQL** — 5432
**MongoDB** — 27017
**Redis** — 6379
**Kafka** — 9092
**Elasticsearch** — 9200
**MinIO API** — 9000
**MinIO Console** — 9001
**Keycloak** — 8180
**Prometheus** — 9090
**Grafana** — 3000
**Loki** — 3100
**Tempo (OTLP gRPC)** — 4317
**Tempo (OTLP HTTP)** — 4318
**Zipkin** — 9411
**pgAdmin** — 5050
**Redis Commander** — 8081
**Kafka UI** — 8082

## Usage Patterns

### Start Everything

```bash
docker compose up -d
```

### Start Only What You Need

```bash
# Just databases
docker compose up -d postgres redis mongodb

# Databases + messaging
docker compose up -d postgres redis kafka

# Full observability
docker compose up -d prometheus grafana loki tempo
```

### Check Health Status

```bash
docker compose ps
```

All services should show `(healthy)` status. If something shows `(health: starting)`, give it another 30 seconds.

### View Logs for a Specific Service

```bash
docker compose logs -f kafka
```

### Reset Everything (Nuclear Option)

```bash
docker compose down -v
```

The `-v` flag removes all named volumes — all data is wiped.

## Spring Boot Application Properties

Here's the complete `application-local.yml` that connects to all services:

```yaml
spring:
  datasource:
    url: jdbc:postgresql://localhost:5432/appdb
    username: appuser
    password: apppass
  data:
    redis:
      host: localhost
      port: 6379
      password: redispass
    mongodb:
      uri: mongodb://mongouser:mongopass@localhost:27017/appdb?authSource=admin
    elasticsearch:
      uris: http://localhost:9200
  kafka:
    bootstrap-servers: localhost:9092
  security:
    oauth2:
      resourceserver:
        jwt:
          issuer-uri: http://localhost:8180/realms/my-realm

management:
  endpoints:
    web:
      exposure:
        include: health,info,prometheus,metrics
  tracing:
    sampling:
      probability: 1.0
  otlp:
    tracing:
      endpoint: http://localhost:4318/v1/traces
```

## Tips from Production Use

**Tip 1: Use profiles for selective startup.** Create a `compose.override.yml` that only starts the services your current feature needs.

**Tip 2: Add `host.docker.internal` to extra_hosts.** If a container needs to call back to your locally running Spring Boot app:

```yaml
extra_hosts:
  - "host.docker.internal:host-gateway"
```

**Tip 3: Pre-create Kafka topics.** Add an init container that creates topics on startup instead of relying on auto-creation:

```yaml
kafka-init:
  image: confluentinc/cp-kafka:7.6.0
  depends_on:
    kafka:
      condition: service_healthy
  command: >
    bash -c "kafka-topics --create --if-not-exists --topic orders 
    --bootstrap-server kafka:29092 --partitions 3 --replication-factor 1"
```

**Tip 4: Resource limits.** If your machine is struggling, add memory limits:

```yaml
deploy:
  resources:
    limits:
      memory: 512M
```

**Tip 5: Makefile shortcuts.** I always add a Makefile to avoid typing docker compose commands:

```makefile
.PHONY: up down logs ps reset

up:
	docker compose up -d

down:
	docker compose down

logs:
	docker compose logs -f $(service)

ps:
	docker compose ps

reset:
	docker compose down -v && docker compose up -d
```

## Final Thoughts

This file has saved me countless hours across projects. Clone it, customize the credentials, strip out services you don't need, and you've got a full development environment in under 2 minutes.

The key insight: health checks and dependency ordering are not optional. Without them, your Spring Boot app starts before Kafka is ready, connection failures cascade, and you waste 10 minutes restarting things manually. Get the health checks right once, never think about it again.
