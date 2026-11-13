# Decades Market — implementation handoff (revision 3, addon 2.5.1)

For whoever implements this inside Decades: the `DecadesGuildOps` addon, the desktop app
and the cloud server. Written to be read cold. We cannot see the Decades source, so
everything here is stated from our side of the boundary. Where a decision is yours it is
listed in section 11, not guessed.

This revision replaces the one shipped with 2.3.0 on 2026-11-11. If you read that one,
section 0a is the difference. Everything else is restated in full, so this file stands on
its own.

## 0. Read this first

**What you are getting.** A working standalone addon, `addon/DecadesMarket/` version
**2.5.1**, plus its headless test rig and a Python pipeline. The addon scans the auction
house itself, keeps its own price history, and provides a trading window with thirteen
pages and an item card. From 2.4.0 it also **posts auctions**, and from 2.5.0 it
**cancels them**. It does not need Auctionator, Decades, a server or a desktop app.

**What we are asking for.** Three things, in order of value to players:

1. Ship the addon's features inside Decades, on Decades' own widgets and theme.
2. Pool guild members' scans on the server and deliver the merged result to everyone.
3. Stop Decades updates from deleting market files (section 9).

**How to work.** Port the maths, rebuild the window. The files that hold calculations
have no UI in them and can move across almost unchanged. The rig checks a port against
known answers. A port that changes a threshold and still passes was aimed at the wrong
tests.

**Reading order.**

1. This file.
1a. `CALCULATIONS.md` — every formula and threshold in one place, read off the code:
   market price, sell-through, trend, momentum, movers, each tool, the smart price, and
   the pipeline board's index, movers, breakout, squeeze and drain rules (its section 23;
   the scripts themselves are `tools/analyze_wow.py`, `trend_wow.py`, `craft_wow.py`,
   now in this archive). Its section 1 answers "sold, or fell off?".
2. `addon/DecadesMarket/README.md` — the rules for trend, momentum, movers, time to sell,
   price source, the smart price, posting, cancelling and the two flip tools. Those
   sections are the contract.
3. `addon/DecadesMarket/DUMP_SPEC.md` — every field that crosses the boundary. Unchanged
   since 2.3.0.
4. `Tools.lua` top of file — conventions, thresholds (`Tools.K`), reasons, help text.
5. Comment blocks at the top of `ToolsPrice.lua`, `ToolsPost.lua`, `ToolsFlips.lua`,
   and in `Data.lua`, `Store.lua` and `ToolsSell.lua` (the owned-auction block).
6. Run the rig (section 10).

`tools/trend_wow.py` is **not** the mover specification. Port the addon's `Data.lua`.

## 0a. What changed since the 2.3.0 handoff

**New files** (all in the toc; load order in section 1):

| File | What it adds |
| --- | --- |
| `ToolsPrice.lua` | The smart price (`Tools.SmartPrice`) and the market-wide weekday pattern (`Tools.Season`) |
| `ToolsPost.lua` | Posting and cancelling. The only code that creates or cancels an auction |
| `ToolsFlips.lua` | Vendor flips computed in game from the player's own scan, no board needed |
| `UIPost.lua` | The Post page and the Vendor flips page (with its Craft to vendor view) |

**Changed files:**

| File | Change |
| --- | --- |
| `Store.lua` | History kept 35 days, was 21. Nothing else in the store format changed |
| `Data.lua` | `Data.VendorSell(id)` gained a second return: `true` when the answer is nil only because the item's data has not arrived yet |
| `Ledger.lua` | Owned-auction unit price corrected; a partly sold auction no longer marks the rest sold; rows still listed are repaired on the next list |
| `ToolsSell.lua` | Asks the house for the player's auctions (`I.RequestOwned`, `Tools.RefreshOwned`); `I.OwnedUnit`; cancelled rows leave the list at once |
| `ToolsCraft.lua` | `Tools.CraftCost(itemID)` (exact material cost, used as a price floor) and `Tools.CraftFlips` (craft and sell to a vendor) |
| `Tools.lua` | New thresholds (`SMART_*`, `SEASON_*`, `POST_CONFIRM_LOW`, `OWNED_*`, `CANCEL_*`, `FLIP_*`, `CFLIP_*`), new reasons, help text for `sell` and `flips`, saved `postLog` and `cancelLog` |
| `UI.lua`, `UISell.lua` | Two new rail entries; cancel and repost buttons on My auctions |
| `Core.lua`, toc | Version |

**Reversed statement.** The 2.3.0 handoff said "the addon posts nothing itself". That is
no longer true. See section 2a and decision 9.

**Three client facts that cost a broken release** (2.5.0 shipped a cancel button that did
nothing; all three are fixed in 2.5.1 and written up in section 4):

1. The client does not know the player's auctions until `QueryOwnedAuctions` is answered.
2. `buyoutAmount` on an owned auction is a unit price while active and a total once sold.
3. A partly sold commodity auction is several rows under one auction ID.

**Rig.** 806 steps, 0 fail, 0 skipped, run on 2026-11-12 against exactly the files in this
archive. It was 500-odd at 2.3.0.

**Live status.** 2.3.0 was run live and accepted. 2.4.0 and 2.5.0 were each installed in
the author's client. 2.5.0's cancel failed there, which is what produced 2.5.1. 2.5.1 is
installed in the author's client and its code is identical to this archive; we have no
written live result for it yet. Treat posting, cancelling, the smart price and both flip
tools as rig-verified against mocks, not live-verified (section 12).

## 1. What exists

Load order is the toc's.

| File | What it holds | UI in it? |
| --- | --- | --- |
| `Core.lua` | Saved-variable defaults, message bus, events, `/dm` router, money and item helpers | no |
| `BoardData.lua` | Generated copy of the pipeline's analysis board | no |
| `Data.lua` | Every lookup: price, market price, history, trend, timeframes, momentum, movers, breadth, categories and heatmap weights, depth, time to sell, bags, vendor buy and sell, price-source selection, board | no |
| `Store.lua` | Own price store: 35 days of daily history, market price, depth ladder, sell-through accumulators | no |
| `Dump.lua` | SavedVariables dump for the pipeline | no |
| `Scan.lua` | Full auction-house scan, cooldown, `/dm scantest` | no |
| `Ledger.lua` | Merchant prices; ledger from the player's owned auctions | no |
| `Tools.lua` | Tool conventions, thresholds, reasons, help text, saved state | no |
| `ToolsBook.lua` | Sniper, reset finder | no |
| `ToolsSell.lua` | Posting advisor, undercut check, owned-auction reading and refresh | no |
| `ToolsPrice.lua` | Smart price, weekday pattern, auction durations | no |
| `ToolsPost.lua` | Post and cancel (checks, the calls, logs) | no |
| `ToolsFlips.lua` | Vendor flips | no |
| `ToolsLists.lua` | Shopping lists | no |
| `Recipes.lua` | Generated recipe table (2,308 recipes, about 160 KB) | no |
| `ToolsCraft.lua` | Craft profit, craft cost, crafting flips, recipe-provider registry | no |
| `ToolsSellers.lua` | Seller lookup for one item | no |
| `ToolsPortfolio.lua` | Portfolio series and realised results | no |
| `Watch.lua` | Watchlist and price alerts | no |
| `Tooltip.lua` | Item tooltip lines | tooltip only |
| `Minimap.lua` | Minimap button | yes |
| `UI.lua`, `UITools.lua`, `UIResets.lua`, `UISell.lua`, `UIPost.lua`, `UICraft.lua`, `UICard.lua` | The window | yes |

`ToolsPost.lua` has no widgets but reads `AuctionHouseFrame` (to tell whether the house is
open, and to hand a pending post to the default frame). Its two calls must be reached
from a real click; see section 2a.

## 2. The tools, and what each needs

All work from one player's own scans. Prices are judged against the **market price**
(the price at which the cheapest 15% of listed quantity is covered), never the single
cheapest listing. The auction house takes 5% of a sale.

| Tool | Function | What it answers | Needs |
| --- | --- | --- | --- |
| Sniper | `Tools.Sniper` | What is listed well under market, and the profit after the cut | A scan under 12 h old; 3 recorded days |
| Reset finder | `Tools.Reset` | Thin books where buying the cheap end and re-listing pays; every clear-to level, partial-success model, risk grade | A scan; evidence improves it |
| Vendor flips | `Tools.VendorFlips` | Merchant items that sell for more on the auction house, by profit per day | A scan; merchant prices (recorded when a merchant window opens) |
| Crafting flips | `Tools.CraftFlips` | Recipes whose product a vendor pays more for than the materials cost, and how many crafts still pay | A scan and a recipe provider |
| Shopping lists | `Tools.Shopping`, `ListSet` … | Buy-now lines under each item's price cap | A scan |
| Smart price | `Tools.SmartPrice`, `Tools.SmartList` | One posting price with every reason that moved it, a quantity, a duration, a confidence word, and POST / HOLD / VENDOR / NO BOOK | A scan; history and evidence improve it |
| Posting advisor | `Tools.Post`, `Tools.PostList` | Price options from quick to patient with time to sell and net proceeds | A scan |
| Post | `Tools.PostInfo`, `PostCheck`, `PostNow` | Posts one item per click at the chosen price | Auction house open; a click |
| Undercut check | `Tools.Undercuts` | Which of your auctions are no longer cheapest | A scan and the owned-auction list |
| Cancel | `Tools.CancelCheck`, `CancelNow` | Cancels one owned auction per click, after stating the cost | Auction house open; a live owned list; a click |
| Weekday pattern | `Tools.Season` | Which weekdays run dear or cheap across the whole market | 2 full weeks of scan days; 4 for a grade of good |
| Craft profit | `Tools.Craft`, `Tools.CraftCost` | Material cost against sale price; profitable-now flag | A scan and a recipe provider |
| Seller view | `Tools.SellersLookup`, `Tools.Sellers` | Who sells an item and whether one seller holds the cheap end | One search request, auction house open |
| Portfolio | `Tools.Portfolio` | Gold, bags and auctions over time; realised sales | A scan; two scan-days for change |

Conventions every tool follows (stated at the top of `Tools.lua`):

- One plain table back. `ok == false` carries `reason` (a sentence for the player) and
  `reasonKey` (a key of `Tools.REASONS`). `ok == true` with empty `rows` carries `note`.
- Cached until `PRICES_CHANGED`, `DUMP_DONE`, `LEDGER_CHANGED`, `SETTINGS_CHANGED`.
  `TOOLS_CHANGED` fires when saved tool state changes. `Tools.Season` keeps its own cache
  keyed on scan day, last scan and source, because the tools cache is dropped on every
  owned-auction update.
- `Tools.HELP[tool]` holds a title, an explanation, and text for each unavailable state
  and each action. The window shows these on mouseover. Reuse them verbatim.
- Each tool has a `/dm` subcommand and a `Tools.XxxLines(result)` text form. New since
  2.3.0: `/dm smart`, `/dm flips`, `/dm craftflip`, `/dm season`.

The reset finder is deliberately not filtered for the player's protection. Every
candidate is computed; the mode (conservative, standard, aggressive) changes only the
filter and the assumed re-list price. Keep that. Unproven candidates must stay visible
and must not lead the default sort.

### 2a. Posting and cancelling: the rules to keep

These are the only two actions in the addon that spend the player's gold or move their
items. Port them as they are.

- **One click, one call.** The client accepts `PostItem`, `PostCommodity` and
  `CancelAuction` only from a hardware event. The click handler calls `Tools.PostNow` or
  `Tools.CancelNow` directly, with no timer and no queue in between. There is no "post
  everything" and no "cancel all", and there cannot be.
- **Check, then act.** `PostCheck` and `CancelCheck` send nothing. They return either a
  refusal with a reason or the numbers that will be used. `PostNow` and `CancelNow` call
  the check first, every time.
- **Refusals are sentences.** Posting: `noapi`, `nobagitem`, `cantsell`, `bound`,
  `ahclosed`, `scanning`, `throttled`, `badprice`, `badqty`, `nomoney`, `posterror`.
  Cancelling: `noapi`, `ahclosed`, `scanning`, `throttled`, `noowned`, `noauction`,
  `sold`, `notloaded`, `nomoney`, `cancelerror`. The button is never hidden; when it is
  off, hovering says which of these applies.
- **Second click.** A post that nets less than a vendor pays, or is under half the market
  price, comes back as `reasonKey == "confirm"` with the sentence; the same call with
  `opts.confirmed = true` posts. Every cancel takes a second click: the first states that
  the deposit is lost, any cancel fee the client reports, and that the items return by
  mail.
- **Never cancel from a list that is not live.** `I.Owned()` returns its source: `live`,
  `saved` or `ledger`. A cancel from anything but `live` is refused with `notloaded` and
  the list is asked for.
- **Logs.** `DecadesMarketDB.tools.postLog[realm]` holds up to 300 rows
  `{ t, i, q, u, d, k, mk, ch, dep }` (time, item, quantity, unit price, duration index,
  which price option, market and cheapest price at the time, deposit).
  `cancelLog[realm]` holds up to 100 rows `{ t, a, i, q, u, cost }`. The post log exists
  so the smart price can be checked against what actually sold. That check has not been
  done.

### 2b. The smart price

`Tools.SmartPrice(id, { count = n })`. The full rule is in the addon README under
"Posting" and at the top of `ToolsPrice.lua`; the two texts are kept the same. In short:
an anchor read off the order book, small capped adjustments (momentum or trend, place in
the 14-day range, supply, weekday; together never more than +10% or −4%), a snap back
onto the book so the price never ties an existing listing, then guards (never under the
player's own cheapest auction, never where a vendor pays more, never under exact material
cost after the cut unless that is over 25% above market, and stepped down until the
sell-time estimate fits one auction).

Every adjustment is returned with its size, and the window shows each one. Keep that: a
price the player cannot see the reasons for is not this feature. None of the thresholds
has been measured against real sales.

## 3. The recipe provider — the one thing we need from Decades' data

Unchanged since 2.3.0, and now feeding three things instead of one: craft profit, the
smart price's material-cost floor, and crafting flips.

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

`known` matters more than it did: `/dm craftflip known` and the Craft to vendor view
filter on it, and the bundled catalog cannot say. Decades can.

## 4. Facts about the client that shape the design

From a passive probe run in the live client (its shapes are reproduced in
`tools/addon_test/ahmock.lua`), from live scans, and, for posting and owned auctions,
from Auctionator 346's source for this client plus the author's saved variables.

**Scanning**

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
- Whether the server answers a scan away from an auctioneer is untested. `/dm scantest`
  runs the experiment and saves its verdict in `DecadesMarketDB.scan.tests`.

**Owned auctions (new)**

- The client only knows the player's auctions after `C_AuctionHouse.QueryOwnedAuctions({})`
  has been answered. Until then `GetNumOwnedAuctions()` reads 0, no
  `OWNED_AUCTIONS_UPDATED` follows a post, and `CancelAuction` is refused. The default
  Auctions tab and Auctionator's Cancelling tab send the query, which is why this stayed
  hidden. We send it on `AUCTION_HOUSE_SHOW`, after `AUCTION_HOUSE_AUCTION_CREATED`, after
  `AUCTION_CANCELED`, and when My auctions is shown with a list older than 20 s; never
  during a scan or while throttled; at most once every 3 s.
- An unloaded list also reads 0. `HasFullOwnedAuctionResults()` tells the two apart.
- `GetOwnedAuctionInfo(i).buyoutAmount` is the price of **one unit** while the auction is
  active and the **whole payment** on a sold row (`status == 1`). Evidence: 27 units of a
  10c fish read 10 while active and 60 for the 6 that sold. `I.OwnedUnit` is the one place
  that encodes this.
- A partly sold commodity auction appears as one sold row per sale plus an active row for
  the remainder, all under one `auctionID`. The active row is the auction; only when none
  exists is it sold.
- `CanCancelAuction` and `GetCancelCost` are treated as optional: used when present.

**Posting (new)**

- Commodities: `PostCommodity(itemLocation, duration, quantity, unitPrice)`. Everything
  else: `PostItem(itemLocation, duration, quantity, bid, buyout)` with bid nil and buyout
  per item. `GetItemCommodityStatus(location) == 2` means commodity.
  `GetAvailablePostCount(location)` is how many the house will take.
- `duration` is an index 1 to 3. On builds 16000 to 19999 (this client) the three are
  2 h, 8 h and 24 h; elsewhere 12, 24, 48. `I.Durations()` holds this. Every rule that
  mentions "one auction" or "the longest auction" reads it from there.
- A `true` return from a post call means the default frame wants to confirm; we pass the
  same arguments to the sell frame's `CachePendingPost`, as Auctionator does.
- Which event reports a refused post on this client is not known. A refusal is detected
  only when the call raises; otherwise the game prints its own error line.
- Deposits: `CalculateCommodityDeposit(itemID, duration, quantity)` and
  `CalculateItemDeposit(location, duration, quantity)`.

**Everything else**

- An addon cannot use the network. Data leaves the game only through SavedVariables,
  written on `/reload` or logout, and enters only at login or `/reload`. That is why a
  sidecar is required and why pooled data always lags by a session.
- A vendor's sell price comes from item info and needs the item cached. For a list that
  touches hundreds of products (crafting flips) many are not cached on the first pass;
  `Data.VendorSell` now says so in its second return, the tool counts them in `loading`
  and does not cache an answer that is still filling in.

## 5. Addon integration

Suggested mapping. The standalone window uses a left rail grouped MARKET, BUY, SELL,
CRAFT, ME; the Decades Market page had five tools (Movers, Flips, Crafts, Breaks,
Squeeze).

| Standalone page | In Decades |
| --- | --- |
| Pulse, category heatmap | Market landing view |
| Movers (live, with momentum sorts) and the pipeline's flips and squeeze | The Movers tool; live data first, board as a second source |
| Sniper, Resets, Shopping lists | New BUY views |
| Vendor flips, with its Craft to vendor view | Replaces the board-fed Flips tool; needs no board |
| Post (bags at a smart price, price options, quantity, duration, the Post button) | New SELL view; the main one |
| Bags with posting advice | SELL view; its "Post this" opens Post |
| My auctions with undercuts, cancel and repost | SELL view |
| Craft profit | Replaces the board-fed Crafts tool |
| Portfolio, Watchlist | New ME views |
| Item card: timeframes, range bar, chart, order book, time to sell, sellers, watch | The Market item card |
| Tooltip lines | Decades' tooltip post-call |
| Watch and undercut alerts, minimap status | Decades' notification and launcher surfaces |
| Scan button, progress, price-source control | Market page header and settings |

Page ids, for anything that deep-links: `pulse`, `movers`, `sniper`, `reset`, `flips`,
`lists`, `sell`, `bags`, `auctions`, `craft`, `portfolio`, `watch`, `settings`. Saved ids
from older builds still open: `ledger` is `auctions`, `snipe` is `sniper`, `post` is `sell`.

This is well past the earlier "no sixth tab" rule in `DECADES_HANDOFF.md`. See section 11.

Lessons already paid for:

- Draw arrows and other symbols from solid textures. The game font cannot be trusted for
  arrow, ellipsis or dash glyphs.
- List rows are recycled. Create decorations once and reset every one on each paint.
- Scanning is an explicit player action with the auction house open and a cooldown (15
  minutes, Auctionator's figure; the server's real limit is unknown). Batch the read on a
  timer with a time budget per tick (ours targets 5 ms).
- Unit price is `floor(buyout / count)` for scan rows. Skip rows with buyout 0 (bid only).
  Owned-auction rows are different; see section 4.
- The lowest listing is a bad price; one bait listing moves it wildly. Rank and trend on
  the market price, with a floor on listings.
- "Vendor price" is two numbers: what a merchant charges (`Data.VendorBuy`, for flips and
  reagents) and what a vendor pays the player (`Data.VendorSell`, for "vendor it" and
  crafting flips).
- A new store has one recorded day, so everything needing history is empty. Say why on
  screen. Past days from Auctionator, when present, are carried in at read time as
  imported history and never written to our store.
- Every disabled control says why it is disabled. Every tool says what it needs when it
  has nothing to show.
- Random-suffix gear is keyed by itemID alone, so its variants share one entry.
- Soulbound bag items must be excluded from posting advice and from Post.
- After a post, hold the Post button until the bags have caught up, or a second click
  posts from a count that is already gone.
- A repost can be asked for before the cancelled items are out of the mailbox. The Post
  page opens on the item anyway, says none is in the bags, and switches Post on when it
  arrives. That pin is for the session only and must not survive a reload.
- A mock that hands over data the real client makes you ask for will pass a feature that
  cannot work. That is how 2.5.0's cancel shipped. When you mock a client call, mock the
  request as well as the answer.

Day-count rules (normative, from the README): trend needs 3 recorded days in 14; momentum
needs 4 in a run with no gap longer than 3; a last scan older than 3 scan-days gives no
momentum reading; the weekday pattern needs 2 full weeks and reaches good at 4.

## 6. Data that crosses the boundary

Exact field lists are in `addon/DecadesMarket/DUMP_SPEC.md`. This section is the map.
Nothing in the dump changed between 2.3.0 and 2.5.1.

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
time, number of scans and distinct scanners. **Carry at least 35 scan-days of daily
history** (it was 21): the weekday pattern reads five 7-day blocks.

**Analysis board.** `BoardData.lua` in the addon folder, generated by
`tools/export_decades.py`. `Data.Board()` uses the newer of it and the global
`DecadesMarketBoard` by `generatedAt`. A sync step must overwrite the installed
`BoardData.lua`. With vendor flips now computed in game, the board matters less than it
did; it still feeds Movers > Pipeline and the tooltip's pipeline signals.

**Not uploaded today, worth considering.** The post log (section 2a). Pooled across
members and joined to the ledger's sold and expired outcomes, it is the only way to
measure whether the smart price is any good. It is personal trading data, so it is a
decision (section 11), not a default.

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
  6 min apart. Twelve hours is a cap on how far a book may have turned over; it is not an
  auction length (the shortest auction on this client is 2 h, and the time-left bands
  already keep expiries out). Count a level only when the interval is shorter than the
  lower bound of its shortest time-left band (under 30 min, 30 min to 2 h, 2 to 12 h,
  over 12 h). Accumulate units per hour per price bucket with pairs and hours behind it.
  Time to sell at a price = units queued at or below it, divided by the summed rate of
  the buckets up to it. Never publish an estimate without its evidence counts. Exact
  rules and a scripted five-scan test are in `Store.lua` and
  `tools/addon_test/scan_steps.lua`. Many scanners at different hours is what makes this
  estimate good; it is the main payoff of pooling, and the reset finder, the posting
  advisor and now the smart price's anchor, quantity and duration all depend on it.
- **Weekday pattern:** `Tools.Season` computes it in the addon from daily history, so the
  server only has to deliver 35 days. With one scanner it says nothing for two weeks and
  takes four to reach good. Pooled history gets a new member there on their first sync.
- **Retention:** raw scan summaries one to two weeks; compact daily history long-term,
  at least 35 days served.
- **Trust:** reject scans that are incomplete, tiny, or that disagree wildly with other
  scans of the same realm in the same hour. Keep scanner identity on every blob.
- **Coordination:** publish "last complete scan at T" per realm so a fresh scan can be
  declined. Thirty members each scanning every 15 minutes is load for no gain.

Tools that become possible only with pooled data and are not built: a guild demand board
(members post wants, holders see them beside the price), time-of-day price patterns, and
a measured smart price (from pooled post logs).

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
the board from Decades. The standalone now carries its own board so it survives. The
author's install is now on Decades v0.2.6 and the market files are still absent from it.

Whatever is integrated needs to be part of the shipped addon and its toc, and any
generated data file needs a toc line that updates preserve and a location that no other
file-replacement step owns.

## 10. Verifying a port

- `node tools/addon_test/run.js`, with `fengari` and `luaparse` on `NODE_PATH`
  (`npm install fengari luaparse` outside the archive). A Lua VM with strict widget mocks
  and an auction-house mock built from a real capture. About three minutes.
  `--only=<scenarios>` runs a subset.
- Expected on this archive: **806 pass, 0 fail**. `real_store` and `tools_real` use the
  author's saved variables, which are not in the archive, so on your machine they skip and
  the pass count is lower. Zero failures is the bar.
- What it covers: the scanner (event storm, incomplete rows, bid-only rows, auction house
  closing mid-scan, cooldown, scantest); ladders and market price against independent
  computation; sell-through over scripted scans; trend, timeframes and momentum over
  scripted histories; movers with bait listings; price-source selection; the board
  bundled, absent and older; the heatmap layout; every tool against scripted books with
  known answers; every window page.
- New since 2.3.0: the smart price step by step (tight book, bait, ties, each adjustment
  and its cap, each guard); the weekday pattern on scripted weeks; posting against a mock
  of the post calls (every refusal, the second click, one call per click, the right call
  per item kind); cancelling against a mock of `CancelAuction` (refusals, confirm, one
  call per click, the row leaving and returning); owned auctions with a mock that counts
  `QueryOwnedAuctions` and returns unit prices the way the client does; partly sold
  auctions; vendor flips and crafting flips against scripted books.
- `tools/addon_test/check_sv.py` runs produced dumps through the Python parser and ingest.
  `check_board.py` covers the exporter and watcher. `python tools/selfcheck.py` checks the
  pipeline.
- None of this checks layout, and none of it proves the real client accepts a post or a
  cancel. Someone has to do both in the client.

## 11. Decisions for the Decades owner

1. May the Market page grow to the full set of pages in section 5?
2. Will you register the recipe catalog as a provider (section 3), or expose it globally?
3. How is an uploader identified: existing Decades login, or a guild key?
4. Where does the merged realm file live in the addon folder, and what is its global?
5. Retention periods and storage budget on the server (at least 35 days of daily history).
6. Is Auctionator kept as an optional companion, or dropped? With Post and cancel built
   in, the addon no longer needs it for anything.
7. Scan etiquette: who may scan, how often, and whether the app enforces it.
8. Whether to use in-game guild messaging for live deltas.
9. **New.** Does Decades want to post and cancel auctions at all? It is the first feature
   here that spends a member's gold. If yes, keep the rules in section 2a. If no, ship
   everything up to the smart price and leave the Post button out; the price, its reasons
   and the advice all work without it.
10. **New.** May post logs be uploaded and pooled, so the smart price can be measured?

## 12. Known gaps and unproven points

- **Not confirmed live in 2.5.1:** that a post is accepted, that a cancel is accepted and
  the items arrive by mail, that the repost flow completes, that `CachePendingPost` is the
  right hand-off when the post call returns true. All are built from Auctionator's source
  for this client and tested against mocks.
- The smart price's thresholds are judgement, not measurement. Nothing here is a forecast.
- A refused post is only detected when the call raises (section 4).
- Ledger rows of auctions that ended before 2.5.1 keep a unit price that was divided by
  the quantity. Rows still listed are corrected when the list next arrives.
- Vendor flips can only list items whose merchant the player has visited.
- Crafting flips use the last scan's prices while other players buy the same materials;
  the recipe catalog may list a product this server prices differently; skill-ups,
  cooldowns and crafting time are not counted.
- Sell-through, time to sell and the reset finder's risk grades need several scans a few
  hours apart. With one scan-day, standard and conservative resets are empty by design.
- Cancelled listings count as sales in sell-through; replicate data cannot tell them apart.
- New supply is measured at scan-day resolution, so within-day resupply is not seen.
- The pipeline (`tools/trend_wow.py`) is weaker than the addon: it ingests no market
  price. Do not copy its ranking.
- Deposits use the client's calculation when it accepts our arguments, else 30% of the
  vendor price, marked inexact.
- Seller lookup on non-commodity gear, and the `observed` recipe provider, are not
  confirmed in the live client.
- Vendor sell price is read from item info return 11, as Auctionator does on this client.
- Trade-goods subclass numbers and time-left band meanings are assumed to match retail.
- Real per-row scan cost, the cost of tool passes over 5,000 items, and saved-variable
  load time are unmeasured. With 35 days kept the price store runs to roughly 7.5 MB at
  one scan a day, more with several.

## Suggested order

1. Move the files marked "no UI" in section 1 into Decades unchanged; run the rig.
2. Register the recipe provider.
3. Rebuild the item card and the Market landing view on Decades widgets.
4. Add the BUY, CRAFT and ME views, and the SELL views without a Post or cancel button.
5. Decide 9. If yes, add Post and cancel last, and test both in the client before release.
6. Server: accept scan summaries, merge, publish the realm file with 35 days of history.
7. App: upload, download, freshness, scan-needed hint.
8. Retire the standalone addon and the local watcher.
