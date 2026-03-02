# Variables & Configuration

Blueprints support three complementary mechanisms for injecting values into tasks: manifest variables, a separate `variables.yml` file, and environment variables loaded from a `.env` file. Each serves a different purpose.

---

## 1. Manifest variables

Define static values in `manifest.yml` under `variables:`. These are available in all task file fields using Go template syntax:

```yaml
# manifest.yml
variables:
  ARM_TOOLCHAIN_DEST: /opt/arm-none-eabi
  DEV_USERNAME: vagrant
  TOOLCHAIN_VERSION: "13.3.rel1"
```

Reference them in task files with `{{.Vars.NAME}}`:

```yaml
# 10-toolchain.yml
- name: Install ARM GCC toolchain
  action: unarchive_from_url
  dest: "{{.Vars.ARM_TOOLCHAIN_DEST}}"
  creates: "{{.Vars.ARM_TOOLCHAIN_DEST}}/bin/arm-none-eabi-gcc"

- name: Configure PATH
  action: write_env_file
  file: /etc/profile.d/10-arm-toolchain.sh
  content: |
    export PATH=$PATH:{{.Vars.ARM_TOOLCHAIN_DEST}}/bin
    export ARM_TOOLCHAIN_VERSION={{.Vars.TOOLCHAIN_VERSION}}
```

Manifest variables are good for values that are the same across all environments: installation paths, tool names, version strings.

---

## 2. variables.yml

For larger blueprints or when you want to separate variable definitions from blueprint metadata, create a `variables.yml` file:

```yaml
# variables.yml
ARM_TOOLCHAIN_DEST: /opt/arm-none-eabi
DEV_USERNAME: vagrant
TOOLCHAIN_VERSION: "13.3.rel1"
SDK_DEST: /opt/sdk
```

`variables.yml` is merged with manifest variables before task execution. **Manifest `variables:` takes precedence** over `variables.yml` , so you can use `variables.yml` as a base and override specific values in `manifest.yml`.

This is useful when:

- Multiple blueprints share common variable definitions.
- You want to keep `manifest.yml` focused on metadata and structure.
- The set of variables is large and would clutter the manifest.

---

## 3. Environment file (`.env`)

Use a `.env` file for values that change per machine or contain credentials: registry tokens, usernames, private artifact URLs. These should **never be committed**.

Create a `.env` file in your blueprint directory:

```bash
# .env — never commit this file
GITLAB_TOKEN=glpat-xxxxxxxxxxxx
SDK_DESTINATION=/opt/my-sdk
DEV_USERNAME=ubuntu
ALLOY_REGISTRY_USERNAME=myuser
ALLOY_REGISTRY_PASSWORD=my-token
```

Source it before running the provisioner:

```bash
set -a && source .env && set +a
sudo -E alloy-provisioner --blueprint-dir /path/to/blueprint
```

Or use the `-env-file` flag:

```bash
sudo alloy-provisioner --blueprint-dir /path/to/blueprint -env-file /path/to/.env
```

When using **alloy-host**, pass the env file at init time or place it in the blueprint directory; alloy-host picks it up when running the provisioner inside the guest.

!!! warning "Never commit `.env` files"
Add `.env` to `.gitignore`. For CI/CD, use pipeline secret management (GitHub Secrets, GitLab CI Variables, etc.) and inject them as environment variables.

---

## 4. Required environment variables

Use `required_env:` in `manifest.yml` to declare which environment variables must be set before provisioning runs. If any are missing, the provisioner exits with a clear error listing all missing variables and instructions for how to set them.

```yaml
# manifest.yml
required_env:
  - name: GITLAB_TOKEN
    description: Personal access token for the private artifact registry
  - name: SDK_DESTINATION
    description: Where to install the proprietary SDK (e.g. /opt/vendor-sdk)
```

When a required variable is missing:

```
Error: missing required environment variables:
  GITLAB_TOKEN — Personal access token for the private artifact registry
  SDK_DESTINATION — Where to install the proprietary SDK

Set them in a .env file and run:
  set -a && source .env && set +a && alloy-provisioner --blueprint-dir .
```

---

## 5. Toolchain alias injection

When a `toolchains:` entry in `manifest.yml` has an `alias`, the provisioner automatically injects two variables after resolving the lockfile:

```yaml
# manifest.yml
toolchains:
  - ref: toolchain.arm-gnu.arm-none-eabi@stable
    alias: ARM_TOOLCHAIN
```

This makes `{{.Vars.ARM_TOOLCHAIN_URL}}` and `{{.Vars.ARM_TOOLCHAIN_SHA}}` , pointing to the exact URL and checksum from `alloy.lock.yml`.

Use them with `unarchive_from_ref` (recommended; uses the lockfile automatically):

```yaml
- name: Install ARM GCC via catalog ref
  action: unarchive_from_ref
  ref: toolchain.arm-gnu.arm-none-eabi@stable
  dest: "{{.Vars.ARM_TOOLCHAIN_DEST}}"
  strip_components: 1
  creates: "{{.Vars.ARM_TOOLCHAIN_DEST}}/bin/arm-none-eabi-gcc"
```

Or with `unarchive_from_url` if you need the URL and SHA explicitly:

```yaml
- name: Install ARM GCC
  action: unarchive_from_url
  url: "{{.Vars.ARM_TOOLCHAIN_URL}}"
  sha256: "{{.Vars.ARM_TOOLCHAIN_SHA}}"
  dest: "{{.Vars.ARM_TOOLCHAIN_DEST}}"
```

---

## Variable resolution order

Variables are resolved in this order (later entries win on conflict):

1. `variables.yml`: base definitions
2. `manifest.yml` `variables:`: overrides `variables.yml`
3. Process environment (including `.env`): overrides manifest variables
4. Toolchain alias injection: appended after lockfile resolution

---

## Security note

The `run_command.command` field is **not expanded** for environment variables. : it prevents command injection when commands come from user-controlled input. All other task fields that accept strings are safely expanded.

---

## Summary

| Mechanism                 | Use for                               | Syntax                | Commit? |
| ------------------------- | ------------------------------------- | --------------------- | ------- |
| `manifest.yml variables:` | Static values, paths, versions        | `{{.Vars.NAME}}`      | Yes     |
| `variables.yml`           | Shared base variables                 | `{{.Vars.NAME}}`      | Yes     |
| `.env` file               | Credentials, per-machine settings     | `$NAME` (env var)     | No      |
| `required_env:`           | Declare mandatory env vars            | Declaration only      | Yes     |
| Toolchain alias           | Auto-injected URL + SHA from lockfile | `{{.Vars.ALIAS_URL}}` | n/a     |
