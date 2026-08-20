# Attribution — NInfer RTX 5090 binaries

The `ninfer` and `ninfer-serve` binaries distributed by this project's GitHub Releases are
**not original work of this project**. They are compiled, unmodified, from:

| | |
| --- | --- |
| Project | [Neroued/ninfer](https://github.com/Neroued/ninfer) |
| Branch | `master` |
| Revision | _pending first build_ |
| Version string | none — upstream publishes no tags or releases |
| License | Apache License 2.0 — see [`LICENSE`](LICENSE) |

## What was and wasn't changed

**No source changes were made.** The binaries are a stock build of the revision above, configured
exactly as upstream's own `Dockerfile` configures them.

The only difference from what upstream would produce is the build environment, recorded here so
the result is reproducible:

| | |
| --- | --- |
| Target architecture | `sm_120a` (RTX 5090) |
| Build image | `nvidia/cuda:13.1.2-devel-ubuntu24.04` |
| Build host | Vast.ai RTX 5090, Ubuntu 24.04 |
| Build time | _pending first build_ |
| Built | _pending first build_ |

Upstream's `CMakeLists.txt` rejects any value of `CMAKE_CUDA_ARCHITECTURES` other than `120a` and
any CUDA compiler older than 13.1, so neither is a choice this project made.

Every tarball also carries a `BUILDINFO.txt` recording the exact revision, CUDA version, driver
version, GPU, job count and wall-clock build time of that specific build.

## Why this exists

Upstream ships no binaries for any platform. There are no releases and no tags on the repository,
and their README states:

> There is no install target or packaged binary distribution; NInfer is run from its source build
> tree.

Their only packaging path is a `Dockerfile` you build yourself, which requires a machine that
already has an RTX 5090 and a CUDA 13.1 toolchain. This project publishes the output of that build
so that renting a 5090 for an hour does not mean spending the first half of it compiling.

## Apache-2.0 obligations

Per §4 of the Apache License 2.0, redistribution of these binaries carries the full text of the
license (included as [`LICENSE`](LICENSE) in this directory and as an asset on every release that
contains binaries), retains all attribution notices, and states the origin above. Upstream ships
no `NOTICE` file, so there is none to propagate.

The Apache-2.0 license covers **the binaries only**. The `ninfer` orchestration script in this
repository is separate work under the MIT License — see the repository root [`LICENSE`](../../LICENSE).

## Model artifacts

The `.ninfer` checkpoints this CLI downloads are **not** redistributed by this project; they are
pulled directly from their Hugging Face repositories at provision time. They are derived from
[Qwen/Qwen3.6-27B](https://huggingface.co/Qwen/Qwen3.6-27B),
[Qwen/Qwen3.8-27B](https://huggingface.co/Qwen/Qwen3.8-27B) and
[Qwen/Qwen3.6-35B-A3B](https://huggingface.co/Qwen/Qwen3.6-35B-A3B), all Apache-2.0.
