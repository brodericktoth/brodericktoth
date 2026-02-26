# Rust Job Execution API + Queue + Worker

This guide shows a practical way to evolve a CLI task runner into a production-ready **job execution platform** with:

- an HTTP API for submitting jobs,
- a queue for durable scheduling,
- worker processes for execution,
- and a shared core library reused by both CLI and workers.

## 1) Target architecture

Split into a Rust workspace:

- `crates/core` → domain logic, task definitions, execution traits, validation, serialization.
- `crates/cli` → existing task runner binary; now just parses args and calls `core`.
- `crates/api` → axum/actix HTTP service that writes jobs to DB/queue.
- `crates/worker` → polling or queue-consumer process executing jobs.
- `crates/common` (optional) → cross-cutting config/logging/telemetry/error helpers.

Typical production flow:

1. Client `POST /jobs` with `{ task_type, payload, priority }`.
2. API validates and stores a `job` row in Postgres as `queued`.
3. Queue signal is emitted (Redis stream, NATS, RabbitMQ, SQS, or DB notify).
4. Worker receives signal, claims job atomically, executes through `core`.
5. Worker updates job state (`running` → `succeeded`/`failed`) and stores output.

## 2) Extract your CLI runner "core"

Your CLI probably has mixed concerns today (arg parsing + execution + output). Refactor by moving all true business logic into `core`.

## 2.1 Identify extractable pieces

Move these out of CLI:

- Task enum / command model (`ResizeImage`, `ImportCsv`, etc.)
- Input schema and validation
- Execution contract (`TaskExecutor` trait)
- Retryability classification (transient vs permanent failures)
- Deterministic serialization for payloads

Keep in CLI:

- clap argument definitions
- terminal rendering / pretty logs
- local dev conveniences (e.g., reading env files)

## 2.2 Define stable job contract

In `core`, define a versioned contract:

```rust
use serde::{Deserialize, Serialize};

#[derive(Debug, Clone, Serialize, Deserialize)]
#[serde(tag = "type", content = "payload")]
pub enum TaskSpec {
    SendEmail { to: String, template_id: String, data_json: String },
    GenerateReport { report_id: String, from: String, to: String },
}

#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct JobInput {
    pub idempotency_key: Option<String>,
    pub task: TaskSpec,
}

#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct JobOutput {
    pub message: String,
    pub artifacts: Vec<String>,
}
```

This is what API receives and worker executes. Keep this format backward-compatible.

## 2.3 Add an executor abstraction

```rust
use async_trait::async_trait;

pub type CoreResult<T> = Result<T, CoreError>;

#[async_trait]
pub trait TaskExecutor: Send + Sync {
    async fn execute(&self, input: JobInput) -> CoreResult<JobOutput>;
}
```

Then implement it with your existing task logic migrated from CLI.

## 3) Data model for durable jobs

Use Postgres for source-of-truth (even if another queue is used):

```sql
create type job_status as enum ('queued', 'running', 'succeeded', 'failed', 'dead_letter');

create table jobs (
  id uuid primary key,
  task_type text not null,
  payload jsonb not null,
  status job_status not null default 'queued',
  attempts int not null default 0,
  max_attempts int not null default 5,
  priority int not null default 100,
  idempotency_key text,
  run_at timestamptz not null default now(),
  locked_by text,
  locked_at timestamptz,
  last_error text,
  output jsonb,
  created_at timestamptz not null default now(),
  updated_at timestamptz not null default now()
);

create unique index jobs_idempotency_key_uniq
  on jobs (idempotency_key)
  where idempotency_key is not null;

create index jobs_poll_idx on jobs (status, run_at, priority, created_at);
```

## 4) Queue strategy options

Pick one first; don’t overengineer.

- **Simplest**: Postgres-only queue using `FOR UPDATE SKIP LOCKED`.
- **Hybrid**: Postgres for durability + Redis/NATS for wake-up signal.
- **Cloud-native**: SQS/PubSub + Postgres metadata.

If you’re starting solo/small team: start with Postgres-only, scale later.

## 5) Worker claim algorithm (critical)

Claim jobs atomically to avoid double-execution:

```sql
with cte as (
  select id
  from jobs
  where status = 'queued'
    and run_at <= now()
  order by priority asc, created_at asc
  for update skip locked
  limit 1
)
update jobs j
set status='running',
    locked_by=$1,
    locked_at=now(),
    attempts=j.attempts + 1,
    updated_at=now()
from cte
where j.id = cte.id
returning j.*;
```

After execution:

- success → `status='succeeded'`, set `output`
- transient failure with attempts left → `status='queued'`, exponential backoff in `run_at`
- permanent failure or max attempts reached → `status='failed'`/`dead_letter`

## 6) API surface (minimum)

- `POST /jobs` → enqueue
- `GET /jobs/:id` → status/details
- `POST /jobs/:id/cancel` (optional)
- `GET /healthz` and `/readyz`

`POST /jobs` behavior:

- Validate `JobInput` in `core`.
- Enforce idempotency (header or body key).
- Persist row and return `202 Accepted` with `job_id`.

## 7) Reusing CLI on top of core

Your CLI becomes a thin adapter:

- parse command into `TaskSpec`
- either execute directly (`core.execute`) OR submit to API (`POST /jobs`)

This gives two modes:

- `task run report --local` (direct local execution)
- `task run report --remote` (enqueue remotely)

Same task schema in both modes means no duplication.

## 8) Reliability patterns you should add early

- **Idempotency key**: dedupe duplicate submissions.
- **Retry policy**: exponential backoff + jitter.
- **Heartbeat/timeout**: detect stuck running jobs.
- **Lease expiry recovery**: requeue jobs abandoned by crashed workers.
- **Dead-letter queue**: inspect poison jobs.
- **Structured logs + trace IDs**: correlate API request to worker execution.

## 9) Crate/tool recommendations

- API: `axum`, `tower`, `utoipa` (optional OpenAPI)
- DB: `sqlx` or `diesel` (sqlx is common for async)
- Queue wakeup: `tokio::sync::Notify` (local), Redis streams, or NATS
- Serialization: `serde`, `serde_json`
- Errors: `thiserror`, `anyhow` (boundary only)
- Tracing: `tracing`, `tracing-subscriber`
- Runtime: `tokio`

## 10) Incremental migration plan (safe)

1. **Refactor CLI**: isolate existing task logic behind `TaskExecutor` in `core`.
2. **Add DB schema + repository layer** in `api`/`worker`.
3. **Build enqueue API** (`POST /jobs`, `GET /jobs/:id`).
4. **Build worker poll loop** with atomic claim SQL.
5. **Wire retries + backoff + dead-letter**.
6. **Add idempotency + observability**.
7. **Deploy one API + one worker**, then scale worker replicas.

## 11) Skeleton workspace layout

```text
.
├─ Cargo.toml
├─ crates/
│  ├─ core/
│  │  ├─ src/lib.rs
│  │  ├─ src/task.rs
│  │  └─ src/executor.rs
│  ├─ api/
│  │  ├─ src/main.rs
│  │  ├─ src/routes/jobs.rs
│  │  └─ src/repo/jobs_repo.rs
│  ├─ worker/
│  │  ├─ src/main.rs
│  │  └─ src/runner.rs
│  └─ cli/
│     └─ src/main.rs
└─ migrations/
```

## 12) Practical pseudo-code

Worker loop:

```rust
loop {
    if let Some(job) = repo.claim_next(worker_id).await? {
        let input = repo.to_job_input(&job)?;
        let result = executor.execute(input).await;
        match result {
            Ok(output) => repo.mark_success(job.id, output).await?,
            Err(err) if err.is_retryable() && job.attempts < job.max_attempts => {
                let next_run = backoff(job.attempts);
                repo.requeue(job.id, err.to_string(), next_run).await?;
            }
            Err(err) => repo.mark_failed(job.id, err.to_string()).await?,
        }
    } else {
        tokio::time::sleep(std::time::Duration::from_millis(500)).await;
    }
}
```

## 13) What to extract first from your current CLI

Start with these exact extractions:

- `TaskSpec` enum
- per-task input structs
- validation functions
- `execute_task(spec) -> Result<JobOutput, CoreError>`
- error typing: `CoreError::{Retryable, Permanent}`

Once this compiles in `core`, building API+worker gets straightforward.

## 14) Common pitfalls

- Putting queue metadata inside task payload (keep separate columns).
- No idempotency support (causes duplicate side effects).
- Non-atomic claim logic (multiple workers run same job).
- Returning `200 OK` on enqueue (prefer `202 Accepted`).
- Making worker logic depend on CLI argument parsing code.

---

If you want, I can generate a concrete starter workspace (`core` + `api` + `worker` + migrations) using `axum + sqlx + Postgres` so you can paste it directly into your repo.
