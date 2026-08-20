# 03 - Integrate: RAG pipeline run

Host `Windows-AMD64` � llama.cpp `b10488` �
retrieval backend: **keyword overlap** � 3 queries

| Query | Contexts retrieved | embed (ms) | retrieve (ms) | llm (ms) | total (ms) |
|:--|--:|--:|--:|--:|--:|
| Why is goodput more useful than raw throughp... | goodput, paged, radix | 0.0 | 0.3 | 13420.7 | 13421.1 |
| What problem does PagedAttention actually so... | paged, radix, disagg | 0.0 | 0.1 | 8237.4 | 8237.6 |
| When does splitting prefill and decode help?... | disagg, radix, batching | 0.0 | 0.1 | 20212.2 | 20212.4 |

Mean per stage (ms): embed **0.0** � retrieve **0.2** �
llm **13956.8** � total **13957.0**
Dominant stage: **llm** (100% of total)

## Answers returned

**Why is goodput more useful than raw throughput?**

> Goodput is more useful than raw throughput because it focuses exclusively on requests that meet the Target Time-to-Fullness (TTFT) and Target Time-to-Poll (TPOT) targets, ensuring that the system is only utilized when it is ready to serve requests. In contrast, raw throughput ignores SLOs, meaning it counts requests that do not meet these targets or that occur at saturation, which can lead to over

**What problem does PagedAttention actually solve?**

> PagedAttention solves the problem of **internal fragmentation in GPU memory** by storing the KV cache in non-contiguous pages, which allows for more efficient memory usage compared to contiguous memory.

**When does splitting prefill and decode help?**

> Splitting prefill and decode helps primarily to **avoid memory bandwidth bottlenecks** during the decoding phase.

Here is the breakdown based on the context:
*   **Prefill is compute-bound:** It requires significant processing power (CPU/GPU) to generate the model's weights and parameters.
*   **Decode is memory-bandwidth-bound:** It requires reading data from memory into registers.
*   **The Pro


## Which N16-N19 pieces are real

- **N16: real.** The pipeline performs retrieval using the keyword-overlap backend
  and returns ranked context IDs for each query.
- **N17: stubbed.** The embedding stage reports `0.0 ms`, and the JSON identifies
  the backend as `keyword overlap`, so no real embedding model is being used.
- **N18: real.** The retrieved contexts are inserted into the prompt sent to the
  llama-server, which returns answers grounded in the selected context.
- **N19: real.** The pipeline records separate embedding, retrieval, LLM, and total
  timings, and includes server-side prompt/evaluation timing in the JSON output.

The dominant stage is the LLM stage, averaging **13,956.8 ms**, or essentially
100% of the **13,957.0 ms** total. This was expected because the local model is
running slowly on a 2-physical-core CPU, while keyword retrieval takes only
0.2 ms and the embedding stage is stubbed.

To halve the pipeline latency, I would attack the LLM stage first. The server
timings show that generation dominates each request, especially the third query,
which takes 20,212.2 ms and generates up to 200 tokens. I would first reduce the
maximum output length and optimize model execution/thread configuration before
optimizing retrieval, because retrieval contributes less than 1 ms and therefore
cannot materially reduce the total latency.