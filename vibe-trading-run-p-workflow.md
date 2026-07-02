# How `vibe-trading run -p "..."` Processes Your Prompt

This document explains the execution workflow and code flow for:

```bash
vibe-trading run -p "Backtest a BTC-USDT 20/50 moving-average strategy for 2024, summarize return and drawdown, then export the report"
```

## 1) CLI Entry and Argument Routing

## Entry point
- Console script: `vibe-trading = "cli:main"` in `pyproject.toml`.
- Package export: `agent/cli/__init__.py` exports `main` from `agent/cli/main.py`.

## Interactive vs non-interactive routing
- `agent/cli/main.py::_is_interactive_invocation(...)` decides whether to use the new interactive loop.
- `run -p ...` is non-interactive, so `agent/cli/main.py::main(...)` delegates to `cli._legacy.main(raw_argv)`.

## Argument parsing for `run -p`
- Parser is built in `agent/cli/_legacy.py::_build_parser()`.
- Subcommand `run` defines:
  - `-p/--prompt -> args.run_prompt`
  - `-f/--prompt-file -> args.run_prompt_file`
  - `--max-iter -> args.run_max_iter`
  - `--json`, `--no-rich`
- Dispatch in `agent/cli/_legacy.py::main(...)`:
  - `if args.command == "run": return _handle_prompt_command(...)`

## Prompt resolution
- `agent/cli/_legacy.py::_handle_prompt_command(...)` calls `_read_prompt_source(...)`.
- Prompt source priority:
  1. CLI `-p` text
  2. `--prompt-file`
  3. stdin (if piped)
  4. interactive fallback prompt
- Non-empty prompt then calls `cmd_run(resolved_prompt, max_iter, ...)`.

## 2) Run Execution Pipeline

`cmd_run(...)` performs:
1. Preflight checks (`src.preflight.run_preflight`) unless `--json`.
2. Calls `_run_agent(...)`.
3. Prints result summary and metrics from `artifacts/metrics.csv`.

Inside `_run_agent(...)`:
- Builds tool registry: `src.tools.build_registry(...)`.
- Creates LLM client: `src.providers.chat.ChatLLM()`.
- Creates agent loop: `src.agent.loop.AgentLoop(...)`.
- Runs loop via `_run_with_graceful_cancel(...)` -> `agent.run(...)`.

## 3) AgentLoop Core Behavior

Main loop implementation: `agent/src/agent/loop.py::AgentLoop.run(...)`.

Per run, it does:
1. Create `run_dir` via `RunStateStore.create_run_dir(...)` under `agent/runs/`.
2. Save original prompt to `req.json`.
3. Build system+user messages via `ContextBuilder.build_messages(...)`.
4. Iterate ReAct cycle up to `max_iterations`:
   - stream model output (`llm.stream_chat(...)`)
   - if tool calls exist, execute tools
   - append tool results back into conversation
   - continue until final text answer or termination condition
5. Persist trace to `trace.jsonl` using `TraceWriter`.
6. Mark state success/failure in `state.json`.

Tool execution details:
- Read-only tools can be parallelized.
- Write tools execute serially.
- `run_dir` is auto-injected/normalized for tool calls.
- Tool results are truncated for context safety and recorded in trace.

## 4) For Your Specific Prompt: Typical Tool Sequence

For a prompt like BTC-USDT 20/50 MA backtest + summary + export, the system prompt and skills strongly bias to this sequence:

1. `load_skill("strategy-generate")`
2. `write_file("config.json", ...)`
   - likely includes `source: "okx"` (or `auto`) and `codes: ["BTC-USDT"]`
   - date range for 2024
3. `write_file("code/signal_engine.py", ...)`
   - strategy logic for MA20/MA50 crossover
4. optional syntax check via `bash(...)`
5. `backtest(run_dir=...)`
6. `read_file("artifacts/metrics.csv")`
7. optionally `read_file("run_card.md")` and/or write a custom report markdown via `write_file(...)`

Why this sequence is likely:
- It is explicitly encoded in `agent/src/agent/context.py` task-routing and in `agent/src/skills/strategy-generate/SKILL.md`.

## 5) Backtest Engine and Metrics Generation

`backtest` tool implementation: `agent/src/tools/backtest_tool.py`.

It validates:
- `config.json` exists and is valid
- `code/signal_engine.py` exists
- run directory is inside allowed roots (`safe_run_dir`)

Then it invokes:
- `agent/src/core/runner.py::Runner.execute(...)`
- subprocess entry: `agent/backtest/runner.py` with `<run_dir>` argument

Backtest runner (`agent/backtest/runner.py`) does:
1. Validate and load config schema.
2. Load and validate `SignalEngine` implementation.
3. Fetch market data (BTC-USDT -> OKX path in common case).
4. Run market engine.
5. Write artifacts under `<run_dir>/artifacts/`, including:
   - `metrics.csv`
   - `equity.csv`
   - `trades.csv`
6. Generate run card automatically:
   - `run_card.json`
   - `run_card.md`

Return and drawdown come from `artifacts/metrics.csv`, and CLI summary reads those fields (`total_return`, `max_drawdown`, etc.).

## 6) "Export the report" Behavior

There is no dedicated generic `export_report` backtest tool in the normal strategy path.

In practice, export usually happens via one of these:
1. Automatic run card files from engine:
   - `<run_dir>/run_card.md`
   - `<run_dir>/run_card.json`
2. Agent writes a custom markdown report file using `write_file(...)` in run dir.
3. Final answer includes report text directly in chat output.

So for your prompt wording, the most reliable exported report artifact is `run_card.md` (auto-generated after successful backtest).

## 7) End-to-End Code Flow Diagram

```mermaid
flowchart TD
  A[CLI: vibe-trading run -p "..."] --> B[pyproject script -> cli:main]
  B --> C[agent/cli/main.py::main]
  C --> D{interactive?}
  D -->|No for run -p| E[delegate to cli._legacy.main]
  E --> F[_build_parser + parse args]
  F --> G[args.command == run]
  G --> H[_handle_prompt_command]
  H --> I[_read_prompt_source]
  I --> J[cmd_run]
  J --> K[_run_agent]
  K --> L[build_registry + ChatLLM + AgentLoop]
  L --> M[AgentLoop.run]
  M --> N[LLM decides tool calls]
  N --> O[write_file config/signal_engine]
  O --> P[backtest tool]
  P --> Q[Runner.execute subprocess]
  Q --> R[backtest/runner.py]
  R --> S[market engine run]
  S --> T[artifacts/metrics.csv equity.csv trades.csv]
  T --> U[run_card.md + run_card.json]
  U --> V[read_file metrics + summarize return/drawdown]
  V --> W[final CLI output + run_id/run_dir]
```

## 8) Key Files to Read (If You Want to Debug/Customize)

- CLI front door: `agent/cli/main.py`
- Legacy parser + run dispatch: `agent/cli/_legacy.py`
- Agent loop: `agent/src/agent/loop.py`
- Routing/system prompt rules: `agent/src/agent/context.py`
- Tool registry: `agent/src/tools/__init__.py`
- Core tools:
  - `agent/src/tools/backtest_tool.py`
  - `agent/src/tools/write_file_tool.py`
  - `agent/src/tools/read_file_tool.py`
  - `agent/src/tools/load_skill_tool.py`
- Subprocess runner: `agent/src/core/runner.py`
- Backtest entry and orchestration: `agent/backtest/runner.py`
- Run card export: `agent/backtest/run_card.py`

## 9) Practical Note for Your Example

For your exact prompt, expect outputs to be in one new run directory under `agent/runs/<run_id>/`, especially:
- `config.json`
- `code/signal_engine.py`
- `artifacts/metrics.csv`
- `run_card.md` (exported report)

If you want, I can also generate a second markdown that maps each function call to approximate line ranges for quicker code navigation.
