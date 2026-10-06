# 02 - Serve: load test + saturation reading

Host `Windows-AMD64` · llama.cpp `b10488` ·
`--parallel 4` · `ctx=2048` · `threads=6` ·
`ngl=0`

| Users | Reqs | RPS | P50 (ms) | P95 (ms) | P99 (ms) | Eff. concurrency | Failures |
|:--|--:|--:|--:|--:|--:|--:|--:|
| 10 | 39 | 0.69 | 12000 | 21000 | 21000 | 8.7 | 0.0% |
| 50 | 47 | 0.83 | 24000 | 50000 | 56000 | 21.7 | 0.0% |

*Effective concurrency = RPS x average latency (Little's Law) -- how many requests were
really in flight, regardless of how many users locust simulated. It counts queued requests
too, so the occupancy/slot ratio can legitimately exceed 1.0; it is occupancy, not
utilisation. For true slot utilisation use the server's own gauges (`make metrics`).*

## What these two runs say

| Going from 10 to 50 users | |
|:--|--:|
| Offered load | 5x |
| Throughput actually delivered | **1.19x** (24% of linear) |
| P95 latency | **2.38x** |
| Effective concurrency at 50 users | 21.7 vs `--parallel 4` slots (occupancy/slot ratio 5.42) |

**Saturated.** Throughput delivered only 1.19x for 5x the offered load, and effective concurrency (21.7) is at or above all 4 decode slots. Saturation sets in somewhere at or below 50 users; the load you added beyond that point became queue time rather than throughput.

Throughput moved 1.19x while P95 moved 2.38x. That gap is the goodput argument: past saturation you buy throughput by spending latency, and if your SLO is a P95 target then the requests you added are no longer being served within it. (This lab does not fix an SLO number for you -- pick one in your write-up and state how much goodput you keep at it.)

## Your reading

The server was saturated at or below 50 users. A 5x increase in offered load
produced only 1.19x more throughput, while P95 latency increased 2.38x.
The effective concurrency reached 21.7 versus only 4 decode slots, showing that
extra requests were waiting in the queue. To improve goodput at the SLO, I would
first test increasing `--parallel` from 4 to 6 or 8 and keep it only if P95 remains acceptable.

