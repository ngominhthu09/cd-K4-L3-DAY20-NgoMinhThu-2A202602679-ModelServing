# 01 - Tune: thread-count sweep

Model `Qwen3.5-0.8B-Q4_K_M.gguf` · host `Windows-AMD64` · llama.cpp `b10488`
CPU: **4 physical · 8 logical** cores · `ngl=99` · metric `tg128`

| threads (-t) | tg128 (tok/s) | vs best |
|:--|--:|--:|
| 1 | 31.0 | 90% |
| 2 | 32.8 | 95% |
| 4 | 34.2 | 99% |
| 8 | 34.2 | 99% |
| 16 | 34.4 | 100% |

**Best**: `-t 16` at 34.4 tok/s
**Slowest tested**: `-t 1` at 31.0 tok/s (1.11x spread)
**Against the physical-core default** (`-t 4`, 34.2 tok/s): 1.01x

Use this in your run:

```bash
LAB_N_THREADS=16 make bench
```

## Your explanation

The knee is at 4 threads: throughput rises from 31.0 tok/s with 1 thread to 34.2 tok/s
with 4 threads, then stays nearly flat at 34.2-34.4 tok/s through 16 threads. With
`ngl=99`, most model computation is offloaded to the Quadro P620, so CPU thread count
is not the main bottleneck. The nominal 0.6% gain from 4 to 16 threads is small enough
to be measurement noise. I would keep 4 threads to avoid oversubscription and needless
CPU use while retaining effectively the same decode throughput.
