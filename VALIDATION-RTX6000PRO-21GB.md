# RTX PRO 4000/5000 21 GiB validation

Profile: Qwen3.8-27B groupwise-int, `rk4v4-e8`, explicit 145,920-token KV/context, C2, MTP3,
Vision8K and prefill512. This keeps NInfer at or below 21 GiB so an existing approximately 2 GiB
GPU workload can coexist on a 24 GB-class Blackwell card.

Measured on an RTX PRO 5000:

- 21,502 MiB idle and 21,504 MiB peak during a real image request.
- `17 * 23` returned `391`.
- `visual_chart.png` returned `NIFER VISION 731 | 3`.
- A 135,311-token multimodal needle test returned the buried code exactly.
- C1 decode was 103.0 tok/s; C2 aggregate was 129.3 tok/s.

The same image reached 524,288 context at 28,182 MiB on a PRO 5000 and passed a 369,313-token
retrieval, but that is a hardware-capacity test and not the 21 GiB profile. On the smaller PRO 4000,
prefill512 is required for very long prefills; prefill1024 can fail with
`cudaErrorCooperativeLaunchTooLarge`. Its constrained profile measured about 21,395 MiB.

Alternate NVFP4 artifacts were tested on an RTX PRO 4000 with the same 145,920-token, C2, MTP3,
Vision8K and `rk4v4-e8` profile:

| Model selection | Artifact | GPU used | C1 | C2 aggregate | Result |
| --- | ---: | ---: | ---: | ---: | --- |
| `qwen38-quasar` | 16.35 GiB | 20,769 MiB | 70.0 tok/s | 137.6 tok/s | arithmetic and vision passed |
| `qwen38-nvfp4full` | 17.07 GiB | 21,503 MiB | 65.0 tok/s | 118.0 tok/s | arithmetic and vision passed |

QUASAR is the practical NVFP4 option for a 24 GB card: it leaves about 735 MiB beneath the 21 GiB
NInfer ceiling. `nvfp4full` is technically inside that ceiling by 1 MiB in this sample and therefore
has no useful safety margin. The groupwise-int profile remains the default because its quality is
better established; the QUASAR model card describes narrower validation coverage.

Use the root `validate-vast` and `benchmark-api` scripts with `--context 145920`. The model,
tokenizer, chat template and vision resources are embedded in the `.ninfer` artifact; no separate
mmproj file is required.
