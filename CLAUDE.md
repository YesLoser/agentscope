# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this is

AgentScope 2.0 is a production-ready multi-agent framework (Python 3.11+, Apache-2.0). It ships as two layers:

1. **SDK layer** — `src/agentscope/`: the framework used to compose agents from building blocks (models, tools, memory, middleware, workspaces).
2. **Agent service** — `src/agentscope/app/`: a FastAPI backend (with a pre-built web UI in `examples/web_ui`) that turns agents into a multi-tenant, multi-session application.

## Development commands

The project uses `uv` for environments and `pre-commit` for formatting/linting. There is no Makefile or tox.

```bash
# One-time environment setup (Python 3.11+; .venv already exists in this repo)
uv venv
uv pip install -e ".[dev]"     # editable install incl. test + lint toolchain

# Enable git hooks (black/flake8/pylint/mypy/…)
pre-commit install

# Format and lint everything (what CI runs)
pre-commit run --all-files

# Run the full test suite (pytest + pytest-forked; no pytest.ini/conftest.py)
pytest tests

# Run a single test file / single test
pytest tests/agent_basic_test.py
pytest tests/agent_basic_test.py::TestAgent::test_reply
pytest tests -k "structured_output"
```

Tests that depend on an optional extra (Redis, Ollama, S3, Docker, …) must skip cleanly when the extra isn't installed.

Run the sample apps:

```bash
python examples/agent_service/main.py          # agent-service backend
# separate terminal:
cd examples/web_ui && pnpm install && pnpm dev   # web UI
```

## Architecture

The SDK is deliberately centered on **one core agent class** — `agentscope.agent.Agent` — which composes every other building block rather than a family of specialized agent classes. `RealtimeAgent` and `A2AAgent` are thin variants in `agentscope/agent/`.

Key building-block subpackages (each is a public API surface re-exported from its `__init__.py`):

- `model/` — chat/embedding/TTS/classifier/ASR. Chat models live per provider in subdirectories (`_openai_chat/`, `_anthropic/`, `_dashscope/`, `_gemini/`, …), each backed by a `_models/*.yaml` model card. A complete chat-model integration spans four pieces: a `credential` class, a `ChatModelBase` subclass, a model-card YAML, and two `formatter` variants (single-user `ChatFormatter` + `MultiAgentFormatter`).
- `tool/` — `Toolkit`, built-in coding tools (`Bash`/`Grep`/`Glob`/`Read`/`Write`/`Edit`), and MCP-server support.
- `message/` + `event/` — the `Msg` type and the event bus that streams reasoning/tool calls/multimodal content.
- `middleware/` — composable hooks across the agent loop (reply, reasoning, acting, model call, permission, context compression, system prompt).
- `formatter/` — converts `Msg` ↔ provider API request/response format.
- `workspace/` — isolated tool/code-execution backends (local, Docker, E2B, K8s, …).
- `permission/`, `memory/`, `rag/`, `sop/`, `pipeline/`, `realtime/`, `console/`, `tui/` — HITL, long-term memory, retrieval, SOP orchestration, pipelines, voice, and terminal UIs.

The `app/` subpackage is the agent service: `_service`/`_router`/`_manager` hold the FastAPI app, routing, and lifecycle; `storage/` provides SQL (Alembic under `_sql/_alembic/`), Redis, and S3 backends; `channel/` integrates IM platforms; `hub/` and `rag/` provide the MCP/skill hub and retrieval service; `workspace_manager/` wires workspaces into the app lifecycle.

`examples/` holds feature demos; production-style apps belong in the separate `agentscope-samples` repo. `learning/` at the repo root is local scratch (user learning scripts), not part of the package.

## Conventions that are enforced

- **Lazy imports (MUST).** Any dependency not listed in `[project.dependencies]` of `pyproject.toml` (i.e. anything from an optional extra — `gemini`, `service`, `storage-*`, `channel`, `workspace-*`, …) must be imported at point of use, not at module top level. New optional dependencies go into the matching extra, not the base list.
- **Underscore-prefixed internals (MUST).** Files under `src/agentscope` and internal classes/functions not meant for users are prefixed `_`; public names are exposed only via `__init__.py`. For lazy base-class imports, use a factory function that imports the base inside it.
- **Docstrings (MUST).** English, reStructuredText, with a strict `Args:` / `Returns:` template using backticked type annotations (see `.github/copilot-instructions.md` for the exact format).
- **Conventional Commits.** Commit messages and PR titles use `type(scope): subject`; scope must be lowercase (validated by CI on PRs against `main`).
- **Tests.** New features need unit tests. Assert against the **entire** data structure rather than individual fields; use `AnyString` / `AnyValue` from `tests/utils.py` for non-deterministic fields.

The full code-review checklist lives in `.github/copilot-instructions.md`; contribution workflow and module-specific guides (chat model, agent, workspace, examples) are in `CONTRIBUTING.md`.
