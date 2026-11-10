# Phase 10 — Command (Guided check)

Complete the [requirements](requirements.md) first. Use this document if you are stuck or to check signatures and JSON. Snippets are **not** a complete solution.

## Pattern

**Command** — encapsulate requests; persist audit.

---

## Target layout

```text
domain/commands/
  base.py               # Command.execute(port)
  pump.py               # StartPumpCommand, StopPumpCommand
  light.py              # SetLightLevelCommand
  invoker.py            # registry + CommandInvoker.run
application/commands/
  dto.py
  service.py            # guard, state update, command_log
infrastructure/persistence/
  models.py             # CommandLogRow
  command_log_repository.py
interfaces/api/commands.py
frontend/src/components/commands/CommandPanel.tsx   # #controls
```

---

## Step 1 — Database

```text
command_log(
  id, device_id FK, command_type, payload JSONB,
  status,               # success | failed | rejected
  error_message, created_at
)
```

```powershell
alembic revision --autogenerate -m "command_log"
alembic upgrade head
```

**Check:** table exists; rejected commands still insert a row (`status=rejected`).

---

## Step 2 — Command objects

```python
class Command(ABC):
    command_type: str
    target_state: str | None
    def execute(self, port: ActuatorPort) -> None: ...

def command_for(command_type: str, device_id: UUID, payload: dict) -> Command: ...

class CommandInvoker:
    def run(self, command: Command, port: ActuatorPort) -> None:
        command.execute(port)
```

Execute path (application service, not the router):

1. **Guard** — load actuator state; if `command_type` is not in `allowed_commands()` → persist `rejected`, return **409** with the log row, **stop**. No `apply`, no state write.
2. **Port** — `execute(port)` calls the Phase 9 stack from `build_actuator_port`.
3. **State** — after a completed apply, `ActuatorContext.transition` only when `target_state` differs (start → `running`, stop → `idle`). `set_light_level` on an already `running` light does not transition.
4. **Log** — `status=success`, or `failed` with `error_message` when the decorator raises `Max run time exceeded`. That failure leaves `actuator_states` unchanged and leaves the execution-log row already written.

Unknown `command_type` → **400** and no `command_log` row. The invoker does not contain `if command_type == "start_pump":`. Map the type string to a class, then `execute(port)`.

**Check:** illegal-state submit writes `rejected` and does **not** change `actuator_states`; legal start changes state to running.

---

## Step 3 — API

`POST /api/commands`

```json
{ "command_type": "start_pump", "device_id": "<uuid>", "payload": {} }
```

Responses: **200** with the log row for `success` and `failed`; **409** with the log row for `rejected`; **404** missing device or a device that is not an actuator; **400** unknown type and no row.

`GET /api/commands?limit=50` — newest first. `limit` is 1–200. Each row is `{ id, command_type, device_id, payload, status, error_message, created_at }`.

**Check:** Scalar shows both routes; rejected POST still appears on GET.

---

## Step 4 — Frontend

`#controls`: show `start_pump`, `stop_pump`, and `set_light_level`. Disable a button when that name is absent from `allowed_commands`. `set_light_level` sends `{ "level": 1 }` unless the field is changed. After a click, reload the actuators and the history table. Confirm dialog optional until Phase 13.

**Check:** button disabled when command not allowed; after a successful start, stop becomes available (via refreshed `allowed_commands`) and the history shows `success`.

---

## Step 5 — Tests and docs

- `test_illegal_command_rejected_logged`
- `test_start_pump_transitions_to_running`
- `docs/patterns/command.md`: object vs function, invoker, audit

---

## Phase 10 completion checklist

- [ ] Command classes + invoker
- [ ] State-aware reject + persist
- [ ] Decorated port used on execute
- [ ] UI + tests + pattern doc

---

## Troubleshooting

| Symptom                         | Fix                                        |
| ------------------------------- | ------------------------------------------ |
| Router implements start/stop    | Command object + invoker                   |
| Rejected command not in history | Insert `command_log` before returning 409  |
| State changes on reject         | Guard first; save state only after success |
| Decorators skipped              | Call `ActuatorPort`, not a raw adapter     |

---

## Next

→ [Phase 11 — Observer (requirements)](../phase-11/requirements.md) · [guided check](../phase-11/guided-check.md)
