# kapta-prices

Centralized material price snapshots for [kapta-desktop-app](https://github.com/faculoyarte/kapta-desktop-app).

A daily GitHub Actions cron runs the scraper here and commits a fresh
`latest.json`. Studio machines fetch it over plain HTTPS — **no secrets live
on user machines**.

## Why this exists

Before centralization, every studio machine that wanted fresh prices needed:

- A MercadoLibre OAuth app + auto-rotating `.meli-token.json`
- (Optionally) a Google service account for cross-machine syncing

Sharing token files between machines doesn't work — they auto-rotate, and the
moment one machine refreshes, the others' copies go stale and MELI invalidates
the old refresh token.

The fix: **run the scraper exactly once, somewhere central, and have user
machines consume the result.** That somewhere is this repo. Secrets live in
GitHub Actions, never on a laptop.

## What lives here

| File | Role | Edited by |
|---|---|---|
| `materials.json` | Catalog: which materials Kapta tracks (id, name, unit, category, search terms) | Admin (via desktop app *Publicar precios*) |
| `manual-prices.json` | Trusted supplier overrides (corralones, Easy, Sodimac…). `confidence: "high"` entries beat scraped prices. | Admin (via desktop app *Publicar precios*) |
| `categories.json` | Documentation: canonical category ids referenced by `materials.json` | Admin (rare; via PR) |
| `latest.json` | Current price snapshot — every collected price per material plus a `selectedPrice` / `selectedSource` chosen by the rules below. | **Bot** (workflow only) |
| `latest.csv` | CSV summary of `latest.json` (one row per material) | **Bot** (workflow only) |
| `history/prices-YYYY-MM-DD.{json,csv}` | Dated snapshots, one pair per workflow run. Useful for tracking price drift. | **Bot** (workflow only) |

## How user machines consume this

In the desktop app's `.env`:

```bash
KAPTA_PRICES_MODE=remote
```

On startup and via the *Refrescar* button, the app fetches:

```
https://raw.githubusercontent.com/faculoyarte/kapta-prices/main/latest.json
https://raw.githubusercontent.com/faculoyarte/kapta-prices/main/manual-prices.json
https://raw.githubusercontent.com/faculoyarte/kapta-prices/main/materials.json
```

No auth. `raw.githubusercontent.com` doesn't apply api.github.com's 60/hr
unauthenticated rate limit, so a studio with multiple machines + restarts is
fine.

If a fetch fails (offline, GitHub down), the app keeps working with the local
cache from the previous successful refresh.

## How prices get refreshed

[`.github/workflows/update-prices.yml`](./.github/workflows/update-prices.yml)
runs daily at **06:00 UTC** (≈ 03:00 ART) and on manual dispatch. The job:

1. Checks out this repo (the source of truth for input files and outputs).
2. Checks out `faculoyarte/kapta-desktop-app` side by side. The scraper code
   lives there — we don't duplicate it here.
3. Copies our `manual-prices.json` + `materials.json` + `categories.json` into
   the app's `data/`.
4. Runs `npm run update-prices` in the app. Same scraper that admins used to
   run locally; uses the MELI secrets from this repo.
5. Copies `latest.json` / `latest.csv` / `history/prices-YYYY-MM-DD.*` back
   into this repo and commits + pushes if anything changed.

Concurrency: a single in-flight job at a time (`concurrency.group: update-prices`)
so a manual run while the cron is in progress queues rather than races.

Failure modes:

- **One scraper source breaks** — the script tolerates per-source failures
  (existing `try/catch` in `updatePrices.ts`). The output `latest.json` is
  still written with whatever did succeed.
- **Whole script crashes** — no commit happens. The previous snapshot stays
  as the latest, which is the desired behavior.
- **No price changes** — the commit step no-ops (`git diff --cached --quiet
  || ...`).

## Required secrets

Set in **Settings → Secrets and variables → Actions**:

| Secret | Purpose | Required? |
|---|---|---|
| `MELI_REFRESH_TOKEN` | Long-lived MercadoLibre refresh token | Recommended |
| `MELI_CLIENT_ID` | MELI app client id | Recommended (with refresh) |
| `MELI_CLIENT_SECRET` | MELI app client secret | Recommended (with refresh) |
| `MELI_ACCESS_TOKEN` | Current 6h access token (manual rotation) | Optional fallback |

If none are set, the MELI source is skipped and the run scrapes only the
open-web suppliers (Santa Fe Materiales, Aremat Puerto). The job still
succeeds.

To get the refresh credentials, complete the Authorization Code flow once at
<https://developers.mercadolibre.com.ar/en_us/tools/authentication-and-authorization>.
The MELI refresh token doesn't expire as long as it's used regularly — the
daily cron keeps it alive.

## How an admin edits prices

On the designated **admin machine** in the desktop app (set
`KAPTA_PRICES_ADMIN=1` in `.env`):

1. Open *Precios* in the UI. Edit `manual-prices.json` (entries) or the
   catalog (`materials.json`) inline. Local writes are immediate.
2. Click **Publicar precios**. The app:
   - Clones this repo to a tempdir
   - Copies the two files over
   - Commits `prices: publish manual + catalog from <username>` and pushes
   - Cleans up

The next cron run picks up the new inputs and regenerates `latest.json`. To
force a regeneration immediately, dispatch the workflow manually
(**Actions → Update prices → Run workflow**).

Other studio machines stay read-only — they pull from this repo but cannot
push. The desktop app enforces this server-side (PUT to `/api/prices/manual`
or `/api/catalog` returns 403 when `KAPTA_PRICES_MODE=remote` and
`KAPTA_PRICES_ADMIN` is unset).

You can also publish from the CLI on the admin machine:

```bash
KAPTA_PRICES_ADMIN=1 npm run publish-prices
```

## Ad-hoc refresh

GitHub UI → **Actions** → *Update prices* → *Run workflow*.

Use this when:

- Verifying the cron works for the first time
- A scraper source was just fixed and you want a fresh snapshot before
  tomorrow's run
- Right after publishing manual prices, to push the changes through to
  `latest.json` immediately

## History

Each run writes `history/prices-YYYY-MM-DD.{json,csv}`. To see how a price
moved over time:

```bash
git clone https://github.com/faculoyarte/kapta-prices
cd kapta-prices
ls history/                                  # list dated snapshots
git log -p latest.json                       # diff latest.json across days
```

Old history files are kept indefinitely — they're tiny.

## Editing categories

`categories.json` is documentation, not enforced. To add or rename a category,
open a PR against this repo. Admins editing through the desktop app *won't*
push category changes today (the publish flow only handles `manual-prices.json`
+ `materials.json`).

## Local preview of the workflow

Since the workflow checks out two repos and uses GitHub-only secrets, running
it locally is awkward. The fastest "does my edit work?" loop is:

1. Edit `manual-prices.json` or `materials.json` here.
2. Push.
3. Trigger the workflow manually.

Or run the scraper directly from the desktop app on a machine that has MELI
credentials in its local `.env` — that's exactly what the workflow does
inside the runner.

## Reverting

If you ever want to take a studio machine off the centralized model, unset
`KAPTA_PRICES_MODE` in its `.env`. The desktop app falls back to running the
scraper locally — the original behavior. The legacy code paths are still in
the desktop app for safety.
