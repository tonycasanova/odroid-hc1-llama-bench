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

Empirical performance benchmark on the ODROID-HC1 running **Qwen2 1B (Q4_K_Medium)** via `llama.cpp` (build `925e11799`):

| Execution Mode | Target Cores | Thread Count | Prompt Processing (pp512) | Text Generation (tg128) |
| :--- | :--- | :--- | :--- | :--- |
| **All Cores (Unbound)** | Cores 0–7 | `-t 4` | 4.48 t/s | 2.42 t/s |
| **Cortex-A15 (Fast)** | **Cores 4–7** | **`-t 4`** | **4.48 ± 0.04 t/s** | **2.49 ± 0.00 t/s** *(Optimal)* |
| **Cortex-A7 (Slow)** | Cores 0–3 | `-t 4` | 1.80 ± 0.00 t/s | 0.88 ± 0.00 t/s |

After Tuning:

odroid@odroid:~/llama.cpp$ taskset -c 4-7 ./build/bin/llama-bench -m qwen0.5b.gguf -p 512 -n 128 -t 4
| model                          |       size |     params | backend    | threads |            test |                  t/s |
| ------------------------------ | ---------: | ---------: | ---------- | ------: | --------------: | -------------------: |
| qwen2 1B Q4_K - Medium         | 462.96 MiB |   630.17 M | CPU        |       4 |           pp512 |         15.75 ± 0.09 |
| qwen2 1B Q4_K - Medium         | 462.96 MiB |   630.17 M | CPU        |       4 |           tg128 |          5.73 ± 0.10 |

build: 925e11799 (10642)


The tiny +1% bump in tg128 (from 5.67 to 5.73 t/s) indicates that compilation flags and static optimization are no longer your bottleneck. Your CPU is actively waiting on the hardware memory bus.
Why the Wall Was Hit
Calculated Memory Bandwidth Limit:
Model Size: 462.96 MiB
Token Rate: ~5.73 tokens/sec
Actual Memory Throughput: 462.96 MiB×5.73 t/s≈2.65 GB/s
The ODROID-HC1's Exynos 5422 features dual-channel 32-bit LPDDR3 @ 933 MHz, which caps out near 2.8 to 3.0 GB/s of real-world effective memory bandwidth under open multi-core loads. You are running at ~90-95% of the absolute hardware ceiling.
The 32-bit ARM Limitation:
Modern 64-bit ARM CPUs (ARMv8.2+) use specialized hardware instructions like DOTPROD and I8MM (int8 matrix multiply) to accelerate tensor math.
Because the Cortex-A15 is an ARMv7-A chip, it relies purely on standard 128-bit NEON SIMD routines. The math execution is as fast as the hardware allows.




> **Key Takeaway:** Pinning execution strictly to the Cortex-A15 fast core cluster yields a **~2.49× speedup** in prompt processing and a **~2.83× speedup** in text generation over the Cortex-A7 cores.


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
