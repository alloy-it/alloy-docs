# Blueprint Syntax Reference

Complete reference for writing Alloy blueprint files.

---

## Directory structure

```text
blueprint/
├── manifest.yml       # required: entry point and metadata
├── 00-system.yml      # task files: named and ordered as you like
├── 10-toolchain.yml
└── 20-sdk.yml
```

---

## manifest.yml

```yaml
name: "My Environment" # required
version: "1.0" # required
description: "..." # required

variables: # optional: reusable values
  KEY: "value"
  ANOTHER: "value"

toolchains: # optional: catalog-sourced tool references
  - ref: "toolchain.arm-gnu.arm-none-eabi@stable"
    alias: "ARM_GCC" # injects ARM_GCC_URL and ARM_GCC_SHA as variables

supported_hosts: # optional: list of OS/arch this blueprint targets
  - linux/amd64
  - linux/arm64

run_order: # required: task files to run, in order
  - "00-system.yml"
  - "10-toolchain.yml"
```

### Variables

Variables are referenced in task files with `{{.Vars.NAME}}`:

```yaml
# manifest.yml
variables:
  GCC_VERSION: "13.3.rel1"
  TOOLCHAIN_DIR: "/opt/toolchains"
```

```yaml
# in a task file:
url: "https://example.com/gcc-{{.Vars.GCC_VERSION}}.tar.xz"
dest: "{{.Vars.TOOLCHAIN_DIR}}/arm-gcc"
```

### Toolchain aliases

When a `toolchains:` entry has an `alias`, the resolved URL and SHA256 are injected as variables `{ALIAS}_URL` and `{ALIAS}_SHA`:

```yaml
toolchains:
  - ref: "toolchain.arm-gnu.arm-none-eabi@stable"
    alias: "ARM_GCC"
```

This makes `{{.Vars.ARM_GCC_URL}}` and `{{.Vars.ARM_GCC_SHA}}` available in task files.

---

## Task file structure

Task files are **flat YAML lists**. Each item is a task object. There is no wrapping key.

```yaml
- name: "Task description" # required
  action: "action_name" # required
  # action-specific fields...
  creates: "/path/to/file" # optional: idempotency guard
  run_as: "username" # optional: run as this user (default: root)
  owner: "user:group" # optional: set ownership after action
  mode: "0755" # optional: set permissions after action
  always_run: true # optional: skip creates check, always run
  skip_if: "shell expr" # optional: skip if expression exits 0
  exclude: [docker, wsl2] # optional: skip on named environment tags
  per_arch: # optional: architecture-specific overrides
    amd64:
      url: "..."
      sha256: "..."
    arm64:
      url: "..."
      sha256: "..."
```

### `creates`

The idempotency guard. If the path exists, the task is skipped. Use it on every task that installs something:

```yaml
- name: "Install ARM GCC"
  action: "unarchive_from_url"
  # ...
  creates: "/opt/toolchains/arm-gcc/bin/arm-none-eabi-gcc"
```

### `always_run`

Bypasses the `creates` check. Useful for tasks that should run every time, like `apt-get update`:

```yaml
- name: "Update APT cache"
  action: "run_command"
  command: "apt-get update -y"
  always_run: true
```

### `skip_if`

Skip a task if a shell expression exits 0:

```yaml
- name: "Reload udev rules"
  action: "run_command"
  command: "udevadm control --reload-rules && udevadm trigger"
  always_run: true
  skip_if: '[ "$ALLOY_BACKEND" = "wsl2" ]'
```

### `exclude`

Skip a task on specific named environments. Cleaner and more readable than `skip_if` for well-known environments.

Built-in tags:

| Tag      | When active                                      |
| :------- | :----------------------------------------------- |
| `docker` | `--docker` flag or `ALLOY_ENV_DOCKER=1` env var  |
| `wsl2`   | `--wsl2` flag or `ALLOY_ENV_WSL2=1` env var      |

When running inside [alloy-host](../getting-started/with-alloy-host.md), `--wsl2` is automatically passed by the WSL2 backend.

```yaml
- name: "Install udev rules"
  action: "udev_rules"
  rule_file: "99-my-device.rules"
  exclude: [docker, wsl2]  # udev doesn't work inside Docker containers or WSL2

- name: "Register QEMU binfmt"
  action: "register_binfmt"
  architecture: "arm"
  interpreter: "/usr/bin/qemu-arm-static"
  exclude: [docker]  # binfmt registration requires privileged mode; skip in Docker
```

You can combine `exclude` with `skip_if`: both are checked independently and either condition can cause the task to be skipped.

### `run_as`

Run the task as a specific user instead of root:

```yaml
- name: "Install West"
  action: "run_command"
  run_as: "vagrant"
  command: "pip3 install --user west"
  creates: "/home/vagrant/.local/bin/west"
```

### `owner` and `mode`

Set file ownership and permissions after the action completes:

```yaml
- name: "Write udev rules"
  action: "get_file"
  url: "https://example.com/99-device.rules"
  sha256: "abc123..."
  dest: "/etc/udev/rules.d/99-device.rules"
  owner: "root:root"
  mode: "0644"
```

`mode` accepts octal strings (`"0755"`) or symbolic strings:

| Symbolic                      | Octal  |
| :---------------------------- | :----- |
| `USER_RWX GROUP_RX OTHER_RX`  | `0755` |
| `USER_RW GROUP_R OTHER_R`     | `0644` |
| `USER_RW GROUP_RW OTHER_NONE` | `0660` |
| `READ_ALL`                    | `0444` |
| `FULL_CONTROL`                | `0777` |

### `per_arch`

Override specific fields per host architecture. Supported keys: `url`, `sha256`, `command`, `source`. Supported architectures: `amd64`, `arm64`.

```yaml
- name: "Install toolchain"
  action: "unarchive_from_url"
  url: "https://example.com/toolchain-x86_64.tar.xz"
  sha256: "aaa111..."
  dest: "/opt/toolchains/arm-gcc"
  strip_components: 1
  creates: "/opt/toolchains/arm-gcc/bin/arm-none-eabi-gcc"
  per_arch:
    arm64:
      url: "https://example.com/toolchain-aarch64.tar.xz"
      sha256: "bbb222..."
```

---

## Actions

### `apt_install`

Install packages from the Debian/Ubuntu APT repository.

```yaml
- name: "Install build tools"
  action: "apt_install"
  packages:
    - "cmake"
    - "ninja-build"
    - "gdb-multiarch"
    - "python3"
    - "git"
```

---

### `run_command`

Run a shell command. **`reason` is required**: it must explain why the task cannot be expressed as one of the dedicated actions. This creates an audit trail and encourages replacing one-off commands with proper primitives over time.

| Field        | Required | Description                              |
| :----------- | :------- | :--------------------------------------- |
| `command`    | Yes      | Shell command to run                     |
| `reason`     | Yes      | Why no dedicated action covers this step |
| `run_as`     | No       | Run as this user (default: root)         |
| `always_run` | No       | Skip `creates` check and always run      |
| `skip_if`    | No       | Shell expression; skip if it exits 0     |
| `creates`    | No       | Skip if this path already exists         |

```yaml
- name: "Update APT cache"
  action: "run_command"
  reason: "apt-get update has no dedicated primitive; must run before any apt_install step."
  command: "apt-get update -y"
  always_run: true
```

Multi-line:

```yaml
- name: "Export Zephyr CMake packages"
  action: "run_command"
  reason: "west zephyr-export registers CMake package paths; no dedicated primitive covers CMake package registration."
  run_as: "vagrant"
  command: |
    export PATH=$HOME/.local/bin:$PATH
    cd /opt/ncs/v2.9.0 && west zephyr-export
  creates: "/opt/ncs/v2.9.0/.zephyr_exported"
```

---

### `get_file`

Download a single file to a path.

```yaml
- name: "Download udev rules"
  action: "get_file"
  url: "https://example.com/99-device.rules"
  sha256: "abc123..."
  dest: "/etc/udev/rules.d/99-device.rules"
  creates: "/etc/udev/rules.d/99-device.rules"
```

Supports `per_arch` for architecture-specific URLs.

---

### `unarchive_from_url`

Download and extract an archive (`.tar.gz`, `.tar.xz`, `.tar.bz2`, `.zip`).

```yaml
- name: "Install ARM GCC"
  action: "unarchive_from_url"
  url: "https://developer.arm.com/.../arm-gnu-toolchain-13.3.rel1-x86_64-arm-none-eabi.tar.xz"
  sha256: "6cd1bbc1d9ae57408d592354e4f59a82a2427e86a35cf2b97209c9b67c4b3ac6"
  dest: "/opt/toolchains/arm-gcc"
  strip_components: 1
  creates: "/opt/toolchains/arm-gcc/bin/arm-none-eabi-gcc"
  mode: "USER_RWX GROUP_RX OTHER_RX"
```

`strip_components` strips N leading directory levels from paths when extracting.

---

### `unarchive_from_ref`

Download and extract a versioned tool from the Alloy catalog.

```yaml
- name: "Install ARM GNU toolchain from catalog"
  action: "unarchive_from_ref"
  ref: "toolchain.arm-gnu.arm-none-eabi@stable"
  dest: "/opt/toolchains/arm-gcc"
  strip_components: 1
  creates: "/opt/toolchains/arm-gcc/bin/arm-none-eabi-gcc"
  mode: "USER_RWX GROUP_RX OTHER_RX"
```

Ref format: `<type>.<vendor>.<tool>@<version-or-alias>`. Run `alloy-host catalog info <ref>` to see available versions.

---

### `write_env_file`

Write arbitrary content to a file with variable substitution. Used for shell profile scripts, CMake toolchain files, config files.

```yaml
- name: "Add toolchain to PATH"
  action: "write_env_file"
  file: "/etc/profile.d/arm-gcc.sh"
  content: |
    #!/bin/sh
    export PATH=$PATH:{{.Vars.TOOLCHAIN_DIR}}/arm-gcc/bin
    export ARM_GCC_ROOT={{.Vars.TOOLCHAIN_DIR}}/arm-gcc
  mode: "USER_RWX GROUP_RX OTHER_RX"
```

```yaml
- name: "Create CMake toolchain file"
  action: "write_env_file"
  file: "/opt/cmake/arm-none-eabi.cmake"
  content: |
    set(CMAKE_SYSTEM_NAME Generic)
    set(CMAKE_C_COMPILER arm-none-eabi-gcc)
    set(CMAKE_CXX_COMPILER arm-none-eabi-g++)
  mode: "USER_RW GROUP_R OTHER_R"
```

---

### `unpack`

Extract a local archive already present inside the VM or at `/vagrant`.

```yaml
- name: "Unpack bundled tools"
  action: "unpack"
  source: "/vagrant/tools/my-tool.tar.gz"
  dest: "/opt/my-tool"
  creates: "/opt/my-tool/bin/tool"
```

---

### `install_target_lib`

Install a package (e.g. a `.deb`) into a sysroot directory. Used for cross-compilation setups that need target-architecture libraries.

```yaml
- name: "Install target library into sysroot"
  action: "install_target_lib"
  url: "https://example.com/libfoo-armhf.deb"
  sha256: "abc123..."
  sysroot: "/opt/sysroot"
  creates: "/opt/sysroot/usr/lib/libfoo.so"
```

---

### `python_package`

Install a Python package via `pip3`. Use this instead of `run_command: pip3 install ...`.

| Field     | Required | Description                               |
| :-------- | :------- | :---------------------------------------- |
| `package` | Yes      | PyPI package name                         |
| `version` | No       | Exact version to install (e.g. `"1.3.0"`) |

```yaml
- name: "Install West"
  action: "python_package"
  package: "west"
  version: "{{.Vars.WEST_VERSION}}"
  creates: "/home/{{.Vars.DEV_USERNAME}}/.local/bin/west"
```

---

### `rust_package`

Install a Rust crate via `cargo install`. `cargo` must already be on `PATH`.

| Field      | Required | Description                      |
| :--------- | :------- | :------------------------------- |
| `package`  | Yes      | Crate name on crates.io          |
| `version`  | Yes      | Exact version to install         |
| `features` | No       | List of Cargo features to enable |

```yaml
- name: "Install cargo-binutils"
  action: "rust_package"
  package: "cargo-binutils"
  version: "0.3.6"
```

---

### `mcu_sdk`

Install a vendor MCU SDK using its native tooling. The install tool (e.g. `west`) must already be on `PATH`; install it first with `python_package`.

| Field     | Required | Description                      |
| :-------- | :------- | :------------------------------- |
| `method`  | Yes      | Install method: `west` or `git`  |
| `url`     | Yes      | Repository URL                   |
| `version` | Yes      | Tag or branch to check out       |
| `dest`    | Yes      | Installation directory           |
| `run_as`  | No       | Run as this user (default: root) |
| `creates` | No       | Skip if this path already exists |

```yaml
- name: "Initialize nRF Connect SDK"
  action: "mcu_sdk"
  method: "west"
  url: "{{.Vars.NCS_URL}}"
  version: "{{.Vars.NCS_VERSION}}"
  dest: "{{.Vars.NCS_BASE}}/{{.Vars.NCS_VERSION}}"
  run_as: "{{.Vars.DEV_USERNAME}}"
  creates: "{{.Vars.NCS_BASE}}/{{.Vars.NCS_VERSION}}/nrf"
```

---

### `cmake_toolchain`

Generate a CMake toolchain file from declared cross-compilation parameters. Prefer this over `write_env_file` for `.cmake` files; the provisioner validates all required fields.

| Field              | Required | Description                                      |
| :----------------- | :------- | :----------------------------------------------- |
| `file`             | Yes      | Output path for the `.cmake` file                |
| `target_system`    | Yes      | `CMAKE_SYSTEM_NAME` (e.g. `Generic`, `Linux`)    |
| `target_processor` | Yes      | `CMAKE_SYSTEM_PROCESSOR` (e.g. `arm`, `aarch64`) |
| `compiler_prefix`  | Yes      | Toolchain prefix (e.g. `arm-none-eabi`)          |
| `compiler_path`    | Yes      | Directory containing the compiler binaries       |
| `sysroot`          | No       | `CMAKE_SYSROOT` path                             |

```yaml
- name: "Generate ARM CMake toolchain"
  action: "cmake_toolchain"
  file: "/opt/cmake/arm-none-eabi.cmake"
  target_system: "Generic"
  target_processor: "arm"
  compiler_prefix: "arm-none-eabi"
  compiler_path: "{{.Vars.ARM_TOOLCHAIN_DEST}}/bin"
```

---

### `install_deb_to_sysroot`

Install a `.deb` package into a cross-compilation sysroot (not the host system). The formalized version of `install_target_lib` that takes `sysroot` explicitly.

| Field     | Required | Description                                  |
| :-------- | :------- | :------------------------------------------- |
| `url`     | Yes      | URL of the `.deb` file (supports `per_arch`) |
| `sha256`  | Yes      | SHA256 of the `.deb` file                    |
| `sysroot` | Yes      | Sysroot directory to install into            |

```yaml
- name: "Install libfoo for ARM"
  action: "install_deb_to_sysroot"
  url: "https://apt.example.com/libfoo_1.0_armhf.deb"
  sha256: "abc123..."
  sysroot: "{{.Vars.SDK_SYSROOT}}"
```

---

### `register_binfmt`

Register a QEMU interpreter for transparent cross-architecture execution via `binfmt_misc`. Use this instead of `run_command: update-binfmts ...`.

| Field          | Required | Description                                            |
| :------------- | :------- | :----------------------------------------------------- |
| `architecture` | Yes      | Target arch: `arm`, `aarch64`, `riscv64`, etc.         |
| `interpreter`  | Yes      | Path to the QEMU static binary                         |
| `package`      | No       | APT package to install first (e.g. `qemu-user-static`) |

```yaml
- name: "Register ARM binfmt for cross-build"
  action: "register_binfmt"
  architecture: "arm"
  interpreter: "/usr/bin/qemu-arm-static"
  package: "qemu-user-static"
```

---

### `sdk_install`

Run a self-extracting SDK installer script (Yocto eSDK / Poky pattern). Downloads, verifies SHA256, and executes the installer.

| Field    | Required | Description                    |
| :------- | :------- | :----------------------------- |
| `url`    | Yes      | URL of the installer script    |
| `sha256` | Yes      | SHA256 of the installer script |
| `dest`   | Yes      | Installation directory         |

```yaml
- name: "Install Yocto eSDK"
  action: "sdk_install"
  url: "https://example.com/poky-toolchain-4.0.sh"
  sha256: "abc123..."
  dest: "/opt/poky/4.0"
  creates: "/opt/poky/4.0/environment-setup-cortexa8hf-neon-poky-linux-gnueabi"
```

---

### `build_artifact`

Download, verify, and build a source tarball using `./configure` + `make`. Use this instead of chaining multiple `run_command` tasks for source builds.

| Field             | Required | Description                                            |
| :---------------- | :------- | :----------------------------------------------------- |
| `artifact_source` | Yes      | Nested block: `url` and `sha256` of the source tarball |
| `make_targets`    | Yes      | List of `make` targets (e.g. `["all", "install"]`)     |
| `dependencies`    | No       | APT packages to install before building                |
| `configure_flags` | No       | Arguments for `./configure`                            |
| `destination`     | No       | Install prefix                                         |

```yaml
- name: "Build libusb from source"
  action: "build_artifact"
  artifact_source:
    url: "https://github.com/libusb/libusb/releases/download/v1.0.27/libusb-1.0.27.tar.bz2"
    sha256: "abc123..."
  dependencies:
    - "libudev-dev"
  configure_flags:
    - "--prefix=/usr"
    - "--disable-static"
  make_targets:
    - "all"
    - "install"
```

---

### `udev_rules`

Download a udev rules file and install it into `/etc/udev/rules.d/`, then reload udev. Use this instead of `write_env_file` for rules sourced from a URL.

| Field       | Required | Description                             |
| :---------- | :------- | :-------------------------------------- |
| `url`       | Yes      | URL of the rules file                   |
| `sha256`    | Yes      | SHA256 of the rules file                |
| `rule_file` | Yes      | Filename to use in `/etc/udev/rules.d/` |

```yaml
- name: "Install Nordic Semiconductor udev rules"
  action: "udev_rules"
  url: "https://raw.githubusercontent.com/NordicSemiconductor/nrf-udev/main/nrf-udev_1.0.1-all.deb"
  sha256: "abc123..."
  rule_file: "99-nordic.rules"
```

---

### `yocto_manifest`

Check out a Yocto layer configuration using `kas`. Install `kas` first with `python_package`.

| Field         | Required | Description                                 |
| :------------ | :------- | :------------------------------------------ |
| `url`         | Yes      | URL of the kas config repository            |
| `dest`        | Yes      | Directory to check out into                 |
| `commit_hash` | No       | Git commit SHA to pin to                    |
| `kas_file`    | No       | Path to the kas config YAML inside the repo |

```yaml
- name: "Configure Yocto layers"
  action: "yocto_manifest"
  url: "https://github.com/myorg/my-kas-config.git"
  dest: "/opt/yocto/build"
  commit_hash: "abc123def456..."
  kas_file: "kas/my-machine.yml"
```

---

### `buildroot_config`

Clone Buildroot at a specific release and apply a defconfig. `git` must be on `PATH`.

| Field           | Required | Description                                    |
| :-------------- | :------- | :--------------------------------------------- |
| `url`           | Yes      | Buildroot git repository URL                   |
| `dest`          | Yes      | Directory to clone into                        |
| `version`       | No       | Tag or branch (e.g. `"2024.02"`)               |
| `defconfig`     | No       | Defconfig name to apply                        |
| `external_tree` | No       | URL of a Buildroot external tree to also clone |

```yaml
- name: "Set up Buildroot"
  action: "buildroot_config"
  url: "https://git.buildroot.net/buildroot"
  dest: "/opt/buildroot"
  version: "2024.02"
  defconfig: "raspberrypi4_defconfig"
```

---

### `pkg_config_env`

Generate a shell environment script that configures `pkg-config` to search a cross-compilation sysroot. Use after `install_deb_to_sysroot` or `sdk_install`.

| Field      | Required | Description                                         |
| :--------- | :------- | :-------------------------------------------------- |
| `sysroot`  | Yes      | Cross-compilation sysroot directory                 |
| `env_file` | Yes      | Output path (e.g. `/etc/profile.d/50-pkgconfig.sh`) |

```yaml
- name: "Configure pkg-config for ARM sysroot"
  action: "pkg_config_env"
  sysroot: "{{.Vars.SDK_SYSROOT}}"
  env_file: "/etc/profile.d/50-pkgconfig.sh"
```

---

## Lockfile (alloy.lock.yml)

Generated by `alloy-host resolve`. Records the exact download URLs and SHA256 checksums for every catalog-sourced tool, resolved for your host architecture. Commit this file.

```yaml
# Generated by alloy-host resolve. Commit this file.
resolved_at: "2026-01-15T10:30:00Z"
host: "linux/amd64"
toolchains:
  toolchain.arm-gnu.arm-none-eabi:
    url: "https://developer.arm.com/downloads/..."
    sha256: "6cd1bbc1d9ae57408d592354e4f59a82a2427e86a35cf2b97209c9b67c4b3ac6"
    method: "archive"
```

Regenerate after changing `toolchains:` entries: `alloy-host resolve`
