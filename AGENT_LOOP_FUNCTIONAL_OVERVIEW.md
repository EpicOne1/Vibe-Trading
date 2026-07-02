# Agent Loop Functional Overview

Source file: `agent/src/agent/loop.py`

## Purpose
The Agent Loop is the runtime controller for an autonomous ReAct-style session. Its job is to move from a user request to a usable final result by repeatedly deciding, acting, and integrating results.

It focuses on:
- making progress toward completion,
- using tools safely and efficiently,
- preserving enough context to continue long tasks,
- and returning a clear terminal state (`success`, `failed`, or `cancelled`).

## Functional Responsibilities

### 1. Run Lifecycle Management
The loop creates and manages a run session, including:
- initializing run state and trace output,
- loading conversation context (including optional goal context),
- executing iteration-by-iteration reasoning/action,
- and marking final run outcome.

### 2. Iterative ReAct Execution
Each iteration functionally does the following:
1. Refresh context with new background notifications if present.
2. Keep context within working size limits.
3. Ask the model for the next step.
4. If the model answers directly, evaluate whether to finalize or continue (goal-aware flows).
5. If the model requests tools, execute tools and feed results back into context.

This repeats until a stop condition is met.

### 3. Context Survivability for Long Sessions
The loop applies multi-layer context reduction to avoid exceeding model limits while retaining continuity:
- lightweight pruning of older tool outputs,
- collapsing long old text blocks,
- structured conversation compaction into handoff-style summaries,
- model-invoked compaction when needed,
- iterative summary updates to reduce information decay.

Functional outcome: long tasks can continue without hard context failure.

### 4. Tool Orchestration and Safety
Tool execution behavior is goaled toward throughput and control:
- read-only tools may run in parallel,
- write tools run serially,
- duplicate non-repeatable successful calls are skipped,
- timeouts are enforced differently for read-only vs write operations,
- tool results are normalized into success/error semantics.

Functional outcome: faster reads, safer writes, reduced accidental repetition.

### 5. Real-Time Progress Emission
The loop emits events for external UI/CLI consumers, including:
- streamed text progress,
- reasoning progress indicators,
- tool call/progress/heartbeat/result events,
- compaction events,
- usage and retry/reset signals.

Functional outcome: users get continuous visibility rather than opaque waiting.

### 6. Goal-Aware Continuation
When a session is linked to an active goal, the loop can:
- account token/turn usage into goal tracking,
- check whether the goal still needs continuation,
- append continuation prompts after intermediate answers,
- stop continuation when capped or when no net progress is detected.

Functional outcome: multi-turn goals can continue automatically but remain bounded.

### 7. Content Filter Resilience
If model output is blocked by provider moderation:
- the loop records and skips blocked turns,
- retries via subsequent iterations,
- and triggers a circuit breaker after too many consecutive blocked responses.

Functional outcome: avoids endless filtered loops and returns a diagnosable failure reason.

### 8. Cooperative Cancellation
Cancellation is polled at key boundaries:
- before/within iterations,
- during streaming,
- and between tool batches.

Functional outcome: the run stops quickly and safely without waiting for full loop exhaustion.

### 9. Traceability and Auditability
The loop persists run artifacts and trace events such as:
- lifecycle checkpoints,
- tool calls/results (with redaction handling),
- compaction summaries,
- reasoning/answer milestones,
- and final status reasons.

Functional outcome: sessions are inspectable for debugging, review, and postmortem analysis.

## Stop Conditions
The loop ends when one of these occurs:
- a final answer is produced,
- user cancellation is detected,
- content-filter circuit breaker trips,
- an unrecoverable runtime/provider error occurs,
- an empty terminal model response is detected,
- or max iterations are reached without completion.

## Final Status Semantics
- `success`: meaningful output or expected completion artifact exists.
- `failed`: unrecoverable error, blocked-loop safety trigger, empty response, or iteration exhaustion.
- `cancelled`: user-triggered stop.

## Why This Design Matters (Functional View)
This loop is designed for long, tool-heavy autonomous workflows where reliability matters more than single-turn speed. It balances:
- progress pressure (bounded iterations + wrap-up behavior),
- memory pressure (multi-layer compaction),
- safety (tool policies, timeout handling, cancellation),
- and operability (rich event stream + durable traces).

In practical terms, it helps the agent keep working productively across complex tasks while still ending with a clear, user-facing outcome.
