# RTX 5090 validation

Profile: Qwen3.8-27B NVFP4, `rk4v4-e8`, C2, MTP3, Vision8K, prefill1024. The pinned artifact is
`neroued/Qwen3.8-27B-nvfp4-NInfer/qwen3_8_27b_nvfp4.ninfer`, SHA-256
`bb3360522a06e136e0367f5703414d26272b7285c8a6ab6194135c17dbd81b32`.

Production context/KV is `524288`. It trades 26,944 tokens below the tested `551232` startup
boundary for roughly 475 MiB calculated reservation margin. The next page, `551296`, failed
startup reservation. The fully measured `548864` profile used 31,904 MiB idle and 31,906 MiB
request peak on a 32,607 MiB card.

Correctness passed:

- `17 * 23` returned `391`.
- `visual_chart.png` returned `NIFER VISION 731 | 3`.
- A 312,922-token request retrieved `SAPPHIRE-568K` exactly.

Measured at context/KV `548864`:

| Concurrency | Per-stream | Aggregate | MTP acceptance |
| ---: | ---: | ---: | ---: |
| 1 | 160.0 tok/s | 154.2 tok/s | 53.8% |
| 2 | 152.9 tok/s | 272.3 tok/s | 53.9% |

Use the root `validate-vast` and `benchmark-api` scripts. The 524K profile is an intentional
headroom-oriented default; measure peak VRAM before deploying on a particular card.
