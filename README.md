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

### AI-API2Chat-Router

Exposes an OpenAI-compatible API backed by your own logged-in browser
sessions on consumer AI chat products (Claude, ChatGPT, Gemini, DeepSeek,
Grok, Perplexity) -- any app that speaks the OpenAI API can use your
existing chat subscriptions instead of a metered API key. **Read
[`ai-api2chat-router/DOCS.md`](ai-api2chat-router/DOCS.md) before
installing** -- this automates consumer chat accounts, which is
typically against those products' Terms of Service even on an account
you already pay for; built for private personal use only.

### AI-API2Chat-Router (Beta)

Same add-on, built from the source repo's `uat` branch. Independent
install from production: its own slug, its own `/data`. Treat it as a
preview, not something to depend on.

## Updates

Each add-on's `config.yaml` here points at a pre-built image in GHCR,
published publicly -- `ghcr.io/a4ng3i/...` for this account's own
add-ons, `ghcr.io/aharoncg/...` for AI-API2Chat-Router (built from its
own, separately-owned source repo). Installing/updating pulls that
image directly -- nothing is built on your Home Assistant instance.
