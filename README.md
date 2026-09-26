# TRMNL plugin: Braiins Miner

Private plugin pro [TRMNL](https://trmnl.com) (laděno na **TRMNL X**). Zobrazuje stav mineru
těžícího na [Braiins Pool](https://pool.braiins.com). Data bere z webového API poolu, ne z lokálního API mineru.

- **Dnes vytěženo** a **poslední výplata** (částka, datum, stav) jako hlavní čísla. Částky jsou v sats.
- **Denní odměny za 14 dní**: sloupcový graf (osa od nuly, propad = výpadek mineru) s tečkami ve dnech výplat.
- **Hashrate**: 24h průměr v TH/s a stav workerů (Online / Offline / Nízký výkon).

Layouty: `full`, `half_horizontal`, `half_vertical`, `quadrant` (mashupy).

## Jak to funguje

Strategie **polling**, tři URL (v šablonách `IDX_0`, `IDX_1`, `IDX_2`):

| | URL | Poznámka |
|---|---|---|
| `IDX_0` | `https://pool.braiins.com/accounts/profile/json/btc/` | hashrate, balance, workeři |
| `IDX_1` | `https://pool.braiins.com/accounts/payouts/json/btc?from={{ "now" \| date: "%s" \| minus: 7776000 \| date: "%Y-%m-%d" }}` | výplaty za posledních 90 dní |
| `IDX_2` | `https://pool.braiins.com/accounts/rewards/json/btc?from={{ "now" \| date: "%s" \| minus: 1209600 \| date: "%Y-%m-%d" }}` | denní odměny za 14 dní (jen uzavřené dny, od nejnovějšího) |

Token se posílá v hlavičce `Pool-Auth-Token={{ pool_token }}`. `pool_token` je custom field typu `password`,
takže není natvrdo v šabloně ani v repu.

Chování API (ověřeno 25. 9. 2026):

- **Výběr výplat:** `from`/`to` jsou volitelné a bez nich API vrátí celou historii. Budoucí datum projde. `from > to` a nevalidní datum vrací HTTP 400.
- **Pořadí:** položky jsou seřazené vzestupně (nejstarší první). Šablona přesto spojí `lightning` + `onchain` a řadí podle `requested_at_ts`.
- **Proč 90denní okno:** celá historie roste zhruba o 640 B na výplatu. TRMNL má limit 100 KB na polovaná data, takže by za necelý rok přestal fungovat.
- **Rate limit:** zhruba 1 req / 5 s. Tři požadavky hned po sobě prošly bez problému.

## Nastavení

### 1. Token v Braiins Pool

V Braiins Pool otevři **Settings → Access Profiles**, zapni **Allow access to web APIs** a klikni na **Generate New token**.

### 2. Lokální vývoj (trmnlp)

Požadavky: Ruby ≥ 4.0 a gem `trmnl_preview`. Systémové Ruby na macOS (2.6) nestačí a Homebrew Ruby je keg-only,
proto je potřeba ho dát do `PATH`. Platí to i pro `trmnlp login` a `trmnlp push`. `bin/serve` si ho přidá sám.

```sh
brew install ruby
export PATH="/opt/homebrew/opt/ruby/bin:$(/opt/homebrew/opt/ruby/bin/gem env gemdir)/bin:$PATH"
gem install trmnl_preview
```

```sh
cp .env.example .env          # a doplň BRAIINS_POOL_TOKEN
bin/serve                     # trmnlp serve s tokenem z .env → http://127.0.0.1:4567
```

V preview zvol model **TRMNL X** a paletu **16 Grays**.

Pomocné skripty:

- `bin/serve`: spustí `trmnlp serve` s `TZ=UTC`. TRMNL servery běží v UTC a šablona posouvá datum přes `trmnl.user.utc_offset`.
- `bin/shot [view] [out.png] [palette]`: screenshot přes headless Chrome v rozlišení TRMNL X (1872×1404, `screen--v2 screen--lg screen--density-2x screen--4bit`).
- `bin/scenario <name>`: podstrčí běžícímu serveru data z `fixtures/scenarios/` (offline miner, žádné výplaty, selhaná výplata, výpadek API…). Návrat k živým datům: `bin/scenario --live`.

`fixtures/profile.json`, `fixtures/payouts.json` a `fixtures/rewards.json` jsou anonymizované reálné odpovědi API.

### 3. Nahrání do TRMNL

```sh
trmnlp login                  # API klíč z trmnl.com → Account
trmnlp push
```

První `push` vytvoří nový private plugin a zapíše jeho `id` do `src/settings.yml`. **Tuhle změnu commitni.**
Bez `id` by každý další `push` založil nový plugin.

Pak v TRMNL otevři nastavení pluginu, vyplň **Braiins Pool API token** a přidej plugin do playlistu.
Data se obnovují každých 15 min (`refresh_interval: 15`). Pool snapshotuje statistiky po 5 min.

## Struktura

```
src/
  settings.yml          # polling, hlavičky, custom field pool_token
  shared.liquid         # výpočty (TH/s, sats, stav workerů, poslední výplata) – vkládá se před každý layout
  full.liquid
  half_horizontal.liquid
  half_vertical.liquid
  quadrant.liquid
.trmnlp.yml             # lokální konfigurace trmnlp (token z env, time_zone)
fixtures/               # vzorová data + scénáře
bin/                    # serve / shot / scenario
```

## Logo

Symbol Braiins je z [design.braiins.com](https://design.braiins.com/braiins/logos/braiins-symbol).
Černá varianta je v `assets/braiins-symbol-black.svg` a v šabloně je vložená inline jako data URI (`shared.liquid`).
Logo je ochranná známka Braiins, plugin s Braiins nijak nesouvisí.

## Omezení

Pool API nezná teploty, příkon ani uptime mineru. Miner musí těžit na `pool.braiins.com`.
