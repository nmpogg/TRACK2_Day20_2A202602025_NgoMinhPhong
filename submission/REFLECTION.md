# Reflection - Day 20 Lab

**Name:** Ngô Minh Phong  
**Cohort:** 3  
**Date:** 2026-08-20

## 1. Hardware & runtime

- OS: Windows 10
- CPU: Intel Core i5-7300U @ 2.60GHz
- Cores: 2 physical / 4 logical
- RAM: 7.8 GB
- Accelerator: Vulkan device present
- llama.cpp: b10488 Vulkan prebuilt
- Model: Qwen3.5 0.8B (`qwen35-0.8b`)
- Quantization: Q4_K_M + UD-Q2_K_XL
- Run location: personal Windows laptop

Bootstrap created the virtual environment, installed dependencies, downloaded the
runtime and models, and generated the hardware and model manifests. Windows
PowerShell required UTF-8 output and ExecutionPolicy Bypass for the scripts.

## 2. Measurement

| Quantization | Size (GB) | Load (ms) | TTFT P50/P95 (ms) | TPOT P50/P95 (ms) | E2E P50/P95/P99 (ms) | Decode (tok/s) |
|---|--:|--:|--:|--:|--:|--:|
| Q4_K_M | 0.50 | 13157 | 2081 / 2379 | 73.9 / 86.6 | 6702 / 7599 / 7599 | 13.5 |
| UD-Q2_K_XL | 0.39 | 31364 | 3241 / 3440 | 1147.8 / 1239.9 | 74943 / 81554 / 81554 | 0.9 |

Q2 is only 0.11 GB smaller but decodes about 15x slower, so it is not worthwhile
on this laptop. Both models produced usable answers, but Q4 was far more practical.

## 3. Serving under load

| Users | RPS | P50 (ms) | P95 (ms) | P99 (ms) | Effective concurrency | Failures |
|--:|--:|--:|--:|--:|--:|--:|
| 10 | 0.09 | 43000 | 43000 | 43000 | 3.5 | 0.0% |
| 50 | 0.09 | 44000 | 44000 | 44000 | 4.0 | 0.0% |

Offered load increased 5x but delivered throughput changed only 0.98x; P95
changed 1.02x. At 50 users, effective concurrency was 4.0 with 4 slots, peak
busy-slot average was 3.28/4, and 46 requests were deferred. The server was
saturated at or below 50 users; extra latency was mainly queue time. I would
reduce max_tokens first to release slots sooner rather than increase parallelism
on a CPU with only two physical cores.

## 4. Integration

| Piece | Status |
|---|---|
| N16 Cloud/IaC | stub |
| N17 keyword-overlap retrieval | real |
| N18 Lakehouse | stub |
| N19 vector/embedding features | stub (keyword backend) |
| N20 llama-server serving | real |

Mean latency was embed 0.0 ms, retrieve 0.2 ms, LLM 13956.8 ms, total 13957.0
ms. The LLM stage was essentially 100% of latency. I would shorten generation
and optimize inference before retrieval.

## 5. The single change that mattered most

**Change:** tune threads from 4 to 2.

```text
before: 12.3 tok/s (-t 4)
after:  13.8 tok/s (-t 2)
speedup: 1.13x
```

Two threads match the two physical cores. Four threads make SMT siblings compete
for execution units, cache, and memory bandwidth; scheduling overhead exceeds the
benefit of extra logical threads.

## 6. Bonus

**Completed:** B2 GPU offload sweep.

```text
before: 12.9 tok/s (-ngl 99)
after: 14.9 tok/s (-ngl 0)
speedup: 1.16x
```

CPU-only execution was faster than full Vulkan offload. For this small model and
low-power Vulkan device, transfer and synchronization overhead outweighed the
accelerator benefit.

## 7. Most surprising result

The smaller Q2 model was dramatically slower than Q4, and full Vulkan offload
was slower than CPU-only execution. Model size and an accelerator flag alone did
not predict end-to-end performance.
