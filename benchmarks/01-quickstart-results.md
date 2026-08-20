# 01 - Measure: latency baseline

Model `Qwen3.5 0.8B` � host `Windows-AMD64` � llama.cpp `b10488`
Settings: `threads=2` `ngl=99` `ctx=2048`
`max_tokens=64` � warm-up discarded
Completed requests: `Q4_K_M` 10/10 � `UD-Q2_K_XL` 10/10

| Quantization | Size (GB) | Load (ms) | TTFT P50/P95 (ms) | TPOT P50/P95 (ms) | E2E P50/P95/P99 (ms) | Decode (tok/s) |
|:--|--:|--:|--:|--:|--:|--:|
| Q4_K_M | 0.50 | 13157 | 2081 / 2379 | 73.9 / 86.6 | 6702 / 7599 / 7599 | 13.5 |
| UD-Q2_K_XL | 0.39 | 31364 | 3241 / 3440 | 1147.8 / 1239.9 | 74943 / 81554 / 81554 | 0.9 |

- **TTFT** = prefill. Short prompts keep it small; long-context RAG is where it explodes.
- **TPOT** = per-output-token decode cost, bounded by memory bandwidth. `decode tok/s = 1000 / TPOT_p50`.
- `UD-Q2_K_XL` decodes **15.00x SLOWER** than `Q4_K_M` here, despite being 0.11 GB smaller. That is a real result, not a mistake: fewer bits only buys speed when decode is limited by memory bandwidth. On a machine that is compute-limited instead � few cores, no GPU offload � the extra dequantization work of a heavily-quantized format can cost more than the bytes it saves. Say which case yours is.

## Your observation

On my Windows laptop, Q2 is smaller than Q4_K_M (0.39 GB vs 0.50 GB), but it is much slower. Q4_K_M achieves 13.5 tokens/s, while UD-Q2_K_XL achieves only 0.9 tokens/s. Therefore, the smaller Q2 model is not worth using for my machine because the storage saving is small compared with the large decoding slowdown. This result may be caused by the CPU, GPU backend, thread count, or limited optimization for this quantization format.

In terms of answer quality, both models were usable for this simple question, but Q4 was preferable because it produced the answer much faster with no obvious quality advantage from the smaller Q2 model.
