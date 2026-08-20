# ninfer

**A prebuilt Linux build of the NInfer RTX 3090 engine — the one thing upstream doesn't ship —
plus a single bash script that rents the GPU to run it on.**

Upstream ships a ready-to-run archive for Windows. Linux users get a Dockerfile and a
"build it yourself" guide. In their own words, from
[`RELEASE_NOTES_0.6.1.md`](https://github.com/Don-Chad/ninfer-3090/blob/main/RELEASE_NOTES_0.6.1.md):

> The project does not publish a prebuilt Linux archive.

This project closes that gap. It provides `sm_86` Linux binaries for **v0.6.1-rtx3090**, and a
CLI that rents an RTX 3090 on [Vast.ai](https://vast.ai), provisions it, serves an
OpenAI-compatible API, and tears it all down so idle cost is **$0.00/hr**.

```
ninfer create     # rent a 3090 and provision it (~2-8 min)
ninfer status     # state, endpoint, hourly cost, health
ninfer destroy    # delete it — idle cost goes to $0.00
```

## Platform support, upstream vs here

| | Linux | Windows |
| --- | --- | --- |
| [Don-Chad/ninfer-3090](https://github.com/Don-Chad/ninfer-3090) | Docker or source build only | Prebuilt archive |
| **this project** | **Prebuilt `sm_86` archive, v0.6.1** | — (use upstream) |

Upstream's last release carrying a Linux tarball was **v0.3.1**. Every release since — v0.4.0,
v0.5.0, v0.6.0 — ships Windows binaries only. This build tracks **v0.6.1**.

Upstream also notes that their Linux validation ran without a real model artifact, so no Linux
throughput figures were published. These binaries have been run against the real 18.2 GB
Qwen3.8-27B checkpoint on Ubuntu 24.04 — see [Verified on](#verified-on) below.

## Why rent, instead of leaving it running

Renting a GPU by the hour only saves money if you actually stop paying when you stop using it.
The friction is that tearing a box down means rebuilding it later — reinstalling CUDA deps,
re-downloading an 18 GB model, recompiling an inference engine. That's slow enough that most
people just leave the instance running, and the savings evaporate.

`ninfer` removes the friction:

- **Prebuilt binaries.** The engine is compiled once for `sm_86` and cached in object storage,
  so provisioning skips an 883-second build.
- **Fast model pull.** On a well-connected host the 18.2 GB checkpoint lands in about 30 seconds.
- **Nothing is assumed stable.** Vast reassigns the public IP, the mapped port *and* the SSH port
  on every start, so all three are re-resolved on every call. Client config is rewritten
  automatically when the address changes.

## Requirements

- A [Vast.ai](https://vast.ai) account with credit and an SSH key registered
- `bash`, `curl`, `python3`, `ssh` (macOS or Linux)
- An SSH keypair at `~/.ssh/id_ed25519` (override with `NINFER_SSH_KEY`)

## Setup

Create a `.env` next to the script:

```bash
VAST_API_KEY=your-vast-api-key
NINFER_API_KEY=any-string-you-choose      # the bearer token your clients will send
```

`NINFER_API_KEY` is yours to invent — it's the key the served API will require. Then:

```bash
ninfer create
```

Roughly two to eight minutes later you'll have an endpoint:

```
ready — http://<ip>:<port>/v1
  costing $0.2789/hr; 'ninfer destroy' when you're done
```

Which behaves like any OpenAI-compatible API:

```bash
curl -H "Authorization: Bearer $NINFER_API_KEY" \
     -H "Content-Type: application/json" \
     -d '{"model":"qwen3.8-27b","messages":[{"role":"user","content":"hello"}]}' \
     "$NINFER_BASE_URL/chat/completions"
```

## Commands

| Command | What it does |
| --- | --- |
| `ninfer create [offer]` | Rent a fresh 3090 and provision it from scratch. Picks the cheapest qualifying offer, or takes an offer id. |
| `ninfer destroy` | Delete the instance entirely — idle cost goes to $0.00. |
| `ninfer up` | Resume a **stopped** instance (~2 min, only if one exists). |
| `ninfer down` | Stop an instance, keeping the disk. **Still bills for storage.** |
| `ninfer status` | State, endpoint, hourly cost, health. |
| `ninfer offers` | List the 3090s worth renting right now. |
| `ninfer ssh [cmd]` | Shell on the box. |
| `ninfer log` | Tail the server log. |
| `ninfer kit-url` | Print a time-limited download URL for the prebuilt binaries. |
| `ninfer restore` | Reinstall those binaries on the current box. |

### down vs destroy

`down` keeps the 60 GB disk and keeps billing you for it — around 4¢/hr, or roughly a dollar a
day, forever. `destroy` costs nothing at all while idle, and because the binaries are cached
off-box a rebuild is only a few minutes. **Unless you're coming back within the hour, destroy.**

## Configuration

Everything has a sane default and an environment override:

| Variable | Default | Purpose |
| --- | --- | --- |
| `NINFER_PROJ` | script's directory | Where `.env` and instance state live |
| `NINFER_SSH_KEY` | `~/.ssh/id_ed25519` | Key used to reach the box |
| `NINFER_DISK` | `60` | Disk size in GB |
| `NINFER_INSTANCE` | saved state file | Target a specific instance |
| `NINFER_B2_ENV` | *(see below)* | Path to Backblaze B2 credentials for the prebuilt kit |
| `NINFER_KIT_KEY` | `ninfer/ninfer-3090-kit-v0.6.1-sm86.tar.gz` | Object key of the kit |

Instance state is kept in `.ninfer-instance`. Both that file and `.env` are gitignored.

### OpenCode integration

If you use [OpenCode](https://opencode.ai), `create` and `up` will rewrite the `baseURL` of a
provider named `ninfer` in any config listed in the `CONFIGS` array, so the address change after
a restart doesn't silently break your client. Configs that don't exist, or that have no `ninfer`
provider, are skipped harmlessly. Adapt the array for other clients.

## How provisioning works

1. **Pick** — query Vast's bundles API for single-3090 offers with enough disk, ≥1 Gbps down,
   CUDA ≥13, ≥23 GB VRAM and `reliability2 ≥ 0.97`, then take the cheapest.
2. **Rent** — create the instance from `nvidia/cuda:13.1.2-devel-ubuntu24.04` with port 8080 mapped.
3. **Wait for SSH** — the box has to be reachable before anything else happens.
4. **Provision** — apt deps and an 18.2 GB checkpoint pulled from Hugging Face, detached so a
   dropped connection doesn't kill it.
5. **Install the kit** — prebuilt `sm_86` binaries, verified by SHA-1.
6. **Serve** — launch under a supervisor loop that restarts on crash, wait for HTTP 200, then
   rewrite client config with the new address.

## The prebuilt Linux kit

Step 5 installs a tarball of `sm_86` binaries — `ninfer` and `ninfer-serve`, built from
[Don-Chad/ninfer-3090](https://github.com/Don-Chad/ninfer-3090) `release/v0.6.0-rtx3090`
(rev `403fc56d`, VERSION `0.6.1-rtx3090`) against `nvidia/cuda:13.1.2-devel-ubuntu24.04`.

Installing it is what lets a rebuilt box come up in minutes instead of paying an **883-second**
compile every single time you destroy and re-create. That build cost is the entire reason people
leave GPU instances running, and the entire reason this kit exists.

The script fetches the tarball over a signed, time-limited URL and verifies it by SHA-1 before
unpacking. `NINFER_B2_ENV` and `NINFER_KIT_KEY` point at the storage holding it; `ninfer kit-url`
prints the signed URL, and `ninfer restore` reinstalls onto a running box.

### Download

The kit is published on this repo's
[**Releases**](https://github.com/coder903/ninfer/releases/latest) page:

```bash
curl -fLO https://github.com/coder903/ninfer/releases/latest/download/ninfer-3090-kit-v0.6.1-sm86.tar.gz
curl -fLO https://github.com/coder903/ninfer/releases/latest/download/SHA256SUMS.txt
sha256sum -c SHA256SUMS.txt
tar xzf ninfer-3090-kit-v0.6.1-sm86.tar.gz -C /root
```

That gives you `/root/kit/bin/{ninfer,ninfer-serve}` ready to run — no CUDA toolchain, no
883-second build. `ninfer restore` does the same thing onto a box the CLI is already managing.

The binaries are Apache-2.0 and are **not** original work of this project — see
[`third_party/ninfer-3090/ATTRIBUTION.md`](third_party/ninfer-3090/ATTRIBUTION.md).

## Verified on

The published binaries have been run end to end, not merely compiled:

| | |
| --- | --- |
| Host | Vast.ai RTX 3090 (24 GB), Ubuntu 24.04 |
| Image | `nvidia/cuda:13.1.2-devel-ubuntu24.04` |
| Engine | `0.6.1-rtx3090`, `sm_86` |
| Model | `neroued/Qwen3.8-27B-NInfer`, 18.2 GB |
| Resident | 21,013 MiB of 24,576 MiB |
| Serving | `GET /v1/models` → 200; `POST /v1/chat/completions` round-trip 0.69 s |
| Profile | 64K context, `int8` KV, spec MTP w/ 3 draft tokens, concurrency 4 |

## Credits

This script is only orchestration. The actual inference engine is someone else's work:

- **[Neroued/ninfer](https://github.com/Neroued/ninfer)** — the upstream high-performance
  single-GPU inference engine (Apache-2.0)
- **[Don-Chad/ninfer-3090](https://github.com/Don-Chad/ninfer-3090)** — the RTX 3090 fork these
  binaries are built from, `release/v0.6.0-rtx3090` (Apache-2.0)
- **[neroued/Qwen3.8-27B-NInfer](https://huggingface.co/neroued/Qwen3.8-27B-NInfer)** — the model
  checkpoint

If you redistribute binaries built from those projects, comply with Apache-2.0: include the
license, keep the attribution notices, and ship any `NOTICE` file.

## License

MIT — see [LICENSE](LICENSE).
