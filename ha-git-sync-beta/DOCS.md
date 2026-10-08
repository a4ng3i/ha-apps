# HA Git Sync

Bidirectional Git sync for `/config`, designed so that a misconfiguration
or a bad pull can never destroy your Home Assistant configuration.

## Why this exists

The old community "Git Pull" add-on has documented reports of wiping
`/config` entirely when the repo was empty or misconfigured. This add-on
never writes into `/config` directly from git. Every incoming change is
staged in an isolated clone, diffed file-by-file against what's already on
disk, validated with Home Assistant's own `check_config`, and only then
applied -- atomically, with the current content of every file about to
change backed up locally immediately beforehand (see "Backups" below). If
validation fails, every file that was written is restored from that
backup automatically.

See the top-level `README.md` in this repository and the design doc history
in this repo for the full rationale.

## Features

- **Config validation before applying.** Every incoming change -- from a
  Pull, a Full Sync, or conflict resolution -- is validated with Home
  Assistant's own `check_config` before it's considered applied. If
  validation fails, every file that was written is rolled back
  automatically; nothing is left half-applied. See "Pull",
  "Compare and Full Sync", and "Conflicts" below.
- **Automatic local backups**, archived per sync operation (not a
  Supervisor snapshot -- see "Backups" below) with configurable
  retention from **Settings -> Backups**.
- **Configurable reload policy** for domain reloads (Automations,
  Scripts, Scenes, Dashboards): Manual (default), Automatic, Scheduled
  (quiet hours), or Selective by domain -- see "Reload vs. restart"
  below. A full Home Assistant restart is never auto-applied by any of
  these; it's always a one-click notice.
- **Secrets excluded by default**, not opt-out: `secrets.yaml`,
  `.storage/`, the recorder database, and several other categories never
  sync unless individually, explicitly turned on with a typed
  confirmation phrase (**Settings -> Secret files**). On top of that, a
  pattern/entropy-based **secret scanner** blocks any push containing
  what looks like an API key or token even in a file that isn't already
  excluded, with a one-time "push anyway" override per push or a
  permanent per-file allowlist (**Settings -> Files flagged by the
  secret scanner**) for files you've already reviewed. See "What syncs,
  and what doesn't" below.
- **SSH deploy key authentication** -- the add-on generates its own
  ed25519 keypair; you add the public half to GitHub as a deploy key
  with write access. No GitHub account credentials or personal access
  token ever touch the add-on. Keys can be rotated from **Settings ->
  Deploy key rotation** without ever being blind about it (the new key
  is tested with a real push before the old one is deleted).
- **Human-in-the-loop conflict resolution** -- a file changed on both
  sides at once always goes to **Conflicts** for manual resolution; it's
  never silently decided by whichever side happened to sync first.

## Setup

1. Install the add-on, then open its Web UI (via the sidebar / Ingress).
2. **Step 1 - Deploy key.** The add-on generates an ed25519 keypair on
   first start. Copy the public key shown into your GitHub repo's
   **Settings -> Deploy keys -> Add deploy key**, with **Allow write
   access** checked (needed for push). Then paste your repo's SSH URL
   (`git@github.com:owner/repo.git`) and target branch.
3. **Step 2 - Home Assistant token.** Create a long-lived access token from
   your HA user profile page and paste it in. This is the only credential
   that can reach Core's `/api/config/core/check_config` endpoint, which is
   why the add-on needs it (the Supervisor's own token can't call it).
   It's encrypted at rest using a key generated locally and stored in this
   add-on's persistent storage -- the same trust boundary as the SSH deploy
   key.
4. **Step 3 - Sync mode.** This only controls the GitHub &rarr; here
   direction; the opposite direction is governed per-file by **Settings ->
   Sync policy** instead (see below). Full Sync is always separate and
   manual either way, no matter what's picked here. **Manual only**
   (default) means you check
   for GitHub changes yourself, from the Status page's **Pull** button.
   **Scheduled checking** checks on a timer instead, and has its own
   sub-choice for what happens when it finds something: **just notify me**
   (default -- nothing is applied, you review and apply from the Pull
   page) or **apply automatically** (the original behavior -- applies
   every safe change with no review). Changeable later from **Settings ->
   Sync mode**.
5. **Step 4 - Sync scope.** Choose **Sync everything except secrets** (the
   default -- see below) or **Selective**, where nothing syncs at all
   unless you explicitly pick it (specific files, whole folders, or
   gitignore-style patterns). Changeable later from **Settings -> Sync
   scope**; see "Changing your mind later" below for what happens then.
6. **Step 5 - Initial import.** A one-time, explicit, file-by-file (or
   bulk) comparison between what's currently in `/config` and what's in the
   GitHub repo, filtered to whatever scope you just picked. Nothing is
   written until you approve it here. This is the step that protects
   against the "empty repo silently wins" failure mode that hurt users of
   the old Git Pull add-on. A hard safety rule applies here regardless of
   what you pick: a file GitHub doesn't have yet is **never** deleted from
   `/config` -- not per-file, and not even via the "GitHub wins for all"
   bulk button. GitHub having nothing for a file isn't a vote to erase it;
   with a brand-new/empty repo, every real file would otherwise be
   "GitHub-only loses," which is exactly the old add-on's data-loss bug.
   Such files are simply left alone and picked up by a normal Push later.

## What syncs, and what doesn't

**Sync scope** controls the overall shape: **Sync everything except
secrets** (the default) syncs all of `/config` automatically; **Selective**
syncs nothing unless you've explicitly included it. Either way, one rule
never changes: files that can hold credentials or other secrets
(`secrets.yaml`, `.storage/`, `*.db*` the recorder database, `.cloud/`,
`.cache/`, `tts/`, log files, and a couple others) are **excluded by
default and never sync on their own**, even if a Selective-mode folder you
included happens to contain one. There's no "sync all secrets" switch.
Each category has to be individually selected in **Settings -> Secret
files**, and turning one on requires typing a confirmation phrase, since
doing so means that category's contents will be written into your GitHub
repo in plaintext from then on.

Selective mode's folder/file picker reflects this: entries under an
excluded category (like `.storage/`) show up greyed out and struck
through, with checkboxes disabled and a "(excluded as a potential
secret)" note, so it's clear up front that checking them wouldn't do
anything -- rather than letting you pick files that quietly never sync.
The same rule applies if you type a path under a secret category
directly into the "additional include patterns" box below the picker:
it's dropped when you save, with a message naming what was dropped and
why.

A folder in the picker shows a dash instead of an empty or checked box,
with a "(some items inside selected)" note, whenever some but not all of
its contents are included -- so picking one file out of a folder doesn't
make the folder itself look like nothing inside it was selected.
Checking every individual file you can currently see inside a folder
still leaves it showing as partial rather than fully checked, though:
checking the folder itself means "everything in here, including files
added later," which isn't the same thing as the specific files you
picked.

Settings also has a separate free-form "additional ignore patterns" box
for arbitrary non-secret exclusions (e.g. a personal notes file you don't
want tracked). It cannot be used to re-include a secret category -- that
always has to go through the confirmation-phrase flow above, so there's
exactly one path to opting a secret file into sync, not two.

A secondary pattern/entropy-based scanner also blocks pushes containing
what look like API keys or tokens in files that aren't already ignored, as
a belt-and-suspenders check -- not a replacement for the ignore list. If
it blocks something during a Push, you land on a confirmation page right
away listing exactly which file(s) and why (e.g. an ESPHome device config
with an embedded OTA password). If that's a false positive -- or you've
decided it's fine for that specific file -- check it and click "Push
checked files anyway" to push it for just that one push; nothing else
changes, and the next push still scans normally. Anything left unchecked
stays held back and also shows up in **History** (marked `partial`).

For a file you already know is fine (an expected embedded device secret,
say), you don't have to wait for a push to hit it: **Settings -> Files
flagged by the secret scanner** proactively scans everything currently
eligible to sync and lists every file that matches, each with its own
on/off toggle and its exact findings shown right next to it -- no typed
confirmation phrase required, since each entry is already one specific,
reviewed file rather than an entire category turned on blind. A file
allow-listed there is permanently exempt from being blocked, in both
Push and Full Sync, until you turn it back off -- unlike "push anyway,"
which only ever covers the push you clicked it on.

### If a file changed on GitHub too

Push only ever acts on files that changed here since the last sync -- if
one of them was *also* changed on GitHub in the meantime (someone edited
it directly, or another device pushed to it), pushing your version would
silently overwrite that change. Push holds that file back and lands you
on the same confirmation page the secret scanner uses, with a diff of
your version against GitHub's for each held-back file. Check any you want
to push anyway (your HA version wins for that file) and click
"Push checked files anyway"; anything left unchecked stays exactly as it
is on both sides -- nothing is discarded -- and you can revisit it on a
later push. Files that don't conflict are pushed immediately regardless,
so one conflicting file never holds up everything else in the same push.

The Status page shows a running **Files** line -- how many files are in
`/config`, how many are excluded as potential secrets, and how many are
actually tracked in GitHub as of the last sync -- so you can sanity-check
the totals without leaving the add-on.

### Changing your mind later

Settings -> Sync scope lets you switch between Full and Selective (or edit
Selective's include list) at any time. **Save selection** just stores the
new scope and stops there -- nothing syncs, and the next normal pull/push
(or file-watcher run) picks it up on its own.

If you'd rather apply it immediately, pick a direction instead ("HA
wins" or "GitHub wins") -- that saves the scope the same way, then
also walks you through the same confirmation page **Full Sync** uses (see
below), so nothing is actually applied until you review the exact file
list and confirm with Yes. Narrowing scope + "HA wins" is
what actually prunes GitHub down to match the new, smaller scope; "GitHub
wins" never deletes an excluded-category file from live `/config` just
because GitHub doesn't have it -- GitHub was never supposed to have it in
the first place.

## Pull

Clicking **Pull** on the Status page never applies anything immediately --
it lands on a confirmation page listing every file GitHub has changed
since the last sync, with a diff (current `/config` content vs. the
incoming GitHub content) available per file. Each file that's safe to
apply (hasn't also changed locally) has its own checkbox, checked by
default; uncheck any you don't want applied this time and click **OK,
pull checked files**, or **Cancel** to apply nothing at all. A file
already changed locally as well is shown too, but always goes to
**Conflicts** regardless of its checkbox state -- pulling never silently
picks a side for those.

Unchecking a safe file doesn't discard the incoming change: it's held
back and shows up in **Conflicts** with the same diff, where you can keep
your version, take GitHub's after all, or hand-edit a merge -- exactly
the same options as a genuine both-sides-changed conflict.

Whether a local change here pushes to GitHub on its own, and whether a
scheduled check auto-applies what it finds, is now controlled per-file by
**Settings -> Sync policy** (see below) rather than always happening
unconditionally -- a **manual**-mode file (the default) never auto-pushes
and is never auto-applied; instead it waits for you to review it. If
**Scheduled checking** (Settings -> Sync mode) is on and set to **just
notify me**, a scheduled check never touches `/config` either way; it
only looks at what's changed on GitHub and, if anything has, sends a
notice (see **Notifications** below for where that actually shows up)
naming the file(s) so you know to come review and apply them yourself --
exactly the same Pull confirmation page as a manual check. Set to
**apply automatically** instead, a scheduled check behaves like clicking
Pull and accepting every safe file whose sync policy allows automatic
GitHub &rarr; here sync, with anything that also changed locally still
going to Conflicts as usual, and every manual-mode file also deferred to
Conflicts for you to apply later rather than left unmentioned.

## Sync policy

**Settings -> Sync policy** controls, for each tracked file, whether it
syncs on its own or waits for you: **Manual** (the default for every file
until you say otherwise) means local edits never auto-push and incoming
GitHub changes never auto-apply -- they just wait for you to review them.
**Automatic** adds a direction: **in** (GitHub wins -- only auto-pulls),
**out** (HA wins -- only auto-pushes), or **both** (auto-syncs either
way). The picker is a folder tree, same as Sync scope's: set a default at
a folder level and individual files inside can override it, or leave
everything on the fixed manual/both default by never touching it. Only
files/folders currently inside **Sync scope** are listed -- a file
excluded as a secret, or (in Selective mode) not covered by an include
pattern, never syncs either way, so there's nothing to set a policy for;
a folder only appears if something underneath it is actually in scope.

This only ever decides what happens when nothing conflicts. A file that
genuinely changed both in Home Assistant and on GitHub since the last
sync always goes to **Conflicts** for manual resolution regardless of its
policy -- automatic direction is a tiebreaker for the ordinary case, not
a way to make one side silently win a real double-edit.

Manual-mode files with local changes waiting show up on the Status page's
**Review pending changes** screen -- a list of exactly what's pending,
each with a diff, and a checkbox to push it now. Nothing pushes until you
check it and click through; leaving a file unchecked just means it's
still there next time, exactly like an unapplied Pull. This is separate
from the Status page's plain **Push** button, which still immediately
pushes every currently push-eligible file regardless of policy, for when
you want to force a push right now without going through the review list.

## Compare and Full Sync

**Compare** (a button on the Status page) is a read-only, on-demand diff
between GitHub's current branch and `/config` right now -- which files
differ, and which only exist on one side. Each row has a **View diff**
that expands to the full content on both sides, side by side ("(file
doesn't exist)" / "(doesn't exist on GitHub)" for the two only-on-one-side
cases). Nothing is written or committed by viewing it.

**Full Sync**, reachable from the Compare page, is different from a normal
Pull/Push: those only ever act on the delta since the last sync, so
neither can clean up drift that falls outside that delta (e.g. a secret
category or scope you just excluded, whose files are still sitting in the
GitHub repo from before). Full Sync instead makes one side match the other
*exactly*, right now -- including **deleting** files the losing side
doesn't have. It's gated behind a confirmation page showing the precise
file list (what gets written vs. deleted) with a Yes/No confirmation
before anything happens. Both directions still go through the same secret
scanner (push side) and backup + `check_config` + automatic rollback
(pull side) that normal Pull/Push use -- Full Sync only skips the
incremental-delta shortcut, not the safety checks.

## Conflicts

If a file changed both in Home Assistant (via the UI) and on GitHub since
the last sync, ha-git-sync never guesses. It shows you a three-pane diff
(common ancestor / your version / GitHub's version) in the **Conflicts**
tab and lets you keep one side, or hand-edit a merge. Everything else that
*didn't* conflict is still applied normally.

## Reload vs. restart

Automations, scripts, scenes, and Lovelace dashboards reload live once
applied. What happens to that reload is controlled by **Settings ->
Reload policy**:

- **Manual** (the default, and the only behavior before this setting
  existed) -- nothing reloads automatically. When a pull, conflict
  resolution, or Full Sync applies a change to one of those domains, the
  Status page shows a **"Reload needed"** banner naming exactly which of
  them (e.g. "Automations, Scripts") with a one-click **Reload now**
  button; nothing reloads until you click it. If more changes land
  before you get to it, the banner just grows to cover everything still
  pending -- nothing is silently dropped.
- **Automatic** -- every reload call runs immediately, with no approval
  step. You still get a notice saying what was reloaded and why.
- **Scheduled** -- reload calls run immediately if they land inside a
  quiet-hours window you configure (e.g. 02:00-05:00); otherwise they're
  queued like Manual, and picked up automatically the moment quiet hours
  arrive (checked every few minutes in the background) rather than
  sitting queued forever if the change happened outside the window.
- **Selective** -- pick which domains (Automations, Scripts, Scenes,
  Dashboards) reload automatically; every other domain is still queued
  like Manual.

Whichever mode you pick, a full Home Assistant **restart** is handled
completely separately and is **never** auto-applied by any reload
policy mode -- restarting all of Core is a much bigger action than a
domain reload, so anything touching `configuration.yaml` itself or
`custom_components/` always just shows a "restart needed" notice, and
you restart from Home Assistant's own UI when ready.

## Notifications

ha-git-sync's own notices -- reload/restart needed, changes detected on
GitHub, a push or pull blocked, and so on -- go to exactly one of three
places, chosen in **Settings -> Notifications**:

- **Just show it on the Notifications page** (the default). Nothing is
  sent to Home Assistant at all; notices appear on ha-git-sync's own
  **Notifications** tab (with a badge showing the count, same as
  Conflicts) until you clear them. This is the default specifically so
  installing or updating the add-on never starts creating Home Assistant
  notifications you didn't ask for.
- **Home Assistant notification** -- a persistent notification in the HA
  UI, the same behavior every version before this setting existed had
  unconditionally.
- **Other** -- calls one of your own configured `notify.*` services
  (e.g. a mobile app push, email, SMS gateway -- whatever you already
  have set up in Home Assistant). The dropdown is populated from your
  actual HA configuration; if it can't be loaded (e.g. no token from
  onboarding yet), type the service name manually instead (the part
  after `notify.`, e.g. `mobile_app_phone`).

In the default mode, the "conflicts need resolution" notice clears itself
automatically once every conflict it could refer to is resolved -- no
need to visit the Notifications tab and clear it by hand. If it covered
several files in one notice, resolving just one of them isn't enough to
clear it; it stays until the last one is resolved too. Every other notice
is unaffected by resolving a conflict.

## Backups

Every applied pull, conflict resolution, or Full Sync backs up the
**current** content of each file it's about to change **before**
touching it -- one dated, timestamped `.tar.gz` archive per operation,
not a Supervisor snapshot (a previous version used a Supervisor partial
backup of the whole `homeassistant` folder here; that made an
otherwise-safe pull depend on the Supervisor backup subsystem being
healthy, and a failure there -- an overloaded Supervisor, low disk
space -- aborted the pull for a safety net nothing in the app ever
actually restored from programmatically anyway).

These live under `/backup/ha-git-sync/`, the same shared **Backups**
storage location Home Assistant itself uses (so they're reachable outside
the add-on too, e.g. from a Samba or File editor add-on if you have one).
Each archive is named after what triggered it and when -- e.g.
`/backup/ha-git-sync/pull-20260101T120000Z.tar.gz` -- and preserves
`/config`'s relative paths inside it, so extracting it (or opening it in
any archive tool) reproduces exactly the files that were about to change,
as they stood right before the change. A file that didn't exist yet
(about to be newly created) has nothing to back up and is simply skipped.

**Settings -> Backups** controls how many of these archives are kept --
the oldest is deleted first, right after each new one is created. Set it
to 0 to never auto-delete (the default before this setting existed);
otherwise pick however many recent sync events you'd want to be able to
go back to. This only prunes the archive files themselves -- it never
touches `/config`.

## Who can use this

Only Home Assistant administrators. Home Assistant's own ingress proxy
does **not** enforce this on its own -- `panel_admin: true` in this
add-on's manifest only hides the sidebar entry and blocks in-app
navigation for non-admin accounts; it does not stop a non-admin user from
reaching the add-on's HTTP endpoints directly if they know how to ask for
an ingress session (verified against the Home Assistant Core and
Supervisor source). Because of that, this add-on checks for itself: every
request is verified against Core's real admin user list, resolved over
Core's WebSocket API using the long-lived token you provide during
onboarding, and a request from anyone who isn't confirmed as an admin (or
whose admin status can't be verified at all) is rejected. The one
exception is a short bootstrap window before you've completed onboarding
step 2 (pasting the token) -- there's genuinely no way to ask Core who's
an admin before that point, so requests are allowed through with a
logged warning until a token is configured.
