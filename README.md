# ha-apps

Home Assistant add-on **install repository**: this is what you point Home
Assistant's Add-on Store at. It carries only what Supervisor needs to list
and install each add-on -- its `config.yaml`, icon, logo, and changelog --
pointing at pre-built, publicly hosted container images. It does not
contain any add-on's source code.

## Installing

In Home Assistant: **Settings -> Add-ons -> Add-on Store -> ⋮ -> Repositories**,
add:

```
https://github.com/a4ng3i/ha-apps
```

Then install any add-on listed below from the store.

## Add-ons in this repository

### HA Git Sync

Safe, bidirectional Git sync for `/config` -- staged pulls,
validate-before-apply, automatic pre-pull backups, and human-in-the-loop
conflict resolution. See [`ha-git-sync/DOCS.md`](ha-git-sync/DOCS.md) for
setup and usage.

### HA Git Sync (Beta)

Same add-on, built automatically from the source repo's `uat` (staging)
branch -- lets you install and run an upcoming release for real, side by
side with the production add-on, before it's promoted to `main`. Fully
independent install: its own slug, its own `/data`, its own SSH deploy
key. **Do not point both the production and beta add-ons at the same
GitHub repo with automatic sync enabled on both** -- two independent sync
engines racing to push/pull the same `/config` is exactly the kind of
conflict this add-on exists to prevent. Use a separate test repo for the
beta instance, or leave its Sync policy on manual (the default).

## Updates

Each add-on's `config.yaml` here points at a pre-built image in GHCR
(`ghcr.io/a4ng3i/...`), published publicly. Installing/updating pulls that
image directly -- nothing is built on your Home Assistant instance.
