#AICoding #ReasoningModels #ChainOfThought #ThinkingTokens #OpenCodeVariants #TestTimeCompute #FastMode

> [!abstract] Executive Summary
> In early foundation model research, practitioners observed a peculiar emergent phenomenon: instructing a model to *"think step-by-step"* or *"explain your reasoning out loud"* dramatically improved its success rate on complex mathematical, logical, and code-refactoring problems. This technique, initially termed **Chain-of-Thought (CoT)** prompting, has since evolved from an ad-hoc prompt engineering trick into a **first-class API primitive known as Test-Time Compute (Reasoning Effort)**.
> 
> Modern reasoning models burn hidden or visible "thinking tokens" before producing final code, allowing them to formulate hypotheses, simulate execution paths, verify AST constraints, and backtrack from faulty assumptions. OpenCode surfaces this capability through **Model Variants**, allowing developers to modulate reasoning effort (`low`, `medium`, `high`, `max`) dynamically via keyboard shortcuts (`Ctrl+T`), slash commands (`/thinking`), or declarative JSONC overrides. This guide explores the mechanics of test-time compute, variant configuration, provider Fast Modes, and practical edge-case debugging.

---

## 1. The Evolution of Test-Time Compute

Prior to reasoning models, Large Language Models operated under fixed computational budgets per output token: every token generated received roughly the same number of floating-point operations. For difficult algorithmic or syntactic challenges, this often resulted in immediate, confident hallucinations.

```
┌──────────────────────────────────────────────────────────────────────────────────┐
│                   STANDARD LLM VS. REASONING MODEL GENERATION                    │
└──────────────────────────────────────────────────────────────────────────────────┘

   [ Standard Autoregressive Model ]
   Input Prompt ──────────► [Direct Output Stream] (Fast, but prone to logic bugs)
   
   [ Reasoning Model with Test-Time Compute ]
   Input Prompt ──────────► ┌────────────────────────────────────────┐ ──► Verified Output
                            │ Hidden Chain-of-Thought Thinking Tokens│     (High accuracy,
                            │ • Explores edge cases & AST constraints│      handles math,
                            │ • Detects sign & truncation errors     │      zero-shot fixes)
                            │ • Backtracks if hypothesis is invalid  │
                            └────────────────────────────────────────┘
```

### Why "Thinking Out Loud" Works:
1. **Working Memory Expansion:** Transformers do not have an infinite internal scratchpad. Every reasoning token emitted into the context acts as physical working memory, allowing attention heads to attend over intermediate logical steps.
2. **Search and Backtracking:** Modern reasoning models are trained with Reinforcement Learning (RL) to evaluate alternative logic trees. If a proposed syntax tree leads to a syntax contradiction, the model backtracks and explores an alternate solution before emitting code.
3. **The Trade-Off:** Reasoning models are **slower (higher Time-to-First-Token)** and consume significantly more tokens (both reasoning tokens and output tokens count against billing or rate limits). However, they can solve difficult multi-step problems where standard models fail completely.

---

## 2. OpenCode Model Variants

Rather than forcing developers to manage cryptic API parameters like `reasoning_effort: "high"`, OpenCode abstracts reasoning depth into **Model Variants**. A variant is a named configuration profile associated with a model that defines its thinking budget and verbosity.

```
┌────────────────────────────────────────────────────────────────────────┐
│                        MODEL VARIANT CONTROLS                          │
├────────────────────────────────────────────────────────────────────────┤
│ • Keyboard Shortcut: `Ctrl+T` (Cycles through available variants)       │
│ • Slash Command:     `/thinking` (Opens interactive variant selector)  │
│ • Default Behavior:  Stick with default/recommended unless stuck       │
└────────────────────────────────────────────────────────────────────────┘
```

### Declarative Variant Overrides in `opencode.jsonc`:
Developers can define custom named variants in their configuration to tailor model behavior for specific workflows:

```jsonc
{
  "$schema": "https://opencode.ai/config.json",
  "providers": {
    "openai": {
      "models": {
        "gpt-5": {
          "variants": [
            {
              // High-thinking, verbose explanation for hard architectural refactoring
              "id": "poindexter",
              "settings": {
                "reasoningEffort": "high",
                "textVerbosity": "high"
              }
            },
            {
              // Fast, concise execution with minimal thinking overhead
              "id": "grug",
              "settings": {
                "reasoningEffort": "low",
                "textVerbosity": "low"
              }
            }
          ]
        }
      }
    }
  }
}
```

---

## 3. Fast Mode: Low-Latency Token Throughput

Both OpenAI and Anthropic offer a dedicated **"Fast Mode"** endpoint setting for selected frontier models.

- **Mechanism:** The provider allocates priority GPU compute clusters to process and stream output tokens at accelerated speeds (substantially higher tokens-per-second).
- **Pricing:** In pay-per-token API tiers, Fast Mode carries a premium price per million tokens.
- **The Subscription Sweet Spot:** If you are operating on a flat-rate subscription plan (such as Claude Max or ChatGPT Pro) and are not exhausting your usage tier, **Fast Mode provides faster agent turnaround with zero extra out-of-pocket cost**.

---

## 4. Hands-On Debugging: Negative Integer Division Truncation

To observe the power of high reasoning effort, consider an arithmetic edge-case bug in an interpreter (e.g., `Sloperator` implementing SlopLang).

### The Problem:
In SlopLang, the division operator `/` between two integers must **truncate toward zero** (consistent with C, Go, and Java). However, the initial implementation relied on Python's native integer division:

```python
# The Buggy Code in Sloperator
result = a // b
```

When evaluating:
```text
hallucinate(-7 / 2)✨
```
- **Buggy Output:** `-4` (Python's `//` operator performs **floor division**, rounding toward negative infinity: $\lfloor -3.5 \rfloor = -4$).
- **Expected Specification:** `-3` (Truncation toward zero discards the fractional part: $\text{trunc}(-3.5) = -3$).

```
┌────────────────────────────────────────────────────────────────────────┐
│                   FLOOR DIVISION VS. TRUNCATION TOWARD ZERO            │
├───────────────────────────┬────────────────────────────────────────────┤
│ Operation                 │ Mathematical Formula & Result              │
├───────────────────────────┼────────────────────────────────────────────┤
│ Python Floor (`-7 // 2`)  │ Rounds down toward -∞:  floor(-3.5) = -4  │
│ SlopLang Trunc (`-7 / 2`) │ Truncates toward zero:  int(-7 / 2) = -3   │
│ Python Floor (`7 // -2`)  │ Rounds down toward -∞:  floor(-3.5) = -4  │
│ SlopLang Trunc (`7 / -2`) │ Truncates toward zero:  int(7 / -2) = -3   │
└───────────────────────────┴────────────────────────────────────────────┘
```

### The Fix via High Reasoning:
When configured with high reasoning (`Ctrl+T` $\rightarrow$ `high`), the agent analyzes the full suite of edge cases, including division by negative denominators, zero numerators, and arbitrary-precision 64-bit integers:

```python
# The Correct Implementation (Truncating Toward Zero in Python)
result = int(a / b)  # or: math.trunc(a / b)
```

### Verification Against Comprehensive Edge Cases:
```bash
# 1. Single expression check
printf '%s\n' 'hallucinate(-7 / 2)✨' | uv run sloperator.py /dev/stdin
# Expected stdout: -3

# 2. Comprehensive sign and large integer edge-case array
printf '%s\n' \
  'hallucinate(7 / 2)✨' \
  'hallucinate(7 / -2)✨' \
  'hallucinate(-7 / -2)✨' \
  'hallucinate(-1 / 2)✨' \
  'hallucinate(0 / -2)✨' \
  'hallucinate(-8 / 2)✨' \
  'hallucinate(-100000000000000000003 / 2)✨' \
  | uv run sloperator.py /dev/stdin

# Expected JSON Output: [3, -3, 3, 0, 0, -4, -50000000000000000001]
```

### Submitting via Bootdev CLI:
```bash
bootdev run -s 565947ae-d096-44ac-bf84-47cb64cd8c08
```
*Expected Result:* All tests pass, division truncates toward zero across all sign variations, exit code `0`.

---

## 5. Summary & Variant Selection Guidelines

| Variant Setting | When to Use | Operational Impact |
| :--- | :--- | :--- |
| **`low` (or default)** | Routine coding, standard CRUD, formatting, documentation, simple unit tests. | Lowest latency, lowest token cost. |
| **`high` / `max`** | Hard mathematical logic, compiler/AST edge cases, concurrency races, security audits. | Spends extra thinking tokens exploring edge cases; solves problems standard models fail. |
| **Fast Mode** | Subscription tiers with ample quota where rapid feedback is prioritized. | Maximizes generation throughput (TPS) at higher unit cost. |
