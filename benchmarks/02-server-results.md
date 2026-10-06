# 02 - Serve: load test + saturation reading

Host `Windows-AMD64` · llama.cpp `b10488` ·
`--parallel 4` · `ctx=2048` · `threads=4` ·
`ngl=99`

| Users | Reqs | RPS | P50 (ms) | P95 (ms) | P99 (ms) | Eff. concurrency | Failures |
|:--|--:|--:|--:|--:|--:|--:|--:|
| 10 | 54 | 0.92 | 8900 | 12000 | 14000 | 8.4 | 0.0% |
| 50 | 59 | 1.03 | 27000 | 49000 | 51000 | 28.9 | 0.0% |

*Effective concurrency = RPS x average latency (Little's Law) -- how many requests were
really in flight, regardless of how many users locust simulated. It counts queued requests
too, so the occupancy/slot ratio can legitimately exceed 1.0; it is occupancy, not
utilisation. For true slot utilisation use the server's own gauges (`make metrics`).*

## What these two runs say

| Going from 10 to 50 users | |
|:--|--:|
| Offered load | 5x |
| Throughput actually delivered | **1.12x** (22% of linear) |
| P95 latency | **4.08x** |
| Effective concurrency at 50 users | 28.9 vs `--parallel 4` slots (occupancy/slot ratio 7.24) |

**Saturated.** Throughput delivered only 1.12x for 5x the offered load, and effective concurrency (28.9) is at or above all 4 decode slots. Saturation sets in somewhere at or below 50 users; the load you added beyond that point became queue time rather than throughput.

Throughput moved 1.12x while P95 moved 4.08x. That gap is the goodput argument: past saturation you buy throughput by spending latency, and if your SLO is a P95 target then the requests you added are no longer being served within it. (This lab does not fix an SLO number for you -- pick one in your write-up and state how much goodput you keep at it.)

## Your reading

The server is already queueing at 10 users because effective concurrency is 8.4,
above the four decode slots, and it is heavily saturated at 50 users. Increasing
offered load by 5x raises throughput only 1.12x while P95 rises 4.08x to 49 seconds;
the metrics run also records 3.91/4 busy slots and 46 deferred requests. The added
latency is therefore predominantly queue time, not extra per-request compute. To
raise goodput at a latency SLO, I would test `--parallel 8` first because deferred
requests show that the four-slot scheduler is the immediate constraint. Extra CPU
threads did not help in the tuning sweep, and Q2 quantization was slower, so neither
is as directly targeted. The trade-off is higher KV-cache memory use, which must be
checked against the 4 GB GPU.
