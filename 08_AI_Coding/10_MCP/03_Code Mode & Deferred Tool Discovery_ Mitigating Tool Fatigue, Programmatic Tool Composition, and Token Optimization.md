#AICoding #MCP #CodeMode #DeferredDiscovery #ToolFatigue #TokenOptimization #SystemDesign #OpenCode #CloudflareCodeMode

> [!abstract] Executive Summary
> As Model Context Protocol ecosystems expand, agents quickly connect to dozens of servers exposing hundreds of specialized tools. This triggers **Tool Fatigue** (or Tool Bloat): the LLM's attention degrades, function selection accuracy collapses, and massive JSON Schema definitions consume up to $30\text{--}50\%$ of the active context window on every turn.
> 
> **Code Mode** (pioneered by Cloudflare and implemented natively in OpenCode via `@opencode/codemode`) resolves tool fatigue by leveraging the capability LLMs perform best: **writing code**.
> 
> Instead of dumping hundreds of raw tool schemas into the model's function catalog, OpenCode:
> 1. Employs **Deferred Tool Discovery**: The model receives a compact catalog and dynamically queries available tools using an in-memory search function inside an execution sandbox.
> 2. Enables **Programmatic Tool Composition**: The model writes sandboxed JavaScript/TypeScript to call tools, compose multi-step API workflows, and programmatically filter/prune payloads before returning them to context.
> 3. Reduces token consumption by up to $\sim 95\%$, preventing massive, unfiltered API payloads from polluting working memory.

---

## 1. The Tool Fatigue Crisis in Large MCP Ecosystems

In traditional agent harnesses, every registered tool must have its complete JSON Schema (parameters, descriptions, types, enums) injected into the model's system prompt or tool catalog on every inference turn.

```
┌─────────────────────────────────────────────────────────────────────────────────────────┐
│                    THE TOOL FATIGUE PROBLEM (TRADITIONAL FUNCTION CALLING)               │
└─────────────────────────────────────────────────────────────────────────────────────────┘

  Connected MCP Servers:
  ├── GitHub MCP Server   ──▶  35 tools (repos, issues, PRs, commits, webhooks...)
  ├── Notion MCP Server   ──▶  18 tools (pages, databases, blocks, comments...)
  ├── Postgres MCP Server ──▶   8 tools (query, explain, schema, tables...)
  └── Stripe MCP Server   ──▶  42 tools (charges, customers, refunds, invoices...)

  TOTAL: 103 Tools in System Catalog!
  
  [ Context Tax ]
  103 Tools × ~350 tokens per schema = ~36,000 TOKENS CONSUMED BEFORE TURN 1!

  [ Attention Degradation ]
  Model suffers "Lost-in-the-Catalog" confusion:
  - Fails to select the correct tool.
  - Inverts argument parameters.
  - Incurs multiple slow API roundtrips to retrieve and filter paginated arrays.
```

```mermaid
graph TD
    subgraph TraditionalHarness["Traditional Tool Calling (High Friction)"]
        LLM1["Model Context"] -->|1. Turn 1: Call list_issues| H1["Harness"]
        H1 -->|Fetch 100 Issues| API1["GitHub API"]
        API1 -->|50KB Raw JSON Payload| H1
        H1 -->|Inject 50KB JSON into Context| LLM1
        LLM1 -->|2. Turn 2: LLM Manually Filters JSON| LLM1
    end

    subgraph CodeModeHarness["Code Mode Architecture (Low Friction)"]
        LLM2["Model Context"] -->|1. Single Turn: Execute Script| Sandbox["JS Sandbox (Code Mode)"]
        Sandbox -->|Call API + Filter in JS Memory| API2["GitHub API"]
        API2 -->|50KB Raw JSON| Sandbox
        Note over Sandbox: Array.filter() & map()<br/>Prunes 50KB down to 200 bytes!
        Sandbox -->|200 Bytes of Clean Data| LLM2
    end
    style TraditionalHarness fill:#fee,stroke:#c33
    style CodeModeHarness fill:#efe,stroke:#2a2
```

---

## 2. Code Mode Architecture: Tools as a JavaScript API

Rather than exposing tools as separate discrete JSON function schemas, OpenCode exposes them through a single meta-tool: `execute`. Inside `execute`, the model has access to a sandboxed JavaScript runtime where all MCP, built-in, and plugin tools are mapped as asynchronous JavaScript namespaces (`tools.<server>.<method>()`).

### 2.1 The Power of Programmatic Composition
Consider an agent searching for open GitHub issues without assignees in a repository.

#### Traditional Function Calling Approach:
The agent calls `list_issues`, waits for the response, ingests 50,000 tokens of raw user metadata, reaction counts, and HTML URLs, and then manually inspects the text.

#### Code Mode Approach:
The model generates a concise 7-line JavaScript snippet executed in the harness sandbox:

```javascript
// Executed in Code Mode Sandbox
const issuePage = await tools.github.list_issues({
  owner: "anomalyco",
  repo: "opencode",
  state: "OPEN",
  orderBy: "CREATED_AT",
  direction: "DESC",
  perPage: 20,
  fields: ["number", "title", "assignees"],
});

// Programmatic filtering BEFORE returning to LLM context
return issuePage.issues
  .filter((issue) => !issue.assignees?.length)
  .map((issue) => ({
    number: issue.number,
    title: issue.title,
  }));
```

### 2.2 Token Economics of Code Mode:
- **Raw API JSON Size:** $48,200\text{ bytes } (\approx 12,000\text{ tokens})$
- **Filtered JavaScript Output:** $320\text{ bytes } (\approx 80\text{ tokens})$
- **Net Token Reduction:** **$> 99.3\%$ savings on working context memory!**

---

## 3. Deferred Tool Discovery: Dynamic Schema Search

How does the model know which tools exist without dumping 100+ schemas into its system prompt?

**Answer:** **Deferred Tool Discovery.**

OpenCode routes MCP tools through Code Mode by default. The agent is provided with a compact meta-catalog and an internal discovery utility: `tools.search()`.

```mermaid
sequenceDiagram
    autonumber
    actor Dev as Developer
    participant LLM as Active Model
    participant Sandbox as Code Mode Sandbox
    participant Catalog as MCP Tool Registry

    Dev->>LLM: "Look up open issues authored by theodore-s-beers"
    Note over LLM: Model sees `execute` tool, but does not have the GitHub schema in context!
    LLM->>Sandbox: execute("return await tools.search('github issue')")
    Sandbox->>Catalog: In-memory fuzzy search across MCP schemas
    Catalog-->>Sandbox: Returns schema: `tools.github.list_issues({ ... })`
    Sandbox-->>LLM: Injects exact function signature on-demand
    
    Note over LLM: Model now possesses exact method signature!
    LLM->>Sandbox: execute("const res = await tools.github.list_issues(...); return res.filter(...)")
    Sandbox-->>LLM: Returns clean, pre-filtered issue list
    LLM-->>Dev: Presents findings without context pollution
```

### Advantages of Deferred Discovery:
1. **Zero Upfront Context Penalty:** Connecting 50 MCP servers costs essentially the same context footprint as connecting 1 server.
2. **Context Preservation:** Only tool signatures relevant to the active problem are retrieved into memory.
3. **Resilience to Schema Changes:** Dynamic discovery always reflects the live, running signature of the MCP server.

---

## 4. Configuring & Disabling Code Mode

OpenCode enables Code Mode across all MCP servers by default. However, certain legacy providers or specialized local tools may require traditional, native function-calling schemas (e.g., if a model has poor JavaScript generation capabilities).

Code Mode can be toggled per server in `opencode.jsonc`:

```jsonc
{
  "$schema": "https://opencode.ai/config.json",
  "mcp": {
    "servers": {
      // 1. Code Mode Enabled (Default): Tools routed through JS sandbox
      "github": {
        "type": "remote",
        "url": "https://api.githubcopilot.com/mcp/",
        "codemode": true
      },

      // 2. Code Mode Disabled: Exposes native tool calling schemas
      "specialized-local-linter": {
        "type": "stdio",
        "command": "uvx",
        "args": ["mcp-linter"],
        "codemode": false
      }
    }
  }
}
```

> [!TIP] When to Disable Code Mode
> Only disable Code Mode (`codemode: false`) if:
> 1. You are running a small or fine-tuned model that struggles to generate valid asynchronous JavaScript code.
> 2. The MCP server exposes only 1–2 simple tools, making the schema overhead negligible.
> 3. The server requires interactive terminal input or non-serializable streaming structures that cannot be piped through a JavaScript VM.

---

## 5. Production Case Study: The SlopLang Lexer Identifier Bug

To demonstrate real-world problem solving with MCP tools, consider the variable-naming bug in the `SlopLang` programming language.

### 5.1 The Defect Specification
A user reported that SlopLang programs fail with a syntax error when variable or function identifiers include numbers:

```slop
slop score2 = 40✨
score2 = score2 + 2✨
slop _player2score = score2 + 1✨
hallucinate(score2, _player2score)✨
clanker add2(value3) {
    ship value3 + 2✨
}
hallucinate(add2(5))✨
```

Expected output: `42 43 7`.  
Actual output: `SlopLang syntax error: unexpected token '2'`.

### 5.2 Root Cause Analysis in the Compiler Lexer
In classic compiler design, an identifier must follow standard lexical scoping rules:
- It **may** begin with an alphabetic character or underscore: `[a-zA-Z_]`.
- Subsequent characters **may** include digits: `[a-zA-Z0-9_]`.

The buggy lexer used an overly restrictive regex or character check that halted identifier scanning upon encountering the first digit:

```python
# ❌ BUGGY LEXER: Halts on digits inside identifier tokens
def is_ident_char(char: str) -> bool:
    return char.isalpha() or char == "_"

# ✅ CORRECT LEXER: Allows alphanumeric characters and underscores
def is_ident_start(char: str) -> bool:
    return char.isalpha() or char == "_"

def is_ident_continue(char: str) -> bool:
    return char.isalnum() or char == "_"
```

Or using regular expressions in the token scanner:
```python
# Regex invariant:
IDENTIFIER_REGEX = re.compile(r"^[a-zA-Z_][a-zA-Z0-9_]*$")
```

### 5.3 Automated Verification Pipeline
The test harness runs end-to-end interpreter assertions and regression test suites:

```bash
# 1. Pipe multi-line program to interpreter stdin
printf '%s\n' \
  'slop score2 = 40✨' \
  'score2 = score2 + 2✨' \
  'slop _player2score = score2 + 1✨' \
  'hallucinate(score2, _player2score)✨' \
  'clanker add2(value3) { ship value3 + 2✨ }' \
  'hallucinate(add2(5))✨' | uv run sloperator.py /dev/stdin 2>&1

# Expected stdout:
# 42 43 7

# Assert: Zero syntax errors
# 2. Run regression test suite
uv run pytest -q
# Asserts: All unit and AST tests pass (Exit code: 0)
```

---

## 6. Synthesis: The Complete MCP Architectural Stack

```
┌─────────────────────────────────────────────────────────────────────────────────────────┐
│                          THE COMPLETE MCP ARCHITECTURAL STACK                           │
└─────────────────────────────────────────────────────────────────────────────────────────┘

  [ Layer 4: Model Execution Layer ]
  Agent writes JavaScript inside Code Mode (`execute`)
  └── Searches tools dynamically (`tools.search()`)
  └── Composes and filters API responses before returning to context

  [ Layer 3: Client Transport Layer ]
  OpenCode Host coordinates 1:1 MCP Clients
  ├── Local: stdio process pipes (IPC)
  └── Remote: Streamable HTTP over TLS 1.3

  [ Layer 2: Authentication & Security Gate ]
  ├── Zero-Config OAuth via RFC 7591 Dynamic Client Registration (Notion)
  ├── Static Pre-Registered OAuth App Credentials via `{env:...}` (GitHub)
  └── Token isolation in developer keyring (`~/.config/opencode/auth.json`)

  [ Layer 1: Protocol & Data Layer ]
  JSON-RPC 2.0 Specification
  └── Tools (Callable actions)
  └── Resources (Passive contextual data)
  └── Prompts (Reusable templates)
```
