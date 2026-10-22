#AICoding #PromptBox #DeveloperErgonomics #OpenCode #TerminalCLI #ContextAttachment #ShellMode

> [!abstract] Executive Summary
> Interacting with an autonomous coding agent through a terminal prompt box is far more sophisticated than typing plain text into a chat window. Modern developer velocity relies on **prompt box ergonomics** that minimize roundtrip latency, eliminate redundant tool execution turns, and streamline context injection.
> 
> OpenCode provides three primary ergonomic primitives: **Direct Context Attachment (`@`)**, which injects file contents directly into the initial prompt without wasting a `read` tool turn; **Direct Shell Mode (`!`)**, which executes local terminal commands without calling an LLM while seamlessly embedding their output into the conversation session; and **Advanced Input Shortcuts** (`Shift+Enter`, `Ctrl+J`, `Ctrl+X, E` for external editor handoff). Mastering these tools accelerates developer-agent interaction and optimizes context efficiency.

---

## 1. Direct Context Attachment with `@`

In standard agent loops, if an agent needs to understand a specification, it must spend an entire turn issuing a tool call:

```
Standard Flow: Developer asks question ──► Model calls `read("docs/spec.md")` ──► Model answers
(Cost: 2 LLM inference turns, higher latency, wasted API calls)
```

By typing `@` in the prompt box, OpenCode opens an interactive fuzzy-search file picker:

```text
Explain how functions are declared in @docs/spec.md and @examples/hello.slop
```

```
┌──────────────────────────────────────────────────────────────────────────────────┐
│                           INLINE CONTEXT INJECTION                               │
└──────────────────────────────────────────────────────────────────────────────────┘

   Developer: "Implement doubling logic according to @docs/spec.md"
   Harness:   Intercepts `@docs/spec.md` before sending to LLM
              Reads file contents directly from disk
              Appends text directly into Turn 1 user prompt
   Model:     Receives prompt + file contents in ONE single turn!
              Immediately implements solution without needing a `read` tool call.
```

### Key Considerations:
- **Zero-Turn Ingestion:** Eliminates the latency of waiting for the model to request and receive file contents.
- **Large File Truncation:** If an attached file exceeds safe token limits, OpenCode truncates it to protect the context window. For multi-thousand-line files, direct the agent to use targeted `grep` or specific line-range reads instead.

---

## 2. Direct Shell Mode (`!`): Zero-Token Command Injection

Developers frequently need to run quick terminal commands while coding: running a test, checking git status, or verifying a build. Typing `!` at the start of an empty prompt box enters **Shell Mode**:

```
!uv run sloperator.py examples/doubling.slop
```

```
┌──────────────────────────────────────────────────────────────────────────────────┐
│                   DIRECT SHELL MODE VS. AGENT SHELL TOOL                         │
└──────────────────────────────────────────────────────────────────────────────────┘

   [ Direct Shell Mode: `!command` ]                  [ Agent Shell Tool Call ]
   • Executed immediately by developer                • Model must plan & request execution
   • ZERO LLM tokens or API latency                   • Burns input/output reasoning tokens
   • Instant local terminal speed                     • Slower multi-turn roundtrip
   • Output SAVED to session context history!         • Output ingested via tool result
```

> [!TIP] The Session History Superpower
> When you run a command via `!`, OpenCode captures `stdout` and `stderr` and **appends it directly into the active session history**. Your very next prompt can reference the output naturally:
> ```text
> "Look at the test failure above from my previous run and fix the assertion in test_lexer.py"
> ```
> This allows developers to inject ground-truth compiler and test errors into the agent's context without spending a single token to generate the execution command.

---

## 3. Essential TUI Keyboard Shortcuts

Navigating complex multiline prompts in a terminal environment requires fluent command of OpenCode's input keybindings:

| Shortcut | Action | Operational Utility |
| :--- | :--- | :--- |
| **`Enter`** | Send Prompt / Steer | Submits prompt. If agent is currently executing, sends a mid-flight steering message. |
| **`Shift+Enter`** | Insert Newline | Formats multiline prompts separating instructions from data. |
| **`Ctrl+J`** | Insert Newline (Fallback) | Terminal fallback for terminals that do not transmit `Shift+Enter` escape codes. |
| **`Up` / `Down`** | Prompt History Recall | Navigates through previous prompt history when cursor is at line boundaries. |
| **`Ctrl+C`** | Clear Input / Exit | Clears current prompt buffer; if already empty, exits OpenCode cleanly. |
| **`Ctrl+X` then `E`** | External Editor Handoff | Opens current draft in `$VISUAL` or `$EDITOR` (e.g., Neovim, VS Code via `code --wait`). |
| **`Esc`** | Exit Modes / Cancel | Exits Shell Mode (`!`); pressing twice (`Esc Esc`) cancels running agent loops. |

### The External Editor Handoff (`Ctrl+X, E`):
When drafting large architectural refactoring prompts spanning multiple paragraphs, writing inside a single-line terminal prompt can be cumbersome. Pressing `Ctrl+X` followed by `E` spawns your configured text editor (Neovim, Vim, or VS Code):
- Edit your prompt with full syntax highlighting, spellcheck, and cursor control.
- Save and close the editor.
- OpenCode automatically ingests the saved draft directly back into the prompt box.

---

## 4. Clipboard & Long Paste Compression

When pasting massive stack traces or configuration dumps into terminal TUIs, traditional terminal emulators flood the screen with thousands of scrolling lines, obscuring the conversation.

### OpenCode Paste Management:
1. **Compact Visual Pill:** When pasting long text blocks, OpenCode visually collapses the input into a compact label (e.g., `[Pasted Text: 4.2 KB]`). The full text is preserved and transmitted to the LLM.
2. **Automatic Terminal Copy:** Selecting text with your mouse inside the OpenCode TUI automatically copies the selection to your system clipboard, accompanied by a *"Copied to clipboard"* toast notification.

---

## 5. Hands-On Workflow: The Doubling Function

Consider an assignment to create an arithmetic function in SlopLang that doubles an input integer.

### Step 1: Multiline Prompt with `@` File Ingestion
In the OpenCode prompt box, we attach references and format requirements across multiple lines using `Shift+Enter`:

```text
Read the language specifications in @docs/spec.md and syntax patterns in @examples/hello.slop.

Create a new file examples/doubling.slop that defines a function doubling its argument:
- Define: clanker double(number) { ship number * 2✨ }
- Call: hallucinate(double(13))✨
- Ensure running the script prints 26.
```

### Step 2: Direct Shell Verification with `!`
Once the agent creates `examples/doubling.slop`, verify execution locally using Direct Shell Mode without invoking the LLM:

```bash
!uv run sloperator.py examples/doubling.slop
```
*Output:* `26`

### Step 3: Prompt Recall with `Up` Arrow
Press the `Up` arrow key to recall your initial prompt. Verify that OpenCode maintains prompt history across sessions, allowing you to inspect, modify, or replay previous instructions without re-typing.

### Step 4: Deterministic Submission via Bootdev CLI
```bash
bootdev run -s 200ee9e9-0f53-4043-9140-1992ad5c6749
```
*Verification Checks:*
1. `uv run sloperator.py examples/doubling.slop` outputs `26`.
2. Regression test suite `uv run pytest tests/test_examples.py -q` passes with exit code `0`.

---

## 6. Summary Reference Card

```
┌────────────────────────────────────────────────────────────────────────┐
│                     PROMPT BOX PRODUCTIVITY MATRIX                     │
├────────────────────┬───────────────────────────────────────────────────┤
│ Context Injection  │ Type `@<path>` to inline files in Turn 1.         │
│ Direct Shell Run   │ Type `!<command>` to run local commands for $0.00. │
│ Multiline Editing  │ Use `Shift+Enter` or `Ctrl+J`.                    │
│ Power Drafting     │ Use `Ctrl+X, E` to open prompt in VS Code/Vim.    │
│ Prompt History     │ Use `Up` / `Down` arrows to replay prompts.       │
│ Runaway Interrupt  │ Use `Esc Esc` to cancel in-flight agent loops.    │
│ Mid-Flight Steer   │ Type message & press `Enter` while agent runs.    │
└────────────────────┴───────────────────────────────────────────────────┘
```
