# 01 - Measure: latency baseline

Model `Qwen3.5 0.8B`, host `Windows-AMD64`, llama.cpp `b10488`
Settings: `threads=6` `ngl=0` `ctx=2048`
`max_tokens=64`, warm-up discarded
Completed requests: `Q4_K_M` 10/10 ï¿½ `UD-Q2_K_XL` 10/10

| Quantization | Size (GB) | Load (ms) | TTFT P50/P95 (ms) | TPOT P50/P95 (ms) | E2E P50/P95/P99 (ms) | Decode (tok/s) |
|:--|--:|--:|--:|--:|--:|--:|
| Q4_K_M | 0.50 | 2922 | 410 / 452 | 28.7 / 30.6 | 2219 / 2380 / 2380 | 34.8 |
| UD-Q2_K_XL | 0.39 | 2806 | 488 / 539 | 27.0 / 30.3 | 2174 / 2450 / 2450 | 37.1 |

- **TTFT** = prefill. Short prompts keep it small; long-context RAG is where it explodes.
- **TPOT** = per-output-token decode cost, bounded by memory bandwidth. `decode tok/s = 1000 / TPOT_p50`.
- `UD-Q2_K_XL` decodes **1.07x faster** than `Q4_K_M` here, for 0.11 GB less on disk.

## Your observation

UD-Q2_K_XL uses 0.11 GB less disk space and decodes about 1.07x faster than Q4_K_M
(37.1 versus 34.8 tok/s). However, Q2 has higher TTFT P50 (488 versus 410 ms)
and higher E2E P95 latency. With the same question, Q4 produced a clearer and more
coherent answer, while Q2 was still usable but slightly more repetitive and less precise.
On this machine, Q2 is worthwhile when speed and lower memory/storage usage matter
more than answer quality.

