# Design a Distributed Job Scheduler (like Airflow)

## Requirements
- **Availability > consistency** — missing a job execution is worse than briefly stale metadata.
- **At-least-once execution** — jobs may run more than once on failure recovery → jobs should be idempotent.
- Quantify: execute within **2s** of scheduled time, support **10K jobs/sec**.
- Out of scope: CI/CD pipelines. In scope: security policy enforcement.

## Core Entities
- **User** — runs jobs, views status.
- **Task** — reusable template (what to do).
- **Job** — concrete instance of a task with specific params/timing/status.
- **Schedule** — CRON expression (recurring) or specific DateTime (one-off).
- **Execution** — a single run record for a job (status, attempt, timestamps).

## API
```
POST /jobs
{ "task_id": "send_email", "schedule": "0 10 * * *", "parameters": {...} }
→ { "job_id": ..., "status": "scheduled", "next_run_at": ... }

GET /jobs?user_id=&status=&start_time=&end_time= → Job[]  (paginated)
```
- Never pass `user_id` as a query param — use an Authorization/JWT header instead (avoids leaking identity via logs/history).
- Any list endpoint needs pagination (offset- or cursor-based).

## High-Level Design
- Client → Scheduling Service (`POST /jobs`) → stores Job + Schedule in DB (Dynamo/Cassandra for scale).
- Separate **Job** (definition) from **Execution** (a scheduled/run instance) so schedule isn't re-evaluated per query — executor just polls `scheduled_time <= now AND status = 'pending'`.
- Worker/executor picks up due Execution rows and runs them, updating status: `pending → running → success/failed`. Execution table = source of truth for monitoring.
- User history view = Execution table (past/in-progress) + pre-created pending Execution rows or Job table (future).
- To query "all jobs for a user" efficiently: DynamoDB GSI with partition key `user_id`, sort key `execution_time + job_id`.

## Deep Dives

**Hitting the 2s execution SLA at 10K jobs/sec**
- Naive polling every 2s doesn't scale. Use a two-tier approach:
  1. A coarse scan (e.g. every 5 min) loads jobs due in the next window and pushes them onto a message queue (SQS) with a **delay** set to the exact time until execution.
  2. Workers poll the queue and execute in order.
- For jobs created with a near-term due time (before the next scan), push directly to the queue from the scheduling service instead of waiting for the next scan cycle.

**Scaling to 10K concurrent jobs**
- Put a queue (Kafka/RabbitMQ) in front of the scheduling service so multiple instances can absorb write bursts without dropping jobs.
- Partition the Job table by `job_id` (naturally even). Partition the Execution table by time bucket, but **shard buckets** (e.g. N sub-partitions per hour) to avoid hot partitions from jobs clustering around the same time.
- Executors scale horizontally, all consuming the same SQS queue (SQS scales well; messages are small).

**Worker crash handling**
- Rely on SQS **visibility timeout**: message is invisible while a worker holds it; if the worker dies before deleting it, the message becomes visible again for another worker. Lease-based recovery, not just active failure reporting.

**Auto-scaling executors**
- Run workers on ECS/Kubernetes with a queue-depth-based scaling policy — capacity grows during bursts, shrinks as the queue drains.

**Retries**
- Exponential backoff **with jitter** to avoid thundering herd.
- Distinguish retryable (timeouts, rate limits) from non-retryable (bad input, missing resource) failures — send non-retryable failures straight to a dead-letter queue.
- Cap retry attempts (user- or system-configurable); after max retries → DLQ → human review.
