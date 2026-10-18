# HA AI Assistant

Reads your Home Assistant installation end to end (logs, configuration,
events, the recorder database, add-ons), finds bugs, misconfigurations,
security gaps and inconsistencies, and uses AI to explain how to fix
them and what is happening in your house.

Everything it reads, it reads **read-only**. The only thing it ever
writes into Home Assistant is an optional persistent notification.

## How it works

1. **Collect** (local, read-only): Supervisor API, Core WebSocket/REST
   API, the YAML files in your config folder, and the recorder database.
2. **Check** (local, no AI): 23 deterministic checks for security,
   reliability, configuration, performance and maintenance problems.
3. **Insights** (local, no AI): presence, empty-house periods, activity
   by hour and area, activity while away, night-time door openings,
   lights left on, locks, climate, energy, automation activity.
4. **AI analysis**: a compact, redacted digest of 1-3 goes to the AI,
   which finds root causes, problems the checks can't see (for example
   automation logic mistakes), writes remediation steps and YAML, and
   explains the house in plain language.

Steps 1-3 work without any AI provider configured.

## Installation

1. Add `https://github.com/a4ng3i/ha-apps` as an add-on repository
   (Settings > Add-ons > Add-on Store > three dots > Repositories).
2. Install **HA AI Assistant**.
3. Configure the AI provider (below), start the add-on, and open the
   **HA AI Assistant** panel in the sidebar.
4. Click **Run analysis now**.

## Configuration

| Option | Default | What it does |
|---|---|---|
| `ai_provider` | `anthropic` | `anthropic` (Claude), `openai_compatible` (any OpenAI-style endpoint, e.g. a local model), or `none` (local checks only). |
| `anthropic_api_key` | | API key from console.anthropic.com. |
| `anthropic_model` | `claude-sonnet-5-5` | Sonnet 5.5 is the economical default. Opus 5.5 is deeper, Haiku 5.5 cheapest, Fable 5.1 most capable. You can also pick a model per run in the panel. |
| `ai_effort` | `high` | How hard the model thinks. Lower is cheaper and faster. |
| `openai_base_url`, `openai_api_key`, `openai_model` | | For `openai_compatible`, e.g. `http://192.168.1.10:11434/v1` for Ollama. |
| `privacy_level` | `strict` | What gets redacted before anything is sent (see Privacy). |
| `schedule`, `schedule_time` | `daily`, `04:30` | Automatic runs (`off`, `daily`, `weekly` on Sunday). |
| `history_hours` | `168` | How far back history and insights look. |
| `read_recorder_db` | `true` | Read statistics from the recorder database. |
| `recorder_db_url` | | Only for MariaDB/MySQL/PostgreSQL recorders, see below. |
| `notify` | `true` | Persistent notification when a run finds critical/high issues. |

### Recorder on MariaDB, MySQL or PostgreSQL

Your recorder's own `db_url` lives in `secrets.yaml`, which this add-on
never reads. Give it a **separate, read-only** database user instead.

MariaDB add-on (run in the MariaDB add-on's console, or any client):

```sql
CREATE USER 'ha_ai_ro'@'%' IDENTIFIED BY 'a-long-random-password';
GRANT SELECT ON homeassistant.* TO 'ha_ai_ro'@'%';
FLUSH PRIVILEGES;
```

Then set `recorder_db_url` to:

```
mysql://ha_ai_ro:a-long-random-password@core-mariadb/homeassistant
```

PostgreSQL:

```sql
CREATE ROLE ha_ai_ro LOGIN PASSWORD 'a-long-random-password';
GRANT CONNECT ON DATABASE homeassistant TO ha_ai_ro;
GRANT USAGE ON SCHEMA public TO ha_ai_ro;
GRANT SELECT ON ALL TABLES IN SCHEMA public TO ha_ai_ro;
```

```
postgresql://ha_ai_ro:a-long-random-password@<host>:5432/homeassistant
```

Even with a user that has more rights, every session is switched to
READ ONLY transactions with a statement timeout. Without
`recorder_db_url`, insights come from Core's history API and the
database statistics (size, noisiest entities, event counts, energy
counters) are skipped.

## Privacy

Nothing is sent anywhere unless an AI provider is configured. What is
sent is a **digest**, not raw data: versions, counts, add-on list,
HTTP settings (no certificates), local findings, clustered log
patterns, repairs, insight numbers and your automations' triggers,
conditions and actions (size-capped).

Before sending, the digest is redacted:

- `minimal`: tokens, JWTs, API keys, passwords and secrets in text,
  credentials in URLs, long random strings.
- `balanced`: minimal + email addresses, public IPs, MAC addresses,
  coordinates, phone numbers, external hostnames.
- `strict` (default): balanced + private IPs, the time zone, and the
  names of people (from your `person` entities) replaced everywhere with
  "Person 1", "Person 2"... including inside entity ids. The AI's answer
  is mapped back to real names locally, for display only.

The **Privacy** tab shows the exact payload before you send it and can
send exactly what you reviewed. Every send is in the audit log with its
SHA-256, size, model and token usage; the last 20 payloads are kept
locally for review.

The AI is told that the digest is data, not instructions, so a log line
or device name crafted to look like an instruction is treated as
evidence only. Its answer is only displayed; it never triggers actions.

## Access this add-on has

- Config folder: mounted **read-only** (`homeassistant_config:ro`).
- Supervisor API, role `manager`: the lowest role that can read other
  add-ons' logs and the host/Core/Supervisor journals. The add-on's
  Supervisor client can only send GET requests.
- Core API through the Supervisor proxy: a fixed allowlist of read-only
  WebSocket commands, and one write, the optional persistent
  notification.
- The panel is admin-only. `panel_admin` only hides the sidebar entry,
  so the add-on checks every request against Core's list of
  administrators and fails closed.

## Costs

One run sends one request. With Sonnet 5.5 and a typical home (a few
hundred entities, 50 automations) the digest is roughly 30-80k input
tokens. Token usage per run is shown in History and the audit log.

## Troubleshooting

- **"AI provider not configured"**: set `anthropic_api_key` (or the
  OpenAI-compatible options) and restart the add-on.
- **"cannot open recorder DB read-only"**: the SQLite file is missing or
  locked; insights fall back to the history API.
- **A data source could not be read**: shown at the bottom of the
  Overview. Usually a Supervisor or Core API that your version doesn't
  have; the rest of the analysis still runs.
