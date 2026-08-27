---
title: "The N+1 Query Problem — And 4 Ways to Kill It"
date: 2026-08-25
categories: [SQL, Performance]
tags: [sql, postgresql, performance, jpa, hibernate, tutorial]
description: "The most common performance bug in ORM-backed applications, explained from the SQL up — what N+1 is, how to detect it, and four concrete fixes with JPA/Hibernate examples."
---
## The Bug That Passes Every Unit Test

The N+1 query problem is the single most common performance issue I see in applications backed by an ORM — Hibernate, JPA, ActiveRecord, Django ORM, Entity Framework, all of them. It's insidious because it's invisible in development. With 10 rows of test data it runs fine. In production, with 10,000 rows, it fires 10,001 queries and the endpoint falls over.

Let me build it from the SQL level first, because once you see the raw queries, the ORM behavior stops being magic.

```sql
CREATE TABLE authors (
    id   BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    name TEXT NOT NULL
);

CREATE TABLE books (
    id        BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    author_id BIGINT NOT NULL REFERENCES authors(id),
    title     TEXT NOT NULL,
    price_cents BIGINT NOT NULL
);

INSERT INTO authors (name)
SELECT 'Author ' || g FROM generate_series(1, 1000) g;

INSERT INTO books (author_id, title, price_cents)
SELECT 1 + (random() * 999)::bigint,
       'Book ' || g,
       (1000 + random() * 5000)::bigint
FROM generate_series(1, 20000) g;

ANALYZE authors;
ANALYZE books;
```

---

## What N+1 Actually Looks Like in SQL

Say you want to list every author with their books. The N+1 pattern issues one query to fetch the authors, then one more query *per author* to fetch that author's books:

```sql
-- Query 1: fetch all authors (the "1")
SELECT id, name FROM authors;
-- returns 1000 rows

-- Then, for EACH author, a separate query (the "N"):
SELECT id, title, price_cents FROM books WHERE author_id = 1;
SELECT id, title, price_cents FROM books WHERE author_id = 2;
SELECT id, title, price_cents FROM books WHERE author_id = 3;
-- ... 997 more times
```

That's **1 + 1000 = 1001 round trips** to the database. Each round trip has fixed overhead: network latency, query parsing, planning, result marshaling. Even if each query is 1ms, that's over a second of pure round-trip cost, and it grows linearly with your data. This is the "N+1": one initial query plus N follow-ups.

The equivalent single query returns the same data in one round trip:

```sql
SELECT a.id, a.name, b.id AS book_id, b.title, b.price_cents
FROM authors a
LEFT JOIN books b ON b.author_id = a.id
ORDER BY a.id;
```

```
Hash Right Join  (actual time=1.2..12.4 rows=20000 loops=1)
  Hash Cond: (b.author_id = a.id)
  ->  Seq Scan on books b (actual time=0.01..3.1 rows=20000 loops=1)
  ->  Hash (actual time=0.9..0.9 rows=1000 loops=1)
        ->  Seq Scan on authors a (rows=1000 loops=1)
Execution Time: 13.9 ms
```

One query, 14ms. Versus 1001 queries and a second-plus of latency. Same data. That gap is the entire problem.

---

## How It Sneaks In Through JPA/Hibernate

In Java, you rarely write those 1001 queries by hand. Hibernate writes them for you, silently, because of lazy loading. Here's the classic setup:

```java
@Entity
public class Author {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    private String name;

    @OneToMany(mappedBy = "author", fetch = FetchType.LAZY)
    private List<Book> books;
}

@Entity
public class Book {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "author_id")
    private Author author;

    private String title;
    private Long priceCents;
}
```

Now this innocent-looking service code:

```java
List<Author> authors = authorRepository.findAll();   // SELECT * FROM authors  -> 1 query
for (Author author : authors) {
    // Accessing the lazy collection triggers a query PER author
    System.out.println(author.getName() + ": " + author.getBooks().size());  // N queries
}
```

`findAll()` runs one query. But `author.getBooks()` is a lazy collection — the first time you touch it, Hibernate fires a `SELECT * FROM books WHERE author_id = ?`. Loop over 1000 authors and you've fired 1000 extra queries. Nobody wrote them; the ORM did, one property access at a time.

The trap: switching `fetch = FetchType.EAGER` does **not** fix it. Eager fetching on a collection just moves the N+1 from your loop into the `findAll()` call itself — Hibernate still fires a query per author to hydrate the collections. Eager is not a fix; it's the same bug that fires earlier and can't be turned off.

---

## Detecting N+1 Before Production Does

You can't fix what you can't see. Turn on SQL logging in dev and *count the queries*.

**Hibernate statistics** — the most reliable signal:

```properties
# application.properties (dev only)
spring.jpa.properties.hibernate.generate_statistics=true
logging.level.org.hibernate.stat=DEBUG
```

After a request you'll see something like:

```
Session Metrics {
    1000000 nanoseconds spent preparing 1001 JDBC statements;
    ...
}
```

`1001 JDBC statements` for one endpoint is a screaming N+1.

**See the actual SQL** with a formatting proxy like p6spy or datasource-proxy (cleaner than `show-sql`):

```properties
# datasource-proxy / p6spy will log each statement with timing
logging.level.net.ttddyy.dsproxy.listener=DEBUG
```

**Best of all — assert query counts in a test** so a regression fails CI, not production:

```java
@Test
void listingAuthorsShouldNotTriggerNPlusOne() {
    QueryCountHolder.clear();

    authorService.listAuthorsWithBooks();

    // One query for authors + books, not 1 + N
    assertThat(QueryCountHolder.getGrandTotal().getSelect())
        .isLessThanOrEqualTo(2);
}
```

That single assertion catches the most expensive bug class in ORM apps before it ships.

---

## Fix #1 — JOIN FETCH (JPQL)

Tell Hibernate to load the association in the same query using `JOIN FETCH`. This is the direct translation of "just write the join."

```java
public interface AuthorRepository extends JpaRepository<Author, Long> {

    @Query("SELECT DISTINCT a FROM Author a LEFT JOIN FETCH a.books")
    List<Author> findAllWithBooks();
}
```

Hibernate emits one SQL statement:

```sql
SELECT a.id, a.name, b.id, b.author_id, b.title, b.price_cents
FROM authors a
LEFT OUTER JOIN books b ON b.author_id = a.id;
```

One round trip instead of 1001. Two things to know:

- Use `LEFT JOIN FETCH` if you want authors with zero books included; plain `JOIN FETCH` drops them (inner join).
- `DISTINCT` deduplicates the author objects (a join with a to-many multiplies the parent row per child). In modern Hibernate you can also set `hibernate.query.passDistinctThrough = false` so `DISTINCT` de-dupes in memory without adding `DISTINCT` to the SQL.

**Limitation:** you cannot `JOIN FETCH` two separate collections in one query without a cartesian explosion (fetching `books` *and* `awards` multiplies them together). For that, use fix #3.

---

## Fix #2 — Entity Graphs

`JOIN FETCH` bakes the fetch plan into a specific query. An `@EntityGraph` lets you attach a fetch plan to a normal repository method declaratively — handy when you want the same finder both with and without the association.

```java
public interface AuthorRepository extends JpaRepository<Author, Long> {

    @EntityGraph(attributePaths = {"books"})
    @Query("SELECT a FROM Author a")
    List<Author> findAllWithBooksGraph();

    // Works on derived queries too
    @EntityGraph(attributePaths = {"books"})
    List<Author> findByNameContaining(String fragment);
}
```

Or define a reusable named graph on the entity:

```java
@Entity
@NamedEntityGraph(
    name = "Author.withBooks",
    attributeNodes = @NamedAttributeNode("books")
)
public class Author { /* ... */ }
```

```java
@EntityGraph(value = "Author.withBooks", type = EntityGraphType.LOAD)
List<Author> findAll();
```

Under the hood Hibernate turns the graph into a join, same as `JOIN FETCH`, but the fetch strategy is separated from the query text. Great for reusing one finder across endpoints with different loading needs.

---

## Fix #3 — Batch Fetching (the N+1 becomes N/k+1)

Sometimes you can't restructure the query — the N+1 is buried deep in existing code you can't easily refactor, or you need to fetch multiple collections. Batch fetching keeps the lazy loading but makes Hibernate load the children in *batches* using `IN (...)` instead of one query each.

Global setting (applies everywhere):

```properties
spring.jpa.properties.hibernate.default_batch_fetch_size=100
```

Or per-association:

```java
@OneToMany(mappedBy = "author", fetch = FetchType.LAZY)
@BatchSize(size = 100)
private List<Book> books;
```

Now instead of 1000 individual queries, Hibernate collects up to 100 author IDs and fetches their books in one shot:

```sql
-- Instead of 1000 queries, ~10 queries like this:
SELECT * FROM books WHERE author_id IN (1, 2, 3, ..., 100);
SELECT * FROM books WHERE author_id IN (101, 102, ..., 200);
-- ...
```

1000 authors with batch size 100 → 1 (authors) + 10 (batches) = **11 queries** instead of 1001. Not as tight as a single join, but a huge improvement and it requires almost no code change. This is my go-to when I inherit a codebase full of N+1s and need a fast, low-risk win. It also solves the "two collections" case that `JOIN FETCH` can't.

**MySQL note:** the `IN (...)` batch approach works identically; watch very large `IN` lists, which some drivers/servers cap — batch size 100–500 is a safe range.

---

## Fix #4 — Projections / DTOs (Don't Load Entities at All)

For read-only endpoints you often don't need managed entities with their whole object graph — you need a flat DTO with a handful of fields. Fetch exactly that, in one query, and skip lazy loading entirely.

Interface-based projection with an explicit query:

```java
public interface AuthorBookView {
    String getAuthorName();
    String getBookTitle();
    Long getPriceCents();
}

public interface AuthorRepository extends JpaRepository<Author, Long> {

    @Query("""
        SELECT a.name AS authorName, b.title AS bookTitle, b.priceCents AS priceCents
        FROM Author a JOIN a.books b
        """)
    List<AuthorBookView> findAuthorBookViews();
}
```

Or a constructor (DTO) projection:

```java
public record AuthorBookDto(String authorName, String bookTitle, Long priceCents) {}

@Query("""
    SELECT new com.example.AuthorBookDto(a.name, b.title, b.priceCents)
    FROM Author a JOIN a.books b
    """)
List<AuthorBookDto> findAuthorBookDtos();
```

Both emit a single join query and return only the columns you asked for:

```sql
SELECT a.name, b.title, b.price_cents
FROM authors a JOIN books b ON b.author_id = a.id;
```

This is the fastest option for read paths. No lazy proxies, no dirty-checking overhead, no accidental N+1 later when someone touches another association — because there are no associations to touch. For list and dashboard endpoints, projections are usually the right answer.

---

## Choosing the Right Fix

| Situation | Best fix |
|---|---|
| Read one collection, need managed entities | `JOIN FETCH` (#1) |
| Same finder, different fetch needs per caller | Entity graph (#2) |
| Legacy code, hard to refactor, or multiple collections | Batch fetching (#3) |
| Read-only endpoint, just need fields | Projection / DTO (#4) |

A rule of thumb: for write flows and detail pages where you'll modify entities, use `JOIN FETCH` or entity graphs so you get managed entities. For list views, dashboards, and APIs, use projections — they're faster and immune to future N+1 regressions.

Avoid two anti-patterns: don't reach for `FetchType.EAGER` (it's a permanent, un-tunable version of the bug), and don't `JOIN FETCH` multiple `@OneToMany` collections in one query (cartesian product) — batch-fetch the second one instead.

---

## A Word on the Raw SQL Underneath

Whatever ORM fix you pick, the goal at the database level is the same: replace N selective lookups with **one join** or **one `IN` batch**. Make sure the join has an index to stand on:

```sql
-- The FK column MUST be indexed or your join degrades to a scan per lookup
CREATE INDEX idx_books_author_id ON books (author_id);
```

PostgreSQL does not auto-index foreign keys, so a missing FK index is a common reason an N+1 fix underperforms — the batch `IN` query or the join still has to scan `books`. Index the FK, then verify with `EXPLAIN ANALYZE` that the join uses it:

```sql
EXPLAIN ANALYZE
SELECT * FROM books WHERE author_id IN (1,2,3,4,5);
```

```
Index Scan using idx_books_author_id on books (actual time=0.02..0.11 rows=98 loops=1)
  Index Cond: (author_id = ANY ('{1,2,3,4,5}'::bigint[]))
```

---

## Final Thought

N+1 is not really an ORM bug — it's what happens when object-graph navigation quietly translates into per-row SQL. The database was always capable of answering in one join; the ORM just didn't know you wanted it to.

Turn on query counting in your tests so N+1 fails CI instead of production. Then pick your fix by intent: joins and graphs for entities you'll modify, batch fetching for legacy rescue, projections for read paths. And always index your foreign keys — the best N+1 fix in the world still needs an index to land on.
