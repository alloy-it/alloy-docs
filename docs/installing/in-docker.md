# In a Docker Image

Use alloy-provisioner inside a Docker build to create reproducible CI base images from a declarative blueprint, the same one your developers use on their workstations.

---

## The problem with complex Dockerfiles

Large embedded projects often end up with hundreds of imperative `RUN` lines in a Dockerfile, chains of `FROM` dependencies between images, and ad-hoc install scripts duplicated across repos. When the toolchain changes, multiple Dockerfiles need updating. When a junior dev needs to understand the environment, they have to parse imperative shell code.

Blueprints solve this:

- **Declarative**: describe what you want, not how to do it.
- **Reproducible**: the same blueprint gives the same result every time, with SHA256-verified downloads.
- **Shared**: the same blueprint works in Docker, in a local VM, and on a CI server. No drift between developer environments and CI.
- **Maintainable**: layers approach splits the environment into readable, independently updatable files.

---

## How it works

Add alloy-provisioner to your Docker build, copy the blueprint in, run it, then remove the blueprint files to keep the final image lean:

```dockerfile
FROM ubuntu:24.04

# Non-interactive APT and safer shell
ENV DEBIAN_FRONTEND=noninteractive
SHELL ["/bin/bash", "-eo", "pipefail", "-c"]

# Install minimal dependencies for alloy-provisioner
RUN apt-get update -qq \
    && apt-get install --no-install-recommends -y \
        wget \
        ca-certificates \
        rsync \
        tar \
    && rm -rf /var/lib/apt/lists/*

# Download and install alloy-provisioner
ARG PROVISIONER_VERSION=latest
RUN ARCH=$(dpkg --print-architecture) \
    && wget -q -O /tmp/alloy-provisioner.deb \
        "https://github.com/alloy-it/alloy-provisioner-releases/releases/download/${PROVISIONER_VERSION}/alloy-provisioner_${PROVISIONER_VERSION}_linux_${ARCH}.deb" \
    && apt-get update -qq \
    && apt-get install -y /tmp/alloy-provisioner.deb \
    && rm -f /tmp/alloy-provisioner.deb \
    && rm -rf /var/lib/apt/lists/*

# Copy the blueprint and run the provisioner
COPY blueprint/ /tmp/blueprint/
RUN alloy-provisioner --blueprint-dir /tmp/blueprint \
    && rm -rf /tmp/blueprint
```

Place this Dockerfile next to your blueprint directory:

```toml
my-ci-image/
├── Dockerfile
└── blueprint/
    ├── manifest.yml
    ├── variables.yml       # optional
    ├── 00-system.yml
    ├── 10-toolchain.yml
    ├── 20-environment.yml
    └── alloy.lock.yml
```

Build the image:

```bash
docker build -t my-arm-ci-env .
```

Pin the provisioner version for reproducible builds:

```bash
docker build --build-arg PROVISIONER_VERSION=1.2.3 -t my-arm-ci-env .
```

---

## Architecture-aware builds

The Dockerfile above auto-detects the build architecture via `dpkg --print-architecture`. Your blueprint should use `per_arch` overrides to supply the correct toolchain URL for each architecture:

```yaml
# blueprint/10-toolchain.yml
- name: Install ARM GCC toolchain
  action: unarchive_from_url
  dest: /opt/arm-none-eabi
  creates: /opt/arm-none-eabi/bin/arm-none-eabi-gcc
  per_arch:
    amd64:
      url: https://example.com/arm-gcc-13-amd64.tar.xz
      sha256: "aaa111..."
    arm64:
      url: https://example.com/arm-gcc-13-arm64.tar.xz
      sha256: "bbb222..."
```

This produces correct images when building on both `linux/amd64` and `linux/arm64` hosts, or when using Docker BuildKit multi-platform builds.

---

## Passing variables and credentials

If your blueprint uses `required_env:` or environment variables (for private registries, credentials, or environment-specific settings), pass them as Docker build secrets or `--build-arg`:

```bash
# Pass as environment variable (visible in docker inspect — use secrets for sensitive values)
docker build --build-arg GITLAB_TOKEN=xxx -t my-env .
```

For sensitive values, use [Docker BuildKit secrets](https://docs.docker.com/build/building/secrets/):

```dockerfile
RUN --mount=type=secret,id=gitlab_token \
    GITLAB_TOKEN=$(cat /run/secrets/gitlab_token) \
    alloy-provisioner --blueprint-dir /tmp/blueprint \
    && rm -rf /tmp/blueprint
```

```bash
docker build --secret id=gitlab_token,src=.env -t my-env .
```

---

## CI integration

Use the built image in GitHub Actions:

```yaml
# .github/workflows/build.yml
jobs:
  build:
    runs-on: ubuntu-latest
    container:
      image: ghcr.io/your-org/my-arm-ci-env:latest
    steps:
      - uses: actions/checkout@v4
      - name: Build firmware
        run: |
          cmake -GNinja -DCMAKE_TOOLCHAIN_FILE=cmake/arm-none-eabi.cmake -B build
          ninja -C build
```

Or build the image as part of CI and push it to a registry:

```yaml
jobs:
  build-image:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - name: Build CI image
        run: docker build -t ghcr.io/your-org/my-arm-ci-env:latest .
      - name: Push
        run: docker push ghcr.io/your-org/my-arm-ci-env:latest
```

---

## Comparing with developer VMs

The same blueprint directory works for both Docker CI images and developer VMs managed by alloy-host. No duplication:

| Context       | How provisioner is invoked                                        |
| ------------- | ----------------------------------------------------------------- |
| Docker build  | `RUN alloy-provisioner --blueprint-dir /tmp/blueprint`            |
| alloy-host VM | Bootstrap script runs `alloy-provisioner --blueprint-dir /config` |
| Native Linux  | `alloy-provisioner --blueprint-dir /path/to/blueprint`            |

All three produce the same environment. Developers and CI use identical tool versions.

---

## Next steps

- [Write your own blueprint](../blueprints/index.md)
- [The layers approach](../blueprints/layers.md): keep your Dockerfile-replacing blueprint maintainable
- [Variables & configuration](../blueprints/variables.md): credentials, env files, and required variables
