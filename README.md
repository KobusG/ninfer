# ninfer

**Prebuilt Linux builds of the NInfer inference engine — the thing upstream doesn't ship — plus a
single bash script that rents the GPU to run one on and tears it down when you're done.**

NInfer is a from-scratch C++/CUDA engine that runs a closed set of registered Qwen checkpoints
faster than a general-purpose runtime does. There are two of them, one per GPU generation, and
neither publishes what a Linux user actually needs:

| Project | Linux binaries | Windows binaries |
| --- | --- | --- |
| [Neroued/ninfer](https://github.com/Neroued/ninfer) — the RTX 5090 engine | none, ever | none, ever |
| [Don-Chad/ninfer-3090](https://github.com/Don-Chad/ninfer-3090) — the RTX 3090 backport | none since v0.3.1 | prebuilt archive |
| **this project** | **`sm_86` v0.6.1 · `sm_120a` rev `feaf4dd0`** | — (use upstream) |

Both say so themselves. The 3090 fork, in
[`RELEASE_NOTES_0.6.1.md`](https://github.com/Don-Chad/ninfer-3090/blob/main/RELEASE_NOTES_0.6.1.md):

> The project does not publish a prebuilt Linux archive.

And the 5090 project, which goes further — it has no releases and no tags at all:

> There is no install target or packaged binary distribution; NInfer is run from its source build
> tree.

So the 3090 gap is Linux-shaped. The 5090 gap is every-platform-shaped — upstream's only path is a
`Dockerfile` you build yourself on a machine that already has the card. As far as we can tell, the
`sm_120a` tarball below is the only prebuilt NInfer binary that exists anywhere.

```
ninfer create     # rent a GPU and provision it
ninfer status     # state, endpoint, hourly cost, health
ninfer destroy    # delete it — idle cost goes to $0.00
```

## Which card

| | `3090/` | `5090/` |
| --- | --- | --- |
| Architecture | `sm_86`, Ampere, 24 GB | `sm_120a`, Blackwell, 32 GB |
| Engine | [Don-Chad/ninfer-3090](https://github.com/Don-Chad/ninfer-3090) v0.6.1 | [Neroued/ninfer](https://github.com/Neroued/ninfer) `master` |
| Default model | Qwen3.8-27B `groupwise-int` | Qwen3.6-27B **NVFP4** |
| Binaries | prebuilt, downloaded in seconds | prebuilt, downloaded in seconds |
| Typical rent | ~$0.28/hr | ~$0.40–0.95/hr |
| Compile, if you skip the kit | 883 s at `-j12` | 302 s at `-j128` |

NVFP4 is the 5090's default because W4A4 tensor cores are the one thing a 3090 physically cannot
do. Upstream measures that profile at 1,146.9 aggregate decode tok/s across eight concurrent
requests — **5.67×** its single-request throughput, where the integer profile manages 2.88×. Set
`NINFER_MODEL=qwen38-27b` to run the same checkpoint the 3090 runs, if you want the comparison
without the variable.

Upstream's 3090 notes also mention that their Linux validation ran without a real model artifact,
so no Linux throughput figures were published. These binaries have been run against the real
18.2 GB Qwen3.8-27B checkpoint on Ubuntu 24.04 — see [Verified on](#verified-on) below.

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

## Layout

The repo is laid out by GPU, because a build is only valid for the architecture it was compiled
against, and the two cards do not even share an upstream:

```
3090/                 sm_86  · Ampere  · Don-Chad/ninfer-3090 v0.6.1
├── ninfer            the CLI
├── .env.example      credentials template
└── third_party/      upstream Apache-2.0 license + attribution

5090/                 sm_120a · Blackwell · Neroued/ninfer master
├── ninfer            the CLI, same shape, different everything else
├── .env.example
└── third_party/
```

Each directory is self-contained: the script resolves its own location, so `.env` and
`.ninfer-instance` live beside the copy you run, and the two cards never share state. Pick one,
work in it. Another card would be another sibling.

## Requirements

- A [Vast.ai](https://vast.ai) account with credit and an SSH key registered
- `bash`, `curl`, `python3`, `ssh` (macOS or Linux)
- An SSH keypair at `~/.ssh/id_ed25519` (override with `NINFER_SSH_KEY`)

## Setup

```bash
git clone https://github.com/coder903/ninfer.git
cd ninfer
cp 3090/.env.example 3090/.env
```

Fill in `3090/.env` — the script reads the `.env` sitting next to it:

```bash
VAST_API_KEY=your-vast-api-key
NINFER_API_KEY=any-string-you-choose      # the bearer token your clients will send
```

Optionally put it on your `PATH`. Symlinks are resolved, so an installed link still finds its own
`.env`:

```bash
ln -s "$PWD/3090/ninfer" ~/.local/bin/ninfer
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

## The prebuilt Linux kits

Both cards install a tarball of binaries — `ninfer` and `ninfer-serve` — rather than compiling on
the box. That build cost is the entire reason people leave GPU instances running, and the entire
reason these kits exist.

| | 3090 kit | 5090 kit |
| --- | --- | --- |
| Architecture | `sm_86` | `sm_120a` |
| Built from | [Don-Chad/ninfer-3090](https://github.com/Don-Chad/ninfer-3090) `release/v0.6.0-rtx3090`, rev `403fc56d`, VERSION `0.6.1-rtx3090` | [Neroued/ninfer](https://github.com/Neroued/ninfer) `master`, rev `feaf4dd0` |
| Build image | `nvidia/cuda:13.1.2-devel-ubuntu24.04` | `nvidia/cuda:13.1.2-devel-ubuntu24.04` |
| Compile it replaces | 883 s at `-j12` | 302 s at `-j128` |
| Size | 375 MB | 297 MB |

Every 5090 tarball also carries a `BUILDINFO.txt` recording the exact revision, CUDA version,
driver, GPU, job count and wall-clock build time of that specific build.

The script fetches a tarball and verifies it by SHA-1 before unpacking. `ninfer kit-url` prints the
URL it would use, and `ninfer restore` reinstalls onto a box the CLI is already managing. If you
have your own object storage, `NINFER_B2_ENV` and `NINFER_KIT_KEY` point at a private copy;
otherwise the public GitHub release is used, which is the normal case.

### Download

Both kits are on this repo's [**Releases**](https://github.com/coder903/ninfer/releases) page.

RTX 5090, `sm_120a`:

```bash
T=rtx5090-linux-sm120a-feaf4dd0
curl -fLO https://github.com/coder903/ninfer/releases/download/$T/ninfer-5090-kit-feaf4dd0983f-sm120a.tar.gz
curl -fLO https://github.com/coder903/ninfer/releases/download/$T/SHA256SUMS.txt
sha256sum -c SHA256SUMS.txt
tar xzf ninfer-5090-kit-feaf4dd0983f-sm120a.tar.gz -C /root
```

RTX 3090, `sm_86`:

```bash
T=v0.6.1-rtx3090-linux-sm86
curl -fLO https://github.com/coder903/ninfer/releases/download/$T/ninfer-3090-kit-v0.6.1-sm86.tar.gz
curl -fLO https://github.com/coder903/ninfer/releases/download/$T/SHA256SUMS.txt
sha256sum -c SHA256SUMS.txt
tar xzf ninfer-3090-kit-v0.6.1-sm86.tar.gz -C /root
```

Either gives you `/root/kit/bin/{ninfer,ninfer-serve}` ready to run — no CUDA toolchain, no
compile. They need the same runtime libraries the build had: `libavcodec`, `libavformat`,
`libavutil`, `libswscale` and `libcurl`.

The binaries are Apache-2.0 and are **not** original work of this project — see
[`3090/third_party/ninfer-3090/ATTRIBUTION.md`](3090/third_party/ninfer-3090/ATTRIBUTION.md) and
[`5090/third_party/ninfer/ATTRIBUTION.md`](5090/third_party/ninfer/ATTRIBUTION.md).

## Verified on

The published binaries have been run end to end, not merely compiled:

| | RTX 3090 | RTX 5090 |
| --- | --- | --- |
| Host | Vast.ai, Ubuntu 24.04 | Vast.ai, Ubuntu 24.04, driver 610.43.03 |
| Image | `nvidia/cuda:13.1.2-devel-ubuntu24.04` | `nvidia/cuda:13.1.2-devel-ubuntu24.04` |
| Engine | `0.6.1-rtx3090`, `sm_86` | rev `feaf4dd0`, `sm_120a` |
| Model | `Qwen3.8-27B-NInfer`, 18.2 GB | `Qwen3.6-27B-nvfp4-NInfer`, 17.07 GiB |
| Serving | `/v1/models` → 200; chat round-trip 0.69 s | `/v1/models` → 200; chat round-trip 0.93 s, 91 output tokens |
| Profile | 64K context, `int8` KV, MTP-3, concurrency 4 | 32K context, `int8` KV, MTP-3 + `--lm-head-draft`, concurrency 8 |

The model artifact is checked against upstream's published SHA-256 after download, and the kit
against its SHA-1 before unpacking.

## Credits

This script is only orchestration. The actual inference engine is someone else's work:

- **[Neroued/ninfer](https://github.com/Neroued/ninfer)** — the RTX 5090 engine, and the upstream high-performance
  single-GPU inference engine (Apache-2.0)
- **[Don-Chad/ninfer-3090](https://github.com/Don-Chad/ninfer-3090)** — the RTX 3090 fork these
  binaries are built from, `release/v0.6.0-rtx3090` (Apache-2.0)
- **[neroued/Qwen3.8-27B-NInfer](https://huggingface.co/neroued/Qwen3.8-27B-NInfer)** — the model
  checkpoint

If you redistribute binaries built from those projects, comply with Apache-2.0: include the
license, keep the attribution notices, and ship any `NOTICE` file.

## License

MIT — see [LICENSE](LICENSE).
