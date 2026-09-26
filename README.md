# TRMNL plugin: Braiins Miner

A [TRMNL](https://trmnl.com) plugin that shows the status of your miners on [Braiins Pool](https://pool.braiins.com).
Built for **TRMNL X** and also tested on the original TRMNL (800×480, 1-bit).
Data comes from the pool's web API, not from the miner's local API.

*Česká verze je [níže](#česky).*

- **Mined today** and **last payout** (amount, date, status) as the main numbers. All amounts are in sats.
- **Daily rewards for the last 14 days**: a bar chart with the axis starting at zero, so a drop means a miner outage. A dot under a day marks a payout.
- **Hashrate**: 24h average in TH/s (PH/s from 1,000 TH/s) and worker status. A single worker is shown in words (Online / Offline / Low hashrate), multiple workers with counts, e.g. "3/12 offline · 1 low".
- **Total mined**: `all_time_reward` from the profile, in sats.

Layouts: `full`, `half_horizontal`, `half_vertical`, `quadrant` (mashups).

**Language:** English and Czech, set in the plugin settings (*Language / Jazyk*). `Auto` uses Czech for users with
locale `cs` and English otherwise. English formats numbers as `1,234` and `1.14`, Czech as `1 234` and `1,14`.

## How it works

Strategy **polling** with three URLs (`IDX_0`, `IDX_1`, `IDX_2` in the templates):

| | URL | Note |
|---|---|---|
| `IDX_0` | `https://pool.braiins.com/accounts/profile/json/btc/` | hashrate, rewards, workers |
| `IDX_1` | `https://pool.braiins.com/accounts/payouts/json/btc?from={{ "now" \| date: "%s" \| minus: 7776000 \| date: "%Y-%m-%d" }}` | payouts in the last 90 days |
| `IDX_2` | `https://pool.braiins.com/accounts/rewards/json/btc?from={{ "now" \| date: "%s" \| minus: 1209600 \| date: "%Y-%m-%d" }}` | daily rewards for 14 days (completed days only, newest first) |

The token is sent in the header `Pool-Auth-Token={{ pool_token }}`. `pool_token` is a custom field of type `password`,
so it is never hard-coded in the templates or stored in the repo.

API behavior (verified on Sep 25, 2026):

- **Payout range:** `from`/`to` are optional; without them the API returns the full history. Future dates are accepted. `from > to` and invalid dates return HTTP 400.
- **Order:** items are sorted ascending (oldest first). The template still merges `lightning` + `onchain` and sorts by `requested_at_ts`.
- **Why a 90-day window:** the full history grows by about 640 B per payout. TRMNL limits polled data to 100 KB, so the plugin would stop working within a year.
- **Rate limit:** about 1 request / 5 s. Three requests in a row worked fine.

## Setup

### 1. Braiins Pool token

In Braiins Pool, open **Settings → Access Profiles**, enable **Allow access to web APIs** and click **Generate New token**.
The token is read-only.

### 2. Local development (trmnlp)

Requirements: Ruby ≥ 4.0 and the `trmnl_preview` gem. The macOS system Ruby (2.6) is too old, and Homebrew Ruby is keg-only,
so it has to be added to `PATH`. This also applies to `trmnlp login` and `trmnlp push`. `bin/serve` adds it on its own.

```sh
brew install ruby
export PATH="/opt/homebrew/opt/ruby/bin:$(/opt/homebrew/opt/ruby/bin/gem env gemdir)/bin:$PATH"
gem install trmnl_preview
```

```sh
cp .env.example .env          # and fill in BRAIINS_POOL_TOKEN
bin/serve                     # trmnlp serve with the token from .env → http://127.0.0.1:4567
```

In the preview, pick the **TRMNL X** model and the **16 Grays** palette (or the original TRMNL with Black & White).

Helper scripts:

- `bin/serve`: runs `trmnlp serve` with `TZ=UTC`. TRMNL servers run in UTC and the template shifts dates via `trmnl.user.utc_offset`.
- `[MODEL=x|og] bin/shot [view] [out.png] [palette]`: screenshot via headless Chrome. `MODEL=x` (default) = TRMNL X (1872×1404, 4-bit), `MODEL=og` = original TRMNL (800×480, 1-bit).
- `bin/scenario <name>`: feeds the running server with data from `fixtures/scenarios/` (offline miner, no payouts, failed payout, API outage, multi-worker farm `farm_multi_worker`, large PH/s farm `farm_large`…). Back to live data: `bin/scenario --live`.

`fixtures/profile.json`, `fixtures/payouts.json` and `fixtures/rewards.json` are anonymized real API responses.

### 3. Upload to TRMNL

```sh
trmnlp login                  # API key from trmnl.com → Account
trmnlp push
```

The first `push` creates a new private plugin and writes its `id` to `src/settings.yml`. **Commit that change.**
Without the `id`, every later `push` would create another plugin.

`push` uploads the whole `settings.yml`, including `recipe_overview`. If you edit the overview on the TRMNL website,
the next `push` pulls it back into `settings.yml`, so commit it as well.

Then open the plugin settings in TRMNL, fill in **Braiins Pool API token** and add the plugin to a playlist.
Data refreshes every 15 minutes (`refresh_interval: 15`); the pool snapshots its statistics every 5 minutes.

## Structure

```
src/
  settings.yml          # polling, headers, custom fields (pool_token, language)
  shared.liquid         # shared logic (translations, TH/s, sats, worker status, payouts, chart) – prepended to every layout
  full.liquid
  half_horizontal.liquid
  half_vertical.liquid
  quadrant.liquid
.trmnlp.yml             # local trmnlp config (token from env, time_zone)
fixtures/               # sample data + scenarios
bin/                    # serve / shot / scenario
```

## Logo

The Braiins symbol is from [design.braiins.com](https://design.braiins.com/braiins/logos/braiins-symbol).
The black variant is in `assets/braiins-symbol-black.svg` and is embedded inline as a data URI in `shared.liquid`.
The logo is a trademark of Braiins. This plugin is not affiliated with Braiins.

## Limitations

The pool API does not know miner temperatures, power draw or uptime. The miner has to mine on `pool.braiins.com`.

---

## Česky

Plugin pro [TRMNL](https://trmnl.com), který zobrazuje stav minerů na [Braiins Pool](https://pool.braiins.com):
dnes vytěžené sats, poslední výplatu, graf denních odměn za 14 dní, 24h hashrate, stav workerů a celkem vytěžené sats.
Laděný na TRMNL X, funguje i na původním TRMNL. Data bere z webového API poolu, ne z lokálního API mineru.

**Nastavení:**

1. V Braiins Pool otevři **Settings → Access Profiles**, zapni **Allow access to web APIs** a vygeneruj token (**Generate New token**). Token je jen pro čtení.
2. Přidej plugin v TRMNL a do nastavení vlož **Braiins Pool API token**.
3. V poli **Language / Jazyk** zvol **Čeština** (nebo `Auto`, které češtinu zvolí podle jazyka účtu).
4. Přidej plugin do playlistu. Data se obnovují každých 15 minut.

**Lokální vývoj:** postup je stejný jako v anglické části výše (`bin/serve`, `bin/shot`, `bin/scenario`, `trmnlp push`).
Po prvním `trmnlp push` commitni `id`, které se zapíše do `src/settings.yml`.

Logo Braiins je ochranná známka Braiins, plugin s Braiins nijak nesouvisí.
