# Using Alloy Hub

[Alloy Hub](https://alloy-it.io) is the web interface and registry for Alloy blueprints. Use it to discover community blueprints, manage your organization’s blueprints, and get the blueprint names you need for `alloy-provisioner install` and `alloy-host init`.

---

## Explore

The **Explore** area lists public blueprints you can use without logging in. Each blueprint has a name (e.g. `community/nordic/nrf91:1.1.3`, `community/raspberry-pi-arm64`) and version tags (e.g. `1.0.0`, `1.1.3`).

- **Blueprint name:** Use this with the provisioner or alloy-host, e.g. `alloy-provisioner install community/nordic/nrf91:1.1.3` or `alloy-host init my-env --blueprint community/nordic/nrf91:1.1.3`.
- **Tags:** Shown on the blueprint detail page. Always pin a version in CI or scripts, e.g. `community/nordic/nrf91:1.1.3`.

---

## My Hub (organizations)

If you have an organization on Alloy Hub, **My Hub** gives you:

- **Repositories:** Create and manage private or org-scoped blueprints. Push new versions from your machine using the Alloy CLI and the credentials from Hub.
- **Collaborations:** Invite members and manage access.
- **Settings:** Billing, usage, and notifications (depending on your plan).

To use a private or org blueprint, log in so the provisioner can authenticate to the registry. See [Installing environments](installing/index.md) and [Blueprint variables](blueprints/variables.md) for env files and registry credentials.

---

## Finding blueprint names for install or clone

1. Open [alloy-it.io](https://alloy-it.io) and go to **Explore** (or **My Hub** for your org’s blueprints).
2. Click a blueprint to open its detail page.
3. Use the **pull** or **install** command shown there, or build the name yourself: `project/repository` (e.g. `community/raspberry-pi-arm64`). Add `:tag` for a specific version.

Examples:

```bash
alloy-provisioner install community/nordic/nrf91:1.1.3
alloy-provisioner install community/raspberry-pi-arm64:1.0.0
alloy-host init nrf-dev --blueprint community/nordic/nrf91:1.1.3
```

---

## Next steps

- [Get started](get-started/index.md): choose your installation path (native, alloy-host, Docker).
  - [Installing environments](installing/index.md): install the provisioner and run a blueprint.
  - [Publishing to Alloy Hub](blueprints/publishing.md): add a community blueprint or publish an org blueprint.
