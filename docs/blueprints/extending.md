# Extend from Alloy Hub

The fastest way to write a blueprint is to start with one that already exists on [Alloy Hub](https://alloy-it.io) and customize it for your project. This is the recommended approach when a similar environment already exists.

---

## 1. Find a blueprint on Alloy Hub

Browse [alloy-it.io](https://alloy-it.io) in your browser, or search the catalog locally:

```bash
alloy-host catalog update
alloy-host catalog search arm
alloy-host catalog info toolchain.arm-gnu.arm-none-eabi@stable
```

Or use alloy-provisioner to list available blueprints:

```bash
alloy-provisioner clone --list community
```

---

## 2. Clone the blueprint

Download a blueprint from Alloy Hub to a local directory:

```bash
alloy-provisioner clone community/arm-none-eabi --output ./my-blueprint
```

This creates a `my-blueprint/` directory with the manifest, task files, and lockfile from the hub.

Alternatively, if the blueprint is in the alloy-catalog GitHub repo, you can clone the relevant directory directly:

```bash
# Find the blueprint directory in alloy-catalog
git clone https://github.com/alloy-it/alloy-catalog.git
cp -r alloy-catalog/blueprints/arm-gnu/arm-none-eabi ./my-blueprint
```

---

## 3. Understand the existing structure

Read the manifest to understand what the blueprint does:

```bash
cat my-blueprint/manifest.yml
```

Typical structure (following the layers approach):

```toml
my-blueprint/
├── manifest.yml        # name, version, variables, run_order
├── 00-system.yml       # base OS packages
├── 10-toolchain.yml    # toolchain download and install
├── 20-environment.yml  # PATH and env vars
└── alloy.lock.yml      # pinned URLs and SHAs
```

Read each task file to understand what is being installed and where. ; these control installation paths and versions.

---

## 4. Customize for your project

**Add packages to an existing layer:**

```yaml
# 00-system.yml — add after existing tasks
- name: Install project-specific tools
  action: apt_install
  packages:
    - libssl-dev
    - doxygen
    - cppcheck
```

**Override a variable by editing `manifest.yml`:**

```yaml
variables:
  ARM_TOOLCHAIN_DEST: /opt/my-project/arm-gcc   # override the default path
  DEV_USERNAME: ubuntu                           # override default username
```

**Add a new layer for project-specific setup:**

Create `30-project.yml`:

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

Add it to `run_order:` in `manifest.yml`:

```yaml
run_order:
  - 00-system.yml
  - 10-toolchain.yml
  - 20-environment.yml
  - 30-project.yml     # add your new layer
```

**Update the version:**

```yaml
name: ARM Cortex-M — My Project
version: 1.1.0          # bump from the original
description: ARM GCC toolchain with project-specific tools for MyProject.
```

---

## 5. Test locally

Test your customized blueprint by provisioning into a local VM with alloy-host:

```bash
alloy-host init test-env --blueprint ./my-blueprint
cd test-env
alloy-host up
alloy-host ssh
```

Or natively on a Linux machine:

```bash
sudo alloy-provisioner --blueprint-dir ./my-blueprint
```

Verify that your additions work as expected. If something fails, fix the task file and re-run; completed tasks are skipped, so only the failed (and new) tasks execute.

---

## 6. Update the lockfile

If you added or changed any `toolchains:` catalog refs in `manifest.yml`, regenerate the lockfile:

```bash
alloy-host resolve
```

This writes a new `alloy.lock.yml` with resolved URLs and SHA256 checksums.

If you only added apt packages or local downloads with explicit `sha256:` fields, you do not need to re-resolve.

---

## 7. Commit

```bash
git add manifest.yml 00-system.yml 30-project.yml alloy.lock.yml
git commit -m "feat: extend arm-none-eabi blueprint with project tools"
```

Commit the blueprint directory and the lockfile. Do not commit `alloy.state.yml` or `.env`.

---

## Next steps

- [Sharing with your team](../installing/with-alloy-host.md): give teammates a one-command setup
- [Publishing to Alloy Hub](publishing.md): contribute your blueprint back to the community
- [Variables & Configuration](variables.md): environment files and required variables
