# ChatRelay

## Why this exists

Exposes a standard, OpenAI-compatible API on one side. On the other side
it drives your own logged-in browser sessions on consumer AI chat
products (Claude, ChatGPT, Gemini, DeepSeek, Grok, Perplexity) to
actually answer requests - so any app that speaks the OpenAI API can use
your existing chat subscriptions instead of a metered API key.

**Read this before installing:** automating a consumer chat web UI to
serve API-style traffic is outside the intended use of these products
and typically against their Terms of Service, even though it's
technically the same account you already pay for. This add-on is built
for your own private, personal use only - not to be resold, shared
publicly, or used to serve other people's traffic. Use good judgement;
heavy or careless use can get the underlying account rate-limited or
banned.

## Setup

Everything below works from inside Home Assistant: click **Open Web
UI** on the add-on page, or **ChatRelay** in the sidebar (HA admins
only). HA handles the login there, so the dashboard opens without the
separate admin password. Direct access on port 8420 also works and
still requires `admin_username`/`admin_password`.

1. Start the add-on (it starts fine with no options set). Open the
   web UI (or `http://homeassistant.local:8420/admin/setup`) and click "Generate
   master key" - this encrypts everything the add-on stores, and is
   written straight into the add-on's Configuration options and
   restarted for you, no separate command, no other machine. Click
   "Generate admin password" the same way to enable the `/admin`
   dashboard.
2. **Log in to each AI provider from the `/admin` dashboard itself**
   (`http://homeassistant.local:8420/admin`, under "Log in to a
   provider"). This runs a real browser inside the add-on's container
   and streams its screen to you over the page - pick a provider and
   plan, click Start, and complete login (SSO redirects, SMS codes,
   authenticator-app TOTP all work normally) right there, then click
   "I'm logged in - save session".

   Hardware security keys and platform passkeys (Touch ID, Windows
   Hello, a FIDO2 key) are the one thing this can't do - the
   authenticator has to be physically present on whatever machine is
   actually running the browser, which here is the add-on's container,
   not your phone or laptop. For those accounts only, fall back to the
   local CLI flow: on your own machine,
   ```bash
   pip install -r requirements.txt
   playwright install chromium
   MIDDLEWARE_MASTER_KEY=<same key as above> MIDDLEWARE_DATA_DIR=./data \
     python scripts/manage_credentials.py login claude --variant pro
   ```
   then copy the resulting `./data/browser_profiles/<provider>.tar.enc`
   into the add-on's `/data/browser_profiles/` (Samba/File editor
   add-on, or `scp` if SSH is enabled) - it's already encrypted, safe
   to transfer as-is.
3. Create API keys for whatever apps will call this, either from the
   `/admin` dashboard or by execing into the add-on and running
   `python scripts/manage_apikeys.py create --name my-app --models
   claude:opus --ips 192.168.1.0/24`. Each key is bound to specific
   `provider:model` refs and, optionally, an IP allowlist.

## Using the API

Point an OpenAI SDK's `base_url` at `http://homeassistant.local:8420/api/v1`,
or call it directly:

```bash
curl http://homeassistant.local:8420/api/v1/chat/completions \
  -H "Authorization: Bearer mw_xxxxxxxx.yyyyyyyy" \
  -H "Content-Type: application/json" \
  -d '{"model": "auto", "messages": [{"role": "user", "content": "..."}]}'
```

`model: "auto"` picks among the models bound to that key based on
prompt length/content. Models that expose a reasoning-effort toggle
(Claude Extended Thinking, ChatGPT Thinking, Gemini Deep Think, DeepSeek
DeepThink, Grok Think) take an optional `thinking` field the same way.

## Admin dashboard

`http://homeassistant.local:8420/admin` - separate HTTP Basic login
(`admin_username`/`admin_password`), not a client API key, since it can
create/revoke keys and remove provider sessions. Leaving
`admin_password` empty disables it entirely rather than leaving it open.
From it: create/revoke/delete API keys, see which providers have a
stored session and log one out, browse the model registry.

## Options

| Option | Meaning |
|---|---|
| `master_key` | Encrypts everything the add-on stores. Required. |
| `log_level` | `debug`\|`info`\|`warning`\|`error`. |
| `headless` | Leave `true`. `false` is only for debugging a broken provider selector with a real display attached. |
| `global_ip_allowlist` | Comma-separated IPs/CIDRs allowed at all, before per-key checks. Empty = no extra restriction. |
| `admin_username` / `admin_password` | `/admin` dashboard login. Empty password = dashboard disabled. |
| `user_agent` | User-Agent the provider browser sends, e.g. to match a firewall/proxy rule. Empty = Chromium's default. Restart the add-on after changing it. |

## When a provider breaks

These are consumer web UIs, not stable APIs - selectors drift when a
provider changes their site. If a provider starts failing, that's
almost always it; there's nothing to configure around it from the add-on
side - it needs a source-code fix (see the project's own repo).

## Beta channel

This add-on has a sibling, "ChatRelay (Beta)", built from the
same source's `uat` branch instead of `main` - an upcoming release you
can run side by side with production, before it's promoted. Independent
install: its own slug, its own `/data`. Don't point both at the same
provider logins with the same API keys in a way you depend on - treat
the beta purely as a preview.
