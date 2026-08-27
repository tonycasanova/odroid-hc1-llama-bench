## 🛑 Why the Wall Was Hit

### 1. Calculated Memory Bandwidth Limit

During token generation (`tg128`), every output token requires loading the entire model's weights into memory once.

* **Model Size:** `462.96 MiB`
* **Token Rate:** `~5.73 tokens/sec`
* **Actual Memory Throughput:**

$$462.96\text{ MiB} \times 5.73\text{ t/s} \approx 2.65\text{ GB/s}$$

> [!NOTE]
> The **ODROID-HC1** (Samsung Exynos 5422) features dual-channel 32-bit LPDDR3 @ 933 MHz, which caps out near **2.8 to 3.0 GB/s** of real-world effective memory bandwidth under multi-core loads. 
> 
> **Current Utilization:** Running at **~90–95%** of the absolute hardware bandwidth ceiling.

---

### 2. 32-bit ARM Architecture Limitations

Modern ARM architectures leverage hardware matrix-multiplication extensions to drastically reduce CPU cycle overhead:

* **ARMv8.2+ (64-bit):** Uses specialized vector instructions such as `DOTPROD` (Dot Product) and `I8MM` (Int8 Matrix Multiply) to accelerate quantized tensor math.
* **Cortex-A15 (ARMv7-A / 32-bit):** Lacks these specialized matrix instructions, relying entirely on standard **128-bit NEON SIMD** routines. 

Execution is already running as fast as the physical hardware and instruction set allow.
