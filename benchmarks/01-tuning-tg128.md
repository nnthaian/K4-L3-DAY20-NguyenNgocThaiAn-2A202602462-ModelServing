# 01 - Tune: thread-count sweep

Model `Qwen3.5-0.8B-Q4_K_M.gguf` · host `Windows-AMD64` · llama.cpp `b10488`
CPU: **6 physical · 12 logical** cores · `ngl=0` · metric `tg128`

| threads (-t) | tg128 (tok/s) | vs best |
|:--|--:|--:|
| 1 | 18.3 | 53% |
| 3 | 30.7 | 90% |
| 6 | 34.2 | 100% |
| 12 | 32.3 | 94% |
| 24 | 21.7 | 63% |

**Best**: `-t 6` at 34.2 tok/s
**Slowest tested**: `-t 1` at 18.3 tok/s (1.87x spread)
**Against the physical-core default** (`-t 6`, 34.2 tok/s): 1.00x

Use this in your run:

```bash
LAB_N_THREADS=6 make bench
```

## Your explanation

The knee is at 6 threads, exactly the number of physical CPU cores, where tg128
peaks at 34.2 tok/s. Increasing to 12 threads uses logical cores but reduces
throughput to 32.3 tok/s because threads compete for shared execution resources,
cache, and memory bandwidth. At 24 threads the CPU is oversubscribed, so context
switching and contention reduce throughput further to 21.7 tok/s. Therefore the
physical-core setting is the best choice for this workload.

