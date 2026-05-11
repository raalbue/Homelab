# Local LLM Setup

## Tools

- **Ollama** — CLI-based model runner, great for terminal and API access
- **LM Studio** — GUI-based, easy model browsing and chat interface
- **Hugging Face** — model repository, source for downloading GGUF/quantized models

---

## Hardware Profile

| Component | Spec |
|---|---|
| CPU | Intel i9 |
| RAM | 64GB |
| GPU | NVIDIA RTX 4090 |
| VRAM | 16GB |

---

## VRAM Sizing Guide

Q4_K_M quantization uses approximately **0.5 bytes per parameter**.

| Model Size | VRAM @ Q4_K_M | Fits in 4090? |
|---|---|---|
| 7B | ~4GB | Yes, fully |
| 14B | ~8GB | Yes, fully |
| 24–27B | ~13–15GB | Yes, fully |
| 32B | ~19GB | Minor CPU offload (~3GB to RAM) |
| 70B | ~40GB | No — heavy offload, slow |

With 64GB RAM, minor CPU offloading (32B models) has negligible speed impact on an i9.

---

## Recommended Models

### Fits Fully in VRAM (Fastest Inference)

| Model | VRAM @ Q4_K_M | Best For |
|---|---|---|
| **Gemma 3 27B** | ~15GB | Best fit for 4090 — excellent reasoning & coding |
| **Qwen2.5-14B** | ~8GB | Blazing fast, strong quality for its size |
| **Mistral Small 3.1 24B** | ~13GB | Reliable, great instruction following |

### Slightly Over VRAM (Minor CPU Offload, Still Fast)

| Model | VRAM @ Q4_K_M | Best For |
|---|---|---|
| **Qwen2.5-32B** | ~19GB | Top-tier general quality |
| **Qwen2.5-Coder-32B** | ~19GB | Best coding model for this hardware |

---

## Top Picks

- **General use**: Gemma 3 27B Q4_K_M — fits perfectly in 16GB VRAM, full GPU acceleration
- **Coding**: Qwen2.5-Coder-32B Q4_K_M — minor offload, best coding quality available at this tier

---

## Ollama Quick Start

```bash
# Pull recommended models
ollama pull gemma3:27b
ollama pull qwen2.5-coder:32b

# Run a model
ollama run gemma3:27b

# List installed models
ollama list
```

## LM Studio

1. Open LM Studio and go to the **Discover** tab
2. Search for the model name (e.g., `gemma-3-27b`)
3. Filter by `Q4_K_M` quantization
4. Download and load the model

---

## Quantization Reference

| Format | Size | Quality | Use When |
|---|---|---|---|
| Q4_K_M | Small | Good | Default — best balance |
| Q5_K_M | Medium | Better | More VRAM available |
| Q8_0 | Large | Near-lossless | Max quality, fits in VRAM |
| F16 | Largest | Full precision | Rarely practical locally |
