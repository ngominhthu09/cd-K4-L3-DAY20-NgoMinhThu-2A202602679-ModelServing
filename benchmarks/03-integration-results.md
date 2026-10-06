# 03 - Integrate: RAG pipeline run

Host `Windows-AMD64` · llama.cpp `b10488` ·
retrieval backend: **keyword overlap** · 3 queries

| Query | Contexts retrieved | embed (ms) | retrieve (ms) | llm (ms) | total (ms) |
|:--|--:|--:|--:|--:|--:|
| Why is goodput more useful than raw throughp... | goodput, paged, radix | 0.0 | 0.1 | 7187.4 | 7187.6 |
| What problem does PagedAttention actually so... | paged, radix, disagg | 0.0 | 0.1 | 4380.0 | 4380.1 |
| When does splitting prefill and decode help?... | disagg, radix, batching | 0.0 | 0.1 | 6750.5 | 6750.6 |

Mean per stage (ms): embed **0.0** · retrieve **0.1** ·
llm **6106.0** · total **6106.1**
Dominant stage: **llm** (100% of total)

## Answers returned

**Why is goodput more useful than raw throughput?**

> Based on the context provided, **Goodput** is more useful than raw throughput because it focuses on **SLO compliance** (specifically the Time-to-Full-Throughput, or TTFT) rather than ignoring SLOs.

According to the text:
> "Goodput@SLO counts only the requests per second that met the TTFT and TPOT targets. Throughput at saturation ignores SLOs."

This implies that while raw throughput measures th

**What problem does PagedAttention actually solve?**

> PagedAttention solves the problem of **internal fragmentation in GPU memory** caused by storing KV cache in non-contiguous pages.

By using non-contiguous pages, it removes the wasted space that would otherwise be consumed by the internal fragmentation of contiguous memory blocks on a GPU.

**When does splitting prefill and decode help?**

> Splitting prefill and decode helps when the **prefill step is compute-bound** and the **decode step is memory-bound**.

This is because the context states that prefill is compute-bound (requiring significant processing power) and decode is memory-bound (requiring significant bandwidth). By separating these operations, the system can utilize different hardware resources more effectively:
*   **Pref


## Which N16-N19 pieces are real

- **N16 Cloud/IaC:** stubbed; the service runs on localhost rather than a cloud or
  container-orchestrated deployment.
- **N17 Data pipeline:** stubbed; documents come from an in-memory list rather than
  an Airflow DAG or batch ingestion job.
- **N18 Lakehouse:** stubbed; the toy dictionary stands in for a Delta/Iceberg table.
- **N19 Vector + features:** stubbed; retrieval uses `TOY_DOCS` and keyword overlap,
  with no embedding endpoint, vector index, or feature store.
- **N20 Serving:** real; all three answers were generated through the OpenAI-compatible
  `llama-server` endpoint.

The LLM being dominant was expected because embedding is disabled and keyword retrieval
over a tiny in-memory corpus takes only 0.1 ms. The LLM averages 6106.0 ms and accounts
for effectively 100% of total latency. To halve end-to-end latency, I would target the
LLM stage first, especially decode/output length or faster serving hardware; optimizing
retrieval cannot materially change a pipeline whose non-LLM work is only 0.1 ms.
