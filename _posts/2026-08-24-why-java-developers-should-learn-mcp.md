---
title: "Why Every Java Developer Should Learn MCP (Model Context Protocol) Now"
date: 2026-08-24
categories: [Spring Boot, AI]
tags: [mcp, spring-ai, java, ai-agents, spring-boot]
description: "Understand what Model Context Protocol is, why it's the USB-C of AI integrations, how Spring AI supports it natively, and build a working MCP server in Spring Boot that lets AI agents interact with your systems."
mermaid: true
---
## The Problem MCP Solves

Here's a scenario every Java developer building AI features has hit:

You have an LLM-powered assistant. You want it to query your database, call your APIs, read files, or trigger deployments. Without a standard protocol, you end up writing custom integration code for every tool, every model, every client.

OpenAI has function calling. Anthropic has tool use. Google has function declarations. Each has slightly different schemas, slightly different invocation patterns, slightly different response formats.

Now imagine you're building an AI assistant that needs to:
- Query your order management system
- Check inventory levels
- Create support tickets
- Search your knowledge base

Without MCP, you write 4 custom integrations per LLM provider. Switch providers? Rewrite everything.

**Model Context Protocol (MCP)** solves this by providing a universal, standardized protocol for connecting AI models to external tools and data sources. Write your tool once, expose it via MCP, and any MCP-compatible AI client can use it.

---

## What is MCP?

MCP is an open protocol (originally developed by Anthropic, now widely adopted) that standardizes how AI applications provide context and tools to language models.

Think of it as **a USB-C port for AI integrations.** One standard interface, any device can plug in.

### Core Concepts

**MCP Server** — Your application that exposes tools, resources, and prompts to AI models

**MCP Client** — The AI application (IDE, chatbot, agent) that connects to MCP servers

**Tools** — Functions the AI can invoke (e.g., "search_orders", "create_ticket")

**Resources** — Data the AI can read (e.g., documentation files, database schemas)

**Prompts** — Pre-built prompt templates for common interactions

### The Architecture

```
AI Client (Cursor, Claude Desktop, Custom App)
    │
    │  MCP Protocol (JSON-RPC over stdio/SSE/HTTP)
    │
    ▼
┌─────────────────────┐
│    MCP Server        │
│  (Your Spring Boot)  │
├─────────────────────┤
│  Tools:              │
│  - search_orders     │
│  - get_inventory     │
│  - create_ticket     │
├─────────────────────┤
│  Resources:          │
│  - api_docs          │
│  - schema_info       │
├─────────────────────┤
│  Prompts:            │
│  - incident_report   │
│  - code_review       │
└─────────────────────┘
```

---

## MCP vs Function Calling — Key Differences

**Function Calling (OpenAI/Anthropic native)**
- Tied to one provider
- Defined at prompt time in each request
- No discovery mechanism
- Stateless
- Tools live inside your AI application code

**MCP**
- Provider-agnostic
- Tools are discovered dynamically by the client
- Server maintains state and context
- Tools live in independent servers (separation of concerns)
- Supports resources and prompts beyond just tools
- Built-in capability negotiation

The critical difference: with function calling, your AI app must know about all tools at compile time. With MCP, tools are discovered at runtime. Add a new MCP server? The AI client automatically discovers its capabilities.

---

## Why Java Developers Should Care

### 1. Spring AI Has First-Class MCP Support

Spring AI 1.0 ships with built-in MCP server and client support. If you're already in the Spring ecosystem, adding MCP is trivial.

### 2. Enterprise Java Systems Are Perfect MCP Servers

Think about what enterprise Java systems have:
- Complex business logic behind well-defined service interfaces
- Database access with rich querying capabilities
- Event systems and workflow engines
- Authentication and authorization already built in

Every one of these is a natural MCP tool. Your existing Spring services become AI-accessible with minimal code.

### 3. MCP Enables AI Agents That Actually Do Things

The next wave isn't chatbots that answer questions. It's agents that take actions. MCP is how those agents interact with your production systems safely and in a standardized way.

### 4. It's Becoming a Standard

Claude Desktop, Cursor, VS Code Copilot, and many other AI tools already support MCP clients. Building an MCP server makes your system accessible to a growing ecosystem of AI applications.

---

## Building an MCP Server with Spring Boot

Let's build a practical MCP server that exposes an order management system to AI agents.

### Step 1: Dependencies

```xml
<parent>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-parent</artifactId>
    <version>3.4.1</version>
</parent>

<dependencies>
    <dependency>
        <groupId>org.springframework.ai</groupId>
        <artifactId>spring-ai-mcp-server-spring-boot-starter</artifactId>
    </dependency>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-web</artifactId>
    </dependency>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-data-jpa</artifactId>
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

### Step 2: Application Configuration

```yaml
# application.yml
spring:
  ai:
    mcp:
      server:
        name: order-management-mcp
        version: 1.0.0
        type: SYNC
        transport: SSE
```

### Step 3: Define MCP Tools

This is where it gets interesting. Each tool is a method annotated with `@Tool`:

```java
@Service
@RequiredArgsConstructor
public class OrderManagementTools {

    private final OrderRepository orderRepository;
    private final OrderService orderService;
    private final InventoryService inventoryService;

    @Tool(description = "Search orders by customer ID, status, or date range. " +
          "Returns order summaries including order ID, status, total amount, and item count.")
    public List<OrderSummary> searchOrders(
            @ToolParam(description = "Customer UUID to filter by (optional)") 
            String customerId,
            @ToolParam(description = "Order status filter: PENDING, CONFIRMED, SHIPPED, DELIVERED, CANCELLED (optional)") 
            String status,
            @ToolParam(description = "Start date in ISO format yyyy-MM-dd (optional)") 
            String fromDate,
            @ToolParam(description = "End date in ISO format yyyy-MM-dd (optional)") 
            String toDate) {

        OrderSearchCriteria criteria = OrderSearchCriteria.builder()
            .customerId(customerId != null ? UUID.fromString(customerId) : null)
            .status(status != null ? OrderStatus.valueOf(status) : null)
            .fromDate(fromDate != null ? LocalDate.parse(fromDate) : null)
            .toDate(toDate != null ? LocalDate.parse(toDate) : null)
            .build();

        return orderRepository.searchOrders(criteria).stream()
            .map(this::toSummary)
            .toList();
    }

    @Tool(description = "Get full details of a specific order including all line items, " +
          "shipping information, payment status, and timeline of status changes.")
    public OrderDetails getOrderDetails(
            @ToolParam(description = "The order UUID") String orderId) {

        Order order = orderRepository.findById(UUID.fromString(orderId))
            .orElseThrow(() -> new OrderNotFoundException(orderId));

        return toDetails(order);
    }

    @Tool(description = "Check real-time inventory availability for a product SKU. " +
          "Returns current stock level, reserved quantity, and available quantity.")
    public InventoryStatus checkInventory(
            @ToolParam(description = "Product SKU code") String sku) {

        return inventoryService.getAvailability(sku);
    }

    @Tool(description = "Cancel an order. Only works for orders in PENDING or CONFIRMED status. " +
          "Triggers refund process and inventory release automatically.")
    public CancellationResult cancelOrder(
            @ToolParam(description = "The order UUID to cancel") String orderId,
            @ToolParam(description = "Reason for cancellation") String reason) {

        return orderService.cancelOrder(UUID.fromString(orderId), reason);
    }

    @Tool(description = "Get order statistics for a time period. " +
          "Returns total orders, revenue, average order value, and status breakdown.")
    public OrderStatistics getOrderStats(
            @ToolParam(description = "Start date in ISO format yyyy-MM-dd") String fromDate,
            @ToolParam(description = "End date in ISO format yyyy-MM-dd") String toDate) {

        return orderService.calculateStatistics(
            LocalDate.parse(fromDate),
            LocalDate.parse(toDate)
        );
    }

    private OrderSummary toSummary(Order order) {
        return new OrderSummary(
            order.getId().toString(),
            order.getStatus().name(),
            order.getTotalAmount(),
            order.getItems().size(),
            order.getCreatedAt().toString()
        );
    }
}
```

### Step 4: Register Tools with MCP Server

```java
@Configuration
public class McpServerConfig {

    @Bean
    ToolCallbackProvider orderManagementTools(OrderManagementTools tools) {
        return MethodToolCallbackProvider.builder()
            .toolObjects(tools)
            .build();
    }
}
```

That's it. Spring AI auto-discovers the `ToolCallbackProvider` beans and registers them with the MCP server.

### Step 5: Add MCP Resources (Readable Context)

Resources let AI clients read reference data without invoking tools:

```java
@Configuration
public class McpResourceConfig {

    @Bean
    McpServerFeatures.Resources orderResources() {
        return McpServerFeatures.Resources.builder()
            .resource("order-api-docs", "text/markdown",
                () -> loadResource("classpath:docs/order-api.md"))
            .resource("order-status-machine", "text/markdown",
                () -> loadResource("classpath:docs/order-status-transitions.md"))
            .resource("cancellation-policy", "text/markdown",
                () -> loadResource("classpath:docs/cancellation-policy.md"))
            .build();
    }

    private String loadResource(String path) {
        try {
            Resource resource = new ClassPathResource(
                path.replace("classpath:", ""));
            return new String(resource.getInputStream().readAllBytes());
        } catch (IOException e) {
            return "Resource not available";
        }
    }
}
```

---

## Building an MCP Client in Spring Boot

Now let's build the other side — a Spring Boot application that connects to MCP servers and uses their tools:

```java
@Configuration
public class McpClientConfig {

    @Bean
    McpSyncClient orderMcpClient() {
        McpSyncClient client = McpClient.sync(
            new SseClientTransport("http://localhost:8081/mcp/sse"))
            .build();
        client.initialize();
        return client;
    }

    @Bean
    ChatClient chatClient(ChatClient.Builder builder, McpSyncClient mcpClient) {
        // Register MCP tools with the chat client
        List<McpToolCallback> tools = McpToolCallback.from(mcpClient);

        return builder
            .defaultTools(tools.toArray(new McpToolCallback[0]))
            .build();
    }
}
```

Now when the AI needs to search orders, it automatically discovers and invokes the MCP tools:

```java
@RestController
@RequestMapping("/api/assistant")
@RequiredArgsConstructor
public class AssistantController {

    private final ChatClient chatClient;

    @PostMapping("/ask")
    public String ask(@RequestBody String question) {
        return chatClient.prompt()
            .system("""
                You are a customer support assistant with access to the order 
                management system. Use the available tools to answer questions 
                about orders, inventory, and cancellations. Always verify 
                information using tools before responding.
                """)
            .user(question)
            .call()
            .content();
    }
}
```

**Example interaction:**

```
User: "What's the status of order abc-123?"

AI internally: [calls getOrderDetails("abc-123")]
AI response: "Order abc-123 is currently in SHIPPED status. It was placed 
on Jan 15 and shipped on Jan 17. It contains 3 items totaling $149.99. 
The tracking shows it's expected to arrive by Jan 20."
```

---

## Transport Options

MCP supports multiple transport mechanisms:

**STDIO** — For local tools running as subprocesses (like CLI tools)

```yaml
spring:
  ai:
    mcp:
      server:
        transport: STDIO
```

**SSE (Server-Sent Events)** — For HTTP-based servers (most common for Spring Boot)

```yaml
spring:
  ai:
    mcp:
      server:
        transport: SSE
        sse-endpoint: /mcp/sse
```

**Streamable HTTP** — The newest transport, supports both streaming and request-response

For Spring Boot services, SSE is the natural choice — it works over standard HTTP, supports streaming, and plays well with existing infrastructure (load balancers, API gateways, service mesh).

---

## Security Considerations

MCP servers in production need proper security. Here's how I handle it:

```java
@Configuration
@EnableWebSecurity
public class McpSecurityConfig {

    @Bean
    SecurityFilterChain mcpSecurityChain(HttpSecurity http) throws Exception {
        return http
            .securityMatcher("/mcp/**")
            .authorizeHttpRequests(auth -> auth
                .requestMatchers("/mcp/sse").hasRole("MCP_CLIENT")
                .anyRequest().authenticated()
            )
            .oauth2ResourceServer(oauth2 -> oauth2.jwt(Customizer.withDefaults()))
            .build();
    }
}
```

**Key security practices:**
- Authenticate MCP clients (OAuth2 JWT works well)
- Rate limit tool invocations
- Log all tool calls for audit
- Implement tool-level authorization (not every AI agent should be able to cancel orders)
- Validate all tool inputs rigorously — AI can hallucinate malformed parameters

```java
@Tool(description = "Cancel an order")
public CancellationResult cancelOrder(
        @ToolParam(description = "The order UUID to cancel") String orderId,
        @ToolParam(description = "Reason for cancellation") String reason) {

    // ALWAYS validate inputs — AI-generated parameters can be malformed
    if (!isValidUUID(orderId)) {
        throw new IllegalArgumentException("Invalid order ID format");
    }
    if (reason == null || reason.isBlank() || reason.length() > 500) {
        throw new IllegalArgumentException("Reason must be 1-500 characters");
    }

    // Authorization check
    SecurityContext ctx = SecurityContextHolder.getContext();
    if (!authorizationService.canCancelOrder(ctx.getAuthentication(), orderId)) {
        throw new AccessDeniedException("Not authorized to cancel this order");
    }

    return orderService.cancelOrder(UUID.fromString(orderId), reason);
}
```

---

## Real-World MCP Server Patterns

### Pattern 1: Read-Heavy Information Tools

These are safe, low-risk tools that provide information:

```java
@Tool(description = "Get system health status across all services")
public SystemHealth getSystemHealth() {
    return healthService.aggregateHealth();
}

@Tool(description = "Search knowledge base articles")
public List<Article> searchKnowledgeBase(
        @ToolParam(description = "Search query") String query) {
    return knowledgeBaseService.search(query);
}
```

### Pattern 2: Write Tools with Confirmation

For state-changing operations, consider a two-step pattern:

```java
@Tool(description = "Preview order cancellation — shows what would happen but does NOT execute. " +
      "Use cancel_order_confirm to actually execute the cancellation.")
public CancellationPreview previewCancellation(
        @ToolParam(description = "Order UUID") String orderId) {
    return orderService.previewCancellation(UUID.fromString(orderId));
}

@Tool(description = "Execute order cancellation after preview. " +
      "Requires the confirmation token from preview_cancellation.")
public CancellationResult confirmCancellation(
        @ToolParam(description = "Confirmation token from preview") String confirmationToken) {
    return orderService.executeCancellation(confirmationToken);
}
```

### Pattern 3: Batch Operations with Limits

```java
@Tool(description = "Bulk update order statuses. Maximum 50 orders per call.")
public BulkUpdateResult bulkUpdateStatus(
        @ToolParam(description = "JSON array of order IDs (max 50)") String orderIdsJson,
        @ToolParam(description = "Target status") String targetStatus) {

    List<String> orderIds = parseOrderIds(orderIdsJson);
    if (orderIds.size() > 50) {
        throw new IllegalArgumentException("Maximum 50 orders per bulk operation");
    }
    return orderService.bulkUpdateStatus(orderIds, OrderStatus.valueOf(targetStatus));
}
```

---

## Testing MCP Servers

```java
@SpringBootTest
@AutoConfigureMockMvc
class OrderMcpServerIntegrationTest {

    @Autowired
    private McpSyncClient mcpClient;

    @Test
    void shouldDiscoverTools() {
        ListToolsResult tools = mcpClient.listTools();

        assertThat(tools.tools()).extracting("name")
            .contains("searchOrders", "getOrderDetails", 
                     "checkInventory", "cancelOrder");
    }

    @Test
    void shouldExecuteSearchOrders() {
        CallToolResult result = mcpClient.callTool(
            new CallToolRequest("searchOrders", Map.of(
                "status", "PENDING",
                "fromDate", "2025-01-01"
            ))
        );

        assertThat(result.content()).isNotEmpty();
        assertThat(result.isError()).isFalse();
    }

    @Test
    void shouldRejectInvalidInputs() {
        CallToolResult result = mcpClient.callTool(
            new CallToolRequest("cancelOrder", Map.of(
                "orderId", "not-a-valid-uuid",
                "reason", "test"
            ))
        );

        assertThat(result.isError()).isTrue();
    }
}
```

---

## Where MCP is Heading

MCP adoption is accelerating. Here's what I'm seeing:

**Now** — IDE integrations (Cursor, Claude Desktop), developer productivity tools

**Next 6 months** — Enterprise tool ecosystems, internal IT systems exposed via MCP

**Next 12 months** — MCP servers as a standard part of service architecture, alongside REST APIs and event streams

**My prediction** — Within two years, every enterprise Java service will expose an MCP interface alongside its REST API. The REST API serves machines and UIs. The MCP interface serves AI agents.

---

## Getting Started Today

1. **Start with a read-only MCP server** — expose search and query tools first
2. **Add it to your development tools** — configure Cursor or Claude Desktop to use your MCP server
3. **Measure usage** — what questions do developers ask? What tools do they invoke?
4. **Expand cautiously** — add write operations with proper guardrails
5. **Standardize** — define MCP tool patterns for your organization

The Java developers who understand MCP now will be the ones building the AI-integrated systems of the next three years. The protocol is simple, Spring AI makes it trivial to implement, and the ecosystem is growing fast.

Don't wait until it's mainstream. Learn it while it's early and shape how your organization adopts it.

---

*If you found this useful, give it up to **50 claps** and follow [@hianupamsinha](https://medium.com/@hianupamsinha) for more practical Spring AI and Java content.*
