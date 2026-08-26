---
title: "System Design: Chat Application with WebSocket, Kafka, and Redis"
date: 2026-08-24
categories: [System Design, Spring Boot]
tags: [system-design, websocket, kafka, redis, spring-boot]
description: "Design a real-time chat system that handles millions of concurrent messages using WebSocket for real-time delivery, Kafka for fan-out, and Redis for presence tracking"
mermaid: true
---
Every system design interview eventually gets around to "design a chat application." It sounds simple until you start thinking about scale. How do you handle millions of users online simultaneously? How do you ensure message ordering? What happens when a server goes down mid-conversation?

I've built real-time messaging systems in production over the past 17 years, and the gap between whiteboard answers and production-grade implementations is enormous. In this post, I'll walk you through designing a chat system that actually works at scale — with Spring Boot code you can run.

## The High-Level Architecture

Before diving into code, let's establish what we're building:

**Requirements** — what the system must support

- One-on-one and group messaging (up to 500 members)
- Real-time message delivery (sub-second latency)
- Online/offline presence tracking
- Read receipts
- Message persistence and history
- Horizontal scaling to millions of concurrent connections

**Core Components** — the building blocks

- **WebSocket Gateway** — maintains persistent connections with clients
- **Chat Service** — handles message routing and business logic
- **Kafka** — event bus for fan-out and decoupling
- **Redis** — presence tracking, session mapping, and caching
- **PostgreSQL** — message persistence and conversation metadata
- **Message Broker Cluster** — Kafka cluster for reliable message delivery

## WebSocket Gateway: The Connection Layer

The gateway is where clients connect. Each instance maintains thousands of WebSocket connections and knows which users are connected to it.

```java
@Configuration
@EnableWebSocketMessageBroker
public class WebSocketConfig implements WebSocketMessageBrokerConfigurer {

    @Override
    public void configureMessageBroker(MessageBrokerRegistry config) {
        config.enableSimpleBroker("/topic", "/queue");
        config.setApplicationDestinationPrefixes("/app");
        config.setUserDestinationPrefix("/user");
    }

    @Override
    public void registerStompEndpoints(StompEndpointRegistry registry) {
        registry.addEndpoint("/ws/chat")
                .setAllowedOrigins("*")
                .withSockJS();
    }
}
```

The critical piece is mapping users to their connected server instances. When User A sends a message to User B, we need to know which gateway server User B is connected to.

```java
@Service
@RequiredArgsConstructor
public class SessionRegistry {

    private final RedisTemplate<String, String> redisTemplate;
    private static final String SESSION_PREFIX = "chat:session:";
    private static final String SERVER_PREFIX = "chat:server:";

    public void registerSession(String userId, String serverId, String sessionId) {
        // Map user -> server instance
        redisTemplate.opsForHash().put(SESSION_PREFIX + userId, sessionId, serverId);
        // Map server -> set of connected users (for graceful shutdown)
        redisTemplate.opsForSet().add(SERVER_PREFIX + serverId, userId);
        // Set TTL for cleanup
        redisTemplate.expire(SESSION_PREFIX + userId, Duration.ofHours(24));
    }

    public void removeSession(String userId, String sessionId) {
        String serverId = (String) redisTemplate.opsForHash()
                .get(SESSION_PREFIX + userId, sessionId);
        redisTemplate.opsForHash().delete(SESSION_PREFIX + userId, sessionId);
        if (serverId != null) {
            redisTemplate.opsForSet().remove(SERVER_PREFIX + serverId, userId);
        }
    }

    public Set<String> getConnectedServers(String userId) {
        Map<Object, Object> sessions = redisTemplate.opsForHash()
                .entries(SESSION_PREFIX + userId);
        return sessions.values().stream()
                .map(Object::toString)
                .collect(Collectors.toSet());
    }
}
```

Why use a hash for user sessions? Because a single user might be connected from multiple devices (phone, laptop, tablet). Each device has its own session, potentially on different gateway servers.

## Presence Tracking with Redis

Presence (online/offline/typing) is one of those features that seems trivial until you handle disconnection edge cases. Here's my approach using Redis sorted sets with heartbeats:

```java
@Service
@RequiredArgsConstructor
public class PresenceService {

    private final RedisTemplate<String, String> redisTemplate;
    private final KafkaTemplate<String, PresenceEvent> kafkaTemplate;
    private static final String PRESENCE_KEY = "chat:presence";
    private static final long TIMEOUT_SECONDS = 30;

    public void heartbeat(String userId) {
        double score = System.currentTimeMillis();
        Boolean wasAbsent = redisTemplate.opsForZSet()
                .add(PRESENCE_KEY, userId, score);

        if (Boolean.TRUE.equals(wasAbsent)) {
            // User just came online — publish presence event
            kafkaTemplate.send("presence-events",
                    userId,
                    new PresenceEvent(userId, PresenceStatus.ONLINE, Instant.now()));
        }
    }

    public void markOffline(String userId) {
        redisTemplate.opsForZSet().remove(PRESENCE_KEY, userId);
        kafkaTemplate.send("presence-events",
                userId,
                new PresenceEvent(userId, PresenceStatus.OFFLINE, Instant.now()));
    }

    @Scheduled(fixedRate = 10000)
    public void evictStalePresence() {
        long cutoff = System.currentTimeMillis() - (TIMEOUT_SECONDS * 1000);
        Set<String> staleUsers = redisTemplate.opsForZSet()
                .rangeByScore(PRESENCE_KEY, 0, cutoff);

        if (staleUsers != null && !staleUsers.isEmpty()) {
            staleUsers.forEach(this::markOffline);
        }
    }

    public boolean isOnline(String userId) {
        Double score = redisTemplate.opsForZSet().score(PRESENCE_KEY, userId);
        if (score == null) return false;
        return (System.currentTimeMillis() - score) < (TIMEOUT_SECONDS * 1000);
    }

    public Set<String> getOnlineFriends(Set<String> friendIds) {
        return friendIds.stream()
                .filter(this::isOnline)
                .collect(Collectors.toSet());
    }
}
```

The sorted set approach is elegant — the score is the last heartbeat timestamp. A scheduled job evicts users whose last heartbeat is older than the timeout. This handles ungraceful disconnects (network drops, app kills) without relying solely on WebSocket disconnect events, which are unreliable under network partitions.

## Message Fan-Out with Kafka

When a user sends a message to a group with 200 members, you don't want the sending service to individually deliver to each member synchronously. This is where Kafka shines.

```java
@Service
@RequiredArgsConstructor
public class MessageService {

    private final KafkaTemplate<String, ChatMessage> kafkaTemplate;
    private final MessageRepository messageRepository;
    private final ConversationRepository conversationRepository;

    public ChatMessage sendMessage(SendMessageRequest request) {
        // 1. Persist the message
        ChatMessage message = ChatMessage.builder()
                .id(UUID.randomUUID().toString())
                .conversationId(request.getConversationId())
                .senderId(request.getSenderId())
                .content(request.getContent())
                .type(request.getType())
                .timestamp(Instant.now())
                .status(MessageStatus.SENT)
                .build();

        messageRepository.save(message);

        // 2. Update conversation metadata
        conversationRepository.updateLastMessage(
                request.getConversationId(), message.getId(), message.getTimestamp());

        // 3. Publish to Kafka for fan-out
        // Partition by conversationId to maintain ordering within a conversation
        kafkaTemplate.send("chat-messages",
                request.getConversationId(),
                message);

        return message;
    }
}
```

The Kafka consumer handles the actual delivery to recipients:

```java
@Service
@RequiredArgsConstructor
@Slf4j
public class MessageDeliveryConsumer {

    private final SessionRegistry sessionRegistry;
    private final SimpMessagingTemplate messagingTemplate;
    private final ConversationMemberRepository memberRepository;
    private final UndeliveredMessageRepository undeliveredRepo;

    @KafkaListener(topics = "chat-messages", groupId = "message-delivery")
    public void onMessage(ChatMessage message) {
        List<String> members = memberRepository
                .getMemberIds(message.getConversationId());

        for (String memberId : members) {
            if (memberId.equals(message.getSenderId())) continue;

            Set<String> connectedServers = sessionRegistry
                    .getConnectedServers(memberId);

            if (connectedServers.isEmpty()) {
                // User is offline — store for later delivery
                undeliveredRepo.save(new UndeliveredMessage(
                        memberId, message.getId(), message.getConversationId()));
            } else {
                // Deliver via WebSocket
                messagingTemplate.convertAndSendToUser(
                        memberId,
                        "/queue/messages",
                        message);
            }
        }
    }
}
```

**Why Kafka instead of direct WebSocket broadcast?**

- **Ordering guarantees** — partitioning by conversationId ensures messages within a conversation arrive in order
- **Decoupling** — the sending service doesn't need to know about delivery mechanics
- **Replay** — if a consumer crashes, Kafka retains messages for reprocessing
- **Back-pressure** — consumers process at their own pace without overwhelming the system

## Scaling WebSocket Connections

A single server can handle roughly 50K-100K concurrent WebSocket connections (depending on message frequency and hardware). To scale to millions, you need multiple gateway instances. The challenge: when Server A receives a message for a user on Server B, how does it reach them?

Two approaches:

**Approach 1: Redis Pub/Sub for inter-server messaging**

```java
@Service
@RequiredArgsConstructor
public class RedisMessageRelay {

    private final RedisTemplate<String, ChatMessage> redisTemplate;
    private final SimpMessagingTemplate messagingTemplate;

    @Value("${app.server-id}")
    private String currentServerId;

    public void relayToServer(String targetServerId, String userId,
                              ChatMessage message) {
        // Publish to server-specific channel
        redisTemplate.convertAndSend(
                "server:" + targetServerId + ":messages",
                new RelayMessage(userId, message));
    }

    @EventListener
    public void onRedisMessage(RelayMessage relay) {
        // Deliver locally to the connected user
        messagingTemplate.convertAndSendToUser(
                relay.getUserId(),
                "/queue/messages",
                relay.getMessage());
    }
}
```

**Approach 2: Dedicated Kafka topic per gateway instance**

Each gateway subscribes to its own topic. When you need to deliver to a user, check Redis for their server, then publish to that server's topic. This scales better than Redis Pub/Sub for high-throughput scenarios.

I prefer Approach 2 in production because Kafka gives you persistence and replay. If a gateway restarts, it can replay unprocessed messages from its topic.

## Read Receipts

Read receipts seem simple but can create massive write amplification in groups. If a group has 200 members and each reads a message, that's 200 read receipt events per message.

```java
@Service
@RequiredArgsConstructor
public class ReadReceiptService {

    private final RedisTemplate<String, String> redisTemplate;
    private final KafkaTemplate<String, ReadReceiptEvent> kafkaTemplate;

    private static final String READ_POSITION_KEY = "chat:read:%s:%s"; // conversationId:userId

    public void markAsRead(String userId, String conversationId,
                           String lastReadMessageId, long messageTimestamp) {
        // Store read position in Redis (not individual message reads)
        String key = String.format(READ_POSITION_KEY, conversationId, userId);
        redisTemplate.opsForValue().set(key, lastReadMessageId);

        // Publish event for sender notification
        kafkaTemplate.send("read-receipts",
                conversationId,
                new ReadReceiptEvent(userId, conversationId,
                        lastReadMessageId, messageTimestamp));
    }

    public Map<String, String> getReadPositions(String conversationId,
                                                 List<String> memberIds) {
        // Batch fetch read positions for all members
        List<String> keys = memberIds.stream()
                .map(id -> String.format(READ_POSITION_KEY, conversationId, id))
                .collect(Collectors.toList());

        List<String> positions = redisTemplate.opsForValue().multiGet(keys);

        Map<String, String> result = new HashMap<>();
        for (int i = 0; i < memberIds.size(); i++) {
            if (positions.get(i) != null) {
                result.put(memberIds.get(i), positions.get(i));
            }
        }
        return result;
    }
}
```

The key insight: don't store read status per-message. Store the **read position** (last read message ID/timestamp) per user per conversation. To determine if a message is "read" by someone, check if their read position is at or past that message. This reduces storage from O(messages * members) to O(members).

## Message Persistence Strategy

For message storage, I use a write-optimized approach:

```java
@Entity
@Table(name = "messages", indexes = {
    @Index(name = "idx_conversation_timestamp",
           columnList = "conversation_id, timestamp DESC")
})
public class MessageEntity {

    @Id
    private String id;
    private String conversationId;
    private String senderId;
    private String content;

    @Enumerated(EnumType.STRING)
    private MessageType type;

    private Instant timestamp;

    @Enumerated(EnumType.STRING)
    private MessageStatus status;
}
```

For high-volume systems, consider:

- **Time-based partitioning** — partition messages table by month. Old messages move to cold storage
- **Separate hot/cold paths** — recent messages (last 7 days) in Redis, older in PostgreSQL, archival in S3
- **Compression** — batch compress messages older than 30 days

```java
@Repository
@RequiredArgsConstructor
public class MessageQueryRepository {

    private final JdbcTemplate jdbcTemplate;
    private final RedisTemplate<String, List<ChatMessage>> redisCache;

    public List<ChatMessage> getRecentMessages(String conversationId,
                                                int limit, Instant before) {
        // Try Redis cache first (last 100 messages per conversation)
        String cacheKey = "chat:messages:" + conversationId;
        List<ChatMessage> cached = redisCache.opsForValue().get(cacheKey);

        if (cached != null && before == null) {
            return cached.stream().limit(limit).collect(Collectors.toList());
        }

        // Fall through to database
        return jdbcTemplate.query(
                "SELECT * FROM messages WHERE conversation_id = ? " +
                "AND timestamp < COALESCE(?, NOW()) " +
                "ORDER BY timestamp DESC LIMIT ?",
                new MessageRowMapper(),
                conversationId, before, limit);
    }
}
```

## Handling Failure Modes

Production chat systems must handle:

**Gateway server crash** — When a gateway dies, all users on that server appear to disconnect. The scheduled presence eviction job (from our PresenceService) will mark them offline after the timeout. When they reconnect to a different gateway, undelivered messages are fetched and pushed.

**Kafka consumer lag** — If message delivery consumers fall behind, users see delayed messages. Monitor consumer lag and auto-scale consumer instances.

**Redis failure** — Presence data is ephemeral. If Redis restarts, all users appear offline until their next heartbeat. Session mappings are rebuilt as users reconnect. Use Redis Sentinel or Cluster for HA.

**Split-brain in groups** — Use Kafka's ordering guarantees (partition by conversationId) to maintain a single source of truth for message ordering.

## Putting It All Together

The complete message flow:

1. Client sends message via WebSocket to their gateway
2. Gateway validates and calls MessageService
3. MessageService persists to PostgreSQL, publishes to Kafka
4. Kafka consumer fans out to each recipient
5. For each recipient, check Redis for connected server(s)
6. Route message to correct gateway via inter-server relay
7. Gateway delivers to client via WebSocket
8. Client acknowledges receipt (triggers delivery receipt)
9. When client reads message, send read position update

**Latency budget** — for the happy path (both users online, same region):

- **WebSocket to gateway** — 5-20ms
- **Kafka publish + consume** — 5-15ms
- **Redis lookup** — 1-3ms
- **Inter-server relay** — 5-10ms
- **WebSocket to recipient** — 5-20ms
- **Total** — 20-70ms end-to-end

## Capacity Planning Numbers

Based on my production experience, rough numbers for capacity planning:

**Per gateway server (8 cores, 16GB RAM)** — 60K concurrent WebSocket connections, 10K messages/second throughput

**Redis (single node)** — 500K presence entries, 100K session mappings, 50K ops/second

**Kafka cluster (3 brokers)** — 500K messages/second, 7-day retention

**PostgreSQL** — 50K writes/second with proper indexing and connection pooling

These numbers assume average message size of 500 bytes and moderate fanout (average group size of 10).

## What I'd Do Differently Today

If I were starting fresh in 2025, I'd consider:

- **Virtual Threads (Java 21)** — for the gateway layer, virtual threads can handle more connections per server with simpler code than reactive approaches
- **Apache Pulsar** — over Kafka for built-in multi-tenancy and tiered storage
- **CRDTs** — for eventual consistency in group metadata instead of strong consistency
- **Edge deployment** — gateway servers deployed at the edge (CDN PoPs) to minimize WebSocket latency

The fundamentals haven't changed though. The combination of WebSocket + Kafka + Redis remains the standard architecture for real-time messaging at scale.
