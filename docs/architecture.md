# Design notes

A deeper look at how the platform is put together. This is a sanitized overview written for
a portfolio — it describes design decisions, not the production source.

## Why one bot per chat

Telegram bots are the unit of isolation. Instead of a single bot juggling many groups, an
admin bot provisions a dedicated "secret" bot per chat and runs them all in one process,
keyed by bot token. Benefits:

- **Identity isolation.** Participants interact under a pseudonym; the bot is the only party
  that knows the mapping, and reviewers never see raw Telegram identities.
- **Per-chat configuration.** Shift hours, requisites, roles, and rate settings live against
  a bot token, so each team is configured independently without cross-talk.
- **Blast-radius control.** A misbehaving chat is one token; the rest are unaffected.

## Data model

A single SQLite database is shared by the relay and accounting services. The relay owns the
tenant and messaging tables; the accounting service reads those and owns the `acc_*` tables.

- **Tenancy & identity** — bots, pseudonyms, per-chat roles (admin / handler / watcher),
  invite links with optional role, expiry, and single-use.
- **Messaging** — a persisted message map so broadcasts and edits survive restarts, plus a
  replayable receipt log (amount, comment, photo, status, history).
- **Requisites** — a list of payment details per bot with persisted pin tracking, so the
  current details stay pinned for every participant.
- **Accounting** — groups, group↔bot and group↔geo links, target chats (several chats per
  group, one per region), FX rates, and stored reconciliation reports.

### Idempotent migrations

Schema changes run on every startup and are safe to re-run: each checks `PRAGMA table_info`
before `ALTER TABLE`, and heavier changes rebuild a table and copy rows. There's no separate
migration tool to orchestrate — the service brings its own schema forward.

## Shifts and the reporting day

Teams work shifts that can cross midnight or run 24h (e.g. a shift that starts 05:01 and
closes 05:00 the next day). The "reporting day" a receipt belongs to is derived from the
chat's shift, not the wall clock, so a receipt entered after midnight still lands on the
correct settlement day. The same rule dates both automated and manual reconciliations, so
every chat's report for a given day agrees.

## Reconciliation

At shift close the accounting service groups the day's approved receipts by team and region,
converts to USD at the region's current rate, applies the group's commission, and renders a
settlement report from one shared template — so manual reports (for teams not on the bot)
look identical to automated ones. A net-negative total (a withdrawal/return) passes through
without commission as a clean deduction. Reports are stored and can be re-sent or re-rendered.

## Delivery & operations

- **Three systemd services** on one droplet, all `enabled` so they auto-start on boot and
  restart on failure.
- **Deploy** is a copy-and-restart with a timestamped backup of the binary and database
  first, then a health check (`is-active`, restart count) before walking away.
- **Observability** through journald and structured logs; issues are diagnosed against a
  read-only copy of the production database.

## Frontend: the dashboard

The operations dashboard (`dashboard/index.html`) is a single self-contained file:

- **No chart libraries** — every chart (area with crosshair, horizontal bars, a 7×24
  heatmap, a stacked status bar, sparklines) is hand-drawn inline SVG against a shared scale.
- **Theme-aware** — a full light and dark palette defined as CSS tokens, following the
  viewer's system theme with a manual toggle.
- **Deterministic sample data** — a seeded PRNG so the figures are stable across reloads,
  which keeps screenshots consistent.
