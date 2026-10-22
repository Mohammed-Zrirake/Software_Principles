#AICoding #LLM #Providers #Models #OpenCode #SystemArchitecture #DevOps #AIInfrastructure

> [!abstract] Executive Summary
> In casual conversation, developers frequently conflate **AI Models** with **AI Providers** (e.g., *"Claude is better than Gemini"*). In real-world software and agentic engineering, this confusion leads to poor architectural decisions, mismatched SLAs, and unexpected billing spikes.
> 
> A **Model** is the mathematical neural network artifact (the trained weights) that determines raw intelligence, reasoning depth, instruction fidelity, and context window limits. A **Provider** is the cloud infrastructure entity hosting that model, determining authentication, latency, rate limits, uptime availability, compliance, and privacy guarantees. OpenCode formalizes this decoupling through the universal `<provider_id>/<model_id>` standard. Understanding this separation allows engineers to select the optimal model for reasoning while choosing the best provider for cost, performance, and data security.

---

## 1. The Core Distinction: Model vs. Provider

When an autonomous agent like OpenCode executes a task, it acts as an execution harness. The actual reasoning and decision-making engine is supplied by the specific **Model**, delivered across the network by a **Provider**.

```
┌──────────────────────────────────────────────────────────────────────────────────┐
│                         PROVIDER VS. MODEL ARCHITECTURE                          │
└──────────────────────────────────────────────────────────────────────────────────┘

   ┌────────────────────────────────────────────────────────────────────────┐
   │                           AI PROVIDER                                  │
   │               (The Hosting & Delivery Infrastructure)                  │
   ├────────────────────────────────────────────────────────────────────────┤
   │ • Authentication & API Keys            • Inference Latency & Region    │
   │ • Rate Limits (Requests/Tokens per Min)• Pricing & Billing Model       │
   │ • Uptime SLA & High Availability       • Data Retention & Privacy Terms│
   └───────────────────────────────────┬────────────────────────────────────┘
                                       │ hosts & serves
                                       ▼
   ┌────────────────────────────────────────────────────────────────────────┐
   │                            AI MODEL                                    │
   │               (The Neural Network Intelligence Core)                   │
   ├────────────────────────────────────────────────────────────────────────┤
   │ • Reasoning & Problem-Solving Ability  • Maximum Context Window Size   │
   │ • Code Generation & Syntax Precision   • Instruction Following Accuracy│
   │ • Mathematical & Algorithmic Logic     • Output Token Speed (TPS)      │
   └────────────────────────────────────────────────────────────────────────┘
```

### The Architectural Analogy
- A **Model** is like an **Engine Blueprint** (e.g., a V8 Turbo vs. an Electric Motor). It defines how much horsepower and torque the vehicle can produce.
- A **Provider** is like the **Dealership and Service Center** (e.g., Local Distributor vs. Direct Manufacturer). It determines warranty, purchase price, delivery time, refueling stations, and customer service.

---

## 2. Decoupling Model Creators from Model Providers

A common misconception among software engineers is assuming that a model can only be accessed through the vendor that trained it:

```
Common Myth:  Claude is only on Anthropic  |  Llama is only on Meta  |  GPT is only on OpenAI
Real World:   Claude on AWS Bedrock       |  Llama on Groq / Fireworks |  GPT on Azure OpenAI
```

The company that trains a foundation model is not necessarily the cloud provider you use to query it. Multiple cloud providers frequently host the exact same model weights:

```
                                  ┌──► anthropic/claude-sonnet-4-5 (Direct API)
                                  │
    claude-sonnet-4-5  ───────────┼──► opencode/claude-sonnet-4-5 (OpenCode Gateway)
    (Same model weights)          │
                                  └──► aws-bedrock/claude-sonnet-4-5 (Enterprise VPC)
```

### Why the Choice of Provider Matters:
Even when the underlying model weights are identical, changing providers dramatically alters the operational environment:

| Operational Vector | Direct Vendor (e.g., Anthropic / OpenAI) | Aggregator Gateway (e.g., OpenCode Console / OpenRouter) | Cloud Hyperscaler (e.g., AWS Bedrock / Azure OpenAI) |
| :--- | :--- | :--- | :--- |
| **Authentication** | Direct Vendor API Key | Unified Console Account / Free Tiers | IAM Roles / Service Principals |
| **Data Privacy** | Subject to vendor consumer terms | Zero-retention proxy routes | Guaranteed VPC boundary; no training on customer data |
| **Rate Limits** | Tier-based RPM/TPM throttles | Dynamic load-balancing across nodes | Provisioned Throughput Units (PTU) |
| **Billing** | Credit card per vendor | Single unified monthly invoice | Enterprise Cloud Billing / Committed Spend |

---

## 3. The OpenCode Standard: `<provider_id>/<model_id>`

To eliminate ambiguity, OpenCode requires fully-qualified model identifiers formatted as:

$$\text{Model Identifier} = \text{provider\_id} \,/\, \text{model\_id}$$

```
opencode/deepseek-v4.1-flash-free   ──► Provider: opencode   | Model: deepseek-v4.1-flash-free
openai/gpt-5                        ──► Provider: openai     | Model: gpt-5
anthropic/claude-sonnet-4-5         ──► Provider: anthropic  | Model: claude-sonnet-4-5
google/gemini-3.1-pro               ──► Provider: google     | Model: gemini-3.1-pro
```

When you select a model in OpenCode, you are configuring **both the intelligence layer and the infrastructure gateway**.

---

## 4. Configuring Default Project Models in `opencode.jsonc`

Teams should establish a standardized default model for their repository in `./opencode.jsonc`. This ensures all developers and CI/CD agents execute against the same model tier.

```jsonc
{
  "$schema": "https://opencode.ai/config.json",
  "model": "opencode/deepseek-v4.1-flash-free",
  "permissions": [
    {
      "action": "shell",
      "resource": "bootdev run *",
      "effect": "allow"
    }
  ]
}
```

### Schema & Pattern Validation:
The OpenCode JSON schema and automated validation test harnesses require the `"model"` field to match the regular expression:

$$\text{Regex Pattern: } \texttt{\textasciicircum[\textasciicircum/]+/.+\$}$$

This expression mandates:
1. `^[^/]+` — One or more characters identifying the provider (cannot start with a slash).
2. `/` — Exactly one forward slash delimiter.
3. `.+$` — One or more characters identifying the specific model.

```bash
# Verify model field conforms to the provider/model pattern using jq
cat opencode.jsonc | jq '.model | if type == "string" then test("^[^/]+/.+$") else false end'
```
*Expected Output:* `true`

---

## 5. Hands-On Verification: Setting the Default Model

To set the project default model and verify it through the test harness:

1. **Update `opencode.jsonc`:**
   Set the `"model"` key to your chosen provider/model identifier (e.g., `"opencode/deepseek-v4.1-flash-free"`).
2. **Review the Git Diff:**
   Verify that only the `"model"` property was added or modified:
   ```bash
   git diff opencode.jsonc
   ```
3. **Run Deterministic Verification:**
   Execute the Bootdev CLI verification suite:
   ```bash
   bootdev run -s 856a221d-7487-4d04-b2dc-8dd4848835a6
   ```
   The harness verifies:
   - `cat opencode.jsonc` exits with code `0`.
   - The jq pattern assertion evaluates to `true`.

---

## 6. Summary & Decision Framework

| Criterion | Model Focus (Intelligence) | Provider Focus (Infrastructure) |
| :--- | :--- | :--- |
| **Primary Question** | *"Is the LLM smart enough to solve this complex architectural refactor?"* | *"Can our servers reach this API reliably without rate-limit throttling or leaking data?"* |
| **Selection Factors** | Reasoning benchmarks, coding benchmarks (HumanEval, SWE-bench), context window. | Cost per 1M tokens, regional latency, SOC2/HIPAA compliance, zero-data-retention policy. |
| **Failure Modes** | Hallucinations, syntax bugs, infinite loops, failing edge-case logic. | HTTP 429 (Rate Limit Exceeded), HTTP 504 (Gateway Timeout), leaked credentials, unexpected bills. |
| **Config Syntax** | The right side of the slash: `model_id` | The left side of the slash: `provider_id` |
