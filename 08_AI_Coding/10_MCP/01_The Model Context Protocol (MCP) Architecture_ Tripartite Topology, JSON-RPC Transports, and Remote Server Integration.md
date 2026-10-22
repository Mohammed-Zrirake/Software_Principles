#AICoding #MCP #ModelContextProtocol #JSONRPC #AgentArchitecture #SystemDesign #OpenCode #APIIntegration

> [!abstract] Executive Summary
> The **Model Context Protocol (MCP)** is an open industry standard that unifies how AI agents and Large Language Models access external **tools**, **live context/resources**, and **prompt templates**. 
> 
> Prior to MCP, connecting $M$ different AI harnesses (OpenCode, Claude Desktop, Cursor, Continue) to $N$ enterprise data silos (PostgreSQL, GitHub, Stripe, Context7) required authoring $M \times N$ proprietary plugins. MCP transforms this into an $M + N$ ecosystem by introducing a standardized client-server protocol over **JSON-RPC 2.0**.
> 
> The architecture relies on three primary actors:
> 1. **MCP Host:** The root agent runtime (e.g., OpenCode) managing agent state and coordinating client connections.
> 2. **MCP Client:** The 1:1 protocol adapter maintaining stateful communication with a single server.
> 3. **MCP Server:** A lightweight provider exposing typed tools, resources, and prompts over either **local process `stdio`** (maximum throughput) or **remote Streamable HTTP** (SaaS & cloud APIs).

---

## 1. The $M \times N$ Integration Crisis & The Need for MCP

Before the establishment of MCP, agent developers faced an unsustainable scaling bottleneck. Every development tool and agent harness implemented its own custom plugin architecture, tool-declaration schema, and parameter-validation pipeline.

```
┌─────────────────────────────────────────────────────────────────────────────────────────┐
│                    THE M × N FRAGMENTATION CRISIS (PRE-MCP)                             │
└─────────────────────────────────────────────────────────────────────────────────────────┘

     AI Harnesses (M)                              Enterprise Services (N)
   ┌─────────────────┐                           ┌────────────────────────┐
   │    OpenCode     │─── Custom TS Plugin ─────▶│   PostgreSQL Engine    │
   └─────────────────┘                           └────────────────────────┘
   ┌─────────────────┐─── Custom Python Ext ────▶┌────────────────────────┐
   │ Claude Desktop  │                           │   Stripe Billing API   │
   └─────────────────┘─── Custom JSON Schema ───▶└────────────────────────┘
   ┌─────────────────┐                           ┌────────────────────────┐
   │     Cursor      │─── Proprietary RPC ──────▶│   GitHub Issues/PRs    │
   └─────────────────┘                           └────────────────────────┘
   
   Result: M × N separate adapters. Brittle, redundant, zero cross-client interoperability.
```

```
┌─────────────────────────────────────────────────────────────────────────────────────────┐
│                    THE M + N STANDARDIZED ECOSYSTEM (WITH MCP)                          │
└─────────────────────────────────────────────────────────────────────────────────────────┘

     AI Harnesses (M)             Standard Protocol             Services & Providers (N)
   ┌─────────────────┐                                        ┌────────────────────────┐
   │    OpenCode     │───┐                                ┌──▶│      PostgreSQL        │
   └─────────────────┘   │                                │   └────────────────────────┘
   ┌─────────────────┐   │         Model Context          │   ┌────────────────────────┐
   │ Claude Desktop  │───┼─────── Protocol (MCP) ─────────┼──▶│    Stripe Billing      │
   └─────────────────┘   │         (JSON-RPC 2.0)         │   └────────────────────────┘
   ┌─────────────────┐   │                                │   ┌────────────────────────┐
   │     Cursor      │───┘                                └──▶│   Context7 Docs API    │
   └─────────────────┘                                        └────────────────────────┘

   Result: M + N adapters. Service authors ship 1 MCP server; all agents consume it universally.
```

### Why Not Just REST APIs and a SKILL.md Prompt?
A common misconception is that MCP is redundant when an agent can simply read documentation in a `SKILL.md` file and execute `curl` commands via the shell. 

However, raw REST APIs impose severe operational liabilities:
1. **Hallucination & Parameter Drifting:** LLMs frequently generate malformed JSON payloads, invent query parameters, or botch URL encodings when calling raw REST endpoints via shell commands.
2. **Context Window Saturation:** Dumping multi-page Swagger/OpenAPI documentation into the context window burns thousands of tokens every turn.
3. **Lack of Lifecycle & Progress Telemetry:** Shell execution cannot natively report real-time chunked streaming progress, token renegotiation, or programmatic cancellation tokens back to the harness.
4. **Zero Capability Negotiation:** With MCP, servers explicitly declare their supported primitives (`tools`, `resources`, `prompts`) during an initial JSON-RPC handshake.

---

## 2. The MCP Tripartite Architecture

The Model Context Protocol specification divides responsibilities across three discrete architectural components:

```
┌─────────────────────────────────────────────────────────────────────────────────────────┐
│                             MCP TRIPARTITE TOPOLOGY                                     │
└─────────────────────────────────────────────────────────────────────────────────────────┘

   ┌───────────────────────────────────────────────────────────────────────────────────┐
   │                                    MCP HOST                                       │
   │  (e.g., OpenCode Runtime, Claude Desktop, Antigravity Agent Engine)               │
   │                                                                                   │
   │   Coordinates user prompts, LLM orchestration, session context, and permissions.  │
   │                                                                                   │
   │   ┌──────────────────────────────┐     ┌──────────────────────────────┐           │
   │   │         MCP Client 1         │     │         MCP Client 2         │           │
   │   │  (Dedicated context7 adapter)│     │  (Dedicated postgres adapter)│           │
   │   └──────────────┬───────────────┘     └──────────────┬───────────────┘           │
   └──────────────────┼────────────────────────────────────┼───────────────────────────┘
                      │ Streamable HTTP (Remote)           │ stdio Pipes (Local)
                      ▼                                    ▼
   ┌──────────────────────────────────┐     ┌──────────────────────────────────┐
   │            MCP SERVER            │     │            MCP SERVER            │
   │       Context7 Docs Daemon       │     │     Local PostgreSQL Daemon      │
   │                                  │     │                                  │
   │   Exposes:                       │     │   Exposes:                       │
   │   - Tool: `resolve_library_docs` │     │   - Tool: `query_database`       │
   │   - Resource: `docs://pytest/...`│     │   - Resource: `db://schema/users`│
   └──────────────────────────────────┘     └──────────────────────────────────┘
```

### Component Breakdown

1. **The MCP Host:**
   - The primary AI application running on the developer's workstation.
   - Manages the active Large Language Model (LLM), context window budgets, user interface (TUI/GUI), and authorization gates.
   - The host does not connect directly to MCP servers; instead, it instantiates and orchestrates **MCP Clients**.

2. **The MCP Client:**
   - A protocol adapter maintained internally by the Host for each configured server.
   - Maintains a stateful 1:1 connection (session lifecycle, ping/pong heartbeats, protocol negotiation).
   - Translates MCP tool definitions into the LLM's native function-calling format (e.g. OpenAI function schemas or Anthropic tool specifications).
   - In OpenCode, when a developer configures five servers in `opencode.jsonc`, the OpenCode Host spins up five dedicated MCP Clients.

3. **The MCP Server:**
   - An independent program or remote microservice that provides contextual data and executable primitives to clients.
   - Operates statelessly or statefully, exposing three core primitives:
     - **Tools:** Callable executable routines with JSON Schema parameters (e.g., `execute_sql`, `create_stripe_refund`).
     - **Resources:** Passive read-only data blobs, files, or URI-addressable streams (e.g., `git://log`, `postgres://schema`).
     - **Prompts:** Pre-engineered prompt templates with parameter inputs.

---

## 3. Communication Transports & JSON-RPC 2.0 Data Layer

MCP decouples the **data layer** (the semantics of function calls and responses) from the **transport layer** (how raw bytes move across process or network boundaries).

### 3.1 The Data Layer: JSON-RPC 2.0
All MCP communication adheres to the **JSON-RPC 2.0 specification**. Every message is a JSON object with a strictly defined structure:

#### Request (Client $\to$ Server):
```json
{
  "jsonrpc": "2.0",
  "id": 1,
  "method": "tools/call",
  "params": {
    "name": "get_documentation",
    "arguments": {
      "library": "pytest",
      "topic": "parametrize"
    }
  }
}
```

#### Response (Server $\to$ Client):
```json
{
  "jsonrpc": "2.0",
  "id": 1,
  "result": {
    "content": [
      {
        "type": "text",
        "text": "pytest.mark.parametrize allows you to define multiple sets of arguments for a single test function..."
      }
    ]
  }
}
```

### 3.2 The Transport Layer: Local vs. Remote

```mermaid
sequenceDiagram
    autonumber
    box Local Subprocess Transport (stdio)
    participant HostStdio as OpenCode Host
    participant ServerStdio as Local Process (e.g. npx @mcp/postgres)
    end
    box Remote Network Transport (Streamable HTTP)
    participant HostHttp as OpenCode Host
    participant ServerHttp as Cloud Server (e.g. Context7)
    end

    Note over HostStdio,ServerStdio: Transport 1: Local stdio (Subprocess)
    HostStdio->>ServerStdio: JSON-RPC over stdin pipe (fd 0)
    ServerStdio-->>HostStdio: JSON-RPC over stdout pipe (fd 1)
    Note over HostStdio,ServerStdio: Ultra-low latency (~0.2ms), zero network overhead

    Note over HostHttp,ServerHttp: Transport 2: Remote Streamable HTTP (2026 Revision)
    HostHttp->>ServerHttp: POST /mcp (JSON-RPC payload over HTTPS)
    ServerHttp-->>HostHttp: HTTP 200 Chunked / SSE Stream with JSON-RPC results
    Note over HostHttp,ServerHttp: SaaS integration, secure remote authentication
```

| Transport Mode | Substrate | Performance | Security Boundary | Best Suited For |
| :--- | :--- | :--- | :--- | :--- |
| **Local (`stdio`)** | OS standard input/output pipes (`stdin`/`stdout`) | **Ultra-Fast** ($< 1\text{ ms}$ IPC latency) | Subprocess spawned under local user permissions | Local DBs, filesystem access, local Git repos, AST parsers |
| **Remote (`Streamable HTTP`)** | Persistent HTTPS connection with chunked streaming | **Network-Bound** ($20\text{--}150\text{ ms}$ RTT) | Transport encryption (TLS 1.3), Bearer tokens, OAuth2 | SaaS APIs (Stripe, GitHub), cloud docs (Context7), enterprise services |

---

## 4. Declarative Configuration in `opencode.jsonc`

OpenCode manages MCP connections declaratively inside `opencode.jsonc`. Configurations can be scoped:
- **Project-Level:** `opencode.jsonc` located at the root of the Git repository.
- **Global:** `~/.config/opencode/opencode.jsonc` for developer-wide defaults.

### 4.1 Schema Definition
```jsonc
{
  "$schema": "https://opencode.ai/config.json",
  "mcp": {
    "servers": {
      // 1. Remote Documentation Server (Context7)
      "context7": {
        "type": "remote",
        "url": "https://mcp.context7.com/mcp",
        "oauth": false
        // Headers omitted for public, rate-limited connection tier
      },

      // 2. Authenticated Remote Server (Stripe API)
      "stripe": {
        "type": "remote",
        "url": "https://mcp.stripe.com/v1",
        "oauth": false,
        "headers": {
          // Secret interpolation: Read dynamically from OS environment
          "Authorization": "Bearer {env:STRIPE_MCP_SECRET_KEY}"
        }
      },

      // 3. Local Subprocess Server (PostgreSQL)
      "postgres-local": {
        "type": "stdio",
        "command": "npx",
        "args": ["-y", "@modelcontextprotocol/server-postgres", "postgresql://dev:secret@localhost:5432/main_db"]
      }
    }
  }
}
```

### 4.2 Credential Hygiene: The `{env:...}` Invariant

> [!CAUTION] Never Commit Plaintext API Keys to `opencode.jsonc`
> Committing plain-text secrets to version-controlled JSON configuration files risks credential leakage.
> 
> OpenCode provides runtime environment variable interpolation using the `{env:VARIABLE_NAME}` syntax. The harness extracts the secret from local memory at boot time without saving it to disk.

---

## 5. Practical Walkthrough: Context7 Integration

Context7 is an authoritative remote MCP documentation server that indexes current library ecosystems, API signatures, and verified framework code examples.

### 5.1 Project-Level Configuration: `opencode.jsonc`
To connect OpenCode to Context7 using its public, rate-limited tier without requiring an API key header:

```jsonc
{
  "$schema": "https://opencode.ai/config.json",
  "mcp": {
    "servers": {
      "context7": {
        "type": "remote",
        "url": "https://mcp.context7.com/mcp",
        "oauth": false
      }
    }
  }
}
```

### 5.2 Verifying Connection Health via CLI
The OpenCode harness provides native CLI tools to inspect client-server connectivity:

```bash
# List all configured MCP servers and their connection state
opencode mcp list
```

Expected output:
```text
NAME       TYPE     STATUS     TOOLS  RESOURCES
context7   remote   connected  3      0
```

Validating via AWK pipeline:
```bash
opencode mcp list | awk '$2 == "context7" { print $3 }'
# Output: connected
```

### 5.3 Autonomous Documentation Lookup (Case Study: Pytest Parametrize)
Once connected, the agent gains access to Context7's documentation search tools. When asked:
> *"How does pytest run the same test with different inputs using parametrize?"*

The agent executes the following interaction behind the scenes:

```mermaid
sequenceDiagram
    autonumber
    actor Dev as Developer
    participant Harness as OpenCode Host
    participant Client as Context7 MCP Client
    participant Server as Context7 Cloud (mcp.context7.com)
    participant LLM as Active Model

    Dev->>Harness: "How do I run parameterized tests in pytest?"
    Harness->>LLM: Ingest prompt + available MCP tools (including Context7)
    LLM-->>Harness: Call tool: `context7_search_docs(library="pytest", query="parametrize")`
    Harness->>Client: Dispatch tool invocation
    Client->>Server: JSON-RPC 2.0 POST `tools/call` over HTTPS
    Server-->>Client: Returns verified documentation for `@pytest.mark.parametrize`
    Client-->>Harness: Ingest documentation result into context
    Harness-->>LLM: Forward tool response
    LLM-->>Dev: Delivers precise, non-hallucinated syntax with code examples!
```

---

## 6. Automated Testing & Boot.dev Validation

When automated grading engines (e.g., Boot.dev CLI) verify MCP configurations, they run dual-tier assertions targeting both the configuration AST and active transport connectivity:

```bash
# Run local verification
bootdev run 252b8131-a982-4b0f-92b2-9fdb69918de3

# Submit graded solution
bootdev run -s 252b8131-a982-4b0f-92b2-9fdb69918de3
```

### Grader Verification Checklist:
1. **JSON AST Structure Check (`cat opencode.jsonc | jq ...`):**
   ```bash
   cat opencode.jsonc | jq '.mcp.servers.context7 | .type == "remote" and .url == "https://mcp.context7.com/mcp" and .disabled != true'
   # Asserts: true
   ```
2. **Daemon Connection Check (`opencode mcp list`):**
   ```bash
   opencode mcp list | awk '$2 == "context7" { print $3 }'
   # Asserts: connected
   ```

---

## 7. Architectural Comparison: MCP vs. Skills vs. Native Tools

```
┌─────────────────────────────────────────────────────────────────────────────────────────┐
│                       INTEGRATION PATTERN COMPARISON MATRIX                             │
└─────────────────────────────────────────────────────────────────────────────────────────┘
```

| Dimension | Native Custom Tools (`2_Tools/`) | Agent Skills (`8_Skills/`) | Model Context Protocol (`10_MCP/`) |
| :--- | :--- | :--- | :--- |
| **Implementation** | TypeScript plugin (`@opencode/plugin`) | Markdown instructions (`SKILL.md`) | Independent JSON-RPC 2.0 Server |
| **Execution Realm** | Local in-process harness execution | LLM context window prompt injection | Out-of-process (Local `stdio` or Remote HTTP) |
| **Ecosystem Portability** | Tied to OpenCode's specific plugin runtime | Open standard across Markdown-based agents | **Universal Standard** (OpenCode, Claude, Cursor, Continue) |
| **Auth & Secrets** | Inherits harness environment variables | Pure text, no direct secrets | Standardized headers, Bearer tokens, `{env:...}` |
| **Maintenance Burden** | Team writes and maintains bespoke code | Team writes and updates Markdown SOPs | **Vendor-Maintained** (Postgres, Stripe, Context7 ship server) |
| **Best For** | Internal CLI scripts, proprietary local code | Multi-step engineering SOPs, PR checklists | Integrating third-party SaaS, DBs, and external docs |
