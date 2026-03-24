# TODOS

## Eng Review (Implementation)

### [A] ADK Session as single source of truth

**What:** Use ADK's `session.id` as the authoritative session ID throughout the platform. Messaging adapters map `platform_channel + platform_user` → ADK session ID at creation time.

**Why:** Prevents drift between two session identity systems. If we store our own `UnifiedMessage.session_id` separate from ADK's `session.id`, they'll eventually diverge and debugging will be painful.

**Context:** Found during eng review. Affects `platform/adapter.py` and `messaging/session.py`. The adapter creates the mapping, but ADK's session service owns the authoritative ID.

**Effort:** S
**Priority:** P1
**Depends on:** Phase 1 foundation (ADK runner setup)

### [B] Orchestrator health check in Phase 1

**What:** Implement a simple watchdog that monitors orchestrator responsiveness and restarts it if hung (e.g., no heartbeat for N seconds).

**Why:** A crash loop or deadlock in the single orchestrator takes down the entire platform. Moving this from Phase 4 to Phase 1 is cheap insurance.

**Context:** Plan originally deferred this to Phase 4. Eng review recommended moving to Phase 1 given it's only ~2 hours of work and prevents painful debugging during early development.

**Effort:** S
**Priority:** P2
**Depends on:** Phase 1 — orchestrator agent running

### [C] Rate limiting in SlackAdapter

**What:** Add token bucket rate limiting before Slack API calls to prevent 429 errors under load.

**Why:** Slack's API rate limits (~1 req/sec) will cause message loss or delay if we send concurrently. Proactive rate limiting is cleaner than reactive retry.

**Context:** Found during eng review. Affects `platform/slack/adapter.py`. Standard token bucket implementation.

**Effort:** M
**Priority:** P2
**Depends on:** Phase 1 — Slack adapter functional

### [D] Async embedding generation in Phase 3

**What:** When adding pgvector in Phase 3, generate embeddings asynchronously in a background task rather than synchronously on every memory write.

**Why:** Embedding API calls add ~200ms latency per memory store. Background async prevents blocking the response path and smooths out API quota usage.

**Context:** Found during performance review. Affects `memory/knowledge/embedding.py`. Use `asyncio.create_task()` or a task queue.

**Effort:** M
**Priority:** P2
**Depends on:** Phase 3 — PostgreSQL + pgvector setup

### [E] Immediate 200 + background task queue

**What:** Slack webhook endpoint returns HTTP 200 immediately after validation, then queues message processing as a background task. Separate worker sends the actual Slack reply.

**Why:** Slack's HTTP webhook timeout is ~3 seconds. Full orchestrator → specialist → reply can easily exceed this. Must acknowledge before processing completes.

**Context:** Found during performance review. Affects `platform/web/adapter.py` and `main.py`. Use `asyncio.create_task()` or a lightweight task queue (e.g., in-memory queue with a worker coroutine).

**Effort:** M
**Priority:** P2
**Depends on:** Phase 1 — orchestrator agent + Slack adapter

## Completed
