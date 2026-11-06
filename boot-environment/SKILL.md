---
name: boot-environment
description: Use when creating, changing, reviewing, or running a personal boot Starlark environment.
---

# Boot Environment

`boot` reads a Starlark recipe, usually `BOOT.star`, and registers tasks for
machine setup. Use this skill for recipe changes, plans, and applies. Read the
recipe and any loaded `.star` files before changing tasks or running commands.

## Recipe model

Top-level code should inspect the host and register tasks with `task(...)`.
Task functions emit actions through modules such as `fs`, `pkg`, `pacman`,
`git`, `go`, `systemd`, and `ssh`. `boot plan` checks those actions; `boot apply`
performs needed changes. Keep top-level code free of host mutations.

Give tasks stable IDs. Use `depends_on` for real ordering requirements, `when`
for host-specific registration, and `continue_on_error` for checks that should
warn without blocking setup. Prefer module actions over `shell.run`, and make
shell actions idempotent with `creates` or `only_if`. `shell.output` runs during
evaluation, so use it only for read-only probes.

For example, an Arch package task can keep the update consent next to the
actions it protects:

```python
def setup_packages():
    pkg.configure("pacman")
    consent.require(
        "Have you read https://archlinux.org/news/ and want to update packages?"
    )
    pkg.update()
    pkg.install(PACKAGES)

task(id="setup_packages", name="Install packages", run=setup_packages)
```

Preserve existing `consent.require(...)` calls unless the user asks to change
the policy. Do not hide warnings or discard dirty Git checkouts to make a plan
look clean. Treat `/etc`, `/usr`, `/boot`, `/efi`, system units, package updates,
and rescue images as host-sensitive changes.

## Work with a recipe

Find the recipe root and inspect `BOOT.star`, loaded files, and relevant host
data such as `environment.d/*.conf` and `etc/<hostname>`. Then use:

```sh
boot -C <root> list
boot -C <root> -only <task_id> plan
boot -C <root> plan
```

Use the focused plan for a changed task and the full plan to catch effects on
other tasks. `change` means an action would change the host; `warn` and `fail`
need review. Use `-verbose` to see skipped actions. Planning may still ask for
sudo credentials or refresh a separate pacman check database.

Review the plan before an authorized `boot -C <root> apply`. Explain package,
service, and system-file changes and any warnings or failures. An apply can
install packages, edit system files, enable services, and run shell commands.
After an apply, plan again to check for remaining changes. A settled recipe
should normally plan without changes.

For unfamiliar flags, check the installed `boot` help. For module APIs, inspect
existing recipes or the `boot` source. Keep changes small and inspect the diff
before handing them back.
