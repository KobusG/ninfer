# Validation

The repository has two sm120a deployment profiles:

| Profile | Compose | Model | Context/KV | Purpose |
| --- | --- | --- | ---: | --- |
| RTX 5090 | `5090/compose.yaml` | Qwen3.8 NVFP4 | 524,288 | throughput and long context |
| RTX PRO 4000/5000 | `rtx6000pro-21gb/compose.yaml` | Qwen3.8 groupwise-int | 145,920 | keep NInfer below 21 GiB |

Both use the same immutable model-free GHCR image and entrypoint contract. The model is downloaded
and SHA-256 verified at runtime. The image always loads the repository's Froggeric chat template
and selects medium reasoning effort. The template is an exact supported NInfer frontend override,
not an arbitrary Jinja interpreter; its SHA-256 is
`035a8f181424fad43fba759164f506771ade9f8ac05d04df1b2f6f9ccbdfbad2`.

## Local VPS

Use the profile matching the GPU. Bind the API to loopback and put Caddy or another authenticated
proxy in front of it. Set `CHAT_API_KEY` for an additional NInfer bearer-token boundary:

```bash
docker compose -f 5090/compose.yaml up -d
docker compose -f 5090/compose.yaml logs -f qwen
curl http://127.0.0.1:8080/v1/models
```

Do not publish `8080:8080` directly on an internet-facing host.

## Vast validation

The root `validate-vast` renders a selected production Compose file and launches it in Vast args
mode. The argument is an on-demand **offer ID**, not an existing contract ID:

```bash
vastai search offers \
  'gpu_name=RTX_5090 num_gpus=1 rentable=true verified=true datacenter=true cuda_max_good>=13.1 direct_port_count>=2' \
  --type on-demand --limit 10 --order dph_total

./validate-vast --compose 5090/compose.yaml OFFER_ID
```

It maps ports 22 and 8080, enables temporary key-only SSH, replaces the Compose API key with a
generated validation key, and writes `.vast-validation.env` plus `.vast-validation-instance` in the
repository root. Destroy the rental when complete:

```bash
set -a; . ./.vast-validation.env; set +a
./benchmark-api "$NINFER_BASE_URL" --context 524288
vastai destroy instance "$(cat .vast-validation-instance)" -y
```

For the constrained target:

```bash
./validate-vast --compose rtx6000pro-21gb/compose.yaml OFFER_ID
./benchmark-api "$NINFER_BASE_URL" --context 145920
```

`benchmark-api` calibrates its record generator against the endpoint tokenizer, sends one near-full
context multimodal request, and reports usage, TTFT, approximate prefill throughput, client-observed
decode throughput, image validation and buried-needle retrieval. Configure the fixture and markers
for a different vision-capable model:

```bash
./benchmark-api https://HOST:PORT/v1 \
  --context 524288 --image-url https://example/image.png \
  --image-marker 'TITLE 731' --needle 'MY-NEEDLE'
```

This is a C1 full-context check. NInfer's explicit KV pool is shared by concurrency slots, so two
simultaneous full-size prompts cannot both occupy the full configured capacity. Use each profile's
`ninfer bench 1,2` for short-prompt C1/C2 throughput.

All Blackwell launch paths pass `--chat-template
/usr/share/doc/ninfer/froggeric-chat-template.jinja` and `--reasoning-effort medium`. The image
contains that template and the engine accepts it only when its exact supported SHA-256 matches;
changing the file requires a new engine/image release. The older `3090/` kit is a separate sm86
engine and does not support this Blackwell-only override.

## Measured profiles

### RTX 5090

The NVFP4 profile ran on a 32,607 MiB RTX 5090. `548,864` held 31,904 MiB idle and peaked at
31,906 MiB; `551,232` started, while the next 64-token page failed reservation. The production
default is `524,288`, about 475 MiB below the measured linear reservation estimate. A 312,922-token
retrieval passed. At 548,864, measured throughput was 160.0 tok/s C1 and 272.3 tok/s aggregate C2,
with 53.8% and 53.9% MTP acceptance.

### RTX PRO 4000/5000

The groupwise-int profile uses `145,920` context/KV, `rk4v4-e8`, C2, MTP3, Vision8K and prefill512.
On the PRO 5000 it peaked at 21,504 MiB during a real image request, passed a 135,311-token
multimodal needle test, and measured 103.0 tok/s C1 and 129.3 tok/s aggregate C2. A separate PRO
5000 capacity test reached 524,288 context at 28,182 MiB with 369,313-token retrieval, but is not
the 21 GiB production profile.

The PRO 4000 requires prefill512 for very long prefills; prefill1024 can fail with
`cudaErrorCooperativeLaunchTooLarge`. Its constrained profile measured about 21,395 MiB.

## Image build

Builds run locally, never on rented GPUs:

```bash
./docker/build-image
```

The shared Dockerfile pins CUDA stages by digest, fetches an exact engine commit, excludes models
and credentials from the build context, and publishes one model-free runtime image. Authenticate
to GHCR separately before `--push`, then deploy the resulting immutable manifest digest in both
Compose files and profile `.env` files. Profile `build-image` wrappers delegate to this command.
