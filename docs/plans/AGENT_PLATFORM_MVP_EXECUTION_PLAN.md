# Agent Platform MVP Status

## Purpose

This document no longer represents a greenfield execution plan.

It now tracks the real status of the Agent Platform MVP already implemented inside ExpaStore, what is complete, what was expanded beyond the original MVP, and what still needs follow-through.

## Original MVP Goal

Build the first stable, scalable version of an agent control layer inside the ExpaStore admin panel.

## Current Status

### MVP outcome

The MVP is effectively implemented and operational at the structural level.

Completed pieces:

- admin permission `agents` exists across backend and frontend access control
- backend module `src/modules/agent-platform/` exists
- database migration for agent platform tables exists
- Sequelize models and associations are registered
- admin routes exist under `/admin/agents/*`
- default agents are bootstrapped automatically
- manual run triggering is implemented
- runs persist steps, logs, summaries, and approval requirements
- approvals can be approved or rejected
- admin UI exists for agents, runs, run detail, approvals, and LLM settings
- agent runtime now includes async queue processing
- conversational admin copilot exists as part of the same platform

### Beyond the original MVP

The implementation already goes beyond the initial scope in a few important ways:

- a second agent exists: `admin_copilot`
- centralized LLM settings and policy screens exist
- LLM orchestration and deterministic fallback exist
- the copilot is now integrated directly into `Admin > Agentes`
- the old floating entrypoint was removed from the global admin layout
- run detail and approvals now expose requester and resolver audit information

## Implemented Scope

### Backend

Key implemented elements:

- `agent.model.ts`
- `agent-tool.model.ts`
- `agent-run.model.ts`
- `agent-run-step.model.ts`
- `agent-approval.model.ts`
- `agent-log.model.ts`
- `agent-platform.service.ts`
- `agent-platform.controller.ts`
- `agent-platform.routes.ts`
- `agent-run-queue.service.ts`
- `llm-orchestrator.service.ts`

Behavior currently implemented:

- bootstrap of `operations_auditor`
- bootstrap of `admin_copilot`
- queued async run execution
- operational audit findings for orders, products, and dormant users
- proposal generation with approval records
- approval resolution flow
- copilot runtime inspection
- copilot conversational endpoint

### Frontend

Implemented admin surfaces:

- `Admin > Agentes`
- `Admin > Ejecuciones IA`
- `Admin > Detalle de ejecución`
- `Admin > Aprobaciones IA`
- `Admin > LLM Settings`

Current UX state:

- agent hub has been redesigned
- copilot is embedded into the agent page
- agent catalog and copilot now share one integrated experience
- execution and approval views are usable and auditable

## Data Model Status

Implemented tables:

- `admin_agents`
- `admin_agent_tools`
- `admin_agent_runs`
- `admin_agent_run_steps`
- `admin_agent_approvals`
- `admin_agent_logs`

The original design rules are still valid and are being followed:

- agent telemetry is isolated from generic audit logs
- runs and approvals relate to admin users
- JSON payloads are persisted for steps, logs, and approval payloads
- status and creation date indexes exist for admin-facing queries

## Security Status

Implemented safeguards:

- dedicated `agents` admin permission
- routes remain under admin namespace
- approvals gate sensitive proposals
- no shell or arbitrary SQL execution from the agent platform
- LLM usage is controlled through policy settings
- deterministic fallback exists when LLM is unavailable or disallowed

## MVP Definition of Done Review

Original criteria and status:

- an admin can open the agent screens: done
- at least one agent exists: done
- an admin can trigger a run manually: done
- the run stores steps and logs: done
- the run detail page renders findings: done
- at least one proposal can be created as an approval item: done
- approvals can be approved or rejected: done
- everything is auditable: partially done
- the structure is ready for more agents and tools: done

### Why "everything is auditable" is only partially done

The module already stores strong telemetry, but there is still room to tighten audit completeness:

- approval decisions do not yet trigger real controlled business actions
- agent-focused tests are still thin compared to the rest of the platform

## Current Gaps

These are the most relevant remaining gaps after the MVP:

### Product gaps

- approval records mostly end at state transition, not action execution
- only one analytical agent exists
- no richer tool registry or capability discovery UI yet
- no agent-specific dashboards or health monitoring

### Engineering gaps

- limited automated test coverage for agent platform flows
- conversational panel logic exists in a dedicated component but can still be modularized further
- there is no dedicated contract document for agent payload schemas

### UX gaps

- agent hub is improved, but runs and approvals screens still use a more conventional table/card style
- no unified "agent health" or "policy posture" overview yet
- no per-agent timeline or capability matrix

## Recommended Next Step

The MVP should now be treated as complete enough to enter a structured Phase 2.

Next planning document:

- [`AGENT_PLATFORM_PHASE2_IMPROVEMENT_PLAN.md`](./AGENT_PLATFORM_PHASE2_IMPROVEMENT_PLAN.md)
