# Toolchain image for RBE node builds: the local runner AND the EngFlow remote
# worker image, so every compile sees the same gcc, headers and libraries no
# matter where it runs. The backend's runner image is built FROM this one
# (src/build/docker/linux_node_build.Dockerfile), so it follows whatever base
# is chosen here.
#
# Built the way upstream builds its own Linux binaries: an old base supplies
# the run-time libraries and a newer compiler is added on top of it, so the
# shipped binary only needs what the old base has. node 26 needs gcc >= 13.2
# (BUILDING.md) -- 12 rejects a non-dependent static_assert in a discarded
# `if constexpr` branch (v8/src/compiler/turboshaft/assembler.h,
# builtins-iterator-inl.h), and libstdc++ 11's std::ranges::split_view iterator
# isn't bidirectional, which src/node_options.cc needs -- and gcc-toolset-13
# gives that on RHEL 8 while linking the symbols newer than the base's
# libstdc++ statically. The binary therefore requires only glibc 2.28 and
# GLIBCXX_3.4.25, which is what upstream's own linux-x64 release requires.
#
# Installing a newer gcc over a newer base instead (a noble base, or jammy with
# the ubuntu-toolchain-r/test PPA) raises both floors with it: the PPA's gcc 13
# brings its own libstdc++, and the binary then needs GLIBCXX_3.4.32, which
# Ubuntu 22.04 does not have -- it would not run on the machines that record.
#
# The build scripts invoke /usr/bin/gcc and /usr/bin/g++ by absolute path, on
# the remote worker as well (cc-wrapper.sh, TOOLCHAIN_PROBE), so the toolset is
# exposed there rather than only on PATH.
#
# Published by toolchain-image.sh as public.ecr.aws/z3x5l6b1/node-rbe-toolchain:
# <hash of this file>; any change here produces a new image (and remote cache
# keys), so each node version's branch pins its own and they coexist in the
# repository. It is PUBLIC: never COPY repo contents or credentials into it.
FROM registry.access.redhat.com/ubi8/ubi:8.10

# RHEL 8's default python3 (3.6) and nodejs (10) are too old for node's own
# build: configure looks for python3.10 or newer, and build.js is ESM-era.
RUN dnf -y module enable nodejs:20 && \
    dnf -y install --setopt=install_weak_deps=False \
      gcc-toolset-13-gcc gcc-toolset-13-gcc-c++ gcc-toolset-13-binutils \
      make git curl python3.12 python3.12-pip nodejs tar gzip diffutils findutils which && \
    alternatives --install /usr/bin/python3 python3 /usr/bin/python3.12 100 && \
    python3.12 -m pip install --no-cache-dir awscli && \
    dnf clean all && rm -rf /var/cache/dnf
RUN for t in gcc g++ cpp gcov ar nm objdump ranlib strip ld; do \
      src=/opt/rh/gcc-toolset-13/root/usr/bin/$t; \
      if [ -x "$src" ]; then printf '#!/bin/sh\nexec %s "$@"\n' "$src" > /usr/bin/$t; chmod 755 /usr/bin/$t; fi; \
    done
RUN git config --system --add safe.directory '*'
# The user EngFlow workers run actions as (same as backend's engflow-builder image).
RUN groupadd --gid 114 engflow && \
    useradd --home-dir /home/engflow --create-home --uid 108 --gid 114 engflow

# build-rbe.py checks it runs in the image matching this Dockerfile.
ARG TOOLCHAIN_TAG
RUN test -n "$TOOLCHAIN_TAG" && echo "$TOOLCHAIN_TAG" > /etc/node-rbe-toolchain
