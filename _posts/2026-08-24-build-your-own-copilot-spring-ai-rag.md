---
title: "Build Your Own Copilot with Spring AI and RAG in 30 Minutes"
date: 2026-08-24
categories: [Spring Boot, AI]
tags: [spring-ai, rag, java, ai, tutorial]
description: "A hands-on tutorial to build a domain-specific AI assistant that answers questions from your own codebase and documentation using Retrieval Augmented Generation (RAG) with Spring AI 1.0 and PGVector."
mermaid: true
---
## Why Build Your Own Copilot?

GitHub Copilot is great for general coding. But it doesn't know your company's architecture, your internal APIs, your team's conventions, or that one service that breaks every Thursday for reasons nobody documented.

What if you could build an AI assistant that:
- Understands your specific codebase patterns
- Answers questions about your internal architecture
- References your actual documentation and ADRs
- Knows your team's coding standards

That's exactly what we'll build. In 30 minutes. With Spring AI and RAG.

I've been running a version of this internally for our fintech platform in Singapore, and it's become the most-used internal tool on my team. New engineers ask it questions instead of interrupting seniors. Architecture decisions are searchable. Tribal knowledge becomes queryable.

---

## What is RAG?

Retrieval Augmented Generation is a pattern where you:

1. **Store** your documents/code as vector embeddings in a database
2. **Retrieve** relevant chunks when a user asks a question
3. **Augment** the LLM prompt with those retrieved chunks
4. **Generate** an answer grounded in your actual documentation

The key insight: instead of fine-tuning an LLM on your data (expensive, slow, stale), you give it relevant context at query time. The model reasons over your current documents without needing retraining.

---

## Architecture Overview

```
User Question
    │
    ▼
┌──────────────┐
│  Spring AI   │
│  Controller  │
└──────┬───────┘
       │
       ▼
┌──────────────┐     ┌─────────────────┐
│   Embedding  │────▶│    PGVector      │
│    Model     │     │  (PostgreSQL)    │
└──────────────┘     └────────┬────────┘
                              │ Top-K similar docs
                              ▼
                     ┌─────────────────┐
                     │   LLM (OpenAI/  │
                     │   Ollama/etc)   │
                     └────────┬────────┘
                              │
                              ▼
                     Grounded Answer
```

---

## Prerequisites

- Java 21+
- Docker (for PostgreSQL + PGVector)
- An OpenAI API key (or Ollama for local)
- 30 minutes

---

## Step 1: Project Setup

Create a new Spring Boot project with these dependencies:

```xml
<parent>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-parent</artifactId>
    <version>3.4.1</version>
</parent>

<dependencies>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-web</artifactId>
    </dependency>

    <!-- Spring AI Core -->
    <dependency>
        <groupId>org.springframework.ai</groupId>
        <artifactId>spring-ai-openai-spring-boot-starter</artifactId>
    </dependency>

    <!-- PGVector for vector storage -->
    <dependency>
        <groupId>org.springframework.ai</groupId>
        <artifactId>spring-ai-pgvector-store-spring-boot-starter</artifactId>
    </dependency>

    <!-- Document readers -->
    <dependency>
        <groupId>org.springframework.ai</groupId>
        <artifactId>spring-ai-tika-document-reader</artifactId>
    </dependency>

    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-jdbc</artifactId>
    </dependency>
</dependencies>

<dependencyManagement>
    <dependencies>
        <dependency>
            <groupId>org.springframework.ai</groupId>
            <artifactId>spring-ai-bom</artifactId>
            <version>1.0.0</version>
            <type>pom</type>
            <scope>import</scope>
        </dependency>
    </dependencies>
</dependencyManagement>
```

---

## Step 2: Docker Compose for PGVector

```yaml
# docker-compose.yml
services:
  postgres:
    image: pgvector/pgvector:pg16
    environment:
      POSTGRES_DB: vectordb
      POSTGRES_USER: postgres
      POSTGRES_PASSWORD: postgres
    ports:
      - "5432:5432"
    volumes:
      - pgdata:/var/lib/postgresql/data

volumes:
  pgdata:
```

Start it:

```bash
docker compose up -d
```

---

## Step 3: Application Configuration

```yaml
# application.yml
spring:
  datasource:
    url: jdbc:postgresql://localhost:5432/vectordb
    username: postgres
    password: postgres

  ai:
    openai:
      api-key: ${OPENAI_API_KEY}
      chat:
        options:
          model: gpt-4o
          temperature: 0.3
      embedding:
        options:
          model: text-embedding-3-small

    vectorstore:
      pgvector:
        initialize-schema: true
        index-type: HNSW
        distance-type: COSINE_DISTANCE
        dimensions: 1536

# Document ingestion settings
copilot:
  documents:
    path: classpath:docs/
    chunk-size: 800
    chunk-overlap: 200
```

---

## Step 4: Document Ingestion Service

This is the component that reads your documents, splits them into chunks, generates embeddings, and stores them in PGVector.

```java
@Service
@RequiredArgsConstructor
@Slf4j
public class DocumentIngestionService {

    private final VectorStore vectorStore;
    private final TokenTextSplitter textSplitter;

    @Value("${copilot.documents.path}")
    private String documentsPath;

    @Value("${copilot.documents.chunk-size:800}")
    private int chunkSize;

    @Value("${copilot.documents.chunk-overlap:200}")
    private int chunkOverlap;

    @Bean
    TokenTextSplitter tokenTextSplitter() {
        return new TokenTextSplitter(chunkSize, chunkOverlap, 5, 10000, true);
    }

    public void ingestDocuments() {
        log.info("Starting document ingestion from: {}", documentsPath);

        try {
            Resource[] resources = new PathMatchingResourcePatternResolver()
                .getResources(documentsPath + "/**/*.*");

            for (Resource resource : resources) {
                ingestSingleDocument(resource);
            }

            log.info("Ingestion complete. Processed {} documents.", resources.length);
        } catch (IOException e) {
            throw new DocumentIngestionException("Failed to read documents", e);
        }
    }

    private void ingestSingleDocument(Resource resource) {
        log.info("Ingesting: {}", resource.getFilename());

        // Use Tika to read any format (PDF, Markdown, HTML, etc.)
        TikaDocumentReader reader = new TikaDocumentReader(resource);
        List<Document> documents = reader.get();

        // Add metadata
        documents.forEach(doc -> {
            doc.getMetadata().put("source", resource.getFilename());
            doc.getMetadata().put("ingested_at", Instant.now().toString());
        });

        // Split into chunks
        List<Document> chunks = textSplitter.apply(documents);

        // Store embeddings in PGVector
        vectorStore.add(chunks);

        log.info("Stored {} chunks from {}", chunks.size(), resource.getFilename());
    }

    public void ingestText(String content, Map<String, Object> metadata) {
        Document document = new Document(content, metadata);
        List<Document> chunks = textSplitter.apply(List.of(document));
        vectorStore.add(chunks);
    }
}
```

---

## Step 5: The RAG Chat Service

This is the core — it retrieves relevant documents and feeds them to the LLM.

```java
@Service
@RequiredArgsConstructor
public class RagCopilotService {

    private final ChatClient chatClient;
    private final VectorStore vectorStore;

    private static final String SYSTEM_PROMPT = """
        You are an AI assistant for our engineering team. You answer questions
        about our codebase, architecture, internal APIs, and team conventions.

        Rules:
        - Only answer based on the provided context documents
        - If the context doesn't contain enough information, say so clearly
        - Reference specific documents/files when possible
        - Provide code examples when relevant
        - Be concise and practical

        Context documents:
        {context}
        """;

    public CopilotResponse askQuestion(String question) {
        // 1. Retrieve relevant documents
        List<Document> relevantDocs = vectorStore.similaritySearch(
            SearchRequest.builder()
                .query(question)
                .topK(5)
                .similarityThreshold(0.7)
                .build()
        );

        // 2. Format context from retrieved documents
        String context = relevantDocs.stream()
            .map(doc -> "Source: %s\n%s".formatted(
                doc.getMetadata().get("source"),
                doc.getContent()))
            .collect(Collectors.joining("\n---\n"));

        // 3. Call LLM with context-augmented prompt
        String answer = chatClient.prompt()
            .system(s -> s.text(SYSTEM_PROMPT).param("context", context))
            .user(question)
            .call()
            .content();

        // 4. Return answer with source references
        List<String> sources = relevantDocs.stream()
            .map(doc -> (String) doc.getMetadata().get("source"))
            .distinct()
            .toList();

        return new CopilotResponse(answer, sources);
    }
}
```

---

## Step 6: The REST Controller

```java
@RestController
@RequestMapping("/api/copilot")
@RequiredArgsConstructor
public class CopilotController {

    private final RagCopilotService copilotService;
    private final DocumentIngestionService ingestionService;

    @PostMapping("/ask")
    public ResponseEntity<CopilotResponse> ask(@RequestBody @Valid AskRequest request) {
        CopilotResponse response = copilotService.askQuestion(request.question());
        return ResponseEntity.ok(response);
    }

    @PostMapping("/ingest")
    public ResponseEntity<String> ingestDocuments() {
        ingestionService.ingestDocuments();
        return ResponseEntity.ok("Ingestion complete");
    }

    @PostMapping("/ingest/text")
    public ResponseEntity<String> ingestText(@RequestBody @Valid IngestTextRequest request) {
        ingestionService.ingestText(
            request.content(),
            Map.of("source", request.source(), "type", request.type())
        );
        return ResponseEntity.ok("Text ingested successfully");
    }
}

// DTOs
public record AskRequest(
    @NotBlank String question
) {}

public record IngestTextRequest(
    @NotBlank String content,
    @NotBlank String source,
    String type
) {}

public record CopilotResponse(
    String answer,
    List<String> sources
) {}
```

---

## Step 7: ChatClient Configuration

```java
@Configuration
public class AiConfig {

    @Bean
    ChatClient chatClient(ChatClient.Builder builder) {
        return builder
            .defaultAdvisors(new SimpleLoggerAdvisor())
            .build();
    }
}
```

---

## Step 8: Ingesting Your Codebase

Here's how I ingest our actual codebase and documentation:

```java
@Component
@RequiredArgsConstructor
@Slf4j
public class CodebaseIngestionRunner implements CommandLineRunner {

    private final DocumentIngestionService ingestionService;

    @Value("${copilot.auto-ingest:false}")
    private boolean autoIngest;

    @Override
    public void run(String... args) {
        if (autoIngest) {
            log.info("Auto-ingesting documents on startup...");
            ingestionService.ingestDocuments();
        }
    }
}
```

For your codebase, place these in `src/main/resources/docs/`:

- Architecture Decision Records (ADRs)
- README files from each service
- API documentation
- Coding standards documents
- Runbooks and playbooks
- Important code files (service interfaces, domain models)

---

## Step 9: Advanced — Ingesting Java Source Code

For code-specific questions, I ingest Java files with special metadata:

```java
@Service
@RequiredArgsConstructor
public class JavaCodeIngestionService {

    private final DocumentIngestionService ingestionService;

    public void ingestJavaFile(Path filePath) {
        try {
            String content = Files.readString(filePath);
            String className = extractClassName(content);
            String packageName = extractPackageName(content);

            Map<String, Object> metadata = Map.of(
                "source", filePath.getFileName().toString(),
                "type", "java-source",
                "class_name", className,
                "package", packageName,
                "path", filePath.toString()
            );

            ingestionService.ingestText(content, metadata);
        } catch (IOException e) {
            log.error("Failed to ingest: {}", filePath, e);
        }
    }

    public void ingestDirectory(Path directory) {
        try (Stream<Path> paths = Files.walk(directory)) {
            paths.filter(p -> p.toString().endsWith(".java"))
                 .filter(p -> !p.toString().contains("/test/"))
                 .forEach(this::ingestJavaFile);
        } catch (IOException e) {
            throw new RuntimeException("Failed to walk directory", e);
        }
    }

    private String extractClassName(String content) {
        Pattern pattern = Pattern.compile(
            "(?:public\\s+)?(?:abstract\\s+)?class\\s+(\\w+)");
        Matcher matcher = pattern.matcher(content);
        return matcher.find() ? matcher.group(1) : "Unknown";
    }

    private String extractPackageName(String content) {
        Pattern pattern = Pattern.compile("package\\s+([\\w.]+);");
        Matcher matcher = pattern.matcher(content);
        return matcher.find() ? matcher.group(1) : "default";
    }
}
```

---

## Step 10: Using the Copilot with Filters

Spring AI's VectorStore supports metadata filtering. Use this to scope queries:

```java
public CopilotResponse askAboutService(String question, String serviceName) {
    FilterExpressionBuilder builder = new FilterExpressionBuilder();

    List<Document> relevantDocs = vectorStore.similaritySearch(
        SearchRequest.builder()
            .query(question)
            .topK(5)
            .similarityThreshold(0.7)
            .filterExpression(builder.eq("source", serviceName).build())
            .build()
    );

    // ... same as before
}
```

---

## Testing It

Start the application and ingest some documents:

```bash
# Start PostgreSQL
docker compose up -d

# Run the application
./mvnw spring-boot:run

# Ingest documents
curl -X POST http://localhost:8080/api/copilot/ingest

# Ask a question
curl -X POST http://localhost:8080/api/copilot/ask \
  -H "Content-Type: application/json" \
  -d '{"question": "How do we handle payment retries in our system?"}'
```

Sample response:

```json
{
  "answer": "Based on our architecture documentation, payment retries use an exponential backoff strategy with a maximum of 3 attempts. The PaymentRetryService uses Spring Retry with @Retryable annotation. Failed payments after max retries are moved to a dead letter queue for manual review. See the Payment Service ADR-012 for the full retry policy.",
  "sources": [
    "adr-012-payment-retry-strategy.md",
    "payment-service-readme.md"
  ]
}
```

---

## Production Tips from Running This 6 Months

### 1. Chunk Size Matters

After experimentation, I found **800 tokens with 200 overlap** works best for technical documentation. Too small and you lose context. Too large and you dilute relevance.

### 2. Re-ingest on Schedule

I run ingestion nightly via a cron job. Documentation changes are picked up within 24 hours.

```java
@Scheduled(cron = "0 0 2 * * *") // 2 AM daily
public void scheduledIngestion() {
    ingestionService.ingestDocuments();
}
```

### 3. Add a Feedback Loop

Track which questions get "I don't have enough information" responses. Those are documentation gaps you need to fill.

### 4. Use Ollama for Sensitive Codebases

If you can't send code to OpenAI, run locally with Ollama:

```yaml
spring:
  ai:
    ollama:
      base-url: http://localhost:11434
      chat:
        options:
          model: llama3.1
      embedding:
        options:
          model: nomic-embed-text
```

### 5. Similarity Threshold Tuning

Start with 0.7. If answers are too sparse, lower to 0.6. If too noisy, raise to 0.75. Monitor and adjust based on result quality.

---

## What's Next

Once you have the basic copilot working, natural extensions include:

- **Slack/Teams integration** — let people ask questions in chat
- **Streaming responses** — use Spring AI's streaming API for better UX
- **Conversation memory** — add chat history for follow-up questions
- **Multi-modal** — ingest architecture diagrams alongside text
- **Auto-ingestion from Git hooks** — re-ingest when docs change

---

## Summary

In 30 minutes you've built a domain-specific AI copilot that:

- Ingests your documentation and codebase
- Stores vector embeddings in PostgreSQL with PGVector
- Retrieves relevant context for any question
- Generates grounded answers using RAG
- Returns source references for verification

The beauty of Spring AI is that switching models (OpenAI to Ollama to Anthropic) requires changing one configuration line. The RAG logic stays the same.

This is the pattern I recommend every team adopt. Not as a replacement for documentation, but as an intelligent interface to it.

---

*If you found this tutorial valuable, give it up to **50 claps** and follow [@hianupamsinha](https://medium.com/@hianupamsinha) for more practical Spring AI and Java deep dives.*
