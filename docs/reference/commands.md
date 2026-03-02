# Command Reference

Complete reference for all `alloy-host` commands.

Most commands accept an optional environment name. If omitted, Alloy uses the environment in the current directory, or prompts you to select if multiple are registered.

**Architecture:** `alloy-host` runs on your machine (the host). It creates and manages VMs (or WSL2 instances) and copies the blueprint into each environment. The actual provisioning (installing packages, downloading toolchains, writing env files) is performed **inside the guest** by **alloy-provisioner**, which alloy-host triggers via the guest's bootstrap script. See [How Alloy works](../concepts.md) for the full model.

---

## Environment lifecycle

### `alloy-host init`

Create a new environment. The environment directory is placed at `~/.alloy-it/envs/<name>/` by default.

```text
alloy-host init <name> --blueprint <blueprint-name> [flags]
```

`--blueprint` is required.

| Flag                | Description                                                                    |
| :------------------ | :----------------------------------------------------------------------------- |
| `--blueprint`, `-b` | Blueprint name to use (required)                                               |
| `--backend`         | Backend type: `vm` (default) or `wsl2`                                         |
| `--wsl-user`        | Linux username inside the WSL2 distro (WSL2 only, default: `vagrant`)          |
| `--path`            | Override base directory for the environment (default: `~/.alloy-it/envs/`)     |

**Examples:**

```bash
alloy-host init arm-dev --blueprint arm-gcc
alloy-host init nrf91-dev --blueprint nrf91
alloy-host init arm-dev --blueprint arm-gcc --backend wsl2
alloy-host init arm-dev --blueprint arm-gcc --path ~/projects/embedded
```

Init registers the environment but does not start the VM.

---

### `alloy-host up`

Start the VM. Creates and provisions it if it doesn't exist yet; resumes it if it's stopped.

```text
alloy-host up [name]
```

**Examples:**

```bash
cd arm-dev && alloy-host up
alloy-host up arm-dev
```

---

### `alloy-host stop`

Suspend the VM. Data and state are preserved. Resume with `up`.

```text
alloy-host stop [name]
```

---

### `alloy-host destroy`

Delete the VM. The environment directory and its files are not affected; recreate the VM any time with `alloy-host up`.

```text
alloy-host destroy [name] [flags]
```

| Flag          | Description                                          |
| :------------ | :--------------------------------------------------- |
| `--all`, `-a` | Also delete the persistent data disk. **Permanent.** |

---

### `alloy-host provision`

Re-run the blueprint on an already-running VM. Tasks that previously completed are skipped. Use this after modifying the blueprint.

```text
alloy-host provision [name]
```

---

### `alloy-host list`

Show all registered environments.

```text
alloy-host list
```

---

## Working with the VM

### `alloy-host ssh`

Open an interactive shell inside the VM. Your host project directory is available at `/vagrant`.

```text
alloy-host ssh [name]
```

---

## Blueprint management

### `alloy-host validate`

Check blueprint YAML files for syntax errors and missing required fields, without touching the VM.

```text
alloy-host validate [name|path]
```

```bash
alloy-host validate
alloy-host validate arm-dev
alloy-host validate /path/to/blueprint-dir
```

---

### `alloy-host resolve`

Read `toolchains:` references from `manifest.yml`, resolve them against the local catalog, and write `alloy.lock.yml` with the exact URLs and checksums for your host architecture.

```text
alloy-host resolve [name]
```

Run this after adding or changing `toolchains:` entries in your manifest. Commit the resulting `alloy.lock.yml`.

```bash
cd arm-dev && alloy-host resolve
```

---

## USB device management

### `alloy-host usb list`

List USB devices connected to the host. Pass a VM name to see which are already attached to it.

```text
alloy-host usb list [name]
```

---

### `alloy-host usb attach`

Attach a USB device to a VM. Identify the device by number (from `usb list`), partial name, or UUID/BUSID.

```text
alloy-host usb attach [device] [name]
```

**Examples:**

```bash
alloy-host usb attach                  # interactive: pick VM then device
alloy-host usb attach 1                # device 1 to current directory's VM
alloy-host usb attach 1 arm-dev        # device 1 to arm-dev
alloy-host usb attach "J-Link" arm-dev # by partial name
```

---

### `alloy-host usb detach`

Release a USB device back to the host.

```text
alloy-host usb detach <device> [name]
```

```bash
alloy-host usb detach 1
alloy-host usb detach 1 arm-dev
alloy-host usb detach "J-Link" arm-dev
```

---

### `alloy-host usb status`

Show which USB devices are currently attached to any VM or WSL2.

```text
alloy-host usb status
```

---

## Catalog

### `alloy-host catalog update`

Clone or pull the latest catalog into `~/.alloy-it/catalog/`.

```text
alloy-host catalog update [flags]
```

| Flag       | Description                                    |
| :--------- | :--------------------------------------------- |
| `--remote` | Remote Git URL (default: public Alloy catalog) |

---

### `alloy-host catalog search`

Search the local catalog by ID, name, or tag.

```text
alloy-host catalog search <query>
```

```bash
alloy-host catalog search arm
alloy-host catalog search nordic
alloy-host catalog search golang
```

---

### `alloy-host catalog info`

Show details about a catalog entry, including all available versions.

```text
alloy-host catalog info <ref>
```

```bash
alloy-host catalog info toolchain.arm-gnu.arm-none-eabi
alloy-host catalog info toolchain.arm-gnu.arm-none-eabi@13.3.rel1
```

---

### `alloy-host catalog add`

Validate a version descriptor YAML file and update the tool's `index.yaml`. Used when contributing to the catalog.

```text
alloy-host catalog add <version-file.yaml>
```

---

## Configuration

### `alloy-host config show`

Show current configuration: config file path, Vagrant path, VBoxManage path.

```text
alloy-host config show
```

---

### `alloy-host config set vagrant-path`

Set a custom path to the Vagrant executable.

```text
alloy-host config set vagrant-path <path>
```

---

### `alloy-host config set vbox-manage-path`

Set a custom path to the VBoxManage executable.

```text
alloy-host config set vbox-manage-path <path>
```

---

### `alloy-host config unset vagrant-path`

Remove the custom Vagrant path and revert to auto-discovery.

```text
alloy-host config unset vagrant-path
```

---

### `alloy-host config unset vbox-manage-path`

Remove the custom VBoxManage path and revert to auto-discovery.

```text
alloy-host config unset vbox-manage-path
```

---

## System

### `alloy-host check-health`

Check that Vagrant and VirtualBox (VBoxManage) are installed and report their versions.

```text
alloy-host check-health
```
