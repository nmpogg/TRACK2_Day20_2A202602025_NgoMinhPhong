# 01 - Tune: thread-count sweep

Model `Qwen3.5-0.8B-Q4_K_M.gguf` � host `Windows-AMD64` � llama.cpp `b10488`
CPU: **2 physical � 4 logical** cores � `ngl=99` � metric `tg128`

| threads (-t) | tg128 (tok/s) | vs best |
|:--|--:|--:|
| 1 | 13.6 | 98% |
| 2 | 13.8 | 100% |
| 4 | 12.3 | 89% |

**Best**: `-t 2` at 13.8 tok/s
**Slowest tested**: `-t 4` at 12.3 tok/s (1.13x spread)
**Against the physical-core default** (`-t 2`, 13.8 tok/s): 1.00x

Use this in your run:

```bash
LAB_N_THREADS=2 make bench
```

## Your explanation

The knee is at 2 threads, which matches the number of physical CPU cores.
Performance increases slightly from 13.6 tok/s with 1 thread to the peak of 13.8 tok/s with 2 threads. Increasing the thread count to 4 reduces performance to 12.3 tok/s.

My CPU has 2 physical cores and 4 logical cores. The additional logical threads do not provide independent physical execution resources. At 4 threads, they compete for shared execution units, cache, and memory bandwidth, while also adding thread-scheduling and synchronization overhead. LLM decoding is often limited by memory bandwidth, so extra threads cannot necessarily improve its throughput.

The small difference between 1 and 2 threads also suggests that this workload does not scale strongly with CPU thread count on my machine. Therefore, 2 threads is the best setting for this system, while using all 4 logical threads causes resource contention and lowers performance.