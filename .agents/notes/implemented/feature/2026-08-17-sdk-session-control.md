# Agent Note: SDK session control

Status: implemented

English | [中文](2026-08-17-sdk-session-control.zh.md)

## Problem

The agent interface supports same-turn steering and cooperative cancellation, but subprocess SDK clients can only enqueue next-turn prompts or terminate the complete runtime. An automation owner cannot redirect a running turn without waiting for the next turn, and stopping one turn destroys every session hosted by that runtime.

## Decision

The SDK protocol exposes `session/steer` and `session/cancel` without changing the [agent control operations](../../../../packages/core/agent/README.md#agent-interface-typests). The JSON-RPC server routes accepted steering to `agent.steer()` and accepted cancellation to `agent.cancel({ kind: 'user' })`; the TypeScript and Python SDKs expose matching low- and high-level methods.

## Wire semantics

`session/steer` accepts the same content blocks as `session/prompt`, creates the session lazily when needed, and returns the durable `MessageId`. The message enters the waking next-step inbox, so a running agent consumes it at the nearest later step boundary in the current turn and an idle agent opens a turn for it.

`session/cancel` never creates a session. It returns `{ accepted: true }` only when the server observes a live running agent and requests cooperative cancellation; unknown and idle sessions return `{ accepted: false }`. Acceptance is not a completion result, so callers observe `session.event` and `session.status` for convergence.

Prompt, steering, and cancellation remain independent requests. The protocol does not assign later assistant messages or `turn/end` events to one input, and it retains no per-session close operation.

## Alternatives considered

**Encode steering as another `session/prompt`.** This loses the next-step versus next-turn distinction and makes a same-turn redirect depend on queue timing, so it cannot preserve the agent interface semantics.

**Cancel by closing the runtime subprocess.** This stops unrelated sessions, discards process-level reuse, and turns a cooperative turn control into infrastructure failure.

**Add control behavior to the agent loop.** The loop already owns the required operations; duplicating them would create a second lifecycle implementation instead of exposing the existing one at the SDK wire.

## Consequences

SDK clients can redirect and stop active work while retaining the runtime and its other sessions. The wire gains two pre-release request/result pairs, both SDK implementations and their documentation move together, and clients still own activity intervals because neither operation supplies a prompt-level completion result.

## Testing

Server tests pin next-step routing, live-agent validation, running-only cancellation, unknown-session behavior, and shutdown rejection. TypeScript and Python subprocess tests pin both client layers and malformed TypeScript receipts.
