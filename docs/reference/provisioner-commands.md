# alloy-provisioner Command Reference

**alloy-provisioner** runs inside a Linux system (PC, laptop, or server, VM, WSL2, or Docker) and applies a blueprint to install packages, download toolchains, and configure the environment. This page documents the CLI; for the full mental model see [How Alloy works](../concepts.md). The [alloy-host](commands.md) reference documents the host-side tool that manages VMs and runs the provisioner inside them.

---

## Subcommands

### `alloy-provisioner install [name[:tag]]`

Install supports two modes:

- `install <name[:tag]>`: pull a blueprint from the registry and run it immediately (recommended for [Alloy Hub](https://alloy-it.io)).
- `install` with no name (and `--blueprint-dir`): run a local blueprint directory without pulling from the registry.

```bash
sudo alloy-provisioner install community/nordic/nrf91:1.1.3
sudo alloy-provisioner install community/nrf91:1.0.13
sudo alloy-provisioner install myproject/my-blueprint:1.0.0 --registry https://api.alloy-it.io

# Local blueprint (already cloned or in a repo)
sudo alloy-provisioner install --blueprint-dir /path/to/blueprint
```

Use named install when your blueprint is hosted in Alloy Hub/registry.
Use `install --blueprint-dir ...` when blueprint files are already on disk.

### `alloy-provisioner clone <name[:tag]>`

Pull blueprint files from the registry without running them. Files are written to:
`<blueprint-dir>/blueprints/<name>` (default base: `~/.alloy-it/`).

```bash
alloy-provisioner clone community/raspberry-pi:1.0.0
# Inspect the blueprint, then:
sudo alloy-provisioner install --blueprint-dir ~/.alloy-it/blueprints/community/raspberry-pi
```

#### `--list <project>`: browse available blueprints

List all blueprints available in a project without cloning anything. Community blueprints do not require authentication.

```bash
alloy-provisioner clone --list community
```

Output:

```text
BLUEPRINT                                           TAGS
community/nordic/nrf91                              1.1.3
community/nordic/nrf52                              1.1.3
community/raspberry-pi/raspberry-pi-5               1.0.3
community/raspberry-pi/raspberry-pi-4-model-b       1.0.2
community/espressif/esp32-idf                       1.0.0
...
```

Use the blueprint name and tag from the list as the argument to `clone` or `install`:

```bash
alloy-provisioner clone community/nordic/nrf91:1.1.3 --output ./my-blueprint
sudo alloy-provisioner install community/nordic/nrf91:1.1.3
```

### Running from a local blueprint

If the blueprint is already on disk (e.g. after `clone`, or a local project):

```bash
sudo alloy-provisioner install --blueprint-dir /path/to/blueprint
```

Use this when blueprint files are already present locally (for example in a repo
folder or after `clone`).

For Docker builds where you want tasks with `exclude: [docker]` to be skipped:

```bash
sudo alloy-provisioner install --docker --blueprint-dir /path/to/blueprint
```

Or set the directory via environment variable:

```bash
export ALLOY_BLUEPRINT_DIR=/path/to/blueprint
sudo -E alloy-provisioner install --blueprint-dir "$ALLOY_BLUEPRINT_DIR"
```

---

## Flags

| Flag             | Description                                                                 | Default           |
| ---------------- | --------------------------------------------------------------------------- | ----------------- |
| `--blueprint-dir` | Path to the blueprint directory. Overrides `ALLOY_BLUEPRINT_DIR`.             | `$HOME/.alloy-it` |
| `--registry`      | Registry base URL for pulling blueprints. Overrides `ALLOY_REGISTRY`.         | `api.alloy-it.io` |
| `--tag`           | Blueprint tag/version for `install`/`clone` when not provided as `name:tag`.  | none (must be specified) |
| `--platform`      | Force OS/architecture (e.g. `linux/arm64`).                                   | Auto-detect       |
| `--env-file`      | Path to an env file for variables and credentials (see [Variables](../blueprints/variables.md)). | – |
| `--version`       | Print version and exit.                                                        | –                 |

Legacy: `-pull`, `-repository` are still supported; prefer the `install` and `clone` subcommands.

---

## Runtime environment tags

Use these flags when you want blueprint `exclude` filters to be evaluated for a
specific runtime context:

| Flag        | Activates tag | Typical usage |
| ----------- | ------------- | ------------- |
| `--docker`  | `docker`      | Docker image builds and containerized provisioning |
| `--wsl2`    | `wsl2`        | WSL2 guest provisioning |

Example (`docker` tag active):

```bash
alloy-provisioner install --docker community/nrf91:1.0.13
```

---

## Environment variables

| Variable                  | Purpose                                                |
| ------------------------- | ------------------------------------------------------ |
| `ALLOY_BLUEPRINT_DIR`     | Blueprint directory path (same as `-blueprint-dir`).   |
| `ALLOY_REGISTRY`          | Registry URL for pulling blueprints.                   |
| `ALLOY_REGISTRY_USERNAME` | Username for private registry authentication.         |
| `ALLOY_REGISTRY_PASSWORD` | Password or token for private registry authentication. |
| `ALLOY_ENV_DOCKER`        | If set to `1`, activates the `docker` environment tag. |
| `ALLOY_ENV_WSL2`          | If set to `1`, activates the `wsl2` environment tag.   |

Custom variables (e.g. for blueprint `required_env:`) can be set in an env file and passed with `-env-file`, or exported before running the provisioner.

---

## Next steps

- [Installing environments](../installing/index.md): native, alloy-host, Docker
- [Blueprint variables and env files](../blueprints/variables.md)
- [alloy-host command reference](commands.md)
