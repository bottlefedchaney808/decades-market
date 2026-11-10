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

`addon/DecadesMarket/` is the dump pipe. `/dmdump` writes SavedVariables.
No window. The window is the Decades page.

## Run it

Python 3.11+. No extra packages.

```
selfcheck.bat
run_watcher.bat
```

`out/wow_market.db` is a working database from a live realm so the tools run
before the first scan. Raw character dumps are not in this archive.
