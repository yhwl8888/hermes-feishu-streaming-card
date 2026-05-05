# Agents Guide — hermes-feishu-streaming-card V3.2.1

## Project identity

A **sidecar-only** plugin that adds Feishu streaming card messages to the
[Hermes Agent](https://github.com/NousResearch/hermes-agent) gateway.  This
project does **not** contain Hermes itself — it patches Hermes's
`gateway/run.py` at install time and runs a separate HTTP sidecar process.

The active source is `hermes_feishu_card/`.  Everything in `legacy/` is
archived V2 code and is NOT active runtime.  Do not edit or rely on it.

## Repo layout (only things you'll touch)

```
hermes_feishu_card/          # active package — pip-installable
  cli.py                     # entry point: hermes-feishu-card
  server.py                  # aiohttp sidecar server
  hook_runtime.py            # code injected INTO Hermes (locals() extraction)
  runner.py                  # sidecar main() — web.run_app
  feishu_client.py           # Feishu IM/tenant-token HTTP client
  bots.py                    # multi-bot BotRegistry + route resolver
  events.py                  # SidecarEvent dataclass + validation
  session.py                 # CardSession — per-message state machine
  render.py                  # card JSON builder
  text.py                    # <think> tag normalizer, flush heuristic
  metrics.py                 # SidecarMetrics counters
  process.py                 # PID file, start/stop/health, os.killpg
  config.py                  # YAML config loader + env overrides
  install/
    detect.py                # AST visitor to verify Hermes version/structure
    patcher.py               # AST-level patch injector (5 marker blocks)
    manifest.py              # SHA256 file hashing
tests/                       # 398 pytest tests (asyncio_mode = auto)
legacy/                      # V2 archive — DO NOT EDIT
```

## Development commands

```bash
# Install in editable mode with test deps
python -m pip install -e ".[test]"

# Run ALL tests (CI does exactly this)
python -m pytest -q

# Run a single test file
python -m pytest tests/unit/test_config.py -q

# Run a single test
python -m pytest tests/unit/test_config.py::test_load_defaults -q

# Run integration tests only
python -m pytest tests/integration/ -q

# Run unit tests only
python -m pytest tests/unit/ -q
```

**No linting, typechecking, or formatter is configured.** There is no `mypy`,
`ruff`, or `black` in `pyproject.toml`.  The CI workflow only runs `pytest -q`.

## Environment setup gotchas

- **The sidecar needs Feishu credentials to do real API calls.**  For tests,
  the `tests/conftest.py` fixtures mock these out — no real credentials needed.
- **`FEISHU_APP_ID` / `FEISHU_APP_SECRET`** env vars are read by `config.py` at
  runtime and override whatever is in the YAML config.
- **`python -m pytest` must be invoked from the repo root** (`pyproject.toml`
  sets `pythonpath = ["."]` so `hermes_feishu_card` is importable).
- **The sidecar depends on a running Hermes Gateway** and the `hermes_feishu_card`
  import must be resolvable inside Hermes's venv.  During development, edit the
  patcher code only; the install step copies the hook into Hermes.

## Architecture rules an agent MUST follow

### Never edit Hermes source directly

All modifications to Hermes's `gateway/run.py` go through the patcher
(`install/patcher.py`).  The patcher:
1. Parses `run.py` as an **AST** — no regex guesses
2. Finds `_handle_message_with_agent` and verifies it contains
   `hooks.emit("agent:end", ...)` via an AST visitor
3. Inserts 5 marker-wrapped code blocks
4. Each block is self-verifying: if markers are missing or corrupted → raises
   `ValueError("corrupt patch markers")`
5. Creates backups with SHA256 manifests; refuses to restore/overwrite if
   `run.py` has changed since install

### The hook runtime is injected, not imported directly

`hook_runtime.py` functions (`emit_from_hermes_locals`,
`emit_from_hermes_locals_threadsafe`, `emit_from_hermes_locals_async`) are
**called from injected code inside Hermes's process**.  They extract data from
the caller's `locals()` dictionary — they do NOT receive typed arguments (except
for `emit_from_hermes_locals_async`'s completion payload which is constructed by
the injected code).

This means:
- Field names in `build_event()` / `_event_data()` **must match the variable
  names in Hermes's `run.py`** (e.g. `source`, `event`, `response`,
  `agent_result`, `_response_time`, `event_message_id`, `_loop_for_step`,
  `_run_still_current`).
- When Hermes renames variables, the hook breaks silently.  The `detect.py`
  AST visitor is the safety net.

### Sidecar state is ephemeral and per-message

- `CardSession` objects live in `request.app[SESSIONS_KEY]` (a plain dict
  keyed by `message_id`).
- The sidecar process is stateless across restarts — there is **no persistence**.
- Each `message_id` has its own `asyncio.Lock` for serialization.
- Terminal events (`message.completed` / `message.failed`) are retried up to
  3 times with exponential backoff (1s, 2s, 4s) if the initial update fails.

### Message ID fallback is complex for a reason

Hermes may not always provide an explicit `message_id` to the hook.  The
`hook_runtime.py` fallback system (`_fallback_message_id`,
`_ACTIVE_FALLBACK_MESSAGE_IDS`, `_CURRENT_FALLBACK_KEYS`,
`created_at_lifecycle_token`) exists to generate deterministic message IDs
when Hermes doesn't, and to de-duplicate across parallel sessions.  Do not
simplify this unless you fully understand the lifecycle race conditions.

## Testing conventions

- **Async tests** — `asyncio_mode = "auto"` in `pyproject.toml`, so just use
  `async def test_...` directly.
- **Mock Feishu** — `tests/integration/test_server.py` spins up a real aiohttp
  test server (not mocked) and sends events against it.
- **Hermes smoke tests** — `tests/integration/test_hook_runtime_integration.py`
  and `test_cli_process.py` require a real or fixture Hermes checkout.
- Tests are organized as `tests/unit/` and `tests/integration/` plus one
  legacy `tests/integration_test.py`.

## 语言约定

所有思考输出、注释、文档和沟通均使用**中文**。以下情况保持英文：

- 字段名、变量名、函数名（如 `chat_id`、`message_id`、`build_event()`）
- 工具名称（如 `web_search`、`file_read`）
- 专用名词（如 `Hermes`、`CardKit`、`aiohttp`、`Sidecar`）
- 特定名词和通俗含义的单词（如 `bot`、`token`、`patch`、`streaming`）

## Key constraints

- Hermes Gateway must be **≥ v2026.4.23** (checked by `detect_hermes()`).
- `gateway/run.py` must not be a symlink.
- The `hermes_feishu_card` package must be installed into Hermes's Python
  environment so the injected `from hermes_feishu_card.hook_runtime import ...`
  resolves.
- Non-terminal card updates are throttled to **minimum 2-second intervals**
  (`UPDATE_MIN_INTERVAL_SECONDS` in `server.py`).
- Feishu tenant token is cached for `expire - 60` seconds (default 7140s).
