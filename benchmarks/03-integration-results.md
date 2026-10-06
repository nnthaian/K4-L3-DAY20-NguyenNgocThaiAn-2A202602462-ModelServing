# 03 - Integrate: RAG pipeline run

Host `Windows-AMD64` · llama.cpp `b10488` ·
retrieval backend: **keyword overlap** · 3 queries

| Query | Contexts retrieved | embed (ms) | retrieve (ms) | llm (ms) | total (ms) |
|:--|--:|--:|--:|--:|--:|
| Why is goodput more useful than raw throughp... | goodput, paged, radix | 0.0 | 0.1 | 8447.8 | 8448.1 |
| What problem does PagedAttention actually so... | paged, radix, disagg | 0.0 | 0.1 | 5054.1 | 5054.2 |
| When does splitting prefill and decode help?... | disagg, radix, batching | 0.0 | 0.1 | 8032.6 | 8032.7 |

Mean per stage (ms): embed **0.0** · retrieve **0.1** ·
llm **7178.2** · total **7178.3**
Dominant stage: **llm** (100% of total)

## Answers returned

**Why is goodput more useful than raw throughput?**

> Based on the context provided, **Goodput** is more useful than raw throughput because it addresses the fundamental limitation of raw throughput: **it ignores SLOs (Service Level Objectives) at saturation**.

Here is the breakdown of why this distinction matters:

1.  **Raw Throughput Ignores SLOs**: As stated in the context, "Throughput at saturation ignores SLOs." This means that if a system hits

**What problem does PagedAttention actually solve?**

> PagedAttention solves the problem of **internal fragmentation in GPU memory** caused by storing key-value pairs (KV cache) in non-contiguous pages.

By storing the cache in non-contiguous pages, the model avoids the wasted space that would occur if all KV data were packed into a single contiguous block of memory. This design allows the engine to remove this fragmentation, thereby maximizing availa

**When does splitting prefill and decode help?**

> Based on the provided context, splitting prefill and decode helps primarily during **continuous batching**.

The reasoning is as follows:
1.  **Context**: The context states that prefill and decode are compute-bound and memory-bandwidth-bound respectively.
2.  **Mechanism**: Continuous batching allows requests to join and leave the running batch each time they are decoded.
3.  **Benefit**: By spli


## Which N16-N19 pieces are real

N16 Cloud/IaC, N17 data pipeline, N18 lakehouse, and N19 vector/features are
stubbed in this run: the pipeline uses the shipped in-memory toy documents and
keyword-overlap retrieval. N20 serving is real because it calls the local
`llama-server` endpoint. The dominant LLM stage (7178.2 ms, 100% of total) was
expected. To halve latency, I would first reduce the output token budget or use
a faster quantization/model, because embed and retrieve take effectively zero time.

