#AICoding #Prompting #AgentControl #Steering #Canceling #OpenCode #ReActLoop #CostOptimization

> [!abstract] Executive Summary
> One of the fastest ways to exhaust API budgets and corrupt repository state is allowing an off-track agent loop to run unmonitored. While the ultimate goal of autonomous coding is to "tab away" and work on other tasks, unattended agents can enter compounding hallucination loops, misinterpret architectural requirements, or blindly implement flawed plans based on bad research.
> 
> Professional agent control relies on three real-time intervention mechanisms: **Hard Cancellation (`Esc Esc`)** to halt runaway loops while preserving context; **Mid-Flight Steering (`Enter`)** to inject forgotten constraints at step boundaries without losing work; and **Proactive Cancel Conditions** baked directly into prompt text to force human check-ins before implementation begins. Mastering these mechanisms prevents wasted tokens and keeps agent trajectories strictly aligned with engineering intent.

---

## 1. The Cost of Runaway Agent Loops

In an autonomous ReAct loop, every turn re-ingests the conversation history, workspace file contents, and tool logs into context. If an agent misinterprets a requirement on Turn 2, turns 3 through 10 will compound that error:

```
┌──────────────────────────────────────────────────────────────────────────────────┐
│                         THE COMPOUNDING ERROR SPIRAL                             │
└──────────────────────────────────────────────────────────────────────────────────┘

   Turn 1: Developer sends prompt with missing constraint
   Turn 2: Agent misinterprets library API ──► Edits wrong file
   Turn 3: Test runner fails ───────────────► Agent attempts fix in wrong file
   Turn 4: Linter fails ────────────────────► Agent creates unnecessary helper functions
   Turn 5: 150,000 tokens burned ──────────► Codebase corrupted with unneeded slop
```

> [!CAUTION] The Worst Response to an Off-Track Agent
> Waiting for a broken agent run to finish before reacting burns tokens unnecessarily. Similarly, terminating the terminal process and restarting from scratch destroys your accumulated context. **The correct response is immediate intervention: cancel, adjust, or steer.**

---

## 2. Hard Cancellation: The `Esc Esc` Pattern

When an agent's real-time reasoning stream or tool choices show it has misunderstood the objective, developers can trigger an immediate hard interrupt:

```
┌────────────────────────────────────────────────────────────────────────┐
│                        HARD CANCELLATION RULES                         │
├────────────────────────────────────────────────────────────────────────┤
│ • Keystroke:          Press `Esc` twice (`Esc Esc`) in the TUI         │
│ • Execution State:    Immediately aborts active tool calls & streaming │
│ • Conversation State: PRESERVED (Conversation history remains intact)  │
│ • Filesystem State:   PRESERVED (Does not revert completed file edits) │
└────────────────────────────────────────────────────────────────────────┘
```

### The Post-Cancel Recovery Workflow:
1. **Press `Esc Esc`** to halt execution.
2. **Inspect the Git Status / Diff** (`git diff` or TUI diff pane) to see what files were touched.
3. **Send a Targeted Corrective Prompt:** You do not need to re-explain the whole project; simply state the correction:
   ```text
   "Stop editing parser.py. The issue is in the string literal scanner in lexer.py. 
   Revert your changes to parser.py and focus strictly on lexer.py."
   ```

### The "10-Second Observation Rule"
While agents exist to free engineers from manual typing, stepping away the millisecond a prompt is dispatched on a complex task is risky. Spending **10 seconds observing the agent's initial tool calls** (e.g., verifying it opens `lexer.py` rather than unrelated services) confirms the trajectory is sound before tabbing away to other work.

---

## 3. Mid-Flight Steering: Injecting Constraints at Step Boundaries

Frequently, a developer dispatches a prompt and immediately realizes they omitted a crucial detail:
- *"I forgot to tell it not to touch the database migration files."*
- *"I forgot to tell it to truncate toward zero instead of floor division."*

Rather than canceling the run and re-starting, OpenCode supports **Mid-Flight Steering**:

```
┌──────────────────────────────────────────────────────────────────────────────────┐
│                       MID-FLIGHT STEERING INTERCEPTION                           │
└──────────────────────────────────────────────────────────────────────────────────┘

   Agent: [Running tool call: grep("error_codes")] ───┐
                                                      │
   Developer: Types "Make sure error codes are uppercase" and presses Enter
                                                      │
   Harness: Buffers message until Step Boundary ◄─────┘
   
   Step Boundary Reached: Tool call completes
   Harness: Injects steering message into context alongside tool results
   Agent: Seamlessly adapts trajectory without restarting or losing work!
```

```mermaid
sequenceDiagram
    autonumber
    actor Developer
    participant Harness as OpenCode Harness
    participant Agent as LLM Agent Loop
    participant Tool as System Tool (e.g., grep)

    Developer->>Agent: Prompt 1: "Refactor error handling"
    Agent->>Tool: Invokes grep("error")
    Note over Tool: Tool execution in progress...
    
    Developer->>Harness: Sends Steering Message: "Keep all error codes in UPPERCASE"
    Note over Harness: Buffers message; waits for active tool completion
    
    Tool-->>Harness: Returns grep matches
    Harness->>Agent: Injects Tool Results + Steering Message
    Note over Agent: Agent incorporates new constraint immediately
    Agent->>Developer: Executes refactor adhering to uppercase formatting
```

### Steering vs. Canceling:
- Use **Canceling (`Esc Esc`)** when the agent is actively executing destructive, incorrect, or runaway actions that must be stopped immediately.
- Use **Steering (`Enter` while running)** when the current research or tool execution is valid, but needs an additional constraint or refinement appended for the next step.

---

## 4. Proactive Defense: Baking Cancel Conditions into Prompts

The most effective way to avoid runaway loops is **declarative prompt-level guardrails**. When a task begins with an exploratory or research phase, explicitly instruct the model to stop if research criteria are not met:

```
┌────────────────────────────────────────────────────────────────────────┐
│                   PROMPT CANCEL CONDITION PATTERNS                     │
├────────────────────────────────────────────────────────────────────────┤
│ ❌ Fragile / Unconstrained:                                            │
│ "Find an API endpoint that handles refunds and implement the handler." │
│ (Risk: If no endpoint exists, agent hallucinates one and writes code)  │
│                                                                        │
│ 🟢 Robust / Guarded:                                                   │
│ "Inspect internal/api/ to find an endpoint supporting refunds.         │
│  CANCEL CONDITION: If you cannot find an existing refund endpoint,     │
│  PAUSE immediately, do not write any code, and ask me what to do next."│
└────────────────────────────────────────────────────────────────────────┘
```

### Common Cancel Condition Triggers:
1. **Missing Dependencies:** *"If this requires updating our PostgreSQL major version, pause and ask before continuing."*
2. **Ambiguous Specs:** *"If the spec in `docs/` contradicts the test cases, stop and ask for clarification."*
3. **Destructive Scope:** *"If this refactor touches more than 3 files, stop and present a plan before editing."*

---

## 5. Summary & Intervention Matrix

| Situation | Action | Operational Impact |
| :--- | :--- | :--- |
| **Agent enters bad loop or edits wrong files** | **Press `Esc Esc` (Cancel)** | Halts execution immediately; preserves conversation history for quick correction. |
| **Forgot an edge case while agent is running** | **Type message & press `Enter` (Steer)** | Injected at next step boundary; agent pivots without restarting. |
| **Exploratory tasks with uncertain outcomes** | **Bake Cancel Condition into prompt** | Prevents agent from blindly implementing hallucinated plans when research fails. |
| **Launching large, multi-file refactors** | **Apply the 10-Second Observation Rule** | Verifies initial tool trajectory before tabbing away to other work. |
