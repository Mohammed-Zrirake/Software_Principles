#AICoding #Permissions #OpenCode #Security #LeastPrivilege #Guardrails #DevOps

> [!abstract] Executive Summary
> In autonomous AI engineering, security cannot be treated as a binary switch. Granting an agent zero permissions renders it useless for automation, while granting it unrestricted execution access creates catastrophic operational risk. 
> 
> OpenCode implements a **declarative, tri-state permission engine** (`allow`, `ask`, `deny`) mapped across **actions** and **resources**. Understanding this engine requires mastering its core evaluation rule: **rules are evaluated in sequential order, and the LAST matching rule wins**. Consequently, configurations must place broad defaults first and narrow exceptions later. Furthermore, developers must understand critical security nuances, such as OpenCode's automatic `.env` protection and the subprocess execution inheritance of custom plugin tools.

---

## 1. The Anatomy of a Permission Rule

Permission policies are declared in the `permissions` array within `./opencode.jsonc` (for project-wide team rules) or `~/.config/opencode/opencode.jsonc` (for personal workstation defaults).

```jsonc
{
  "$schema": "https://opencode.ai/config.json",
  "permissions": [
    {
      "action": "shell",
      "resource": "*",
      "effect": "ask"
    },
    {
      "action": "edit",
      "resource": "*",
      "effect": "allow"
    },
    {
      "action": "webfetch",
      "resource": "*",
      "effect": "deny"
    }
  ]
}
```

```
┌────────────────────────────────────────────────────────────────────────┐
│                        PERMISSION RULE COMPONENTS                      │
├────────────────────────────────────────────────────────────────────────┤
│ • action:   The tool or category (e.g., `shell`, `edit`, `web*`)       │
│ • resource: The target path, command glob, URL, or subagent ID         │
│ • effect:   The policy outcome (`allow` | `ask` | `deny`)              │
└────────────────────────────────────────────────────────────────────────┘
```

### The Three Permission Effects:
1. **`allow`:** The tool executes immediately in the background without prompting the developer. Essential for fast, non-destructive test runners, linters, and custom tools.
2. **`ask`:** The agent pauses execution, displays the exact command or file modification in the terminal, and waits for explicit human confirmation (`y/n`).
3. **`deny`:** The action is hard-blocked. The harness immediately informs the model that execution was denied by policy.

---

## 2. Action Grouping & Wildcard Matching

A single permission `action` can govern multiple related tools simultaneously, and both `action` and `resource` support glob wildcards (`*`):

| Action Group | Tools & Capabilities Governed | Example Resource |
| :--- | :--- | :--- |
| **`edit`** | Governs `edit`, `write`, and `patch` tools. | `src/**`, `*.md` |
| **`shell`** | Governs terminal commands and shell execution. | `bootdev run *`, `git push *` |
| **`web*`** | Governs both `webfetch` and `websearch`. | `*`, `https://docs.rs/*` |
| **`subagent`** | Governs spawning and delegating to child agents. | `*`, `researcher` |
| **`<custom_tool>`** | Governs user-defined plugin tools (e.g., `run_slop`). | `*`, `examples/*.slop` |

```jsonc
// Example: Force user approval for all web-related operations
{
  "action": "web*",
  "resource": "*",
  "effect": "ask"
}
```

---

## 3. The Core Evaluation Rule: "Last Matching Rule Wins"

> [!IMPORTANT] Precedence Algorithm: Broad Defaults First, Narrow Exceptions Later
> OpenCode iterates sequentially through the `permissions` array from top to bottom. If multiple rules match an incoming tool call, **the LAST matching rule in the array overrides all previous matches**.

```
                       ┌─────────────────────────────────────┐
                       │ Tool Call: `shell("bootdev run 1")` │
                       └──────────────────┬──────────────────┘
                                          │
                                          ▼
  ┌───────────────────────────────────────────────────────────────────────────────┐
  │ Rule 1: {"action": "shell", "resource": "*", "effect": "ask"}                 │
  │ Status: MATCHES! ──► Current Effect: `ask`                                    │
  └───────────────────────────────────────┬───────────────────────────────────────┘
                                          │
                                          ▼
  ┌───────────────────────────────────────────────────────────────────────────────┐
  │ Rule 2: {"action": "shell", "resource": "bootdev run *", "effect": "allow"}   │
  │ Status: MATCHES! ──► Current Effect: `allow` (OVERRIDES RULE 1!)              │
  └───────────────────────────────────────┬───────────────────────────────────────┘
                                          │
                                          ▼
                                [FINAL EFFECT: ALLOW]
                            (Executes without prompt)
```

### The Inversion Anti-Pattern:
If an engineer places the broad rule *after* the specific rule:
```jsonc
// ❌ BROKEN ORDER: Broad rule overrides the specific exception!
[
  { "action": "shell", "resource": "bootdev run *", "effect": "allow" },
  { "action": "shell", "resource": "*", "effect": "ask" } // OVERWRITES ALLOW!
]
```
Under this broken configuration, `bootdev run *` will still prompt for approval because the trailing wildcard rule evaluated last.

---

## 4. Built-In Defaults & The `.env` Guardrail

OpenCode operates with sensible out-of-the-box defaults designed to balance speed with baseline safety:
- **General Tools:** Most tools (`read`, `edit`, `glob`, `grep`) default to `allow`.
- **External Directories (`external_directory`):** Accessing files located outside the folder where OpenCode was initialized defaults to `allow`.
- **The Sensitive File Exception (`.env`):** While `read` defaults to `allow` across general source code, **reading any `.env` file automatically defaults to `ask`**. This prevents agents from silently ingesting production database passwords, JWT secrets, or cloud API keys into external LLM context windows.

---

## 5. The Custom Tool Subprocess Caveat

> [!CAUTION] Custom Tools Bypass Shell Restrictions
> Granting `"allow"` to a custom plugin tool (such as `run_slop`) authorizes that tool **and whatever subprocesses or shell commands it executes internally**.

Consider this configuration:
```jsonc
{
  "permissions": [
    { "action": "shell", "resource": "*", "effect": "ask" },
    { "action": "run_slop", "resource": "*", "effect": "allow" }
  ]
}
```

```
┌──────────────────────────────────────────────────────────────────────────────────┐
│                         CUSTOM TOOL EXECUTION BOUNDARY                           │
└──────────────────────────────────────────────────────────────────────────────────┘

   1. Direct Shell Call: `uv run pytest`
      ──► Evaluated against "shell" action
      ──► Result: Prompts user with `ask` [Approval Required]

   2. Custom Tool Call: `run_slop("examples/hello.slop")`
      ──► Evaluated against "run_slop" action
      ──► Result: Evaluates to `allow` [No prompt]
      ──► Plugin execute() runs `spawnSync("uv", ["run", ...])`
      ──► Shell restrictions DO NOT intercept internal plugin subprocesses!
```

*Architectural Takeaway:* Permission policies govern the **agent-to-harness boundary**, not internal system calls executed within compiled plugin code.

---

## 6. Hands-On Configuration & CLI Testing

### Goal:
Configure OpenCode so that:
1. `run_slop` executes automatically without prompting (`allow`).
2. `bootdev run *` executes automatically without prompting (`allow`).
3. All other shell commands require explicit human confirmation (`ask`).

### Step 1: Updating `opencode.jsonc`
Place the broad `shell` rule first, followed by the specific `bootdev run *` and `run_slop` exceptions at the end of the `permissions` array:

```jsonc
{
  "$schema": "https://opencode.ai/config.json",
  "permissions": [
    {
      "action": "shell",
      "resource": "*",
      "effect": "ask"
    },
    {
      "action": "shell",
      "resource": "bootdev run *",
      "effect": "allow"
    },
    {
      "action": "run_slop",
      "resource": "*",
      "effect": "allow"
    }
  ]
}
```

### Step 2: Testing Interactive Behavior
- **Test 1:** Prompt the agent to run `uv run pytest -q`.
  *Expected Behavior:* OpenCode halts and prompts: *"The agent wants to run: uv run pytest -q. Allow? (y/n)"*. Approve this single request.
- **Test 2:** Prompt the agent to run `examples/hello.slop` using `run_slop`.
  *Expected Behavior:* The tool executes immediately without prompting.

### Step 3: Verification via Bootdev CLI
```bash
bootdev run -s f46c7f76-eff8-4e1f-9e27-6bf1a65db76b
```

The test runner verifies via `jq`:
1. The last three elements of `.permissions` contain rules for `shell: *` (`ask`), `shell: bootdev run *` (`allow`), and `run_slop: *` (`allow`).
2. The index of the broad `shell` rule is strictly less than the index of the specific `bootdev` rule:
   $$\text{index}(\text{shell: *}) < \text{index}(\text{shell: bootdev run *})$$

---

## 7. Summary & Best Practices

| Policy Principle | Implementation | Security Benefit |
| :--- | :--- | :--- |
| **Broad Defaults First** | Set `*` wildcards early in the array | Establishes a secure baseline policy. |
| **Exceptions Later** | Append specific command globs at the end | Ensures exceptions override broad restrictions without side effects. |
| **Gate Destructive Shells** | Set `git push *` and `rm *` to `"ask"` | Enforces human-in-the-loop oversight for permanent operations. |
| **Whitelist Safe Tools** | Set linters and test harnesses to `"allow"` | Enables autonomous self-correction without confirmation fatigue. |
| **Audit Plugin Code** | Review TypeScript plugin implementations | Prevents malicious or unconstrained subprocess execution inside custom tools. |
