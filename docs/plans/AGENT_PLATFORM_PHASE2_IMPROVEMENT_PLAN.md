# Agent Platform Phase 2 Improvement Plan

## Goal

Move the Agent Platform from "implemented MVP" to "trusted operating layer" for the ExpaStore admin.

Phase 2 focuses on:

- deeper integration with real admin workflows
- stronger auditability and execution safety
- better UX across the whole agent vertical
- cleaner architecture for adding more agents and tools
- stronger automated validation

## Strategic Themes

### 1. Real Actions After Approval

Current state:

- approvals can be created and resolved
- approval resolution mostly changes status and audit state

Target:

- approved items should be able to trigger explicit, controlled business actions
- rejected items should leave a complete audit trail with decision context

Examples:

- queue a marketing follow-up workflow
- create a notification batch draft
- create a support task or review task for admin teams

### 2. Agent UX as a First-Class Admin Module

Current state:

- `Admin > Agentes` is now the main hub
- runs and approvals pages are functional but visually simpler

Target:

- a cohesive design language across agents, runs, approvals, and LLM settings
- stronger visual hierarchy and cross-linking
- better insight into capability, risk, and health

### 3. Expand the Agent Surface Safely

Current state:

- `operations_auditor`
- `admin_copilot`

Target:

- add narrowly scoped agents with explicit permissions and bounded tools
- keep each tool deterministic or approval-gated by default

### 4. Raise Engineering Confidence

Current state:

- structure is in place
- test coverage for agent-specific flows is still limited

Target:

- strong service, controller, and UI coverage
- explicit payload contracts
- fewer legacy leftovers

## Phase 2 Workstreams

## Workstream A: Approval Execution Pipeline

### Objective

Turn approvals into actionable, controlled workflows instead of terminal records.

### Tasks

1. Define approval action handlers by `actionType`.
2. Add a service layer for approval execution.
3. Execute action handlers only after approved status is persisted.
4. Persist execution result logs linked to the approval and run.
5. Surface execution outcome in run detail and approvals UI.

### Deliverables

- approval execution service
- action handler registry
- richer approval result payloads
- UI state for "approved and executed", "approved with failure", or equivalent safe statuses

## Workstream B: Agent Catalog and Capability Registry

### Objective

Make the platform ready for more agents without turning the UI and service layer into hardcoded branches.

### Tasks

1. Define richer metadata per agent:
   - category
   - owner
   - execution mode
   - supported tools
   - approval posture
2. Improve tool metadata:
   - label
   - purpose
   - access level
   - output shape summary
3. Expose these metadata cleanly from backend to frontend.
4. Render a capability matrix or richer card views in the hub.

### Deliverables

- richer agent definition model
- more descriptive tool registry
- UI improvements for discoverability

## Workstream C: New Safe Agents

### Objective

Add one or two narrowly scoped agents that prove the platform can scale.

### Candidate agents

- `support_triage_agent`
- `catalog_health_agent`
- `checkout_guard_agent`

### Selection rule

Only add agents that:

- depend on already trusted backend modules
- produce high admin value
- can stay read-only or approval-gated in first release

### Deliverables

- at least one additional analytical agent
- new tools with bounded read or proposal behavior

## Workstream D: UI and Design Consolidation

### Objective

Bring the full agent vertical up to the standard of the redesigned agent hub.

### Tasks

1. Redesign `AdminAgentRuns`.
2. Redesign `AdminAgentRunDetail`.
3. Redesign `AdminAgentApprovals`.
4. Make LLM settings visually consistent with the rest of the vertical.
5. Add shared visual primitives for:
   - status badges
   - risk badges
   - capability chips
   - timeline blocks
   - runtime state

### Deliverables

- unified visual system for agent pages
- better mobile layout consistency
- stronger cross-navigation within the vertical

## Workstream E: Code Cleanup and Modularization

### Objective

Reduce duplication and remove transitional pieces left after the MVP.

### Tasks

1. Keep the integrated copilot as the canonical UI entrypoint.
2. Extract shared copilot chat primitives if needed.
3. Review whether any admin routes still reference old copilot entry assumptions.
4. Document canonical agent UI entrypoints.

### Deliverables

- less dead code
- cleaner ownership of the copilot UI

## Workstream F: Tests and Contracts

### Objective

Make agent flows safe to evolve.

### Tasks

1. Add backend tests for:
   - list agents
   - trigger run
   - execute queued run
   - create approval
   - approve and reject flows
   - approval execution handlers
2. Add frontend tests for:
   - agent hub rendering
   - copilot interaction basics
   - runs filters
   - approvals resolution UX
3. Add contract documentation for:
   - run payloads
   - step payloads
   - approval payloads
   - copilot response shapes

### Deliverables

- higher confidence for refactors
- explicit platform contracts

## Suggested Execution Order

1. Approval execution pipeline
2. Tests around current flows
3. UI consolidation for runs and approvals
4. Capability registry improvements
5. Add one new safe agent
6. Final cleanup and documentation pass

## Definition of Done for Phase 2

- approvals can trigger controlled post-approval execution
- the agent vertical has a consistent, modern UX across all pages
- at least one new agent is added safely
- core agent flows are covered by automated tests
- legacy copilot entrypoints are removed or intentionally retained with documentation
- payload contracts are documented
- the platform is ready for incremental agent expansion without major redesign

## Immediate Next Task Recommendation

The best next concrete implementation step is:

1. build the approval execution pipeline

Why:

- it closes the biggest gap between demo flow and operational value
- it improves auditability
- it creates a stronger foundation for adding more agents later
