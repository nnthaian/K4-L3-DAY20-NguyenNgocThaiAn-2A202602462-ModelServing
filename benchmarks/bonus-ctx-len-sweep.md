# Bonus - Context-length sweep (prefill cost)

Host `Windows-AMD64` Â· llama.cpp `b10488` Â·
`threads=6` `ngl=0` Â· RAM 23.4 GB

| Prompt tokens | Prefill (tok/s) | TTFT contribution (ms) | vs linear scaling |
|:--|--:|--:|--:|
| 256 | 160.7 | 1593.1 | 1.00x |
| 1024 | 143.7 | 7126.0 | 1.12x |
| 2048 | 148.2 | 13817.3 | 1.08x |

At 2048 tokens, prefill costs **13817 ms**, which is
**1.08x** linear scaling -- so on this hardware, over this range, prefill is
still growing **roughly linearly**, not quadratically.

That is the correct finding, not a failed experiment. Attention is O(N^2), but it is only
one term: the per-layer linear projections and MLP are O(N), and on a 2B-class model at
short prompts they dominate. The quadratic term only overtakes them once N gets large
enough. Your prefill cost is currently bounded by throughput, not by sequence length.

To find where it *does* bend, extend the grid:

```bash
.venv/bin/python bonus/sweeps/ctx-len-sweep.py --grid 1024,4096,8192,16384,32768
```

Watch the "vs linear" column: the first row that climbs meaningfully above 1.0 is where
attention starts to matter on your machine. Report that crossover point.

Either way, this is the number to remember when someone proposes stuffing more retrieved
context into a RAG prompt "because the context window allows it". Prefill is paid in full,
on every request, before the first token appears.

## Your finding

At 2048 tokens, prefill reached 13817.3 ms, compared with 1593.1 ms at 256
tokens. The vs-linear ratio was only 1.08x, so this range did not show a clear
quadratic bend; prefill was still roughly linear. However, the absolute TTFT cost
grew substantially, which means a RAG pipeline should limit retrieved chunks and
avoid filling the context window just because the model supports it.
