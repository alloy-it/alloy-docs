# Alloy

**Reproducible build environments for embedded software teams.**

Setting up a cross-compilation toolchain shouldn't take two days. Onboarding a new developer shouldn't mean walking them through a wall of manual steps. And "it builds on my machine" is not an acceptable answer.

Alloy solves this by letting you define your build environment as code: a single blueprint that anyone on your team can use to install an identical build environment in minutes — on a native Linux machine, in WSL2, in Docker, or in an isolated VM. Define your environment once; others can reuse it with one command. No README archaeology, no copy-paste, no "it works on my machine."

---

## How it works

### 1. Choose your blueprint from Alloy Hub

Browse [alloy-it.io](https://alloy-it.io) to find a blueprint for your toolchain or board. Note the blueprint name (e.g. `community/arm-none-eabi`, `community/raspberry-pi-arm64`) and optional version tag.

### 2. Download and install alloy-provisioner

On a Linux machine (amd64 or arm64), run the install script. It detects your architecture and installs the binary to `/usr/local/bin/`.

```bash
curl -fsSL \
  https://raw.githubusercontent.com/alloy-it/alloy-provisioner-releases/main/scripts/install.sh \
  | bash
```

Requirements: `curl` or `wget`, `sudo` access. See [Installing on Linux](installing/native.md) for other options (manual .deb, specific version).

### 3. Install the environment

Run the provisioner with your blueprint. Use the name from Alloy Hub in the form `<vendor>/<board>:version` (version is optional; defaults to latest).

```bash
sudo alloy-provisioner install community/arm-none-eabi
sudo alloy-provisioner install community/raspberry-pi-arm64:1.0.0
```

The provisioner pulls the blueprint, verifies checksums, and installs the full build environment. Re-runs are safe; completed tasks are skipped.

---

## What you get

**A defined, versioned build environment**: not a README full of instructions that slowly goes out of date.

**Isolation**: your Zephyr project and your Embedded Linux project live in separate environments. Upgrading one doesn't break the other.

**Hardware access**: USB passthrough lets you flash and debug real hardware directly from your environment (when using alloy-host for VM-managed environments).

**Reproducibility**: a lockfile records the exact version of every tool. Run it six months later, on any machine, and get the same result.

**Same definition everywhere**: one blueprint for local dev and CI; no separate setup docs or divergent tool versions.

---

## Who it's for

Alloy-it is built for **embedded software developers** who are tired of:

- Spending the first day of a new project setting up their toolchain
- Onboarding a new team member with a multi-page setup doc
- Breaking their environment when they upgrade an SDK
- Dealing with "works on my machine" between macOS, Linux, and Windows hosts

---

## Ready to start?

<div class="grid cards" markdown>

- :material-clock-fast: **[Get started in 15 minutes](get-started/index.md)**

    Install alloy-provisioner, pick your deployment path (native, VM, or Docker), and build.

- :material-file-code: **[Browse the catalog](catalog/index.md)**

    Use a ready-made blueprint for popular embedded toolchains.

- :material-book-open-variant: **[Concepts](concepts.md)**

    How blueprints, the catalog, and Alloy Host work together.

- :material-help-circle: **[Why Alloy-it?](why-alloy.md)**

    Not sure if you need this? See what problem it solves.

</div>
