# Why Alloy-it?

## The problem with embedded dev environments

Setting up a build environment for embedded software is one of the most frustrating parts of the job. It's not a skill issue; it's that there's no standard way to do it, and every project has a slightly different setup.

Here's what a typical first day on a new embedded project looks like:

1. Clone the repo
2. Read the README: it says "install ARM GCC 12.3 and the vendor SDK"
3. Install ARM GCC, but Homebrew gives you 13.2, not 12.3
4. Find the right download for your OS from the vendor site
5. Set up PATH, check it's the right version
6. Install the SDK, but it depends on Python 3.10 and your system has 3.12
7. Install Python 3.10 alongside your existing Python; now you have three
8. Follow the vendor's SDK setup guide, written for Ubuntu 20.04, but you're on macOS
9. Run `cmake ..` and hit a missing dependency
10. Search Stack Overflow for the next two hours
11. It finally works, but your colleague hits a different error because they're on Windows

This is not exceptional. This is Tuesday.

---

## What makes it hard

The core issue is that embedded toolchains have **hard version dependencies** that don't always align with what your system package manager provides. The ARM GCC compiler, the vendor SDK, the required Python version, the right version of CMake: they all have to match, and the window of acceptable versions is narrow.

Beyond that:

**Toolchains are large and opinionated.** They install files in global locations, set environment variables, and sometimes conflict with other projects on the same machine.

**Documentation drifts.** A README written for Ubuntu 20.04 isn't tested on macOS Sonoma. By the time someone finds the gap, it's already caused a day of lost work.

**New team members suffer most.** An experienced developer can debug a broken toolchain setup. A new hire trying to get their first build working is staring at opaque errors with no context.

**CI/CD diverges from local.** You get it working on your machine, then discover CI is using different tool versions. You fix CI, break your local. This cycle is a productivity killer. With Alloy, **one blueprint** fixes that; local and CI use the same definition.

---

## How Alloy solves it

Alloy separates **defining** an environment from **using** one.

You write a blueprint, a single universal declaration that says exactly what your environment needs: which tools, which versions, where they come from, how they're configured. This blueprint lives in your repository alongside your source code. The **same blueprint** runs on local VMs, on servers, and in CI: same file, same lockfile, same result.

When anyone on your team runs `alloy-host up`, Alloy:

1. Creates an isolated Linux VM on their machine
2. Reads the blueprint
3. Downloads and installs exactly the tools specified
4. Configures environment variables and paths

The result is a running VM where everything is already set up. SSH in and build.

The blueprint is versioned in git. When you update the toolchain, you update the blueprint, commit it, and your teammates pick up the change with a `git pull` + `alloy-host up`. No manual steps, no README updates, no "did you upgrade yet?" conversations.

alloy-host is one way to use Alloy — it manages VM and container lifecycles for you. You can also run alloy-provisioner directly on any Linux machine, inside WSL2, or as part of a Docker build. The blueprint is the same in all cases; only the host environment changes. See [Get Started](get-started/index.md) for all deployment paths.

---

## Isn't this what Docker is for?

Docker is a great tool, but it makes specific trade-offs that don't always fit embedded development.

|                           | Docker                  | Alloy                         |
| ------------------------- | ----------------------- | ----------------------------- |
| USB device access         | Limited, host-dependent | First-class support           |
| Vendor SDK support        | Often incompatible      | Full Linux environment        |
| Windows host support      | Requires WSL2 tuning    | Native via WSL2 or VirtualBox |
| Persistent workspace      | Volume mounts           | Dedicated persistent disk     |
| Environment isolation     | Process-level           | Full VM isolation             |
| Familiar to embedded devs | Sometimes               | Always (it's just a VM)       |

USB passthrough in particular is a consistent pain point with Docker. Alloy treats hardware access as a first-class feature: you can attach your target board to the VM and flash firmware without leaving your development environment.

---

## Who is Alloy for?

**Individual developers** who are tired of maintaining a mess of installed tools and want clean, project-specific environments they can throw away and rebuild.

**Teams** who want everyone to use the same toolchain without spending time on environment support.

**Companies** who need to reproduce a build environment exactly (same compiler, same SDK, same everything) for compliance, debugging, or long-term support of deployed firmware.

**Open source projects** that want contributors to get a consistent build environment with minimal friction: one blueprint, one command, no setup docs to maintain.

If you've ever said "it builds on my machine" or spent more than an hour setting up a toolchain, Alloy is for you.

---

[Get started →](get-started/index.md)
