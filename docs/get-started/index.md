# Get Started

**alloy-provisioner** is the engine that installs build environments from blueprints. It runs inside any Linux system. Choose the path that matches your situation:

---

<div class="grid cards" markdown>

- :material-linux: **On Linux (Native)**

    ---

    You have a Linux machine (PC, laptop, or server, test rack, an existing VM managed by your own tools like Vagrant, VirtualBox, or Proxmox, or a WSL2 distro). Install alloy-provisioner and run blueprints directly.

    Best for: CI servers, test racks, shared build machines, manually provisioning your own VMs.

    [Install natively &rarr;](../installing/native.md)

- :material-monitor: **With Alloy Host**

    ---

    You are on a developer workstation (Windows, macOS, or Linux) and want isolated environments that don't touch your host machine. alloy-host manages VM and Docker lifecycles for you.

    Best for: local development, quickly testing a blueprint before publishing, switching between project environments.

    [Use alloy-host &rarr;](../installing/with-alloy-host.md)

- :material-docker: **In a Docker Image**

    ---

    You want to replace a complex, hard-to-maintain Dockerfile with a declarative blueprint. alloy-provisioner runs inside the Docker build and produces a reproducible image using the same definition used by developers and CI.

    Best for: CI base images, replacing multi-stage Dockerfiles and chains of dependent images.

    [Build Docker images &rarr;](../installing/in-docker.md)

</div>

---

## Which path should I choose?

| Situation | Recommended path |
|---|---|
| CI server or test rack (Linux PC/laptop/server) | [Native](../installing/native.md) |
| Existing VM I manage myself | [Native](../installing/native.md): run alloy-provisioner inside the VM |
| Developer workstation (want isolation) | [alloy-host](../installing/with-alloy-host.md) |
| Testing a blueprint before publishing | [alloy-host](../installing/with-alloy-host.md) |
| Docker CI image | [Docker](../installing/in-docker.md) |
| Multiple devs, same environment | Any path, all using the same blueprint |

---

## Before you start

All paths require a Linux environment with:

- **curl** and **wget**: for downloading alloy-provisioner and toolchains
- **apt**: alloy-provisioner targets Debian/Ubuntu-based systems
- **sudo** or root access: provisioner tasks install system packages and tools

If you are using alloy-host or Docker, these requirements apply to the guest (the VM or container), not your host machine.

For a full list of provisioner subcommands and flags, see [Provisioner CLI](../reference/provisioner-commands.md).
