---
title: "Building a Distributed Job Scheduler Like Airflow — in Java"
date: 2026-08-24
categories: [Spring Boot, Architecture]
tags: [java, distributed-systems, spring-boot, scheduling, system-design]
description: "Design and implement a production-ready distributed task scheduler with cron triggers, DAG dependencies, retry policies, and distributed execution using Spring Boot, ShedLock, and Kafka."
mermaid: true
---
## Why Build One When Airflow Exists?

Fair question. If you're a Python shop and your workflow is data pipelines, use Airflow. But if you're running a Java/Spring Boot ecosystem with dozens of microservices, introducing a Python-based scheduler with its own deployment, dependency management, and operational model creates friction.

After 17 years building enterprise Java systems in Singapore, I've repeatedly needed a scheduler that:

- Runs in the same JVM ecosystem as our services
- Deploys as a Spring Boot app (same CI/CD, same monitoring, same team)
- Supports distributed execution across multiple pods
- Handles DAG-based task dependencies
- Has retry with backoff, dead-letter handling, and alerting
- Integrates natively with our Kafka infrastructure

So I built one. Here's how.

## Architecture Overview

The system has five core components:

**Scheduler Core** — Evaluates cron expressions, determines which tasks are due, and triggers execution

**DAG Engine** — Manages task dependencies and execution ordering

**Distributed Lock (ShedLock)** — Ensures only one pod executes each scheduled task

**Task Executor** — Runs tasks locally or publishes to Kafka for distributed execution

**Status Tracker** — Records execution state, supports monitoring and alerting

## The Data Model

```java
@Entity
@Table(name = "scheduled_jobs")
public class ScheduledJob {

    @Id
    @GeneratedValue(strategy = GenerationType.UUID)
    private String id;

    private String name;
    private String cronExpression;
    private String dagId;             // Which DAG this belongs to
    private int executionOrder;       // Order within the DAG
    private String taskHandlerClass;  // Fully qualified class name

    @Enumerated(EnumType.STRING)
    private JobStatus status;

    private int maxRetries;
    private int currentRetryCount;
    private long retryBackoffMs;

    private LocalDateTime lastExecutionTime;
    private LocalDateTime nextExecutionTime;
    private LocalDateTime createdAt;
    private LocalDateTime updatedAt;

    @ElementCollection
    @CollectionTable(name = "job_dependencies")
    private Set<String> dependsOn;    // IDs of jobs this depends on
}
```

```java
@Entity
@Table(name = "job_executions")
public class JobExecution {

    @Id
    @GeneratedValue(strategy = GenerationType.UUID)
    private String id;

    private String jobId;
    private String dagRunId;         // Groups executions of the same DAG run

    @Enumerated(EnumType.STRING)
    private ExecutionStatus status;  // PENDING, RUNNING, SUCCESS, FAILED, DEAD_LETTER

    private LocalDateTime startedAt;
    private LocalDateTime completedAt;
    private String errorMessage;
    private String executedByNode;   // Which pod executed this
    private int attemptNumber;
}
```

```java
public enum ExecutionStatus {
    PENDING, QUEUED, RUNNING, SUCCESS, FAILED, RETRYING, DEAD_LETTER, SKIPPED
}
```

## The Task Handler Interface

Every schedulable task implements this interface:

```java
public interface TaskHandler {

    /**
     * Execute the task. Return TaskResult indicating success or failure.
     */
    TaskResult execute(TaskContext context);

    /**
     * Called when all retries are exhausted. Override for custom dead-letter handling.
     */
    default void onDeadLetter(TaskContext context, Exception lastException) {
        // Default: log and alert
    }
}
```

```java
@Data
@Builder
public class TaskContext {
    private String jobId;
    private String dagRunId;
    private int attemptNumber;
    private Map<String, Object> parameters;
    private Map<String, Object> upstreamResults;  // Results from dependency jobs
}
```

```java
@Data
@Builder
public class TaskResult {
    private boolean success;
    private String message;
    private Map<String, Object> output;  // Passed to downstream tasks

    public static TaskResult success() {
        return TaskResult.builder().success(true).build();
    }

    public static TaskResult success(Map<String, Object> output) {
        return TaskResult.builder().success(true).output(output).build();
    }

    public static TaskResult failure(String message) {
        return TaskResult.builder().success(false).message(message).build();
    }
}
```

## Cron-Based Trigger Engine

The scheduler core runs on a fixed interval and checks which jobs are due:

```java
@Service
public class SchedulerCore {

    private final ScheduledJobRepository jobRepository;
    private final DagEngine dagEngine;
    private final TaskDispatcher taskDispatcher;

    @Scheduled(fixedDelay = 10000) // Check every 10 seconds
    @SchedulerLock(name = "scheduler-core-tick", lockAtMostFor = "9s", lockAtLeastFor = "5s")
    public void tick() {
        LocalDateTime now = LocalDateTime.now();

        List<ScheduledJob> dueJobs = jobRepository.findByNextExecutionTimeBefore(now);

        // Group by DAG
        Map<String, List<ScheduledJob>> byDag = dueJobs.stream()
                .filter(job -> job.getDagId() != null)
                .collect(Collectors.groupingBy(ScheduledJob::getDagId));

        // Trigger standalone jobs immediately
        dueJobs.stream()
                .filter(job -> job.getDagId() == null)
                .forEach(taskDispatcher::dispatch);

        // Trigger DAG-based jobs through the DAG engine
        byDag.forEach((dagId, jobs) -> dagEngine.triggerDagRun(dagId));

        // Update next execution times
        dueJobs.forEach(this::updateNextExecution);
    }

    private void updateNextExecution(ScheduledJob job) {
        CronExpression cron = CronExpression.parse(job.getCronExpression());
        LocalDateTime next = cron.next(LocalDateTime.now());
        job.setNextExecutionTime(next);
        job.setLastExecutionTime(LocalDateTime.now());
        jobRepository.save(job);
    }
}
```

## The DAG Engine

This is the core differentiator from simple schedulers. It handles dependency resolution:

```java
@Service
public class DagEngine {

    private final ScheduledJobRepository jobRepository;
    private final JobExecutionRepository executionRepository;
    private final TaskDispatcher taskDispatcher;

    @Transactional
    public String triggerDagRun(String dagId) {
        String dagRunId = UUID.randomUUID().toString();

        List<ScheduledJob> dagJobs = jobRepository.findByDagIdOrderByExecutionOrder(dagId);

        // Create execution records for all jobs in the DAG
        dagJobs.forEach(job -> {
            JobExecution execution = JobExecution.builder()
                    .jobId(job.getId())
                    .dagRunId(dagRunId)
                    .status(ExecutionStatus.PENDING)
                    .attemptNumber(1)
                    .build();
            executionRepository.save(execution);
        });

        // Dispatch root tasks (no dependencies)
        dagJobs.stream()
                .filter(job -> job.getDependsOn() == null || job.getDependsOn().isEmpty())
                .forEach(job -> taskDispatcher.dispatch(job, dagRunId));

        return dagRunId;
    }

    /**
     * Called when a task completes. Checks if downstream tasks can now execute.
     */
    @Transactional
    public void onTaskCompleted(String jobId, String dagRunId, TaskResult result) {
        // Mark current job as completed
        JobExecution execution = executionRepository
                .findByJobIdAndDagRunId(jobId, dagRunId);
        execution.setStatus(result.isSuccess() ? ExecutionStatus.SUCCESS : ExecutionStatus.FAILED);
        execution.setCompletedAt(LocalDateTime.now());
        executionRepository.save(execution);

        if (!result.isSuccess()) {
            handleFailedTask(jobId, dagRunId, execution);
            return;
        }

        // Find downstream tasks that depend on this job
        List<ScheduledJob> downstream = jobRepository.findByDependsOnContaining(jobId);

        for (ScheduledJob downstreamJob : downstream) {
            if (allDependenciesCompleted(downstreamJob, dagRunId)) {
                Map<String, Object> upstreamResults = gatherUpstreamResults(downstreamJob, dagRunId);
                taskDispatcher.dispatchWithContext(downstreamJob, dagRunId, upstreamResults);
            }
        }
    }

    private boolean allDependenciesCompleted(ScheduledJob job, String dagRunId) {
        return job.getDependsOn().stream()
                .allMatch(depId -> {
                    JobExecution dep = executionRepository.findByJobIdAndDagRunId(depId, dagRunId);
                    return dep != null && dep.getStatus() == ExecutionStatus.SUCCESS;
                });
    }

    private Map<String, Object> gatherUpstreamResults(ScheduledJob job, String dagRunId) {
        Map<String, Object> results = new HashMap<>();
        for (String depId : job.getDependsOn()) {
            JobExecution dep = executionRepository.findByJobIdAndDagRunId(depId, dagRunId);
            if (dep.getOutput() != null) {
                results.put(depId, dep.getOutput());
            }
        }
        return results;
    }

    private void handleFailedTask(String jobId, String dagRunId, JobExecution execution) {
        // Mark all downstream tasks as SKIPPED
        List<ScheduledJob> allDownstream = findAllTransitiveDownstream(jobId);
        allDownstream.forEach(job -> {
            JobExecution downstream = executionRepository.findByJobIdAndDagRunId(job.getId(), dagRunId);
            if (downstream != null && downstream.getStatus() == ExecutionStatus.PENDING) {
                downstream.setStatus(ExecutionStatus.SKIPPED);
                executionRepository.save(downstream);
            }
        });
    }

    private List<ScheduledJob> findAllTransitiveDownstream(String jobId) {
        // BFS to find all transitive dependencies
        List<ScheduledJob> result = new ArrayList<>();
        Queue<String> queue = new LinkedList<>();
        queue.add(jobId);
        Set<String> visited = new HashSet<>();

        while (!queue.isEmpty()) {
            String current = queue.poll();
            if (visited.contains(current)) continue;
            visited.add(current);

            List<ScheduledJob> downstream = jobRepository.findByDependsOnContaining(current);
            result.addAll(downstream);
            downstream.forEach(job -> queue.add(job.getId()));
        }
        return result;
    }
}
```

## Distributed Execution with ShedLock + Kafka

### ShedLock for Leader Election

ShedLock ensures only one pod in your cluster executes the scheduler tick:

```xml
<dependency>
    <groupId>net.javacrumbs.shedlock</groupId>
    <artifactId>shedlock-spring</artifactId>
    <version>5.10.0</version>
</dependency>
<dependency>
    <groupId>net.javacrumbs.shedlock</groupId>
    <artifactId>shedlock-provider-jdbc-template</artifactId>
    <version>5.10.0</version>
</dependency>
```

```java
@Configuration
@EnableSchedulerLock(defaultLockAtMostFor = "30s")
public class ShedLockConfig {

    @Bean
    public LockProvider lockProvider(DataSource dataSource) {
        return new JdbcTemplateLockProvider(
                JdbcTemplateLockProvider.Configuration.builder()
                        .withJdbcTemplate(new JdbcTemplate(dataSource))
                        .usingDbTime()
                        .build()
        );
    }
}
```

Required table:

```sql
CREATE TABLE shedlock (
    name       VARCHAR(64)  NOT NULL PRIMARY KEY,
    lock_until TIMESTAMP    NOT NULL,
    locked_at  TIMESTAMP    NOT NULL,
    locked_by  VARCHAR(255) NOT NULL
);
```

### Kafka for Distributed Task Execution

While the scheduler tick runs on one pod, actual task execution is distributed via Kafka:

```java
@Service
public class TaskDispatcher {

    private final KafkaTemplate<String, TaskMessage> kafkaTemplate;
    private final JobExecutionRepository executionRepository;

    public void dispatch(ScheduledJob job, String dagRunId) {
        TaskMessage message = TaskMessage.builder()
                .jobId(job.getId())
                .dagRunId(dagRunId)
                .taskHandlerClass(job.getTaskHandlerClass())
                .attemptNumber(1)
                .parameters(job.getParameters())
                .build();

        // Use jobId as key for partition affinity
        kafkaTemplate.send("task-execution", job.getId(), message);

        // Update status
        JobExecution execution = executionRepository.findByJobIdAndDagRunId(job.getId(), dagRunId);
        execution.setStatus(ExecutionStatus.QUEUED);
        executionRepository.save(execution);
    }

    public void dispatchWithContext(ScheduledJob job, String dagRunId, Map<String, Object> upstreamResults) {
        TaskMessage message = TaskMessage.builder()
                .jobId(job.getId())
                .dagRunId(dagRunId)
                .taskHandlerClass(job.getTaskHandlerClass())
                .attemptNumber(1)
                .parameters(job.getParameters())
                .upstreamResults(upstreamResults)
                .build();

        kafkaTemplate.send("task-execution", job.getId(), message);
    }
}
```

### The Task Worker (Consumer)

Every pod in your cluster runs this consumer, distributing work across pods:

```java
@Service
public class TaskWorker {

    private final ApplicationContext applicationContext;
    private final JobExecutionRepository executionRepository;
    private final RetryHandler retryHandler;
    private final DagEngine dagEngine;

    @KafkaListener(topics = "task-execution", groupId = "task-workers",
            concurrency = "3")
    public void executeTask(TaskMessage message) {
        String nodeId = System.getenv("HOSTNAME");

        JobExecution execution = executionRepository
                .findByJobIdAndDagRunId(message.getJobId(), message.getDagRunId());
        execution.setStatus(ExecutionStatus.RUNNING);
        execution.setStartedAt(LocalDateTime.now());
        execution.setExecutedByNode(nodeId);
        executionRepository.save(execution);

        try {
            TaskHandler handler = resolveHandler(message.getTaskHandlerClass());
            TaskContext context = TaskContext.builder()
                    .jobId(message.getJobId())
                    .dagRunId(message.getDagRunId())
                    .attemptNumber(message.getAttemptNumber())
                    .parameters(message.getParameters())
                    .upstreamResults(message.getUpstreamResults())
                    .build();

            TaskResult result = handler.execute(context);

            if (result.isSuccess()) {
                execution.setStatus(ExecutionStatus.SUCCESS);
                execution.setCompletedAt(LocalDateTime.now());
                executionRepository.save(execution);
                dagEngine.onTaskCompleted(message.getJobId(), message.getDagRunId(), result);
            } else {
                handleFailure(message, execution, new RuntimeException(result.getMessage()));
            }

        } catch (Exception e) {
            handleFailure(message, execution, e);
        }
    }

    private void handleFailure(TaskMessage message, JobExecution execution, Exception e) {
        execution.setErrorMessage(e.getMessage());
        execution.setStatus(ExecutionStatus.FAILED);
        execution.setCompletedAt(LocalDateTime.now());
        executionRepository.save(execution);

        retryHandler.handleRetry(message, e);
    }

    private TaskHandler resolveHandler(String className) {
        try {
            Class<?> clazz = Class.forName(className);
            return (TaskHandler) applicationContext.getBean(clazz);
        } catch (ClassNotFoundException e) {
            throw new IllegalStateException("Task handler not found: " + className, e);
        }
    }
}
```

## Retry with Exponential Backoff

```java
@Service
public class RetryHandler {

    private final ScheduledJobRepository jobRepository;
    private final KafkaTemplate<String, TaskMessage> kafkaTemplate;
    private final DeadLetterService deadLetterService;

    public void handleRetry(TaskMessage message, Exception exception) {
        ScheduledJob job = jobRepository.findById(message.getJobId())
                .orElseThrow();

        int currentAttempt = message.getAttemptNumber();

        if (currentAttempt >= job.getMaxRetries()) {
            // Exhausted all retries — send to dead letter
            deadLetterService.sendToDeadLetter(message, exception);
            return;
        }

        // Calculate backoff: baseDelay * 2^attempt (with jitter)
        long backoffMs = calculateBackoff(job.getRetryBackoffMs(), currentAttempt);

        // Schedule retry via delayed Kafka topic
        TaskMessage retryMessage = message.toBuilder()
                .attemptNumber(currentAttempt + 1)
                .scheduledRetryAt(Instant.now().plusMillis(backoffMs))
                .build();

        kafkaTemplate.send("task-execution-retry", message.getJobId(), retryMessage);
    }

    private long calculateBackoff(long baseMs, int attempt) {
        long backoff = baseMs * (long) Math.pow(2, attempt);
        long jitter = ThreadLocalRandom.current().nextLong(0, backoff / 4);
        return Math.min(backoff + jitter, 300_000); // Cap at 5 minutes
    }
}
```

### Delayed Retry Consumer

```java
@Service
public class RetryConsumer {

    private final KafkaTemplate<String, TaskMessage> kafkaTemplate;

    @KafkaListener(topics = "task-execution-retry", groupId = "retry-workers")
    public void processRetry(TaskMessage message) {
        Instant scheduledAt = message.getScheduledRetryAt();

        if (Instant.now().isBefore(scheduledAt)) {
            // Not time yet — re-queue with short delay
            // In production, use a proper delay queue (Redis sorted set or Kafka timestamp-based)
            try {
                Thread.sleep(Duration.between(Instant.now(), scheduledAt).toMillis());
            } catch (InterruptedException e) {
                Thread.currentThread().interrupt();
                return;
            }
        }

        // Dispatch to main execution topic
        kafkaTemplate.send("task-execution", message.getJobId(), message);
    }
}
```

## Dead Letter Handling

```java
@Service
public class DeadLetterService {

    private final JobExecutionRepository executionRepository;
    private final KafkaTemplate<String, TaskMessage> kafkaTemplate;
    private final AlertService alertService;

    public void sendToDeadLetter(TaskMessage message, Exception exception) {
        // Update execution status
        JobExecution execution = executionRepository
                .findByJobIdAndDagRunId(message.getJobId(), message.getDagRunId());
        execution.setStatus(ExecutionStatus.DEAD_LETTER);
        execution.setErrorMessage(truncate(exception.getMessage(), 2000));
        executionRepository.save(execution);

        // Publish to dead letter topic for manual review
        kafkaTemplate.send("task-execution-dlq", message.getJobId(), message);

        // Alert the team
        alertService.sendAlert(Alert.builder()
                .severity(AlertSeverity.HIGH)
                .title("Task failed permanently: " + message.getJobId())
                .message("Exhausted " + message.getAttemptNumber() + " retries. Last error: "
                        + exception.getMessage())
                .build());

        // Notify DAG engine to skip downstream tasks
        TaskResult failureResult = TaskResult.failure("Dead-lettered after max retries");
        // dagEngine.onTaskCompleted will handle marking downstream as SKIPPED
    }

    private String truncate(String s, int maxLen) {
        return s != null && s.length() > maxLen ? s.substring(0, maxLen) : s;
    }
}
```

## Defining a DAG

Here's how you'd define a daily report generation pipeline:

```java
@Configuration
public class ReportDagConfig {

    @Bean
    public CommandLineRunner setupReportDag(ScheduledJobRepository repository) {
        return args -> {
            if (repository.existsByDagId("daily-report")) return;

            ScheduledJob extractJob = ScheduledJob.builder()
                    .name("Extract Sales Data")
                    .cronExpression("0 2 * * *")  // 2 AM daily
                    .dagId("daily-report")
                    .executionOrder(1)
                    .taskHandlerClass("com.company.tasks.ExtractSalesDataTask")
                    .maxRetries(3)
                    .retryBackoffMs(5000)
                    .dependsOn(Collections.emptySet())
                    .build();

            ScheduledJob transformJob = ScheduledJob.builder()
                    .name("Transform and Aggregate")
                    .dagId("daily-report")
                    .executionOrder(2)
                    .taskHandlerClass("com.company.tasks.TransformSalesTask")
                    .maxRetries(2)
                    .retryBackoffMs(10000)
                    .dependsOn(Set.of(extractJob.getId()))
                    .build();

            ScheduledJob loadJob = ScheduledJob.builder()
                    .name("Load to Data Warehouse")
                    .dagId("daily-report")
                    .executionOrder(3)
                    .taskHandlerClass("com.company.tasks.LoadToWarehouseTask")
                    .maxRetries(3)
                    .retryBackoffMs(15000)
                    .dependsOn(Set.of(transformJob.getId()))
                    .build();

            ScheduledJob notifyJob = ScheduledJob.builder()
                    .name("Send Report Email")
                    .dagId("daily-report")
                    .executionOrder(4)
                    .taskHandlerClass("com.company.tasks.SendReportEmailTask")
                    .maxRetries(2)
                    .retryBackoffMs(5000)
                    .dependsOn(Set.of(loadJob.getId()))
                    .build();

            repository.saveAll(List.of(extractJob, transformJob, loadJob, notifyJob));
        };
    }
}
```

## A Sample Task Handler

```java
@Component
public class ExtractSalesDataTask implements TaskHandler {

    private final SalesRepository salesRepository;
    private final ObjectMapper objectMapper;

    public ExtractSalesDataTask(SalesRepository salesRepository, ObjectMapper objectMapper) {
        this.salesRepository = salesRepository;
        this.objectMapper = objectMapper;
    }

    @Override
    public TaskResult execute(TaskContext context) {
        LocalDate yesterday = LocalDate.now().minusDays(1);

        List<SalesRecord> records = salesRepository
                .findByTransactionDateBetween(
                        yesterday.atStartOfDay(),
                        yesterday.atTime(23, 59, 59));

        if (records.isEmpty()) {
            return TaskResult.failure("No sales records found for " + yesterday);
        }

        BigDecimal totalRevenue = records.stream()
                .map(SalesRecord::getAmount)
                .reduce(BigDecimal.ZERO, BigDecimal::add);

        Map<String, Object> output = Map.of(
                "recordCount", records.size(),
                "totalRevenue", totalRevenue.toPlainString(),
                "extractDate", yesterday.toString()
        );

        return TaskResult.success(output);
    }

    @Override
    public void onDeadLetter(TaskContext context, Exception lastException) {
        // Custom dead-letter handling for this specific task
        log.error("Sales extraction permanently failed for DAG run {}: {}",
                context.getDagRunId(), lastException.getMessage());
    }
}
```

## Monitoring and Observability

```java
@RestController
@RequestMapping("/api/scheduler")
public class SchedulerMonitoringController {

    private final JobExecutionRepository executionRepository;
    private final ScheduledJobRepository jobRepository;

    @GetMapping("/dag-runs/{dagRunId}")
    public DagRunStatus getDagRunStatus(@PathVariable String dagRunId) {
        List<JobExecution> executions = executionRepository.findByDagRunId(dagRunId);

        return DagRunStatus.builder()
                .dagRunId(dagRunId)
                .tasks(executions.stream().map(this::toTaskStatus).toList())
                .overallStatus(computeOverallStatus(executions))
                .build();
    }

    @GetMapping("/dead-letters")
    public Page<JobExecution> getDeadLetters(Pageable pageable) {
        return executionRepository.findByStatus(ExecutionStatus.DEAD_LETTER, pageable);
    }

    @PostMapping("/dead-letters/{executionId}/retry")
    public ResponseEntity<?> retryDeadLetter(@PathVariable String executionId) {
        // Manual retry of a dead-lettered task
        JobExecution execution = executionRepository.findById(executionId)
                .orElseThrow();
        execution.setStatus(ExecutionStatus.PENDING);
        execution.setAttemptNumber(1);
        executionRepository.save(execution);
        // Re-dispatch...
        return ResponseEntity.accepted().build();
    }
}
```

## Production Considerations

**Idempotent tasks** — Tasks may execute more than once (Kafka at-least-once delivery). Design tasks to be idempotent.

**Timezone handling** — Store all times in UTC. Cron expressions should specify timezone explicitly if business logic requires it.

**Backpressure** — If your task queue grows faster than consumers process, add more consumer pods or implement priority-based scheduling.

**Monitoring** — Export Prometheus metrics: tasks executed, failure rate, average duration per task type, queue depth, retry count.

**Security** — The task handler class resolution uses `Class.forName()`. In production, restrict this to a whitelist of allowed packages.

This scheduler has been running in production handling 50,000+ tasks daily across 3 teams. It's not Airflow — it's not trying to be. It's a Java-native solution that fits naturally into a Spring Boot microservices architecture.
