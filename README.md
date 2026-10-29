# Fuse Energy+ — Home Assistant Custom Integration

Home Assistant integration for [Fuse Energy](https://www.fuseenergy.com) that provides live sensors and full historical import for the Energy Dashboard.

## About this fork

This is a fork of [andrejkurlovic/fuse_energy](https://github.com/andrejkurlovic/fuse_energy),
made on 29 October 2026 to fix gas figures in the Energy Dashboard. Everything else
is the same as upstream.

### The problem

Fuse only gives gas as one total per day, for the day before. Just after midnight
that total is provisional: Fuse keeps revising it through the next morning, and the
first figure is often well below the final one. Upstream writes each day's gas into
HA's long-term statistics once, on the first hourly poll after midnight, and never
looks at that day again. So the Energy Dashboard keeps the provisional figure. Over
the days checked when this fork was made, stored gas was about 23% below Fuse's
final figures, and gas cost was low by a similar amount. Upstream issue
[#5](https://github.com/andrejkurlovic/fuse_energy/issues/5) describes the same thing.

### What this fork changes

- **Gas days are revised.** Every hourly poll re-reads the last 7 days from Fuse. If a
  day's kWh or cost has changed, that day and every later day are written again,
  with the running totals recomputed (HA's statistics store a running total, so one
  day can't be changed on its own). Each rewrite is logged at WARNING as
  `FuseEnergy: revised ...` with the old and new figures. This is Jeff Aronow's fix
  from upstream PR [#6](https://github.com/andrejkurlovic/fuse_energy/pull/6)
  (commits `f342261`, `787ab89`).
- **Any bar type counts.** PR #6 only revises from bars Fuse labels `REALISED`. Fuse
  revises the figure while the bar can still say `FORECAST`, so this fork uses the
  latest figure whatever the label, the same figure the Yesterday sensor shows
  (commit `4a65524`).
- **HA 2026.11.** Cost statistics now pass `unit_class`, which HA 2026.11 requires
  (also from PR #6).

Days older than 7 days are not revised. To correct older history, run
`fuse_energy.import_history` with gas only (see below). One catch: the import skips
days where Fuse now reports 0 kWh, so a day that had a figure before and is 0 now
keeps its old row and old running total, which shows as a dip and a spike in the
dashboard. After a rebuild, check that each day's running total equals the day
before plus that day's value, and delete any row that doesn't fit.

### Versions and updates

Releases are tagged by date (`v2026.10.29` is the first). HACS installs the latest
release and only offers an update when a new release is published, not on every
commit. To ship a change: commit it, set `version` in
`custom_components/fuse_energy/manifest.json` to the new date, then publish a release
with a matching tag. Upstream fixes don't arrive here on their own: merge them in.
If upstream merges PR #6 and fixes the bar-type point, this fork can be dropped
and HACS pointed back at upstream.

## Features

- **Two-device model**: separate Electricity Meter and Gas Meter devices
- **Live sensors**: today's kWh, cost, tariff name, unit rate, standing charge
- **Account balance** sensor
- **Historical import service**: injects past energy and cost data into HA long-term statistics
- **Energy Dashboard**: imported statistics appear as selectable energy sources
- **Auto re-auth**: when tokens expire, HA shows a re-authentication notification

---

## Installation

### HACS (recommended)

1. Add this repository as a **custom repository** in HACS:
   - URL: `https://github.com/davedaspade/fuse_energy`
   - Category: Integration
2. Install **Fuse Energy+**
3. Restart Home Assistant

### Manual

Copy `custom_components/fuse_energy/` into your HA config's `custom_components/` directory, then restart.

---

## Authentication

Fuse Energy uses a magic-link / OTP flow:

1. Go to **Settings → Devices & Services → Add Integration → Fuse Energy+**
2. Enter your Fuse Energy account email
3. Depending on your account:
   - **Magic link**: Fuse emails you a link. Open it on a desktop, right-click → *Copy link address*, paste the full URL into the HA form.
   - **OTP**: Fuse texts a code; enter it in the HA form.
4. Setup completes and two meter devices + one account device appear.

### Re-authentication

When HA shows a **"Re-authentication required"** notification for Fuse Energy+:

1. Click the notification (or go to Settings → Devices & Services → Fuse Energy+ → ⋯ → Re-authenticate)
2. Click **Submit** — this sends a new magic link / OTP to your email/phone
3. Follow the magic link or enter the OTP code
4. HA reloads the integration automatically

> **Why does re-auth happen?** Fuse access tokens have a limited lifetime. Normally the integration silently refreshes them; re-auth is only needed when the refresh token itself expires (typically after many months of inactivity or a password change). The integration now persists refreshed tokens to the config entry so daily restarts do **not** require re-auth.

---

## Devices & Entities

### Electricity Meter device

| Entity | Description |
|---|---|
| `sensor.electricity_meter_today` | kWh consumed today (daily snapshot) |
| `sensor.electricity_meter_yesterday` | kWh consumed yesterday (disabled by default) |
| `sensor.electricity_meter_cost_today` | Cost today (GBP) |
| `sensor.electricity_meter_tariff` | Tariff name |
| `sensor.electricity_meter_unit_rate` | Unit rate (£/kWh) |
| `sensor.electricity_meter_standing_charge` | Standing charge (£/day) |

### Gas Meter device

Same sensors with `gas_meter_` prefix. Gas today may show `0` — this is normal (smart gas meters upload daily with a 1-day lag; use the Yesterday sensor for confirmed data).

### Account device (Hollins / premises name)

| Entity | Description |
|---|---|
| `sensor.balance` | Account balance (GBP) |

---

## History Import Service

The integration provides a `fuse_energy.import_history` service that fetches your full energy history from the Fuse API and injects it into HA's long-term statistics database.

### Statistic IDs created

| Statistic ID | Description | Unit |
|---|---|---|
| `fuse_energy:electricity_import_kwh` | Electricity energy | kWh |
| `fuse_energy:electricity_import_cost` | Electricity cost | GBP |
| `fuse_energy:gas_import_kwh` | Gas energy | kWh |
| `fuse_energy:gas_import_cost` | Gas cost | GBP |

### Service parameters

| Parameter | Default | Description |
|---|---|---|
| `start_date` | `2025-09-09` | Earliest date to import (YYYY-MM-DD) |
| `end_date` | Yesterday | Latest date to import (YYYY-MM-DD) |
| `include_electricity` | `true` | Import electricity statistics |
| `include_gas` | `true` | Import gas statistics |
| `include_cost` | `true` | Import cost alongside energy |
| `granularity` | `auto` | `auto`/`hourly` = hourly electricity, daily gas; `daily` = daily for both |
| `dry_run` | `true` | If true, log what would be imported without writing to HA |

> **Always run with `dry_run: true` first** to verify the date range and point counts before committing.

### Step 1 — Dry run (verify)

In **Developer Tools → Services**:

```yaml
service: fuse_energy.import_history
data:
  start_date: "2025-09-09"
  end_date: "2026-05-30"
  include_electricity: true
  include_gas: true
  include_cost: true
  granularity: auto
  dry_run: true
```

The service returns immediately; check **Settings → System → Logs** for a summary line like:

```
FuseEnergy import_history (dry_run=True): 273 API calls |
elec_kwh=6344 elec_cost=6344 gas_kwh=264 gas_cost=264 |
range=2025-10-07 05:00:00+00:00..2026-05-30 23:00:00+00:00 |
granularity=hourly
```

### Step 2 — Live import

Once the dry run looks correct, set `dry_run: false`:

```yaml
service: fuse_energy.import_history
data:
  start_date: "2025-09-09"
  dry_run: false
```

The import runs in the background (~1–2 minutes for hourly mode, ~5 seconds for daily mode). Check HA logs for per-statistic injection counts.

### Granularity details

- **`auto` / `hourly`**: fetches the hourly chart endpoint (1 API call per day, ~270 calls for 9-month backfill). Electricity gets 24 hourly data points per day. Gas is always daily.
- **`daily`**: fetches the monthly chart endpoint only (~9 calls for full backfill). Both electricity and gas get one data point per day.

> ⚠️ **Important — do not mix granularities for electricity:**
> Electricity history is stored as **hourly** statistics (24 bars/day, timestamped per hour).
> Gas history is stored as **daily** statistics (1 bar/day).
> If you have already imported electricity with `auto` or `hourly`, **do not re-import using `daily`** — the daily bars share the midnight (00:00) timestamp with the first hourly bar of each day, and will overwrite it with the full day's total. Always use `granularity: auto` (or `hourly`) for any subsequent electricity import.

---

## Energy Dashboard Setup

After running `import_history` with `dry_run: false`:

1. Go to **Settings → Dashboards → Energy**
2. Under **Grid consumption**, click **Add consumption**
3. Select `fuse_energy:electricity_import_kwh` (Fuse Energy Electricity Import)
4. Optionally add `fuse_energy:electricity_import_cost` as the cost tracking source
5. Under **Gas consumption**, click **Add gas source**
6. Select `fuse_energy:gas_import_kwh` (Fuse Energy Gas Import)
7. Save — the dashboard will populate with your full history

---

## Known Limitations

- **Gas today = 0**: Smart gas meters upload daily with a 1-day lag. Use the Yesterday sensor for confirmed gas data.
- **Electricity granularity**: Hourly (not 30-minute) — Fuse's API returns 24 bars per day.
- **Gas granularity**: Daily only — no sub-daily gas data is available from the Fuse API.
- **Historical data**: Starts from your Fuse switch-in date (tested from 2025-09-09). Dates before switch-in return empty data (handled gracefully).
- **Bar types**: electricity history imports only REALISED bars. Gas imports bars of any type, because Fuse labels recent gas days FORECAST while it is still revising them (see "About this fork").
- **Rate limits**: Hourly backfill uses ~270 API calls spaced 0.3s apart. No known rate limit issues, but re-running immediately is fine (upsert is idempotent).

---

## API Source

Endpoints extracted from Fuse Energy APK v2.0.65 via JADX decompilation. All endpoints documented in `ai-working/FUSE_API_MAP.md`.
