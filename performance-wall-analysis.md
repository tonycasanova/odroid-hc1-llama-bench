## 📊 ODROID-HC1 Baseline Performance

**Hardware:** Samsung Exynos 5422 (4x Cortex-A15 @ 2.0GHz pinned via `taskset -c 4-7`)  
**Model:** `Qwen2-0.5B-Instruct` (Q4_K_M — 462.96 MiB)  
**Flags:** `-O3 -ffast-math -fno-finite-math-only -ftree-vectorize -DGGML_LTO=ON`

| Model | Size | Params | Backend | Threads | Test | Speed (t/s) |
| :--- | ---: | ---: | :--- | ---: | :--- | ---: |
| Qwen2 0.5B (Q4_K_M) | 462.96 MiB | 630.17 M | CPU | 4 | pp512 | 15.75 ± 0.09 |
| Qwen2 0.5B (Q4_K_M) | 462.96 MiB | 630.17 M | CPU | 4 | tg128 | 5.73 ± 0.10 |

---

## 🛑 Hardware Bottleneck Analysis

### 1. Calculated Memory Bandwidth Limit

During token generation (`tg128`), every output token requires loading the entire model's weights into memory once.

* **Model Size:** `462.96 MiB`
* **Token Rate:** `~5.73 tokens/sec`
* **Actual Memory Throughput:**

```math
462.96\text{ MiB} \times 5.73\text{ t/s} \approx 2.65\text{ GB/s}


For token generation (tg128), memory bandwidth is your absolute hardware wall.



When running local LLMs, execution behavior splits into two distinct operational modes:
Mode	Main Bottleneck	Why?
Prompt Processing (pp512)	CPU / Compute Bound	The CPU processes many tokens simultaneously in parallel matrix multiplications. Core clock speeds, SIMD vector instructions, and compute units dictate performance.
Token Generation (tg128)	Memory Bandwidth Bound	Tokens are generated sequentially, one at a time. To predict the single next token, the CPU must stream the entire 462.96 MiB model file out of RAM and into the cache.
At 5.73 tokens per second, your ODROID-HC1's memory controller is reading that 463 MiB file from system RAM 5.73 times every second:
462.96 MiB×5.73 reads/sec=2,652.76 MiB/s≈2.65 GB/s
Because your dual-channel 32-bit LPDDR3 bus tops out around 2.8 to 3.0 GB/s in real-world conditions, the CPU cores spend a majority of their clock cycles simply waiting for memory lines to fetch over the bus.
How to Prove It
If you want to verify this without changing code:
Overclock/Underclock CPU: Dropping the CPU clock from 2.0 GHz to 1.6 GHz will drastically hurt pp512, but tg128 will barely move because the memory bus speed hasn't changed.
Shrink Model Size: If you run a smaller quantization like Q2_K (~290 MiB), the CPU has to pull 37% less data from RAM per token, and your tg128 output speed will jump proportionally to around ~8 t/s.
