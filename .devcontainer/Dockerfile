FROM nixos/nix

# This is so VSCode can tell it's Nix and skip chcking glibc since it doesn't know how to do it
RUN echo 'ID=nixos' > /etc/os-release

RUN nix-channel --update
RUN nix-env --install --attr \
    nixpkgs.nil \
    nixpkgs.nixd \
    nixpkgs.alejandra \
    nixpkgs.nixfmt \
    nixpkgs.gnused \
    nixpkgs.patchelf

# Luckily, vscode code-server has support for ELF patching the node binary, we will leverage that to make it work in Nix
# If they ever remove this support, it's easy enough to just run patchelf outselves.  You can see how it works at
# https://github.com/microsoft/vscode/blob/31ec1c528f3352405f329821df31b91e6deba824/resources/server/bin/code-server-linux.sh
# We need an easy to reference location for the binaries and libraries we will need for the binary patch
RUN mkdir -p /root/lib /root/bin && \
    ln -s $(nix-instantiate --eval-only --expr '(import <nixpkgs> {}).glibc.outPath'  | sed -e 's/"//g') /root/lib/glibc && \
    ln -s $(nix-instantiate --eval-only --expr '(import <nixpkgs> {}).libgcc.lib.outPath'  | sed -e 's/"//g') /root/lib/libgcc && \
    ln -s $(which patchelf) /root/bin/patchelf

# Finally we will set some env vars to tell vscode where to find them so it can patch nodejs
# Detect the architecture at build time and use it to set the ENV
ARG BUILDARCH
RUN ARCH=$(uname -m) && \
    echo "Building for architecture: $ARCH" && \
    if [ "$ARCH" = "x86_64" ]; then \
    echo "ld-linux-x86-64.so.2" > /tmp/linker-name; \
    elif [ "$ARCH" = "aarch64" ]; then \
    echo "ld-linux-aarch64.so.1" > /tmp/linker-name; \
    else \
    echo "Unsupported architecture: $ARCH" >&2 && exit 1; \
    fi && \
    echo "Linker name: $(cat /tmp/linker-name)"

ENV VSCODE_SERVER_CUSTOM_GLIBC_PATH="/root/lib/glibc/lib:/root/lib/libgcc/lib"
ENV VSCODE_SERVER_PATCHELF_PATH="/root/bin/patchelf"

# Set the linker path based on detected architecture
# Note: Docker ENV can't use command substitution, so we have to set this statically
# The build script or devcontainer.json will need to handle multi-arch builds
RUN mkdir -p /etc/profile.d && \
    ARCH=$(uname -m) && \
    if [ "$ARCH" = "aarch64" ]; then \
    echo "export VSCODE_SERVER_CUSTOM_GLIBC_LINKER=/root/lib/glibc/lib/ld-linux-aarch64.so.1" > /etc/profile.d/00-vscode-linker.sh; \
    else \
    echo "export VSCODE_SERVER_CUSTOM_GLIBC_LINKER=/root/lib/glibc/lib/ld-linux-x86-64.so.2" > /etc/profile.d/00-vscode-linker.sh; \
    fi && \
    chmod +x /etc/profile.d/00-vscode-linker.sh

# For ARM64 systems (like Apple Silicon)
ENV VSCODE_SERVER_CUSTOM_GLIBC_LINKER="/root/lib/glibc/lib/ld-linux-aarch64.so.1"
