# TASKS.md — Spec-Driven Development

> **Rule:** Before asking Cursor to code anything, break the ticket down into GIVEN-WHEN-THEN scenarios. Tests come first; code comes after tests pass.

---

## The Loop

```
┌─────────────────────────────────────────────────────────────────┐
│  1. INSTRUCTION: You provide the requirement                    │
│     (e.g., "Add a resource trade validator")                    │
└─────────────────────────────────────────────────────────────────┘
                                │
                                ▼
┌─────────────────────────────────────────────────────────────────┐
│  2. AI RESPONSE: Cursor generates the TEST first, not the code  │
│     - GIVEN-WHEN-THEN scenarios in /tests                       │
│     - Test validates the logic against WorldState constraints   │
└─────────────────────────────────────────────────────────────────┘
                                │
                                ▼
┌─────────────────────────────────────────────────────────────────┐
│  3. HUMAN/AGENT VERIFICATION: Run the test                      │
│     - If it FAILS → AI must fix until the test passes           │
│     - If it PASSES → proceed                                    │
└─────────────────────────────────────────────────────────────────┘
                                │
                                ▼
┌─────────────────────────────────────────────────────────────────┐
│  4. FINALIZE: Only after the test passes is the code "correct"  │
└─────────────────────────────────────────────────────────────────┘
```

---

## GIVEN-WHEN-THEN Format

For each ticket, define scenarios in this format:

| Keyword | Meaning | Example |
|---------|---------|---------|
| **GIVEN** | Preconditions, current state | Actor A has 100 energy; Actor B has 50 energy |
| **WHEN** | The action or trigger | A proposal to transfer 30 energy from A to B is submitted |
| **THEN** | Expected outcome | Ledger records the transfer; A has 70, B has 80 |

---

## Ticket Template

When opening a new task, copy this template and fill it out **before** writing code:

```markdown
## Ticket: [Title]

**Requirement:** [One-line description]

### Scenarios

#### Scenario 1: [Happy path / Primary case]
- **GIVEN** [state]
- **WHEN** [action]
- **THEN** [expected outcome]

#### Scenario 2: [Edge case / Rejection]
- **GIVEN** [state]
- **WHEN** [action]
- **THEN** [expected outcome]

#### Scenario 3: [Idempotency / Retry]
- **GIVEN** [state]
- **WHEN** [action]
- **THEN** [expected outcome]

### Test File
- Path: `/tests/[feature]_test.py` (or equivalent)
- Run: `pytest tests/[feature]_test.py -v`
```

---

## Example: Resource Trade Validator

**Requirement:** Add a resource trade validator that accepts proposals and applies them atomically.

### Scenarios

| # | GIVEN | WHEN | THEN |
|---|-------|------|------|
| 1 | Actor A has 100 energy; Actor B has 50 energy; proposal_id = uuid-123 | Valid proposal: transfer 30 energy A→B | Ledger has 1 row; balances: A=70, B=80; 200 OK |
| 2 | Actor A has 10 energy | Proposal: transfer 30 energy A→B | Rejection; no ledger write; 422 INSUFFICIENT_BALANCE |
| 3 | proposal_id = uuid-123 already in ledger | Same proposal retried | 200 OK; no duplicate ledger entry; balances unchanged |
| 4 | Actor C does not exist | Proposal: transfer from C to B | Rejection; 422 INVALID_ACTOR |

### Test File
- Path: `/tests/validator_trade_test.py`
- Run: `pytest tests/validator_trade_test.py -v`

---

## QA Gate

Every new feature **must** include a `/tests` file that validates the logic against:

- WorldState constraints (min_balance, max_transfer, etc.)
- Ledger invariants (idempotency, atomicity)
- Rejection cases (insufficient balance, invalid actor, constraint violation)
