# 01 - Measure: latency baseline

Model `Qwen3.5 0.8B` · host `Windows-AMD64` · llama.cpp `b10488`
Settings: `threads=4` `ngl=99` `ctx=2048`
`max_tokens=64` · warm-up discarded
Completed requests: `Q4_K_M` 10/10 · `UD-Q2_K_XL` 10/10

| Quantization | Size (GB) | Load (ms) | TTFT P50/P95 (ms) | TPOT P50/P95 (ms) | E2E P50/P95/P99 (ms) | Decode (tok/s) |
|:--|--:|--:|--:|--:|--:|--:|
| Q4_K_M | 0.50 | 44734 | 525 / 555 | 30.2 / 30.3 | 2420 / 2459 / 2459 | 33.1 |
| UD-Q2_K_XL | 0.39 | 4556 | 615 / 860 | 34.7 / 37.5 | 2784 / 3094 / 3094 | 28.8 |

- **TTFT** = prefill. Short prompts keep it small; long-context RAG is where it explodes.
- **TPOT** = per-output-token decode cost, bounded by memory bandwidth. `decode tok/s = 1000 / TPOT_p50`.
- `UD-Q2_K_XL` decodes **1.15x SLOWER** than `Q4_K_M` here, despite being 0.11 GB smaller. That is a real result, not a mistake: fewer bits only buys speed when decode is limited by memory bandwidth. On a machine that is compute-limited instead — few cores, no GPU offload — the extra dequantization work of a heavily-quantized format can cost more than the bytes it saves. Say which case yours is.

## Your observation

Q2 is 0.11 GB (22%) smaller, but its decode throughput is about 13% lower than Q4;
its median TTFT is 17% higher and median end-to-end latency is 15% higher. With the
same prompt, Q4 gave a shorter, more direct answer, although neither answer defined
Goodput@SLO completely correctly; Q2 was more circular and confused it with an SLA.
Therefore, Q2 is not worth using on this machine: it is slower and produced the weaker
answer despite its smaller size.
