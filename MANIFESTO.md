# Grid Governance Engine — Manifesto

## Vision

The Grid Governance Engine is an **autonomous, resource-negotiation simulation**. Agents, nodes, and participants negotiate over scarce resources—energy, water, credits—without a central orchestrator. Deals are proposed, validated, and executed by code. The system runs continuously, adapting to changing constraints and demand, while remaining auditable and recoverable.

---

## The "No-Crash" Rules

Three invariants keep the system safe and deterministic:

### 1. Code-as-Law

The rules of the simulation are encoded in code, not in natural language prompts. LLMs propose *intent*; the engine enforces *rules*. Balances, constraints, and validity checks live in Python/JavaScript, never in the model. The LLM suggests *what* to do; the code decides *whether* it is allowed and *how* to execute it.

### 2. Proposal vs. Execution

Every state change follows two phases:

- **Proposal**: The AI generates a candidate action (e.g., "Transfer 10 units from A to B"). This is a suggestion, not a commitment.
- **Execution**: The engine validates the proposal against the current ledger, applies it atomically, and appends an immutable record. If validation fails, nothing is written.

No proposal reaches the ledger until it passes all checks. Proposals that violate constraints are rejected and logged for observability.

### 3. Observability

All proposals, executions, and rejections are observable:

- **Ledger**: Every applied transaction is stored in Postgres with a deterministic order and full context.
- **Logs**: Workflow runs, validation failures, and AI outputs are traceable through n8n and application logs.
- **Linear**: Decisions and incidents are tracked in Linear for governance and incident response.

---

## State Recovery

The system must be recoverable from any point in time.

### Rehydration via Postgres Ledger Replay

The current world state is derived from the ledger, not stored as a mutable blob. Recovery works by:

1. **Starting from a known checkpoint** (optional, for performance) or from the beginning of the ledger.
2. **Replaying transactions** in order, applying each valid entry to reconstruct balances and derived state.
3. **Rejecting invalid entries** during replay (e.g., corrupted or out-of-order records) and flagging them for investigation.

The ledger is the single source of truth. Any component can rehydrate state by replaying it. No separate "current state" database is required—the ledger *is* the state, and rehydration is deterministic.

---

## Summary

| Principle        | Meaning                                                      |
|------------------|--------------------------------------------------------------|
| **Code-as-Law**  | Rules live in code; LLMs propose, never decide.              |
| **Proposal vs. Execution** | Validate first; execute only if valid; atomically.    |
| **Observability** | Ledger + logs + Linear = full traceability.                |
| **State Recovery** | Rehydrate by replaying the Postgres ledger.                 |
