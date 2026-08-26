---
title: "How I Ace System Design Interviews — My Complete Framework"
date: 2026-08-24
categories: [System Design, Interviews]
tags: [system-design, interview-prep, software-engineering, career, java]
description: "The exact 5-step framework I've used across 17 years of system design interviews — both as a candidate and as a hiring manager — with a fully worked URL shortener example."
mermaid: true
---
## Why Most Candidates Fail System Design Interviews

I've been on both sides of the system design interview table for 17 years. As a Technical Architect in Singapore, I've interviewed hundreds of senior candidates. The failure pattern is remarkably consistent.

Candidates don't fail because they lack knowledge. They fail because they lack structure. They jump into database schemas before understanding requirements. They draw boxes without explaining tradeoffs. They design for Google's scale when the problem calls for a startup's.

The framework I'm sharing here isn't theoretical. It's the exact mental model I use when I'm the candidate, and it's what I look for when I'm the interviewer.

---

## The 5-Step Framework

**Step 1** — Clarify Requirements (5 minutes)

**Step 2** — Estimate Scale (5 minutes)

**Step 3** — High-Level Design (10 minutes)

**Step 4** — Deep Dive (15 minutes)

**Step 5** — Identify Bottlenecks and Improvements (5 minutes)

The time allocation matters. I've seen candidates spend 25 minutes on requirements and run out of time for the actual design. I've also seen candidates jump to a load balancer diagram in the first minute without knowing if the system serves 100 users or 100 million.

Let me walk through each step, then apply the entire framework to a real problem.

---

## Step 1: Clarify Requirements

This is where you separate yourself from junior candidates. Seniors don't ask generic questions. They ask questions that reveal constraints.

### Functional Requirements (What does it do?)

Ask about:
- **Core use cases** — What are the 2-3 things users do most?
- **Actors** — Who uses this? End users? Internal services? Both?
- **Input/Output** — What goes in, what comes out?
- **Edge cases** — What happens when X fails? Is data loss acceptable?

### Non-Functional Requirements (How well does it do it?)

Ask about:
- **Scale** — How many users? How many requests per second?
- **Latency** — What's the acceptable P99 latency?
- **Availability** — 99.9%? 99.99%? This changes everything
- **Consistency** — Strong consistency or eventual consistency acceptable?
- **Data retention** — How long do we keep data? What's the storage growth?

### What I Actually Say in an Interview

"Before I design anything, let me make sure I understand the problem correctly. I'm going to ask a few clarifying questions that will drive my architectural decisions."

This single sentence tells the interviewer: this person thinks before they code.

---

## Step 2: Estimate Scale

Back-of-envelope calculations serve two purposes: they size your infrastructure AND they demonstrate quantitative thinking.

### My Estimation Template

- **Users** — total registered, daily active (DAU), peak concurrent
- **Read:Write ratio** — this determines your architecture fundamentally
- **Data size** — per record × records per day × retention period
- **Bandwidth** — requests per second × average payload size
- **Storage** — daily growth × years of retention

### Numbers Every Engineer Should Know

**Latency numbers:**
- L1 cache reference — 0.5 ns
- Main memory reference — 100 ns
- SSD random read — 150 μs
- Network round trip (same datacenter) — 0.5 ms
- Network round trip (cross-continent) — 150 ms

**Scale numbers:**
- 1 million requests/day ≈ 12 requests/second
- 100 million requests/day ≈ 1,200 requests/second
- 1 billion requests/day ≈ 12,000 requests/second

**Storage numbers:**
- 1 character = 1 byte (ASCII) or 2-4 bytes (UTF-8)
- 1 million rows × 1 KB each = 1 GB
- 1 billion rows × 1 KB each = 1 TB

These approximations are good enough. Interviewers want to see you think in orders of magnitude, not exact figures.

---

## Step 3: High-Level Design

Start with the simplest architecture that solves the problem. Then evolve it.

### My Drawing Approach

1. **Start with the client** — mobile, web, or API consumer
2. **Add the entry point** — load balancer, API gateway
3. **Core services** — the 2-3 services that handle primary business logic
4. **Data stores** — choose based on access patterns (relational, document, cache)
5. **Async components** — message queues, background workers

### What I Say While Drawing

"Let me start with the simplest version that handles our core use case, then we'll evolve it to handle scale."

This is critical. If you draw 15 boxes immediately, you can't explain why each one exists. Start with 3-4 components and add complexity with justification.

### API Design First

Before drawing boxes, I define the API contract:

```
POST /api/v1/resource
GET /api/v1/resource/{id}
DELETE /api/v1/resource/{id}
```

This forces clarity on what the system actually does before you decide how it does it.

---

## Step 4: Deep Dive

This is where senior candidates shine. Pick 2-3 components and go deep.

### What to Deep Dive On

- **The component the interviewer asks about** — always prioritize their interest
- **The hardest scaling challenge** — where does this break at 10x traffic?
- **The data model** — schema design, indexing strategy, partitioning
- **The consistency model** — what happens during network partitions?

### How to Structure a Deep Dive

1. **State the problem** — "The challenge here is X"
2. **Present options** — "We could do A, B, or C"
3. **Evaluate tradeoffs** — "A gives us X but costs us Y"
4. **Decide and justify** — "I'd go with B because in our context..."

This is the difference between a senior and a mid-level answer. Mid-level candidates state a solution. Seniors present a decision with reasoning.

---

## Step 5: Identify Bottlenecks and Improvements

End strong by proactively identifying what could go wrong.

### My Checklist

- **Single points of failure** — what happens when this component dies?
- **Hot spots** — is one partition/shard getting all the traffic?
- **Data growth** — what happens in 2 years when we have 10x data?
- **Latency spikes** — what's the worst-case path through the system?
- **Operational concerns** — how do we deploy, monitor, and debug this?

### What I Say

"If I had more time, here's what I'd address next: [list 2-3 improvements]. The most critical one is X because..."

This shows the interviewer you know the design isn't perfect and you have the maturity to prioritize improvements.

---

## Worked Example: Design a URL Shortener

Let me apply the full framework to a classic problem.

### Step 1: Clarify Requirements

**Functional:**
- Given a long URL, generate a short URL
- Given a short URL, redirect to the original URL
- Optional: custom aliases, expiration, analytics

**Non-Functional:**
- Read-heavy (100:1 read-to-write ratio)
- URL redirection must be fast (<100ms P99)
- High availability (99.99%) — downtime means broken links everywhere
- Shortened URLs should be as short as possible

**Scope decisions:**
- Focus on core shortening + redirection
- Analytics as a follow-up if time permits
- No authentication required for creating URLs (public service)

### Step 2: Estimate Scale

- 100 million new URLs per month (write)
- 10 billion redirections per month (read) — 100:1 ratio
- Write QPS: 100M / (30 × 24 × 3600) ≈ 40 URLs/second
- Read QPS: 10B / (30 × 24 × 3600) ≈ 4,000 redirects/second
- Peak: 5× average ≈ 20,000 redirects/second

**Storage:**
- Each URL record: ~500 bytes (short URL + long URL + metadata)
- 100M/month × 12 months × 5 years = 6 billion records
- 6B × 500 bytes = 3 TB total storage

**Short URL length:**
- Using base62 (a-z, A-Z, 0-9): 62^7 = 3.5 trillion combinations
- 7 characters is sufficient for years of growth

### Step 3: High-Level Design

```
Client → Load Balancer → API Service → Cache (Redis)
                                      → Database (Primary + Replicas)
                                      → ID Generator Service
```

**API Design:**

```
POST /api/v1/shorten
  Request: { "long_url": "https://...", "custom_alias": "optional" }
  Response: { "short_url": "https://short.ly/abc1234" }

GET /{short_url_key}
  Response: HTTP 301 Redirect to original URL
```

**Why 301 vs 302?**
- 301 (Permanent Redirect): Browser caches it — reduces server load but kills analytics
- 302 (Temporary Redirect): Every request hits our server — enables click tracking

Decision: Use 302 to maintain analytics capability.

### Step 4: Deep Dive

**Deep Dive 1: URL Key Generation**

Option A — Hash the long URL (MD5/SHA-256, take first 7 chars)
- Pro: Same URL always gets the same short URL (deduplication free)
- Con: Hash collisions need handling, 7 chars of MD5 has ~1:62^7 collision rate

Option B — Pre-generated unique IDs (counter-based)
- Pro: Guaranteed unique, no collision handling
- Con: Sequential IDs are predictable (security concern)

Option C — Base62 encoding of a distributed counter
- Pro: Unique, compact, simple
- Con: Need a distributed counter (single point of failure risk)

**My Choice: Snowflake-style ID generation with base62 encoding**

```java
@Service
public class UrlShortenerService {

    private final SnowflakeIdGenerator idGenerator;
    private final UrlRepository urlRepository;
    private final RedisTemplate<String, String> cache;

    public String shorten(String longUrl) {
        // Check if already shortened
        String existing = urlRepository.findByLongUrl(longUrl);
        if (existing != null) return existing;

        // Generate unique key
        long id = idGenerator.nextId();
        String shortKey = Base62.encode(id); // 7-8 chars

        // Store
        urlRepository.save(new UrlMapping(shortKey, longUrl, Instant.now()));
        cache.opsForValue().set(shortKey, longUrl, 24, TimeUnit.HOURS);

        return shortKey;
    }

    public String resolve(String shortKey) {
        // Cache first
        String cached = cache.opsForValue().get(shortKey);
        if (cached != null) return cached;

        // Database fallback
        String longUrl = urlRepository.findByShortKey(shortKey);
        if (longUrl != null) {
            cache.opsForValue().set(shortKey, longUrl, 24, TimeUnit.HOURS);
        }
        return longUrl;
    }
}
```

**Deep Dive 2: Database Schema and Scaling**

```sql
CREATE TABLE url_mappings (
    short_key VARCHAR(10) PRIMARY KEY,
    long_url VARCHAR(2048) NOT NULL,
    created_at TIMESTAMP NOT NULL,
    expires_at TIMESTAMP,
    click_count BIGINT DEFAULT 0
);

CREATE INDEX idx_long_url ON url_mappings(long_url);
```

Scaling strategy:
- **Read replicas** for the 100:1 read ratio — route all resolves to replicas
- **Range-based sharding** on short_key — distributes evenly with base62 keys
- **Redis cluster** in front handles 80%+ of reads (hot URLs get cached)

**Deep Dive 3: Cache Strategy**

- **Write-through** for new URLs: write to DB and cache simultaneously
- **Cache-aside** for reads: check cache first, backfill from DB on miss
- **TTL-based eviction**: 24-hour TTL, popular URLs stay warm via access patterns
- **Cache hit ratio target**: 80%+ (most redirects are for popular links)

### Step 5: Bottlenecks and Improvements

**Bottleneck 1: ID Generator as single point of failure**
- Solution: Deploy multiple ID generators with different ID ranges
- Each generator owns a block of IDs (e.g., generator-1 gets IDs 1-1M, generator-2 gets 1M-2M)

**Bottleneck 2: Hot URLs overwhelming a single cache node**
- Solution: Consistent hashing with replicas across Redis cluster
- Add a local in-memory cache (Caffeine) for the top 1000 URLs

**Bottleneck 3: Database write throughput**
- Solution: Batch writes with a write-behind buffer
- URL creation isn't latency-sensitive (writes can be async)

**Future improvements:**
- Analytics pipeline: stream click events to Kafka → Flink → analytics DB
- Geo-distributed caches for global latency reduction
- Rate limiting on URL creation to prevent abuse

---

## Framework Summary — The Cheat Sheet

**Step 1 (5 min):** Ask functional and non-functional requirements. Confirm scope boundaries.

**Step 2 (5 min):** Calculate QPS, storage, bandwidth. Identify read:write ratio.

**Step 3 (10 min):** Draw simplest architecture. Define API first. Add components with justification.

**Step 4 (15 min):** Deep dive into 2-3 components. Present options, evaluate tradeoffs, decide.

**Step 5 (5 min):** Identify bottlenecks. Propose solutions. Show awareness of operational concerns.

---

## Tips from the Interviewer's Side

Having interviewed hundreds of candidates, here's what separates the top 10%:

- **They drive the conversation.** They don't wait for prompts. They say "Let me tackle the hardest part next" and go.
- **They quantify everything.** Not "it needs to be fast" but "P99 under 100ms at 20K QPS."
- **They acknowledge tradeoffs.** "This gives us availability but we lose strong consistency."
- **They know what they don't know.** "I'd need to benchmark this to be certain, but my intuition is..."
- **They stay practical.** They don't over-engineer. They design for the stated requirements, not for Google's traffic.

The biggest red flag? Candidates who name-drop technologies without explaining why. "We'll use Kafka" without explaining what problem Kafka solves here tells me you're pattern-matching, not thinking.

System design interviews test your ability to make decisions under uncertainty, communicate clearly, and balance competing concerns. The framework gives you structure. Practice gives you fluency.
