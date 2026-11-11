# Decades Market

The auction tools behind the Market page in Decades. Built to be part of that
install, not a side project.

## What he ships in the addon

`decades/Market.lua` goes in `DecadesGuildOps/Features/`, loaded after
`decades/MarketBoard.lua`. See `DECADES_HANDOFF.md`. The page is the window.
This repo is what fills it.

## What produces the numbers

`run_watcher.bat` runs `python tools\wowd.py --interval 300`.

Each cycle:

1. `tools/ingest_wow.py` reads the `DecadesMarket` SavedVariables dump.
2. `tools/analyze_wow.py`, `trend_wow.py`, `craft_wow.py` write the boards.
3. `tools/export_decades.py` writes `out/decades/market.json`.

`Features/MarketBoard.lua` is that board as one Lua global,
`DecadesMarketBoard`. The desktop app should overwrite that file on sync.
Do not put it in `DecadesGuildOps_Data/Data.lua`. That file is replaced whole.

## The other addon

`addon/DecadesMarket/` (2.0.0) is a separate addon with its own small window (`/dm`):
watchlist and alerts, tooltip trends, bag value, movers. It is also still the dump pipe:
`/dmdump` writes the SavedVariables this pipeline ingests. The Decades Market page is the
analysis; the window is the player's own tools. See `addon/DecadesMarket/README.md`.

## Run it

Python 3.11+. No extra packages.

```
selfcheck.bat
run_watcher.bat
```

`out/wow_market.db` is a working database from a live realm so the tools run
before the first scan. Raw character dumps are not in this archive.

## Downloads

Each release carries three separate packages:

| Package | For | What it is |
| --- | --- | --- |
| `decades-market-handoff.zip` | The Decades app owner | `Features/Market.lua`, a sample `MarketBoard.lua`, the exporter, and `DECADES_HANDOFF.md`. |
| `DecadesMarket-2.0.0.zip` | Players | The standalone addon. Unzip into `Interface/AddOns/`. Needs Auctionator. Not required for the Decades Market page. |
| `decades-market.zip` | Whoever runs the pipeline | Everything: page, standalone addon, tools, watcher, headless addon tests, working database. |

The page and the standalone addon have been checked headlessly against a mocked
client. Neither has been reviewed in the live client beyond the author's install.
