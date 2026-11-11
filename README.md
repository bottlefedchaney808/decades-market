# Decades Market

The auction tools behind the Market page in Decades. Built to be part of that
install, not a side project.

## What he ships in the addon

`decades/Market.lua` goes in `DecadesGuildOps/Features/`, loaded after
`decades/MarketBoard.lua`. See `DECADES_HANDOFF.md` for mounting that page.

The larger job — scanner, price store, pooled guild scans, desktop sync — is
`HANDOFF_INTEGRATION.md`. Start there. `addon/DecadesMarket/` 2.2.0 is the reference
the author accepted in the live client. Port its maths. Do not treat the Python mover
ranking as the spec.

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

`addon/DecadesMarket/` (2.2.0) is the standalone addon: its own scanner, window (`/dm`),
watchlist, tooltip trends, bag value, movers, and a momentum meter. It is also the dump
pipe: `/dmdump` writes the SavedVariables this pipeline ingests. The Decades Market page
is the analysis; the window is the reference for the features still to be ported. See
`addon/DecadesMarket/README.md`.

## Run it

Python 3.11+. No extra packages.

```
selfcheck.bat
run_watcher.bat
```

`out/wow_market.db` is a working database from a live realm so the tools run
before the first scan. Raw character dumps are not in this archive.
