# ODROID-HC1 Local LLM Benchmarks & Execution Guide

Benchmarking edge AI inference performance on the ODROID-HC1 (Samsung Exynos 5422 SoC) using `llama.cpp` and small language models like Qwen 0.5B.

## Hardware Specifications ⚙️
* **Device:** ODROID-HC1 (Home Cloud One)
* **SoC:** Samsung Exynos 5422
* **Architecture:** 32-bit ARMv7l (`armv7l`)
* **CPU Config:** big.LITTLE Octa-Core
  * **Cortex-A15 (Big):** Cores 4–7 @ 2.0 GHz
  * **Cortex-A7 (LITTLE):** Cores 0–3 @ 1.4 GHz
* **RAM:** 2 GB LPDDR3
* **Cooling:** Integrated passive aluminum frame heatsink

---

## Benchmark Results 📊

Performance comparison showing why thread pinning to high-performance cores outperforms multi-cluster CPU threading on big.LITTLE architectures (`llama.cpp` / Qwen 0.5B Q4_K_M):

| Execution Mode | Target Cores | Thread Count | Prompt Processing (pp512) | Text Generation (tg128) |
| :--- | :--- | :--- | :--- | :--- |
| **All Cores (Unbound)** | Cores 0–7 | `-t 4` | 4.48 t/s | 2.42 t/s |
| **Cortex-A15 (Fast)** | **Cores 4–7** | **`-t 4`** | **4.49 t/s** | **2.55 t/s** *(Optimal)* |
| **Cortex-A7 (Slow)** | Cores 0–3 | `-t 4` | ~1.35 t/s | ~0.85 t/s |

> **Key takeaway:** Pinning matrix multiplication strictly to the Cortex-A15 cluster (`taskset -c 4-7`) prevents layer synchronization delays caused by the slower Cortex-A7 cores, unlocking maximum tokens-per-second.

---

## Production Execution Commands 💻

### 1. Optimal Text Completion
Runs inference using strictly the 4 Cortex-A15 cores:
```bash
taskset -c 4-7 ./build/bin/llama-completion \
  -m qwen0.5b.gguf \
  -p "The top 3 reasons to use a single-board computer are:" \
  -t 4 -no-cnv -n 128
