# Synthesis: Evidence-Governed Cognitive Runtime

## Architecture Name

EGCR: Evidence-Governed Cognitive Runtime.

EGCR is a concrete combined architecture for Cognitive-OS that extracts the strongest repeated ideas from the 10 collected model outputs. It does not assume AGI has been achieved. It defines a local-first control plane where model calls are replaceable proposal generators and durable runtime services own memory, evidence, policy, execution, and recovery.

## Core Principle

The LLM context window is an L1 cache, not the agent's mind.

The durable mind of Cognitive-OS is the event-sourced cognitive state machine: goals, evidence, hypotheses, world/self state, policies, procedures, leases, action attempts, verifier decisions, outcomes, and recovery records. Models read a scoped projection of that state, propose changes, and return structured candidates. The runtime decides what becomes real.

## Runtime Services

### 1. Event Ledger

The event ledger is the source of truth. Every material transition is appended as a typed event:

- `goal.created`
- `evidence.observed`
- `hypothesis.proposed`
- `memory.retrieved`
- `plan.proposed`
- `policy.checked`
- `lease.granted`
- `action.requested`
- `verifier.passed` or `verifier.failed`
- `tool.executed`
- `outcome.observed`
- `reflection.recorded`
- `memory.promoted`
- `patch.proposed`
- `patch.promoted` or `patch.rolled_back`

Each event should include: timestamp, actor, input hash, output hash, policy version, linked evidence IDs, runtime mode, and replay instructions where applicable.

### 2. State Store

The state store is a materialized view over the ledger, not an independent truth source. It holds current active goals, open hypotheses, entity state, tool leases, memory indexes, and recovery checkpoints.

A practical first implementation can use SQLite for events and materialized tables. Later versions can split hot state into a key-value store and vector index, but the first prototype should optimize for inspectability.

### 3. Memory Manager

Memory has four layers:

- Working memory: bounded active task state with explicit TTL.
- Episodic memory: append-only trace of actions, observations, and outcomes.
- Semantic memory: stable facts and entity relationships promoted from evidence.
- Procedural memory: reusable skills, plans, tool recipes, and verifier templates.

Memory promotion is not automatic summarization. It is a governed transition that requires evidence links, contradiction checks, and utility scoring.

### 4. Planner

The planner receives a scoped state projection and emits a typed plan:

```json
{
  "goal_id": "...",
  "steps": [
    {
      "action": "adapter.method",
      "arguments": {},
      "required_capability": "filesystem.read",
      "preconditions": [],
      "expected_observations": [],
      "rollback_hint": "..."
    }
  ],
  "uncertainty": {
    "unknowns": [],
    "confidence": 0.0,
    "escalation_threshold_hit": false
  }
}
```

The planner does not execute. It proposes.

### 5. Policy And Lease Engine

The policy engine decides whether a proposed action is allowed in the current runtime mode. A lease is a temporary capability grant with scope, expiration, actor, and reason.

Policy decisions should themselves be events. That lets future planning retrieve governance precedent: why similar actions were allowed, denied, or escalated.

### 6. Verifier Gate

Before execution, each step passes verifier gates:

1. Type and schema validation.
2. Capability lease validation.
3. Precondition validation against state store.
4. Risk classification.
5. Dry-run or simulation when available.
6. Human approval check when required.

After execution, postcondition verifiers compare expected and observed outcomes.

### 7. Action Executor

The executor is the only service that mutates the outside world. It calls local-machine adapters or tool adapters inside managed boundaries. Every side effect is recorded before and after execution.

The executor should support three action classes:

- Read-only actions.
- Reversible writes with rollback hints.
- High-impact writes requiring human approval or stronger leases.

### 8. Reflection And Learning

Reflection compares predicted outcomes with observed outcomes and writes discrepancy records. Learning consumes discrepancy records offline and proposes patches to:

- retrieval scoring,
- procedures,
- prompts,
- route policies,
- verifier predicates,
- memory consolidation rules,
- and documentation.

No learning patch is promoted without a benchmark and regression gate. Rollback must be a normal path, not an emergency path.

## Multi-Agent Design

EGCR should start with five roles:

- Planner: proposes plans.
- Critic: identifies missing evidence and risk.
- Verifier: checks policy and preconditions.
- Executor: performs approved tool calls.
- Curator: consolidates memory and proposes procedural updates.

Agents communicate through typed events, not hidden shared chat. Routine work does not require voting. Consensus is reserved for high-impact decisions where independent plans or critiques materially reduce risk.

## Runtime Modes

The model outputs converge on explicit modes. Cognitive-OS already has runtime-mode language, so EGCR should preserve it:

- `SLEEP`: consolidation, cleanup, memory scoring.
- `IDLE`: no active external mutation; safe diagnostics.
- `ROUTINE_RUN`: bounded routine task execution.
- `DEEP_THINK`: extended planning and hypothesis work.
- `ACTING`: approved tool execution.
- `WAITING_HUMAN`: blocked by approval, ambiguity, or policy.
- `DEGRADED_RECOVERY`: replay, rollback, and state repair.

Mode transitions should be ledger events and policy inputs.

## 90-Day Prototype Plan

### Days 1-30: Ledger And State Spine

- Implement append-only event schema.
- Materialize current state from events.
- Add evidence IDs to goals, hypotheses, plans, and actions.
- Add replay tests for crash recovery.
- Expose a minimal operator report showing active goal, mode, leases, last verifier decisions, and latest outcome deltas.

### Days 31-60: Governed Action Path

- Define capability leases.
- Add policy checks for read/write/network/credential/sync-back classes.
- Route local-machine actions through verifier gates.
- Add dry-run support where possible.
- Record side-effect ledger events before and after execution.
- Add red-team tests for bypass attempts and stale leases.

### Days 61-90: Memory And Learning Loop

- Add working, episodic, semantic, and procedural memory records.
- Implement retrieval with provenance and freshness metadata.
- Add memory promotion workflow.
- Add reflection records for predicted vs observed outcomes.
- Generate offline learning candidates from successful and failed traces.
- Promote only through benchmark gates and rollbackable patches.

## Highest-Risk Assumptions

1. Verifier predicates can cover enough real actions to be useful.
2. Event replay remains fast enough as the ledger grows.
3. Memory promotion can avoid polluting semantic memory with weak model inferences.
4. Operators will tolerate approval friction for high-impact actions.
5. Local sandboxing remains reliable across developer machines.

## Implementation Decision Record

Recommended first implementation choices:

- SQLite event ledger and materialized state tables.
- JSON schema for plan/action/event payloads.
- Local vector index only after typed provenance fields exist.
- Capability leases stored as first-class events.
- Managed VM or sandbox boundary for writes.
- Shadow-mode learning patches before promotion.
- Runtime-mode transition tests in CI.

## What To Avoid

- Do not let raw model output become memory without provenance.
- Do not let the planner execute tools directly.
- Do not optimize for multi-agent debate before deterministic verifier gates exist.
- Do not treat benchmark success as permission for unbounded online learning.
- Do not hide policy decisions in logs that cannot be retrieved by future planning.

## Final Combined Insight

The strongest architecture is not a bigger prompt or a bigger model. It is an auditable cognitive transaction system: every belief, plan, permission, action, and learning update has provenance, a lifecycle, and a rollback story. That is the practical bridge from today's agent runtime to an AGI-direction control plane that can improve without losing governance.
