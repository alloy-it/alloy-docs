# Troubleshooting

Solutions to the most common problems when using **alloy-provisioner**: installing the binary, pulling blueprints from the registry, and running provisioning. If you use **alloy-host** to manage VMs, see [alloy-host and VM issues](#alloy-host-and-vm-issues) at the end for where to look.

---

## Installing alloy-provisioner

### `alloy-provisioner: command not found`

The binary is not in your `PATH`. The `.deb` package installs to `/usr/local/bin/`. Check:

```bash
which alloy-provisioner
ls -la /usr/local/bin/alloy-provisioner
```

If you installed from a tar.gz, ensure the directory containing the binary is in your `PATH`, or move it:

```bash
sudo mv alloy-provisioner /usr/local/bin/
```

---

### Download fails or wrong architecture

- **404 or download error:** Confirm the [releases](https://github.com/alloy-it/alloy-provisioner-releases/releases) page and use the correct URL. For "latest" use `releases/download/latest/alloy-provisioner_latest_linux_amd64.deb` (or `arm64`). For a specific version use the versioned asset name.
- **Wrong .deb for this machine:** On Debian/Ubuntu, use `dpkg --print-architecture` (e.g. `amd64` or `arm64`) and pick the matching `.deb`. See [On Linux (Native)](../installing/native.md) for copy-paste commands.

---

### `apt-get install` fails on the .deb (dependency errors)

Install dependencies first, then the .deb:

```bash
sudo apt-get update
sudo apt-get install -y wget
# then install the .deb as in the native install guide
```

If `dpkg -i` reports missing dependencies, run `sudo apt-get install -f` to fix, or install the provisioner via the steps in [Installing on Linux (Native)](../installing/native.md) which use `apt-get install -y /tmp/alloy-provisioner.deb` so apt resolves dependencies.

---

## Pulling blueprints (install / clone)

### Registry unreachable or timeout

The provisioner pulls blueprints from the registry (default: `api.alloy-it.io`). If `alloy-provisioner install <blueprint>` or `alloy-provisioner clone <blueprint>` fails with a connection or timeout error:

1. **Network:** Ensure the machine can reach the registry (no firewall or proxy blocking HTTPS).
2. **Custom registry:** If you use a private registry, set `--registry` or `ALLOY_REGISTRY`:
   ```bash
   sudo alloy-provisioner install myorg/my-blueprint --registry https://my-registry.example.com
   ```

---

### Authentication failed (private or org blueprints)

For private or organization blueprints, the registry may require credentials. Set them before running:

```bash
export ALLOY_REGISTRY_USERNAME="your-username"
export ALLOY_REGISTRY_PASSWORD="your-token-or-password"
sudo -E alloy-provisioner install myorg/my-blueprint
```

Or use an env file (see [Blueprint variables](../blueprints/variables.md)) and pass it with `-env-file`. Never commit credentials to the blueprint or to git.

---

### Blueprint not found (404 or "unknown repository")

- Check the blueprint name and tag (e.g. `community/arm-none-eabi`, `community/raspberry-pi-arm64:latest`). Browse [Alloy Hub](https://alloy-it.io) to confirm the exact name and available tags.
- For a custom registry, ensure the repository path and tag exist and that your credentials have access.

---

## Provisioning fails (during or after pull)

### Required environment variables missing

If the blueprint declares `required_env:` in `manifest.yml`, the provisioner exits with a clear error when a variable is missing. Set the variables in your environment or in an env file and pass them when running:

```bash
export MY_TOKEN="secret"
sudo -E alloy-provisioner --blueprint-dir /path/to/blueprint
# or
sudo alloy-provisioner --blueprint-dir /path/to/blueprint -env-file ~/.alloy-env
```

See [Variables & Configuration](../blueprints/variables.md).

---

### SHA256 checksum mismatch

All downloads in blueprints (e.g. `get_file`, `unarchive_from_url`, `install_target_lib`) require a `sha256` field. If the downloaded file does not match, the provisioner fails. This usually means:

- The upstream URL changed (new version, different file). Update the blueprint with the new URL and new `sha256`.
- For toolchain refs resolved in `alloy.lock.yml`, run `alloy-host resolve` again (if using alloy-host) to refresh the lockfile, or update the catalog/blueprint that provides the ref.

---

### Out of disk space or "no space left on device"

The first run can download and install several GB of tools. Ensure the system (or guest VM) has enough free space (e.g. at least 20 GB). Free space and retry; the provisioner is idempotent and will resume where it left off.

---

### Permission denied or "cannot write"

Most provisioning tasks need root (e.g. installing packages, writing to `/opt`, `/etc`). Run with `sudo`:

```bash
sudo alloy-provisioner install community/arm-none-eabi
sudo alloy-provisioner --blueprint-dir /path/to/blueprint
```

Tasks that use `run_as: someuser` still require the process to start as root so the provisioner can switch user where needed.

---

### Task fails (apt, download, or run_command)

- **apt_install fails:** Check that the machine has internet access and that `apt-get update` can run. If you use a proxy, set the usual `http_proxy` / `https_proxy` (and optionally `apt` proxy config) before running the provisioner.
- **Download or run_command fails:** Inspect the error message; it often points to a missing dependency, a wrong path, or a failed script. Fix the blueprint task (URL, command, or `creates:` path) and re-run. Remove `alloy.state.yml` in the blueprint directory to force that task to run again, or run the provisioner again (it will retry failed work where applicable).

---

## Local blueprint (--blueprint-dir)

### "No such file or directory" or blueprint not found

- Ensure the path passed to `--blueprint-dir` exists and contains `manifest.yml` and the task files listed in `run_order`.
- You can set the directory via environment: `export ALLOY_BLUEPRINT_DIR=/path/to/blueprint`, then run `alloy-provisioner` (no subcommand).

---

### Invalid manifest or "run_order" errors

The provisioner expects a valid `manifest.yml`: `name`, `version`, and `run_order` (list of task filenames). Each file in `run_order` must exist in the blueprint directory. Fix the manifest and ensure all referenced task files are present. See [Blueprint Structure](../blueprints/structure.md) and [Blueprint Syntax](../reference/blueprint-syntax.md).

---

## Re-provisioning and state

- Re-running the provisioner is safe: it skips completed tasks (tracked in `alloy.state.yml`). To force a full re-run, remove the state file and run again:
  ```bash
  rm /path/to/blueprint/alloy.state.yml
  sudo alloy-provisioner --blueprint-dir /path/to/blueprint
  ```
- For blueprint from registry: `sudo alloy-provisioner install community/arm-none-eabi` again will re-use the pulled blueprint and only re-run tasks that need to run.

---

## Getting more help

- **Provisioner version:** `alloy-provisioner -version`
- **Installing the provisioner:** [On Linux (Native)](../installing/native.md)
- **Subcommands and flags:** [Provisioner CLI](../reference/provisioner-commands.md)
- **Blueprint authoring:** [Developing Blueprints](../blueprints/index.md)

---

## alloy-host and VM issues

If you use **alloy-host** to create and manage VMs (and the provisioner runs inside the guest), VM lifecycle, SSH, and USB passthrough are handled by alloy-host, not by the provisioner. For issues such as:

- `alloy-host: command not found`, `check-health` failing (Vagrant/VirtualBox not found)
- VM won't start, Vagrant timeout, "directory already exists"
- SSH to the VM, USB device attachment
- macOS Apple Silicon (M1/M2/M3) and VirtualBox

see the [With Alloy Host](../installing/with-alloy-host.md) installation guide and the alloy-host documentation. Provisioning failures *inside* the VM are the same as above (registry, auth, disk, checksum, etc.); ensure the guest has network access and enough disk space.
