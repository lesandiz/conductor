# Performance Analysis

This document captures the performance bottlenecks identified in the Conductor orchestrator, organized by severity and root cause. These findings should inform the design of a new implementation.

## Critical Bottlenecks (Seconds-Level Delays)

### Provider Session Creation — No Pooling or Reuse

Every agent execution creates a new provider session. There is no connection pooling, session reuse, or warm-start optimization.

- **Stateful SDK provider**: Spawns a CLI subprocess on first use, then creates a new session per agent via `create_session()`. Even sequential agents that use the same provider and model get fresh sessions.
- **Stateless API provider**: Creates a new client per execution.
- **Cost**: 200-400ms per agent, multiplied by every agent in the workflow.
- **In parallel groups**: First agent triggers subprocess spawn (500ms-2s); all others block waiting on a single lock, then each creates its own session sequentially.

### Idle Detection — Busy-Wait with Long Timeout

The stateful SDK provider monitors for stuck sessions with a 90-second default idle timeout and up to 5 recovery attempts.

- A tight loop checks a completion flag continuously without meaningful backoff.
- When the idle timeout fires, the provider sends a recovery prompt and waits again.
- **Worst case**: 90s timeout x 5 recovery attempts = 450+ seconds of artificial latency on a stuck session.
- This happens transparently — the user sees the workflow hanging with no indication of what's happening.

### MCP Late Initialization — Cold Start Mid-Workflow

MCP server connections are initialized lazily on first tool use, not at workflow startup.

- Each `connect_server()` call spawns a subprocess, initializes a session, and discovers tools.
- **Cost**: 1-3 seconds per MCP server, paid during the first agent that uses tools.
- The engine has already started executing agents before MCP is ready, meaning the first tool-using agent pays the full cold-start cost.

### Parse Recovery Loops — Repeated Round-Trips on Schema Mismatch

When an agent's response doesn't match the declared output schema, providers attempt in-session recovery:

- **Stateful SDK**: Up to 5 in-session recovery attempts, each requiring a full round-trip.
- **Stateless API**: Up to 2 recovery attempts via tool-use guidance.
- **Cost**: 1-2 seconds per recovery attempt. On persistent schema mismatch, 5-10 seconds burned before failing.
- This happens every time an agent returns non-schema-compliant JSON, which is common with prompt-based schema enforcement.

### Provider Cold Start — Subprocess Spawn

The first provider execution incurs a one-time subprocess spawn cost:

- **Stateful SDK**: 1-3 seconds to spawn and initialize the CLI subprocess.
- **Stateless API**: 100-500ms for SDK instantiation and connection pool setup.
- This cost is paid once per provider type, but it's paid during the first agent execution (not at workflow startup), delaying the first visible result.

## Moderate Bottlenecks (100ms-Level, Compounds Across Agents)

### Context Accumulation — O(N^2) Growth

In accumulate mode (the default), `build_for_agent()` deep-copies all prior agent outputs for every agent execution:

```
Agent 1: copies 0 prior outputs     ->   0 KB
Agent 2: copies Agent 1             ->  10 KB
Agent 3: copies Agents 1-2          ->  20 KB
Agent 4: copies Agents 1-3          ->  30 KB
...
Agent N: copies Agents 1 to N-1     -> (N-1) * size KB
```

**Total allocation**: `N*(N-1)/2 * avg_output_size`. For 10 agents at 10KB each: 450KB allocated across all builds. For 50 agents: 12.25MB.

The cost is not the final context size but the cumulative allocation across all builds. Each build does a recursive deep copy, which is expensive for nested objects.

### Sequential MCP Tool Execution

When the stateless API provider receives multiple tool calls from the model, it executes them one at a time in a loop. Even when the model specifies independent parallel tool calls, they are serialized.

- **Cost**: N x 500ms average per MCP call, where N is the number of concurrent tool calls.
- Could use async gather for concurrent execution with no semantic change.

### Event Callback Overhead

Synchronous callbacks fire on every SDK event during agent execution:

- Debug logging for every event type.
- Event forwarding with data extraction.
- Rich formatting for verbose output.
- **Cost**: 1-5ms per event x hundreds of events per agent = 100-500ms per agent.

### Blocking Checkpoint I/O

Checkpoint file writes are synchronous, blocking the async event loop:

- `write_text()` — synchronous file write
- `chmod()` — synchronous permission change
- `rename()` — synchronous atomic rename
- **Cost**: 50-200ms per checkpoint write, depending on storage speed.

### Config Validation

Full schema validation runs on every workflow load:

- YAML parsing with env var resolution
- Pydantic model validation of all nested objects
- Cross-field semantic validation
- **Cost**: 100-300ms for large workflows, compounds on sub-workflow loads.

## Architectural Issues (Design-Level)

### No Connection Pooling

Each provider instance creates fresh connections. There is no mechanism to:
- Reuse sessions across sequential agents (same provider, same model)
- Pool HTTP connections across API calls
- Share MCP connections across providers

### No Eager Initialization

Expensive resources are initialized lazily:
- Provider connection validation happens on first use
- MCP servers connect on first tool call
- Session creation happens per-agent

All of these could be initialized eagerly at workflow startup, overlapping with config validation.

### Synchronous Event Emission

`emit()` blocks until all subscribers return. A slow subscriber (e.g., WebSocket broadcast to many clients) blocks the engine's hot path. The engine cannot proceed to the next step until all subscribers have processed the event.

### No Context Sharing Optimization

Parallel agents deep-copy the entire context independently. For a parallel group with 5 agents, the same context is cloned 5 times. A single snapshot shared as read-only views would reduce this to 1 copy.

### No Work Pipelining

The single-cursor execution model means the engine blocks on the current agent before considering the next one. There is no opportunity to:
- Pre-render the next agent's prompt while the current one executes
- Warm up provider connections for upcoming agents
- Pre-resolve tool lists for agents later in the graph

### Blocking Subprocess I/O

The stateful SDK provider forces synchronous I/O for subprocess communication, preventing async I/O from being used for large payloads.

## Per-Agent Hot Path Cost Breakdown

For each agent in the main loop, the system pays:

| Step                                    | Cost                         |
| --------------------------------------- | ---------------------------- |
| Context building (deep copy)            | 5-50ms (scales with history) |
| Prompt rendering (Jinja2)               | 10-50ms                      |
| Session creation                        | 200-400ms                    |
| Event callbacks during execution        | 100-500ms                    |
| Idle detection (only on stuck sessions) | 0-90,000ms                   |
| Token accumulation                      | ~5ms                         |
| Output template rendering               | 5-50ms                       |
| Context storage                         | 5-20ms                       |

**Minimum per-agent overhead (non-stuck)**: 330-650ms

For a 10-agent sequential workflow, orchestration overhead alone is 3.3-6.5 seconds, independent of actual LLM response time.

## Quantified Impact Summary

| Bottleneck           | Per-Occurrence Cost | Frequency        | Worst-Case Impact |
| -------------------- | ------------------- | ---------------- | ----------------- |
| Session creation     | 200-400ms           | Every agent      | N * 400ms         |
| Idle detection       | 90s per timeout     | Stuck sessions   | 450s              |
| MCP cold start       | 1-3s per server     | Once per server  | S * 3s            |
| Parse recovery       | 1-2s per attempt    | Schema mismatch  | 10s               |
| Provider cold start  | 1-3s                | Once per type    | 3s                |
| Context accumulation | O(N^2)              | Every agent      | 12MB at 50 agents |
| Sequential tools     | 500ms per tool      | Multi-tool calls | N * 500ms         |
| Event callbacks      | 100-500ms           | Every agent      | N * 500ms         |
| Checkpoint I/O       | 50-200ms            | Every failure    | 200ms             |
