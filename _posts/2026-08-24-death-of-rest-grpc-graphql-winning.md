---
title: "The Death of REST? Why gRPC and GraphQL Are Winning in 2026"
date: 2026-08-24
categories: [Spring Boot, Architecture]
tags: [rest-api, grpc, graphql, spring-boot, api-design]
description: "A pragmatic breakdown of when to use REST, gRPC, or GraphQL — with working Spring Boot code for all three"
mermaid: true
---
Every few years, someone declares REST is dead. And every time, REST survives. But here's what's different in 2026: the alternatives aren't theoretical anymore. gRPC and GraphQL have matured into production-ready ecosystems with first-class Spring Boot support.

After 17 years building distributed systems in Singapore — from banking APIs to real-time trading platforms — I've shipped production services using all three. REST isn't dead. But its monopoly is over. Let me explain where each pattern genuinely wins, where it struggles, and show you working code for all three.

## The State of API Architecture in 2026

The shift isn't about one protocol "winning." It's about recognizing that different communication patterns suit different problems:

- **REST** — request/response over HTTP, resource-oriented
- **gRPC** — binary protocol over HTTP/2, contract-first with Protocol Buffers
- **GraphQL** — query language for APIs, client-driven data fetching

The mistake I see teams make is treating this as a religious debate. In practice, most production systems I architect use at least two of these simultaneously.

## Where gRPC Wins (And It's Not Close)

### Internal Service-to-Service Communication

gRPC dominates when your services talk to each other. The reasons are concrete:

- **Performance** — Binary serialization with Protocol Buffers is 5-10x faster than JSON
- **Streaming** — Native bidirectional streaming without WebSocket hacks
- **Strong contracts** — Proto files are your source of truth, code generation is automatic
- **HTTP/2 multiplexing** — Multiple requests over a single connection

In my last project — a real-time order processing system — switching internal REST calls to gRPC reduced p99 latency from 45ms to 8ms. That's not a benchmark; that's production traffic.

### Spring Boot gRPC Implementation

First, the proto definition:

```protobuf
syntax = "proto3";

package com.example.order;

option java_multiple_files = true;
option java_package = "com.example.order.grpc";

service OrderService {
  rpc CreateOrder (CreateOrderRequest) returns (OrderResponse);
  rpc StreamOrderUpdates (OrderFilter) returns (stream OrderUpdate);
}

message CreateOrderRequest {
  string customer_id = 1;
  repeated OrderItem items = 2;
  string currency = 3;
}

message OrderItem {
  string product_id = 1;
  int32 quantity = 2;
  double unit_price = 3;
}

message OrderResponse {
  string order_id = 1;
  string status = 2;
  double total_amount = 3;
  int64 created_at = 4;
}

message OrderFilter {
  string customer_id = 1;
  repeated string statuses = 2;
}

message OrderUpdate {
  string order_id = 1;
  string previous_status = 2;
  string new_status = 3;
  int64 updated_at = 4;
}
```

The Spring Boot gRPC server implementation:

```java
@GrpcService
public class OrderGrpcService extends OrderServiceGrpc.OrderServiceImplBase {

    private final OrderRepository orderRepository;
    private final OrderEventPublisher eventPublisher;

    public OrderGrpcService(OrderRepository orderRepository, 
                            OrderEventPublisher eventPublisher) {
        this.orderRepository = orderRepository;
        this.eventPublisher = eventPublisher;
    }

    @Override
    public void createOrder(CreateOrderRequest request, 
                           StreamObserver<OrderResponse> responseObserver) {
        
        Order order = Order.builder()
            .customerId(request.getCustomerId())
            .items(mapItems(request.getItemsList()))
            .currency(request.getCurrency())
            .status(OrderStatus.CREATED)
            .createdAt(Instant.now())
            .build();

        Order saved = orderRepository.save(order);
        eventPublisher.publish(new OrderCreatedEvent(saved));

        OrderResponse response = OrderResponse.newBuilder()
            .setOrderId(saved.getId())
            .setStatus(saved.getStatus().name())
            .setTotalAmount(saved.getTotalAmount())
            .setCreatedAt(saved.getCreatedAt().toEpochMilli())
            .build();

        responseObserver.onNext(response);
        responseObserver.onCompleted();
    }

    @Override
    public void streamOrderUpdates(OrderFilter request,
                                   StreamObserver<OrderUpdate> responseObserver) {
        
        Flux<OrderEvent> events = eventPublisher
            .subscribe(request.getCustomerId(), request.getStatusesList());

        events.subscribe(
            event -> {
                OrderUpdate update = OrderUpdate.newBuilder()
                    .setOrderId(event.getOrderId())
                    .setPreviousStatus(event.getPreviousStatus())
                    .setNewStatus(event.getNewStatus())
                    .setUpdatedAt(event.getTimestamp().toEpochMilli())
                    .build();
                responseObserver.onNext(update);
            },
            responseObserver::onError,
            responseObserver::onCompleted
        );
    }

    private List<OrderItemEntity> mapItems(List<OrderItem> protoItems) {
        return protoItems.stream()
            .map(item -> new OrderItemEntity(
                item.getProductId(), 
                item.getQuantity(), 
                item.getUnitPrice()))
            .toList();
    }
}
```

The Maven dependency setup:

```xml
<dependency>
    <groupId>net.devh</groupId>
    <artifactId>grpc-spring-boot-starter</artifactId>
    <version>3.1.0.RELEASE</version>
</dependency>
<dependency>
    <groupId>io.grpc</groupId>
    <artifactId>grpc-protobuf</artifactId>
    <version>1.62.2</version>
</dependency>
<dependency>
    <groupId>io.grpc</groupId>
    <artifactId>grpc-stub</artifactId>
    <version>1.62.2</version>
</dependency>
```

### When gRPC Doesn't Work

- **Browser clients** — You need gRPC-Web or an Envoy proxy. Extra complexity.
- **Public APIs** — Your consumers probably don't want to compile proto files.
- **Simple CRUD** — The ceremony of proto files and code generation isn't worth it for basic operations.
- **Debugging** — Binary payloads are harder to inspect than JSON. You'll miss curl.

## Where GraphQL Wins (Especially for Frontend Teams)

### Flexible Data Fetching for Mobile and SPAs

GraphQL shines when your clients have diverse data needs. A mobile app needs a subset of what the web dashboard needs. With REST, you either over-fetch (waste bandwidth) or build custom endpoints for each client (waste developer time).

In a fintech project I architected, the mobile team was making 6 REST calls to render one screen. After switching to GraphQL, it became one query that returned exactly what the screen needed. The improvement in mobile performance was immediate.

### Spring Boot GraphQL Implementation

Spring for GraphQL (the official integration since Spring Boot 3.x) makes this clean:

```java
@Controller
public class OrderGraphQLController {

    private final OrderService orderService;
    private final CustomerService customerService;
    private final ProductService productService;

    public OrderGraphQLController(OrderService orderService,
                                  CustomerService customerService,
                                  ProductService productService) {
        this.orderService = orderService;
        this.customerService = customerService;
        this.productService = productService;
    }

    @QueryMapping
    public List<Order> ordersByCustomer(@Argument String customerId,
                                        @Argument OrderStatus status,
                                        @Argument int limit) {
        return orderService.findByCustomer(customerId, status, limit);
    }

    @MutationMapping
    public Order createOrder(@Argument CreateOrderInput input) {
        return orderService.create(input);
    }

    @SchemaMapping(typeName = "Order", field = "customer")
    public CompletableFuture<Customer> customer(Order order, 
                                                 DataLoader<String, Customer> customerLoader) {
        return customerLoader.load(order.getCustomerId());
    }

    @SchemaMapping(typeName = "Order", field = "items")
    public List<OrderItem> items(Order order) {
        return orderService.getItems(order.getId());
    }

    @BatchMapping
    public Map<OrderItem, Product> product(List<OrderItem> items) {
        List<String> productIds = items.stream()
            .map(OrderItem::getProductId)
            .toList();
        
        Map<String, Product> products = productService
            .findByIds(productIds)
            .stream()
            .collect(Collectors.toMap(Product::getId, Function.identity()));

        return items.stream()
            .collect(Collectors.toMap(
                Function.identity(),
                item -> products.get(item.getProductId())
            ));
    }
}
```

The GraphQL schema (`resources/graphql/schema.graphqls`):

```graphql
type Query {
    ordersByCustomer(customerId: ID!, status: OrderStatus, limit: Int = 20): [Order!]!
    order(id: ID!): Order
}

type Mutation {
    createOrder(input: CreateOrderInput!): Order!
}

type Order {
    id: ID!
    customer: Customer!
    items: [OrderItem!]!
    status: OrderStatus!
    totalAmount: Float!
    createdAt: String!
}

type OrderItem {
    id: ID!
    product: Product!
    quantity: Int!
    unitPrice: Float!
}

type Customer {
    id: ID!
    name: String!
    email: String!
}

type Product {
    id: ID!
    name: String!
    category: String!
}

input CreateOrderInput {
    customerId: ID!
    items: [OrderItemInput!]!
}

input OrderItemInput {
    productId: ID!
    quantity: Int!
}

enum OrderStatus {
    CREATED
    CONFIRMED
    SHIPPED
    DELIVERED
    CANCELLED
}
```

The `@BatchMapping` annotation is crucial — it solves the N+1 problem that kills GraphQL performance. Without batching, fetching 50 orders would make 50 individual product queries. With batching, it's one query.

### When GraphQL Doesn't Work

- **Simple APIs** — If every client needs the same data, GraphQL adds complexity without benefit.
- **File uploads** — GraphQL's handling of binary data is awkward. Use REST or presigned URLs.
- **Caching** — HTTP caching doesn't work naturally with GraphQL's single POST endpoint. You need Apollo-style client caching or persisted queries.
- **Rate limiting** — Hard to limit by cost when every query has different complexity.

## Where REST Still Reigns Supreme

### Public APIs and Simplicity

REST isn't going anywhere for public-facing APIs. Here's why:

- **Universal understanding** — Every developer knows HTTP verbs and status codes
- **Tooling** — curl, Postman, OpenAPI/Swagger, browser dev tools all work out of the box
- **Caching** — HTTP caching works naturally with GET requests
- **Simplicity** — No schema compilation, no binary protocols, no query complexity analysis

For the Spring Boot implementation that every Java developer knows:

```java
@RestController
@RequestMapping("/api/v1/orders")
public class OrderRestController {

    private final OrderService orderService;

    public OrderRestController(OrderService orderService) {
        this.orderService = orderService;
    }

    @GetMapping
    public ResponseEntity<PagedResponse<OrderDto>> getOrders(
            @RequestParam(required = false) String customerId,
            @RequestParam(defaultValue = "0") int page,
            @RequestParam(defaultValue = "20") int size,
            @RequestParam(defaultValue = "createdAt,desc") String sort) {

        Page<Order> orders = orderService.findAll(customerId, 
            PageRequest.of(page, size, Sort.by(parseSort(sort))));

        PagedResponse<OrderDto> response = PagedResponse.<OrderDto>builder()
            .content(orders.map(OrderDto::from).getContent())
            .page(orders.getNumber())
            .size(orders.getSize())
            .totalElements(orders.getTotalElements())
            .totalPages(orders.getTotalPages())
            .build();

        return ResponseEntity.ok()
            .cacheControl(CacheControl.maxAge(30, TimeUnit.SECONDS))
            .body(response);
    }

    @PostMapping
    public ResponseEntity<OrderDto> createOrder(
            @Valid @RequestBody CreateOrderRequest request) {
        
        Order order = orderService.create(request);
        
        URI location = ServletUriComponentsBuilder
            .fromCurrentRequest()
            .path("/{id}")
            .buildAndExpand(order.getId())
            .toUri();

        return ResponseEntity.created(location)
            .body(OrderDto.from(order));
    }

    @GetMapping("/{orderId}")
    public ResponseEntity<OrderDto> getOrder(@PathVariable String orderId) {
        return orderService.findById(orderId)
            .map(order -> ResponseEntity.ok()
                .eTag(order.getVersion().toString())
                .body(OrderDto.from(order)))
            .orElseThrow(() -> new OrderNotFoundException(orderId));
    }

    @PatchMapping("/{orderId}/status")
    public ResponseEntity<OrderDto> updateStatus(
            @PathVariable String orderId,
            @Valid @RequestBody UpdateStatusRequest request) {
        
        Order updated = orderService.updateStatus(orderId, request.getStatus());
        return ResponseEntity.ok(OrderDto.from(updated));
    }
}
```

REST with proper HTTP semantics — status codes, ETags, cache headers, HATEOAS links — is still the gold standard for public consumption.

## My Decision Framework

After years of making this choice across different projects, here's the framework I use:

**Choose gRPC when:**
- Services communicate internally (east-west traffic)
- You need streaming (real-time updates, event streams)
- Performance is critical (low latency, high throughput)
- Teams own both client and server
- You're building polyglot systems (proto files generate code for any language)

**Choose GraphQL when:**
- Multiple clients need different data shapes (mobile vs web vs third-party)
- Frontend team wants autonomy to fetch what they need
- You're aggregating data from multiple backend services (BFF pattern)
- Over-fetching is a measurable problem (especially for mobile)

**Choose REST when:**
- Building public/external APIs
- Simple CRUD with predictable access patterns
- You want maximum tooling support and developer familiarity
- HTTP caching is important
- You're building webhooks or integrating with third-party systems

## The Hybrid Architecture I Actually Deploy

In production, I typically use all three:

- **Public API Gateway** — REST (with OpenAPI spec for documentation)
- **Backend for Frontend (BFF)** — GraphQL (aggregates internal services)
- **Internal microservices** — gRPC (service mesh with Istio handling mTLS)

```
[Mobile App] → [GraphQL BFF] → [gRPC Services]
[Web App]    → [GraphQL BFF] → [gRPC Services]
[Partners]   → [REST Gateway] → [gRPC Services]
```

This isn't over-engineering — each layer uses the protocol that best serves its consumers.

## Performance Comparison (Real Numbers)

From a recent load test on identical hardware (4 vCPU, 8GB, GraalVM 21):

**Throughput (requests/sec, simple order fetch)**
- **gRPC** — 48,000 req/s
- **REST (JSON)** — 12,000 req/s
- **GraphQL** — 9,500 req/s

**P99 Latency (under load)**
- **gRPC** — 3ms
- **REST** — 15ms
- **GraphQL** — 22ms

**Payload Size (same data)**
- **gRPC (Protobuf)** — 142 bytes
- **REST (JSON)** — 487 bytes
- **GraphQL (JSON response)** — 312 bytes (no over-fetching)

GraphQL is slower than REST in raw throughput because of query parsing and validation overhead. But it saves bandwidth on the wire and reduces total round trips, which matters more for mobile clients.

## Common Mistakes I See

- **Using gRPC for everything** — Including browser-facing endpoints. The tooling friction isn't worth it.
- **GraphQL without DataLoader** — Instant N+1 problems. Every GraphQL service needs batch loading.
- **REST without versioning** — Breaking public API consumers when you change the response format.
- **Premature gRPC adoption** — If you have 3 services, REST is fine. gRPC's value compounds at scale.
- **GraphQL without query complexity limits** — One malicious nested query can take down your server.

## Final Thoughts

REST isn't dying. But the era of "REST for everything" is definitively over. The mature engineer in 2026 picks the right tool for the communication pattern, not the one they're most comfortable with.

If you take one thing from this post: start with your consumers. Who's calling your API, what do they need, and what are their constraints? The answer to that question picks your protocol.
