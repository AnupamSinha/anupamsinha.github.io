---
title: "Building a Production Search Engine with Elasticsearch and Spring Boot"
date: 2026-08-24
categories: [Spring Boot, Data]
tags: [elasticsearch, spring-boot, search, java, full-text-search]
description: "A complete implementation guide covering index design, custom analyzers, multi-language support, autocomplete, faceted search, relevance tuning, pagination strategies, and zero-downtime reindexing."
mermaid: true
---
## Why I Wrote This

Search is one of those features that looks simple on the surface — "just add a search box" — but hides enormous complexity underneath. After implementing production search for an e-commerce marketplace with 2 million products and 15 million searches per day, I want to share what actually works and what the tutorials skip.

This isn't a "getting started with Elasticsearch" article. This is the production playbook — the mapping decisions, analyzer configurations, relevance tuning, and operational concerns that make the difference between search that delights users and search that frustrates them.

## The Technology Stack

**Elasticsearch** — 8.x (we'll use the new Java client, not the deprecated RestHighLevelClient)

**Spring Boot** — 3.2+ with Spring Data Elasticsearch

**Java** — 21

## Index Design: Get This Right First

Your index mapping is the foundation. Changing it later requires a full reindex, so invest time upfront.

### The Product Mapping

```java
@Document(indexName = "products")
@Setting(settingPath = "/elasticsearch/product-settings.json")
public class ProductDocument {

    @Id
    private String id;

    @MultiField(
        mainField = @Field(type = FieldType.Text, analyzer = "product_analyzer"),
        otherFields = {
            @InnerField(suffix = "keyword", type = FieldType.Keyword),
            @InnerField(suffix = "autocomplete", type = FieldType.Text, 
                       analyzer = "autocomplete_analyzer", searchAnalyzer = "standard"),
            @InnerField(suffix = "exact", type = FieldType.Text, analyzer = "keyword")
        }
    )
    private String name;

    @Field(type = FieldType.Text, analyzer = "product_analyzer")
    private String description;

    @Field(type = FieldType.Keyword)
    private String category;

    @Field(type = FieldType.Keyword)
    private String brand;

    @Field(type = FieldType.Double)
    private double price;

    @Field(type = FieldType.Integer)
    private int stockQuantity;

    @Field(type = FieldType.Float)
    private float averageRating;

    @Field(type = FieldType.Integer)
    private int reviewCount;

    @Field(type = FieldType.Date, format = DateFormat.epoch_millis)
    private long createdAt;

    @Field(type = FieldType.Keyword)
    private List<String> tags;

    @Field(type = FieldType.Nested)
    private List<ProductAttribute> attributes;

    @Field(type = FieldType.Boolean)
    private boolean inStock;

    @Field(type = FieldType.Integer)
    private int salesCount;

    @Field(type = FieldType.Keyword)
    private String sellerId;
}
```

### Custom Analyzers

The default analyzer is rarely good enough for product search. Users type "running shoe", "running shoes", "RUNNING SHOES", and expect the same results.

```json
{
  "analysis": {
    "analyzer": {
      "product_analyzer": {
        "type": "custom",
        "tokenizer": "standard",
        "filter": ["lowercase", "asciifolding", "english_stemmer", "synonym_filter"]
      },
      "autocomplete_analyzer": {
        "type": "custom",
        "tokenizer": "standard",
        "filter": ["lowercase", "asciifolding", "edge_ngram_filter"]
      },
      "search_analyzer": {
        "type": "custom",
        "tokenizer": "standard",
        "filter": ["lowercase", "asciifolding", "english_stemmer", "synonym_filter"]
      }
    },
    "filter": {
      "english_stemmer": {
        "type": "stemmer",
        "language": "english"
      },
      "edge_ngram_filter": {
        "type": "edge_ngram",
        "min_gram": 2,
        "max_gram": 15
      },
      "synonym_filter": {
        "type": "synonym",
        "synonyms_path": "analysis/synonyms.txt"
      }
    }
  },
  "index": {
    "number_of_shards": 3,
    "number_of_replicas": 2,
    "refresh_interval": "1s",
    "max_result_window": 50000
  }
}
```

### Why Multi-Fields Matter

The `name` field has four analyzers for different use cases:

**name (product_analyzer)** — Full-text search with stemming. "running" matches "runs", "runner"

**name.keyword** — Exact match for aggregations and sorting

**name.autocomplete** — Edge n-grams for type-ahead suggestions. "ru" matches "Running Shoes"

**name.exact** — No stemming, for phrase matching where precision matters

## Multi-Language Search

Our marketplace operated in Singapore — English, Chinese, Malay, and Tamil content. Each language needs its own analyzer:

```java
@Configuration
public class MultiLanguageIndexConfig {

    public String createMultiLanguageMapping() {
        return """
        {
          "properties": {
            "name": {
              "type": "text",
              "fields": {
                "en": {
                  "type": "text",
                  "analyzer": "english"
                },
                "zh": {
                  "type": "text",
                  "analyzer": "smartcn"
                },
                "ms": {
                  "type": "text",
                  "analyzer": "malay_analyzer"
                }
              }
            }
          }
        }
        """;
    }
}
```

At query time, we search across all language fields:

```java
public Query buildMultiLanguageQuery(String userQuery, String detectedLanguage) {
    // Boost the detected language field higher
    float primaryBoost = 3.0f;
    float secondaryBoost = 1.0f;

    return NativeQuery.builder()
        .withQuery(q -> q.multiMatch(mm -> mm
            .query(userQuery)
            .fields(List.of(
                "name.en^" + (detectedLanguage.equals("en") ? primaryBoost : secondaryBoost),
                "name.zh^" + (detectedLanguage.equals("zh") ? primaryBoost : secondaryBoost),
                "name.ms^" + (detectedLanguage.equals("ms") ? primaryBoost : secondaryBoost),
                "name^" + secondaryBoost,
                "description^0.5"
            ))
            .type(TextQueryType.BEST_FIELDS)
            .tieBreaker(0.3)
        ))
        .build();
}
```

## Autocomplete with Edge N-Grams

Autocomplete needs to be fast (under 50ms) and return results as the user types each character:

```java
@Service
public class AutocompleteService {

    private final ElasticsearchOperations elasticsearchOperations;

    public List<AutocompleteSuggestion> suggest(String prefix, int limit) {
        var query = NativeQuery.builder()
            .withQuery(q -> q.bool(b -> b
                .should(s -> s.match(m -> m
                    .field("name.autocomplete")
                    .query(prefix)
                    .boost(2.0f)
                ))
                .should(s -> s.match(m -> m
                    .field("brand.autocomplete")
                    .query(prefix)
                    .boost(1.5f)
                ))
                .should(s -> s.match(m -> m
                    .field("category")
                    .query(prefix)
                    .boost(1.0f)
                ))
                .minimumShouldMatch("1")
            ))
            .withSourceFilter(new FetchSourceFilter(
                new String[]{"name", "brand", "category", "id"}, null))
            .withMaxResults(limit)
            .build();

        SearchHits<ProductDocument> hits = elasticsearchOperations.search(
            query, ProductDocument.class);

        return hits.getSearchHits().stream()
            .map(hit -> new AutocompleteSuggestion(
                hit.getContent().getName(),
                hit.getContent().getCategory(),
                hit.getScore()))
            .distinct()
            .toList();
    }
}
```

### The Completion Suggester — An Alternative

For pure prefix matching with extreme performance needs (millions of suggestions), use the completion suggester:

```java
@Field(type = FieldType.Search_As_You_Type)
private String nameSuggest;

// Or using the completion type
@CompletionField(maxInputLength = 100)
private Completion suggest;
```

```java
public List<String> completionSuggest(String prefix) {
    var suggest = SuggestBuilder.builder()
        .suggesters("product-suggest", s -> s
            .prefix(prefix)
            .completion(c -> c
                .field("suggest")
                .size(10)
                .skipDuplicates(true)
                .fuzzy(f -> f.fuzziness("AUTO"))
            )
        )
        .build();

    // Returns suggestions in under 5ms even with millions of documents
    return extractSuggestions(elasticsearchOperations.suggest(suggest, ProductDocument.class));
}
```

**Performance comparison:**

**Edge n-gram query** — 15-40ms, flexible scoring, works with any query

**Completion suggester** — 2-5ms, prefix-only, limited scoring options

We used edge n-grams for the main search box and the completion suggester for category/brand type-ahead fields.

## Faceted Search

Faceted search lets users refine results by category, brand, price range, etc. This is implemented using Elasticsearch aggregations:

```java
@Service
public class FacetedSearchService {

    private final ElasticsearchOperations elasticsearchOperations;

    public SearchResult searchWithFacets(SearchRequest request) {
        var query = NativeQuery.builder()
            .withQuery(buildMainQuery(request.getQuery(), request.getFilters()))
            .withAggregation("categories", buildCategoryAggregation())
            .withAggregation("brands", buildBrandAggregation())
            .withAggregation("price_ranges", buildPriceRangeAggregation())
            .withAggregation("ratings", buildRatingAggregation())
            .withAggregation("in_stock", buildStockAggregation())
            .withPageable(PageRequest.of(request.getPage(), request.getSize()))
            .build();

        SearchHits<ProductDocument> hits = elasticsearchOperations.search(
            query, ProductDocument.class);

        return new SearchResult(
            extractProducts(hits),
            extractFacets(hits.getAggregations()),
            hits.getTotalHits(),
            request.getPage()
        );
    }

    private Aggregation buildCategoryAggregation() {
        return Aggregation.of(a -> a
            .terms(t -> t
                .field("category")
                .size(20)
                .order(List.of(NamedValue.of("_count", SortOrder.Desc)))
            )
        );
    }

    private Aggregation buildPriceRangeAggregation() {
        return Aggregation.of(a -> a
            .range(r -> r
                .field("price")
                .ranges(
                    NumberRangeExpression.of(re -> re.to(25.0).key("Under $25")),
                    NumberRangeExpression.of(re -> re.from(25.0).to(50.0).key("$25 - $50")),
                    NumberRangeExpression.of(re -> re.from(50.0).to(100.0).key("$50 - $100")),
                    NumberRangeExpression.of(re -> re.from(100.0).to(200.0).key("$100 - $200")),
                    NumberRangeExpression.of(re -> re.from(200.0).key("$200+"))
                )
            )
        );
    }

    private Query buildMainQuery(String queryText, Map<String, List<String>> filters) {
        var boolQuery = new BoolQuery.Builder();

        // Main search query
        if (queryText != null && !queryText.isBlank()) {
            boolQuery.must(m -> m.multiMatch(mm -> mm
                .query(queryText)
                .fields(List.of("name^3", "description", "brand^2", "tags^1.5"))
                .type(TextQueryType.BEST_FIELDS)
                .fuzziness("AUTO")
            ));
        }

        // Apply facet filters
        if (filters.containsKey("category")) {
            boolQuery.filter(f -> f.terms(t -> t
                .field("category")
                .terms(tv -> tv.value(
                    filters.get("category").stream()
                        .map(FieldValue::of).toList()
                ))
            ));
        }

        if (filters.containsKey("brand")) {
            boolQuery.filter(f -> f.terms(t -> t
                .field("brand")
                .terms(tv -> tv.value(
                    filters.get("brand").stream()
                        .map(FieldValue::of).toList()
                ))
            ));
        }

        if (filters.containsKey("inStock")) {
            boolQuery.filter(f -> f.term(t -> t
                .field("inStock")
                .value(true)
            ));
        }

        return Query.of(q -> q.bool(boolQuery.build()));
    }
}
```

### Post-Filter for Accurate Facet Counts

A common mistake: applying filters inside the main query affects aggregation counts. Use `post_filter` to filter results without affecting facet counts:

```java
var query = NativeQuery.builder()
    .withQuery(buildTextQuery(request.getQuery())) // Only text match
    .withAggregation("brands", buildBrandAggregation()) // Counts all brands
    .withFilter(buildFilterQuery(request.getFilters())) // post_filter: filters results only
    .build();
```

This way, when a user selects "Nike" as a brand filter, the brand facet still shows all brands with their counts (so the user can see what else is available), but the results are filtered to Nike only.

## Relevance Tuning

Default relevance scoring (TF-IDF / BM25) doesn't account for business signals. A product with 5,000 sales and 4.8 stars should rank higher than a product with 2 sales and no reviews, even if both match the query equally.

### Function Score Query

```java
@Service
public class RelevanceTuningService {

    public Query buildScoredQuery(String queryText) {
        return Query.of(q -> q.functionScore(fs -> fs
            .query(baseQuery(queryText))
            .functions(List.of(
                // Boost popular products
                FunctionScore.of(f -> f
                    .fieldValueFactor(fvf -> fvf
                        .field("salesCount")
                        .modifier(FieldValueFactorModifier.Log1p)
                        .factor(0.5)
                    )
                    .weight(2.0)
                ),
                // Boost highly rated products
                FunctionScore.of(f -> f
                    .fieldValueFactor(fvf -> fvf
                        .field("averageRating")
                        .modifier(FieldValueFactorModifier.None)
                        .factor(1.0)
                        .missing(3.0) // Default for unrated products
                    )
                    .weight(1.5)
                ),
                // Boost in-stock products heavily
                FunctionScore.of(f -> f
                    .filter(fq -> fq.term(t -> t
                        .field("inStock")
                        .value(true)))
                    .weight(5.0)
                ),
                // Recency boost — newer products get slight preference
                FunctionScore.of(f -> f
                    .exp(e -> e
                        .field("createdAt")
                        .placement(dp -> dp
                            .origin(JsonData.of(Instant.now().toEpochMilli()))
                            .scale(JsonData.of("30d"))
                            .decay(0.5)
                        )
                    )
                    .weight(0.8)
                )
            ))
            .scoreMode(FunctionScoreMode.Sum)
            .boostMode(FunctionBoostMode.Multiply)
        ));
    }

    private Query baseQuery(String queryText) {
        return Query.of(q -> q.multiMatch(mm -> mm
            .query(queryText)
            .fields(List.of("name^3", "name.exact^5", "brand^2", "description", "tags"))
            .type(TextQueryType.BEST_FIELDS)
            .fuzziness("AUTO")
            .prefixLength(2)
        ));
    }
}
```

### Boosting Rules Explained

**salesCount (log1p, weight 2.0)** — Products with 10,000 sales get a moderate boost over products with 10 sales, but logarithmic scaling prevents mega-sellers from dominating everything.

**averageRating (weight 1.5)** — 4.8-star products rank above 3-star products, all else being equal.

**inStock (weight 5.0)** — Never show out-of-stock products above in-stock products. This is the strongest signal.

**createdAt (decay, weight 0.8)** — Products older than 30 days get diminishing boost. Prevents stale catalog items from dominating.

### A/B Testing Relevance

We A/B tested our scoring function by routing 10% of traffic to different scoring configurations:

```java
@Service
public class SearchABTestService {

    public Query getQueryForUser(String queryText, String userId) {
        String variant = getVariant(userId); // Consistent hashing

        return switch (variant) {
            case "control" -> buildDefaultQuery(queryText);
            case "popularity_boost" -> buildPopularityBoostedQuery(queryText);
            case "recency_boost" -> buildRecencyBoostedQuery(queryText);
            default -> buildDefaultQuery(queryText);
        };
    }

    private String getVariant(String userId) {
        int hash = Math.abs(userId.hashCode() % 100);
        if (hash < 80) return "control";
        if (hash < 90) return "popularity_boost";
        return "recency_boost";
    }
}
```

We measured click-through rate and conversion rate per variant. The popularity boost increased conversion by 12%.

## Pagination Strategy

### The Problem with Deep Pagination

Elasticsearch's default `from + size` pagination has a hard limit (default 10,000). Deep pages are expensive because ES must fetch and score all preceding documents.

```java
// PROBLEM: This gets slower with each page
// Page 100 requires scoring 10,000 documents, returning only the last 100
var query = NativeQuery.builder()
    .withQuery(mainQuery)
    .withPageable(PageRequest.of(99, 100)) // from=9900, slow!
    .build();
```

### Solution: Search After (Cursor-Based Pagination)

```java
@Service
public class PaginatedSearchService {

    private final ElasticsearchOperations elasticsearchOperations;

    public CursorSearchResult search(String query, String[] searchAfter, int pageSize) {
        var nativeQuery = NativeQuery.builder()
            .withQuery(buildMainQuery(query))
            .withSort(Sort.by(Sort.Direction.DESC, "_score"))
            .withSort(Sort.by(Sort.Direction.ASC, "id")) // Tiebreaker
            .withMaxResults(pageSize)
            .build();

        // Resume from cursor position
        if (searchAfter != null && searchAfter.length > 0) {
            nativeQuery.setSearchAfter(List.of(searchAfter));
        }

        SearchHits<ProductDocument> hits = elasticsearchOperations.search(
            nativeQuery, ProductDocument.class);

        List<ProductDocument> products = hits.getSearchHits().stream()
            .map(SearchHit::getContent)
            .toList();

        // Extract cursor for next page
        String[] nextCursor = null;
        if (!hits.getSearchHits().isEmpty()) {
            SearchHit<ProductDocument> lastHit = hits.getSearchHits()
                .get(hits.getSearchHits().size() - 1);
            List<Object> sortValues = lastHit.getSortValues();
            nextCursor = sortValues.stream()
                .map(Object::toString)
                .toArray(String[]::new);
        }

        return new CursorSearchResult(products, nextCursor, hits.getTotalHits());
    }
}
```

### When to Use Which

**from/size (offset pagination)** — Use for the first 100 pages (most users never go beyond page 5). Familiar UX with page numbers.

**search_after (cursor pagination)** — Use for infinite scroll UIs and deep pagination. Consistent performance regardless of depth.

**scroll API** — Use for bulk data export only. Not suitable for real-time search UIs (holds context server-side).

## Zero-Downtime Reindexing

When you change your mapping (new analyzer, new fields, restructured data), you need to rebuild the entire index. In production, you can't just delete it and start over.

### The Blue-Green Index Strategy

```java
@Service
public class ReindexService {

    private final ElasticsearchClient esClient;
    private final ProductRepository productRepository;
    private final ElasticsearchOperations operations;

    private static final String INDEX_ALIAS = "products";

    public void reindexZeroDowntime() {
        // Step 1: Create new index with updated mapping
        String newIndexName = INDEX_ALIAS + "_" + System.currentTimeMillis();
        createIndexWithNewMapping(newIndexName);

        // Step 2: Bulk index all documents into new index
        reindexAllProducts(newIndexName);

        // Step 3: Atomic alias swap
        swapAlias(INDEX_ALIAS, newIndexName);

        // Step 4: Delete old index (after verification)
        String oldIndex = getOldIndexName();
        if (oldIndex != null && verifyNewIndex(newIndexName)) {
            esClient.indices().delete(d -> d.index(oldIndex));
        }
    }

    private void reindexAllProducts(String targetIndex) {
        int batchSize = 1000;
        int page = 0;
        long totalIndexed = 0;

        while (true) {
            Page<Product> products = productRepository.findAll(
                PageRequest.of(page, batchSize));

            if (products.isEmpty()) break;

            List<IndexQuery> queries = products.getContent().stream()
                .map(this::toIndexQuery)
                .toList();

            operations.bulkIndex(queries, IndexCoordinates.of(targetIndex));
            totalIndexed += queries.size();

            log.info("Reindexed {} / {} products",
                totalIndexed, products.getTotalElements());
            page++;
        }

        // Wait for index to be ready
        esClient.indices().refresh(r -> r.index(targetIndex));
    }

    private void swapAlias(String alias, String newIndex) {
        String currentIndex = getCurrentIndexForAlias(alias);

        // Atomic operation: remove alias from old, add to new
        esClient.indices().updateAliases(ua -> ua
            .actions(List.of(
                Action.of(a -> a.remove(r -> r.index(currentIndex).alias(alias))),
                Action.of(a -> a.add(ad -> ad.index(newIndex).alias(alias)))
            ))
        );

        log.info("Alias '{}' swapped from {} to {}", alias, currentIndex, newIndex);
    }

    private void createIndexWithNewMapping(String indexName) {
        esClient.indices().create(c -> c
            .index(indexName)
            .settings(s -> s
                .numberOfShards("3")
                .numberOfReplicas("0") // No replicas during bulk indexing (faster)
                .refreshInterval(t -> t.time("-1")) // Disable refresh during bulk
            )
            .mappings(m -> m
                // ... your updated mapping
            )
        );
    }

    private boolean verifyNewIndex(String newIndex) {
        long oldCount = esClient.count(c -> c.index(INDEX_ALIAS)).count();
        long newCount = esClient.count(c -> c.index(newIndex)).count();

        // Allow 1% variance (new products added during reindex)
        double variance = Math.abs(oldCount - newCount) / (double) oldCount;
        return variance < 0.01;
    }
}
```

### Performance Tips for Bulk Indexing

```java
private void optimizeForBulkIndexing(String indexName) {
    // Disable replicas and refresh during bulk load
    esClient.indices().putSettings(ps -> ps
        .index(indexName)
        .settings(s -> s
            .numberOfReplicas("0")
            .refreshInterval(t -> t.time("-1"))
        )
    );
}

private void restoreNormalSettings(String indexName) {
    // Re-enable after bulk load completes
    esClient.indices().putSettings(ps -> ps
        .index(indexName)
        .settings(s -> s
            .numberOfReplicas("2")
            .refreshInterval(t -> t.time("1s"))
        )
    );

    // Force merge for optimal search performance
    esClient.indices().forcemerge(fm -> fm
        .index(indexName)
        .maxNumSegments(5)
    );
}
```

**Bulk indexing speed comparison:**

**With replicas + refresh** — ~3,000 docs/sec

**Without replicas, refresh disabled** — ~18,000 docs/sec

**6x faster** — For our 2M products, this means 2 minutes vs 11 minutes for a full reindex.

## The Search Controller: Putting It All Together

```java
@RestController
@RequestMapping("/api/search")
public class SearchController {

    private final FacetedSearchService searchService;
    private final AutocompleteService autocompleteService;
    private final RelevanceTuningService relevanceService;

    @GetMapping
    public ResponseEntity<SearchResult> search(
            @RequestParam String q,
            @RequestParam(required = false) Map<String, List<String>> filters,
            @RequestParam(defaultValue = "0") int page,
            @RequestParam(defaultValue = "20") int size,
            @RequestParam(defaultValue = "relevance") String sort) {

        if (q.isBlank()) {
            return ResponseEntity.badRequest().build();
        }

        SearchRequest request = SearchRequest.builder()
            .query(q.trim())
            .filters(filters != null ? filters : Map.of())
            .page(page)
            .size(Math.min(size, 100)) // Cap at 100
            .sort(sort)
            .build();

        SearchResult result = searchService.searchWithFacets(request);
        return ResponseEntity.ok(result);
    }

    @GetMapping("/autocomplete")
    public ResponseEntity<List<AutocompleteSuggestion>> autocomplete(
            @RequestParam String prefix,
            @RequestParam(defaultValue = "8") int limit) {

        if (prefix.length() < 2) {
            return ResponseEntity.ok(List.of());
        }

        return ResponseEntity.ok(
            autocompleteService.suggest(prefix.trim(), Math.min(limit, 15)));
    }
}
```

## Production Monitoring

Monitor these metrics to catch search quality issues before users complain:

```java
@Component
public class SearchMetricsCollector {

    private final MeterRegistry meterRegistry;

    public void recordSearch(String query, long hits, long tookMs, boolean hasResults) {
        meterRegistry.timer("search.latency").record(tookMs, TimeUnit.MILLISECONDS);
        meterRegistry.counter("search.total").increment();

        if (!hasResults) {
            meterRegistry.counter("search.zero_results").increment();
            log.info("Zero results for query: '{}'", query);
            // Feed this into your synonym/analyzer improvement pipeline
        }

        meterRegistry.summary("search.hit_count").record(hits);
    }
}
```

**Key metrics to track:**

**Zero result rate** — Should be under 5%. High rate means your analyzers or synonyms need work.

**P99 search latency** — Should be under 200ms. If higher, check your query complexity and shard count.

**Click-through rate by position** — If position 1 has low CTR, your relevance scoring needs tuning.

**Autocomplete acceptance rate** — How often users select a suggestion vs. typing their full query.

## Lessons from Production

- **Synonyms are never done.** "Laptop", "notebook", "MacBook" — you'll keep adding synonyms for months. Build a pipeline where product managers can add them without deployments.

- **Edge n-grams inflate index size.** Our index grew 3x after adding autocomplete fields. Budget for this.

- **Search-after pagination UX is tricky.** Users can't jump to "page 47." Design your UI for infinite scroll or sequential pagination.

- **Shard count matters more than you think.** Too few shards (1) limits parallelism. Too many (50) adds overhead. Rule of thumb: keep shards between 10-50GB each.

- **Test with real queries.** We collected the top 1,000 actual user queries and built regression tests. After every analyzer change, we verified the top 100 queries still returned sensible results.
