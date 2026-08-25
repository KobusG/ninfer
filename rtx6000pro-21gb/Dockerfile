# syntax=docker/dockerfile:1

ARG BUILD_IMAGE=nvidia/cuda:13.1.2-devel-ubuntu24.04@sha256:b9f64abf7226fdb3463ca202bc99878ec847171e6c5f77bd34c8d1403fbf1eca
ARG RUNTIME_IMAGE=nvidia/cuda:13.1.2-runtime-ubuntu24.04@sha256:bff001d3257971cc4752e15ac2d354befa70995ded8e141741ade50569fc192e

FROM ${BUILD_IMAGE} AS build

ARG DEBIAN_FRONTEND=noninteractive
ARG NINFER_REPO=https://github.com/KobusG/ninfer-engine.git
ARG NINFER_REF

RUN apt-get update \
    && apt-get install --yes --no-install-recommends \
        ca-certificates \
        cmake \
        git \
        libavcodec-dev \
        libavformat-dev \
        libavutil-dev \
        libcurl4-openssl-dev \
        libswscale-dev \
        ninja-build \
        pkg-config \
    && rm -rf /var/lib/apt/lists/*

WORKDIR /src
RUN test -n "$NINFER_REF" \
    && git init \
    && git remote add origin "$NINFER_REPO" \
    && git fetch --depth 1 origin "$NINFER_REF" \
    && git checkout --detach FETCH_HEAD \
    && test "$(git rev-parse HEAD)" = "$NINFER_REF"

RUN cmake -S . -B /build -G Ninja \
        -DCMAKE_BUILD_TYPE=Release \
        -DCMAKE_CUDA_ARCHITECTURES=120a \
        -DNINFER_BUILD_APPS=ON \
        -DBUILD_TESTING=OFF \
        -DNINFER_BUILD_BENCHMARKS=OFF \
    && cmake --build /build -j --target ninfer ninfer-serve

FROM ${RUNTIME_IMAGE}

ARG DEBIAN_FRONTEND=noninteractive
ARG NINFER_REPO=https://github.com/KobusG/ninfer-engine.git
ARG NINFER_REF

LABEL org.opencontainers.image.source="${NINFER_REPO}" \
      org.opencontainers.image.revision="${NINFER_REF}" \
      org.opencontainers.image.description="NInfer sm120a quantized-KV and bounded-vision runtime" \
      org.opencontainers.image.licenses="Apache-2.0"

RUN apt-get update \
    && apt-get install --yes --no-install-recommends \
        ca-certificates \
        curl \
        ffmpeg \
        python3 \
        python3-venv \
    && rm -rf /var/lib/apt/lists/*

RUN mkdir -p /root/ninfer-5090/build-sm120a/apps /usr/share/doc/ninfer
COPY --from=build /build/apps/ninfer /root/ninfer-5090/build-sm120a/apps/ninfer
COPY --from=build /build/apps/ninfer-serve /root/ninfer-5090/build-sm120a/apps/ninfer-serve
COPY --from=build /src/LICENSE /usr/share/doc/ninfer/LICENSE
RUN ln -s /root/ninfer-5090/build-sm120a/apps/ninfer /usr/local/bin/ninfer \
    && ln -s /root/ninfer-5090/build-sm120a/apps/ninfer-serve /usr/local/bin/ninfer-serve \
    && printf 'source %s\nrevision %s\narchitecture sm_120a\nCUDA 13.1.2\n' \
        "$NINFER_REPO" "$NINFER_REF" > /usr/share/doc/ninfer/BUILDINFO

WORKDIR /workspace
EXPOSE 8080
STOPSIGNAL SIGTERM
CMD ["ninfer-serve", "--help"]
