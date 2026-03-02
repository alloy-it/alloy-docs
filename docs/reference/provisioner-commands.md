# alloy-provisioner Command Reference

**alloy-provisioner** runs inside a Linux system (bare metal, VM, WSL2, or Docker) and applies a blueprint to install packages, download toolchains, and configure the environment. This page documents the CLI; for the full mental model see [How Alloy works](../concepts.md). The [alloy-host](commands.md) reference documents the host-side tool that manages VMs and runs the provisioner inside them.

---

## Subcommands

### `alloy-provisioner install <name[:tag]>`

Pull a blueprint from the registry and run it immediately. This is the recommended way to provision from [Alloy Hub](https://alloy-it.io).

```bash
sudo alloy-provisioner install community/arm-none-eabi
sudo alloy-provisioner install community/nrf91:1.0.13
sudo alloy-provisioner install myproject/my-blueprint --registry my-registry.example.com
```

### `alloy-provisioner clone <name[:tag]>`

Pull blueprint files from the registry without running them. Files are written to the blueprint directory (default `~/.alloy-it/`). Run the provisioner afterward with `alloy-provisioner` (no subcommand) to apply the blueprint.

```bash
alloy-provisioner clone community/raspberry-pi
# Inspect the blueprint, then:
sudo alloy-provisioner
```

### Running from a local blueprint (no subcommand)

If the blueprint is already on disk (e.g. after `clone`, or a local project):

```bash
sudo alloy-provisioner --blueprint-dir /path/to/blueprint
```

Or set the directory via environment variable:

```bash
export ALLOY_BLUEPRINT_DIR=/path/to/blueprint
sudo -E alloy-provisioner
```

---

## Flags

| Flag             | Description                                                                 | Default           |
| ---------------- | --------------------------------------------------------------------------- | ----------------- |
| `-blueprint-dir`| Path to the blueprint directory. Overrides `ALLOY_BLUEPRINT_DIR`.           | `$HOME/.alloy-it` |
| `-registry`     | Registry base URL for pulling blueprints. Overrides `ALLOY_REGISTRY`.       | `api.alloy-it.io` |
| `-tag`          | Blueprint tag/version (with `-pull`).                                      | `latest`          |
| `-platform`     | Force OS/architecture (e.g. `linux/arm64`).                                | Auto-detect       |
| `-env-file`     | Path to an env file for variables and credentials (see [Variables](../blueprints/variables.md)). | —        |
| `-version`      | Print version and exit.                                                     | —                 |

Legacy: `-pull`, `-repository` are still supported; prefer the `install` and `clone` subcommands.

---

## Environment variables

| Variable                  | Purpose                                                |
| ------------------------- | ------------------------------------------------------ |
| `ALLOY_BLUEPRINT_DIR`     | Blueprint directory path (same as `-blueprint-dir`).   |
| `ALLOY_REGISTRY`          | Registry URL for pulling blueprints.                   |
| `ALLOY_REGISTRY_USERNAME` | Username for private registry authentication.         |
| `ALLOY_REGISTRY_PASSWORD` | Password or token for private registry authentication. |

Custom variables (e.g. for blueprint `required_env:`) can be set in an env file and passed with `-env-file`, or exported before running the provisioner.

---

## Next steps

- [Installing environments](../installing/index.md) — native, alloy-host, Docker
- [Blueprint variables and env files](../blueprints/variables.md)
- [alloy-host command reference](commands.md)
