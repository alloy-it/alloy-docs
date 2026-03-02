# Write a Blueprint from Scratch

This guide walks through writing a complete ARM Cortex-M blueprint from scratch: manifest, task files (using the layers approach), and testing locally before publishing.

---

## What we'll build

An ARM Cortex-M development environment with:

- Base OS packages (cmake, ninja, git, python3)
- ARM GCC 13.3 cross-compilation toolchain
- PATH and CMake toolchain file configuration

By the end you'll have a working blueprint that can provision a native Linux machine, a VM, or a Docker container.

---

## Step 1: Create the directory

```bash
mkdir my-arm-env && cd my-arm-env
```

---

## Step 2: Write `manifest.yml`

```yaml
name: ARM Cortex-M Development Environment
version: 1.0.0
description: ARM GCC 13.3 toolchain for Cortex-M embedded development.

variables:
  ARM_TOOLCHAIN_DEST: /opt/arm-none-eabi
  DEV_USERNAME: vagrant

run_order:
  - 00-system.yml
  - 10-toolchain.yml
  - 20-environment.yml
```

Start with the minimum required fields: `name`, `version`, and `run_order`. Add `variables:` for any value that appears in more than one task file, or that you might want to override.

---

## Step 3: Write `variables.yml` (optional)

For larger blueprints, move variable definitions to a separate file to keep the manifest clean:

```yaml
# variables.yml
ARM_TOOLCHAIN_DEST: /opt/arm-none-eabi
DEV_USERNAME: vagrant
TOOLCHAIN_VERSION: "13.3.rel1"
```

If you use `variables.yml`, you can remove the same keys from `manifest.yml variables:`. The manifest always takes precedence on conflict.

---

## Step 4: Write the system layer (`00-system.yml`)

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
    - wget
    - curl
    - ca-certificates
    - xz-utils
    - gdb-multiarch
```

---

## Step 5: Write the toolchain layer (`10-toolchain.yml`)

Download and extract the ARM GCC toolchain. Use `per_arch` to support both `amd64` and `arm64` host machines:

```yaml
- name: Install ARM GCC 13.3 toolchain
  action: unarchive_from_url
  dest: "{{.Vars.ARM_TOOLCHAIN_DEST}}"
  strip_components: 1
  creates: "{{.Vars.ARM_TOOLCHAIN_DEST}}/bin/arm-none-eabi-gcc"
  per_arch:
    amd64:
      url: https://developer.arm.com/-/media/Files/downloads/gnu/13.3.rel1/binrel/arm-gnu-toolchain-13.3.rel1-x86_64-arm-none-eabi.tar.xz
      sha256: "aaabbbccc111222333444555666777888999000aaabbbccc111222333444555666"
    arm64:
      url: https://developer.arm.com/-/media/Files/downloads/gnu/13.3.rel1/binrel/arm-gnu-toolchain-13.3.rel1-aarch64-arm-none-eabi.tar.xz
      sha256: "dddeeefff111222333444555666777888999000dddeeefff111222333444555666"
```

!!! warning "SHA256 is mandatory"
    Always include `sha256` on every download. The provisioner verifies the hash and fails if it does not match. This is what makes blueprints reproducible: anyone who runs this blueprint gets byte-for-byte identical toolchain binaries.

To find the SHA256:

```bash
sha256sum arm-gnu-toolchain-13.3.rel1-x86_64-arm-none-eabi.tar.xz
```

---

## Step 6: Write the environment layer (`20-environment.yml`)

Configure the shell environment and generate a CMake toolchain file:

```yaml
- name: Add ARM GCC to PATH
  action: write_env_file
  file: /etc/profile.d/10-arm-toolchain.sh
  content: |
    export PATH=$PATH:{{.Vars.ARM_TOOLCHAIN_DEST}}/bin
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

## Step 7: Validate the blueprint

Check for syntax errors before provisioning:

```bash
alloy-host validate ./my-arm-env
```

This catches missing required fields, unknown action types, and YAML syntax errors without starting a VM.

---

## Step 8: Test by provisioning

**With alloy-host (recommended):**

```bash
alloy-host init test-env --blueprint ./my-arm-env
cd test-env
alloy-host up
```

Watch the provisioner execute each task. If a task fails, fix the issue and re-run `alloy-host provision`; completed tasks are skipped.

**Natively on a Linux machine:**

```bash
sudo alloy-provisioner --blueprint-dir ./my-arm-env
```

---

## Step 9: Verify inside the environment

```bash
alloy-host ssh

# Inside the VM:
arm-none-eabi-gcc --version
# arm-none-eabi-gcc (Arm GNU Toolchain 13.3.Rel1) 13.3.1

cmake --version
ninja --version
ls /opt/cmake/arm-none-eabi.cmake
```

Try building a real project:

```bash
cd /vagrant/my-project
cmake -GNinja -DCMAKE_TOOLCHAIN_FILE=/opt/cmake/arm-none-eabi.cmake -B build
ninja -C build
```

---

## Step 10: Resolve the lockfile (if using catalog refs)

If you referenced catalog toolchains in `manifest.yml toolchains:`, generate the lockfile:

```bash
alloy-host resolve
```

If you specified all URLs and SHA256 values directly in task files (as in this guide), you do not need a lockfile.

---

## Step 11: Commit

```bash
# Add to .gitignore
echo "alloy.state.yml" >> .gitignore
echo ".env" >> .gitignore

git add manifest.yml 00-system.yml 10-toolchain.yml 20-environment.yml
git add alloy.lock.yml  # if it exists
git commit -m "feat: add ARM Cortex-M development environment blueprint"
```

---

## Complete file listing

```toml
my-arm-env/
├── .gitignore
├── manifest.yml
├── 00-system.yml
├── 10-toolchain.yml
└── 20-environment.yml
```

---

## Next steps

- [Add variables.yml and env files](variables.md): credentials and per-machine settings
- [Extend with more layers](layers.md): project-specific tools and udev rules
- [Publish to Alloy Hub](publishing.md): share with the community
