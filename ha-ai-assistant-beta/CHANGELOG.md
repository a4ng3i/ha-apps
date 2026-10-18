# Changelog

Versioning: `x.y.z` - x = major, y = minor, z = bugfix.
Bumping y resets z to 0. Bumping x resets y and z to 0.

## [0.1.0] - 2026-10-17

First release.

### Added
- **Read-only collection** of everything Home Assistant knows about
  itself: Supervisor (host, OS, Core, Supervisor, every add-on, the
  resolution center, backups), recent logs of Core, Supervisor, host and
  every running add-on, Core state, entity/device/area registries,
  integrations, repairs, the structured system log, users (roles only),
  and your YAML configuration from a read-only mount. `secrets.yaml` is
  never opened; `!secret` references are recognised without reading the
  value.
- **Recorder database access** for SQLite (default, read-only mount) and
  MariaDB/MySQL/PostgreSQL through a separate read-only database user
  (`recorder_db_url`). Sessions are forced to READ ONLY and get a
  statement timeout. Without database access, history comes from Core's
  history API.
- **23 local checks** that run without AI: plain-HTTP remote access, IP
  ban, broad trusted proxies/networks, CORS wildcard, credentials written
  inline in YAML, add-ons with protection mode off or elevated
  privileges, admin account count, debug logging, Supervisor
  unhealthy/unsupported/issues, pending updates, open repairs, failed
  integrations, long-unavailable and orphaned entities, add-ons that
  should run but don't, stale or local-only backups, disk usage, large
  log file, recorder size/stall/noisy entities/retention, recurring log
  errors, low batteries, recovery mode, automations and scripts pointing
  at missing entities or devices, failed/disabled/never-triggered
  automations, duplicate names, devices without area, incomplete core
  settings.
- **House insights**: presence per person, when the house was empty,
  motion by hour and area, activity while nobody was home, doors and
  windows opened at night, lights left on while away, lock usage,
  temperature/humidity by area, energy counters, automation activity.
- **AI analysis** with Claude (model picker: Sonnet 5.5 default, Opus
  5.5, Haiku 5.5, Fable 5.1) or any OpenAI-compatible endpoint for a
  local model. Returns a health score, findings with remediation steps
  and YAML examples, a narrative of what happens in the house, and
  improvement suggestions. A model can also be picked per run from the
  panel.
- **Privacy gate**: three redaction levels (strict default: credentials,
  emails, IPs, MACs, coordinates, phone numbers, hostnames, and people's
  names replaced with pseudonyms that are mapped back only locally).
  "Privacy" tab previews the exact payload and can send exactly that
  payload; every send is logged with its SHA-256, size and token usage.
- **Panel in Home Assistant** (ingress): overview, findings with
  filters, house insights, run history, privacy/audit log, effective
  settings. Admin-only, enforced on every request.
- Daily or weekly scheduled runs and an optional persistent notification
  when critical or high severity issues are found.
