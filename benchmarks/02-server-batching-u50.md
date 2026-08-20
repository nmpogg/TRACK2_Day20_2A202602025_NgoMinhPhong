# 02 - Continuous batching under load (u50)

Host `Windows-AMD64` · `--parallel 4` · 9 samples over
60s at 2.0s intervals · raw CSV: `02-server-metrics-u50.csv`

| Gauge | Peak observed |
|:--|--:|
| `n_busy_slots_per_decode` (avg/decode) | 3.28 of 4 slots (82%) |
| `requests_processing` | 4 |
| `requests_deferred` | 46 |
| `kv_cache_usage_ratio` | n/a — not exported by llama.cpp `b10488` |
| `tokens_predicted_total` (final) | 1295 |

Highest sampled value was **3.28 of 4** slots. Note this gauge is llama.cpp's *average* busy slots per decode step, so the number below is the highest average we sampled, not an instantaneous maximum batch width. A peak near 1 means
requests were served one at a time -- either the load was too light to overlap, or
they arrived too far apart. A peak approaching `--parallel` means the scheduler was
genuinely packing concurrent requests into shared decode steps.
`requests_deferred` went above zero: more requests arrived than there were slots, so some waited. That wait is the queue time in your P95.

## Your observation (required -- replace this line)

The peak observed batch width was **3.28 busy slots out of 4** (82%), while the
server reported 4 processing requests at the same time. This is consistent with
the effective concurrency of **4.0** in `02-server-results.md`: both indicate
that all four decode slots were occupied and that the server was operating near
its configured concurrency limit.

The values are not expected to be identical because they measure different
things. Effective concurrency is calculated from completed requests using
Little's Law and includes queued occupancy, whereas `n_busy_slots_per_decode`
is an average slot count during sampled decode steps. I trust the server gauge
for actual batching width and the effective-concurrency value as evidence of
queue occupancy; together with **46 deferred requests**, they show that the
server was saturated under the 50-user load.
