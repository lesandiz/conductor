# Recommended Improvements

This document captures architectural improvements for a new orchestrator implementation, based on the weaknesses and bottlenecks identified in the Conductor analysis.

## 1. Replace Single-Cursor Engine with Scheduler/Executor Split

**Problem**: The engine tracks one `current_agent_name` and advances sequentially. Parallelism is bolted on via explicit parallel groups. Independent DAG branches cannot execute concurrently. No pipelining is possible.

**Recommendation**: Split into two components with distinct responsibilities.

### Scheduler (Pure Logic, No Side Effects)

The scheduler maintains a **frontier** of ready-to-execute nodes and makes scheduling decisions:

```
Scheduler:
    Input:  workflow graph + completion notifications
    Output: set of nodes ready to execute

    on_workflow_start():
        frontier = { nodes with no dependencies }
        emit all frontier nodes to executor

    on_node_complete(node, output):
        store output
        evaluate routes from completed node
        for each routed-to node:
            if all dependencies satisfied:
                add to frontier
                emit to executor
        if routed to "$end":
            build final output

    on_node_failed(node, error):
        apply failure mode logic
        save checkpoint
```

The scheduler is a pure function of (graph state, completion events). It has no knowledge of providers, I/O, or side effects. This makes it testable, predictable, and swappable.

### Executor (Side Effects, Concurrency)

The executor pulls nodes from the scheduler's frontier and runs them:

```
Executor:
    Input:  node to execute + context view
    Output: completion notification (output or error)

    Responsibilities:
        - Provider management (connection pooling, session reuse)
        - Prompt rendering
        - Tool execution
        - Event emission
        - Output validation
        - Retry logic
```

### Benefits

- **Natural concurrency**: Independent branches parallelize automatically. No explicit parallel group needed just to get concurrency.
- **Pipelining**: Executor can prepare the next node (render prompt, warm connection) while the current one runs.
- **Parallel groups become barrier nodes**: A parallel group is a constraint ("these must all complete before routing") rather than a mechanism ("please run these concurrently").
- **Testability**: Scheduler logic tested in isolation without provider mocks.
- **Swappable execution**: Replace the executor with a distributed version without changing scheduling logic.

### Parallel Groups in This Model

Parallel groups don't disappear — they become **barrier nodes** with collection semantics:

```
Normal DAG nodes:
    Agent A -> Agent B          (B starts when A completes)
    Agent A -> Agent C          (C starts when A completes; B and C run concurrently)

Barrier node (parallel group):
    Agent A -> [ParallelBarrier: B, C, D] -> Agent E
    B, C, D execute concurrently (natural from graph)
    Barrier collects outputs + applies failure mode
    E starts only after barrier releases
```

The barrier adds: output collection into a single log entry, failure mode enforcement, and a synchronization point. Concurrency itself falls out of the graph topology.

## 2. Replace Deep-Copy Context with Append-Only Log

**Problem**: Context is rebuilt from scratch for every agent via deep copy. Cost is O(N^2) in agent count.

**Recommendation**: Maintain a single append-only context store with read-only views.

### Append-Only Log

```
Context Store (single instance, grows linearly):
+----------+----------+----------+----------+
| Agent 1  | Agent 2  | Agent 3  | Group 4  |  <- append-only
| output   | output   | output   | outputs  |
+----------+----------+----------+----------+
      ^          ^          ^          ^
   Agent 2    Agent 3    Agent 4    Agent 5     <- read-only views
   sees [0]   sees [0-1] sees [0-2] sees [0-3]    (no copies)
```

### Design Principles

- **Prior outputs are immutable**: Once Agent 3 finishes, its output never changes. No copy needed.
- **Read-only views**: Each agent gets a view bounded by an index, not a materialized copy.
- **Copy only at parallel boundaries**: When a parallel group starts, snapshot the log once. All agents in the group share this single snapshot.
- **Parallel group output**: The group itself appends a single entry containing all individual outputs + errors.

### Lazy Materialization

The context dict only exists to feed template rendering. Most agents reference a small subset of prior outputs. Instead of eagerly building a dict with everything:

1. Give the template renderer a **resolver function** instead of a materialized dict.
2. The resolver reads from the append-only log on demand.
3. Only values actually referenced in the template get accessed.

### Cost Reduction

| Approach                | Per-Agent Cost                   | Total (N agents) |
| ----------------------- | -------------------------------- | ---------------- |
| Current (deep copy all) | O(K) where K = all prior output  | O(N^2)           |
| Read-only view          | O(1) — bounded reference         | O(N)             |
| + Lazy materialization  | O(R) where R = fields referenced | O(N * R)         |
| + Parallel snapshot     | O(S) only at barrier entry       | O(N + P * S)     |

### What Agents See (Unchanged)

The data available to agents doesn't change — every agent can still access all prior outputs, workflow inputs, and iteration metadata. The improvement is in how that data is accessed (reference vs. copy).

### What Agents Don't See

Prior agent prompts should NOT flow through context. Prompts are orchestrator instructions — implementation details, not inter-agent data. If a downstream agent needs to understand what a prior agent was doing, the output schema should include that information (e.g., a `reasoning` or `approach` field). The original workflow input (user's intent) is always available via `workflow.input`.

## 3. Eager Resource Initialization

**Problem**: Providers, MCP servers, and sessions are initialized lazily during execution, causing cold-start delays mid-workflow.

**Recommendation**: Initialize all resources at workflow startup, overlapping with config validation.

### Initialization Pipeline

```
Workflow Start:
    |
    +-- Parse + validate config (already sequential)
    |
    +-- In parallel:
    |       +-- Initialize all providers (validate_connection)
    |       +-- Connect all MCP servers (discover tools)
    |       +-- Pre-warm connection pools
    |
    +-- Begin execution (all resources ready)
```

### Provider Connection Pooling

Maintain a pool of warm provider connections, keyed by (provider_type, model):

- **Session reuse**: Sequential agents using the same provider and model should reuse a session rather than creating a new one.
- **Pool sizing**: For parallel groups, pre-create sessions for all agents in the group before execution starts.
- **Lifecycle**: Pool connections closed atomically on workflow completion or failure.

### MCP Eager Connection

MCP servers should connect at workflow startup, not on first tool use:

- Analyze the workflow graph to determine which agents use tools.
- Connect required MCP servers during the initialization phase.
- Tool discovery completes before any agent executes.
- If a server fails to connect, fail fast with a clear error rather than failing mid-workflow.

## 4. Async Event Emission

**Problem**: `emit()` blocks until all subscribers process the event. A slow subscriber (WebSocket broadcast) blocks the engine.

**Recommendation**: Make event emission non-blocking.

### Options

**Option A: Fire-and-Forget Queue**

Subscribers receive events via an async queue. The emitter enqueues and returns immediately. Subscribers process at their own pace. Risk: queue grows unbounded if a subscriber falls behind.

**Option B: Bounded Queue with Backpressure**

Same as Option A but with a bounded queue. If a subscriber falls behind, oldest events are dropped (lossy) or the emitter blocks (backpressure). Acceptable for observability events — dropping a `route_taken` event is better than blocking the engine.

**Option C: Dual-Mode Emission**

Critical events (workflow_completed, workflow_failed) are delivered synchronously to ensure subscribers see them. Observability events (agent_message, tool_start) are delivered asynchronously. This preserves ordering guarantees for lifecycle events while unblocking the hot path.

### Recommendation

Option C. Lifecycle events are rare and must be reliable. Observability events are frequent and can tolerate slight delays or drops.

## 5. Parallel Tool Execution

**Problem**: When the model returns multiple independent tool calls, they are executed sequentially. Cost is N x latency.

**Recommendation**: Execute independent tool calls concurrently.

```
Model returns: [tool_call_A, tool_call_B, tool_call_C]

Current:
    execute(A) -> 500ms
    execute(B) -> 500ms
    execute(C) -> 500ms
    Total: 1500ms

Improved:
    gather(execute(A), execute(B), execute(C))
    Total: max(500ms, 500ms, 500ms) = 500ms
```

This is a straightforward change within the provider's agentic loop. The model already indicates which tool calls are independent.

## 6. Structured Output via Tool-Use, Not Prompt Instructions

**Problem**: Prompt-based schema enforcement (appending JSON instructions to the prompt) is unreliable and triggers expensive parse recovery loops (up to 5 retries).

**Recommendation**: Use tool-based structured output exclusively.

The tool-based approach (synthetic `emit_output` tool with input schema matching the output schema) is more reliable because:

- The model is trained to produce valid tool call inputs.
- The schema is enforced by the API's tool-use validation, not by prompt compliance.
- Parse recovery is cheaper (retry with "use the tool" guidance vs. re-parsing free-form text).
- Recovery attempts needed: typically 0-1 vs. 0-5.

For providers that don't support tool-use natively, the executor should synthesize the tool-use pattern at the executor level rather than falling back to prompt-based enforcement.

## 7. Configurable Parse Recovery Budget

**Problem**: Parse recovery is hardcoded (5 attempts for one provider, 2 for another). No user control.

**Recommendation**: Make parse recovery configurable per agent:

```yaml
agent:
  name: analyzer
  output:
    parse_recovery:
      max_attempts: 2        # Default: 2
      strategy: tool_guided  # Or: re_prompt, fail_fast
```

This lets workflow authors trade latency for reliability on a per-agent basis. Critical agents get more retries; fast-feedback agents fail fast.

## 8. Non-Blocking Checkpoint I/O

**Problem**: Checkpoint file writes are synchronous, blocking the async event loop for 50-200ms.

**Recommendation**: Offload checkpoint writes to a background thread or use async file I/O.

Checkpoints are only written on failure, so the frequency is low. But when they do occur, they shouldn't block the event loop — especially during error handling where responsiveness matters (the user may be waiting for an error message).

## 9. Separate Concerns in the Engine

**Problem**: The engine is a god object that handles scheduling, execution, interrupt handling, event emission, checkpoint I/O, usage tracking, web dashboard interaction, gate handling, and more.

**Recommendation**: Decompose into focused components:

| Component              | Responsibility                                         |
| ---------------------- | ------------------------------------------------------ |
| **Scheduler**          | Graph traversal, frontier management, route evaluation |
| **Executor**           | Agent execution, provider management, tool resolution  |
| **Context Store**      | Append-only log, views, trimming                       |
| **Event Bus**          | Async pub/sub with subscriber management               |
| **Checkpoint Manager** | State serialization, file I/O, resume logic            |
| **Gate Manager**       | Human-in-the-loop interaction, CLI/Web racing          |
| **Interrupt Handler**  | Keyboard listening, signal delivery, pause/resume      |
| **Usage Tracker**      | Token counting, cost calculation                       |

Each component has a narrow interface and can be tested, replaced, or extended independently. The scheduler orchestrates by coordinating between these components, not by implementing their logic directly.

## Priority Order

Ranked by impact on a new implementation:

1. **Scheduler/Executor split** — foundational architecture change; everything else is easier to implement correctly on top of this
2. **Append-only context log** — eliminates O(N^2) scaling; enables efficient parallel execution
3. **Eager resource initialization** — eliminates mid-workflow cold starts
4. **Tool-based structured output** — reduces parse recovery cost and latency variance
5. **Parallel tool execution** — low-effort, high-impact for tool-heavy workflows
6. **Async event emission** — prevents observability from impacting execution
7. **Non-blocking checkpoint I/O** — minor but easy win
8. **Configurable parse recovery** — user-facing flexibility
