# Grid Governance Engine — Architectural Blueprint

> **Production-grade specification** for running distributed AI agents in a resource-negotiation simulation.

---

## 1. System Overview

### Vision & Business Value

The Grid Governance Engine is an **autonomous, resource-negotiation simulation** where agents negotiate over scarce resources (energy, water, credits) without a central orchestrator. The system delivers:

- **Continuous operation**: Agents propose, validate, and execute trades in a deterministic loop.
- **Auditability**: Every decision is traceable via the ledger and governance tooling.
- **Recoverability**: State can be rehydrated from the ledger at any point in time.

### Tech Stack

| Layer         | Technology | Role                                                     |
|---------------|------------|----------------------------------------------------------|
| **Workflow**  | n8n        | Orchestrates triggers, AI calls, validation, and logging |
| **Database**  | Postgres   | Immutable ledger, source of truth for state              |
| **AI**        | Gemini     | Generates proposals from context and constraints         |
| **App**       | Python     | Validator service; atomic writes; health checks          |
| **Dashboard** | Streamlit  | Observability, monitoring, and manual overrides          |

---

## 2. Data Governance

### SQL Schema

| Table         | Purpose                                                    |
|---------------|------------------------------------------------------------|
| `ledger`      | Append-only log of applied transactions (source of truth)  |
| `actors`      | Registry of participants (nodes, agents)                   |
| `resources`   | Resource types (energy, water, credits)                    |
| `balances`    | Cached balance per actor per resource; reconstructible     |
| `rejected_proposals` | Audit log of rejected trades (proposal_id, error_code) |

### Ledger Table (with Versioning)

| Column           | Type        | Description                                      |
|------------------|-------------|--------------------------------------------------|
| `id`             | bigint      | Monotonic sequence                               |
| `proposal_id`    | uuid        | **UNIQUE** — idempotency key; no duplicate trades |
| `actor_from`     | uuid        | Source actor                                     |
| `actor_to`       | uuid        | Destination actor                                |
| `resource`       | text        | Resource type                                    |
| `amount`         | numeric     | Transfer amount                                  |
| `validator_version` | text     | Git commit hash or version tag of approving code |
| `executed_at`    | timestamptz | When applied                                     |
| `metadata`       | jsonb       | Optional context (AI reasoning, etc.)            |

**Integrity constraints:**

- `UNIQUE(proposal_id)` — Prevents duplicate execution on n8n retries.
- `CHECK(amount > 0)` — No zero or negative transfers.
- Foreign keys: `actor_from`, `actor_to` → `actors.id`; `resource` → `resources.name`.

### WorldState (JSON)

A snapshot derived from the ledger for AI context and dashboard display:

```json
{
  "epoch": 42,
  "timestamp": "2025-02-28T12:00:00Z",
  "actors": [
    { "id": "actor-1", "type": "node", "balances": { "energy": 100, "water": 50 } },
    { "id": "actor-2", "type": "node", "balances": { "energy": 80, "water": 120 } }
  ],
  "resources": ["energy", "water", "credits"],
  "constraints": { "max_transfer": 50, "min_balance": 10 }
}
```

- **Source**: Replay ledger or materialized view. Not authoritative—the ledger is.
- **Purpose**: Input to Gemini; display in Streamlit.

### Idempotency

Every transaction must have a **unique `proposal_id`**. If an n8n workflow retries due to a network glitch:

1. First attempt: Proposal executes; ledger row inserted with `proposal_id`.
2. Retry: Validator checks `SELECT 1 FROM ledger WHERE proposal_id = ?`. If exists → return success without re-executing.

This prevents double-spend and duplicate ledger entries.

---

## 3. Control Plane

### n8n ↔ Validator Interaction

```
┌─────────────┐     ┌─────────────┐     ┌─────────────┐     ┌─────────────┐
│   Trigger   │────▶│  n8n Flow   │────▶│   Gemini    │────▶│  Proposal   │
│ (schedule,  │     │             │     │  (propose   │     │  TradeJSON  │
│  webhook)   │     │             │     │   action)   │     │             │
└─────────────┘     └─────────────┘     └─────────────┘     └──────┬──────┘
                                                                   │
                                                                   ▼
┌─────────────┐     ┌─────────────┐     ┌─────────────┐     ┌─────────────┐
│   Linear    │◀────│  n8n        │◀────│  Validator  │◀────│  POST       │
│  (ticket)   │     │  (orchestr.)│     │  Service    │     │  /execute   │
└─────────────┘     └─────────────┘     └─────────────┘     └─────────────┘
```

**Flow:**

1. **Trigger**: n8n runs on schedule or webhook.
2. **Fetch WorldState**: `GET /world-state` from Validator (derived from Postgres).
3. **AI Proposal**: n8n sends WorldState + constraints to Gemini; Gemini returns `TradeJSON` with `proposal_id`.
4. **Execute**: n8n calls `POST /execute` with the proposal. Validator validates → atomic write or reject.
5. **Log**: On reject, n8n opens Linear ticket; on success, optionally logs.

**Timeouts:** n8n must set a timeout (e.g., 30s) on the Validator call. If the Validator hangs, n8n times out and can retry with the same `proposal_id` (idempotent).

---

## 4. Governance & HITL

### Health Checks

**`GET /health`** — Returns service status. Must verify:

| Check              | Pass condition                                      |
|--------------------|-----------------------------------------------------|
| Database reachable | Postgres connection succeeds                        |
| Ledger–Balances consistency | Balances table matches ledger-derived state |
| Circuit breaker    | Not in "halt" mode                                  |

**Ledger drift detection:** Recompute expected balances from ledger; compare to `balances` table. If mismatch → return `503` and log alert. This catches corruption or partial writes.

### Dispute Logic (Conflict Loop)

When a trade is **rejected**:

```
Rejection → rejected_proposals INSERT (proposal_id, validation_error_code, proposal_json)
         → Linear ticket created with: proposal_id, validation_error_code, timestamp
         → Discord alert (optional)
         → n8n workflow continues (does not block next proposal)
```

**Linear ticket payload:**

- `proposal_id`: Unique reference for audit.
- `validation_error_code`: e.g., `INSUFFICIENT_BALANCE`, `INVALID_ACTOR`, `CONSTRAINT_VIOLATION`.
- Link to proposal JSON (or store in ticket body).

Human board members review these tickets and tune the Validator code to reduce future disputes.

### Human Intervention Interface ("God-Mode" Overrides)

Manual override actions for emergency or policy correction. Require authentication (API key or admin role).

| Action                | Endpoint           | Example                                                                 |
|-----------------------|--------------------|-------------------------------------------------------------------------|
| Force-update balance  | `POST /admin/balance` | `curl -X POST .../admin/balance -d '{"actor_id":"...","resource":"credits","amount":1000}'` |
| Reset agent credit   | `POST /admin/reset-credit` | `curl -X POST .../admin/reset-credit -d '{"actor_id":"..."}'`       |
| Circuit breaker ON   | `POST /admin/circuit-breaker` | `curl -X POST .../admin/circuit-breaker -d '{"enabled":true}'`   |
| Circuit breaker OFF  | Same               | `{"enabled":false}`                                                     |

**Audit**: All admin actions must be logged to `ledger` or a separate `admin_audit` table with `admin_user`, `action`, `payload`, `timestamp`.

---

## 5. Operational Lifecycle

### Deployment

- Validator: Deploy as container or serverless (e.g., Cloud Run, Lambda).
- n8n: Self-hosted or n8n Cloud; ensure network access to Validator and Postgres.
- Postgres: Managed (e.g., Supabase, RDS) with backups and point-in-time recovery.
- Streamlit: Deploy separately; read-only access to WorldState and Ledger.

### State Rehydration Protocol

Reconstructs the `balances` table from the `ledger`. **See Section 8 (Observability)** for the full Rehydration Protocol, algorithm, and `rehydrate_state.py` specification.

**Summary:** Truncate `balances` → replay ledger from T=0 (or `--from-id N`) → debit/credit per row → persist. Run manually or as a scheduled reconciliation job.

### Failure Modes & Cascades

| Failure                    | Behavior                                                                 |
|----------------------------|--------------------------------------------------------------------------|
| **Validator hangs**        | n8n times out (e.g., 30s). Retry uses same `proposal_id` → idempotent; no double-execution. |
| **Validator crashes mid-write** | Postgres transaction rolls back; no partial ledger entry. Retry safe.    |
| **Postgres unreachable**   | Validator returns 503. n8n retries; no state change until DB is back.    |
| **n8n workflow fails after execution** | Ledger already written. Linear ticket may not be created; consider idempotent Linear create. |
| **Ledger–Balances drift**  | `/health` fails; alert fires. Run `rehydrate_state.py` to repair.        |
| **Gemini API failure**     | n8n catches error; no proposal sent. Retry on next trigger.              |

**Cascade containment:** Validator does not block on external services (Linear, Discord). If those fail, log locally and continue. Ledger integrity is the only hard dependency.

---

## 6. Fault Tolerance

### Retries & Double-Spend Prevention

Production systems experience network flakes, timeouts, and partial failures. The Validator and n8n must handle retries without executing the same trade twice.

**Invariant:** A `proposal_id` may be submitted multiple times, but must result in **at most one** ledger entry.

### Idempotency Flow

```
n8n receives proposal from Gemini (with proposal_id)
                    │
                    ▼
         POST /execute { proposal_id, ... }
                    │
                    ▼
    ┌───────────────────────────────────┐
    │ Validator: SELECT 1 FROM ledger    │
    │ WHERE proposal_id = ?              │
    └───────────────────────────────────┘
                    │
         ┌──────────┴──────────┐
         │                     │
    EXISTS?                 NOT EXISTS?
         │                     │
         ▼                     ▼
  Return 200 OK          Validate proposal
  (already applied)      (balance, actors, constraints)
         │                     │
         │              ┌──────┴──────┐
         │              │             │
         │         VALID          INVALID
         │              │             │
         │              ▼             ▼
         │     BEGIN TRANSACTION   Return 422
         │     INSERT ledger       + error_code
         │     UPDATE balances     (no write)
         │     COMMIT
         │              │
         └──────────────┴──► Return 200 OK
```

### Implementation Rules

| Rule | Implementation |
|------|----------------|
| **Check before write** | Validator MUST query `ledger` for `proposal_id` before any validation or write. If found → return success immediately. |
| **Unique constraint** | `UNIQUE(proposal_id)` on `ledger` as a second line of defense; duplicate INSERT fails with constraint violation (treated as idempotent success). |
| **Atomic write** | Ledger INSERT and balances UPDATE in a single transaction. No partial state. |
| **n8n retry policy** | Use same `proposal_id` on retry. Do not generate a new proposal_id for the same logical trade. |
| **Client-generated ID** | `proposal_id` is generated by n8n or Gemini *before* the first `/execute` call, not by the Validator. |

### Double-Spend Scenarios (Handled)

| Scenario | Outcome |
|----------|---------|
| n8n times out after Validator commits | Retry sends same `proposal_id`; Validator finds existing row → 200 OK; no second debit/credit. |
| n8n retries due to network error | Same as above. |
| Two n8n instances run same workflow | First INSERT wins; second hits `UNIQUE` violation → treat as success (or return 200 with "already applied"). |
| Validator crashes after BEGIN, before COMMIT | Transaction rolls back. Retry safe; no ledger entry. |

---

## 7. Governance Workflow

### Dispute Branch: Rejected Trade → Linear Ticket

When a trade is **rejected** by the Validator, the system follows a defined governance path. This is the **Dispute** branch of the control flow.

### Dispute Flow Diagram

```
                    POST /execute
                          │
                          ▼
                   Validator validates
                          │
              ┌───────────┴───────────┐
              │                       │
         VALID ✓                 INVALID ✗
              │                       │
              ▼                       ▼
    ┌─────────────────┐     ┌─────────────────────────┐
    │ Atomic write    │     │ DISPUTE BRANCH          │
    │ to ledger       │     │                         │
    │ Return 200      │     │ 1. INSERT rejected_     │
    └─────────────────┘     │    proposals            │
                            │    (proposal_id,        │
                            │     validation_error_   │
                            │     code, proposal_json)│
                            │                         │
                            │ 2. Return 422 to n8n    │
                            │    { proposal_id,       │
                            │      error_code,        │
                            │      message }          │
                            └────────────┬────────────┘
                                         │
                                         ▼
                            ┌─────────────────────────┐
                            │ n8n receives 422        │
                            │                         │
                            │ 3. Create Linear ticket │
                            │    - Title: "Dispute:   │
                            │      {proposal_id}"     │
                            │    - Body: proposal_json│
                            │      + error_code       │
                            │    - Label: dispute     │
                            │                         │
                            │ 4. (Optional) Discord   │
                            │    alert                │
                            │                         │
                            │ 5. Workflow continues   │
                            │    (next proposal)      │
                            └─────────────────────────┘
```

### Linear Ticket Schema

| Field | Source | Purpose |
|-------|--------|---------|
| `title` | `"Dispute: {proposal_id}"` | Unique, searchable reference |
| `body` | `proposal_json` + `validation_error_code` + timestamp | Full audit context |
| `labels` | `dispute`, optional `{error_code}` | Filtering, triage |
| `custom.proposal_id` | From response | Link to `rejected_proposals` and ledger |

### Governance Resolution Loop

1. **Human board** reviews Linear tickets in a "Dispute" queue.
2. **Root cause** is classified (e.g., insufficient balance, bad actor, constraint too tight).
3. **Validator tuning** or **policy change** is applied if the rejection reveals a systemic issue.
4. **Ticket closed** with resolution notes; pattern may inform future Validator versions.

---

## 8. Observability

### Rehydration Protocol

The `balances` table is a **materialized cache** derived from the `ledger`. It can drift due to bugs, partial writes, or manual edits. The **Rehydration Protocol** reconstructs `balances` from the ledger, ensuring consistency.

#### Scope

- **Input:** `ledger` table (append-only, ordered by `id`).
- **Output:** Reconstructed `balances` table (actor_id, resource, amount).
- **Invariant:** For each (actor, resource), balance = sum of credits − sum of debits from ledger.

#### Algorithm

```
REHYDRATE(ledger, [from_id = 0]):
  1. Initialize balances = {}  # (actor_id, resource) → amount

  2. FOR each row in ledger WHERE id >= from_id ORDER BY id ASC:
       a. debit(actor_from, resource, amount)
       b. credit(actor_to, resource, amount)

  3. IF from_id = 0:
       TRUNCATE balances_table
     ELSE:
       # Checkpoint mode: only update actors touched by replayed rows
       (more complex; merge with existing balances for unaffected actors)

  4. INSERT OR UPSERT reconstructed balances into balances_table

  5. (Optional) Verify invariants:
       - Total supply per resource unchanged
       - No negative balances for actors that had initial allocation
```

#### Implementation: `rehydrate_state.py`

| Step | Action |
|------|--------|
| 1 | Parse args: `--from-id N` (optional), `--dry-run` (optional). |
| 2 | `SELECT * FROM ledger ORDER BY id [WHERE id >= N]`. |
| 3 | Build in-memory balance map: for each row, debit `actor_from`, credit `actor_to`. |
| 4 | If `--dry-run`: print diff vs current `balances`, exit. |
| 5 | `BEGIN`; truncate or upsert `balances`; `COMMIT`. |
| 6 | Log row count replayed, duration, checksum. |

#### Invocation

| Trigger | Command |
|---------|---------|
| Suspected drift | `python rehydrate_state.py` |
| Post-incident recovery | `python rehydrate_state.py --from-id 0` |
| Nightly reconciliation | Cron: `python rehydrate_state.py`; alert if `/health` still fails. |
| Large ledger (checkpoint) | `python rehydrate_state.py --from-id <last_known_good>` |

#### Verification

After rehydration, `GET /health` should pass the Ledger–Balances consistency check. If it fails, investigate ledger integrity (e.g., corrupted rows, constraint violations).

### Health & Drift Detection

`GET /health` performs Ledger–Balances consistency by:

1. Recomputing expected balances from ledger (subset or full, depending on performance).
2. Comparing to `balances` table.
3. Returning `200` if match, `503` if drift detected (with optional diff in response or logs).

---

## Component Responsibilities (Summary)

| Component   | Responsibility                                                                 |
|-------------|---------------------------------------------------------------------------------|
| **n8n**     | Triggers, AI calls, Linear integration; no state mutation; timeout handling     |
| **Postgres**| Ledger storage; balance cache; source of truth                                  |
| **Validator** | Validation logic; atomic writes; `/health`, `/execute`, `/world-state`; idempotency |
| **Streamlit** | Read-only dashboard; WorldState and Ledger visualization; manual override UI  |
