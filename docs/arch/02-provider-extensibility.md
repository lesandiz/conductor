# Provider Extensibility

This document captures the provider abstraction, factory pattern, tool integration, and output normalization architecture. It is language-agnostic and focuses on the extensibility surface.

## Provider Contract

All providers implement three abstract methods:

### execute()

```
execute(
    agent_def,          - Agent configuration (name, model, output schema, tools, etc.)
    context,            - Workflow context dict built by the context manager
    rendered_prompt,    - Jinja2-rendered prompt string (rendering done by executor, not provider)
    tools?,             - List of tool names available to this agent (null = all, [] = none)
    interrupt_signal?,  - Async event that, when set, signals the provider to abort
    event_callback?,    - Callback for streaming provider events upstream
) -> AgentOutput
```

### validate_connection()

```
validate_connection() -> bool
```

Verifies the provider can reach its backend (API key valid, SDK available, etc.). Called once during initialization.

### close()

```
close() -> void
```

Releases provider resources: close connections, terminate subprocesses, disconnect MCP servers.

## Normalized Output

All providers return the same output structure regardless of backend:

```
AgentOutput:
    content: dict           - Structured output matching agent's output schema
    raw_response: any       - Provider-specific response (for debugging)
    tokens_used: int?       - Total token count (input + output)
    input_tokens: int?      - Input token count
    output_tokens: int?     - Output token count
    cache_read_tokens: int? - Prompt cache read tokens
    cache_write_tokens: int? - Prompt cache write tokens
    model: string?          - Actual model used (may differ from requested)
    partial: bool           - True if output is incomplete (mid-agent interrupt)
```

The normalization principle: consumers of `AgentOutput` (engine, context, event subscribers, dashboard) never need to know which provider produced it. The `raw_response` field exists solely for debugging.

## Provider Factory

Providers are instantiated via a factory function using runtime dispatch:

```
create_provider(
    provider_type,         - "copilot", "claude", etc.
    validate?,             - Whether to call validate_connection() after creation
    mcp_servers?,          - MCP server configuration dict
    default_model?,        - Model override
    temperature?,          - Temperature override
    max_tokens?,           - Max tokens override
    timeout?,              - Per-request timeout
    max_session_seconds?,  - Agent session wall-clock limit
    max_agent_iterations?, - Tool-use loop iteration limit
) -> AgentProvider
```

Adding a new provider requires:
1. Implement the abstract contract (3 methods)
2. Add a case to the factory dispatch
3. No other code changes needed

## Provider Registry (Multi-Provider Support)

The registry enables workflows where different agents use different providers:

```
Workflow YAML:
  workflow:
    runtime:
      provider: copilot        <- default

  agents:
    - name: planner
      provider: claude         <- override for this agent
    - name: executor           <- uses default (copilot)
```

### Registry Behavior

- **Lazy instantiation**: Providers are created on first use, not at workflow start.
- **Caching**: One instance per provider type, reused across all agents using that type.
- **Per-agent resolution**: Check agent's `provider` field first, fall back to workflow default.
- **Lifecycle**: All cached providers closed atomically when workflow completes.

```
Agent execution request
    |
    v
registry.get_executor_for_agent(agent_def)
    |
    +-- agent.provider set? -> use it
    +-- else -> use workflow default
    |
    v
registry.get_or_create_provider(provider_type)
    |
    +-- cached? -> return existing instance
    +-- else -> create_provider() -> cache -> return
    |
    v
AgentExecutor(provider=resolved_provider)
```

## Executor Layer (Composition Pattern)

The executor sits between the engine and the provider, handling concerns that are provider-agnostic:

```
Engine
    |
    v
AgentExecutor
    |-- TemplateRenderer (Jinja2 prompt rendering)
    |-- tool resolution (null=all, []=none, [list]=subset)
    |-- output validation (schema conformance)
    |
    v
Provider (SDK/API interaction)
```

### Executor Pipeline

1. **Render model field** if it contains template expressions (dynamic model selection)
2. **Render prompt** with Jinja2 using workflow context
3. **Append guidance** from human-in-the-loop interrupts (if any)
4. **Emit event**: prompt rendered (for observability)
5. **Resolve tool list**: agent's tool whitelist against workflow's global tool list
6. **Execute via provider**: delegate to the provider contract
7. **Normalize output**: ensure `content` is a dict (parse JSON if needed)
8. **Validate output**: check against agent's declared output schema (skip if partial)

### Separation of Concerns

| Component     | Responsibility                                                         |
| ------------- | ---------------------------------------------------------------------- |
| **Executor**  | Template rendering, tool resolution, output validation, error wrapping |
| **Provider**  | SDK interaction, tool execution, structured output extraction, retries |
| **Renderer**  | Jinja2 template evaluation with custom filters                         |
| **Validator** | Output schema conformance checking with detailed error messages        |

## Tool / MCP Integration

### Tool Resolution

Tools are resolved at three levels:

1. **Workflow level**: Global tool list available to the workflow
2. **Agent level**: Per-agent tool configuration
   - `null` (not specified): Inherit all workflow tools
   - `[]` (empty list): No tools available
   - `["tool_a", "tool_b"]`: Whitelist specific tools
3. **Provider level**: Provider receives the resolved tool list and handles execution

### MCP Server Lifecycle

```
1. CONNECTION (lazy, on first tool use)
   MCPManager.connect_server(name, command, args, env, timeout)
       |
       v
   Spawn subprocess (stdio transport) or connect (HTTP/SSE)
       |
       v
   Initialize session
       |
       v
   Discover tools via session.list_tools()

2. TOOL NAMESPACING
   Tools are prefixed with server name: {server}__{tool_name}
   Example: "web-search__search" (tool "search" from server "web-search")
   Mapping stored: prefixed_name -> server_name (for routing)

3. TOOL EXECUTION
   MCPManager.call_tool(prefixed_name, arguments)
       |
       v
   Route to correct server via mapping
       |
       v
   Execute via session.call_tool(original_name, arguments)
       |
       v
   Extract text content from result

4. CLEANUP
   MCPManager.close()
       |
       v
   Close all stdio transports / HTTP connections
   Clear internal state
```

### Provider-Specific MCP Handling

| Aspect             | Stateful SDK Provider                                        | Stateless API Provider                   |
| ------------------ | ------------------------------------------------------------ | ---------------------------------------- |
| **Configuration**  | Passed at session creation; SDK handles execution internally | Lazy-init MCPManager on first tool use   |
| **Tool Execution** | SDK-managed (opaque to provider)                             | Provider-managed agentic loop (explicit) |
| **Tool Format**    | SDK-native tool definitions                                  | Converted to API-specific tool format    |

## Structured Output Enforcement

Two approaches observed across providers:

### Prompt-Based (Stateful SDK)

Schema appended to prompt as text instructions:
```
"You MUST respond with a JSON object matching this schema:
{field_name: {type, description}, ...}
Return ONLY the JSON object, no other text."
```

Parse recovery: up to 5 in-session retries with targeted error guidance.

### Tool-Based (Stateless API)

Structured output enforced via a synthetic tool:
- Build a tool named `emit_output` with input schema matching the agent's output schema
- Instruct the agent to use this tool to return results
- If the agent returns text instead of a tool call, retry with guidance to use the tool

Parse recovery: up to 2 retries via tool guidance.

### Output Validation Pipeline

After provider returns output:

1. Check each declared field exists in output
2. Type-check values against schema (`string`, `number`, `boolean`, `array`, `object`)
3. Recursively validate nested objects and array items
4. Special case: `bool` is not a valid `number` (prevents language-specific subtype issues)

## Provider Divergence Points

Current providers differ in fundamental execution models, creating tension with the unified contract:

| Concern               | Stateful Session Model                    | Stateless API Model                   |
| --------------------- | ----------------------------------------- | ------------------------------------- |
| **Session**           | Create -> send -> disconnect (long-lived) | New request per execution (stateless) |
| **Tool execution**    | SDK-managed internally (opaque)           | Provider-managed explicit loop        |
| **Structured output** | Schema in prompt text                     | Synthetic tool with input schema      |
| **Idle recovery**     | Timeout + recovery prompts (90s default)  | API-level timeout only                |
| **Parse recovery**    | 5 in-session retries                      | 2 retries via tool                    |
| **Interrupts**        | Session abort method                      | Task cancellation                     |
| **Temperature**       | Not supported (SDK limitation)            | Validated and passed to API           |

This divergence means provider parity is aspirational. The abstraction holds for the happy path but leaks at the edges (idle detection, parse recovery, session lifecycle).

## Event Callback Protocol

Providers emit events upstream via a callback function:

```
callback(event_type: string, data: dict) -> void
```

Event types forwarded from providers:
- `agent_turn_start`: Turn number or "awaiting_model" (before each API call)
- `agent_message`: Text content streamed from agent
- `agent_tool_start`: Tool name and arguments
- `agent_tool_complete`: Tool name and result/error
- `agent_reasoning`: Internal reasoning content
- `agent_prompt_rendered`: Full rendered prompt (from executor level)
- `agent_retry`: Retry attempt with error context

## Extensibility Summary

| Extension Point                | Steps Required                              | Blast Radius            |
| ------------------------------ | ------------------------------------------- | ----------------------- |
| New provider                   | Implement 3 abstract methods + factory case | Isolated by contract    |
| New MCP transport              | Extend MCPManager.connect_server()          | MCP layer only          |
| Custom template filters        | Register in TemplateRenderer init           | Template rendering only |
| New tool protocol              | Extend tool resolution + provider handling  | Executor + provider     |
| Custom output validation       | Extend validate_output()                    | Validator only          |
| New structured output strategy | Implement in provider                       | Single provider only    |

## Script Executor

A separate executor for shell command agents (no provider involvement):

- Template rendering applied to command, args, working_dir
- Async subprocess execution with per-script timeout
- Environment variable merging (OS env + agent env)
- Output captured as stdout/stderr/exit_code
- Stored in context like any other agent output
