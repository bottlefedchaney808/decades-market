# Decades Market

The auction tools behind the Market page in Decades. Built to be part of that
install, not a side project.

## What he ships in the addon

`decades/Market.lua` goes in `DecadesGuildOps/Features/`, loaded after
`decades/MarketBoard.lua`. See `DECADES_HANDOFF.md` for mounting that page.

The larger job — scanner, price store, pooled guild scans, desktop sync — is
`HANDOFF_INTEGRATION.md`. Start there. `addon/DecadesMarket/` 2.3.0 is the reference
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

**The standalone addon carries its own copy of the board (2.3.0).** DecadesGuildOps used to load
`MarketBoard.lua`; an update of that addon removed the file, and the Movers tab's Pipeline view
disappeared with it. `tools/export_decades.py` (run every cycle by `tools/wowd.py`) now also writes
`addon/DecadesMarket/BoardData.lua`, listed in the addon's toc before `Data.lua`. It keeps the board
in the addon's own namespace and publishes the global `DecadesMarketBoard` only when none exists or
the existing one has an older `generatedAt`; the addon reads whichever of the two is newer. **The
`BoardData.lua` in the INSTALLED addon folder
(`...\_forever_\Interface\AddOns\DecadesMarket\BoardData.lua`) is what a sync step must overwrite**
to refresh the board in game; nothing here writes into the game folder. Without a board the
Pipeline control stays on screen, disabled, and its tooltip says what produces one.

## The other addon

`addon/DecadesMarket/` (2.3.0) is the standalone addon: its own scanner, window (`/dm`),
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

## Downloads

Each release carries three separate packages:

| Package | For | What it is |
| --- | --- | --- |
| `decades-market-handoff.zip` | The Decades maintainers | `HANDOFF_INTEGRATION.md`, the addon source as reference implementation, the headless test rig, the Market page files, and the exporter. Start with the handoff document. |
| `DecadesMarket-2.3.0.zip` | Players | The standalone addon. Unzip into `Interface/AddOns/`. Works on its own; Auctionator optional. |
| `decades-market.zip` | Whoever runs the pipeline | Everything: addon, page, tools, watcher, tests, working database. |
