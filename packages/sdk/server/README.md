# @deepseek-ai/dsh-sdk-jsonrpc-server

English | [中文](README.zh.md)

The `jsonrpc` plugin serves newline-delimited JSON-RPC over stdio so out-of-process SDK clients can drive harness agents. [`HarnessSdkJsonRpcServer`](src/server.ts) owns the protocol methods and notifications; the transport and the named wire types live in [`dsh-sdk-protocol`](../protocol/README.md), shared with the client SDKs; [`jsonrpc-demo`](../../examples/jsonrpc-demo/README.md) supplies the surrounding `cordis.yml` application.

## Wiring

`inject: ['agents']`. The server gets or creates one agent per `sessionId`. It forwards subagent completions only when the service-snapshotted lifecycle `local` flag is true; provider names, child ids, and durable lineage never establish locality. A registered adapter wins, an unowned `deepseek-official` route mounts `dsh-llm-deepseek`, and any other unowned provider fails initialization. Other capabilities come from the surrounding `cordis.yml`.

## Config

`maxTokensAsSuccess` defaults to `false` and affects only the deployment-mapped status on `subagent.finished`; root-session prompts have no prompt-level status. `JsonRpcConfig.input`, `output`, and `exit` are runtime-only transport hooks; production uses process stdio and `process.exit`.

## stdout is the protocol

Stdout carries only JSON-RPC frames. The deployment must not compose a stdout logger; diagnostics belong on stderr.

## Shutdown and exit semantics

The plugin answers `shutdown`, flushes the response, disposes the root context so SDK-owned agents, subscriptions, and persistence reach quiescence, then exits with code 0. EOF and signal exits belong to the app bin, which also disposes the root context. Unloading only this plugin stops serving without exiting the process.

## Wire notes

`initialize.serverInfo.name` is the wire-stable `deepseek-harness-sdk-runtime`. In a Loader deployment, the response waits for every sibling entry to settle, so the first prompt cannot race asynchronous capability setup such as MCP tool discovery. An optional positive `initialize.maxTokens` becomes the request output cap of each SDK-created agent and its in-process descendants; invalid values reject initialization, while omission sends no SDK cap and allows the selected adapter or provider route default to apply. `session/prompt` queues one identified next-turn user message, while `session/steer` queues one identified waking next-step message; both immediately return `{ messageId }`. `session/cancel` requests cooperative user cancellation only for a running session and returns whether the request was accepted. The server streams every durable fact as `session.event` and every whole-agent lifecycle transition as `session.status`; it does not assign an assistant message or `turn/end` to one input. Independent requests may enqueue more work on the same session. Persistence roots and persona come from `cordis.yml`.

## Model Experience

### SDK user message

#### What the model sees

For each accepted `session/prompt` or `session/steer`, the conversation model receives the caller-supplied `contentBlocks` verbatim as one user message in that SDK session. A prompt enters the next turn; steering enters the nearest later step, which is in the current turn when the agent is already running. This package adds no system-prompt prose or tool schema; those come from the plugins in the surrounding `cordis.yml`.

#### Token effect

Data-dependent user-message tokens enter retained session history and are resent on later turns until another package compacts them. The JSON-RPC frames, session notifications, and server bookkeeping add zero model-context tokens.

#### KV Cache effect

Append-only; newly visible content follows the reusable request prefix and does not invalidate existing KV-cache entries.

## Known Limitations and Deferred Work

- **The wire has no per-session close method** — SDK-created agents remain live until process shutdown.
- **There is no per-prompt result** — `MessageId` identifies inbox admission only; clients that own an automation interval must define and observe that interval themselves.
- **stdout purity is deployment-enforced** — a surrounding config can still load a stdout logger and corrupt the JSON-RPC channel; this plugin does not inspect or veto sibling loggers.
- **Automatic adapter mounting is DeepSeek-specific** — `initialize` can reuse any pre-registered model adapter, but its only fallback mounts `dsh-llm-deepseek`.
