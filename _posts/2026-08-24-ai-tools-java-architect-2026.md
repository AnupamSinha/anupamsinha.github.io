---
title: "The AI Tools I Use Daily as a Java Architect (2026 Edition)"
date: 2026-08-24
categories: [Spring Boot, AI]
tags: [ai, java, software-architecture, developer-tools, productivity]
description: "A practical walkthrough of the AI tools integrated into my daily workflow — from code generation to architecture review, documentation, testing, and production monitoring — with honest assessments of what actually delivers value."
mermaid: true
---
## My Context

I've been a Java Technical Architect in Singapore for 17 years. My current stack is Java 21, Spring Boot 3.x, Kubernetes, Kafka, PostgreSQL — enterprise fintech. I manage architecture for 12 microservices and guide a team of engineers.

This isn't a "top 10 AI tools" listicle. This is what I actually use every day, why I use it, and what I've stopped using after the novelty wore off. Every tool here has survived at least 3 months in my workflow.

---

## Category 1: Code Generation and Editing

### Cursor (Primary IDE)

**What I use it for** — Day-to-day coding, refactoring, writing implementation code from specs

**Why it stuck** — The codebase-aware context (@ mentions for files, docs, and symbols) means it actually understands my project structure. When I say "implement this interface following the pattern in PaymentService," it looks at PaymentService and follows the pattern.

**My configuration:**
- Model: Claude Sonnet for complex architecture work, GPT-4o for routine coding
- Rules file with our team's conventions, naming standards, and architectural principles
- Docs indexed: Spring AI docs, our internal ADRs, team coding standards

**Daily usage** — 4-5 hours

**What it's bad at** — Multi-file refactoring that requires understanding distributed system interactions. It'll refactor one service perfectly but miss that three other services depend on the contract it just changed.

### GitHub Copilot (In CI/CD Scripts)

**What I use it for** — GitHub Actions workflows, Dockerfile optimization, shell scripts

**Why it stuck** — For infrastructure-as-code and CI/CD, the inline completion model works better than chat. I'm often writing YAML where the next line is predictable from context.

**Daily usage** — 1 hour (in VS Code for infra work)

---

## Category 2: Architecture Review and Design

### Custom RAG Architecture Advisor

**What I use it for** — Reviewing design documents, validating architecture decisions against our principles, catching anti-patterns in proposals

**The setup** — I built a custom RAG-based assistant (Spring AI + PGVector) fed with:
- Our 15 Architecture Decision Records
- Cloud architecture best practices
- Our non-functional requirements document
- Past incident post-mortems

**How I use it daily:**

I paste a design proposal and ask it to review against our architecture principles. It catches things like:

- Synchronous calls in critical paths (we mandate async for inter-service communication)
- Missing circuit breakers on external dependencies
- Database coupling between services
- Violation of our data ownership boundaries

**Example output:**

```
CONCERN: The proposal adds a synchronous HTTP call from order-service 
to inventory-service during checkout flow.

VIOLATION: ADR-003 states "All inter-service communication in the 
critical purchase path must be asynchronous."

RECOMMENDATION: Use the reservation pattern — publish an 
InventoryReservationRequested event to Kafka and implement a 
saga for the checkout flow.

REFERENCE: See payment-service implementation of the same pattern 
in ADR-007.
```

**Daily usage** — 30-45 minutes (during design reviews and PR assessments)

**Honest assessment** — It catches about 70% of what I'd catch manually. The 30% it misses are subtle issues that require understanding team dynamics, operational maturity, and organizational constraints. But it saves me from repeatedly explaining the same architectural principles.

### Claude for System Design Thinking

**What I use it for** — Brainstorming system design tradeoffs, exploring architectural alternatives, rubber-duck debugging design decisions

**Why it works** — When I'm designing a new system, I use Claude as a sparring partner. I describe constraints, it suggests approaches, I poke holes, it refines. The conversation format works well for iterative design.

**Example interaction:**

```
Me: "I need to implement idempotency for our payment processing. 
We process 50K transactions/day. Considering: idempotency key in 
Redis with TTL vs. database unique constraint on request_id. 
What are the tradeoffs I'm missing?"
```

It'll bring up things like:
- Redis eviction under memory pressure losing idempotency guarantees
- Clock skew issues with TTL-based approaches
- Network partition scenarios where both Redis and DB are needed
- The need for idempotency at the payment gateway level, not just application level

**Daily usage** — 1-2 hours for design work

---

## Category 3: Testing

### AI Test Generation (Integrated in Cursor)

**What I use it for** — Generating unit test scaffolding, edge case identification, test data generation

**My workflow:**

1. Write the implementation
2. Ask AI to generate comprehensive tests
3. Review and fix the 10-15% that's wrong
4. Add edge cases the AI missed (usually concurrency and state-dependent scenarios)

**What I've learned to prompt for:**

```
Generate tests for this service method. Include:
- Happy path
- All validation failures
- External service failures (timeout, 5xx, network error)
- Concurrent access scenarios
- Null/empty inputs
- Boundary values

Use our patterns: @ExtendWith(MockitoExtension.class), AssertJ assertions, 
@Nested classes for grouping, @DisplayName for readability.
```

**Time saved** — Approximately 40% reduction in test writing time

**What it consistently gets wrong:**
- Stateful integration tests
- Tests that require understanding temporal ordering
- Mocking configurations for our custom abstractions
- Tests for race conditions (it writes them, but they don't actually test concurrency)

### Diffblue Cover (Enterprise License)

**What I use it for** — Bulk unit test generation for legacy code that has zero coverage

**Why it's different from AI chat** — Diffblue actually runs the code, observes behavior, and generates tests that pass. It's not guessing from patterns — it's recording actual behavior.

**When I use it** — When we acquire legacy services or need to add test coverage before refactoring. Not for new code.

**Honest assessment** — Generates tests that verify current behavior (good for regression). But those tests encode existing bugs too. You still need human judgment about whether the current behavior is correct.

---

## Category 4: Documentation

### AI Documentation Generation (Post-Commit Hook)

**What I use it for** — Keeping API docs, ADRs, and runbooks synchronized with code

**The setup:**

```java
// Custom Spring Boot actuator endpoint that generates docs
@RestController
@RequestMapping("/internal/docs")
public class DocumentationController {

    private final OpenApiGenerator openApiGenerator;
    private final ChangelogGenerator changelogGenerator;

    @PostMapping("/generate")
    public DocumentationResult generateDocs(
            @RequestParam String fromCommit,
            @RequestParam String toCommit) {
        // Analyze git diff
        List<ChangedFile> changes = gitService.getChanges(fromCommit, toCommit);

        // Generate API doc updates
        if (changes.stream().anyMatch(f -> f.isController())) {
            openApiGenerator.regenerate();
        }

        // Generate changelog entry
        String changelog = changelogGenerator.fromCommits(fromCommit, toCommit);

        return new DocumentationResult(changelog, changes.size());
    }
}
```

**Integrated with CI:**

```yaml
# .github/workflows/docs.yml
- name: Generate Documentation
  run: |
    curl -X POST "http://localhost:8080/internal/docs/generate?fromCommit=${{ github.event.before }}&toCommit=${{ github.sha }}"
```

**Impact** — Documentation staleness dropped from "months behind" to "days behind." New team members cite this as the single biggest improvement in onboarding experience.

### Notion AI (For Non-Technical Docs)

**What I use it for** — Drafting technical proposals, meeting summaries, stakeholder communications

**Daily usage** — 20-30 minutes

**Why it stuck** — It's already in Notion where our docs live. The context-awareness (it can read the page and surrounding pages) means it writes in our team's voice and references our terminology.

---

## Category 5: Code Review

### AI-Assisted PR Review (Custom GitHub Action)

**What I use it for** — First-pass review of every PR before human review

**The setup** — A GitHub Action that runs on every PR:

```yaml
name: AI Code Review
on:
  pull_request:
    types: [opened, synchronize]

jobs:
  ai-review:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
        with:
          fetch-depth: 0

      - name: Run AI Review
        env:
          OPENAI_API_KEY: ${{ secrets.OPENAI_API_KEY }}
        run: |
          git diff origin/main...HEAD > /tmp/diff.patch
          python scripts/ai-review.py /tmp/diff.patch
```

**What it checks:**
- Compliance with our coding standards
- Security anti-patterns (SQL injection, hardcoded secrets, missing input validation)
- Performance concerns (N+1 queries, missing indexes, unbounded collections)
- Missing error handling
- Test coverage gaps

**False positive rate** — About 15%. Acceptable because it's non-blocking (comments, doesn't reject PRs).

**Most valuable catches:**
- @Transactional methods calling external services (3 times this quarter)
- Missing null checks on Optional.get() (weekly)
- Unbounded pagination in list endpoints (monthly)

---

## Category 6: Production Monitoring and Debugging

### AI-Powered Log Analysis

**What I use it for** — Correlating errors across services, identifying root causes from log patterns

**The approach** — Our logs flow to OpenSearch. I have a script that pulls error patterns and feeds them to an AI for correlation:

```java
@Service
public class LogAnalysisService {

    private final ChatClient chatClient;
    private final OpenSearchClient logClient;

    public IncidentAnalysis analyzeIncident(String timeWindow, String affectedService) {
        // Pull error logs from the time window
        List<LogEntry> errors = logClient.searchErrors(timeWindow, affectedService);

        // Pull related service logs
        List<String> relatedServices = serviceRegistry.getDependencies(affectedService);
        Map<String, List<LogEntry>> relatedLogs = relatedServices.stream()
            .collect(Collectors.toMap(
                Function.identity(),
                svc -> logClient.searchErrors(timeWindow, svc)
            ));

        // Feed to AI for correlation
        String analysis = chatClient.prompt()
            .system(INCIDENT_ANALYSIS_PROMPT)
            .user(formatLogContext(errors, relatedLogs))
            .call()
            .content();

        return parseAnalysis(analysis);
    }
}
```

**Real example:** Last month, our payment service started returning 503s. The AI correlated logs across payment-service, redis-cluster, and kubernetes events to identify that a Redis node failover triggered connection pool exhaustion. Manual correlation would have taken 30+ minutes. AI did it in 45 seconds.

**Daily usage** — On-demand during incidents (2-3 times per week)

---

## Category 7: Learning and Staying Current

### AI for Technology Evaluation

**What I use it for** — Rapidly evaluating new frameworks, libraries, and patterns against our constraints

**My process:**

1. Feed the AI our technical context (stack, team size, operational maturity)
2. Ask it to evaluate a new technology against our specific constraints
3. Get a structured analysis: benefits, risks, migration effort, operational burden

**Example:** When evaluating whether to adopt Spring Modulith:

```
Given:
- 12 microservices, 4 team members
- High operational overhead from distributed deployment
- Some services should never have been separated

Evaluate Spring Modulith for consolidating our notification, 
audit-log, and user-preferences services into a modular monolith.
```

This gives me a structured starting point for a proper evaluation. It's not the final decision, but it surfaces considerations I might miss.

---

## Tools I Stopped Using (And Why)

### Amazon CodeWhisperer

**Why I dropped it** — Completions were noticeably less accurate for Spring Boot patterns compared to Copilot and Cursor. It excels in AWS SDK code but falls behind for general Java enterprise work.

### ChatGPT for Code

**Why I dropped it** — No codebase context. Every conversation starts from zero. Cursor and Claude with project context are strictly better for code-related work.

### Tabnine

**Why I dropped it** — The "trains on your codebase" promise sounded great. In practice, the local model wasn't smart enough to surface useful patterns beyond simple completions. Copilot does better with just public training data.

---

## My Daily Workflow Timeline

**8:00 AM** — Review AI-generated PR comments from overnight work (15 min)

**8:30 AM** — Architecture review using RAG advisor for new proposals (30 min)

**9:00 AM** — Coding in Cursor with AI pair programming (3 hours)

**12:00 PM** — Use Claude for design thinking on complex problems (1 hour)

**1:00 PM** — Generate tests for morning's code, review and fix (1 hour)

**2:00 PM** — Documentation sync check, update ADRs (30 min)

**3:00 PM** — Code reviews with AI pre-analysis (1 hour)

**4:00 PM** — Production monitoring, AI log analysis if needed (30 min)

---

## Cost Breakdown (Monthly)

**Cursor Pro** — $20/month

**GitHub Copilot Business** — $19/month

**Claude Pro** — $20/month

**OpenAI API (for custom RAG + review agents)** — $150-200/month

**Diffblue Cover** — Company license (approximately $50/seat/month)

**Total** — approximately $300/month personal, $50/month company tools

**ROI** — I estimate these tools save me 15-20 hours per week. At architect billing rates in Singapore, that's a return of roughly 50x on the tool investment.

---

## Advice for Getting Started

**1. Start with code review automation.** It's the safest entry point — AI can only suggest, not break. The feedback loop is immediate.

**2. Invest in context.** Every AI tool performs better with project-specific context. Write down your standards, architectural principles, and conventions. Feed them to your tools.

**3. Don't use AI for decisions.** Use it for information gathering, option generation, and analysis. The decision — the tradeoff between competing concerns — is still yours.

**4. Track your false positive rate.** If an AI tool has more than 20% false positives, it's creating noise, not value. Tune or drop it.

**5. Build custom where generic fails.** The RAG architecture advisor took me two days to build and provides more value than any off-the-shelf tool because it knows MY system.

---

## Looking Ahead

The biggest shift I see coming in 2026-2027 is AI agents that can operate across multiple services simultaneously — understanding the distributed system as a whole, not just individual files. When that arrives, the role of architect becomes even more about defining the boundaries and principles that guide those agents.

Until then, the winning strategy is: leverage AI for the routine, reserve your attention for the architectural and business decisions that define system success.

---

*If you found this useful, give it up to **50 claps** and follow [@hianupamsinha](https://medium.com/@hianupamsinha) for more practical engineering insights.*
