# On Linux (Native)

Install alloy-provisioner directly on any Linux machine: bare metal, a test rack, a CI server, or any existing VM you already manage (VirtualBox, Proxmox, Vagrant, WSL2, etc.). No wrapper tool needed.

---

## Install alloy-provisioner

### Quick install (script)

On Linux (amd64 or arm64), run the install script. It detects your architecture and installs the binary to `/usr/local/bin/`.

```bash
curl -fsSL https://raw.githubusercontent.com/alloy-it/alloy-provisioner-releases/main/scripts/install.sh | bash
```

Requirements: `curl` or `wget`, `sudo` access. Then verify: `alloy-provisioner -version`.

### Manual install (.deb)

Download and install the `.deb` package from [alloy-provisioner-releases](https://github.com/alloy-it/alloy-provisioner-releases):

=== "amd64"

```bash
wget -q -O /tmp/alloy-provisioner.deb \
  https://github.com/alloy-it/alloy-provisioner-releases/releases/download/latest/alloy-provisioner_latest_linux_amd64.deb
sudo apt-get install -y /tmp/alloy-provisioner.deb
rm /tmp/alloy-provisioner.deb
```

=== "arm64"

```bash
wget -q -O /tmp/alloy-provisioner.deb \
      https://github.com/alloy-it/alloy-provisioner-releases/releases/download/latest/alloy-provisioner_latest_linux_arm64.deb
sudo apt-get install -y /tmp/alloy-provisioner.deb
rm /tmp/alloy-provisioner.deb
```

=== "Auto-detect arch"

```bash
ARCH=$(dpkg --print-architecture)
wget -q -O /tmp/alloy-provisioner.deb \
      "https://github.com/alloy-it/alloy-provisioner-releases/releases/download/latest/alloy-provisioner_latest_linux_${ARCH}.deb"
sudo apt-get install -y /tmp/alloy-provisioner.deb
rm /tmp/alloy-provisioner.deb
```

Verify the installation:

```bash
alloy-provisioner -version
```

---

## Option A: Install from Alloy Hub

Pull a blueprint from [Alloy Hub](https://alloy-it.io) and provision in one step:

```bash
sudo alloy-provisioner install community/arm-none-eabi
```

To install a specific version:

```bash
sudo alloy-provisioner install community/arm-none-eabi:1.2.0
```

The provisioner downloads the blueprint, then executes all tasks. It prints progress as it goes and records completed tasks so re-runs skip what is already done.

---

## Option B: Provision from a local blueprint

If you have a blueprint directory (e.g. committed alongside your project):

```bash
sudo alloy-provisioner --blueprint-dir /path/to/blueprint
```

Or set the directory via environment variable:

```bash
export ALLOY_BLUEPRINT_DIR=/path/to/blueprint
sudo -E alloy-provisioner
```

---

## Environment variables and credentials

Blueprint tasks can use environment variables for credentials, usernames, or environment-specific settings. Set them before running the provisioner using an env file:

```bash
# Create an env file (never commit this file)
cat > ~/.alloy-env << 'EOF'
GITLAB_TOKEN=glpat-xxxxx
SDK_DESTINATION=/opt/sdk
DEV_USERNAME=ubuntu
EOF
chmod 600 ~/.alloy-env

# Source and run
set -a && source ~/.alloy-env && set +a
sudo -E alloy-provisioner --blueprint-dir /path/to/blueprint
```

Or use the `-env-file` flag directly:

```bash
sudo alloy-provisioner --blueprint-dir /path/to/blueprint -env-file ~/.alloy-env
```

!!! warning "Never commit `.env` files"
    Env files contain credentials. Add them to `.gitignore`. Use CI/CD secret management for pipeline runners.

If the blueprint declares `required_env:` in `manifest.yml`, the provisioner validates all required variables are set before running and exits with a clear error if any are missing.

---

## Re-provisioning

Running the provisioner again is always safe. It uses three layers of idempotency:

1. **`creates:` guard**: if the output path exists, the task is skipped immediately.
2. **State file**: `alloy.state.yml` records a fingerprint of each completed task; tasks with unchanged inputs are skipped.
3. **Adapter-level checks**: e.g. `apt_install` only installs packages that are not already present.

To force all tasks to re-run:

```bash
rm alloy.state.yml
sudo alloy-provisioner --blueprint-dir /path/to/blueprint
```

---

## Next steps

- [Browse blueprints on Alloy Hub](https://alloy-it.io)
- [Write your own blueprint](../blueprints/index.md)
- [Variable and env file reference](../blueprints/variables.md)
