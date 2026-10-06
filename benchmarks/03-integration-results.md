# 03 - Integrate: RAG pipeline run

Host `Windows-AMD64` · llama.cpp `b10488` ·
retrieval backend: **keyword overlap** · 3 queries

| Query | Contexts retrieved | embed (ms) | retrieve (ms) | llm (ms) | total (ms) |
|:--|--:|--:|--:|--:|--:|
| Why is goodput more useful than raw throughp... | goodput, paged, radix | 0.0 | 0.3 | 23737.9 | 23738.3 |
| What problem does PagedAttention actually so... | paged, radix, disagg | 0.0 | 0.1 | 16315.0 | 16315.3 |
| When does splitting prefill and decode help?... | disagg, radix, batching | 0.0 | 0.1 | 18677.9 | 18678.1 |

Mean per stage (ms): embed **0.0** · retrieve **0.2** ·
llm **19576.9** · total **19577.2**
Dominant stage: **llm** (100% of total)

## Answers returned

**Why is goodput more useful than raw throughput?**

> Based on the provided context, **Goodput** is more useful than raw throughput because it **ignores SLOs (Service Level Objectives)** when calculating throughput at saturation.

While raw throughput measures the total requests per second (TPOT) that met the targets, Goodput specifically counts only the requests that met the **TTFT** (Throughput Target for Functionality) and **TPOT** (Throughput Tar

**What problem does PagedAttention actually solve?**

> PagedAttention solves the problem of **internal fragmentation in GPU memory** caused by storing the Key-Value (KV) cache in non-contiguous pages.

By using non-contiguous pages, the model avoids the wasted space that would exist if all KV entries were stored contiguously in a single contiguous block of memory. This allows the model to utilize more of the available GPU memory for computation and in

**When does splitting prefill and decode help?**

> Based on the provided context, splitting prefill and decode helps when **prefill is compute-bound and decode is memory-bandwidth-bound**.

This is because the context explicitly states that disaggregated serving splits these operations:
1.  **Prefill** is compute-bound (requires significant CPU/GPU time).
2.  **Decode** is memory-bandwidth-bound (requires significant memory bandwidth).

By splitti


## N16-N19 components and interpretation

| Day | Piece | Status in this run |
|---|---|---|
| N16 | Cloud/IaC | Stub — localhost only; no cluster or IaC is connected |
| N17 | Data pipeline | Stub — documents are supplied in-memory; no ingestion job is connected |
| N18 | Lakehouse | Stub — no lakehouse is connected; documents live in `TOY_DOCS` |
| N19 | Vector + features | Stub — `TOY_DOCS` with keyword-overlap retrieval; no vector index or feature store |
| N20 | Serving | Real — requests go to the running `llama-server` on localhost |

The LLM stage dominating latency was expected on this CPU: it averaged 19,576.9 ms
of 19,577.2 ms total (about 100%). Embed and keyword retrieval together averaged
only 0.2 ms. To halve latency in this toy pipeline, I would first cap generated
tokens: decode is the largest part of each LLM call, and shorter answers should
reduce its duration. The trade-off is less complete answers.
