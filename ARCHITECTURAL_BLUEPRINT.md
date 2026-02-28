# Grid Governance Engine — Architectural Blueprint

## Tech Stack

| Layer         | Technology | Role                                                     |
|---------------|------------|----------------------------------------------------------|
| **Workflow**  | n8n        | Orchestrates triggers, AI calls, validation, and logging |
| **Database**  | Postgres   | Immutable ledger, source of truth for state              |
| **AI**        | Gemini     | Generates proposals from context and constraints         |
| **Dashboard** | Streamlit  | Observability, monitoring, and manual overrides          |

---

## Data Architecture

### WorldState (JSON)

A snapshot of the current simulation state, derived from the ledger for context and display:

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

- **Purpose**: Input to the AI and display in the dashboard.
- **Source**: Computed by replaying the ledger or aggregating from a materialized view.
- **Ephemeral**: Not the source of truth; the ledger is.

### Ledger (SQL Schema)

The ledger stores immutable, ordered records of all executed transactions.

| Table         | Purpose                                                    |
|---------------|------------------------------------------------------------|
| `ledger`      | Append-only log of applied transactions                    |
| `actors`      | Registry of participants (nodes, agents)                   |
| `resources`   | Resource types (energy, water, credits)                    |
| `balances`    | Current balance per actor per resource (derived/reconciled)|

**Ledger table (conceptual):**

| Column       | Type      | Description                                |
|-------------|-----------|--------------------------------------------|
| `id`        | bigint    | Monotonic sequence                         |
| `proposal_id` | uuid    | Unique ID for the proposal                 |
| `actor_from`| uuid      | Source actor                               |
| `actor_to`  | uuid      | Destination actor                          |
| `resource`  | text      | Resource type                              |
| `amount`    | numeric   | Transfer amount                            |
| `executed_at` | timestamptz | When applied                            |
| `metadata`  | jsonb     | Optional context (AI reasoning, etc.)      |

---

## Flow

### End-to-End: AI Proposal → Validation → Ledger → Linear

```
┌─────────────┐     ┌─────────────┐     ┌─────────────┐     ┌─────────────┐
│   Trigger   │────▶│  n8n Flow   │────▶│   Gemini    │────▶│  Proposal   │
│ (schedule,  │     │             │     │  (propose   │     │  JSON       │
│  webhook)   │     │             │     │   action)   │     │             │
└─────────────┘     └─────────────┘     └─────────────┘     └──────┬──────┘
                                                                   │
                                                                   ▼
┌─────────────┐     ┌─────────────┐     ┌─────────────┐     ┌─────────────┐
│   Linear    │◀────│   n8n       │◀────│  Postgres   │◀────│  Validation │
│  (log,      │     │  (orchestr.)│     │  (ledger,   │     │  (balance   │
│   ticket)   │     │             │     │   balances) │     │   checks)   │
└─────────────┘     └─────────────┘     └─────────────┘     └─────────────┘
```

1. **Trigger**: n8n runs on schedule or webhook (e.g., new event, manual run).
2. **Fetch WorldState**: n8n reads the current WorldState from the app (derived from Postgres).
3. **AI Proposal**: n8n sends WorldState + constraints to Gemini; Gemini returns a proposal (e.g., transfer action).
4. **Validation**: The app validates the proposal against Postgres (sufficient balance, valid actors, constraints).
5. **Execution**: If valid, the app applies the transaction atomically and appends to the ledger.
6. **Linear**: n8n creates or updates a Linear issue with the proposal, outcome, and link to the ledger entry.

---

## Component Responsibilities

| Component | Responsibility                                                |
|-----------|---------------------------------------------------------------|
| **n8n**   | Triggers, AI calls, Linear integration; no state mutation     |
| **Postgres** | Ledger storage; balance reconciliation; source of truth   |
| **App**   | Validation logic; atomic writes; WorldState computation       |
| **Streamlit** | Dashboard; view ledger, balances, proposals; manual ops  |
