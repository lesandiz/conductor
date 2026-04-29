# Sandbox Execution Architecture

This document defines the execution dispatch layer and sandbox isolation model for the orchestrator. It covers the trust boundary, two isolation strategies, credential injection, context marshaling, and MCP server placement.

## Threat Model

The threat is **data exfiltration by a compromised or misbehaving agent**. An agent has access to workflow context (which may contain sensitive data) and could use any outbound channel to leak it:

- Encoding data in web search queries
- Sending data via HTTP to an attacker-controlled endpoint
- Writing data to a shared filesystem
- Exfiltrating via DNS queries
- Using MCP tools as side channels

The sandbox is a **trust boundary**, not a resource boundary. Its purpose is to prevent any data from leaving except through the orchestrator's output channel.

## Execution Dispatch Layer

The dispatch layer sits between the scheduler (which decides what to run) and the provider (which talks to the LLM). It decides **where** to run an agent and manages the execution environment.

```
Scheduler: "Node X is ready"
    |
    v
Execution Dispatch
    |
    +-- Resolve execution environment for this agent
    |
    +-- LocalExecutor
    |       Direct provider call, same process
    |       No isolation boundary
    |       Context passed by reference
    |
    +-- SandboxExecutor
    |       Container or microVM on same machine
    |       Full isolation boundary
    |       Context serialized in, output serialized out
    |
    +-- RemoteExecutor
            Container or microVM on remote host
            Full isolation boundary + network transport
            Context serialized in, output serialized out
    |
    v
AgentOutput (normalized, same shape from all three paths)
```

The scheduler and provider are unaware of the execution environment. The dispatch layer is transparent to both sides.

### Environment Resolution

Two mechanisms, evaluated in order (per-agent overrides policy):

**Per-agent declaration:**
```yaml
agents:
  - name: code_executor
    execution:
      environment: sandbox
```

**Workflow-level policy:**
```yaml
workflow:
  execution_policy:
    default: local
    rules:
      - when: <condition>
        environment: sandbox
      - when: <condition>
        environment: remote
```

The decision is binary per agent: sandboxed or not. If sandboxed, everything runs inside.

## Two Isolation Models

### Model A: Container Sandbox (Namespace Isolation)

Based on the devcontainer pattern. Uses standard Docker containers with iptables firewall rules.

```
Host
+--------------------------------------------------+
|  Container (namespace isolation)                  |
|                                                   |
|  Agent process                                    |
|  MCP servers (all of them)                        |
|  Provider credentials (bind-mounted, read-only)   |
|                                                   |
|  iptables:                                        |
|    Default policy: DROP                           |
|    Allow: LLM API endpoint                        |
|    Allow: DNS (for domain resolution)             |
|    Allow: loopback                                |
|    Allow: established connections                 |
|    Deny: everything else                          |
+--------------------------------------------------+
```

**How it works:**
- Container starts from a pre-built image containing the agent runtime and MCP server binaries.
- `init-firewall.sh` runs as a post-start hook, setting iptables to default-deny and allowlisting only the LLM API endpoint.
- Credentials are bind-mounted read-only from the host. They never enter the image.
- Serialized context is written to a mounted volume or passed via stdin.
- Output is collected from stdout or a mounted volume.
- Container is destroyed after execution.

**Strengths:**
- Works anywhere Docker runs. No hypervisor required.
- Fine-grained iptables control over allowed domains.
- Well-understood, stable technology.
- Credential injection via bind mounts is proven.

**Weaknesses:**
- Shares host kernel. A kernel exploit could escape the sandbox.
- iptables rules run inside the container. An agent with `NET_ADMIN` capability could flush them (though the orchestrator controls the container definition, so `NET_ADMIN` can be withheld for sandboxed agents — unlike the devcontainer reference which grants it for the firewall setup itself).
- Agent cannot run Docker inside the container without Docker-in-Docker hacks.

**Mitigation for iptables tampering:**
Run the firewall setup from outside the container after start, or use Docker's own network policy (`--network=none` + explicit endpoint allowlisting at the Docker daemon level) instead of iptables inside the container. This removes the need for `NET_ADMIN` entirely.

### Model B: MicroVM Sandbox (Hypervisor Isolation)

Based on the Docker AI Sandbox pattern. Uses hypervisor-level isolation with a dedicated VM per agent.

```
Host
+--------------------------------------------------+
|  microVM (hypervisor isolation)                   |
|                                                   |
|  Own kernel                                       |
|  Own Docker daemon                                |
|  Agent process                                    |
|  MCP servers (all of them)                        |
|                                                   |
|  Networking:                                      |
|    All outbound -> host-side HTTP/HTTPS proxy     |
|    Proxy enforces allowlist                       |
+--------------------------------------------------+
```

**How it works:**
- microVM starts from a template image (pulled from a registry, not shared with host).
- Workspace (serialized context) mounted via passthrough at identical paths.
- All outbound traffic routes through an HTTP/HTTPS proxy on the host. Network policy is enforced outside the VM — the agent cannot tamper with it.
- Own Docker daemon allows agents to build/run containers inside the sandbox.
- VM is destroyed after execution.

**Strengths:**
- Full VM boundary. No shared kernel.
- Network policy enforced outside the VM. Agent cannot tamper with firewall rules.
- Agent can run Docker natively (own daemon).
- Stronger isolation for untrusted or highly autonomous agents.

**Weaknesses:**
- Requires hypervisor support (KVM on Linux or equivalent).
- Images must be pulled from a registry — no local image sharing.
- Higher overhead per sandbox (full VM boot vs container start).
- Experimental technology. Less mature than containers.

### When to Use Which

| Scenario                              | Recommended Model                                                                    |
| ------------------------------------- | ------------------------------------------------------------------------------------ |
| Trusted agents with tool access       | Container sandbox (Model A)                                                          |
| Untrusted or highly autonomous agents | MicroVM sandbox (Model B)                                                            |
| Agent needs to build/run containers   | MicroVM sandbox (Model B)                                                            |
| No hypervisor available               | Container sandbox (Model A)                                                          |
| Multiple concurrent sandboxed agents  | MicroVM sandbox (Model B) — full isolation between agents with minimal configuration |
| Production / high-security workflows  | MicroVM sandbox (Model B)                                                            |

The orchestrator should support both models behind the same dispatch interface. The agent definition or policy selects which model to use.

## What Crosses the Sandbox Boundary

Only two things cross the boundary, in controlled directions:

| Direction | What                                                          | How                      |
| --------- | ------------------------------------------------------------- | ------------------------ |
| **In**    | Serialized context (log entries the agent needs)              | Mounted volume or stdin  |
| **In**    | Agent definition (name, model, prompt, output schema, tools)  | Mounted volume or stdin  |
| **In**    | Provider credentials (API keys)                               | Read-only bind mount     |
| **In**    | MCP server configurations                                     | Mounted volume           |
| **Out**   | Serialized `AgentOutput` (structured output + token metadata) | stdout or mounted volume |

Nothing else. No network proxy for tools, no shared filesystem beyond the input/output channel, no MCP bridge from inside to outside.

## Credential Injection

Credentials needed inside the sandbox:

| Credential                      | Purpose                             | Injection Method                             |
| ------------------------------- | ----------------------------------- | -------------------------------------------- |
| LLM API key                     | Provider needs it to call the model | Read-only bind mount or environment variable |
| MCP server credentials (if any) | Tools that require authentication   | Read-only bind mount                         |

**Principles:**
- Credentials are authenticated on the host before sandbox creation.
- They are bind-mounted read-only — the sandbox cannot modify them.
- Ephemeral tokens are preferred over long-lived secrets.
- Credentials are scoped to the minimum required (only the LLM API, not general cloud access).
- Credentials never enter the container image. They exist only at runtime via mounts.

## MCP Server Placement

**All MCP servers run inside the sandbox.** No exceptions, no proxying.

The rationale is the trust boundary. If you proxy an MCP server from outside the sandbox into it, you create an exfiltration channel:

```
WRONG — proxy creates an exfiltration channel:
    Sandbox Agent -> proxy -> Host MCP (web search) -> internet
    Agent encodes context data in search queries.
    Proxy cannot inspect tool semantics to prevent this.

RIGHT — everything inside, network controls egress:
    Sandbox Agent -> Sandbox MCP (web search) -> sandbox network policy
    Network policy blocks all egress except LLM API.
    Web search tool fails. Agent cannot exfiltrate.
```

The security decision is at the network layer, not the tool layer. Classifying tools as "safe" or "dangerous" is a losing strategy — any tool with outbound access is a potential exfiltration channel. The network boundary is the right abstraction.

**Implication:** If an agent is sandboxed and needs web search, the workflow author must explicitly add the search API endpoint to the network allowlist. This is a conscious security decision, not a default.

## Network Policy

### Minimum Allowlist for a Sandboxed Agent

```
Allow: LLM API endpoint (required for the provider to function)
Allow: DNS port 53 (required for domain resolution)
Allow: loopback (required for internal MCP communication)
Allow: established/related connections (required for response traffic)
Deny: everything else
```

### Extended Allowlist (per workflow configuration)

Workflow authors can extend the allowlist for specific agents:

```yaml
agents:
  - name: researcher
    execution:
      environment: sandbox
      network:
        allow:
          - "api.anthropic.com:443"
          - "search-api.example.com:443"
```

Each allowed endpoint is an explicit, auditable security decision. The orchestrator should log what was allowed and why.

### Enforcement Location

| Model               | Where policy is enforced                                    | Agent can tamper?                              |
| ------------------- | ----------------------------------------------------------- | ---------------------------------------------- |
| Container (Model A) | iptables inside container, or Docker network policy outside | Inside: yes if NET_ADMIN granted. Outside: no. |
| MicroVM (Model B)   | Host-side HTTP/HTTPS proxy                                  | No — policy is outside the VM                  |

**Recommendation for Model A:** Use Docker network policy (`--network=none` + explicit endpoint routing) or run iptables setup from outside the container. Do not grant `NET_ADMIN` to sandboxed agent containers. This differs from the devcontainer reference, which grants `NET_ADMIN` because the firewall script runs inside — for agent sandboxes, the orchestrator controls the container and can set up networking from outside.

## Sandbox Lifecycle

### For Ephemeral Agent Execution

```
1. PREPARE
   - Resolve execution environment (container vs microVM)
   - Serialize context (only log entries the agent references)
   - Prepare credential mounts
   - Prepare MCP server configs

2. CREATE
   - Start container/VM from pre-built image
   - Mount serialized context (read-only)
   - Mount credentials (read-only)
   - Apply network policy

3. INITIALIZE
   - Start MCP servers inside sandbox
   - Verify network policy (test that disallowed endpoints are blocked)
   - Verify provider connectivity (test that LLM API is reachable)

4. EXECUTE
   - Run agent process
   - Agent calls provider (inside sandbox)
   - Provider calls LLM API (allowed by network policy)
   - Agent uses MCP tools (inside sandbox, network-restricted)
   - Agent produces output

5. COLLECT
   - Read serialized AgentOutput from sandbox
   - Deserialize and validate
   - Return normalized output to dispatch layer

6. DESTROY
   - Terminate all processes
   - Remove container/VM
   - No state persists
```

### Verification Step

Before executing the agent, the sandbox should verify its own security posture:

```
Verification:
  - Confirm: blocked endpoint is unreachable (e.g., example.com)
  - Confirm: LLM API endpoint is reachable
  - Confirm: credentials are mounted and valid
  - If any check fails: abort with clear error, do not execute agent
```

This mirrors the devcontainer's `init-firewall.sh` verification step and ensures the sandbox is correctly configured before any agent code runs.

## Pre-Built Sandbox Images

Sandbox images should be pre-built and published to a registry, not built on-the-fly during workflow execution.

### Image Composition

```
Base image (minimal OS + runtime)
    |
    +-- Agent runtime (language runtime, orchestrator agent binary)
    +-- MCP server binaries (pre-installed)
    +-- Provider SDK (pre-installed)
    +-- No credentials (injected at runtime)
    +-- No context (injected at runtime)
```

### Image Variants

Following the Docker AI Sandbox template pattern, maintain purpose-specific variants:

| Variant       | Contents                                     |
| ------------- | -------------------------------------------- |
| `minimal`     | Agent runtime only, no tools                 |
| `with-tools`  | Agent runtime + common MCP servers           |
| `with-docker` | Agent runtime + Docker daemon (microVM only) |

Workflow authors can extend base images for domain-specific tooling:

```dockerfile
FROM orchestrator/sandbox:with-tools
RUN apt-get update && apt-get install -y protobuf-compiler
```

Custom images must be pushed to a registry. The sandbox pulls at creation time — no local image sharing (especially for microVM model where the image store is isolated).

## Dispatch Interface

The dispatch layer exposes a single async method:

```
async dispatch(
    agent_def,              - Agent configuration
    context_view,           - Read-only view into the append-only log
    execution_environment,  - "local" | "sandbox" | "remote"
    sandbox_config?,        - Image, network policy, credential mounts
) -> AgentOutput
```

### How the Scheduler Uses It

The scheduler does NOT await `dispatch()` inline. It spawns each dispatch as a concurrent task and continues scheduling other ready nodes:

```
Scheduler loop:

    ready_nodes = find_nodes_with_satisfied_dependencies()

    for node in ready_nodes:
        task = spawn( dispatch(node) )     <-- concurrent task, returns Future<AgentOutput>
        pending.add(task)

    completed = wait_for_any(pending)      <-- yields until at least one finishes

    for task in completed:
        output = task.result()             <-- AgentOutput
        store_in_log(output)
        evaluate_routes(node)
        # may produce new ready_nodes -> next iteration
```

The scheduler never blocks on a single dispatch. It manages a set of pending futures and processes completions as they arrive.

### What Happens Inside dispatch()

The implementation varies by environment, but the signature is always the same — async function that eventually returns `AgentOutput`:

```
async dispatch(node):

    match environment:

        local:
            return await provider.execute(...)
            // Direct call, direct return. Milliseconds to minutes.

        sandbox:
            create_container()
            inject_context()
            start_agent()
            output = await read_output_from_container()
            destroy_container()
            return output
            // Waits for container to finish. Local I/O. Seconds to minutes.

        remote:
            completion = new CompletionSignal()
            job_id = POST /execute { callback_url: our_endpoint }
            register_handler(job_id, completion)

            // SUSPEND HERE. This task yields.
            // The scheduler continues running other nodes.
            //
            // ...minutes pass...
            //
            // The remote host finishes and POSTs to callback_url.
            // The orchestrator's HTTP server receives it.
            // The registered handler resolves the CompletionSignal
            // with the AgentOutput from the callback payload.

            output = await completion       // resumes here
            return output
```

### The Callback Is Internal to the Remote Executor

The callback from the remote host does NOT go to the scheduler. It goes to an HTTP endpoint inside the dispatch layer, which resolves the completion signal that `dispatch()` is suspended on:

```
Scheduler                    Dispatch (remote)              Remote Host
    |                              |                             |
    +-- spawn(dispatch(node))      |                             |
    +-- add future to pending      |                             |
    +-- continue scheduling        |                             |
    |                              +-- POST /execute ----------> |
    |                              +-- register handler          +-- create sandbox
    |                              +-- await completion          +-- execute agent
    |                              |   (task suspended)          |
    |   (running other nodes)      |                             |
    |                              |                             +-- agent completes
    |                              |   <--- callback POST ------+-- POST /results
    |                              +-- handler fires             |
    |                              +-- completion.resolve(output)|
    |                              +-- return AgentOutput        |
    |                              |                             |
    +-- future resolves            |                             |
    +-- store output in log        |                             |
    +-- evaluate routes            |                             |
    +-- enqueue new ready nodes    |                             |
```

The scheduler sees the same thing for all three environments: a future that eventually resolves to `AgentOutput`. It never knows whether the dispatch completed via a direct return (local), a container exit (sandbox), or an HTTP callback (remote).

### Polling Fallback

If the orchestrator cannot expose an inbound HTTP endpoint for callbacks (NAT, firewall), the remote executor falls back to polling:

```
async dispatch(node):   // remote, polling mode
    job_id = POST /execute { ... }    // no callback_url

    loop:
        await sleep(poll_interval)
        response = GET /status/{job_id}
        if response.status == "completed":
            return deserialize(response.output)
        if response.status == "failed":
            raise ExecutionError(response.error)
```

Polling is a degraded mode — it adds latency (average half the poll interval) and wastes resources. Callback is always preferred.

## Context Serialization for Sandbox Execution

The append-only context log enables efficient serialization for sandbox execution:

### What Gets Serialized

For an agent with explicit inputs (`input: [planner.output, workflow.input.goal]`), only those specific log entries are serialized. For accumulate mode, all log entries up to the current index are serialized.

### Serialization Format

A single JSON document containing:

```json
{
  "workflow_input": { ... },
  "agent_outputs": [
    { "name": "agent_1", "output": { ... } },
    { "name": "agent_2", "output": { ... } }
  ],
  "agent_def": { ... },
  "iteration": 5,
  "execution_history": ["agent_1", "agent_2"]
}
```

### Output Deserialization

The sandbox writes a single JSON document:

```json
{
  "content": { ... },
  "tokens_used": 1234,
  "input_tokens": 800,
  "output_tokens": 434,
  "model": "claude-sonnet-4-6",
  "partial": false
}
```

The dispatch layer deserializes this into an `AgentOutput` and returns it to the scheduler. The scheduler stores it in the append-only log like any other output.

## Remote Sandbox Execution

Local sandboxes (container or microVM) share a filesystem with the orchestrator — context goes in via mounted volumes, output comes out the same way. Remote sandboxes (Kubernetes pod, EC2 instance, any VPS-like compute) have no shared filesystem. This introduces a **transport problem**: how does context get to the remote host, and how does output get back?

### Architecture

The remote host runs a lightweight **agent runtime service** — a thin shim that accepts execution requests, manages sandboxes, and returns results. It does not understand workflows, routing, or scheduling. It only knows how to run a single agent in a sandbox.

```
Orchestrator (scheduler + dispatch)              Remote Host (agent runtime)
    |                                                 |
    +-- serialize context + agent def                 |
    +-- POST /execute  -----------------------------> |
    |   { context, agent_def, sandbox_config,         +-- pull sandbox image
    |     credentials, network_policy }               +-- create sandbox
    |                                                  +-- inject context (volume)
    |   <-- 202 Accepted { job_id } -----------------+-- inject credentials (RO mount)
    |                                                  +-- apply network policy
    |                                                  +-- start MCP servers
    |                                                  +-- verify sandbox posture
    |                                                  +-- execute agent
    |   (orchestrator continues other work)            |
    |                                                  +-- agent completes
    |   <-- callback POST /results { AgentOutput } ---+-- collect output
    |                                                  +-- destroy sandbox
    +-- deserialize output                             |
    +-- return to scheduler                            |
```

### Transport Protocol

The dispatch layer communicates with the remote agent runtime via HTTP. Two phases:

**Submission (orchestrator to remote):**

```
POST /execute
Content-Type: application/json

{
  "job_id": "uuid",
  "agent_def": { ... },
  "context": {
    "workflow_input": { ... },
    "agent_outputs": [ ... ],
    "iteration": 5,
    "execution_history": [ ... ]
  },
  "sandbox_config": {
    "image": "orchestrator/sandbox:with-tools",
    "isolation": "container",
    "network": {
      "allow": ["api.anthropic.com:443"]
    }
  },
  "callback_url": "https://orchestrator.example.com/results/uuid",
  "timeout_seconds": 600
}

Response: 202 Accepted { "job_id": "uuid" }
```

**Completion (remote to orchestrator):**

```
POST {callback_url}
Content-Type: application/json

{
  "job_id": "uuid",
  "status": "completed",
  "output": {
    "content": { ... },
    "tokens_used": 1234,
    "input_tokens": 800,
    "output_tokens": 434,
    "model": "claude-sonnet-4-6",
    "partial": false
  }
}
```

Or on failure:

```
POST {callback_url}
Content-Type: application/json

{
  "job_id": "uuid",
  "status": "failed",
  "error": {
    "type": "ExecutionError",
    "message": "Agent exceeded timeout",
    "retryable": true
  }
}
```

### Why Callback, Not Polling

Agent execution can take minutes. Polling wastes resources and adds latency (average half the poll interval before the orchestrator learns of completion). The callback model is preferred because:

- The dispatch task suspends (yields) and resumes only when the callback arrives. Zero idle CPU.
- The scheduler continues running other nodes immediately. No waiting.
- Latency from completion to scheduler notification is near-zero (HTTP round-trip only).

Polling is supported as a fallback when the orchestrator cannot expose an inbound HTTP endpoint (see Dispatch Interface section for details).

### Credential Forwarding to Remote Hosts

Credentials cannot be bind-mounted on a remote host — there's no shared filesystem. Two approaches:

**Option A: Credentials in the execution request (encrypted)**

```
POST /execute
{
  ...
  "credentials": {
    "llm_api_key": "<encrypted>",
    "mcp_auth": "<encrypted>"
  }
}
```

The agent runtime decrypts and injects into the sandbox as environment variables or files. The transport must be TLS-encrypted. Credentials exist in memory on the remote host only for the duration of execution.

**Option B: Remote host has its own credential store**

The remote host retrieves credentials from a secrets manager (Vault, AWS Secrets Manager, GCP Secret Manager) at sandbox creation time. The orchestrator sends a credential reference, not the credential itself.

```
POST /execute
{
  ...
  "credentials": {
    "llm_api_key": { "source": "vault", "path": "secret/orchestrator/anthropic" }
  }
}
```

**Recommendation:** Option B for production. The orchestrator never handles raw credentials for remote execution. Option A is acceptable for development/prototyping with the constraint that transport is always TLS.

### The Agent Runtime Service

A minimal service deployed on each remote host capable of running sandboxed agents.

**Responsibilities:**
- Accept execution requests via HTTP
- Pull sandbox images from a registry
- Create and manage sandboxes (container or microVM)
- Inject context and credentials into the sandbox
- Apply network policy
- Monitor execution (timeout enforcement, health checks)
- Collect output from the sandbox
- Report results via callback or polling endpoint
- Destroy sandbox after execution

**What it does NOT do:**
- Workflow scheduling or routing
- Context accumulation or management
- Route evaluation
- Event emission to the orchestrator's event bus
- Checkpoint management

It is a **dumb executor** — it runs what it's told and reports the result.

**Interface:**

```
POST   /execute              Submit an agent execution request
GET    /status/{job_id}      Poll for job status (fallback)
DELETE /jobs/{job_id}         Cancel a running job
GET    /health               Health check (for load balancing)
```

### Remote Execution Lifecycle

```
1. SUBMIT
   Orchestrator serializes context + agent def + sandbox config.
   POST /execute to remote agent runtime.
   Receive job_id.

2. PROVISION
   Remote host pulls sandbox image (cached after first pull).
   Creates container/VM.
   Writes serialized context to volume inside sandbox.
   Retrieves and injects credentials.
   Applies network policy.

3. INITIALIZE
   Starts MCP servers inside sandbox.
   Verifies network policy (blocked endpoint test).
   Verifies provider connectivity (LLM API reachable).

4. EXECUTE
   Runs agent process inside sandbox.
   Agent runtime monitors for timeout and health.
   Agent produces output.

5. REPORT
   Agent runtime reads output from sandbox.
   POSTs result to callback URL (or stores for polling).
   Destroys sandbox.

6. RECEIVE
   Orchestrator receives callback (or polls for result).
   Deserializes AgentOutput.
   Returns to scheduler. Scheduler stores in append-only log.
```

### Failure Handling

| Failure                             | Who detects it                | What happens                                                |
| ----------------------------------- | ----------------------------- | ----------------------------------------------------------- |
| Agent exceeds timeout               | Agent runtime                 | Kills sandbox, reports `failed` with `retryable: true`      |
| Agent process crashes               | Agent runtime                 | Collects exit code, reports `failed` with error details     |
| Sandbox creation fails              | Agent runtime                 | Reports `failed` before execution starts                    |
| Network policy verification fails   | Agent runtime                 | Reports `failed`, sandbox destroyed without executing       |
| Remote host unreachable             | Orchestrator (dispatch layer) | Retries with backoff, then fails the node                   |
| Callback delivery fails             | Agent runtime                 | Retries callback with backoff; result available via polling |
| Orchestrator restarts mid-execution | Agent runtime                 | Result stored; orchestrator polls on recovery               |

### Timeout Enforcement

Timeouts are enforced at two levels:

- **Agent runtime**: Hard timeout on the sandbox. If the agent doesn't produce output within the configured limit, the sandbox is killed.
- **Orchestrator dispatch**: Deadline on the overall remote execution (includes provisioning, initialization, execution, and reporting). If the agent runtime doesn't report back within this deadline, the dispatch layer marks the job as failed and optionally sends a cancellation.

The orchestrator's deadline should be longer than the agent runtime's timeout to account for provisioning overhead.

### Scaling: Multiple Remote Hosts

When multiple remote hosts are available, the dispatch layer can distribute work:

```
Dispatch Layer
    |
    +-- RemoteExecutor
            |
            +-- Host Pool / Load Balancer
            |       |
            |       +-- Remote Host A (agent runtime)
            |       +-- Remote Host B (agent runtime)
            |       +-- Remote Host C (agent runtime)
            |
            +-- Selection strategy:
                    - Round-robin
                    - Least-loaded
                    - Capability-based (GPU, memory, region)
```

The dispatch layer does not need to implement scheduling across hosts — a standard load balancer or Kubernetes service can handle this. The agent runtime service is stateless (all state is in the sandbox, which is ephemeral), so any host can handle any request.

### Kubernetes as a Remote Runtime

A Kubernetes cluster is a natural host for remote agent execution:

```
Orchestrator (outside or inside cluster)
    |
    POST /execute
    |
    v
Agent Runtime Controller (Kubernetes operator or Job controller)
    |
    +-- Creates a Kubernetes Job with:
    |       - Pod spec from sandbox image
    |       - Resource limits (CPU, memory)
    |       - Network policy (Kubernetes NetworkPolicy)
    |       - Secrets mounted from Kubernetes Secrets
    |       - ConfigMap with serialized context
    |       - Timeout via activeDeadlineSeconds
    |
    +-- Monitors Job status
    |
    +-- On completion: reads output from Pod logs or mounted volume
    +-- Reports result via callback
    +-- Deletes Job and Pod
```

**Kubernetes-specific advantages:**
- NetworkPolicy objects for network isolation (no iptables or proxy needed)
- Secrets for credential injection (native, encrypted at rest)
- Resource limits enforced by the kubelet
- Auto-scaling of worker nodes for burst workloads
- Job TTL controller for automatic cleanup

### Cloud VM as a Remote Runtime

For maximum isolation or specialized hardware (GPU):

```
Orchestrator
    |
    POST /execute
    |
    v
Agent Runtime Service (on a provisioned VM)
    |
    +-- Creates sandbox (container or microVM inside the VM)
    +-- Executes agent
    +-- Reports result
    +-- VM can be terminated after execution (spot/preemptible for cost)
```

The VM itself can be ephemeral — provisioned on demand, destroyed after use. This adds provisioning latency (30s-120s for a VM) but provides the strongest isolation and supports specialized hardware.

### Transport Security

All communication between the orchestrator and remote agent runtime must be:

- **TLS-encrypted**: Context may contain sensitive data. Credentials may be in-flight (Option A).
- **Authenticated**: The agent runtime must verify that requests come from an authorized orchestrator. Mutual TLS or bearer tokens.
- **Authorized**: The orchestrator must verify that callbacks come from a legitimate agent runtime. Signed payloads or shared secrets per job.

### Dispatch Interface (with Remote Config)

The remote executor extends the base dispatch interface with remote-specific configuration:

```
async dispatch(
    agent_def,              - Agent configuration
    context_view,           - Read-only view into the append-only log
    execution_environment,  - "local" | "sandbox" | "remote"
    sandbox_config?,        - Image, network policy, isolation model
    remote_config?,         - Target host/cluster, credential source, timeout
) -> AgentOutput
```

Internally, the remote executor submits work via HTTP, suspends on a completion signal, and resumes when the callback arrives — returning `AgentOutput` through the same future mechanism the scheduler uses for all environments. See the Dispatch Interface section for the full async wiring.

## Comparison: All Execution Models

| Concern                  | Local (no sandbox)             | Container Sandbox                 | MicroVM Sandbox              | Remote Sandbox                                          |
| ------------------------ | ------------------------------ | --------------------------------- | ---------------------------- | ------------------------------------------------------- |
| **Isolation**            | None (same process)            | Namespace (shared kernel)         | Hypervisor (full VM)         | Hypervisor or namespace (on remote host)                |
| **Context transport**    | By reference                   | Mounted volume / stdin            | Mounted volume / passthrough | HTTP (serialized JSON)                                  |
| **Credential injection** | In-process env vars            | RO bind mount                     | RO bind mount or injected    | Secrets manager or encrypted in request                 |
| **Network policy**       | None                           | Docker network policy or iptables | Host-side proxy              | Kubernetes NetworkPolicy, iptables, or proxy            |
| **Agent tamper risk**    | N/A                            | Low (no NET_ADMIN)                | None (policy outside VM)     | None (policy outside sandbox)                           |
| **Provisioning latency** | None                           | ~1-5s (container start)           | ~5-15s (VM boot)             | ~5-120s (container/VM + network)                        |
| **Docker inside**        | N/A                            | No                                | Yes (own daemon)             | Depends on sandbox model used                           |
| **Requires**             | Nothing                        | Docker                            | Docker + hypervisor          | Remote host + agent runtime service                     |
| **Scaling**              | Single machine                 | Single machine                    | Single machine               | Multiple hosts, Kubernetes, cloud VMs                   |
| **Best for**             | Trusted agents, speed-critical | Trusted agents with tool access   | Untrusted/autonomous agents  | Burst workloads, specialized hardware, strong isolation |

### DevContainer Reference Comparison

The devcontainer pattern studied as a reference implementation differs from agent sandboxes in key ways:

| Concern               | DevContainer (reference)                        | Agent Sandbox (any model)                     |
| --------------------- | ----------------------------------------------- | --------------------------------------------- |
| **Purpose**           | Developer environment                           | Agent execution                               |
| **Lifetime**          | Hours/days                                      | Seconds/minutes                               |
| **Network allowlist** | ~30 domains (npm, NuGet, GitHub, VS Code, etc.) | 1 domain (LLM API) unless explicitly extended |
| **State**             | Persistent volumes across rebuilds              | Ephemeral, destroyed after each execution     |
| **NET_ADMIN**         | Granted (firewall script runs inside)           | Not granted (policy applied from outside)     |
| **Interactivity**     | Shell, IDE, human user                          | None — serialized input in, output out        |
