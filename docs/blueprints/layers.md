# The Layers Approach

## Why layers matter

A typical embedded build environment involves several independent concerns: base OS packages, a cross-compilation toolchain, SDK setup, environment variables, and project-specific tools. Without structure, these all end up in one sprawling script or a single task file that becomes impossible to maintain.

The **layers approach** splits a blueprint into numbered files, each responsible for one logical layer of the environment. This is the pattern followed by all blueprints published in the alloy-catalog, and it is strongly recommended for any blueprint you write.

---

## Recommended layer structure

```
my-blueprint/
├── manifest.yml
├── 00-system.yml       # base OS packages
├── 10-toolchain.yml    # cross-compiler / SDK download and install
├── 20-environment.yml  # env vars, profile scripts, CMake config
├── 30-project.yml      # project-specific tools, udev rules (optional)
└── alloy.lock.yml
```

Each file is independent, idempotent, and focused on a single concern.

---

## Layer 0: System base (`00-system.yml`)

Install the base OS packages and system-level configuration. These are the foundation everything else depends on.

```yaml
- name: Update APT cache
  action: apt_install
  packages: []
  always_run: true

- name: Install build essentials
  action: apt_install
  packages:
    - git
    - cmake
    - ninja-build
    - python3
    - python3-pip
    - wget
    - curl
    - ca-certificates
    - rsync
    - tar
    - xz-utils

- name: Install debugging tools
  action: apt_install
  packages:
    - gdb-multiarch
    - openocd
```

Keep this layer minimal. Avoid putting toolchain-specific packages here; put them in the toolchain layer.

---

## Layer 1: Toolchain (`10-toolchain.yml`)

Download and install the cross-compilation toolchain or SDK. This is usually the heaviest layer, as it downloads large archives.

```yaml
- name: Install ARM GCC toolchain
  action: unarchive_from_url
  dest: "{{.Vars.ARM_TOOLCHAIN_DEST}}"
  strip_components: 1
  creates: "{{.Vars.ARM_TOOLCHAIN_DEST}}/bin/arm-none-eabi-gcc"
  per_arch:
    amd64:
      url: https://developer.arm.com/-/media/Files/downloads/gnu/13.3.rel1/binrel/arm-gnu-toolchain-13.3.rel1-x86_64-arm-none-eabi.tar.xz
      sha256: "aaabbbccc..."
    arm64:
      url: https://developer.arm.com/-/media/Files/downloads/gnu/13.3.rel1/binrel/arm-gnu-toolchain-13.3.rel1-aarch64-arm-none-eabi.tar.xz
      sha256: "dddeeefff..."
```

!!! tip "Always include SHA256"
    Every download must include a `sha256` field. The provisioner verifies the hash and fails if it does not match. This is what makes blueprints reproducible.

If your toolchain is in the catalog, use `unarchive_from_ref` instead; it reads the URL and SHA from `alloy.lock.yml`:

```yaml
- name: Install ARM GCC toolchain
  action: unarchive_from_ref
  ref: toolchain.arm-gnu.arm-none-eabi@stable
  dest: "{{.Vars.ARM_TOOLCHAIN_DEST}}"
  strip_components: 1
  creates: "{{.Vars.ARM_TOOLCHAIN_DEST}}/bin/arm-none-eabi-gcc"
```

---

## Layer 2: Environment (`20-environment.yml`)

Configure the environment so tools are available and correctly referenced. This layer writes shell profile scripts, CMake toolchain files, and pkg-config configuration.

```yaml
- name: Configure PATH for ARM toolchain
  action: write_env_file
  file: /etc/profile.d/10-arm-toolchain.sh
  content: |
    export PATH=$PATH:{{.Vars.ARM_TOOLCHAIN_DEST}}/bin
    export ARM_NONE_EABI_GCC={{.Vars.ARM_TOOLCHAIN_DEST}}/bin/arm-none-eabi-gcc
  owner: root:root
  mode: "0644"

- name: Generate CMake toolchain file
  action: cmake_toolchain
  file: /opt/cmake/arm-none-eabi.cmake
  target_system: Generic
  target_processor: ARM
  compiler_prefix: arm-none-eabi-
  compiler_path: "{{.Vars.ARM_TOOLCHAIN_DEST}}/bin"
  owner: root:root
  mode: "0644"
```

---

## Layer 3: Project tools (`30-project.yml`, optional)

Project-specific tools that are not part of the general toolchain: hardware debug probes, flashing utilities, udev rules for USB devices.

```yaml
- name: Install J-Link udev rules
  action: udev_rules
  rule_file: /etc/udev/rules.d/99-jlink.rules
  content: |
    SUBSYSTEM=="usb", ATTRS{idVendor}=="1366", MODE="0666", GROUP="plugdev"

- name: Install pyOCD
  action: python_package
  package: pyocd
  version: "0.36.0"
  creates: /usr/local/bin/pyocd
```

---

## Benefits of the layers approach

**Readability**: someone reading the blueprint immediately understands what each file does from its name and position.

**Independent re-provisioning**: if you update the toolchain URL, only `10-toolchain.yml` changes. The other layers are fingerprint-identical and are skipped on the next `alloy-host provision`.

**Easier review**: PRs that update the toolchain touch only `10-toolchain.yml`. PRs that add a package touch only `00-system.yml`. Changes are easy to understand and approve.

**Testability in isolation**: during development, you can temporarily comment out later layers to test just the system base or just the toolchain install.

**Composition**: for complex environments (e.g. Yocto with cross-compilation and an MCU SDK), layers make it natural to see the dependency chain: base → toolchain → SDK → project setup.

---

## Layer naming convention

The `00-`, `10-`, `20-` prefix is a convention. The actual execution order is controlled by `run_order:` in `manifest.yml`. Leave gaps between numbers so you can insert new layers without renumbering:

```yaml
run_order:
  - 00-system.yml
  - 10-toolchain.yml
  - 15-sdk.yml        # inserted later between toolchain and env
  - 20-environment.yml
  - 30-project.yml
```

---

## Real-world examples

Browse the [alloy-catalog](https://github.com/alloy-it/alloy-catalog) to see how community blueprints use the layers approach. Every blueprint in the catalog follows this structure.
