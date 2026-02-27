### 1) Goal and Non-Goals

**Goal:** For every incoming notification event, decide:

- **NOW** → send immediately
- **LATER** → defer/schedule (or batch/digest)
- **NEVER** → suppress (duplicate, spammy, expired, irrelevant, too noisy)

**Non-goals (explicit tradeoffs):**

- We are not building a full personalization recommender end-to-end; we use AI only where it adds value (semantic dedupe, intent classification).
- “Never” does not silently lose important events: we always keep an audit log and can optionally provide user-visible “suppressed notifications inbox.”

---

## 2) High-Level Architecture (Components + Data Flow)

### 2.1 Components

1. **Ingress API / Event Gateway**
    - Validates schema, assigns `event_id`, normalizes fields
    - Publishes event to a queue/stream (Kafka / PubSub)
2. **Prioritization Service (low-latency)**
    - Core decision engine: Now/Later/Never
    - Stateless compute with fast reads from Redis + rules cache
    - Produces a `Decision` record and emits downstream commands
3. **Rules Engine (human-configurable)**
    - Dynamic rules stored in DB and cached
    - UI/admin API to update rules without redeploy
4. **Deduplication Layer**
    - **Exact dedupe:** key-based and fingerprint-based
    - **Near-dedupe:** embeddings/simhash + time window + similarity threshold
    - Implemented with Redis + vector store (optional)
5. **Fatigue/Rate Limiter**
    - Per-user counters, cooldowns, caps, channel limits
    - Supports batching/digests
6. **Scheduler / Deferred Queue**
    - Stores “Later” notifications with `send_at`
    - Re-check eligibility at send time (important for safety)
7. **Notification Dispatcher**
    - Sends via channel providers (Push/Email/SMS/In-app)
    - Retries, DLQ, idempotency
8. **Audit & Observability**
    - Immutable decision log (append-only)
    - Metrics, tracing, dashboards, alerts

---

### 2.2 Data Flow (Happy path)

1. Producer service → `POST /v1/events`
2. Ingress validates → Stream `notification_events`
3. Prioritization service consumes event:
    - fetch user state + recent history summary
    - evaluate rules + dedupe + fatigue + AI (if needed)
    - output decision: NOW/LATER/NEVER with explanation
4. If NOW → send command to dispatcher
5. If LATER → store schedule + enqueue to scheduler
6. If NEVER → suppress but log, optionally store in “suppressed inbox”

---

## 3) Decision Logic Design (Now / Later / Never)

### 3.1 Deterministic-first, AI-assisted approach

**Why:** Low latency and auditability demand deterministic gates. AI should be **optional** and **bounded**.

Decision is a pipeline of gates:

#### Gate A: Hard validity checks (always deterministic)

- If `expires_at` exists and `timestamp > expires_at` → **NEVER** (expired)
- If missing critical fields (user_id/event_type/channel) → **LATER** to retry enrichment OR **NEVER** if invalid (based on policy)

#### Gate B: Rules engine (human-configurable)

Rules can force:

- Always NOW (ex: security alerts)
- Always NEVER (ex: deprecated promo types)
- Channel constraints (ex: never SMS for promotions)
- Quiet hours (convert NOW→LATER)
- User segment overrides

#### Gate C: Deduplication

- Exact duplicates: suppress (NEVER) or collapse into one (NOW with merged content)
- Near-duplicates: suppress or batch, depending on event type

#### Gate D: Fatigue / frequency control

- per-user/channel caps (ex: max 3 push per 10 minutes)
- cooldown windows per event_type
- if cap exceeded → **LATER** (schedule) or **NEVER** for low-value class

#### Gate E: Priority resolution (conflicts)

Resolve conflicts like “urgent but noisy” with a **score + policy**:

- compute `base_priority` (rules or hint)
- compute `urgency` (time sensitivity)
- compute `noise_penalty` (recent volume, duplicates, user fatigue)
- compute `value_class` (transactional vs promotional)

Then:

- if score ≥ threshold → NOW
- else if still relevant later → LATER with `send_at`
- else → NEVER

#### Gate F: Final safety checks

- If downstream provider unhealthy → **LATER** (retry) instead of losing important notifications
- If AI timed out → proceed with deterministic result + log `ai_used=false`

---

### 3.2 A Practical Scoring Model (explainable)

Compute:

- `P` = priority (0–100)
- `U` = urgency boost (0–30)
- `N` = noise penalty (0–50)
- `T` = time sensitivity decay (0–40)

Example:

`score = P + U - N - T`

**Mapping**

- `score >= 70` → NOW
- `40 <= score < 70` → LATER (schedule)
- `score < 40` → NEVER (suppress)

Explainability: log the components used.

---

## 4) Minimal Data Model (Tables / Records)

### 4.1 `notification_event`

- `event_id` (uuid)
- `user_id`
- `event_type`
- `source`
- `channel`
- `title`
- `body`
- `metadata` (json)
- `priority_hint` (optional)
- `timestamp`
- `expires_at` (optional)
- `dedupe_key` (optional)
- `normalized_text` (optional, for hashing/embedding)

### 4.2 `user_notification_state` (fast cache in Redis, source of truth in DB)

- `user_id`
- `quiet_hours` (json)
- `channel_preferences` (json)
- `fatigue_score` (rolling)
- `counters` (windowed counts per channel/type)

### 4.3 `dedupe_index` (Redis)

- `user_id + fingerprint` → last_seen_event_id, last_seen_ts
- TTL based on event_type (e.g., 5 min for system updates, 24h for promos)

### 4.4 `deferred_notification`

- `defer_id`
- `event_id`
- `user_id`
- `send_at`
- `reason`
- `status` (scheduled/sent/canceled/expired)

### 4.5 `decision_audit_log` (append-only)

- `decision_id`
- `event_id`
- `user_id`
- `decision` (NOW/LATER/NEVER)
- `final_channel`
- `send_at` (if later)
- `explanation` (json)
    - `rules_applied`
    - `dedupe_result`
    - `fatigue_metrics`
    - `score_breakdown`
    - `ai_used`, `ai_latency_ms`, `fallback_reason`
- `created_at`

**Important:** Even suppressed events are logged here → “Important notifications should not be silently lost.”

---

## 5) API / Service Interfaces (3–5 endpoints)

### 5.1 Ingest event

**POST** `/v1/notifications/events`

Request:

```json
{
  "user_id": "u123",
  "event_type": "payment_failed",
  "source": "billing",
  "channel": "push",
  "title": "Payment failed",
  "message": "Your card was declined",
  "priority_hint": "high",
  "timestamp": "2026-02-27T10:00:00Z",
  "dedupe_key": "billing:payment_failed:invoice_99",
  "expires_at": "2026-02-27T12:00:00Z",
  "metadata": {"invoice_id": "99"}
}
```

Response:

```json
{
  "event_id": "evt_abc",
  "status": "accepted"
}
```

### 5.2 Get decision (optional, for debugging)

**GET** `/v1/notifications/events/{event_id}/decision`

### 5.3 Update rules (admin)

**PUT** `/v1/notification-rules/{rule_id}`

### 5.4 List rules (admin/UI)

**GET** `/v1/notification-rules?active=true`

### 5.5 User preferences (optional)

**PATCH** `/v1/users/{user_id}/notification-preferences`

---

## 6) Duplicate Prevention (Exact + Near-Duplicate)

### 6.1 Exact dedupe (fast)

Use a fingerprint fallback when `dedupe_key` is missing/unreliable.

Fingerprint inputs:

- `user_id`
- `event_type`
- `channel`
- normalized title/body (lowercase, strip punctuation, remove ids if policy says so)
- stable metadata fields (invoice_id, order_id, thread_id if present)

Fingerprint techniques:

- `sha256(normalized_string)`
- store in Redis with TTL window

Decision:

- if fingerprint seen within TTL:
    - for critical transactional → suppress duplicates (NEVER) but keep latest if newer info
    - for chat/messages → collapse (“3 new messages”) and possibly LATER for batching

### 6.2 Near dedupe (bounded AI or hashing)

For near duplicates (e.g., same meaning different wording):

- Use **SimHash** or **MinHash** on normalized tokens for cheap approximate similarity
- Or embeddings with a small vector index (per user or per event_type)

Policy:

- similarity >= 0.92 within window → near-dup
- action: LATER (batch/digest) for low/medium value, NEVER for promos, NOW only for truly urgent classes

Fail-safe:

- if embedding service fails → skip near-dedupe and rely on exact dedupe.

---

## 7) Alert Fatigue Strategy (Cooldowns, Caps, Batching)

### 7.1 Controls (per user + channel)

- **Cooldown by type:** “system_update” min 30 min
- **Window caps:** max 3 push / 10 min, max 1 SMS / 30 min
- **Escalation policy:** if too many low-value events → auto LATER/NEVER for promos
- **Digest mode:** group “updates/promotions” into daily digest email or in-app inbox

### 7.2 Batching rules examples

- If 5 events of type “comment_added” within 2 minutes → send 1 notification: “5 new comments”
- For messages: send NOW only for @mentions or priority contacts; else LATER by 2–5 minutes.

---

## 8) Fallback Strategy (AI slow/unavailable, dependent services down)

### 8.1 Timeouts + circuit breaker

- AI calls have hard timeout (e.g., 50–100ms budget)
- If timeout:
    - continue with deterministic rules + exact dedupe + fatigue
    - log `ai_used=false`, `fallback_reason="AI_TIMEOUT"`

### 8.2 Dependency failure modes

- Redis down → fail open or fail closed depends on event class:
    - **Critical transactional/security:** default NOW (fail-open) but still log + best-effort dedupe in memory
    - **Promotional:** default LATER or NEVER (fail-closed)
- Dispatcher down → LATER (enqueue retry) to avoid loss

### 8.3 “Important should not be silently lost”

- For any NOW decision that cannot be delivered after retries:
    - move to DLQ
    - alert on-call
    - expose to user “Notification Inbox” as fallback (product choice)

---

## 9) Explainability & Auditability (What to log)

Every decision should produce a structured explanation:

Example explanation JSON:

```json
{
  "decision": "LATER",
  "send_at": "2026-02-27T10:05:00Z",
  "rules_applied": ["quiet_hours_push", "promo_deprioritize_when_noisy"],
  "dedupe": {"exact": false, "near": true, "similarity": 0.95},
  "fatigue": {"push_count_10m": 3, "cap_10m": 3, "cooldown_active": true},
  "score": {"P": 40, "U": 0, "N": 25, "T": 0, "final": 15},
  "ai_used": true,
  "ai_latency_ms": 42
}
```

This makes the system auditable and debuggable.

---

## 10) Metrics & Monitoring Plan

### 10.1 Core metrics

- Decision distribution: `%NOW`, `%LATER`, `%NEVER`
- Delivery success rate by channel
- Mean decision latency (p50/p95/p99)
- AI usage rate + AI latency + AI timeout rate
- Duplicate suppression rate (exact vs near)
- Fatigue suppression rate (by event type)
- Deferred queue depth + time-to-send
- DLQ volume and reasons

### 10.2 Quality metrics (product health)

- User opt-out/uninstall rates (push)
- Notification open/click rate (by type)
- Complaint signals (muted channels, “too many notifications”)
- “Important missed” indicators (support tickets, critical event not delivered)

### 10.3 Alerts

- p99 decision latency above threshold
- dispatcher failure spikes
- DLQ > threshold
- Redis/vector store down
- sudden change in NOW rate (regression)

---

## 11) Example Rule Set (Human-configurable)

Rule format (simple DSL):

- conditions: event_type, source, channel, time window, user segment, priority_hint
- actions: force NOW/LATER/NEVER, set cooldown, set batch window, set channel

Examples:

1. `event_type in ["security_alert", "payment_failed"] => force NOW`
2. `channel="sms" and event_type="promotion" => NEVER`
3. `quiet_hours=true and channel="push" and event_type not critical => LATER send_at=next_allowed_time`
4. `if push_count_10m >= 3 and value_class="promo" => NEVER`
5. `if near_duplicate=true => LATER batch_window=5m`

---

# What I need from you to finalize this into a perfect submission

1) Do you want your deliverable as:

- a [**README.md](http://README.md) system design doc**, or
- a **PDF-style report**, or
- both?

2) Are you expected to include **diagrams** (architecture/data flow)? If yes, I can give you a Mermaid diagram you can paste into GitHub.
