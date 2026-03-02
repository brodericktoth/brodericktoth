# Rust Job Execution API + Queue + Worker

This guide shows a practical way to evolve a CLI task runner into a production-ready **job execution platform** with:

- an HTTP API for submitting jobs,
- a queue for durable scheduling,
- worker processes for execution,
- and a shared core library reused by both CLI and workers.

## 0) First step (the part most people get stuck on)

If this felt abstract, start with this exact setup.

Create **1 Rust workspace** with **4 projects (crates)**:

1. `core` (library crate) — shared task types + execution logic.
2. `cli` (binary crate) — your existing command-line app, now calling `core`.
3. `api` (binary crate) — HTTP service that accepts jobs.
4. `worker` (binary crate) — background process that executes queued jobs.

### Minimum files to create right now

You can begin with just **7 files**:

1. `Cargo.toml` (workspace root)
2. `crates/core/Cargo.toml`
3. `crates/core/src/lib.rs`
4. `crates/cli/Cargo.toml`
5. `crates/cli/src/main.rs`
6. `crates/api/Cargo.toml`
7. `crates/worker/Cargo.toml`

Then add these two when you are ready to run API/worker processes:

- `crates/api/src/main.rs`
- `crates/worker/src/main.rs`

So:

- **Day 1 scaffold only**: 7 files
- **Runnable 4-crate baseline**: 9 files

### What goes in each file (minimal)

- `core/src/lib.rs`: define `TaskSpec`, `JobInput`, `JobOutput`, and `TaskExecutor` trait.
- `cli/src/main.rs`: parse args and call `core` directly.
- `api/src/main.rs`: expose `POST /jobs` + `GET /jobs/:id`.
- `worker/src/main.rs`: poll queue, claim job, call `core`, update status.

### Why this is step 1

You’re not building everything at once. You’re only creating boundaries so the same task logic is reused by:

- local CLI runs (`--local`), and
- background worker runs (`--remote` via API + queue).

If you do only this first step, you already avoid the biggest rewrite mistake: duplicating business logic in CLI and worker.


## 0.1 Exact terminal walkthrough (copy/paste)

If you are starting from scratch, run these commands from your repo root:

```bash
# 1) Create workspace folders
mkdir -p crates

# 2) Create crates
cargo new crates/core --lib
cargo new crates/cli --bin
cargo new crates/api --bin
cargo new crates/worker --bin

# 3) Create/replace workspace Cargo.toml at repo root
cat > Cargo.toml <<'EOF'
[workspace]
members = [
  "crates/core",
  "crates/cli",
  "crates/api",
  "crates/worker"
]
resolver = "2"
EOF

# 4) Verify the workspace compiles
cargo check
```

### Do I install crates with terminal or just import in files?

Both, but in Rust that means **adding dependencies in each crate's `Cargo.toml`**, then importing in code.

- You can edit `Cargo.toml` manually, or
- use terminal helpers like `cargo add serde --features derive`.

You **do not** install crates globally like `npm -g`. Dependencies are project-local per crate.

### Minimal dependency set per crate

`crates/core/Cargo.toml`

```toml
[dependencies]
serde = { version = "1", features = ["derive"] }
serde_json = "1"
thiserror = "1"
async-trait = "0.1"
```

`crates/cli/Cargo.toml`

```toml
[dependencies]
core = { path = "../core" }
clap = { version = "4", features = ["derive"] }
tokio = { version = "1", features = ["macros", "rt-multi-thread"] }
```

`crates/api/Cargo.toml`

```toml
[dependencies]
core = { path = "../core" }
axum = "0.7"
serde = { version = "1", features = ["derive"] }
serde_json = "1"
tokio = { version = "1", features = ["macros", "rt-multi-thread"] }
sqlx = { version = "0.8", features = ["runtime-tokio-rustls", "postgres", "uuid", "json", "chrono"] }
uuid = { version = "1", features = ["v4", "serde"] }
tracing = "0.1"
tracing-subscriber = { version = "0.3", features = ["fmt", "env-filter"] }
```

`crates/worker/Cargo.toml`

```toml
[dependencies]
core = { path = "../core" }
tokio = { version = "1", features = ["macros", "rt-multi-thread", "time"] }
sqlx = { version = "0.8", features = ["runtime-tokio-rustls", "postgres", "uuid", "json", "chrono"] }
uuid = { version = "1", features = ["v4", "serde"] }
tracing = "0.1"
tracing-subscriber = { version = "0.3", features = ["fmt", "env-filter"] }
```

## 0.2 Exactly how many files you should code in first

For the first working milestone, focus on **10 files total**:

1. `Cargo.toml` (workspace root)
2. `crates/core/Cargo.toml`
3. `crates/core/src/lib.rs`
4. `crates/cli/Cargo.toml`
5. `crates/cli/src/main.rs`
6. `crates/api/Cargo.toml`
7. `crates/api/src/main.rs`
8. `crates/worker/Cargo.toml`
9. `crates/worker/src/main.rs`
10. `migrations/0001_create_jobs.sql`

Everything else can wait.

## 0.3 Copy code from task runner or import it?

Short answer: **copy once, then import forever**.

1. Copy your real task logic from current CLI into `core` (one-time move).
2. Replace old CLI logic with a call into `core`.
3. API and worker also call `core`.

So after extraction:

- `core` owns business logic,
- `cli`, `api`, and `worker` only orchestrate.

Avoid keeping duplicate task logic in both `cli` and `worker`.

## 0.4 Function of each file you write

- `Cargo.toml` (root): declares workspace members.
- `core/src/lib.rs`: types + trait + task execution entrypoint.
- `cli/src/main.rs`: parse command, build `TaskSpec`, call `core`.
- `api/src/main.rs`: receive HTTP request, validate, insert `jobs` row.
- `worker/src/main.rs`: claim queued jobs, call `core`, write status/result.
- `migrations/0001_create_jobs.sql`: creates durable queue table.

## 0.5 Concepts you are applying (and why)

- **Separation of concerns**: transport (CLI/API) is separated from business logic (`core`).
- **Single source of truth**: one executor path avoids drift/bugs.
- **Durable queue**: DB keeps jobs safe across crashes/restarts.
- **Idempotency**: same request key won’t execute side effects twice.
- **At-least-once processing**: retries handle transient failures.
- **Atomic claim**: `FOR UPDATE SKIP LOCKED` prevents duplicate workers on same job.

## 0.6 First milestone checklist (what to finish now)

- [ ] Workspace with 4 crates compiles.
- [ ] `core` defines `TaskSpec`, `JobInput`, `JobOutput`, `TaskExecutor`.
- [ ] CLI runs one real task through `core`.
- [ ] Migration creates `jobs` table.
- [ ] API can enqueue (`POST /jobs`).
- [ ] Worker can claim + execute + mark success/failure.
- [ ] `GET /jobs/:id` returns status.

If these are done, you already have a real end-to-end system.

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

### Do you need a CLI if jobs come from SQL?

Short answer: **No, a CLI is optional**.

If your product only accepts jobs through API -> DB queue -> worker, you can skip a separate CLI crate entirely.

Why teams still keep a CLI:

- **Operator tooling**: manually submit/retry/cancel jobs during incidents.
- **Local development**: run one task directly without booting API + DB + worker.
- **Smoke tests**: quick pre-deploy verification path.
- **Backfill/batch scripts**: trigger controlled maintenance tasks.

So the decision is:

- No operational need for command tooling -> don't build CLI now.
- Need fast manual/ops workflows -> keep a thin CLI adapter that calls `core` or hits the API.

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
