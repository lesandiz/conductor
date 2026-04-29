# Core Architecture

This document captures the fundamental architecture of the Conductor orchestrator, distilled from codebase analysis. It is language-agnostic and focuses on the structural patterns, data flow, and design decisions that define the system.

## Abstraction Layers

The orchestrator decomposes into six isolation layers, each with a single responsibility:

| Layer        | Responsibility                                               | Key Abstraction                                |
| ------------ | ------------------------------------------------------------ | ---------------------------------------------- |
| **Config**   | Declarative workflow definition to validated in-memory graph | Schema models + cross-reference validator      |
| **Engine**   | State machine driving execution order                        | While-loop with node-type dispatch             |
| **Context**  | Data flow between agents                                     | Scoped context builder with accumulation modes |
| **Executor** | Prompt rendering + output validation                         | Composition over providers (not inheritance)   |
| **Provider** | SDK/API interaction                                          | Abstract contract returning normalized output  |
| **Event**    | Observability (decoupled from execution)                     | Synchronous pub/sub with subscriber isolation  |

The engine never talks to an LLM directly. It talks to executors, which compose providers. This triple indirection (engine -> executor -> provider) is what makes provider-swapping possible without touching the engine.

## Configuration Pipeline

```
YAML File
    |
    v
YAML Parser (with !file tag support for includes)
    |
    v
Environment Variable Resolution (${VAR:-default}, up to 10 levels deep)
    |
    v
Schema Validation (type-safe models with required fields and defaults)
    |
    v
Cross-Field Validation
    - Agent/route reference checking (all routes point to existing agents)
    - Input declaration validation (referenced outputs exist)
    - Parallel group constraints (no cross-agent dependencies within groups)
    - Path coverage analysis (warnings if template refs may not execute on all paths)
    |
    v
Validated Workflow Config (in-memory object graph)
```

Key features:
- **Circular file detection**: File include stack prevents recursive `!file` references.
- **Two validation layers**: Schema validation catches type errors; cross-field validation catches semantic errors (dangling references, unreachable agents).
- **Source location tracking**: YAML line numbers extracted for error messages.

## Execution Model: Single-Cursor State Machine

The core loop is a while-true state machine with node-type dispatch:

```
+------------------------------------------------------+
|                    MAIN LOOP                         |
|                                                      |
|  resolve current_node_name                           |
|       |                                              |
|       +-- for_each_group? -> batch-execute agent     |
|       +-- parallel_group? -> concurrent agent tasks  |
|       +-- human_gate?     -> race CLI vs Web input   |
|       +-- script?         -> subprocess execution    |
|       +-- sub_workflow?   -> recursive engine        |
|       +-- llm_agent?      -> executor.execute()      |
|                                                      |
|  store output in context                             |
|  evaluate routes (first-match wins)                  |
|  route.target == "$end" ? -> build final output      |
|  else -> current_node_name = route.target; continue  |
|                                                      |
|  check interrupts between nodes                      |
+------------------------------------------------------+
```

Design decisions:
- **First-match routing**: Routes are evaluated sequentially. First `when` condition that evaluates to truthy wins. A route with no `when` clause always matches (default/fallback).
- **Termination signal**: Any route targeting `"$end"` exits the loop and builds final output from templates.
- **Interrupts at two points**: mid-agent (partial output from abort) and between-agent (pause/guidance collection).
- **Safety limits**: Iteration counter + wall-clock timeout enforced before each node execution.

## Routing

Routes are evaluated by a router component using two expression types:

1. **Template expressions** (if `{{` and `}}` present): Rendered through Jinja2 and evaluated as a boolean.
2. **Simple expressions** (otherwise): Evaluated via a safe arithmetic evaluator with a flattened context.

Route definitions include:
- `to`: Target agent name or `"$end"`
- `when`: Optional condition string (first match wins; no `when` = always matches)
- `output`: Optional output transformation templates applied before storing

## Context Management

### Three Accumulation Modes

| Mode                     | What's Visible to an Agent                   | Use Case                                 |
| ------------------------ | -------------------------------------------- | ---------------------------------------- |
| **accumulate** (default) | All prior agent outputs                      | Linear pipelines needing full history    |
| **last_only**            | Only the immediately previous agent's output | Memory-constrained sequential processing |
| **explicit**             | Only declared input references               | Strict contracts, minimal data leakage   |

### Input Reference Syntax

Agents declare what context they need via reference strings:

```
workflow.input.param_name         - Workflow input parameter
agent_name.output                 - Full output of a prior agent
agent_name.output.field           - Specific field from output
agent_name.field                  - Shorthand for .output.field
parallel_group.outputs            - All outputs from a parallel group
parallel_group.outputs.agent_name - Specific agent within a group
for_each_group.outputs            - All outputs from a for-each group
for_each_group.errors             - Errors from a for-each group
```

Suffix with `?` for optional references (missing = null instead of error).

### Context Object Structure

The context maintains:
- **workflow_inputs**: Initial workflow input parameters
- **agent_outputs**: Map of agent name to structured output
- **current_iteration**: Execution count (for loop detection)
- **execution_history**: Ordered list of agent names executed
- **user_guidance**: Accumulated guidance from interactive interrupts

### Context Trimming

Three strategies available when context exceeds size limits:
- **drop_oldest**: Remove entire agent outputs FIFO until within budget
- **truncate**: Shorten string fields in oldest agents (keep at least 50 chars)
- **summarize**: Use an LLM to condense old outputs (fallback to drop_oldest)

## Parallel Execution

### Static Parallel Groups

Named set of agents executed concurrently. Three failure modes:

| Mode                  | Behavior                                   |
| --------------------- | ------------------------------------------ |
| **fail_fast**         | Cancel remaining agents on first error     |
| **continue_on_error** | Continue all; fail only if ALL agents fail |
| **all_or_nothing**    | All must succeed or entire group fails     |

Context isolation: parallel agents receive a snapshot of context at group entry. They cannot see each other's outputs. The group acts as a barrier — nothing downstream executes until the group completes.

Output is stored as a single log entry:

```
parallel_group_name:
  type: parallel
  outputs:
    agent_a: { ... }
    agent_b: { ... }
  errors:
    agent_c: { error_type, message }
```

### Dynamic Parallel Groups (For-Each)

Iterate over a runtime-resolved array, executing an inline agent template for each item:

- **source**: Dotted path to an array in prior context (resolved at runtime)
- **loop variable**: Injected as `{{ item }}`, `{{ _index }}`, `{{ _key }}`
- **max_concurrent**: Batching limit (default 10)
- **key_by**: Optional path to extract dict keys (outputs become a dict instead of list)
- Same failure modes as static parallel groups

## Event System

### Pattern

Synchronous pub/sub with exception isolation.

```
Engine._emit(type, data)
    |
    v
EventEmitter.emit(WorkflowEvent)
    | (snapshot subscriber list under lock, then iterate)
    |
    +-- ConsoleSubscriber (formatted CLI output)
    +-- EventLogSubscriber (JSONL to temp file)
    +-- WebDashboard (queues -> WebSocket broadcast)
```

Design decisions:
- **Synchronous delivery**: Callbacks execute in order before `emit()` returns.
- **Thread-safe subscription**: Lock protects list mutations; snapshot copy prevents deadlock during iteration.
- **Exception isolation**: One failing subscriber does not prevent others from receiving the event.
- **Zero overhead when disabled**: Early return when no emitter is configured.

### Event Types

Approximately 20 event types covering:
- Workflow lifecycle (started, completed, failed)
- Agent execution (started, completed, failed, paused, resumed)
- Human gates (presented, resolved)
- Parallel/for-each progress (started, item completed, item failed, group completed)
- Routing (route taken)
- Tool calls (tool start, tool complete)
- Observability (prompt rendered, reasoning, messages)
- Recovery (checkpoint saved)

## Error Handling & Recovery

### Exception Hierarchy

All errors extend a base error type with:
- **suggestion**: Actionable fix advice for the user
- **file_path, line_number**: Source location in workflow YAML
- **error_type**: Display name for categorization

Categories: configuration, validation, template, provider, execution (with sub-types for max iterations, timeout, gate errors, interrupts, retryable errors), and checkpoint errors.

### Checkpoint/Resume

On any failure, the engine serializes full state to a JSON file:
- Workflow path and content hash
- Failure details (error type, message, agent name, iteration)
- Full context state (all agent outputs)
- Limit state (iteration counter)
- Provider session IDs (for session-resumable providers)

Resume path: load checkpoint, restore context and limits, create new engine, re-enter main loop from the failed agent. A hash check warns if the workflow YAML changed since checkpoint creation.

### Per-Agent Retry

Configurable per agent with:
- Max attempts (1-10)
- Backoff strategy (fixed or exponential)
- Delay between attempts
- Error category filter (`provider_error`, `timeout`)
- Non-retryable errors (validation, schema mismatch) are never retried

## Human-in-the-Loop

### Gates

Gates are workflow nodes that pause execution for human input:
- Present labeled options, each mapping to a route target
- Optionally collect text input via `prompt_for` fields
- Result includes selected option + any collected text

### Dual-Path Input

CLI and web dashboard race concurrently for gate responses:
- CLI: Rich-formatted prompt with blocking stdin read (offloaded to thread to avoid blocking event loop)
- Web: WebSocket message from browser
- First response wins; other is cancelled
- Auto-skip mode selects first option (for CI/CD automation)

### Mid-Agent Interrupts

- Keyboard listener runs in daemon thread (Esc or Ctrl+G detection)
- Delivers interrupt signal via thread-safe event
- Provider aborts current execution, returns partial output
- Engine can: resume with guidance (CLI), pause/resume/kill (Web), or auto-resume (no clients connected)

## Web Dashboard

### Architecture

In-process async HTTP + WebSocket server running alongside the engine:
- Engine emits events -> dashboard queues them -> broadcasts via WebSocket
- Full event history accumulated for late-joiner replay
- HTTP endpoints for control: stop, resume, kill

### Late-Joiner Protocol

1. New client fetches full event history via HTTP
2. Client replays all events to reconstruct UI state
3. Client subscribes to live WebSocket feed
4. No gap between history and live — events are accumulated atomically

### Background Mode

Detached subprocess with:
- Auto-port selection
- PID file tracking for discovery by stop command
- Auto-shutdown after workflow completes and all clients disconnect (30s grace period)

## Workflow Registry

### Reference Resolution

Workflows can be referenced by name instead of file path:

```
workflow[@registry][@version]

Examples:
  "my-workflow"              -> default registry, latest version
  "my-workflow@team"         -> "team" registry, latest version
  "my-workflow@team@1.2.3"   -> "team" registry, explicit version
```

Resolution order: if it exists as a file on disk, treat as file; otherwise parse as registry reference.

### Registry Types

| Type       | Source                         | Caching                                                 |
| ---------- | ------------------------------ | ------------------------------------------------------- |
| **github** | GitHub repository (owner/repo) | Cached per version (immutable snapshots)                |
| **path**   | Local filesystem directory     | No caching (reads directly, reflects edits immediately) |

### Registry Index

Each registry contains an index file listing available workflows with:
- Name, description, relative path within the registry
- Version list (latest = last in list)

### Cache Structure

```
~/.conductor/cache/registries/{registry_name}/{workflow_name}/{version}/
```

GitHub registries fetch workflow YAML + sibling files. Explicit versions are treated as immutable (cache hit = skip fetch).
