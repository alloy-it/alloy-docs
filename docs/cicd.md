# CI/CD Integration

Use the same blueprint that developers use locally to provision build environments in CI. **alloy-provisioner** runs on any Linux runner (bare metal, VM, or Docker) and installs the exact toolchain and dependencies from your blueprint.

---

## Options

| Approach | When to use | Doc |
| -------- | ----------- | --- |
| **Native on Linux runner** | Runner is already Linux (e.g. GitHub-hosted `ubuntu-latest`, self-hosted Linux agent). Install alloy-provisioner, then run `alloy-provisioner install <blueprint>`. | [On Linux (Native)](installing/native.md) |
| **Docker image** | You want a pre-baked image with the environment, or your pipeline uses Docker. Build an image that runs alloy-provisioner during the Docker build. | [In a Docker Image](installing/in-docker.md) |

Both use the same blueprint and lockfile, so CI and local dev stay in sync.

---

## Quick example (GitHub Actions, native)

Runner is Ubuntu; install the provisioner, then install the blueprint and build:

```yaml
jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Install alloy-provisioner
        run: |
          curl -fsSL https://raw.githubusercontent.com/alloy-it/alloy-provisioner-releases/main/scripts/install.sh | bash

      - name: Provision environment
        run: sudo alloy-provisioner install community/arm-none-eabi

      - name: Build
        run: |
          source /etc/alloy/env   # or the path your blueprint writes to
          make build
```

Pin the blueprint version for reproducibility:

```bash
sudo alloy-provisioner install community/arm-none-eabi:1.2.0
```

---

## Private blueprints and secrets

For organization or private blueprints, configure registry auth using secrets (e.g. GitHub Actions secrets, GitLab CI variables):

```bash
export ALLOY_REGISTRY_USERNAME="$REGISTRY_USER"
export ALLOY_REGISTRY_PASSWORD="$REGISTRY_TOKEN"
sudo -E alloy-provisioner install myorg/my-blueprint
```

Never commit credentials; use your CI system’s secret storage.

---

## More

- [Installing on Linux (Native)](installing/native.md) — install steps, env file, re-provisioning
- [In a Docker Image](installing/in-docker.md) — Dockerfile pattern for CI base images
- [Provisioner CLI](reference/provisioner-commands.md) — flags and environment variables
