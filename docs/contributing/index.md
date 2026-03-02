# Contributing

Alloy is open source and contributions are welcome. This page covers how to contribute blueprints, report issues, and contribute to the core tools.

---

## Contributing a blueprint to Alloy Hub

The fastest way to help the community is to contribute a blueprint for a toolchain you're already using.

See the full guide: [Publishing to Alloy Hub](../blueprints/publishing.md)

### What makes a good catalog blueprint

- Works on both `amd64` and `arm64` hosts (use `per_arch` overrides for architecture-specific downloads)
- Includes `sha256:` checksums on all downloads
- Installs tools to predictable, documented paths
- Sets up environment variables via `write_env_file` so tools are immediately usable after provisioning
- Includes udev rules for common USB debuggers (if applicable)
- Has been tested by actually provisioning a VM or machine with it
- Follows the [layers approach](../blueprints/layers.md): `00-system.yml`, `10-toolchain.yml`, `20-environment.yml`

### Quick steps

1. Fork [alloy-catalog](https://github.com/alloy-it/alloy-catalog) on GitHub
2. Create a branch and add your blueprint under `blueprints/<vendor>/<target>/`
3. Include a `README.md` documenting installed tools, paths, env vars, and tested platforms
4. Open a pull request; automated checks validate schema, SHA256 presence, and structure
5. On merge, the blueprint appears on [Alloy Hub](https://alloy-it.io)

---

## Reporting issues

- **Alloy Host (CLI):** [github.com/alloy-it/alloy-host/issues](https://github.com/alloy-it/alloy-host/issues)
- **Alloy Provisioner:** [github.com/alloy-it/alloy-provisioner/issues](https://github.com/alloy-it/alloy-provisioner/issues)
- **Catalog blueprints:** [github.com/alloy-it/alloy-catalog/issues](https://github.com/alloy-it/alloy-catalog/issues)
- **Documentation:** [github.com/alloy-it/alloy-docs/issues](https://github.com/alloy-it/alloy-docs/issues)

When reporting a bug, please include:

- Your host OS and architecture
- `alloy-host check-health` output (if using alloy-host)
- The command you ran and the full output
- Your blueprint (if relevant)

---

## Contributing to Alloy Provisioner

Alloy Provisioner is also written in Go and uses a ports-and-adapters architecture. Each task action is an independent adapter implementing a single interface, making it straightforward to add new actions.

```bash
git clone https://github.com/alloy-it/alloy-provisioner
cd alloy-provisioner
make build
make test
```

---

## Improving the documentation

Documentation is in the [alloy-docs](https://github.com/alloy-it/alloy-docs) repository. To run it locally:

```bash
cd alloy-docs
source .venv/bin/activate    # activate the Python venv
mkdocs serve
```

Open [http://localhost:8000](http://localhost:8000) to preview your changes. Submit a pull request with corrections, clarifications, or new content.

---

## Keeping documentation accurate

Documentation is the single source of truth for users (the dashboard and marketing website link here). To keep it correct and complete:

### When changing CLI or tools

- **alloy-provisioner / alloy-host:** When you add or change subcommands, flags, or behavior, update the docs in the same PR (or a follow-up). Use `alloy-provisioner -version` / `alloy-host --help` and the real binaries as the source of truth.
- **Download URLs and release assets:** When cutting a new provisioner or alloy-host release, check that [Installing on Linux (Native)](../installing/native.md) and the [Home](../index.md) quick-install snippets still match the published assets (e.g. `.deb` filenames and `releases/download/latest/` or versioned URLs). Fix any broken or outdated URLs.

### When changing Alloy Hub or the dashboard

- If you add a major Hub feature (e.g. new auth flow, new UI), add or update the [Using Alloy Hub](../using-alloy-hub.md) page so the dashboard Help and other entry points can link to it.

### Doc review

- As part of a release or periodically (e.g. quarterly), walk the main user paths: install → first blueprint → publish; alloy-host flow; CI. Fix outdated steps, wrong commands, and broken internal or external links.
- Confirm the env file path (e.g. `/etc/alloy/env` or per-blueprint) where the provisioner writes the environment after a run, and update the docs if it differs.
