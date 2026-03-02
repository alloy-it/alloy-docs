# Publishing to Alloy Hub

[Alloy Hub](https://alloy-it.io) indexes blueprints from the **alloy-catalog** GitHub repository. To publish a blueprint, open a pull request to alloy-catalog; automated checks run, and on merge the blueprint becomes available on Alloy Hub.

---

## Prerequisites

Before publishing, make sure your blueprint:

- [ ] Provisions successfully on both `linux/amd64` and `linux/arm64` (use `per_arch` overrides)
- [ ] Includes `sha256:` on every downloaded file
- [ ] Installs tools to documented, predictable paths
- [ ] Sets up environment variables via `write_env_file` (so tools are on `PATH` after provisioning)
- [ ] Includes udev rules for common USB debuggers (if applicable)
- [ ] Has been tested by actually provisioning it into a VM or natively

---

## Step 1: Fork alloy-catalog

1. Go to [github.com/alloy-it/alloy-catalog](https://github.com/alloy-it/alloy-catalog)
2. Click **Fork** to create your own copy

---

## Step 2: Clone your fork

```bash
git clone https://github.com/YOUR_USERNAME/alloy-catalog.git
cd alloy-catalog
```

---

## Step 3: Create a branch

```bash
git checkout -b add-my-toolchain-blueprint
```

---

## Step 4: Add your blueprint

Place your blueprint under `blueprints/<vendor>/<target>/`:

```
blueprints/
└── my-vendor/
    └── my-toolchain/
        ├── manifest.yml
        ├── 00-system.yml
        ├── 10-toolchain.yml
        ├── 20-environment.yml
        ├── alloy.lock.yml     # if using catalog refs
        └── README.md          # required: see below
```

---

## Step 5: Write README.md

A `README.md` is required for catalog blueprints. Include:

```markdown
# My Vendor — My Toolchain

Brief description of what this environment installs.

## Included tools

| Tool | Version | Installed path |
|---|---|---|
| my-toolchain | 1.2.3 | `/opt/my-toolchain` |
| cmake | system | system |

## Environment variables

Set by this blueprint:

| Variable | Value |
|---|---|
| `PATH` | Includes `/opt/my-toolchain/bin` |
| `MY_TOOLCHAIN_ROOT` | `/opt/my-toolchain` |

## Supported boards / targets

- Board / chip family A
- Board / chip family B

## Tested on

- `linux/amd64`: Ubuntu 22.04
- `linux/arm64`: Ubuntu 22.04
```

---

## Step 6: Commit and push

```bash
git add blueprints/my-vendor/
git commit -m "feat: add my-toolchain blueprint"
git push origin add-my-toolchain-blueprint
```

---

## Step 7: Open a pull request

Go to your fork on GitHub and click **Compare & pull request**. Target the `main` branch of `alloy-it/alloy-catalog`.

In the PR description, include:
- What the blueprint installs and which versions
- Which boards or targets it is meant for
- Which platforms you have tested it on

---

## Automated checks

When you open the PR, CI runs automatically:

- **YAML schema validation**: checks that `manifest.yml` and task files follow the blueprint schema
- **SHA256 presence**: verifies that every download task includes a `sha256` field
- **Structure check**: confirms `README.md` exists and `run_order` matches actual task files

Fix any errors reported by CI and push new commits to the same branch; the checks re-run automatically.

---

## After merge

Once the PR is merged into `alloy-catalog/main`:

- The blueprint appears on [Alloy Hub](https://alloy-it.io) within minutes.
- Users can install it with `alloy-provisioner install <your-vendor>/<your-blueprint>`.
- Users with alloy-host can find it with `alloy-host catalog search`.

---

## Updating a published blueprint

To update a blueprint (new toolchain version, added package, bug fix):

1. Create a new branch in your fork (or the alloy-catalog fork if you are a maintainer).
2. Edit the relevant task files.
3. **Bump the version** in `manifest.yml`.
4. Re-run `alloy-host resolve` if catalog refs changed.
5. Open a new PR.

---

## Sharing within your team (without publishing)

You do not need to publish to Alloy Hub to share a blueprint with your team. Commit the blueprint directory and lockfile to your project repo:

```bash
git add blueprint/ alloy.lock.yml
git commit -m "add development environment blueprint"
git push
```

Teammates clone the repo and provision:

```bash
# With alloy-host
alloy-host init dev --blueprint ./blueprint
alloy-host up

# Native Linux
sudo alloy-provisioner --blueprint-dir ./blueprint
```

The lockfile ensures everyone uses identical tool versions.

---

## Next steps

- [Blueprint Structure](structure.md): ensure your blueprint follows the expected layout
- [The Layers Approach](layers.md): required for catalog blueprints
- [Browse existing blueprints](https://github.com/alloy-it/alloy-catalog): see how published blueprints are structured
