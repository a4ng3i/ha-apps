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

## Updates

Each add-on's `config.yaml` here points at a pre-built image in GHCR
(`ghcr.io/a4ng3i/...`), published publicly. Installing/updating pulls that
image directly -- nothing is built on your Home Assistant instance.
