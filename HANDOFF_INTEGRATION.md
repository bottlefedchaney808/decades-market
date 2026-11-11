# Decades Market — implementation handoff

For whoever implements this inside Decades: the `DecadesGuildOps` addon, the desktop app
and the cloud server. Written to be read cold. We cannot see the Decades source, so
everything here is stated from our side of the boundary. Where a decision is yours it is
listed in section 11, not guessed.

## 0. Read this first

**What you are getting.** A working standalone addon, `addon/DecadesMarket/` version
**2.3.0**, plus its headless test rig and a Python pipeline. The addon scans the auction
house itself, keeps its own price history, and provides a trading window with fourteen
pages. Its author has run 2.3.0 in the live client and reported that it tested well. It
does not need Auctionator, Decades, a server or a desktop app.

**What we are asking for.** Three things, in order of value to players:

1. Ship the addon's features inside Decades, on Decades' own widgets and theme.
2. Pool guild members' scans on the server and deliver the merged result to everyone.
3. Stop Decades updates from deleting market files (section 9; this has now happened
   twice in effect, once for real on 2026-11-11).

**How to work.** Port the maths, rebuild the window. The files that hold calculations
have no UI in them and can move across almost unchanged. The rig checks a port against
known answers. A port that changes a threshold and still passes was aimed at the wrong
tests.

**Reading order.**

1. This file.
2. `addon/DecadesMarket/README.md` — the rules for trend, momentum, movers, time to sell
   and price source. Those sections are the contract.
3. `addon/DecadesMarket/DUMP_SPEC.md` — every field that crosses the boundary.
4. `Tools.lua` top of file — conventions, thresholds (`Tools.K`), reasons, help text.
5. Comment blocks in `Data.lua` and `Store.lua`.
6. Run the rig (section 10).

`tools/trend_wow.py` is **not** the mover specification. Port the addon's `Data.lua`.

## 1. What exists

Load order is the toc's. "Live" means the author ran it in the real client.

| File | What it holds | UI in it? |
| --- | --- | --- |
| `Core.lua` | Saved-variable defaults, message bus, events, `/dm` router, money and item helpers | no |
| `BoardData.lua` | Generated copy of the pipeline's analysis board | no |
| `Data.lua` | Every lookup: price, market price, history, trend, timeframes, momentum, movers, breadth, categories and heatmap weights, depth, time to sell, bags, vendor buy and sell, price-source selection, board | no |
| `Store.lua` | Own price store: daily history, market price, depth ladder, sell-through accumulators | no |
| `Dump.lua` | SavedVariables dump for the pipeline | no |
| `Scan.lua` | Full auction-house scan, cooldown, `/dm scantest` | no |
| `Ledger.lua` | Merchant prices; ledger from the player's owned auctions | no |
| `Tools.lua` | Tool conventions, thresholds, reasons, help text, saved state | no |
| `ToolsBook.lua` | Sniper, reset finder | no |
| `ToolsSell.lua` | Posting advisor, undercut check | no |
| `ToolsLists.lua` | Shopping lists | no |
| `Recipes.lua` | Generated recipe table (2,308 recipes, about 160 KB) | no |
| `ToolsCraft.lua` | Craft profit, recipe-provider registry | no |
| `ToolsSellers.lua` | Seller lookup for one item | no |
| `ToolsPortfolio.lua` | Portfolio series and realised results | no |
| `Watch.lua` | Watchlist and price alerts | no |
| `Tooltip.lua` | Item tooltip lines | tooltip only |
| `Minimap.lua` | Minimap button | yes |
| `UI.lua`, `UITools.lua`, `UIResets.lua`, `UISell.lua`, `UICraft.lua`, `UICard.lua` | The window | yes |

Status: the scanner, store, window, tooltips, watchlist and dump have been used live
across 2.0 to 2.3. The eight trading tools were added in 2.3.0; the author ran that build
live and reported it went well, without itemising each tool, so treat individual tool
pages as "seen working once" and the tool maths as rig-verified.

## 2. The tools, and what each needs

All work from one player's own scans. Prices are judged against the **market price**
(the price at which the cheapest 15% of listed quantity is covered), never the single
cheapest listing. The auction house takes 5% of a sale.

| Tool | Function | What it answers | Needs |
| --- | --- | --- | --- |
| Sniper | `Tools.Sniper` | What is listed well under market, and the profit after the cut | A scan under 12 h old; 3 recorded days |
| Reset finder | `Tools.Reset` | Thin books where buying the cheap end and re-listing pays; every clear-to level, partial-success model, risk grade | A scan; evidence improves it |
| Posting advisor | `Tools.Post`, `Tools.PostList` | Price options from quick to patient with time to sell and net proceeds | A scan |
| Undercut check | `Tools.Undercuts` | Which of your auctions are no longer cheapest | A scan and the owned-auction list |
| Shopping lists | `Tools.Shopping`, `ListSet` … | Buy-now lines under each item's price cap | A scan |
| Craft profit | `Tools.Craft` | Material cost against sale price; profitable-now flag | A scan and a recipe provider |
| Seller view | `Tools.SellersLookup`, `Tools.Sellers` | Who sells an item and whether one seller holds the cheap end | One search request, auction house open |
| Portfolio | `Tools.Portfolio` | Gold, bags and auctions over time; realised sales | A scan; two scan-days for change |

Conventions every tool follows (stated at the top of `Tools.lua`):

- One plain table back. `ok == false` carries `reason` (a sentence for the player) and
  `reasonKey`. `ok == true` with empty `rows` carries `note`.
- Cached until `PRICES_CHANGED`, `DUMP_DONE`, `LEDGER_CHANGED`, `SETTINGS_CHANGED`.
  `TOOLS_CHANGED` fires when saved tool state changes.
- `Tools.HELP[tool]` holds a title, an explanation, and text for each unavailable state
  and each action. The window shows these on mouseover. Reuse them verbatim.
- Each tool has a `/dm` subcommand and a `Tools.XxxLines(result)` text form.

The reset finder is deliberately not filtered for the player's protection. Every
candidate is computed; the mode (conservative, standard, aggressive) changes only the
filter and the assumed re-list price. Keep that. Unproven candidates must stay visible
and must not lead the default sort.

## 3. The recipe provider — the one thing we need from Decades' data

Craft profit is written against this interface. Decades ships a recipe catalog but keeps
it private to its namespace, so the standalone falls back to a table we generate from
the same catalog. Inside Decades, register the real one:

```
ns.Tools.RegisterRecipeProvider(name, provider, priority)   -- priority: lower wins, default 50
provider.Recipes() -> array of recipe ids (any stable integer; spell ids in practice)
provider.Get(recipeId) -> nil, or
  { out = itemID,                                   -- required
    outQty = n,                                     -- default 1
    reagents = { { id = itemID, qty = n }, ... },   -- required, non-empty
    profession = skillLineID|nil, professionName = "Alchemy"|nil,
    reqSkill = n|nil,
    known = true|false|nil,                         -- does THIS character know it; nil = cannot tell
    name = text|nil }
provider.Version -> optional string
```

Built-in providers: `decades` (priority 10; switches on by itself if
`DecadesGuildOps.ProfCatalog` is exposed globally in this shape), `catalog` (20;
`Recipes.lua`, generated by `tools/gen_recipes_lua.py`), `observed` (30; read from the
open profession window, mock-tested only).

## 4. Facts about the client that shape the design

From a passive probe run in the live client (`addon/DecadesMarketProbe/`; its shapes are
reproduced in `tools/addon_test/ahmock.lua`) and from live scans.

- WoW Forever, Interface 16001, build 1.60.1. Only the modern `C_AuctionHouse` API exists
  (86 members). The legacy auction globals are absent.
- `C_AuctionHouse.ReplicateItems()` works. Live scans returned 225,165 and 240,670
  auctions; the answer arrives about 25 s after the call; our read took about 31 s end to
  end. `REPLICATE_ITEM_LIST_UPDATE` fired 28,393 times for one scan: do no work per
  event, start one batched reader, ignore the rest.
- `GetReplicateItemInfo(i)` is 0-based, 18 values: [1] name, [3] count, [10] buyout
  (copper, whole stack), [17] itemID, [18] hasAllInfo. A row whose item data is not loaded
  has name "" but already has correct itemID, count and buyout. No owner and no auction
  ID in replicate rows.
- Item and commodity searches do return seller names, exact listings and auction IDs.
  The client names at most four owners per price row.
- Throttle: `C_AuctionHouse.IsThrottledMessageSystemReady()`. `CanSendAuctionQuery` does
  not exist.
- An addon cannot use the network. Data leaves the game only through SavedVariables,
  written on `/reload` or logout, and enters only at login or `/reload`. That is why a
  sidecar is required and why pooled data always lags by a session.
- Whether the server answers a scan away from an auctioneer is untested. `/dm scantest`
  runs the experiment and saves its verdict in `DecadesMarketDB.scan.tests`.

## 5. Addon integration

Suggested mapping. The standalone window uses a left rail grouped MARKET, BUY, SELL,
CRAFT, ME; the Decades Market page had five tools (Movers, Flips, Crafts, Breaks,
Squeeze).

| Standalone page | In Decades |
| --- | --- |
| Pulse, category heatmap | Market landing view |
| Movers (live, with momentum sorts) and the pipeline's flips and squeeze | The Movers tool; live data first, board as a second source |
| Sniper, Resets, Shopping lists | New BUY views |
| Bags with posting advice, My auctions with undercuts | New SELL views |
| Craft profit | Replaces the board-fed Crafts tool |
| Portfolio, Watchlist | New ME views |
| Item card: timeframes, range bar, chart, order book, time to sell, sellers, watch | The Market item card |
| Tooltip lines | Decades' tooltip post-call |
| Watch and undercut alerts, minimap status | Decades' notification and launcher surfaces |
| Scan button, progress, price-source control | Market page header and settings |

This is well past the earlier "no sixth tab" rule in `DECADES_HANDOFF.md`. See section 11.

Lessons already paid for:

- Draw arrows and other symbols from solid textures. The game font cannot be trusted for
  arrow, ellipsis or dash glyphs.
- List rows are recycled. Create decorations once and reset every one on each paint.
- Scanning is an explicit player action with the auction house open and a cooldown (15
  minutes, Auctionator's figure; the server's real limit is unknown). Batch the read on a
  timer with a time budget per tick (ours targets 5 ms).
- Unit price is `floor(buyout / count)`. Skip rows with buyout 0 (bid only).
- The lowest listing is a bad price; one bait listing moves it wildly. Rank and trend on
  the market price, with a floor on listings.
- "Vendor price" is two numbers: what a merchant charges (`Data.VendorBuy`, for flips and
  reagents) and what a vendor pays the player (`Data.VendorSell`, for "vendor it").
- A new store has one recorded day, so everything needing history is empty. Say why on
  screen. Past days from Auctionator, when present, are carried in at read time as
  imported history and never written to our store.
- Every disabled control says why it is disabled. Every tool says what it needs when it
  has nothing to show.
- Random-suffix gear is keyed by itemID alone, so its variants share one entry.
- Soulbound bag items must be excluded from posting advice.

Day-count rules (normative, from the README): trend needs 3 recorded days in 14; momentum
needs 4 in a run with no gap longer than 3; a last scan older than 3 scan-days gives no
momentum reading.

## 6. Data that crosses the boundary

Exact field lists are in `addon/DecadesMarket/DUMP_SPEC.md`. This section is the map.

**Upload: one scan summary per (realm, faction).** Not raw auctions. Per item: minimum
unit price, market price, total quantity, auction count, and a depth ladder of up to 12
levels (8 exact prices, then buckets at +10%, +25%, +50%, beyond), each level with its
shortest time-left band. About 5,300 items, about 2.4 MB as Lua text, far less
compressed. Envelope: realm, faction, scan time, scanner identity, addon version, rows
read, whether the scan completed.

**Download: the merged realm file.** One Lua file written into the addon folder, a single
global replaced whole on every sync. Per item: current minimum and market price,
quantity, listings, ladder, daily history (low, high, market, quantity per scan-day),
sell-through rates with their evidence counts. Per realm: generated time, newest scan
time, number of scans and distinct scanners.

**Analysis board.** `BoardData.lua` in the addon folder, generated by
`tools/export_decades.py`. `Data.Board()` uses the newer of it and the global
`DecadesMarketBoard` by `generatedAt`. A sync step must overwrite the installed
`BoardData.lua`.

**Interim path, working today on one machine.** `Dump.lua` writes `DecadesMarketDump` per
character; `tools/ingest_wow.py` ingests it; `tools/export_decades.py` writes the board;
`tools/wowd.py` loops. The dump carries the per-item summary (`prices`) and the ladder
(`depth`). The Python is a reference for server-side analysis but was written for one
scanner.

## 7. Server

Inputs are near-duplicate snapshots of one auction house taken by different people at
different times. The work is merging them.

- **Current view:** newest complete scan per (realm, faction).
- **Daily history:** per item per scan-day across that day's scans: low and high of the
  minimum, a market price (median of the scans' market prices), maximum quantity. Days
  count from 2020-01-01T00:00:00Z so existing history lines up.
- **Sell-through:** compare each item's ladder between consecutive scans. Units that left
  a level were sold, cancelled or expired. Discard pairs more than 12 h or less than
  6 min apart. Count a level only when the interval is shorter than the lower bound of
  its shortest time-left band (under 30 min, 30 min to 2 h, 2 to 12 h, over 12 h).
  Accumulate units per hour per price bucket with pairs and hours behind it. Time to sell
  at a price = units queued at or below it, divided by the summed rate of the buckets up
  to it. Never publish an estimate without its evidence counts. Exact rules and a scripted
  five-scan test are in `Store.lua` and `tools/addon_test/scan_steps.lua`. Many scanners
  at different hours is what makes this estimate good; it is the main payoff of pooling,
  and the reset finder and posting advisor both depend on it.
- **Retention:** raw scan summaries one to two weeks; compact daily history long-term.
- **Trust:** reject scans that are incomplete, tiny, or that disagree wildly with other
  scans of the same realm in the same hour. Keep scanner identity on every blob.
- **Coordination:** publish "last complete scan at T" per realm so a fresh scan can be
  declined. Thirty members each scanning every 15 minutes is load for no gain.

Two tools become possible only with pooled data and are not built: a guild demand board
(members post wants, holders see them beside the price) and time-of-day price patterns.

## 8. Desktop app (sidecar)

1. **Upload.** Watch the SavedVariables file; after it changes, send any scan summary the
   server lacks. Key by realm, faction, scan time and scanner so re-sends are harmless.
   Queue while offline.
2. **Download.** Fetch the merged realm file and the board; write each whole and
   atomically. They take effect at the next login or reload.
3. **Freshness.** Carry generated time and scanner count so the addon can show "updated
   20 minutes ago from 3 scanners".
4. **Scan-needed hint.** Carry the server's last-complete-scan time.

Optional: Decades' in-game messaging can pass a compact "prices changed" delta between
online members after a scan, so they see fresh numbers without a reload.

## 9. Updates must not delete market files

On 2026-11-11 a Decades update to v0.2.5 removed `Features/Market.lua`,
`Features/MarketBoard.lua` and their toc lines from an install where they were working.
The Market page vanished, and so did the standalone's pipeline view, which was reading
the board from Decades. The standalone now carries its own board so it survives.

Whatever is integrated needs to be part of the shipped addon and its toc, and any
generated data file needs a toc line that updates preserve and a location that no other
file-replacement step owns.

## 10. Verifying a port

- `node tools/addon_test/run.js`, with `fengari` and `luaparse` on `NODE_PATH`
  (`npm install fengari luaparse` outside the archive). A Lua VM with strict widget mocks
  and an auction-house mock built from a real capture. See the release notes for the
  step count of the build you received.
- What it covers: the scanner (event storm, incomplete rows, bid-only rows, auction house
  closing mid-scan, cooldown, scantest); ladders and market price against independent
  computation; sell-through over scripted scans; trend, timeframes and momentum over
  scripted histories; movers with bait listings; price-source selection; the board
  bundled, absent and older; the heatmap layout; every tool against scripted books with
  known answers; every window page.
- `real_store` and `tools_real` use the author's saved variables and skip without them.
- `tools/addon_test/check_sv.py` runs produced dumps through the Python parser and ingest.
  `check_board.py` covers the exporter and watcher. `python tools/selfcheck.py` checks the
  pipeline.
- None of this checks layout. Someone has to look at it in the client.

## 11. Decisions for the Decades owner

1. May the Market page grow to the full set of pages in section 5?
2. Will you register the recipe catalog as a provider (section 3), or expose it globally?
3. How is an uploader identified: existing Decades login, or a guild key?
4. Where does the merged realm file live in the addon folder, and what is its global?
5. Retention periods and storage budget on the server.
6. Is Auctionator kept as an optional companion for its own posting interface, or dropped?
7. Scan etiquette: who may scan, how often, and whether the app enforces it.
8. Whether to use in-game guild messaging for live deltas.

## 12. Known gaps and unproven points

- Sell-through, time to sell and the reset finder's risk grades need several scans a few
  hours apart. With one scan-day, standard and conservative resets are empty by design.
- Cancelled listings count as sales in sell-through; replicate data cannot tell them apart.
- New supply is measured at scan-day resolution, so within-day resupply is not seen.
- The pipeline (`tools/trend_wow.py`) is weaker than the addon: it ingests no market
  price. Do not copy its ranking.
- Deposits use `C_AuctionHouse.CalculateCommodityDeposit` when it accepts our arguments,
  else 30% of the vendor price, marked inexact.
- Seller lookup on non-commodity gear, and the `observed` recipe provider, are not
  confirmed in the live client.
- Vendor sell price is read from item info return 11, as Auctionator does on this client.
- Trade-goods subclass numbers and time-left band meanings are assumed to match retail.
- Real per-row scan cost, the cost of tool passes over 5,000 items, and saved-variable
  load time with a 5 to 6 MB price store are unmeasured.
- The addon posts nothing itself: it advises a price, the player posts through the game's
  or Auctionator's interface.

## Suggested order

1. Move the files marked "no UI" in section 1 into Decades unchanged; run the rig.
2. Register the recipe provider.
3. Rebuild the item card and the Market landing view on Decades widgets.
4. Add the BUY, SELL, CRAFT and ME views.
5. Server: accept scan summaries, merge, publish the realm file.
6. App: upload, download, freshness, scan-needed hint.
7. Retire the standalone addon and the local watcher.
