# Changelog

Versioning: `x.y.z` - x = major, y = minor, z = bugfix.
Bumping y resets z to 0. Bumping x resets y and z to 0.

## [0.10.1] - 2026-10-10

### Fixed
- The HA add-on's per-arch GHCR images were published to the wrong path.
  `publish-images.yml`'s `prep` job computed `image_base` as
  `ghcr.io/<owner>/<repo>` (needed by the standalone image job), but the
  `ha-addon` job then appended `/{arch}-chatrelay` onto that same value,
  producing a nested path (`ghcr.io/a4ng3i/chatrelay/amd64-chatrelay`)
  instead of `ghcr.io/a4ng3i/amd64-chatrelay` - which is what
  `config.yaml`'s `image:` field actually pulls. Supervisor install
  failed with a 403 on the manifest HEAD request as a result. This bug
  predates the ChatRelay rename - it's been there since the 0.9.0
  per-arch image naming change. Added a separate `owner_base`
  (`ghcr.io/<owner>`, no repo segment) output for the `ha-addon` job to
  use instead.

## [0.10.0] - 2026-10-09

### Changed
- **Renamed the project to "ChatRelay"** (was "AI-API2Chat-Router").
  Updated everywhere it appeared: README, `DOCS.md`, app title
  (`app/main.py`, `app/web/admin.html`), `config.yaml`
  (name/slug/url/image: `chatrelay`), `run.sh`, `docker-compose.yml`,
  and the `publish-images.yml` workflow's image names and `a4ng3i/ha-apps`
  manifest paths (`chatrelay/`, `chatrelay-beta/`). GHCR images become
  `ghcr.io/a4ng3i/chatrelay`, `ghcr.io/a4ng3i/{arch}-chatrelay[-beta]`.
  Entries below this one describe the project under its old name(s) and
  are left as-is - a historical record, not current naming.
- The GitHub repository itself still needs a manual rename (Settings ->
  repository name, `a4ng3i/AI-API2Chat-Router` -> `a4ng3i/ChatRelay`) -
  no tool available to this session can do that part.

## [0.9.1] - 2026-10-09

### Fixed
- The source repo itself transferred accounts (`aharoncg` -> `a4ng3i`)
  between the 0.9.0 push and this one. `config.yaml`'s `image:`/`url:`,
  `docker-compose.yml`'s `image:`, the README's image/URL references,
  and the workflow's commit-message text were all still hardcoded to
  `aharoncg` - corrected to `a4ng3i` throughout (the actual GHCR push
  target in the workflow was already dynamic via `github.repository`,
  so the published images themselves were never wrong, only these
  hardcoded references pointing at them). Also rewrote the README's
  "Home Assistant add-on" section: it described adding this (private)
  repo directly to HA, which was never actually how install worked
  once the public `a4ng3i/ha-apps` manifest repo existed - now it
  correctly points at that instead.

## [0.9.0] - 2026-10-09

### Changed
- **Breaking (HA add-on identity):** HA slug renamed `ai_api2chat_router`
  -> `ai-api2chat-router` (dashes, matching this account's other add-on's
  convention); `image:` renamed from the arch-suffixed
  `ai-api2chat-router-{arch}` to the arch-prefixed
  `{arch}-ai-api2chat-router`, split into separate production/beta image
  names (`-beta` suffix) instead of separate tags on one image.

### Added
- `DOCS.md`: end-user-facing setup/usage doc for the HA add-on listing
  (separate from the dev-focused README).
- `.github/workflows/publish-images.yml` gained `publish-manifest` (main)
  and `publish-beta-manifest` (uat) jobs: push this add-on's public
  install manifest (config.yaml/DOCS.md/CHANGELOG.md) to the public
  `a4ng3i/ha-apps` install repository on every push, mirroring this
  account's existing `ha-git-sync` add-on's publishing pattern exactly -
  private source repo, public manifest-only listing, separate
  production/beta channel folders. Requires a new repo secret,
  `HA_APPS_PUSH_TOKEN` (a fine-grained PAT scoped to `a4ng3i/ha-apps`,
  Contents: Read and write) - not yet added, so these two jobs will fail
  until it is.

## [0.8.0] - 2026-09-06

### Changed
- **Breaking (deployment):** `docker-compose.yml`'s `middleware` and
  `cli` services now pull the prebuilt image
  (`ghcr.io/aharoncg/ai-api2chat-router:latest`) instead of building
  locally - `docker compose pull && docker compose up -d` is now the
  default flow. Prefer building from source? Swap the `image:` line
  back for a `build:` block (`context: .`, `dockerfile:
  docker/Dockerfile`) - documented in the README.

## [0.7.2] - 2026-09-04

### Fixed
- The 0.7.1 fix itself broke the workflow file: a comment explaining
  the lowercase issue split a literal `${{ ... }}` across two lines.
  GitHub Actions' expression parser scans the whole `run:` block's
  text regardless of `#` comments, so that half-expression was invalid
  syntax and made the entire workflow file unparseable (0 jobs,
  immediate failure, run name falling back to the file path). Reworded
  the comment to not contain a literal `${{ }}` at all.

## [0.7.1] - 2026-09-04

### Fixed
- `.github/workflows/publish-images.yml` built invalid image tags after
  the repo was renamed to `AI-API2Chat-Router` (mixed case): Docker/OCI
  image names must be lowercase, but `github.repository` preserves the
  repo's actual casing, so `ghcr.io/aharoncg/AI-API2Chat-Router` failed
  with "repository name must be lowercase". The workflow now lowercases
  it once in the `prep` job and reuses that everywhere.

## [0.7.0] - 2026-09-04

### Changed
- **Renamed the project to "AI-API2Chat-Router"** (was "AI API to Chat
  Middleware"). Updated everywhere it appeared: README, app title
  (`app/main.py`, `app/web/admin.html`), the Home Assistant add-on
  (`config.yaml` name/slug/url/image, `run.sh` log messages), and
  `docker-compose.yml`'s image/container names. New slugs:
  `ai-api2chat-router` (repo, Docker image, standalone GHCR package)
  and `ai_api2chat_router` (HA add-on slug); the HA per-arch GHCR
  packages become `ai-api2chat-router-{amd64,aarch64}`.
  Entries below this one describe the project under its old name and
  are left as-is - they're a historical record, not current naming.
- The GitHub repository itself still needs a manual rename (Settings ->
  repository name) - no tool available to this session could do that
  part. Old GHCR packages published under the previous name
  (`ai-api-to-chat-middleware*`) are now orphaned leftovers; delete
  them once the new ones are confirmed working.

## [0.6.0] - 2026-09-03

### Added
- `.github/workflows/publish-images.yml`: publishes two distinct
  outputs to GHCR on every push to `uat`/`main` - a multi-arch
  standalone image (`ghcr.io/aharoncg/ai-api-to-chat-middleware`) and
  one Home Assistant add-on image per architecture
  (`ghcr.io/aharoncg/ai-api-to-chat-middleware-{amd64,aarch64}`, since
  Supervisor's `image:` templating needs literal per-arch tags, not a
  manifest list). Every build is tagged with the exact `VERSION` file
  content; `uat` pushes also get `edge`, `main` pushes also get
  `latest`.
- `config.yaml` now sets `image:` to the prebuilt per-arch HA image, so
  Supervisor pulls instead of building the (Chromium-based) image
  on-device.

## [0.5.0] - 2026-09-03

### Changed
- **Breaking:** the client-facing API moved from `/v1/*` to `/api/v1/*`
  (e.g. `/v1/chat/completions` -> `/api/v1/chat/completions`). Point an
  OpenAI SDK's `base_url` at `.../api/v1`.

### Added
- Admin dashboard at `/admin`: create/revoke/delete API keys, view and
  log out provider sessions, browse the model registry - a small HTML
  page (`app/web/admin.html`) backed by JSON endpoints
  (`app/api/admin.py`). Gated by its own HTTP Basic login
  (`MIDDLEWARE_ADMIN_USERNAME`/`MIDDLEWARE_ADMIN_PASSWORD`), separate
  from client API keys; disabled (503) until a password is set.
  Available in both the Docker and Home Assistant add-on builds.

### Fixed
- `CredentialStore` and `ApiKeyStore` (both module-level singletons)
  cached their encrypted file's path at construction instead of
  reading `settings` fresh each time, so it could never reflect a
  `MIDDLEWARE_DATA_DIR` change after first import - harmless with a
  real env var (set once at process start) but broke test isolation
  and any other reconfigure-after-import scenario. Both now resolve
  the path on every access.

## [0.4.0] - 2026-09-03

### Added
- Perplexity (perplexity.ai) as a sixth provider: `perplexity:sonar` and
  `perplexity:sonar-pro` (with a reasoning-mode thinking level).
- Root `docker-compose.yml` (replacing `docker/docker-compose.yml`), with
  a `cli` service for one-off `manage_apikeys.py` runs and a tmpfs mount
  for decrypted browser-profile material.
- `.env` (gitignored) with a freshly generated `MIDDLEWARE_MASTER_KEY`,
  ready to use with the new compose file.

### Changed
- Provider sessions are now stored as full, encrypted browser profiles
  (`app.storage.browser_profile`, tar+Fernet) instead of just cookies/
  localStorage - needed for SSO "remember this device" trust and
  WebAuthn/passkey state to actually survive across restarts. Decrypted
  copies live only in `MIDDLEWARE_RUNTIME_DIR` (tmpfs in Docker) for as
  long as a provider is in use, and are re-encrypted after every request.
- `manage_credentials.py login` now documents and is built around running
  locally (not inside the server container) so a real display is
  available for SSO redirects and hardware-key/passkey MFA; the encrypted
  profile is what moves to a remote/headless host, not a live session.
  `credentials.py` now only stores account metadata (variant/updated_at).

## [0.3.0] - 2026-09-03

### Changed
- Removed subscription-tier gating from the model registry entirely:
  providers generally expose the same model lineup to every plan (usage
  limits differ, not model access), so `ModelInfo.min_tier` is gone.
  `claude:opus-1m-context` is folded back into `claude:opus`.

### Added
- "Thinking power" selection: models that expose a reasoning-effort /
  extended-thinking toggle (Claude, ChatGPT's Thinking model, Gemini
  Deep Think, DeepSeek DeepThink, Grok Think) can now be driven via a
  `thinking` field on `/v1/chat/completions`, with `"auto"` (default)
  picking a level from the same prompt-complexity heuristic used for
  model auto-selection. `GET /v1/models` reports each model's
  `thinking_levels`.

## [0.2.1] - 2026-09-03

### Fixed
- Claude model tiering corrected: Opus is reachable starting on Pro (not
  Max-only), and Sonnet is available on Free. Renamed the model registry's
  `variant` field to `min_tier` to make this "lowest tier that unlocks it"
  semantics explicit, and renamed the `claude:opus-enterprise` ref to
  `claude:opus-1m-context` since Enterprise/Team doesn't gate Opus itself,
  just its larger context option.

## [0.2.0] - 2026-09-03

### Added
- Claude (claude.ai) adapter, covering Free, Pro, Max and Enterprise/Team
  tiers via the `claude:*` model refs (`haiku`, `sonnet`, `opus`,
  `opus-enterprise`).

## [0.1.0] - 2026-09-03

Initial UAT build.

### Added
- FastAPI app exposing OpenAI-compatible `/v1/chat/completions` and `/v1/models`.
- Browser-session adapter framework (Playwright) with initial adapters for
  ChatGPT, Gemini, DeepSeek and Grok (free/pro variants).
- Encrypted (Fernet) at-rest storage for provider session credentials and
  for API key records.
- API key management: per-key model binding, per-key IP allowlist, active
  flag, plus an optional global IP allowlist.
- Manual model selection and a heuristic smart model auto-router based on
  prompt length and content signals.
- Structured JSON activity logging (rotating file) covering auth, routing
  decisions and provider calls, without logging secrets or full message content.
- CLI scripts for provider login/credential management and API key management.
- Docker image + docker-compose for standalone deployment.
- Home Assistant add-on packaging (config.yaml, Dockerfile, run.sh).
