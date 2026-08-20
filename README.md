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

- A [Vast.ai](https://vast.ai) account with credit and an SSH key registered on it
- `bash`, `curl`, `python3`, `ssh` (macOS or Linux)
- An SSH keypair at `~/.ssh/id_ed25519` (override with `NINFER_SSH_KEY`)

Nothing needs to be installed on your machine beyond that — no CUDA, no Python packages, no
Docker. Everything heavy happens on the rented box.

## Setup

**Pick your card first.** Every command below runs from inside `3090/` or `5090/`, and the two
never share state. Substitute whichever you have:

```bash
git clone https://github.com/coder903/ninfer.git
cd ninfer

CARD=5090                                  # or 3090
cp $CARD/.env.example $CARD/.env
```

Fill in `$CARD/.env` — the script reads the `.env` sitting next to it, not one in the repo root:

```bash
VAST_API_KEY=your-vast-api-key
NINFER_API_KEY=any-string-you-choose      # the bearer token your clients will send
```

`NINFER_API_KEY` is yours to invent. It isn't issued by anybody — it's simply the key the served
API will demand. `NINFER_BASE_URL` is written for you on every `create` and `up`; leave it blank.

Optionally put the script on your `PATH`. Symlinks are resolved, so an installed link still finds
its own `.env` and its own card:

```bash
ln -s "$PWD/5090/ninfer" ~/.local/bin/ninfer5090
ln -s "$PWD/3090/ninfer" ~/.local/bin/ninfer3090
```

Then:

```bash
cd $CARD
./ninfer create
```

A few minutes later you have an endpoint:

```
ready — http://<ip>:<port>/v1
  model:    qwen3_6_27b_nvfp4.ninfer (17.07 GiB, nvfp4-27b)
  opencode: pick ninfer5090/qwen3.6-27b
  costing $0.4296/hr; 'ninfer destroy' when you're done
```

It behaves like any OpenAI-compatible API. **Send the model id the line above printed** — it is
the artifact's own identity, and the two cards do not share one (`qwen3.6-27b` on the 5090,
`qwen3.8-27b` on the 3090). `ninfer status` reprints it, and `GET /v1/models` is authoritative:

```bash
source .env
curl -H "Authorization: Bearer $NINFER_API_KEY" "$NINFER_BASE_URL/models"

curl -H "Authorization: Bearer $NINFER_API_KEY" \
     -H "Content-Type: application/json" \
     -d '{"model":"qwen3.6-27b","messages":[{"role":"user","content":"hello"}]}' \
     "$NINFER_BASE_URL/chat/completions"
```

## Commands

| Command | What it does |
| --- | --- |
| `ninfer create [offer]` | Rent a GPU and provision it. Picks the cheapest qualifying offer, or takes an offer id from `ninfer offers`. |
| `ninfer status` | State, endpoint, hourly cost, model, health. |
| `ninfer destroy` | Delete the instance entirely — idle cost goes to $0.00. |
| `ninfer down` | Stop an instance, keeping the disk. **Still bills for storage.** |
| `ninfer up` | Resume a stopped instance, or restart serving on a running one. |
| `ninfer offers` | List the cards worth renting right now. |
| `ninfer ssh [cmd]` | Shell on the box. |
| `ninfer log` | Tail the server log. |
| `ninfer restore` | Reinstall the prebuilt binaries on the current box. |
| `ninfer kit-url` | Print the URL the kit would be fetched from. |
| `ninfer provision` | *(5090)* Finish a box that `create` left half-built, instead of renting another. |
| `ninfer build` | *(5090)* Compile `sm_120a` binaries on the current box from upstream source. |
| `ninfer kit-pack` | *(5090)* Tar this box's binaries and download the kit to `dist/`. |
| `ninfer log-build` | *(5090)* Tail the compile log. |
| `ninfer bench [C,C,…]` | *(5090)* Measure decode throughput across concurrency levels. |

`create` is `provision` plus renting, so a run that dies late — a timeout, a dropped SSH — can be
finished with `ninfer provision` on the box you already paid for. On the 3090 that split does not
exist yet; a failed `create` there means destroying and starting over.

### down vs destroy

`down` keeps the disk and keeps billing you for it — around 4¢/hr, roughly a dollar a day,
forever. `destroy` costs nothing at all while idle, and because the binaries come from a release
and the model comes from Hugging Face at multi-gigabit speed, a rebuild is only a few minutes.
**Unless you're coming back within the hour, destroy.**

### When a host can't start

Vast will occasionally rent you a machine whose own device map cannot hand the container its GPU:

```
failed to inject CDI devices: unresolvable CDI devices D.<hash>/gpu=0: unknown
```

Nothing in the offer listing predicts this — the two we hit scored 0.9951 and 0.9971 on
reliability. The script reads Vast's `status_msg` rather than guessing, fails immediately instead
of waiting out a timeout, and records the machine id in `.ninfer-badhosts` so `offers` and
`create` skip it from then on. Delete a line from that file to give a machine another chance.

## Configuration

Everything has a sane default and an environment override:

| Variable | Default | Purpose |
| --- | --- | --- |
| `NINFER_PROJ` | script's directory | Where `.env` and instance state live |
| `NINFER_SSH_KEY` | `~/.ssh/id_ed25519` | Key used to reach the box |
| `NINFER_DISK` | `60` (3090) / `80` (5090) | Disk size in GB |
| `NINFER_INSTANCE` | saved state file | Target a specific instance |
| `NINFER_MODEL` | `nvfp4-27b` | *(5090)* Which registered artifact to serve — see below |
| `NINFER_PROVIDER` | `ninfer5090` | *(5090)* Provider key written into OpenCode config |
| `NINFER_KIT_URL` | the published release | Where to fetch the kit |
| `NINFER_KIT_SHA1` | the published kit's SHA-1 | Blank it to compile from source instead |
| `NINFER_SRC_BRANCH` | `master` | *(5090)* Upstream branch `ninfer build` compiles |
| `NINFER_B2_ENV` | *(unset)* | Path to Backblaze B2 credentials, if you host your own copy |
| `NINFER_KIT_KEY` | *(unset)* | Object key of that private copy |

Per-card state lives beside the script and is all gitignored: `.env`, `.ninfer-instance`,
`.ninfer-badhosts`, and `dist/`.

### Choosing a model (5090)

Upstream accepts a closed set of artifacts and refuses everything else. Four of them fit in 32 GB:

| `NINFER_MODEL` | Artifact | Size | Note |
| --- | --- | --- | --- |
| `nvfp4-27b` *(default)* | Qwen3.6-27B NVFP4 | 17.07 GiB | W4A4 tensor cores; 5.67× at concurrency 8 |
| `int-27b` | Qwen3.6-27B `groupwise-int` | 16.29 GiB | 2.88× at concurrency 8 |
| `qwen38-27b` | Qwen3.8-27B `groupwise-int` | 16.96 GiB | The same checkpoint the 3090 runs |
| `moe-35b` | Qwen3.6-35B-A3B | 21.22 GiB | Fastest at concurrency 8; DFlash, text-only |

Each is checked against upstream's published SHA-256 after download. One engine holds one resident
artifact, so switching models means `destroy` and `create` again.

### OpenCode integration

If you use [OpenCode](https://opencode.ai), `create` and `up` rewrite your client config so the
address change after a restart doesn't silently break it. The two cards behave differently, on
purpose:

- **5090** — owns the provider key `ninfer5090` and **writes the whole block** if it is missing,
  reading the model id off `GET /v1/models` rather than assuming it. Then pick
  `ninfer5090/<model-id>` in the TUI. Restart OpenCode; config is read at startup.
- **3090** — updates the `baseURL` of an existing provider named `ninfer`, and skips a config that
  doesn't define one.

Edit the `CONFIGS` array near the top of the script to point at your own files, or at a different
client entirely.

## How provisioning works

`create` rents the box, then hands off to `provision`:

1. **Pick** — query Vast's bundles API for single-GPU offers with enough disk, ≥1 Gbps down, a new
   enough CUDA, enough VRAM, a reliability floor, and no entry in `.ninfer-badhosts`; take the
   cheapest survivor.
2. **Rent** — create the instance from `nvidia/cuda:13.1.2-devel-ubuntu24.04` with port 8080 mapped.
3. **Wait for SSH** — nothing else can happen until the box is reachable.
4. **Provision** — apt dependencies and the checkpoint pulled from Hugging Face, detached so a
   dropped connection doesn't kill it, then checksummed.
5. **Install the binaries** — the prebuilt kit, verified by SHA-1. With `NINFER_KIT_SHA1` blank the
   5090 compiles from upstream source here instead, which is what happened before a kit existed.
6. **Wait** — for the model download to land.
7. **Serve** — launch under a supervisor loop that restarts on crash, wait for HTTP 200, then
   rewrite client config with the new address.

Steps 4 and 5 run concurrently on the box. Provisioning owns `apt`, and the compile waits on a
marker file, because two `apt-get` runs at once deadlock on the dpkg lock.

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

## Measured throughput

`ninfer bench` runs on the box over localhost — no network in the number — and reports
`ninfer-serve`'s own per-request figures rather than counting stream chunks. That distinction
matters: MTP speculative decoding commits two to four tokens per round, so counting SSE deltas
undercounts decode by roughly 4×.

RTX 5090, Qwen3.6-27B NVFP4, 600 tokens per stream, `int8` KV, MTP-3 with `--lm-head-draft`:

| Concurrency | Per-stream tok/s | Aggregate tok/s | MTP tok/round | Acceptance |
| ---: | ---: | ---: | ---: | ---: |
| 1 | 180 | 180 | 2.7 | 57% |
| 2 | 175 | 338 | 2.6 | 53% |
| 4 | 170 | 649 | 2.7 | 57% |
| 8 | 147 | **1,056** | 2.7 | 56% |

Aggregate is total tokens over the wall time the burst occupied, computed from the requests
themselves. The engine's own 5-second interval report is *not* used, because it averages in
whatever idle time the window happens to span — at concurrency 1 that reads ~40% low, and at 8 it
swings by a third depending on where the burst lands.

**Sampling moves these numbers.** Acceptance, and therefore throughput, depends on how closely the
draft head predicts the sampler. The server's non-thinking defaults are `temperature 0.70`,
`top_p 0.80`, `presence_penalty 1.50`; sending `"temperature": 0` on the request instead:

| | Acceptance | Per-stream tok/s |
| --- | ---: | ---: |
| Server default, C=1 | 52% | 172 |
| Greedy, C=1 | **63%** | **194** |
| Server default, C=8 | 56% | 148 |
| Greedy, C=8 | **61%** | **155** |

For reference, upstream publishes 202.4 tok/s at C=1 and 1,146.9 at C=8 for this profile, at
68–69% acceptance, over 8,192-token generations. Shorter generations and a hotter sampler account
for most of the gap.

## Verified on

The published binaries have been run end to end, not merely compiled:

| | RTX 3090 | RTX 5090 |
| --- | --- | --- |
| Host | Vast.ai, Ubuntu 24.04 | Vast.ai, Ubuntu 24.04, driver 610.43.03 |
| Image | `nvidia/cuda:13.1.2-devel-ubuntu24.04` | `nvidia/cuda:13.1.2-devel-ubuntu24.04` |
| Engine | `0.6.1-rtx3090`, `sm_86` | rev `feaf4dd0`, `sm_120a` |
| Model | `Qwen3.8-27B-NInfer`, 18.2 GB | `Qwen3.6-27B-nvfp4-NInfer`, 17.07 GiB |
| Resident | 21,013 MiB of 24,576 MiB | not recorded before teardown |
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
