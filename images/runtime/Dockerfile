# Copyright Authors of Cilium
# SPDX-License-Identifier: Apache-2.0

ARG TESTER_IMAGE=quay.io/cilium/image-tester:0a7ee27812441d95926aec83929d97e93ce96aae@sha256:e96542b4f71dbc9cbe77feaf5b1fa9bd5e13122ee6418094731212f1c5667c67
ARG GOLANG_IMAGE=docker.io/library/golang:1.23.2@sha256:ad5c126b5cf501a8caef751a243bb717ec204ab1aa56dc41dc11be089fafcb4f
ARG UBUNTU_IMAGE=docker.io/library/ubuntu:24.04@sha256:99c35190e22d294cdace2783ac55effc69d32896daaa265f0bbedbcde4fbe3e5

ARG CILIUM_LLVM_IMAGE=quay.io/cilium/cilium-llvm:02191b3a68f4003f075e69a61b61faa8d12d4b6d@sha256:54ca04f65a6de3aa5243a7da392aed342c64f579ebd94ea1290590ce77a433a8
ARG CILIUM_BPFTOOL_IMAGE=quay.io/cilium/cilium-bpftool:5a9c4852a21287686009bfe1cdc1fed6e7aabdea@sha256:188e398ee30456373530698d0d29de66dda0c4428068ea430027ceb7e9c15b7c
ARG CILIUM_IPTABLES_IMAGE=quay.io/cilium/iptables:1331e9b1b03f70c9d8b08626d9a7126963f86478@sha256:d761d967243aced2729adde1e332a9c9def6baeb61f5f6cde5758b04e9a79355

FROM ${CILIUM_LLVM_IMAGE} AS llvm-dist
FROM ${CILIUM_BPFTOOL_IMAGE} AS bpftool-dist
FROM ${CILIUM_IPTABLES_IMAGE} AS iptables-dist

FROM --platform=${BUILDPLATFORM} ${GOLANG_IMAGE} AS gops-cni-builder

RUN apt-get update && apt-get install -y binutils-aarch64-linux-gnu binutils-x86-64-linux-gnu

# build-gops.sh will build both archs at the same time
WORKDIR /go/src/github.com/cilium/cilium/images/runtime
RUN --mount=type=bind,readwrite,target=/go/src/github.com/cilium/cilium/images/runtime --mount=target=/root/.cache,type=cache --mount=target=/go/pkg/mod,type=cache \
    ./build-gops.sh
# download-cni.sh will build both archs at the same time
RUN --mount=type=bind,readwrite,target=/go/src/github.com/cilium/cilium/images/runtime --mount=target=/root/.cache,type=cache --mount=target=/go/pkg/mod,type=cache \
    ./download-cni.sh

FROM ${UBUNTU_IMAGE} AS rootfs

# Change the number to force the generation of a new git-tree SHA. Useful when
# we want to re-run 'apt-get upgrade' for stale images.
ENV FORCE_BUILD=4

# Update ubuntu packages to the most recent versions
RUN apt-get update && \
    apt-get upgrade -y && \
    apt-get install -y jq

WORKDIR /go/src/github.com/cilium/cilium/images/runtime
RUN --mount=type=bind,readwrite,target=/go/src/github.com/cilium/cilium/images/runtime \
    ./install-runtime-deps.sh

COPY --from=iptables-dist /iptables /iptables
RUN dpkg -i /iptables/*\.deb && rm -rf /iptables

RUN --mount=type=bind,readwrite,target=/go/src/github.com/cilium/cilium/images/runtime \
    ./iptables-wrapper-installer.sh --no-sanity-check

COPY --from=llvm-dist /usr/local/bin/clang /usr/local/bin/llc /usr/local/bin/
COPY --from=bpftool-dist /usr/local /usr/local

ARG TARGETPLATFORM
COPY --from=gops-cni-builder /out/${TARGETPLATFORM}/bin/loopback /cni/loopback
COPY --from=gops-cni-builder /out/${TARGETPLATFORM}/bin/gops /bin/gops

FROM ${TESTER_IMAGE} AS test
COPY --from=rootfs / /
COPY --from=llvm-dist /test /test
COPY --from=bpftool-dist /test /test
RUN /test/bin/cst -C /test/llvm
RUN /test/bin/cst -C /test/bpftool

FROM scratch
LABEL maintainer="maintainer@cilium.io"
COPY --from=rootfs / /
