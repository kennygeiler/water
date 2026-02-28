# Grid Governance Engine — Manifesto

## Vision

The Grid Governance Engine is an **autonomous, resource-negotiation simulation**. Agents, nodes, and participants negotiate over scarce resources—energy, water, credits—without a central orchestrator. Deals are proposed, validated, and executed by code. The system runs continuously, adapting to changing constraints and demand, while remaining auditable and recoverable.

---

## The "No-Crash" Rules

Six invariants keep the system safe and deterministic:

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
- **Discord**: Critical events (rejected proposals, circuit breaker) trigger alerts for real-time awareness.

### 4. Governance & Human-in-the-Loop (HITL)

Humans act as the Policy Layer and Emergency Arbitrators:

- **Policy Tuning**: Humans define global constraints (e.g., carbon tax, credit interest rates) within which agents negotiate.
- **Emergency Override**: A "Circuit Breaker" halts agentic trading when the grid hits critical levels; the system escalates to a Human-in-the-Loop checkpoint.
- **Audit Governance**: Human board members review Linear tickets from systemic conflicts (rejected proposals) and use insights to tune the Validator code.

### 5. Versioned Law

Every transaction is tagged with `validator_version`—the Git commit hash or version tag of the code that approved it. This ensures we can audit exactly what "Law" was in effect at the time of execution.

### 6. The Technical Loop

Every resource trade follows a strict 3-stage lifecycle:

1. **Generation**: Agent (Gemini) analyzes WorldState and proposes a `TradeJSON`.
2. **Validation**: Validator script (Python/JS) checks the proposal against hard constraints.
   - **If Valid**: Atomic transaction executes → Ledger updated → WorldState reconciled.
   - **If Invalid**: Transaction rejected → Conflict logged to Linear → Discord alert sent.
3. **Visualization**: Streamlit dashboard consumes WorldState and Ledger to render live resource flows.


---

## State Recovery

The system must be recoverable from any point in time.

### Rehydration via Postgres Ledger Replay

The ledger is the single source of truth. Recovery works by:

1. **Starting from a checkpoint** (optional, for performance) or from the beginning of the ledger.
2. **Replaying transactions** in order, applying each valid entry to reconstruct balances and derived state.
3. **Rejecting invalid entries** during replay (e.g., corrupted or out-of-order records) and flagging them for investigation.

**Materialized state**: For performance, the `world_state` table caches current balances. This cache must always be reconstructible by replaying the ledger from T=0. No cache is authoritative—the ledger is. Rehydration is deterministic.

---

## Summary

| Principle              | Meaning                                                                 |
|------------------------|-------------------------------------------------------------------------|
| **Code-as-Law**        | Rules live in code; LLMs propose, never decide.                         |
| **Proposal vs. Execution** | Validate first; execute only if valid; atomically.                  |
| **Observability**      | Ledger + logs + Linear + Discord = full traceability.                   |
| **Governance (HITL)**  | Humans set policy, circuit-break, and audit conflicts.                  |
| **Versioned Law**      | Every tx tagged with validator_version for auditability.                |
| **State Recovery**     | Rehydrate by replaying the ledger; world_state is a reconstructible cache. |
