# Phase 9 — Decorator (Requirements)

This is the **assignment**. Implement from these requirements. If you get stuck or want to check your design, use the [guided check](guided-check.md) (signatures and partial snippets — not a complete solution).

After you implement, answer the [questions](./questions.md).

## Purpose

Wrap the actuator **port** with **decorators** that apply policies (logging, max run time, and similar) and **persist each attempt** so the dashboard can show an execution log.

## Scope and naming rules

- Decorators implement the same port as the core actuator adapter. Callers use the port; they must not need to know how many wrappers are stacked.
- Log rows belong in `actuator_execution_log`, not only in application logs.
- Keep using `device_id`; location scope remains `location_id` where relevant.

## Outcome required at end of phase

- At least two decorators wrap `ActuatorPort` (logging + max run-time or equivalent).
- Every apply/attempt writes an execution row (success or fail, policies applied).
- UI lists recent executions from the API.
- A blocked max-runtime (or policy) attempt is visible as `success=false` with policy metadata.
- Scalar documents the executions endpoint.
- Pattern note at `docs/patterns/decorator.md`.

---

## Prerequisites

- Phase 8: `actuator_states` and domain transitions.
- Phase 5: `ActuatorPort` and `SimulationActuatorAdapter` in `infrastructure/adapters/actuators/simulation.py` (required—not optional; decorators wrap this innermost port).
- Sensor readings may arrive from the sampler or MQTT ingest, not only `POST /read`. Do not add a scheduler, MQTT client, or the device readings route in this phase.
- Commands stay on `device_id`. Do not add a second zone-assignment API.

---

## Required architecture for this phase

- **Domain:** `ActuatorPort.apply(...)`; decorator classes implementing the same port; innermost component is the real/sim adapter.
- **Application:** compose the chain; map execution rows to DTOs.
- **Infrastructure:** `actuator_execution_log` ORM + repository; innermost adapter.
- **API:** list executions; apply still goes through decorated port (may be invoked from a small demo route until Phase 10).
- **Frontend:** execution log table/list.

---

## Suggested file locations

Paths are relative to **`yourpath\project\`**.

```text
yourpath\project\
├── backend/src/
│   ├── domain/actuators/ports.py
│   ├── infrastructure/adapters/actuators/simulation.py
│   ├── infrastructure/adapters/actuators/logging_decorator.py
│   ├── infrastructure/adapters/actuators/max_runtime_decorator.py
│   ├── infrastructure/persistence/models.py                 # ActuatorExecutionLogRow
│   ├── infrastructure/persistence/execution_log_repository.py
│   └── interfaces/api/actuators.py
└── frontend/src/.../ExecutionLog.tsx
```

## Type hints

`ActuatorPort.apply(device_id: UUID, command: str, payload: dict) -> None`.

ORM `actuator_execution_log`:

| Column           | SQL hint                                                |
| ---------------- | ------------------------------------------------------- |
| `id`             | UUID PK                                                 |
| `device_id`      | UUID FK → `devices`                                     |
| `success`        | BOOLEAN                                                 |
| `policy_applied` | JSONB — list of `str` (e.g. `["maxRunTime","logging"]`) |
| `message`        | TEXT, nullable                                          |
| `created_at`     | timestamptz                                             |

Execution DTO JSON: `{ id, device_id, success: bool, policy_applied: string[], message: string \| null, created_at }`.

---

## Step-by-step implementation requirements

## Step 1 — Database

`actuator_execution_log`:

| Column           | Intent                    |
| ---------------- | ------------------------- |
| Identifier       | PK                        |
| `device_id`      | FK → `devices`            |
| `success`        | Boolean                   |
| `policy_applied` | JSON list of policy names |
| `message`        | Optional text             |
| `created_at`     | Timestamptz               |

Autogenerate and apply.

Acceptance criteria:

- Table exists; FK valid.

## Step 2 — Decorator stack

Implement:

- Logging decorator — records attempt (may write via a callback/repository injected into the decorator or an application service around `apply`). Logging is outermost.
- Max-run-time decorator — rejects apply when the persisted state is `running` and `actuator_states.updated_at` is at least 30 minutes before the clock. Do not call the inner adapter. Use `updated_at` as that clock’s comparison point.

The command string is forwarded to `apply` and is not validated in this phase. An unknown command still succeeds unless max-run-time blocks it. Phase 10 consults `allowed_commands`.

A completed attempt stores `success: true`, `policy_applied: ["logging"]`, and `message: null`. A blocked attempt stores `success: false`, `policy_applied: ["maxRunTime", "logging"]`, and message `Max run time exceeded`.

Composition order is logging, then max-run-time, then `SimulationActuatorAdapter`.

Acceptance criteria:

- Decorators share `ActuatorPort`.
- Blocked policy does not call the inner adapter (or inner is not reached — prove in a test).
- Failed policy still creates a log row with `success=false` and policy name in JSON.

## Step 3 — API and UI

- `GET /api/actuators/executions?limit=20` (or equivalent) from the table.
- Optional apply/demo endpoint until Command phase.
- **Execution log** UI in a sensible dashboard section (controls or a dedicated block). Tailwind; empty state. Instruction text explains that any command succeeds until the actuator has been `running` for 30 minutes, that a success row has an empty message and policy `logging`, and that a blocked row shows `Max run time exceeded` with policies `maxRunTime` and `logging`.

Acceptance criteria:

- Scalar lists the endpoint.
- Blocked attempt appears in the UI after refresh.

## Step 4 — Tests and docs

- Decorator unit tests (max-run-time blocks; logging always records).
- Assert execution log insert.
- `docs/patterns/decorator.md`: wrapping, order, vs inheritance.

---

## Definition of done

- Decorated port in use.
- Execution log persisted and shown.
- Policy failure visible as unsuccessful row.
- Tests and pattern doc complete.

---

## Common pitfalls to avoid

- Changing `ActuatorPort` method names in each decorator.
- Logging only with `print` / logger and skipping the table.
- Putting policy `if` checks in the router instead of decorator classes.

---

## Handoff to next phases

- Phase 10 (Command) will execute through this decorated port and add `command_log`.

→ [Phase 10 — Command (requirements)](../phase-10/requirements.md) · [guided check](../phase-10/guided-check.md)
