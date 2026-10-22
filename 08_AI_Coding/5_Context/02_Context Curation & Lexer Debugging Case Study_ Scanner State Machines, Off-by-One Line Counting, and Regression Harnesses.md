#AICoding #Compilers #Lexer #ContextCuration #Debugging #Pytest #SystemDesign #OpenCode

> [!abstract] Executive Summary
> Context efficiency is not merely an optimization for token billing; it is a prerequisite for precise software debugging. When tasking an autonomous agent with resolving subtle compiler bugs—such as off-by-one line counting errors in lexical analysis—injecting extraneous project modules pollutes the agent's attention window and increases the likelihood of hallucinated fixes.
> 
> This note analyzes a classic **Lexer Line-Counting State Flaw** in Sloperator: comments starting with em-dashes (`—`) consuming newline characters without incrementing internal line-pointer state. We examine how targeted context ingestion enables the agent to pinpoint the state machine defect, patch `sloperator/lexer/scanner.py`, construct defensive regression test assertions in `tests/test_lexer.py`, and validate behavior via Unix pipeline streams and automated test harnesses.

---

## 1. Targeted Context Ingestion: Surgical vs. Monolithic Debugging

When a developer encounters a compiler diagnostic error such as an incorrect line number in a syntax error report, the instinct of a "context hoarder" is to tell the agent: *"Fix the bug in the lexer and read all files in `sloperator/`."*

This dumps thousands of unnecessary tokens into context: AST nodes, parser grammar definitions, bytecode emitters, and runtime interpreters.

```
┌────────────────────────────────────────────────────────────────────────────────────────┐
│                        SURGICAL CONTEXT CURATION FOR COMPILER BUGS                     │
└────────────────────────────────────────────────────────────────────────────────────────┘

    [ Bloated Context (Monolithic) ]
    ├── AST Node Definitions (12 files) ──────> Irrelevant to token scanning
    ├── Parser Grammar Rules (8 files) ───────> Irrelevant to character streaming
    ├── Bytecode Generator (5 files) ─────────> Irrelevant to lexical state
    └── Result: High latency, diluted attention, potential hallucination in parser.

    [ Surgical Context (High-Density) ]
    ├── Target Source: `sloperator/lexer/scanner.py` (The state machine)
    ├── Target Test:   `tests/test_lexer.py` (The regression assertion)
    └── Result: Immediate, isolated root-cause identification and zero side-effects.
```

---

## 2. Anatomy of the Bug: Scanner State & Comment Line-Tracking

In compiler frontends, the **Scanner (Lexer)** converts raw source text streams into discrete **Tokens**. The scanner maintains internal pointer coordinates:
- `start`: Byte offset of the current lexeme being accumulated.
- `current`: Byte offset of the character currently being inspected.
- `line`: Physical source line index (1-indexed) used for error reporting and debug symbols.

### The Failure Scenario:
Consider the following SlopLang source snippet:

```sloplang
— a comment
@
```

SlopLang syntax defines comments with an em-dash (`—`) or standard comment prefix running until the end of the line. The character `@` is an invalid token in SlopLang. 

- **Expected Behavior:** An unexpected character error at **line 2**.
- **Observed Bug:** The lexer reports `SlopLang lexical error at line 1: unexpected character '@'`.

### Testing Multiple Comments & Blank Lines:
```sloplang
— first

— third
@
```
- Line 1: `— first\n` (Comment)
- Line 2: `\n` (Empty blank line)
- Line 3: `— third\n` (Comment)
- Line 4: `@` (Invalid token)
- **Observed Bug:** The error is reported on line 1 or line 2 instead of line 4!

---

## 3. Root-Cause Analysis in `sloperator/lexer/scanner.py`

When inspecting `sloperator/lexer/scanner.py`, character dispatch typically handles newlines and comments in a `scan_token()` loop:

```python
# ==============================================================================
# THE BUGGY IMPLEMENTATION (Off-by-One Comment Termination)
# ==============================================================================

class Scanner:
    def __init__(self, source: str):
        self.source = source
        self.start = 0
        self.current = 0
        self.line = 1

    def scan_token(self):
        c = self.advance()
        if c == '\n':
            self.line += 1
            return
        elif c in (' ', '\r', '\t'):
            return
        elif c == '—':  # Comment delimiter
            # BUG: Consumes characters until newline, but either:
            # 1. Consumes '\n' without incrementing self.line, OR
            # 2. Skips past the newline delimiter inside the loop without tracking line state!
            while self.peek() != '\n' and not self.is_at_end():
                self.advance()
            # If the newline is consumed here without self.line += 1:
            if not self.is_at_end() and self.peek() == '\n':
                self.advance()  # Consumed '\n', but self.line was NEVER incremented!
```

### The State Machine Invariant:
**Every single consumption of a `\n` character anywhere in the scanner MUST increment `self.line` by 1.**

If comment scanning consumes the trailing newline delimiter to finalize the comment token, it **must** update the line coordinate. Alternatively, if the comment scanner leaves the newline character in the stream, the subsequent iteration of `scan_token()` must properly match and increment the line counter.

---

## 4. Remediation in `sloperator/lexer/scanner.py`

The hardened implementation guarantees that all newline transitions—whether encountered inside comments, whitespace skips, or string literals—reliably advance `self.line`:

```python
# ==============================================================================
# HARDENED SCANNER IMPLEMENTATION
# ==============================================================================

class Scanner:
    def __init__(self, source: str):
        self.source = source
        self.start = 0
        self.current = 0
        self.line = 1

    def is_at_end(self) -> bool:
        return self.current >= len(self.source)

    def advance(self) -> str:
        c = self.source[self.current]
        self.current += 1
        return c

    def peek(self) -> str:
        if self.is_at_end():
            return '\0'
        return self.source[self.current]

    def scan_token(self):
        c = self.advance()

        if c == ' ' or c == '\r' or c == '\t':
            # Ignore non-newline whitespace
            return

        elif c == '\n':
            # Standard newline outside comments
            self.line += 1
            return

        elif c == '—':
            # Consume comment content until newline or end of file
            while self.peek() != '\n' and not self.is_at_end():
                self.advance()
            # Do NOT consume the newline here; let the next scan_token iteration 
            # encounter '\n', ensuring self.line is incremented uniformly!
            return

        # Token identification & error dispatch
        elif c == '@':
            raise LexicalError(f"SlopLang lexical error at line {self.line}: unexpected character '@'")
        
        # ... remaining grammar tokens ...
```

> [!NOTE] Architectural Invariant
> By allowing `c == '—'` to break upon reaching `\n` without advancing past it, the newline naturally becomes the next character processed by `scan_token()`, triggering `self.line += 1` through the canonical whitespace branch.

---

## 5. Defensive Regression Test Design in `tests/test_lexer.py`

To permanently prevent regressions, test cases must assert both single-line and multiline comment/whitespace scenarios using `pytest`:

```python
# ==============================================================================
# tests/test_lexer.py (Regression Test Suite)
# ==============================================================================

import pytest
from sloperator.lexer.scanner import Scanner
from sloperator.lexer.exceptions import LexicalError

def test_comment_preserves_correct_error_line_number():
    """Ensure unexpected character immediately following a comment reports line 2."""
    source = "— a comment\n@"
    scanner = Scanner(source)
    
    with pytest.raises(LexicalError) as exc_info:
        while not scanner.is_at_end():
            scanner.scan_token()
            
    assert "SlopLang lexical error at line 2: unexpected character '@'" in str(exc_info.value)

def test_multiline_comments_and_blank_lines_reporting():
    """Ensure blank lines and multiple comments track line numbers accurately."""
    source = "— first\n\n— third\n@"
    scanner = Scanner(source)
    
    with pytest.raises(LexicalError) as exc_info:
        while not scanner.is_at_end():
            scanner.scan_token()
            
    assert "SlopLang lexical error at line 4: unexpected character '@'" in str(exc_info.value)
```

---

## 6. Verification Workflow & Automated Test Submission

The fix is validated across both Unix stdin streams and automated test runners:

```bash
# Test 1: Single comment followed by unexpected character on line 2
printf '%s\n' '— a comment' '@' | uv run sloperator.py /dev/stdin 2>&1
# Expected output:
# SlopLang lexical error at line 2: unexpected character '@'
# Expected exit code: 1

# Test 2: Multiple comments separated by blank line, error on line 4
printf '%s\n' '— first' '' '— third' '@' | uv run sloperator.py /dev/stdin 2>&1
# Expected output:
# SlopLang lexical error at line 4: unexpected character '@'
# Expected exit code: 1

# Test 3: Run comprehensive pytest regression suite
uv run pytest tests/test_lexer.py tests/test_lexer_regression.py -q
# Expected: All tests passing (exit code 0)

# Submit verified solution via Bootdev CLI
bootdev run 009798d6-7775-47f9-8bf4-641183e6101d
bootdev run -s 009798d6-7775-47f9-8bf4-641183e6101d
```

---

## 7. Context Management Summary: The Five Rules of Agent Debugging

```
┌────────────────────────────────────────────────────────────────────────────────────────┐
│                        THE 5 RULES OF CONTEXT-EFFICIENT DEBUGGING                      │
└────────────────────────────────────────────────────────────────────────────────────────┘

    1. Scope Ingestion: Ingest only the failing source file and its test file (@file).
    2. Reproduce First: Provide the exact input snippet and the expected error line.
    3. State Invariants: State the mechanical invariant explicitly ("Every \n must increment line").
    4. Assert Boundaries: Verify edge cases (empty blank lines, EOF without newline).
    5. Cleanse History: Once a bug is fixed and tests pass, start a new session for the next feature.
```
