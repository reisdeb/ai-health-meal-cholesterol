# API / Orchestrator Scaffold

## Responsibilities
- Authenticate requests and enforce idempotency.
- Run deterministic upload checks.
- Route accepted images through input guardrails, meal inference, output guardrails, and state update.
- Validate all model responses against shared schemas.
- Emit structured telemetry for latency, cost, safety, and quality monitoring.

## Recommended backend modules
- `routes/` — request handlers and API contracts
- `services/` — orchestration, OpenAI integration, state updates
- `schemas/` — shared request/response schemas
- `prompts/` — versioned prompt loaders
- `telemetry/` — logs, metrics, traces
- `eval_hooks/` — sampling and export utilities for offline review

## Core backend design rules
- Never update daily totals before output guardrails approve the inference.
- Store request IDs and idempotency keys to prevent double counting.
- Log prompt version, schema version, and model name for every inference request.
- Prefer safe fallback responses over retry loops when a request enters an uncertain or policy-sensitive state.
