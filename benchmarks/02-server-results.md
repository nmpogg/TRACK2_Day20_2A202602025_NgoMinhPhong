# 02 - Serve: load test + saturation reading

Host `Windows-AMD64` � llama.cpp `b10488` �
`--parallel 4` � `ctx=2048` � `threads=2` �
`ngl=99`

| Users | Reqs | RPS | P50 (ms) | P95 (ms) | P99 (ms) | Eff. concurrency | Failures |
|:--|--:|--:|--:|--:|--:|--:|--:|
| 10 | 4 | 0.09 | 43000 | 43000 | 43000 | 3.5 | 0.0% |
| 50 | 4 | 0.09 | 44000 | 44000 | 44000 | 4.0 | 0.0% |

*Effective concurrency = RPS x average latency (Little's Law) -- how many requests were
really in flight, regardless of how many users locust simulated. It counts queued requests
too, so the occupancy/slot ratio can legitimately exceed 1.0; it is occupancy, not
utilisation. For true slot utilisation use the server's own gauges (`make metrics`).*

## What these two runs say

| Going from 10 to 50 users | |
|:--|--:|
| Offered load | 5x |
| Throughput actually delivered | **0.98x** (20% of linear) |
| P95 latency | **1.02x** |
| Effective concurrency at 50 users | 4.0 vs `--parallel 4` slots (occupancy/slot ratio 1.00) |

**Saturated.** Throughput delivered only 0.98x for 5x the offered load, and effective concurrency (4.0) is at or above all 4 decode slots. Saturation sets in somewhere at or below 50 users; the load you added beyond that point became queue time rather than throughput.

Throughput moved 0.98x while P95 moved 1.02x. That gap is the goodput argument: past saturation you buy throughput by spending latency, and if your SLO is a P95 target then the requests you added are no longer being served within it. (This lab does not fix an SLO number for you -- pick one in your write-up and state how much goodput you keep at it.)

> **Small sample.** Only 4 requests completed in the
> shorter run, so these percentiles are indicative rather than solid. Note also that
> locust averages only *completed* requests: when the run ends with requests still
> queued, effective concurrency is an **under**-estimate. Trust the throughput-scaling
> row over the concurrency row here, and run longer (`-t 3m`) if you want firmer numbers.

## Your reading

The server is already close to saturation at 10 users and is clearly saturated
before or at 50 users. The strongest evidence is that increasing the offered
load from 10 to 50 users (5x) did not increase throughput: delivered throughput
changed from 0.094 RPS to 0.092 RPS, only 0.98x. Meanwhile, P95 latency remained
very high at approximately 43-44 seconds.

At 50 users, effective concurrency reached 4.0, matching all four server slots.
The server metrics also recorded 4 processing requests and as many as 46 deferred
requests. Therefore, additional users were mostly added to the queue instead of
increasing completed request throughput. The cancellation messages at the end of
the test are also consistent with requests remaining queued when the load test
stopped.

For this machine, I would define an initial P95 SLO of 45 seconds. The measured
throughput within this SLO is only about 0.09 requests per second. To raise
goodput, I would first reduce the generation length (`max_tokens`) because decode
dominates request time and the machine produces tokens slowly. Shorter responses
would release each slot sooner, reduce queueing, and allow more requests to
complete within the latency SLO.

I would not increase `--parallel` first. All four existing slots are occupied,
46 requests were deferred, and the machine has only 2 physical CPU cores.
Adding more slots would increase contention and memory pressure without
necessarily increasing token throughput.