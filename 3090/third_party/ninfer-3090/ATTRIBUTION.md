# Attribution — NInfer RTX 3090 binaries

The `ninfer` and `ninfer-serve` binaries distributed by this project's GitHub Releases are
**not original work of this project**. They are compiled, unmodified, from:

| | |
| --- | --- |
| Project | [Don-Chad/ninfer-3090](https://github.com/Don-Chad/ninfer-3090) |
| Upstream of that | [Neroued/ninfer](https://github.com/Neroued/ninfer) |
| Branch | `release/v0.6.0-rtx3090` |
| Revision | `403fc56d71576aa1feddb771cfed3264e7378b20` |
| Version string | `0.6.1-rtx3090` |
| License | Apache License 2.0 — see [`LICENSE`](LICENSE) |

## What was and wasn't changed

**No source changes were made.** The binaries are a stock build of the revision above.

The only difference from what upstream would produce is the build environment, recorded here so
the result is reproducible:

| | |
| --- | --- |
| Target architecture | `sm_86` (RTX 3090 / 3090 Ti) |
| Build image | `nvidia/cuda:13.1.2-devel-ubuntu24.04` |
| Build host | Vast.ai RTX 3090, Ubuntu 24.04 |
| Build time | 883 s at `-j12` |
| Built | 2026-08-19 14:58:43 UTC |

This build exists because upstream does not ship one. Their
[`RELEASE_NOTES_0.6.1.md`](https://github.com/Don-Chad/ninfer-3090/blob/main/RELEASE_NOTES_0.6.1.md)
states: *"The project does not publish a prebuilt Linux archive."* Their last release carrying a
Linux tarball was `v0.3.1-rtx3090`; every release since ships Windows binaries only.

## Apache-2.0 obligations

Per §4 of the Apache License 2.0, redistribution of these binaries carries the full text of the
license (included as [`LICENSE`](LICENSE) in this directory and as an asset on every release that
contains binaries), retains all attribution notices, and states the origin above. Upstream ships
no `NOTICE` file, so there is none to propagate.

The Apache-2.0 license covers **the binaries only**. The `ninfer` orchestration script in this
repository is separate work under the MIT License — see the repository root [`LICENSE`](../../LICENSE).
