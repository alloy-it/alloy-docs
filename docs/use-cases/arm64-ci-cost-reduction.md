# Migrating CI/CD from x86_64 to arm64 to Reduce Costs

## The opportunity

arm64 cloud runners are substantially cheaper than their x86_64 equivalents. AWS Graviton instances
deliver 20–40 % better price/performance than comparable x86 instances. GitHub-hosted arm64 runners
cost less than their x86 counterparts for the same compute capacity. Self-hosted arm64 machines
(Raspberry Pi clusters, Ampere VMs, on-prem Apple Silicon) have even lower running costs.

For embedded teams doing continuous builds (firmware, BSPs, cross-compiled packages), CI minutes
add up fast. Switching the CI runner architecture is one of the highest-leverage cost levers available.

The catch is that CI environments on x86 often rely on scripts, Docker base images, or ad-hoc
toolchain installs that were never tested on arm64. Migrating means untangling all of that.

Alloy removes the untangling. **The same blueprint that provisions your x86 CI runner provisions
your arm64 runner.** alloy-provisioner is a native arm64 binary; it detects the runner architecture
and pulls the correct toolchain variants automatically.

---

## What you need

- A blueprint that already works for your local or x86 CI build
- An arm64 runner (AWS Graviton, GitHub arm64-hosted, self-hosted Raspberry Pi/Ampere, etc.)
- No changes to the blueprint itself: architecture is handled at the provisioner level

---

## Migration steps

### 1. Confirm your blueprint installs on arm64 locally

If you have an arm64 machine available (Apple Silicon Mac with Alloy Host, an AWS Graviton instance,
a Raspberry Pi 4), verify the install before touching CI:

```bash
sudo alloy-provisioner install community/arm-none-eabi
# or your private blueprint
sudo alloy-provisioner install myorg/my-blueprint:1.4.0
```

alloy-provisioner's install script detects the host architecture and selects the correct binary.
Most toolchains in the Alloy catalog ship fat blueprints with both `amd64` and `arm64` layers.
If a tool in your custom blueprint has an x86-only step, the provisioner will surface that clearly
rather than silently producing a broken environment.

### 2. Add an arm64 job to your CI pipeline

Start by running the arm64 job in parallel with the existing x86 job. Do not remove x86 until you
have confirmed the arm64 output is identical.

=== "GitHub Actions"

    ```yaml
    jobs:
      build-x86:
        runs-on: ubuntu-latest          # existing job, untouched
        steps:
          - uses: actions/checkout@v4
          - name: Install provisioner
            run: curl -fsSL https://raw.githubusercontent.com/alloy-it/alloy-provisioner-releases/main/scripts/install.sh | bash
          - name: Provision
            run: sudo alloy-provisioner install myorg/my-blueprint:1.4.0
          - name: Build
            run: source /etc/alloy/env && make build

      build-arm64:
        runs-on: ubuntu-24.04-arm        # GitHub arm64 runner
        steps:
          - uses: actions/checkout@v4
          - name: Install provisioner
            run: curl -fsSL https://raw.githubusercontent.com/alloy-it/alloy-provisioner-releases/main/scripts/install.sh | bash
          - name: Provision
            run: sudo alloy-provisioner install myorg/my-blueprint:1.4.0
          - name: Build
            run: source /etc/alloy/env && make build
    ```

=== "GitLab CI"

    ```yaml
    build-x86:
      tags: [linux-amd64]
      script:
        - curl -fsSL https://raw.githubusercontent.com/alloy-it/alloy-provisioner-releases/main/scripts/install.sh | bash
        - sudo alloy-provisioner install myorg/my-blueprint:1.4.0
        - source /etc/alloy/env && make build

    build-arm64:
      tags: [linux-arm64]       # arm64 runner tag
      script:
        - curl -fsSL https://raw.githubusercontent.com/alloy-it/alloy-provisioner-releases/main/scripts/install.sh | bash
        - sudo alloy-provisioner install myorg/my-blueprint:1.4.0
        - source /etc/alloy/env && make build
    ```

=== "Self-hosted (any CI)"

    ```bash
    # On the arm64 runner, same commands as any Linux machine
    curl -fsSL https://raw.githubusercontent.com/alloy-it/alloy-provisioner-releases/main/scripts/install.sh | bash
    sudo alloy-provisioner install myorg/my-blueprint:1.4.0
    source /etc/alloy/env
    make build
    ```

### 3. Validate build output parity

Cross-compilation means your build target (e.g. `arm-none-eabi`, `aarch64-linux-gnu`) does not
change when you switch the CI host architecture. The compiler running on the arm64 host still
produces firmware for your embedded target.

Useful checks:

```bash
# Confirm the cross-compiler is the expected version
arm-none-eabi-gcc --version

# Confirm the build target hasn't changed
file build/firmware.elf
# should still say: ELF 32-bit LSB executable, ARM, ...

# Compare binary checksums between x86 and arm64 CI builds
# (expect identical output for deterministic builds)
sha256sum build/firmware.elf
```

### 4. Cut over to arm64 only

Once parity is confirmed, remove the x86 job and update the runner target:

```yaml
jobs:
  build:
    runs-on: ubuntu-24.04-arm
    steps:
      - uses: actions/checkout@v4
      - name: Install provisioner
        run: curl -fsSL https://raw.githubusercontent.com/alloy-it/alloy-provisioner-releases/main/scripts/install.sh | bash
      - name: Provision
        run: sudo alloy-provisioner install myorg/my-blueprint:1.4.0
      - name: Build
        run: source /etc/alloy/env && make build
```

---

## Speed up repeated CI runs with blueprint caching

alloy-provisioner is idempotent: re-running `install` on an already-provisioned runner skips
completed steps. On persistent self-hosted runners or when using a cached runner image, you can
pre-provision once and reuse:

```yaml
- name: Provision (cached)
  run: |
    # Only re-provisions if the blueprint version changed
    sudo alloy-provisioner install myorg/my-blueprint:1.4.0
```

For ephemeral runners, build a base image with the blueprint already installed and use that as
your runner image. See [In a Docker Image](../installing/in-docker.md) for the Docker pattern.

---

## Troubleshooting

**`exec format error` when running a tool after provisioning**

A binary in your blueprint was packaged for x86 only. Check your blueprint's tool sources: any
`download_url` pointing to an `x86_64`/`amd64`-specific binary will fail silently or with this
error on arm64. Replace with a multi-arch download or add an arm64 variant. See
[Variables & Configuration](../blueprints/variables.md) for using `{{ arch }}` substitution.

**Build output differs between x86 and arm64 CI**

The cross-compiler itself is the same (same blueprint, same pinned version) so the compiled
firmware should be bit-identical. A difference usually means the build script has a host-arch
assumption (e.g. a helper binary, a Python script with a native extension, or a host-compiled
intermediate step). Check `uname -m` usage in your Makefile or build system.

**Runner label not found (GitHub Actions)**

`ubuntu-24.04-arm` requires a GitHub Team or Enterprise plan for hosted runners. For free accounts,
use a self-hosted arm64 runner registered with your repository.

---

## Related

- [CI/CD Integration](../cicd.md): general CI setup guide
- [In a Docker Image](../installing/in-docker.md): pre-bake your environment into a runner image
- [Provisioner CLI Reference](../reference/provisioner-commands.md): flags and env vars
- [Variables & Configuration](../blueprints/variables.md): `{{ arch }}` and other blueprint variables
