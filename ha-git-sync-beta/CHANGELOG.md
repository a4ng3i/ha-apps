# Changelog

## 1.17.0 (2026-10-08)

**New: configurable backup retention, and a configurable reload policy for
domain reloads (Automations/Scripts/Scenes/Dashboards).**

- **Backups.** Every pull, conflict resolution, or Full Sync now backs up
  the files it's about to change into one dated, timestamped `.tar.gz`
  archive per operation (e.g. `pull-20260101T120000Z.tar.gz`) under the
  add-on's shared `/backup/ha-git-sync/` folder, instead of one loose
  file copy per changed file. **Settings -> Backups** lets you set how
  many of these archives to keep -- the oldest is deleted first after
  each new one, or set it to 0 to never auto-delete (the previous,
  unlimited behavior). This only ever prunes old backup archives; it
  never touches `/config`.
- **Reload policy.** Previously, a domain reload (Automations, Scripts,
  Scenes, Dashboards) triggered by a pull or Full Sync always waited for
  a manual "Reload now" click on the Status page -- no exceptions.
  **Settings -> Reload policy** now lets you choose: **Manual** (that
  same behavior, still the default), **Automatic** (reload immediately,
  no approval step), **Scheduled** (reload immediately during a quiet-
  hours window you configure, otherwise queued and picked up
  automatically once quiet hours arrive), or **Selective** (automatic
  for specific domains you pick, manual for the rest). A full Home
  Assistant restart is unaffected by this setting either way -- it's
  never auto-applied by any reload policy mode, and still just shows a
  one-click "restart needed" notice.

36 new tests (backup archive creation/retention/pruning, the reload
policy's four modes and quiet-hours logic, the new Settings routes, and
the background sweep job that applies a queued "Scheduled"-mode reload
once quiet hours arrive). Full suite (300 tests) passes.

## 1.16.1 (2026-10-07)

**Security/correctness fix: checking a single file in Settings -> Sync
scope's picker (Selective mode) could silently also include a
different, unrelated file elsewhere in `/config` that happens to share
the same name.**

- Checking a root-level file like `configuration.yaml` saved the include
  pattern as the bare name `configuration.yaml` with no leading `/`. In
  gitignore/gitwildmatch syntax (what Sync scope's patterns use), a
  pattern with no `/` matches that basename **at any depth**, not just
  where you picked it -- so that one checkbox also silently brought
  `zigbee2mqtt/configuration.yaml` into scope (Zigbee2MQTT's own bridge
  config file, which commonly holds the MQTT broker's username and
  password), with no indication anywhere that anything beyond the
  checked box was included. This is how an unrelated folder like
  `zigbee2mqtt/` could show up in Settings -> Sync policy's picker, with
  an inheritable/settable policy, despite never being checked in Sync
  scope at all -- the giveaway that led to finding this.
- Every pattern the picker saves is now anchored to `/config`'s root (a
  leading `/`), so it only ever matches the exact file/folder you
  checked. A hand-typed wildcard in the free-form "Additional include
  patterns" box (anything with `*`, `?`, `[`, or `!`) is unaffected --
  cross-tree matching is exactly what that box is for. Applies
  automatically to patterns saved before this update too, the next time
  they're read -- no manual re-saving needed.

6 new tests, including an end-to-end reproduction of the exact scenario
above. Full suite (264 tests) passes.

## 1.16.0 (2026-10-07)

**The "conflicts need resolution" notice now clears itself automatically
once every conflict it covers is resolved.**

- Previously, resolving every open conflict from the Conflicts tab left
  the matching notice sitting on the Notifications tab until you visited
  it and clicked "Clear notices" by hand -- easy to forget, and stale
  once there's nothing left to act on. It's now cleared automatically
  the moment the last open conflict it could refer to is resolved. A
  notice covering several files together isn't cleared until all of them
  are resolved, not just the first one. Only this specific notice is
  affected -- every other kind (restart needed, etc) is untouched.
  Default (status_page) notification mode only; "Home Assistant
  notification" and "Other" modes don't keep a list here to clear.

6 new tests. Full suite (258 tests) passes.

## 1.15.4 (2026-10-07)

**Safety fix: Initial import can no longer delete files from /config,
even via "GitHub wins for all" against an empty or partial repo.**

- The onboarding Initial Import screen's "GitHub wins" resolution, for a
  file that only exists in `/config` (GitHub doesn't have it yet), used
  to delete that file from `/config` -- correct in isolation, but with a
  brand-new or empty GitHub repo, *every* real file in `/config` is
  "config-only," so a single click on the "GitHub wins for all" bulk
  button could wipe your entire live configuration in one step. This is
  the exact data-loss failure mode this add-on exists to prevent in the
  old Git Pull add-on, reintroduced in a different screen. A file GitHub
  doesn't have is now never deleted here, no matter what resolution is
  picked (per-file or bulk) -- it's simply left untouched in `/config`
  and picked up normally by a later Push. Enforced server-side
  regardless of what the form submits; the per-file picker also no
  longer offers "GitHub wins" at all for a file in this situation, and
  the screen now states the guarantee up front.
- Renamed the "/config wins" label to **"HA wins"** throughout the UI
  (Initial import, Settings -> Sync scope, Full Sync, Compare, Status,
  the Push conflict screen) and in the two places it becomes a real git
  commit message in your repo's history -- matching the "HA wins" /
  "GitHub wins" naming Sync policy already used, instead of a second,
  differently-worded label for the same local side. Internal
  values/URLs (`config_wins`, stored history rows) are unchanged; this
  is a display-only rename.

2 new regression tests covering both the bulk-button and a tampered
per-file form submission. Full suite (252 tests) passes.

## 1.15.3 (2026-10-07)

**Updated GitHub references after the repo owner's account rename.**

- `config.yaml`'s `image:` field now points at
  `ghcr.io/a4ng3i/{arch}-ha-git-sync`. The build workflow already
  publishes new images under the account's current name automatically,
  so leaving this field on the previous name would have meant Supervisor
  kept pulling a namespace that stopped receiving new version tags --
  existing installs would look for every future update and never find
  one.
- Repository/source links in `repository.yaml`, `README.md`, and the
  add-on's OCI image label now point at `github.com/a4ng3i/...` instead
  of the account's previous username.

## 1.15.2 (2026-10-05)

**Settings -> Sync policy's picker now only lists files/folders actually
inside Sync scope, instead of everything under /config.**

- The sync-policy picker was listing every file and folder regardless of
  whether it could ever sync -- including files excluded as secrets, and
  (in Selective scope mode) files not covered by any include pattern.
  Setting a policy for something that never syncs is a no-op and just
  added noise. Out-of-scope entries are now left out of the picker
  entirely rather than shown; a folder only appears if at least one file
  somewhere underneath it is actually in scope.

## 1.15.1 (2026-10-04)

**Two follow-up fixes to 1.15.0's per-file sync policy, found in a
post-release review.**

- A malformed sync-policy save (a mode/direction value that isn't one
  ha-git-sync itself ever sends -- a tampered form submission, not
  something reachable through the Settings UI) used to silently delete
  any existing valid override for that same file/folder instead of being
  rejected. Fixed to leave the existing override untouched, same
  tolerance this store already had for corrupted data on read.
- A scheduled, unattended sync that deferred a manual-mode file (per its
  sync policy) reused the exact notification wording for a human
  explicitly unchecking a file on the Pull confirmation page -- "held
  back **at your request** during pull review," which was simply false
  for a poll nobody reviewed anything on. The notice now names the real
  reason (the file's sync policy) when that's what actually held it back.

## 1.15.0 (2026-09-12)

**Per-file sync policy: choose whether each tracked file syncs automatically
or waits for you to review it, and which direction "automatic" means.**

- Every local edit used to auto-push to GitHub unconditionally, with no
  way to opt out -- which meant a Home Assistant-side edit could race a
  concurrent GitHub-side edit to the same file. **Settings -> Sync
  policy** now lets you set, per file or per folder, **Manual** (the
  default for every file, including on upgrade -- nothing auto-syncs
  until you say otherwise) or **Automatic** with a direction: **in**
  (GitHub wins -- only auto-pulls), **out** (HA wins -- only auto-pushes),
  or **both**. The picker is a folder tree, same interaction as Sync
  scope's: set a folder's default, or override an individual file inside
  it; anything you never touch stays on the fixed manual/both default.
- This only ever decides the no-conflict case. A file that genuinely
  changed both in Home Assistant and on GitHub since the last sync still
  always goes to **Conflicts** for manual resolution regardless of its
  policy -- automatic direction is a tiebreaker for the ordinary case,
  never a way to make one side silently win a real double-edit.
- New **Review pending changes** screen (Status page), for manual-mode
  files with local changes waiting: lists exactly what's pending, with a
  diff per file and a checkbox, so you push what you choose, when you
  choose. The existing **Push** button is unchanged and still immediately
  pushes every currently push-eligible file regardless of policy, for
  forcing a push right now without going through the review list.
- Because manual is now the default for every file, installs updating
  from an earlier version will see local auto-push stop until files or
  folders are explicitly marked automatic in Settings -> Sync policy --
  a deliberate, safety-first change, not a bug.

## 1.14.0 (2026-09-02)

**Installs and updates now pull the pre-built image from GHCR instead of
building the `Dockerfile` locally.**

- The two per-arch GHCR packages (`ghcr.io/a4ng3i/{arch}-ha-git-sync`)
  that `.github/workflows/build.yml` has been publishing since 1.9.0 are
  now public, so Supervisor -- which can only ever pull anonymously -- can
  actually reach them. `config.yaml` now sets `image:` accordingly. This
  should make installs and updates noticeably faster (no more building
  the Dockerfile from scratch on your own hardware on every update); it
  doesn't change any add-on behavior otherwise.

## 1.13.0 (2026-09-02)

**Security audit follow-up: dependency updates, a git argument-injection
hardening fix, and split-file automation/script/scene reload support.**

- **Dependencies updated** to current releases: `cryptography` 44.0.0 ->
  50.0.1, `fastapi` 0.115.6 -> 0.141.1, `jinja2` 3.1.5 -> 3.1.6 (fixes a
  sandboxed-template bypass, CVE-2025-27516 -- low real-world exposure here
  since only this add-on's own static templates are ever rendered, but no
  reason to stay behind it), `python-multipart` 0.0.20 -> 0.0.32,
  `websockets` 14.1 -> 17.1. `starlette` and `h11` (previously pulled in
  transitively, unpinned) are now pinned explicitly (1.6.0 and 0.16.0) so a
  fresh build can't silently resolve a different version than what was
  tested -- h11 0.16.0 also fixes a request-smuggling issue
  (CVE-2025-43859). The add-on's build base image
  (`ghcr.io/home-assistant/{arch}-base`) is bumped from `3.19` to `3.24`,
  the latest published tag, since `3.19` is old enough that its own
  upstream package repository no longer receives security updates.
- Starlette's newer `Jinja2Templates` API (pulled in by the `fastapi`
  bump) dropped the `autoescape=` constructor argument entirely --
  `web/routes.py` now builds an explicit `jinja2.Environment(...,
  autoescape=True)` and passes that in instead, preserving the same
  unconditional (not file-extension-dependent) autoescaping this app has
  relied on since conflicts/history started rendering synced file content.
- **Git argument-injection hardening.** `repo_url` and `branch` are
  admin-supplied text (saved as-is from the onboarding form) that flow
  straight into `git clone`/`fetch`/`remote set-url`/`ls-remote` as
  positional arguments. A value starting with `-` (e.g. a branch of
  `--upload-pack=...`) could previously be parsed by git as an *option*
  instead of a literal ref/URL. Every one of those calls in `git_layer.py`
  now puts `--` immediately before the admin-supplied value, which is
  git's own standard "everything after this is positional" marker --
  closing this off without restricting what a legitimate repo URL or
  branch name can look like.
- **Split-file automations/scripts/scenes now get a reload prompt.**
  `reload_policy.py` only recognized `automations.yaml`, `scripts.yaml`,
  and `scenes.yaml` at `/config`'s root. A change to a file under
  `automations/`, `scripts/`, or `scenes/` (the equally common
  `!include_dir_merge_list` split-file layout) used to fall into the
  generic "not recognized by the reload policy" notice instead of
  offering the one-click reload those domains already support --
  everything still applied safely either way, just without the
  convenience. Fixed the same way the 1.9.1 storage-mode-dashboard gap
  was: a directory-prefix match alongside the existing exact-filename one.

## 1.12.0

**Notices (from the "just show it on the Status page" notification mode
added in 1.11.0) now live on their own Notifications tab instead of a
card on Status.**

- Adds a **Notifications** tab in the main nav, next to Conflicts --
  same badge-with-a-count treatment. The Status page no longer shows
  notices inline. Settings' wording updated to match ("Notifications
  page" instead of "Status page"); the underlying setting value is
  unchanged, so no migration needed for anyone who already chose it.

## 1.11.0

**Notifications are now opt-in and configurable, instead of always
creating a Home Assistant persistent notification.**

- Every notice (reload/restart needed, changes detected on GitHub, a
  push/pull blocked, ...) used to unconditionally create a Home Assistant
  persistent notification -- no way to turn it off. **Settings ->
  Notifications** now offers three delivery modes: **just show it on the
  Status page** (the new default -- nothing is sent to Home Assistant
  unless you opt in), **Home Assistant notification** (the original
  behavior), or **other** (call one of your own configured `notify.*`
  services -- e.g. a phone push, email, SMS gateway -- picked from a
  dropdown populated from your actual HA configuration). The Status
  page's new **Notices** card shows recent notices when the default mode
  is selected, with a **Clear notices** button.

## 1.10.0

**History now shows what actually changed in each file, not just its
name.**

- Each row's file list used to be a bare list of filenames -- no way to
  see what changed without leaving the app and checking GitHub directly.
  Every listed file is now its own expandable "View diff" showing the
  before/after content side by side, fetched on demand when you open it
  (not eagerly for every file of every row, which would be a lot of `git
  show` calls across 200 rows of history for diffs most of which nobody
  will ever look at).

## 1.9.2

**Resolving a conflict on any file not directly at `/config`'s root
(e.g. anything under `packages/` or `.storage/`) 404'd instead of
resolving.**

- "Keep HA version" / "Keep GitHub version" / "Save merged version" on
  the Conflicts page built the resolve URL by percent-encoding the
  file's `/` as `%2F` so it would fit in one route segment, but the
  ASGI server decodes `%2F` back into a literal `/` before Starlette's
  router ever sees it -- so the route's single-segment matcher never
  matched, and the request 404'd with a bare `{"detail": "Not Found"}`
  instead of resolving anything. Only files sitting directly at
  `/config`'s root (no `/` in their path) worked. Fixed by switching
  the route to Starlette's path converter, which matches `/` correctly.

## 1.9.1

**Storage-mode Lovelace dashboard changes (`.storage/lovelace.lovelace`)
now correctly trigger a restart-required notice instead of silently doing
nothing.**

- The reload policy only recognized YAML-mode dashboards (`ui-lovelace.yaml`,
  `dashboards/*.yaml`) as Lovelace changes. A pulled change to
  `.storage/lovelace.lovelace` -- the default storage-mode dashboard, which
  is what most installs actually use -- matched neither check, so it fell
  into the "unrecognized file" bucket: written to disk correctly, but with
  no reload or restart ever suggested. Storage-mode dashboards are also
  Core's own in-memory state, loaded once at startup with no live-reload
  service (unlike `automation.reload` and friends), so even a recognized
  match would have needed a restart, not a domain reload call. Both are
  fixed: any `.storage/lovelace.*` file now correctly sets
  `restart_required` with a clear reason shown on the Status page.

## 1.9.0

**Pre-change backups no longer depend on the Supervisor backup
subsystem, and Compare now shows a full diff per file.**

- Every pull/conflict-resolution/Full Sync used to take a Supervisor
  partial-backup snapshot of the whole `homeassistant` folder before
  applying anything -- and if that failed (an overloaded Supervisor, low
  disk space), the entire operation aborted for a safety net nothing in
  the app ever actually restored from programmatically anyway. Backups
  are now plain local file copies instead, taken before each file is
  changed and saved under `/backup/ha-git-sync/` (the same shared
  Backups storage location Home Assistant itself uses, so they're
  reachable outside the add-on too). The folder hierarchy mirrors
  `/config`'s, with the date and time worked into each file's own name --
  e.g. `packages/kitchen.yaml` backs up to
  `/backup/ha-git-sync/packages/kitchen.20260101T120000Z.yaml` -- so a
  folder shows every past version of a file together. No external
  service to be unavailable, so this class of false-abort should no
  longer happen.
- **Compare** now has a **View diff** per file, expanding to the full
  content on both sides side by side -- previously it only showed
  whether a file differed, only existed on GitHub, or only existed in
  `/config`, with no way to see what actually changed without leaving
  the page.

## 1.8.0

**Scheduled sync no longer silently overwrites GitHub-side edits, and
reloads are never applied without approval.** Two related changes:

- **Settings -> Sync mode** (previously an onboarding-only, one-time
  choice with no way to revisit it) can now be changed anytime. Scheduled
  checking has a new sub-choice for what happens when it finds changes on
  GitHub: **just notify me** (new default -- detects changes and sends a
  notification, never touches `/config`; you review and apply from the
  Pull page) or **apply automatically** (the original behavior, still
  available if you want it). Full Sync was already, and remains, entirely
  manual -- nothing has ever triggered one on a schedule.
- **Reloads are now queued for approval, not applied immediately.**
  Previously, the moment a pull/conflict-resolution/Full Sync applied a
  change to a file like `automations.yaml`, the matching reload service
  (`automation.reload`, etc.) was called right away with no confirmation
  -- only a full Home Assistant restart got a heads-up notification
  first. Now every reload works that way: the Status page shows a
  **"Reload needed"** banner naming exactly what changed (e.g.
  "Automations, Scripts") with a one-click **Reload now** button, and
  nothing reloads until you click it. Multiple pending changes before
  you get to it are merged into one banner, not dropped.

## 1.7.2

**Removed the typed confirmation phrase from Settings -> Files flagged by
the secret scanner.** That requirement made sense for the secret-category
toggles above it (turning one on opts an entire *class* of file into
plaintext sync, sight unseen), but the scanner allowlist is different:
each entry there is already one specific file, with its exact flagged
findings shown right next to the toggle -- typing "I UNDERSTAND THE RISK"
on top of that was friction without adding a real safety check.

"Save allow-listed files" now just needs the toggle and a click, with a
plain-language disclaimer in place of the phrase input. The secret
category toggles above it are unchanged and still require the phrase.

## 1.7.1

**Fixed: unchecking a file deep in the Selective-scope picker didn't
actually remove it from the selection.** Any picker entry that needs a
folder expand to reach (i.e. isn't at the `/config` root) is also
pre-listed as a raw line in the "Additional include patterns" textarea
below the tree, so it can be seen/edited without hunting through nested
folders. But unchecking the tree box didn't touch that textarea line --
and since the two are combined on save, the path came right back via the
textarea, making the uncheck silently revert the moment you saved.

Unchecking a tree entry now also strips the matching line from the
patterns textarea, so the removal actually sticks. Checking a box, and
every other selection, is unaffected.

## 1.7.0

**Settings -> Sync scope now has a "Save selection" button that just
saves.** Previously the only two submit buttons in that form were
"/config wins" and "GitHub wins" -- both save the new scope, but they
also immediately walk you into the Full Sync confirmation flow, which
warns about deleting files and asks for a Yes/No. There was no way to
just persist a picker change (say, adding one more file to a Selective
scope) without also being routed through a sync-confirmation screen,
which understandably made people hesitate to click through at all --
and if you don't submit the form, nothing is saved.

"Save selection" stores the scope mode and include patterns and lands
you right back on Settings with a "Scope saved." confirmation -- nothing
syncs, nothing is deleted, and the next normal pull/push (or file
watcher run) picks up the new scope on its own. The two Full Sync
buttons are unchanged, for when you do want to reconcile GitHub and
`/config` with the new scope right away.

## 1.6.2

**Fixed: a file with a lot of secret-scanner matches blew out its table
row's height** on both Settings -> Files flagged by the secret scanner
and the push-held-back confirmation page. The "Why it was flagged" cell
rendered every match (rule, redacted excerpt, line number) as one long
wrapped block of text, so a file with many hits made its row several
lines tall.

That cell now always shows a single-line summary ("N flagged patterns")
collapsed by default; click it to expand the full per-match list.

## 1.6.1

**Fixed: the Selective-scope picker showed a folder as plain unchecked
even when some of its files were included.** Each row's checked state
only ever compared its own path against the saved patterns, with no
notion of "some of what's inside this folder is selected." Picking one
file inside a folder made every ancestor folder look empty, with nothing
in the UI indicating that anything underneath had actually been
selected -- including right after reopening Settings, before expanding
anything.

Folders with a partial selection now render as a tri-state checkbox (a
dash, same convention as file-tree checkboxes elsewhere) with a "(some
items inside selected)" note, computed both from the saved selection (so
it's correct immediately on page load, even collapsed) and live as you
check or uncheck items in an expanded folder. Checking every visible file
in a folder individually still leaves it as partial rather than
promoting it to fully checked, since checking the folder itself is a
broader, different selection -- it also covers files added under it
later, which the individual files it currently contains does not.

## 1.6.0

**The Selective-scope file picker no longer lets you select files that can
never actually sync.** Files under a secret category (`.storage/`,
`secrets.yaml`, the recorder database, etc.) are always excluded, even if
you've explicitly checked them in the picker -- that exclusion is checked
first and always wins over an include selection. Previously the picker
gave no indication of this, so checking, say, specific files under
`.storage/` (a natural thing to try, since Home Assistant's own entity
registry lives there) silently saved a selection that could never sync,
with nothing explaining why it never showed up in the repo.

Entries under an excluded category now render disabled, greyed out, and
struck through in the picker, with a "(excluded as a potential secret)"
note -- so it's clear before you even click. The same rule is now
enforced as a backstop when saving the free-form "additional include
patterns" text box (which bypasses the picker's own tree): a pattern that
targets an excluded category is dropped at save time with a flash message
naming what was dropped and why, instead of being silently accepted.

## 1.5.0

**Push conflicts (a file changed both here and on GitHub) can now
actually be resolved from the UI.** Previously, Push tried a single `git
rebase` of the whole pending commit onto GitHub's branch, and if any one
file's changes conflicted, aborted the *entire* push -- including
unrelated, non-conflicting files -- with a flash saying "resolve
manually, then push again" and no way to actually do that anywhere in the
app. If the same file kept conflicting (e.g. the file watcher retrying
every debounce interval), every retry hit the identical dead end,
producing a wall of repeated `conflict` rows in History with no path
forward.

Conflicts are now detected per file, the same way Pull already does it:
a changed file is only held back if GitHub *also* changed that specific
file since the last sync, so one conflict no longer blocks anything else
in the same push. When something is held back, Push lands on a
confirmation page -- the same one the secret scanner already uses --
showing a diff (your `/config` version vs. GitHub's) for each file, with
a checkbox to push it anyway (your version wins) or leave it for later.
Nothing is ever force-pushed; the resulting commit is built directly on
top of GitHub's current branch, so it always applies as a normal
fast-forward.

Also fixed the root cause of the repeated-conflict pileup: a failed push
used to leave the local staging clone checked out on an orphaned commit
that `LAST_SYNC_REF` didn't point to, so every retry silently built on
top of the *previous* failed attempt instead of starting clean --
stacking up local commits behind the scenes on every retry of a stuck
conflict. Push now re-establishes a known-clean state at the start of
every attempt, which also self-heals this on upgrade for anyone already
stuck in that loop.

Covered by 10 new tests: per-file conflict detection, that a conflict
doesn't block unrelated safe files, force-push overriding a conflict, that
retrying a stuck conflict doesn't accumulate orphan commits, and the full
confirmation-page flow end-to-end through the real routes.

## 1.4.2

**Fixed a crash on every truly fresh install.** `GET /onboarding` checks
`has_last_sync()` on every load, right after the deploy-key and token
steps -- but the staging clone (`ensure_staging_clone()`) isn't created
until later, inside the scope/import steps. On a brand-new `/data`
volume, that directory doesn't exist yet, and `git_layer._run()` let a
raw `FileNotFoundError` escape instead of treating "no repo there yet"
like any other git failure -- crashing the onboarding page with a bare
500 immediately after entering a valid token. Root-caused and fixed at
the source: `_run()` now treats a missing `cwd` the same way it already
treats git exiting nonzero, so `check=False` callers (like `rev_parse()`)
get their usual graceful "nothing there" result instead of a crash, and
`check=True` callers get a proper `GitError` with a clear message. This
never showed up in this project's own live-device testing, since that
device's staging clone -- created the first time it ever synced -- has
persisted in `/data` through every re-onboarding since; only a device
that has genuinely never run this add-on before hits the code path that
was broken. Reproduced against the real app end-to-end (deploy key ->
repo URL -> token, the exact sequence that crashed) before and after the
fix, and covered by 4 new tests (two at the `git_layer` level, two
end-to-end through the real onboarding routes).

**Corrected the onboarding copy describing how the HA token is
encrypted.** It claimed the encryption key is "derived from this add-on's
own Supervisor token" -- that approach was deliberately abandoned early
in this project (it broke on every update) in favor of a key generated
once and stored locally in `/data`, which is what the code has actually
done ever since. Copy now matches reality, in both the onboarding page
and DOCS.md.

## 1.4.1

Four fixes from a full-codebase security/correctness review:

**Onboarding's initial "/config wins" import now runs the secret
scanner** before pushing, same as every other push path (push(), Full
Sync). It used to skip this entirely -- a credential the ignore-category
list doesn't already cover (e.g. a hardcoded token in a template sensor)
went straight into the plaintext GitHub repo with no warning, at the
single highest-risk moment: the first sync of a never-before-reviewed
`/config`. Held-back files aren't pushed that round; they still get
picked up by a normal Push afterward, or can be reviewed/allow-listed
from Settings. `secret_scanner.scan_bytes()` is now the single shared
decode+scan+format implementation, replacing three separate copies of
the same logic.

**Fixed a real bypass of the secret-category confirmation gate.** The
check that blocks the "extra ignore patterns" box from re-including a
secret category (`.storage/`, `secrets.yaml`, etc.) used `str.rstrip("*")`
to strip a glob wildcard -- which only strips a *trailing* `*`, a no-op
for patterns like `*.db` or `esphome/*.bin`. A pattern like
`!home-assistant_v2.db` sailed through unrejected and silently
re-included the recorder database with no "I UNDERSTAND THE RISK"
confirmation at all. Fixed by asking `pathspec` itself whether a
candidate pattern changes the match result for a concrete example,
instead of re-implementing gitwildmatch's matching rules by hand.

**Fixed git renames being silently mishandled on Pull.** `git diff
--name-status` emits a rename (git's default rename detection, not an
edge case) as 3 tab-separated fields -- old path and new path -- but only
the new path was kept. A file renamed on GitHub was added under its new
name while the old one was left behind in `/config` forever, then
resurrected as a spurious "new" file on the next Push. Renames are now
expanded into an explicit delete of the old path plus an add of the new
one.

**The file watcher and scheduled-pull job now (re)start the moment
onboarding finishes**, not just once at container boot. They used to
only ever start in `lifespan()` based on onboarding state read at
process startup -- for a fresh install that's necessarily *before*
onboarding can possibly be done, so completing onboarding through the
running web UI never started them; automatic push-on-save and scheduled
pulls silently did nothing until the container happened to restart for
an unrelated reason. This logic now lives in a new `background.py`
module with an `ensure_running()` that's safe to call repeatedly, called
both at startup and right after the initial import completes. Also
retired the "Webhook" sync mode option (onboarding, the add-on's
Configuration-tab schema, and translations) -- it had no receiving
endpoint anywhere in the app and was permanently non-functional; only
Manual and Scheduled polling are offered now. `sync_mode` is also now
validated server-side against the two real values instead of accepting
any string.

Covered by 17 new tests across every layer touched: the scanner-bypass
regression, git rename handling (both at the git_layer level and
end-to-end through a real pull()), the onboarding-import scan (blocked,
clean, and already-allow-listed cases, plus that it triggers the
background jobs), and `background.ensure_running()`'s start/idempotency/
polling-mode behavior.

## 1.4.0

**Pull now shows a confirmation page before applying anything.**
Clicking Pull lands on a new page listing every file GitHub has changed
since the last sync, each with a per-file checkbox (checked by default)
and an expandable diff (current `/config` content vs. the incoming
GitHub content). Uncheck any file you don't want applied right now and
click "OK, pull checked files" -- unchecked files aren't discarded, they
show up in **Conflicts** with the same diff, resolvable there (keep
yours, take GitHub's, or hand-edit) just like a genuine both-sides-changed
conflict. A file that's already changed locally is shown read-only and
always goes to Conflicts regardless of its checkbox, since Pull never
silently picks a side for those. Scheduled/webhook pulls are unaffected
-- this confirmation only applies to the manual Pull button, since nobody
would be there to review it otherwise.

**Fixed a pre-existing bug found while building the above:** the
Conflicts page's three-way diff could silently show the wrong "Base" --
collapsed to match "Theirs" (GitHub's new content) instead of the true
common ancestor. Root cause: Base was computed as
`merge_base(LAST_SYNC_REF, pending_ref)` at *view* time, but pull()
always advances LAST_SYNC_REF to pending_ref in the same call that
creates the conflict, so by the time anyone views the page,
`merge_base(X, X)` is just `X`. Fixed by having pull() capture
LAST_SYNC_REF's value from *before* that advancement into a new
`base_ref` column on the conflict, at detection time, and using that
directly instead of recomputing a now-meaningless merge-base later.
Conflicts already open before this update fall back to the old
best-effort computation (unaffected, since nothing about existing rows
changed). Reproduced against a real running server with a genuine
conflict and verified visually before and after.

Covered by new tests at every layer: `plan_pull()`'s read-only preview,
`pull()`'s per-file selection semantics, the confirmation and apply
routes end-to-end, and the conflicts `base_ref` fix specifically.

## 1.3.0

**Added a permanent, per-file allow-list for the secret scanner in
Settings.** Until now, a file the content scanner flagged (like the
ESPHome device configs from 1.2.0) could only be pushed past on a
push-by-push basis via "push anyway" -- there was no way to say "this
file is fine, stop asking." Settings now has a new **Files flagged by
the secret scanner** card, right under the existing category-based
Secret files list: it proactively scans every file currently eligible to
sync (not just ones in a pending push) and lists each match with an
on/off toggle, using the same typed-confirmation-phrase requirement
("I UNDERSTAND THE RISK") as turning on a secret category, since it's the
same kind of decision -- letting something that looks like a credential
sync in plaintext. A file allow-listed here stays exempt from being
blocked in both Push and Full Sync until switched back off; "push
anyway" is unchanged and still covers the one-off case.

Covered by module-level tests (scan/allowlist/confirmation-phrase
semantics), an engine-level test (push() honoring the persisted
allowlist with no per-call override), and route-level tests (the
Settings page lists flagged files and reflects toggle state after
save).

## 1.2.1

**Fixed the whole UI rendering completely unstyled through ingress** --
every page loaded, but `/static/style.css` 404'd on every single request
that arrived via Home Assistant's ingress proxy (i.e. every real-world
request). Root cause: `IngressPathMiddleware` stashed the ingress prefix
(from `X-Ingress-Path`) as the ASGI `root_path`, which Starlette's own
route matching also reads and assumes is still a literal prefix of the
request path -- but Supervisor has already stripped that prefix before
forwarding to the container, so the assumption didn't hold. That mismatch
is a no-op for plain top-level routes (nothing to strip either way,
so pages rendered fine), but corrupts the path arithmetic for nested
`Mount`s -- the only one in this app being the `/static` StaticFiles
mount -- causing it to look for the stylesheet inside a directory that's
already `.../static`, i.e. a nonexistent `.../static/static/style.css`.
Fixed by keeping the ingress prefix out of the reserved `root_path` key
entirely; it now lives on `request.state` instead, read only by this
app's own link-building helpers and never seen by Starlette's routing.
Reproduced against the real `app.main.app` (middleware + static mount
included, not just the router, which is why no earlier test caught this)
and added as a permanent regression test.

Also made the one remaining hardcoded-light-only color (the warning
banner) theme-aware, so it no longer renders as a jarring bright-yellow
strip on a dark page. Automatic dark mode itself was already implemented
(follows the OS/browser's `prefers-color-scheme`, same as the rest of
the UI) -- it just had never been visible before, since the 404 above
meant no stylesheet, dark or light, was ever actually loading.

## 1.2.0

**Added a "push anyway" option for files the secret scanner blocks.**
Real-world case that prompted this: ~28 ESPHome device config files
(embedded OTA passwords / API encryption keys -- legitimate flags, not
false positives) were held back on every push with no way to get them
into GitHub short of disabling the scanner's protection for everything.
Now, when Push holds files back, it lands on a confirmation page listing
exactly which files and why; checking specific ones and clicking "Push
checked files anyway" re-runs the push with just those files exempted
for that push only -- the scanner still runs normally on every push after
that. Covered by both an engine-level test (`allow_secrets` actually
bypasses the scanner for the named file) and route-level tests (the
confirm page renders on both full and partial blocks, checked files
push, unchecked files stay held back).

**Simplified Full Sync's confirmation to a plain Yes/No.** It previously
required typing `FULL SYNC` exactly before anything applied. The file
list (what gets written vs. deleted) is still shown in full first --
only the confirmation mechanism changed, from a typed phrase to two
buttons.

## 1.1.3

**Fixed Push crashing with a raw Internal Server Error** whenever the
staging clone had an untracked directory sitting in its working tree at
the same time a commit attempt turned out to be a no-op -- e.g. an
`esphome/` folder whose device configs permanently trip the secret
scanner (ESPHome's native API encryption key is exactly the kind of
high-entropy string it looks for), so nothing under it has ever
actually been committed. git reports that case as "nothing added to
commit but untracked files present" -- a different phrasing from the
plain "nothing to commit, working tree clean" this code already treated
as a harmless no-op, so it fell through to a hard failure instead of
being recognized as the same "nothing to actually do" outcome.

Reproduced exactly from a real device log (same file/directory names,
same git message) and fixed in `stage_and_commit()` to recognize both
phrasings. Verified directly against a real git repo in that exact
state, and added as a permanent regression test.

If your file-watcher logs showed repeated "watcher-triggered push
failed" entries mentioning an untracked directory, this was why.

## 1.1.2

**Found and fixed the real bug behind "selective scope only saves"** --
1.1.1's fix was real but incomplete. Root-caused this time with a live
headless-browser reproduction (not just a static render check): the
"loading feedback" JS added a while back (button text -> "Working...",
disables the button) was disabling the clicked submit button
**synchronously inside the `submit` event handler**. Browsers build the
submitted form data from the still-enabled controls only *after* every
submit listener finishes -- so by the time that happened, the button was
already disabled, and its own `name=value` pair (e.g.
`direction=config_wins`) silently never reached the server at all.

This didn't just affect Settings' Sync scope buttons. It hit **every
button whose own value carries meaning**, with three different
failure modes depending on the route:
- Settings' "/config wins" / "GitHub wins": rejected with a "pick which
  side should win" flash, same as what turned up in the recording --
  visibly wrong, not silently wrong.
- Onboarding's "/config wins for all" / "GitHub wins for all": the
  missing field fell through to a per-file default of "/config wins"
  regardless of which button was actually clicked -- silently applied
  the wrong side.
- Conflicts' "Keep HA version" / "Keep GitHub version": that field is
  required server-side, so a missing value surfaced as a raw error page
  rather than resolving anything -- annoying, but at least not silent.

Fixed by deferring the disable to the next tick (`setTimeout(fn, 0)`),
so the browser reads the button while it's still enabled and only
disables it after. Verified with a headless-browser reproduction of the
exact failing request, confirming `direction=config_wins` now reaches
the server, before and after the fix.

If onboarding's "wins for all" buttons ever seemed to import everything
as "/config wins" no matter which one you clicked, or Conflicts' Keep
buttons ever threw a raw error instead of resolving, this was why.

## 1.1.1

**Fixed the actual "selective scope doesn't work" bug**, found from a
screen recording: the file/folder picker was showing (and letting you
check boxes in) the Selective panel even while **"Sync everything except
secrets" was still the selected option** -- the panel's show/hide relied
on a CSS rule (`:checked ~ .scope-selective-panel`) that, while correct
in isolation (verified in a clean browser test), apparently wasn't taking
effect on the real device, most likely a stale cached stylesheet rather
than a logic bug. The practical effect: checking files in the always-open
picker felt like it should do something, but since the "Selective" radio
itself was never actually selected, submitting saved nothing but the
unchanged "full" mode -- "only saves," as reported.

The show/hide is now driven by inline JavaScript (part of the freshly
served page itself, not the separately cached stylesheet), which can't
go stale independently of the radios it reads. Verified with a headless
browser, including with the stylesheet deliberately blocked entirely, to
confirm the panel still hides/shows correctly either way.

If you tried Selective mode before this update, check Settings -> Sync
scope again -- it likely reverted to "Sync everything except secrets"
without saving your file selection.

## 1.1.0

The selective-scope picker (onboarding + Settings) could previously only
select whole top-level folders or files, with anything deeper needing a
hand-typed gitignore-style pattern. It's now an **expandable tree**:
click a folder to lazily load and browse its contents, and check
individual files or folders at any depth -- checking a folder still
includes everything under it. The free-text patterns box is still there
alongside it for wildcard patterns that don't map to a literal
file/folder pick.

## 1.0.0

Added **sync scope**: a choice, made during onboarding and changeable
later from Settings, between:

- **Sync everything except secrets** (the original, still-default
  behavior) -- everything in `/config` syncs automatically except the
  excluded-by-default secret categories.
- **Selective** -- nothing syncs at all unless you explicitly include
  it (specific files, whole folders, or gitignore-style patterns).
  Secret exclusions still apply on top of this, unconditionally -- an
  included folder that happens to contain `secrets.yaml` still never
  syncs it.

Changing scope later (Settings -> Sync scope) doesn't touch anything by
itself: you pick the new scope, pick which side should win, and land
on the same confirmation page Full Sync already uses (0.10.0) --
nothing applies until you type `FULL SYNC`. Choosing "/config wins"
after narrowing scope is what actually prunes GitHub down to match
(cleanly generalizing what 0.9.0/0.10.0 already did for a single
newly-excluded category); "GitHub wins" never deletes an
excluded-category file from live `/config` just because GitHub doesn't
have it.

Existing installs are unaffected: onboarding is only shown once, so
already-onboarded installs skip straight past the new step and keep
their current "sync everything except secrets" behavior exactly as
before -- selective mode is opt-in, reachable from Settings.

Bumped 0.10.0 -> 1.0.0 (breaking change to onboarding/data format: a
new required step for fresh installs, plus new on-disk state files).

## 0.10.0

Added **Full sync**, reachable from the Compare page: unlike Pull/Push
(which only ever act on the delta since the last sync), Full sync makes
one side match the other *exactly* right now, including deleting files
the losing side doesn't have. This is what actually cleans up drift
like the `.cache/` files added in 0.9.0 -- a normal Push doesn't remove
already-tracked files just because a category got excluded afterward,
since it never re-examines files outside the current delta.

Two directions, each behind its own confirmation page requiring you to
type `FULL SYNC` before anything happens, with the exact file list
(what gets written vs. deleted) shown first:

- **/config &rarr; GitHub**: GitHub ends up matching `/config` exactly.
  Deletes from GitHub (a mirror, so low-risk) -- including files that
  are still tracked there but now excluded by an ignore rule.
- **GitHub &rarr; /config**: `/config` ends up matching GitHub exactly.
  Takes a Supervisor backup and runs `check_config` first, same as a
  normal Pull, and reverts automatically if validation fails. This
  direction never deletes an excluded-category file (`secrets.yaml`,
  `.storage/`, etc.) from `/config` just because GitHub doesn't have
  it -- GitHub was never supposed to have it in the first place.

Both directions still go through the secret scanner (push side) and
check_config + backup + rollback (pull side) -- Full sync skips the
incremental-delta shortcut, not the existing safety checks.

## 0.9.0

Added `.cache/` to the excluded-by-default list on the Settings page,
next to `.storage/`, `.cloud/`, etc. It's build/dependency cache data,
not secrets, but it churns constantly and isn't meaningful to keep in
git history -- same reasoning as the existing `tts/` and `esphome/*.bin`
entries. Like every entry in that list, it's off by default and can be
turned on from Settings if you actually want it synced.

Note: this only affects future Push/Compare runs. If `.cache/` content
was already pushed to GitHub before this update, those files stay in
the repo until removed manually or in a future push cycle that account
for deletions -- this change doesn't retroactively rewrite history.

## 0.8.0

Added a **Files** line to the Status page: total files currently in
`/config`, how many of those are excluded as potential secrets, and how
many are actually tracked in GitHub as of the last sync. Pull/Push/
Compare only ever report what changed or currently differs, which made
their small counts (tens or low hundreds) look alarming next to a
`/config` that can easily hold several thousand files (HACS-installed
integrations especially) -- this makes the full picture visible without
needing to dig through the filesystem separately to sanity-check it.

## 0.7.1

**Fixed:** when Push found files that looked like they contain secrets,
it silently skipped them and still reported plain "success" -- no flash
message, no distinct entry in History, nothing. The only symptom was the
"Files" count on a push being smaller than expected, with no indication
why. This is almost certainly the cause of the "Files" count on Compare
(which shows *everything* that currently differs) running well ahead of
the "Files" count on Push (which only counted files it actually
committed) after a push that looked successful.

Push now records these as a `partial` result (shown in History like
pulls with held-back conflicts already were), sends a Home Assistant
persistent notification naming the file(s), and flashes a message on
the Status page right after clicking Push -- for both "some files
blocked" and "every changed file was blocked" cases. If a flagged file
is a false positive, it can be opted into sync from Settings.

## 0.7.0

Moved **Compare** off the top tab bar and onto the Status page as a
third button next to Pull/Push (like the other action buttons), since
it's an action from there rather than its own top-level section.

The Status page no longer goes stale while you're sitting on it. If you
load (or stay on) the page while a pull/push/compare is running, it now
polls in the background and refreshes itself automatically the moment
that operation finishes -- previously the "Busy: ..." state and
disabled buttons stuck around until you manually left and came back.
The top progress bar also now shows for plain page navigation (like
opening Compare), not just for Pull/Push, since a slow page load used
to give no feedback at all beyond the browser's own barely-visible tab
spinner.

## 0.6.0

Iterated on the UI based on how other real Home Assistant add-ons
(Matter Server, the Alarm panel add-on) structure their ingress panels:

- Replaced the left sidebar/drawer navigation with a horizontal tab bar
  under the top app bar (Status / Compare / Conflicts / History /
  Settings) -- a better fit for five flat pages than a full drawer.
- The top app bar now shows the add-on's version (e.g. `v0.6.0`), like
  several other add-ons' panels do, so it's obvious at a glance whether
  an update has actually taken effect.
- The secret-file sync toggle on the Settings page is now a proper
  on/off switch instead of a bare checkbox.

## 0.5.1

**Fixed the bug where onboarding (repo URL, branch, and your saved HA
token) reset back to step 1 after every single update.** The encryption
key for that saved state was derived from `SUPERVISOR_TOKEN`, which the
Supervisor reissues whenever this add-on's container is recreated -- and
an update always recreates the container, not just a reinstall. Every
update therefore silently rotated the key, made the existing encrypted
file undecryptable, and onboarding looked wiped even though nothing was
actually deleted from disk. The key is now generated once and stored
locally in `/data` (the same place the SSH deploy key already lives),
so it survives ordinary updates.

**One-time consequence of this fix:** because the *old* encrypted file
can no longer be decrypted with the new key either, you'll need to go
through onboarding **one more time** after updating to 0.5.1 -- your
existing deploy key is untouched and already registered on GitHub (you
won't need to add a new one), you'll just need to re-enter the repo
URL/branch and your HA long-lived token. After that, updates will no
longer reset anything.

Also fixed the redesigned UI (0.5.0) sometimes rendering with no
styling at all after updating -- browsers (including the HA companion
app's webview) were caching `/static/style.css` under a URL that never
changes, so a stale pre-redesign stylesheet stuck around instead of
picking up the new one. The stylesheet link now carries a version tag
that changes on every update.

## 0.5.0

New **Compare** page: a read-only, on-demand diff between GitHub's current
branch and this Home Assistant's live `/config` -- lists files that only
exist on one side or differ in content, without touching anything. Useful
to sanity-check before clicking Pull or Push, or just to see where things
stand. Nothing is written to disk or committed when you view it.

The Status page now shows the date/time and direction of the last
successful sync (e.g. "2026-08-02T14:03:11+00:00 -- pulled GitHub ->
/config"), not just the last synced commit hash. Pull/Push (and every
other form submission) now shows immediate feedback -- the button
disables and relabels itself, and a top progress bar appears -- instead
of an unresponsive-looking wait while the request is in flight.

Redesigned the web UI's chrome to match Home Assistant's own frontend: a
top app bar, a left sidebar with icons (collapsing to a slide-out drawer
on narrow/mobile screens), and Home Assistant's theme colors/typography
for light and dark mode. Purely visual/structural -- no page's
functionality changed.

## 0.4.0

Clarified the Status page's Pull/Push buttons, which just said "Pull now"
and "Push now" with no indication of direction. They now read "Pull from
GitHub -> /config" and "Push /config -> GitHub", with a short explanation
line underneath.

## 0.3.2

Fixed onboarding crashing with a raw Internal Server Error when the
target GitHub repo has zero commits (a brand-new empty repo has no
branches at all, so `git fetch origin main` fails with "couldn't find
remote ref main"). A friendly message explaining this and how to fix it
already existed for this exact case, but a `fetch()` call earlier in the
same code path was raising before ever reaching it. Now fails gracefully
on both the initial-import screen and the apply step, with actionable
guidance ("add at least one file on GitHub to create the branch").

## 0.3.1

`GitError` tracebacks in the add-on Log now include the actual git error
text (e.g. the real auth/network failure reason) instead of just "git
fetch origin main failed" with no detail -- stdout/stderr were being
captured but never surfaced anywhere, so diagnosing a failed clone/fetch
required extra round-trips through the log. No behavior change beyond
that; purely a diagnostics fix.

## 0.3.0

Added a new **Home Assistant Core API URL** option (Configuration tab):
the base URL used to reach Core's REST API for check_config, reload/
restart calls, and validating the long-lived token during onboarding.
Previously this was a hardcoded two-URL fallback list, which had a bug
(see 0.2.2) and added complexity for no real benefit once a bug like that
is possible. Now it's a single, explicit, user-configurable value.

Default: `http://homeassistant:8123/api` (reaches Core directly by its
internal hostname -- confirmed working end-to-end against a real HA OS
install). Alternative: `http://supervisor/core/api` (via the Supervisor's
own reverse proxy for Core's REST API instead of hitting Core directly) --
switch to that in this add-on's Configuration tab if the default doesn't
work on your setup.

## 0.2.2

Fixed a bug in the token-validation check added in 0.2.0: it treated a
401 from the Supervisor-proxied Core path as definitive rejection and
never tried the direct path as a fallback, so a genuinely valid long-lived
token could be reported as "rejected" if the proxied path 401'd for
unrelated reasons. Now matches check_config()'s existing (correct)
fallback behavior -- only reports a token as rejected if every reachable
path rejects it.

## 0.2.1

No functional changes -- version-only bump to confirm Home Assistant's
"Update available" flow shows up correctly for this add-on.

## 0.2.0

Security hardening and a real-install bugfix round:

- App-level admin enforcement (Home Assistant's own ingress proxy doesn't
  gate this on its own -- see app/auth.py).
- Path traversal guard on every git-sourced filesystem write.
- Git transport restricted to ssh/https by default.
- Fixed a conflict-tracking bug where already-applied safe changes could
  be spuriously re-flagged as conflicts on a later pull.
- CSRF protection on all state-changing routes.
- Fixed an onboarding lockout: an invalid Home Assistant long-lived token
  used to get saved without validation, which then blocked every route
  (including onboarding itself) since admin status couldn't be verified.
  The token is now checked against Core before being saved.
- Fixed a broken-clone-retry bug in the staging clone lifecycle.
- Added repository.yaml so this repo is recognized as a valid Home
  Assistant add-on repository.

## 0.1.0

Initial scaffold: staged pulls with validate-before-apply and automatic
rollback, atomic per-file apply, human-in-the-loop conflict resolution,
secret scanner, selectable secret-category exclusions, Supervisor backup snapshots
before every applied pull, audit trail, and an Ingress web UI (onboarding
wizard, status dashboard, conflicts, history, settings).
