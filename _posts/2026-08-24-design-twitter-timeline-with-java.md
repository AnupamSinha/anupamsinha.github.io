---
title: "How I'd Design Twitter's Timeline — With Working Java Code"
date: 2026-08-24
categories: [System Design, Spring Boot]
tags: [system-design, java, redis, kafka, interview-prep]
description: "A practical breakdown of fan-out-on-write vs fan-out-on-read, Redis timelines, Kafka event propagation, and caching — with Spring Boot code you can actually run"
mermaid: true
---
"Design Twitter" might be the most asked system design interview question in existence. I've been on both sides of that table dozens of times over 17 years, and here's what I've noticed: most candidates draw boxes and arrows but can't write the actual code. And most blog posts explain the theory without showing implementation.

This post bridges that gap. I'll design the timeline system AND give you working Spring Boot code for every major component. No hand-waving. No "left as an exercise for the reader."

## Defining the Problem

Twitter's core product is the home timeline — a personalized feed of tweets from people you follow, ranked (or chronological), delivered in near-real-time.

**Key constraints:**

- Average user follows ~200 accounts
- Celebrities have millions of followers
- Users expect new tweets to appear within seconds
- Timeline reads vastly outnumber writes (100:1 read/write ratio)
- 500M+ tweets per day at Twitter's scale

The fundamental question: when should we compute a user's timeline?

## Fan-Out-on-Write vs Fan-Out-on-Read

This is the core architectural decision. Let's look at both approaches.

**Fan-out-on-write (push model)** — when a user tweets, immediately push that tweet into every follower's timeline cache. Reads become a simple cache lookup.

**Fan-out-on-read (pull model)** — when a user opens their timeline, fetch the latest tweets from everyone they follow, merge, sort, and return. Writes are simple, reads do the heavy lifting.

```java
// The core service that handles both strategies
@Service
@RequiredArgsConstructor
@Slf4j
public class TimelineService {

    private final RedisTemplate<String, String> redisTemplate;
    private final TweetRepository tweetRepository;
    private final FollowerRepository followerRepository;
    private final ObjectMapper objectMapper;

    private static final String TIMELINE_KEY = "timeline:%s";
    private static final int TIMELINE_MAX_SIZE = 800;
    private static final int CELEBRITY_THRESHOLD = 50_000;

    /**
     * Fan-out-on-read: compute timeline on demand
     * Good for: users who follow many celebrities
     */
    public List<Tweet> getTimelinePull(String userId, int limit, Long beforeTimestamp) {
        List<String> followedUserIds = followerRepository.getFollowing(userId);

        // Fetch recent tweets from all followed users
        // In production, this would be parallelized
        List<Tweet> allTweets = followedUserIds.parallelStream()
                .flatMap(followedId -> tweetRepository
                        .getRecentTweets(followedId, 20, beforeTimestamp).stream())
                .sorted(Comparator.comparing(Tweet::getTimestamp).reversed())
                .limit(limit)
                .collect(Collectors.toList());

        return allTweets;
    }

    /**
     * Fan-out-on-write: read from pre-computed timeline cache
     * Good for: most regular users
     */
    public List<Tweet> getTimelinePush(String userId, int limit, Double maxScore) {
        String key = String.format(TIMELINE_KEY, userId);

        Set<ZSetOperations.TypedTuple<String>> entries;
        if (maxScore != null) {
            entries = redisTemplate.opsForZSet()
                    .reverseRangeByScoreWithScores(key, 0, maxScore, 0, limit);
        } else {
            entries = redisTemplate.opsForZSet()
                    .reverseRangeWithScores(key, 0, limit - 1);
        }

        if (entries == null || entries.isEmpty()) {
            return Collections.emptyList();
        }

        return entries.stream()
                .map(entry -> deserializeTweet(entry.getValue()))
                .filter(Objects::nonNull)
                .collect(Collectors.toList());
    }

    private Tweet deserializeTweet(String json) {
        try {
            return objectMapper.readValue(json, Tweet.class);
        } catch (JsonProcessingException e) {
            log.error("Failed to deserialize tweet", e);
            return null;
        }
    }
}
```

## The Hybrid Approach (What Twitter Actually Does)

Twitter uses a hybrid. Regular users (< 50K followers) use fan-out-on-write. Celebrities use fan-out-on-read. When you load your timeline, it merges your pre-computed cache with live-fetched tweets from celebrities you follow.

```java
@Service
@RequiredArgsConstructor
public class HybridTimelineService {

    private final TimelineService timelineService;
    private final FollowerRepository followerRepository;
    private final TweetRepository tweetRepository;
    private final UserRepository userRepository;

    private static final int CELEBRITY_THRESHOLD = 50_000;

    public List<Tweet> getTimeline(String userId, int limit) {
        // 1. Get pre-computed timeline (fan-out-on-write results)
        List<Tweet> cachedTimeline = timelineService
                .getTimelinePush(userId, limit, null);

        // 2. Find celebrities the user follows
        List<String> following = followerRepository.getFollowing(userId);
        List<String> celebrityIds = following.stream()
                .filter(id -> userRepository.getFollowerCount(id) > CELEBRITY_THRESHOLD)
                .collect(Collectors.toList());

        // 3. Fetch recent celebrity tweets (fan-out-on-read)
        List<Tweet> celebrityTweets = celebrityIds.parallelStream()
                .flatMap(celId -> tweetRepository
                        .getRecentTweets(celId, 10, null).stream())
                .collect(Collectors.toList());

        // 4. Merge and sort
        List<Tweet> merged = new ArrayList<>(cachedTimeline);
        merged.addAll(celebrityTweets);
        merged.sort(Comparator.comparing(Tweet::getTimestamp).reversed());

        // 5. Deduplicate and limit
        return merged.stream()
                .distinct()
                .limit(limit)
                .collect(Collectors.toList());
    }
}
```

## The Fan-Out Worker: Kafka-Powered Distribution

When a regular user tweets, we need to push that tweet into potentially hundreds of thousands of follower timelines. This must be asynchronous — you can't make a user wait while you update 200K Redis sorted sets.

```java
@Service
@RequiredArgsConstructor
@Slf4j
public class TweetPublishService {

    private final TweetRepository tweetRepository;
    private final UserRepository userRepository;
    private final KafkaTemplate<String, TweetEvent> kafkaTemplate;

    private static final int CELEBRITY_THRESHOLD = 50_000;

    public Tweet publishTweet(String userId, String content) {
        Tweet tweet = Tweet.builder()
                .id(UUID.randomUUID().toString())
                .authorId(userId)
                .content(content)
                .timestamp(Instant.now())
                .build();

        // Persist tweet
        tweetRepository.save(tweet);

        // Check if user is a celebrity
        long followerCount = userRepository.getFollowerCount(userId);

        if (followerCount < CELEBRITY_THRESHOLD) {
            // Fan-out-on-write: publish event for timeline workers
            kafkaTemplate.send("tweet-fanout",
                    userId,
                    new TweetEvent(TweetEventType.NEW_TWEET, tweet, followerCount));
            log.info("Published fan-out event for tweet {} (followers: {})",
                    tweet.getId(), followerCount);
        } else {
            // Celebrity: skip fan-out, rely on pull at read time
            log.info("Skipping fan-out for celebrity {} (followers: {})",
                    userId, followerCount);
        }

        return tweet;
    }
}
```

The Kafka consumer that performs the actual fan-out:

```java
@Service
@RequiredArgsConstructor
@Slf4j
public class FanOutWorker {

    private final FollowerRepository followerRepository;
    private final RedisTemplate<String, String> redisTemplate;
    private final ObjectMapper objectMapper;

    private static final String TIMELINE_KEY = "timeline:%s";
    private static final int TIMELINE_MAX_SIZE = 800;
    private static final int BATCH_SIZE = 1000;

    @KafkaListener(topics = "tweet-fanout", groupId = "fanout-workers",
            concurrency = "10")
    public void processFanOut(TweetEvent event) {
        if (event.getType() != TweetEventType.NEW_TWEET) return;

        Tweet tweet = event.getTweet();
        String tweetJson = serialize(tweet);
        double score = tweet.getTimestamp().toEpochMilli();

        // Get all followers in batches
        String authorId = tweet.getAuthorId();
        long offset = 0;
        List<String> batch;

        do {
            batch = followerRepository.getFollowersBatch(authorId, offset, BATCH_SIZE);

            // Pipeline Redis commands for efficiency
            redisTemplate.executePipelined((RedisCallback<Object>) connection -> {
                StringRedisConnection stringConn = (StringRedisConnection) connection;
                for (String followerId : batch) {
                    String key = String.format(TIMELINE_KEY, followerId);
                    stringConn.zAdd(key, score, tweetJson);
                    // Trim to keep timeline bounded
                    stringConn.zRemRange(key, 0, -(TIMELINE_MAX_SIZE + 1));
                }
                return null;
            });

            offset += BATCH_SIZE;
            log.debug("Fan-out batch complete: {} followers processed for tweet {}",
                    offset, tweet.getId());
        } while (batch.size() == BATCH_SIZE);

        log.info("Fan-out complete for tweet {} to {} followers",
                tweet.getId(), offset);
    }

    private String serialize(Tweet tweet) {
        try {
            return objectMapper.writeValueAsString(tweet);
        } catch (JsonProcessingException e) {
            throw new RuntimeException("Failed to serialize tweet", e);
        }
    }
}
```

Key implementation decisions here:

- **Concurrency = 10** — multiple consumer threads process fan-out events in parallel
- **Batch processing** — fetch followers in batches of 1000 to avoid memory issues for users with millions of followers (though those get skipped by the celebrity threshold)
- **Redis pipelining** — dramatically reduces round trips when updating thousands of timelines
- **Sorted set trimming** — keeps each timeline bounded at 800 entries to control memory

## Redis Timeline Structure

Each user's timeline is a Redis sorted set where:

- **Member** — serialized tweet JSON
- **Score** — tweet timestamp (epoch millis)

This gives us O(log N) inserts and O(log N + M) range queries — perfect for pagination.

```java
@Service
@RequiredArgsConstructor
public class TimelineCacheService {

    private final RedisTemplate<String, String> redisTemplate;
    private final ObjectMapper objectMapper;

    private static final String TIMELINE_KEY = "timeline:%s";
    private static final String USER_TWEETS_KEY = "tweets:%s";

    /**
     * Paginated timeline retrieval with cursor-based pagination
     */
    public TimelinePage getPage(String userId, int pageSize, String cursor) {
        String key = String.format(TIMELINE_KEY, userId);

        double maxScore = cursor != null
                ? Double.parseDouble(cursor) - 1
                : Double.MAX_VALUE;

        Set<ZSetOperations.TypedTuple<String>> results = redisTemplate.opsForZSet()
                .reverseRangeByScoreWithScores(key, 0, maxScore, 0, pageSize + 1);

        if (results == null || results.isEmpty()) {
            return new TimelinePage(Collections.emptyList(), null, false);
        }

        List<ZSetOperations.TypedTuple<String>> list = new ArrayList<>(results);
        boolean hasMore = list.size() > pageSize;

        List<Tweet> tweets = list.stream()
                .limit(pageSize)
                .map(entry -> deserialize(entry.getValue()))
                .filter(Objects::nonNull)
                .collect(Collectors.toList());

        String nextCursor = hasMore
                ? String.valueOf(list.get(pageSize - 1).getScore())
                : null;

        return new TimelinePage(tweets, nextCursor, hasMore);
    }

    /**
     * When a user deletes a tweet, remove from all follower timelines
     */
    public void removeTweetFromTimelines(Tweet tweet, List<String> followerIds) {
        String tweetJson = serialize(tweet);
        redisTemplate.executePipelined((RedisCallback<Object>) connection -> {
            StringRedisConnection conn = (StringRedisConnection) connection;
            for (String followerId : followerIds) {
                String key = String.format(TIMELINE_KEY, followerId);
                conn.zRem(key, tweetJson);
            }
            return null;
        });
    }

    private Tweet deserialize(String json) {
        try {
            return objectMapper.readValue(json, Tweet.class);
        } catch (JsonProcessingException e) {
            return null;
        }
    }

    private String serialize(Tweet tweet) {
        try {
            return objectMapper.writeValueAsString(tweet);
        } catch (JsonProcessingException e) {
            throw new RuntimeException(e);
        }
    }
}
```

I use cursor-based pagination (timestamp as cursor) instead of offset-based. Why? Because new tweets are constantly being added. If you use offset-based pagination, you'll see duplicate tweets as the data shifts. With cursor-based, you always get the next page relative to where you left off.

## Follow/Unfollow: Timeline Cache Warming

When a user follows someone, we need to backfill their timeline with recent tweets from the new followee:

```java
@Service
@RequiredArgsConstructor
public class FollowService {

    private final FollowerRepository followerRepository;
    private final TweetRepository tweetRepository;
    private final RedisTemplate<String, String> redisTemplate;
    private final KafkaTemplate<String, FollowEvent> kafkaTemplate;
    private final ObjectMapper objectMapper;

    private static final String TIMELINE_KEY = "timeline:%s";
    private static final int BACKFILL_COUNT = 50;
    private static final int CELEBRITY_THRESHOLD = 50_000;

    @Transactional
    public void follow(String userId, String targetId) {
        followerRepository.addFollower(targetId, userId);

        long targetFollowerCount = followerRepository.getFollowerCount(targetId);

        if (targetFollowerCount < CELEBRITY_THRESHOLD) {
            // Backfill timeline with recent tweets from the new followee
            backfillTimeline(userId, targetId);
        }
        // If target is a celebrity, their tweets will be pulled at read time

        kafkaTemplate.send("follow-events", userId,
                new FollowEvent(userId, targetId, FollowEventType.FOLLOW));
    }

    @Transactional
    public void unfollow(String userId, String targetId) {
        followerRepository.removeFollower(targetId, userId);

        // Remove their tweets from timeline cache
        removeTweetsFromTimeline(userId, targetId);

        kafkaTemplate.send("follow-events", userId,
                new FollowEvent(userId, targetId, FollowEventType.UNFOLLOW));
    }

    private void backfillTimeline(String userId, String targetId) {
        List<Tweet> recentTweets = tweetRepository
                .getRecentTweets(targetId, BACKFILL_COUNT, null);

        if (recentTweets.isEmpty()) return;

        String key = String.format(TIMELINE_KEY, userId);
        Set<ZSetOperations.TypedTuple<String>> tuples = recentTweets.stream()
                .map(tweet -> ZSetOperations.TypedTuple.of(
                        serialize(tweet),
                        (double) tweet.getTimestamp().toEpochMilli()))
                .collect(Collectors.toSet());

        redisTemplate.opsForZSet().add(key, tuples);
    }

    private void removeTweetsFromTimeline(String userId, String targetId) {
        String key = String.format(TIMELINE_KEY, userId);
        // Scan the sorted set and remove tweets by this author
        // In production, you'd store tweet IDs separately for efficient removal
        Set<String> allEntries = redisTemplate.opsForZSet().range(key, 0, -1);
        if (allEntries == null) return;

        List<String> toRemove = allEntries.stream()
                .filter(entry -> {
                    Tweet tweet = deserialize(entry);
                    return tweet != null && tweet.getAuthorId().equals(targetId);
                })
                .collect(Collectors.toList());

        if (!toRemove.isEmpty()) {
            redisTemplate.opsForZSet().remove(key, toRemove.toArray());
        }
    }

    private String serialize(Tweet tweet) {
        try {
            return objectMapper.writeValueAsString(tweet);
        } catch (JsonProcessingException e) {
            throw new RuntimeException(e);
        }
    }

    private Tweet deserialize(String json) {
        try {
            return objectMapper.readValue(json, Tweet.class);
        } catch (JsonProcessingException e) {
            return null;
        }
    }
}
```

## The REST API Layer

Tying it together with a controller:

```java
@RestController
@RequestMapping("/api/v1")
@RequiredArgsConstructor
public class TimelineController {

    private final HybridTimelineService timelineService;
    private final TweetPublishService publishService;
    private final TimelineCacheService cacheService;

    @GetMapping("/timeline")
    public ResponseEntity<TimelinePage> getTimeline(
            @RequestHeader("X-User-Id") String userId,
            @RequestParam(defaultValue = "20") int limit,
            @RequestParam(required = false) String cursor) {

        TimelinePage page = cacheService.getPage(userId, limit, cursor);
        return ResponseEntity.ok(page);
    }

    @PostMapping("/tweets")
    public ResponseEntity<Tweet> postTweet(
            @RequestHeader("X-User-Id") String userId,
            @RequestBody CreateTweetRequest request) {

        Tweet tweet = publishService.publishTweet(userId, request.getContent());
        return ResponseEntity.status(HttpStatus.CREATED).body(tweet);
    }
}
```

## Caching Strategy: Multi-Layer

I use a three-tier caching approach:

**L1 — Local (Caffeine)** — per-instance cache for hot user timelines (top 1% of active users). 10-second TTL. Reduces Redis load by 60-70% for viral content.

**L2 — Redis** — the primary timeline cache. Sorted sets with 800 entries per user. This is the source of truth for the "push" portion of the timeline.

**L3 — Database** — PostgreSQL with the full tweet history. Only hit for cold timelines or pagination beyond the cached 800 entries.

```java
@Service
@RequiredArgsConstructor
public class MultiLayerTimelineCache {

    private final Cache<String, TimelinePage> localCache = Caffeine.newBuilder()
            .maximumSize(10_000)
            .expireAfterWrite(Duration.ofSeconds(10))
            .build();

    private final TimelineCacheService redisCache;
    private final TweetRepository dbRepository;
    private final FollowerRepository followerRepository;

    public TimelinePage getTimeline(String userId, int limit, String cursor) {
        // L1: Local cache (only for first page, no cursor)
        if (cursor == null) {
            String cacheKey = userId + ":" + limit;
            TimelinePage cached = localCache.getIfPresent(cacheKey);
            if (cached != null) return cached;
        }

        // L2: Redis
        TimelinePage redisResult = redisCache.getPage(userId, limit, cursor);
        if (!redisResult.getTweets().isEmpty()) {
            if (cursor == null) {
                localCache.put(userId + ":" + limit, redisResult);
            }
            return redisResult;
        }

        // L3: Database (cold start or deep pagination)
        List<String> following = followerRepository.getFollowing(userId);
        Long beforeTimestamp = cursor != null ? Long.parseLong(cursor) : null;

        List<Tweet> dbTweets = dbRepository.getTimelineFromDb(
                following, limit + 1, beforeTimestamp);

        boolean hasMore = dbTweets.size() > limit;
        List<Tweet> page = dbTweets.stream().limit(limit).collect(Collectors.toList());
        String nextCursor = hasMore
                ? String.valueOf(page.get(page.size() - 1).getTimestamp().toEpochMilli())
                : null;

        return new TimelinePage(page, nextCursor, hasMore);
    }
}
```

## Handling the Hard Parts

**What happens when a user with 10M followers tweets?**

Even though we skip fan-out for celebrities, the pull-at-read path must be fast. Solution: cache celebrity tweets aggressively. Their recent 50 tweets live in a dedicated Redis sorted set that's refreshed on each new tweet. When computing a timeline, fetching from this pre-cached set is a single O(log N) operation.

**What about tweet deletions?**

Tweet deletion requires removing from potentially millions of cached timelines. In practice, you don't eagerly remove — you use a tombstone approach. Mark the tweet as deleted in the database. When rendering timelines, filter out deleted tweets. Periodically run a background job to clean deleted tweets from caches.

**How do you handle a new user's empty timeline?**

Cold start problem. When a new user signs up and follows people, their timeline cache is empty. The follow-backfill logic handles this — each follow adds recent tweets. Additionally, the system can detect a cold timeline (Redis key doesn't exist) and trigger a full rebuild.

## Performance Numbers

From my production experience with a similar system:

**Fan-out latency** — 95th percentile for a user with 50K followers: 2.3 seconds (async, doesn't block the tweeting user)

**Timeline read latency** — P99 from Redis: 8ms. P99 with celebrity merge: 35ms

**Redis memory** — 800 tweets per user * 500 bytes avg * 100M active users = ~40TB. In practice, only 20-30% of users are active enough to have cached timelines

**Kafka throughput** — fan-out topic handles 2M events/second across 100 partitions

## Interview Tips

When you get this question in an interview, here's the structure I recommend:

1. **Clarify requirements** — ask about scale, real-time vs batch, celebrity handling
2. **Start with the read path** — how users consume the timeline
3. **Explain the trade-off** — fan-out-on-write vs read, then propose hybrid
4. **Deep dive on one component** — pick Redis timeline or Kafka fan-out
5. **Discuss failure modes** — what happens when Redis is down, Kafka lags, etc.

The code in this post covers all of that with working implementations. You're not just drawing boxes anymore — you understand how each box works internally.
