# Global ARGs shared by all stages
ARG DEBIAN_FRONTEND=noninteractive
ARG GOPATH=/usr/local/go

### first stage - builder ###
FROM debian:bullseye-slim as builder

ARG DEBIAN_FRONTEND
ARG GOPATH
ENV GOPATH=${GOPATH}

# install debos build and unit-test dependencies
RUN apt-get update && \
    apt-get install -y --no-install-recommends \
        ca-certificates \
        gcc \
        git \
        golang-go \
        libc6-dev \
        libostree-dev \
        unzip && \
    rm -rf /var/lib/apt/lists/*

# Build debos
COPY . $GOPATH/src/github.com/go-debos/debos
WORKDIR $GOPATH/src/github.com/go-debos/debos/cmd/debos
RUN go install ./...

# install pacman build dependencies
# NOTE: the list of packages is copied verbatum from pacman's Debian CI step.
# Some bits may be repetative or not strictly needed - but we can live with that
# in the short them.
RUN apt-get update && \
    apt-get install -y --no-install-recommends \
        git pkg-config meson gcc libtool \
        libgpgme-dev libarchive-dev libcurl4-openssl-dev libssl-dev curl \
        gettext python3 python3-setuptools dash gawk ca-certificates \
        fakeroot fakechroot && \
    rm -rf /var/lib/apt/lists/*

# Build pacman
RUN git clone https://gitlab.archlinux.org/pacman/pacman/ && \
    cd pacman && \
    git checkout -B latest-release v6.0.2 && \
    meson setup build && \
    DESTDIR=/pacman ninja install -C build

### second stage - runner ###
FROM debian:bullseye-slim as runner

ARG DEBIAN_FRONTEND
ARG GOPATH

# Set HOME to a writable directory in case something wants to cache things
ENV HOME=/tmp

LABEL org.label-schema.name "debos"
LABEL org.label-schema.description "Debian OS builder"
LABEL org.label-schema.vcs-url = "https://github.com/go-debos/debos"
LABEL org.label-schema.docker.cmd 'docker run \
  --rm \
  --interactive \
  --tty \
  --device /dev/kvm \
  --user $(id -u) \
  --workdir /recipes \
  --mount "type=bind,source=$(pwd),destination=/recipes" \
  --security-opt label=disable'

# debos runtime dependencies
# ca-certificates is required to validate HTTPS certificates when getting debootstrap release file
RUN apt-get update && \
    apt-get install -y --no-install-recommends \
        apt-transport-https \
        binfmt-support \
        bmap-tools \
        btrfs-progs \
        busybox \
        bzip2 \
        ca-certificates \
        debootstrap \
        dosfstools \
        e2fsprogs \
        equivs \
        fdisk \
        f2fs-tools \
        git \
        gzip \
        pigz \
        libostree-1-1 \
        libslirp-helper \
        linux-image-amd64 \
        openssh-client \
        parted \
        pkg-config \
        qemu-system-x86 \
        qemu-user-static \
        qemu-utils \
        rsync \
        systemd \
        systemd-container \
        u-boot-tools \
        unzip \
        user-mode-linux \
        xfsprogs \
        xz-utils \
        zip && \
    rm -rf /var/lib/apt/lists/*

# Enable backports for the Arch dependencies
RUN echo "deb http://ftp.debian.org/debian bullseye-backports main" >> /etc/apt/sources.list

# Install Arch dependencies
RUN apt-get update && \
    apt-get install -y --no-install-recommends \
        arch-install-scripts \
        archlinux-keyring && \
    rm -rf /var/lib/apt/lists/*

# debian's qemu-user-static package no longer registers binfmts
# if running inside a virtualmachine; dockerhub builds are inside a vm
RUN for arch in aarch64 alpha arm armeb cris hexagon hppa m68k microblaze mips mips64 mips64el mipsel mipsn32 mipsn32el ppc ppc64 ppc64le riscv32 riscv64 s390x sh4 sh4eb sparc sparc32plus sparc64 xtensa xtensaeb; do \
      update-binfmts --import qemu-$arch; \
    done

COPY --from=builder $GOPATH/bin/debos /usr/local/bin/debos

COPY --from=builder /pacman/* /

RUN ls -lah /usr/bin/pacman*
RUN ls -lah /usr/share/makepkg/

ENTRYPOINT ["/usr/local/bin/debos"]
