#AICoding #SelfHosting #LocalLLM #VRAM #Quantization #LMStudio #Ollama #SystemArchitecture

> [!abstract] Executive Summary
> Relying on commercial cloud providers introduces third-party dependency, recurring token costs, and potential data privacy concerns. Self-hosting **open-weight Large Language Models** provides total data sovereignty, zero per-token billing, and offline independence. However, running local LLMs introduces a strict hardware barrier: **GPU Video RAM (VRAM)**.
> 
> Because an LLM must keep its active weights and Key-Value (KV) cache in high-bandwidth memory to achieve interactive tokens-per-second, memory sizing dictates what models you can physically run. Through **quantization** (reducing precision from FP16 to INT8/INT4), **Mixture-of-Experts (MoE)** routing, and local network inference servers (via **LM Studio**, **Ollama**, or **llama.cpp**), engineers can integrate self-hosted models directly into OpenCode. This guide details VRAM mathematics, configuration patterns, and the architectural trade-offs between local and hosted inference.

---

## 1. The Core Trade-Offs of Self-Hosting

Before investing in local inference infrastructure, software engineers must evaluate the operational trade-offs:

```
┌──────────────────────────────────────────────────────────────────────────────────┐
│                          LOCAL SELF-HOSTING VS. CLOUD APIS                       │
└──────────────────────────────────────────────────────────────────────────────────┘

   [ Local Self-Hosting (Ollama / LM Studio) ]       [ Cloud Providers (Anthropic / OpenAI) ]
   ┌─────────────────────────────────────────┐       ┌──────────────────────────────────────┐
   │ 🟢 100% Data Sovereignty & Offline Ready│       │ 🔴 Prompts sent across public network│
   │ 🟢 Zero per-token monthly billing       │       │ 🔴 Pay-as-you-go per million tokens  │
   │ 🟢 Full control over quantization/engine│       │ 🔴 Vendor rate limits & downtime SLAs│
   │ 🔴 High upfront hardware cost (GPUs)    │       │ 🟢 Zero hardware setup required      │
   │ 🔴 Limited by local VRAM capacity       │       │ 🟢 Instant access to massive SOTA    │
   │ 🔴 Inferior reasoning to frontier SOTA  │       │ 🟢 Blazing fast inference throughput │
   └─────────────────────────────────────────┘       └──────────────────────────────────────┘
```

---

## 2. The VRAM Bottleneck & Parameter Mathematics

The single most critical hardware constraint in local LLM inference is **VRAM (Video Random Access Memory)**. Running inference on standard system RAM via the CPU results in a severe memory-bandwidth bottleneck, causing generation speed to crawl from 40 tokens/sec down to 1–4 tokens/sec.

### The 16-Bit Precision Rule of Thumb:
At standard FP16 (16-bit floating point) or BF16 precision, each parameter occupies **2 bytes** of memory:

$$\text{Weight VRAM (FP16)} \approx \text{Parameter Count (in Billions)} \times 2\text{ GB}$$

- A **7 Billion** parameter model requires $\approx \mathbf{14\text{ GB}}$ of VRAM just to store the weights.
- A **32 Billion** parameter model requires $\approx \mathbf{64\text{ GB}}$ of VRAM.
- A **480 Billion** parameter model (like `Qwen3 Coder`) requires $\approx \mathbf{960\text{ GB}}$ of VRAM.

```
┌────────────────────────────────────────────────────────────────────────┐
│                        COMPONENTS OF VRAM DEMAND                       │
├────────────────────────────────────────────────────────────────────────┤
│ Total VRAM Required = Weight Memory + KV Cache + CUDA Overhead         │
├────────────────────────────────────────────────────────────────────────┤
│ 1. Static Weight Memory: Base parameters scaled by quantization bits.  │
│ 2. Dynamic KV Cache: Grows linearly with context window length.        │
│ 3. Runtime Buffers: CUDA workspace memory & activation tensors.        │
└────────────────────────────────────────────────────────────────────────┘
```

> [!WARNING] VRAM is Not a Static Number
> The memory required to run a model is not just the file size of the weights. As the agent ingests large context windows (32,000 to 128,000 tokens), the **Key-Value (KV) cache** consumes substantial additional gigabytes of VRAM. If context spills out of VRAM into system swap, execution freezes.

---

## 3. Quantization: Fitting Big Models on Small Hardware

To run capable models on consumer hardware, engineers use **Quantization**—compressing high-precision 16-bit floating-point weights down to 8-bit, 4-bit, or even 2-bit integers.

```
FP16 (16 bits / weight) ──► 2.0 Bytes / param (Baseline uncompressed)
INT8 (8 bits / weight)  ──► 1.0 Byte  / param (50% memory savings, negligible loss)
INT4 (4 bits / weight)  ──► 0.5 Bytes / param (75% memory savings, minor accuracy drop)
```

$$\text{Weight VRAM (4-bit)} \approx \text{Parameter Count (in Billions)} \times 0.5\text{ GB}$$

### Memory Sizing Table by Quantization Level:
| Model Size | FP16 (16-bit) | INT8 / Q8 (8-bit) | INT4 / Q4_K_M (4-bit) | Minimum Hardware Tier |
| :--- | :--- | :--- | :--- | :--- |
| **8B Parameters** | $\approx 16\text{ GB}$ | $\approx 8.5\text{ GB}$ | $\approx 5.5\text{ GB}$ | Consumer GPU (RTX 4060 8GB / Mac 16GB) |
| **14B Parameters** | $\approx 28\text{ GB}$ | $\approx 15\text{ GB}$ | $\approx 9.5\text{ GB}$ | Mid-tier GPU (RTX 4070 12GB / RTX 4080 16GB) |
| **32B Parameters** | $\approx 64\text{ GB}$ | $\approx 34\text{ GB}$ | $\approx 20\text{ GB}$ | High-end GPU (RTX 3090/4090 24GB / Mac 36GB) |
| **70B Parameters** | $\approx 140\text{ GB}$ | $\approx 75\text{ GB}$ | $\approx 42\text{ GB}$ | Dual RTX 3090s (48GB) / Mac Studio (64GB+) |

### Dense Models vs. Mixture-of-Experts (MoE):
Modern open-weight models frequently use **MoE (Mixture of Experts)** architectures:
- **Kimi K2.6:** 1 Trillion total parameters, but only **32 Billion active** parameters per token.
- **Qwen3 Coder:** 480 Billion total parameters, but only **35 Billion active** parameters.

*The Catch:* While an MoE only routes compute through 32B–35B active parameters during inference (offering the speed of a smaller model), **all 480B–1T total parameters must reside in memory** simultaneously. Storing 1T parameters uncompressed would require 2 Terabytes of RAM, requiring massive quantization and multi-GPU clusters.

---

## 4. Connecting OpenCode to Local Inference Servers

Running local models directly on your primary development laptop often starves your IDE and compiler of RAM. The industry best practice is deploying a **Dedicated Local Network Inference Server**:

```
┌──────────────────────────────────────────────────────────────────────────────────┐
│                      LOCAL NETWORK INFERENCE ARCHITECTURE                        │
└──────────────────────────────────────────────────────────────────────────────────┘

   [ Developer Laptop ]                               [ Dedicated Inference Box ]
   - Thin dev client                                  - Dedicated Linux / Mac workstation
   - Runs OpenCode TUI / Web UI                       - High-VRAM GPUs (e.g., RTX 4090 / Mac Studio)
   - Zero GPU load on laptop                          - Runs LM Studio / Ollama / llama-server
   └────────────────┬─────────────────┘               └────────────────┬─────────────────┘
                    │                                                  │
                    │   HTTP REST API (OpenAI Compatible: Port 1234)   │
                    └──────────────────────────────────────────────────┘
```

### 1. Popular Local Inference Engines:
- **[LM Studio](https://lmstudio.ai/):** User-friendly desktop application that downloads GGUF models from Hugging Face and exposes an OpenAI-compatible REST server.
- **[Ollama](https://ollama.com/):** Lightweight, CLI-driven local model runner optimized for background service operation.
- **[llama.cpp / llama-server](https://github.com/ggml-org/llama.cpp):** The high-performance C++ inference engine underlying most local tooling.

### 2. Configuring OpenCode for Local Inference
In your project’s `./opencode.jsonc` (or global config), configure a custom provider pointing to your local inference server's base URL:

```jsonc
{
  "$schema": "https://opencode.ai/config.json",
  "providers": {
    "lmstudio": {
      "settings": {
        // Point OpenCode to your local GPU machine's IP on port 1234
        "baseURL": "http://192.168.1.150:1234/v1"
      }
    }
  },
  "model": "lmstudio/qwen3-coder-32b-instruct"
}
```

---

## 5. System Design Interview: Hardware vs. Cloud Trade-Offs

> **Interview Scenario:** *You are working on `Sloperator`, a public open-source project with no confidential or private data. You just traded a Yu-Gi-Oh deck for a laptop with 16 GB of unified RAM and no dedicated GPU. You have a reliable internet connection and a budget of \$20/month. Would you choose a local model or a hosted provider? Explain the trade-offs and describe one change that would reverse your choice.*

```
┌────────────────────────────────────────────────────────────────────────────────────────┐
│                           COMPREHENSIVE INTERVIEW ANSWER                               │
├────────────────────────────────────────────────────────────────────────────────────────┤
│ 1. The Decision:                                                                       │
│    Choose a HOSTED CLOUD PROVIDER.                                                     │
│                                                                                        │
│ 2. The Architectural Rationale:                                                        │
│    - Hardware Limitations: A laptop with 16 GB RAM and no dedicated GPU cannot run     │
│      capable coding models locally. 16 GB must be shared with the OS, browser, and     │
│      editor. The largest model that fits is a heavily quantized 7B/8B model (Q4).      │
│    - CPU Crawl: Without a GPU, inference runs entirely on the CPU, yielding dismal      │
│      token generation speeds (1–3 tokens/sec), high thermal throttling, and battery drain.│
│    - Project Privacy: The project is public with zero confidential data; cloud API      │
│      transmission poses zero security risk.                                            │
│    - Economic Value: $20/month on cloud APIs (or free tiers like DeepSeek Flash)        │
│      delivers millions of tokens of top-tier intelligence with blazing fast latency.   │
│                                                                                        │
│ 3. Conditions That Would Reverse the Choice (Switch to Local):                         │
│    - Condition A (Strict Data Privacy): The project transitions to highly sensitive,    │
│      confidential proprietary code under strict regulatory compliance (HIPAA, SOC2,     │
│      defense contracting) where transmitting code outside the device is illegal.       │
│    - Condition B (Loss of Connectivity): Transitioning to an environment with zero or   │
│      unreliable internet access (e.g., coding on flights, remote field environments).  │
│    - Condition C (Hardware Upgrade): Acquiring a high-end dedicated GPU workstation on  │
│      the local network (e.g., 24GB–48GB VRAM) capable of serving 32B+ models fast.     │
└────────────────────────────────────────────────────────────────────────────────────────┘
```

---

## 6. Summary & Hardware Sizing Guidelines

| Hardware Configuration | Max Practical Model Tier | Recommended Workflow |
| :--- | :--- | :--- |
| **Laptop: 16GB RAM, No GPU** | 7B / 8B Q4 (Slow on CPU) | **Hosted Cloud APIs** (DeepSeek Flash Free, Claude Haiku, GPT-5.6 Luna) |
| **Workstation: Single 8GB GPU** | 8B Q4 / Q8 (Fast on GPU) | Local 8B models for fast autocomplete; Cloud for complex agent refactors |
| **Workstation: 24GB GPU (RTX 4090)** | 32B Q4 / 14B Q8 | Excellent daily-driver local coding agent (e.g., `Qwen2.5-Coder-32B`) |
| **Server: Multi-GPU / Mac Studio (64GB+)**| 70B Q4 / MoE architectures | Complete enterprise privacy; capable of running near-frontier local models |
