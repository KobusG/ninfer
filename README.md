# ninfer

Run **Qwen3.8-27B** on a rented **RTX 3090**, and pay nothing when you're idle.

`ninfer` is a single bash script that rents a GPU on [Vast.ai](https://vast.ai), provisions it
from scratch, serves an OpenAI-compatible API, and tears the whole thing down when you're done.
Idle cost is **$0.00/hr** — you destroy the box instead of leaving it parked, and a rebuild takes
a couple of minutes because the compiled binaries are cached off-box.

A typical session costs about a quarter an hour, and nothing at all overnight.

```
ninfer create     # rent a fresh 3090 and provision it (~2-8 min)
ninfer status     # state, endpoint, hourly cost, health
ninfer destroy    # delete it — idle cost goes to $0.00
```

## Why this exists

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

## The prebuilt kit

Step 5 pulls a tarball of binaries from a **private** Backblaze B2 bucket. That bucket is not
public, so **`ninfer create` will fail at step 5 unless you supply your own.**

You have two options:

- **Build the engine yourself** from [Don-Chad/ninfer-3090](https://github.com/Don-Chad/ninfer-3090),
  tar up `bin/` and `scripts/`, and host it wherever you like — then point `NINFER_B2_ENV`,
  `NINFER_KIT_KEY` and `KIT_SHA1` at your copy.
- **Compile on the box** each time, by replacing the `install_kit` step with a build. Costs about
  883 seconds per provision, which is exactly the tax this script exists to avoid.

`ninfer kit-url` shows how the signed URL is minted if you want to mirror the approach.

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
