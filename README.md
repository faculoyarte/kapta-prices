# kapta-prices

Centralized material price snapshots for [kapta-desktop-app](https://github.com/faculoyarte/kapta-desktop-app).

A daily GitHub Actions cron runs the scraper and commits a fresh `latest.json`
that user machines pull over plain HTTPS. **No secrets live on user machines.**

## What lives here

| File | Role | Edited by |
|---|---|---|
| `materials.json` | Catalog: which materials Kapta tracks | Admin (via desktop app *Publicar*) |
| `manual-prices.json` | Trusted supplier overrides (corralones, Easy, Sodimac…) | Admin (via desktop app *Publicar*) |
| `categories.json` | Documentation: canonical category ids | Admin (rare; via PR) |
| `latest.json` | Current price snapshot | **Bot** (workflow only) |
| `latest.csv` | CSV summary of `latest.json` | **Bot** (workflow only) |
| `history/prices-YYYY-MM-DD.{json,csv}` | Dated snapshots, one per workflow run | **Bot** (workflow only) |

## How user machines consume this

The desktop app, when started with `KAPTA_PRICES_MODE=remote`, fetches:

```
https://raw.githubusercontent.com/faculoyarte/kapta-prices/main/latest.json
https://raw.githubusercontent.com/faculoyarte/kapta-prices/main/manual-prices.json
https://raw.githubusercontent.com/faculoyarte/kapta-prices/main/materials.json
```

No auth, no rate limits at this volume.

## How prices get refreshed

`.github/workflows/update-prices.yml` runs daily at 06:00 UTC and on manual
dispatch. It:

1. Checks out this repo and `kapta-desktop-app` side by side.
2. Copies our `manual-prices.json` + `materials.json` into the app's `data/`.
3. Runs `npm run update-prices` in the app — same scraper users used to run locally.
4. Copies `latest.json` / `latest.csv` / `history/*` back here and commits.

The scraper code lives in `kapta-desktop-app/src/pricing/` — we don't duplicate it.
This repo only owns *data* and the *cron*.

## Required secrets

Set these in **Settings → Secrets and variables → Actions**:

- `MELI_REFRESH_TOKEN` — long-lived MercadoLibre refresh token
- `MELI_CLIENT_ID`     — MELI app client id
- `MELI_CLIENT_SECRET` — MELI app client secret

Optional fallback (if you don't want to set up refresh):

- `MELI_ACCESS_TOKEN` — current 6h access token (manual rotation)

If none are set the MELI source is skipped — the run still succeeds, scraping
the open-web suppliers only.

## How an admin edits prices

On the designated **admin machine** in the desktop app:

1. Open *Precios*, edit `manual-prices.json` and/or the catalog.
2. Click **Publicar precios**. The app commits the changes back to this repo
   via `gh` CLI. The next workflow run picks them up.

Other studio machines stay read-only — they pull from this repo but cannot
push.

## Ad-hoc refresh

To run the workflow on demand: GitHub UI → **Actions** → *Update prices* →
*Run workflow*.

## History

Each run writes `history/prices-YYYY-MM-DD.{json,csv}`. To see how a price
moved over time, scan that directory or use `git log -p latest.json`.
