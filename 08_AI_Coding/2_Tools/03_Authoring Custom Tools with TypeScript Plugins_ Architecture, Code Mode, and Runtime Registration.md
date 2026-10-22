#AICoding #CustomTools #TypeScript #Plugins #OpenCode #Bun #CodeMode #ToolCalling #SoftwareEngineering

> [!abstract] Executive Summary
> While built-in tools (`read`, `grep`, `shell`, `edit`) cover standard filesystem and terminal interactions, real-world repositories frequently require **specialized, domain-specific actions**. Forcing an agent to repeatedly generate multi-line Bash or Python scripts to query an internal database, invoke a proprietary API, or execute a custom compiler wastes tokens, introduces syntax errors, and creates security risks.
> 
> OpenCode solves this through a **TypeScript Plugin Architecture**. Engineering teams can define custom tools in `.opencode/plugins/` using the `@opencode/plugin` SDK. Custom tools participate in the same ReAct execution loop, schema validation, and permission checks as native built-in tools. This guide deconstructs plugin anatomy, direct invocations versus **Code Mode**, runtime registration with **Bun**, hot-reloading mechanics via `/reload`, and practical implementation patterns.

---

## 1. When Should You Build a Custom Tool?

An agent equipped with a `shell` tool can technically execute arbitrary scripts. However, relying on raw shell commands for frequent operations is an architectural anti-pattern.

```
┌──────────────────────────────────────────────────────────────────────────────────┐
│                     RAW SHELL RE-INVENTION VS. CUSTOM TOOLS                      │
└──────────────────────────────────────────────────────────────────────────────────┘

   [ Ad-Hoc Shell Approach ]                          [ Custom Plugin Tool ]
   • Agent writes a 30-line Python script             • Agent emits clean JSON:
     to query an internal database                      `{"tool": "query_db", "args": {"id": 42}}`
   • Spends hundreds of tokens generating boilerplate • Fast, deterministic, single-turn execution
   • Risk of SQL injection or unquoted shell bugs     • Business logic & sanitization handled in TS
   • Must re-invent script logic on every task        • Consistent interface shared across team
```

### Ideal Scenarios for Custom Tools:
- **Repetitive Domain Actions:** The agent frequently needs to perform a specialized action with a consistent interface (e.g., executing a domain-specific language compiler, verifying a schema, running a local test fixture).
- **Safe Database Access:** Wrapping database interactions in read-only parameterized queries rather than exposing raw connection strings or dangerous write privileges to the model.
- **Internal Service APIs:** Querying staging environments, service discovery registries, or feature flag endpoints.
- **Complex Multi-Step Scripts:** Encapsulating a brittle multi-step shell pipeline behind a clean, strongly-typed JSON input schema.

---

## 2. OpenCode Plugin Architecture & Directory Layout

OpenCode plugins are native JavaScript or TypeScript modules loaded directly by the agent harness.

```
┌─────────────────────────────────────────────────────────────────────────────────────────┐
│                              PLUGIN RESOLUTION HIERARCHY                                │
└─────────────────────────────────────────────────────────────────────────────────────────┘

   [ 1. Project-Specific Plugins ]                     [ 2. Personal Global Plugins ]
   ./.opencode/plugins/*.ts                            ~/.config/opencode/plugins/*.ts
   ┌────────────────────────────────────────┐          ┌────────────────────────────────────────┐
   │ • Scope: Current repository            │          │ • Scope: All projects on workstation   │
   │ • Shared: Tracked in Git with team     │          │ • Personal: Developer-specific tools   │
   │ • Examples: Domain DSL runners, DB APIs│          │ • Examples: Jira lookup, personal notes│
   └────────────────────────────────────────┘          └────────────────────────────────────────┘
```

### Runtime Environment: Bun
OpenCode uses [Bun](https://bun.com) as its preferred TypeScript runtime and package manager due to its zero-configuration TypeScript compilation and instant startup speed.

#### Setting Up Plugin Dependencies:
```bash
# 1. Create the .opencode directory and install the plugin SDK
mkdir -p .opencode && cd .opencode
bun add @opencode/plugin@2.0.3

# 2. Add node_modules to .gitignore so installed packages are not committed to Git
echo "node_modules/" >> ../.gitignore
```

---

## 3. Dissecting `Plugin.define()` & The Tool Contract

Every custom tool plugin exports a default `Plugin.define()` object. Below is the canonical anatomy of a custom tool:

```typescript
import { Plugin } from "@opencode/plugin";

export default Plugin.define({
  // Unique identifier for the plugin
  id: "my-team.database-tool",

  setup: async (ctx) => {
    // Transform the registry to add or modify tools
    await ctx.tool.transform((tools) => {
      tools.add({
        // The tool name exposed to the LLM in its prompt registry
        name: "database",

        // Instructions telling the LLM WHEN and HOW to use the tool
        description: "Execute a read-only SQL query against the development database",

        // JSON Schema defining expected input arguments
        input: {
          type: "object",
          properties: {
            query: { 
              type: "string",
              description: "The SQL SELECT statement to execute" 
            },
          },
          required: ["query"],
          additionalProperties: false,
        },

        // Execution mode configuration
        options: { codemode: false },

        // The execution implementation running on the host OS
        execute: async (input, meta) => {
          const { query } = input as { query: string };

          // Optional metadata available to the tool
          // meta.sessionID, meta.agent, meta.messageID, meta.progress("Processing...")

          // Business logic / subprocess execution
          return {
            content: JSON.stringify({ status: "success", rows: [] }),
          };
        },
      });
    });
  },
});
```

### Critical Fields:
1. **`name`:** Normalized tool identifier. Avoid naming collisions with built-ins (`read`, `shell`, `grep`) unless explicitly intending to override them.
2. **`description`:** The model reads this description to decide when the tool is relevant. A clear, unambiguous description improves tool-selection accuracy.
3. **`input`:** A standard JSON Schema object. The LLM must conform to this schema when generating tool call arguments.
4. **`execute`:** The host function executed by the harness. It receives validated `input` and returns model-visible text content (plain text or serialized JSON).

---

## 4. Execution Modes: Direct Calling vs. Code Mode

OpenCode supports two operational modes for custom tools:

```
┌──────────────────────────────────────────────────────────────────────────────────┐
│                           DIRECT INVOCATION VS. CODE MODE                        │
└──────────────────────────────────────────────────────────────────────────────────┘

   [ Direct Invocation: options: { codemode: false } ]
   • Model emits a standard single-turn JSON tool call:
     `{"tool": "run_slop", "args": {"path": "examples/hello.slop"}}`
   • Harness runs the execute() function directly.
   • Ideal for straightforward, discrete actions.

   [ Code Mode: options: { codemode: true } (Default) ]
   • Model writes executable JavaScript/TypeScript that orchestrates multiple tools.
   • OpenCode evaluates the script in an isolated execution sandbox.
   • Allows the model to chain tools, filter outputs, and aggregate results in memory
     without streaming massive intermediate payloads back into context.
```

---

## 5. Hands-On Implementation: The `run_slop` Tool

Consider an engineering task where the agent must frequently execute programs written in a custom domain-specific language (`SlopLang`) via the project's `Sloperator` interpreter.

### Step 1: Writing the Plugin (`.opencode/plugins/run-slop.ts`)
Create `.opencode/plugins/run-slop.ts` using Bun and `@opencode/plugin`:

```typescript
import { Plugin } from "@opencode/plugin";
import { spawnSync } from "child_process";

export default Plugin.define({
  id: "sloperator.runner",
  setup: async (ctx) => {
    await ctx.tool.transform((tools) => {
      tools.add({
        name: "run_slop",
        description: "Run a SlopLang (.slop) program file using Sloperator and return its output",
        input: {
          type: "object",
          properties: {
            path: {
              type: "string",
              description: "The path to the .slop file to execute",
            },
          },
          required: ["path"],
          additionalProperties: false,
        },
        options: { codemode: false },
        execute: async (input) => {
          const { path } = input as { path: string };

          // Execute Sloperator using uv run --no-active to ensure stable venv paths
          const proc = spawnSync("uv", ["run", "--no-active", "sloperator.py", path], {
            encoding: "utf-8",
          });

          return {
            content: JSON.stringify({
              stdout: proc.stdout ?? "",
              stderr: proc.stderr ?? "",
              exitCode: proc.status ?? 0,
            }),
          };
        },
      });
    });
  },
});
```

### Step 2: Runtime Hot-Reloading (`/reload`)
If OpenCode is already running in an interactive session, you do not need to restart the process. Use the built-in reload command:

```text
/reload
```
OpenCode re-scans `.opencode/plugins/`, compiles the TypeScript source via Bun, and registers `run_slop` into the active model's tool registry immediately.

### Step 3: Invoking the Custom Tool
Prompt the agent:
> *"Use `run_slop` to execute `examples/hello.slop` and report its output."*

The model emits the tool call:
```json
{
  "tool": "run_slop",
  "args": {
    "path": "examples/hello.slop"
  }
}
```
The tool runs `uv run --no-active sloperator.py examples/hello.slop` and returns:
```json
{
  "stdout": "hello, slop\n",
  "stderr": "",
  "exitCode": 0
}
```

### Step 4: Verification via Bootdev CLI
```bash
bootdev run -s 8d8e80aa-2598-40ff-9659-3e2553025c14
```

The test runner validates:
1. **Plugin Export Validity:** Validates that `setup` is an exported function via `bun -e 'import plugin ...'`.
2. **Headless Tool Execution:** Runs `opencode run --format json` and validates via `jq` that the agent issued a `tool_use` event specifically targeting `run_slop` with completed status, matching `stdout: "hello, slop\n"`, and `exitCode: 0`.

---

## 6. Summary & Custom Tool Best Practices

| Best Practice | Implementation | Operational Rationale |
| :--- | :--- | :--- |
| **Commit to Repo** | Store under `.opencode/plugins/*.ts` | Shares custom tools with the entire development team via Git. |
| **Ignore Node Modules** | Add `.opencode/node_modules/` to `.gitignore` | Prevents vendor bloat from polluting Git history. |
| **Use TypeScript** | Leverage `Plugin.define` types | Catches schema and parameter typing errors before runtime. |
| **Descriptive Schema** | Provide rich `description` properties | Guides model tool selection and prevents incorrect argument hallucination. |
| **Hot Reloading** | Use `/reload` during plugin dev | Eliminates the need to terminate and restart active agent sessions. |
