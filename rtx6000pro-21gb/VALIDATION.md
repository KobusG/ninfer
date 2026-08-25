# RTX Pro 21 GiB validation

This records how the Qwen3.8-27B `sm_120a` profile was validated on an RTX PRO 5000 Blackwell.
The same immutable, model-free image is used by `ninfer`; the model is downloaded at runtime.

## Profile

```dotenv
NINFER_KV_DTYPE=rk4v4-e8
NINFER_MAX_CONTEXT=145920
NINFER_KV_CAPACITY=145920
NINFER_MAX_CONCURRENCY=2
NINFER_PREFILL_CHUNK=512
NINFER_SPEC=mtp
NINFER_DRAFT_TOKENS=3
NINFER_VISION=1
NINFER_VISION_MAX_TOKENS=8192
```

The image is pinned in `ninfer` by digest. Its entrypoint reads context, KV capacity, concurrency,
speculation and vision budgets from environment variables, so changing these values does not require
rebuilding the image.

Copy `.env.example` to `.env`, populate `VAST_API_KEY` and `NINFER_API_KEY`, then run commands
directly from this directory. The script sources `.env` before resolving the profile, so no variables
need to be prepended:

```bash
./ninfer create
./ninfer status
./ninfer bench 1,2
./ninfer destroy
```

## Building a replacement image

Builds run locally, never on a rented GPU. `build-image` uses the adjacent multi-stage `Dockerfile`,
fetches an exact commit from [`KobusG/ninfer-engine`](https://github.com/KobusG/ninfer-engine),
compiles `sm_120a` binaries in the CUDA development stage and copies only binaries, license and
provenance into the runtime stage. No model or credential enters the build context or image.

The default source is the commit used by the validated image. Rebuild and layout-check it with:

```bash
./build-image
```

To build a newer engine revision, always use its full commit hash:

```bash
./build-image --ref FULL_40_CHARACTER_COMMIT
```

The default local tag is `<short-commit>-rk4v4e8-vision8k`. To publish to the existing public GHCR
package, authenticate, build and push:

```bash
gh auth token | docker login ghcr.io -u KobusG --password-stdin
./build-image --ref FULL_40_CHARACTER_COMMIT --tag DESCRIPTIVE_IMMUTABLE_TAG --push
docker logout ghcr.io
```

After the push, copy the reported `sha256:` manifest digest into the default `IMAGE` value in
`ninfer`. Do not deploy a mutable tag. Run the vision, memory, needle and C1/C2 checks below before
replacing the currently validated digest.

The runtime image includes the Hugging Face CLI, `curl`, FFmpeg, SSH and `tini`. Its entrypoint
optionally starts key-only SSH, downloads and verifies the registered model, then replaces itself
with `ninfer-serve`. SSH host keys are deliberately removed from the image and generated per
container. Binaries are linked into `/usr/local/bin`.

Vast uses `runtype=args`, which preserves that entrypoint instead of installing Vast's SSH/Jupyter
bootstrap. Ports 22 and 8080 are explicitly mapped. On the 2026-08-25 Secure Cloud smoke test, Vast
reached `running` in 50 seconds, entrypoint SSH was ready immediately, and the API was ready 45
seconds later after downloading the 16.96 GiB model. Total `create` wall time was 129.7 seconds.

## VPS Docker Compose

`compose.yaml` uses the same image entrypoint and runs the validated 21 GiB profile directly on an
owned `sm_120a` VPS. NInfer's own
authentication is optional and the container port is bound exclusively to host loopback. Leave
`CHAT_API_KEY` empty when authenticated Caddy is the sole authentication boundary, or set
it to enable NInfer bearer-token authentication as an additional layer.

```dotenv
# Optional. Empty means Caddy-only authentication.
CHAT_API_KEY=
# Optional for authenticated Hugging Face downloads:
HF_TOKEN=
```

```bash
docker compose up -d
docker compose logs -f qwen
curl http://127.0.0.1:8080/v1/models
```

When `CHAT_API_KEY` is set, clients must send either
`Authorization: Bearer <value>` or `x-api-key: <value>`. Caddy must preserve or inject one of those
headers. `NINFER_API_KEY` belongs to the Vast orchestration script and is deliberately not consumed
by Compose.

The named `qwen-models` volume retains the 16.96 GiB artifact and its small Hugging Face download
environment across container replacement. On first start the container downloads the public
Qwen3.8 NInfer artifact and verifies its published SHA-256 before serving. The model, credentials
and cache are not part of the image. Do not change the port mapping to `8080:8080`; that could bypass
Caddy and expose the unauthenticated API. A host-installed Caddy should proxy to
`127.0.0.1:8080`. If Caddy itself runs in Docker, put both services on a private shared Docker
network, remove the `ports` mapping, and proxy to `qwen:8080` instead.

The Compose command fixes context/KV at 145,920, concurrency 2, `rk4v4-e8`, prefill chunk 512,
MTP3, Vision and an 8K vision workspace. NInfer's tokenizer, chat template and vision resources are
embedded in the `.ninfer` artifact, so no Jinja template or separate `mmproj` mount is required.

## Memory ceiling

VRAM was sampled every 100 ms around server startup and real requests:

```bash
nvidia-smi --query-compute-apps=used_memory \
  --format=csv,noheader,nounits -lms 100 -f /tmp/ninfer-vram.csv
```

Host memory was sampled from `MemAvailable` in `/proc/meminfo`. GPU utilization, clocks, power and
PCIe state were also checked with `nvidia-smi`; no host-RAM spill or PCIe-bound offload appeared.

The context search used 64-token KV pages and a fixed 8K vision workspace:

| Context/KV capacity | Idle | Real image-request peak | Result |
| ---: | ---: | ---: | --- |
| 262,144 | 23,552 MiB | not run | Over budget |
| 155,648 | 21,672 MiB | not run | Over budget |
| 146,048 | 21,504 MiB | 21,506 MiB | 2 MiB over |
| **145,920** | **21,502 MiB** | **21,504 MiB** | **21 GiB ceiling** |

NInfer reserves weights, explicit KV, sequence state, maximum-phase scratch, vision transient and
CUDA graph allowances at startup. Requests do not intentionally grow these pools, but driver-level
accounting varied by a few MiB, which is why the request peak, not idle residency, set the limit.

The current entrypoint image was smoke-tested at the same profile on an RTX PRO 4000 Blackwell. It
held 21,395 MiB both idle and across a 100 ms-sampled real image request, enforced bearer auth,
returned `391` for `17 * 23`, and returned `NIFER VISION 731 | 3` from the fixture below. The host
still reported about 229 GiB available RAM, so no host-memory spill was involved.

## Hardware-max profiles

The preceding image revision was also profiled at its 262,144-token compiled ceiling. These profiles
use concurrency 2, MTP3, `rk4v4-e8`, Vision 8K and explicit KV capacity equal to context.

| GPU | Context/KV | Prefill chunk | Idle | Peak | C1 decode | C2 aggregate | Result |
| --- | ---: | ---: | ---: | ---: | ---: | ---: | --- |
| RTX PRO 4000 Blackwell 24,467 MiB | 262,144 | 512 | 23,450 MiB | 23,452 MiB | 54.5 tok/s | 61.6 tok/s | 250,311-token multimodal needle passed |
| RTX 5090 32,607 MiB | 262,144 | 1024 | 23,734 MiB | 23,736 MiB | 159.6 tok/s | 189.6 tok/s | 250,311-token multimodal needle passed |

The PRO 4000 failed a long prefill at chunk 1024 with `cudaErrorCooperativeLaunchTooLarge`; chunk
512 passed the same request. This was a launch-capacity limit on the smaller GPU, not an out-of-memory
failure. Its final C1/C2 MTP acceptance was 54.9%/48.7%; sampled power peaked at 145.9 W and the host
retained about 114.6 GiB available RAM with unchanged swap use. The 5090 C1/C2 acceptance was
62.1%/55.5%; sampled power peaked at 466.6 W, clock at 2,932 MHz and PCIe at Gen5 x16.

Both returned exactly:

```text
NIFER VISION 731 | COBALT-8421
```

The PRO 4000 hardware-max profile leaves only about 1 GiB free and is not the 21 GiB production
profile. Keep 145,920 in `.env.example` when the GPU must coexist with the separate 2 GiB workload.
The current `c711bf57...` image includes direct global block-table lookup, a 1,048,576-token envelope
and YaRN support from engine commit `b549d912`. Its normal 145,920-token profile passed the Secure
Cloud smoke test, but contexts above 262,144 have not yet been GPU-validated and are not production
defaults.

## Vision inference

The Qwen3.8 `.ninfer` artifact contains its vision resources; there is no separate `mmproj` download.
The server must start with `--vision`. Validation sent this public fixture as OpenAI `image_url`:

```text
https://raw.githubusercontent.com/UDPSendToFailed/ninfer-4090/11aae2d6342d1fbdc7ef2b9ee17096858be44271/examples/cli/media/visual_chart.png
```

The instruction requested only the chart title and final value. The response was exactly:

```text
NIFER VISION 731 | 3
```

That confirms pixels were processed rather than merely accepting a structured text request.

## Long-context needle

The final test generated 8,000 records, replaced record 5,777 with `ORCHID-7319`, attached the image
above and asked for both the chart title and buried code. This is the request generator used against
the provisioned OpenAI-compatible endpoint:

```python
import json
import os
import urllib.request

base = os.environ["NINFER_BASE_URL"]
headers = {
    "Authorization": "Bearer " + os.environ["NINFER_API_KEY"],
    "Content-Type": "application/json",
}
models = urllib.request.Request(base + "/models", headers=headers)
model = json.load(urllib.request.urlopen(models))["data"][0]["id"]

records = [f"Record {i}: code ITEM-{i:05d}." for i in range(8000)]
records[5777] = "Record 5777: code ORCHID-7319."
content = [
    {
        "type": "image_url",
        "image_url": {
            "url": "https://raw.githubusercontent.com/UDPSendToFailed/ninfer-4090/"
                   "11aae2d6342d1fbdc7ef2b9ee17096858be44271/examples/cli/media/visual_chart.png"
        },
    },
    {
        "type": "text",
        "text": "\n".join(records)
                + "\nWhat is the image title and the code in record 5777? "
                  "Reply only: TITLE | CODE",
    },
]
body = json.dumps({
    "model": model,
    "messages": [{"role": "user", "content": content}],
    "max_tokens": 48,
    "temperature": 0,
}).encode()
request = urllib.request.Request(base + "/chat/completions", body, headers)
result = json.load(urllib.request.urlopen(request, timeout=900))
print(result["usage"]["prompt_tokens"])
print(result["choices"][0]["message"]["content"])
```

Observed result:

```text
135311
NIFER VISION 731 | ORCHID-7319
```

## Throughput and correctness

The built-in benchmark runs on the GPU host over localhost and extracts engine-reported decode and
MTP figures. The final command was:

```bash
NINFER_BENCH_TOKENS=128 ./ninfer bench 1,2
```

| Concurrency | Per-stream decode | Aggregate | MTP tokens/round | Acceptance |
| ---: | ---: | ---: | ---: | ---: |
| 1 | 103.0 tok/s | 97.0 tok/s | 2.76 | 59.6% |
| 2 | 73.6 tok/s | 129.3 tok/s | 2.88 | 63.9% |

Earlier controls compared `rk4v4-e8` against INT8 on the same host. Quantized KV saved about
270 MiB with MTP3 while remaining within run-to-run throughput variance. Exact arithmetic and a
10,923-token text needle test also passed before the maximum-context run.
