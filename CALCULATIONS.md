# Decades Market — every calculation (addon 2.5.1)

Companion to `HANDOFF_INTEGRATION.md`. That file says what to build and where; this one
states every number the addon and the pipeline compute, as the code computes it today.
Each section names the file and function it was read from. Where this file and the code
disagree, the code wins and this file is wrong.

Written 2026-11-12 against the files in this archive. Nothing in the code changed to
produce it.

## Contents

0. Conventions
1. Sold, or fell off? What can and cannot be known
2. From a scan to one record per item
3. The price store (daily history)
4. Price source and imported history
5. Sell-through and time to sell
6. Trend, timeframes, range
7. Momentum
8. Movers, breadth, market median
9. Categories and the heatmap
10. Shared helpers used by the tools
11. Sniper
12. Reset finder
13. Vendor flips
14. Crafting: profit, cost, crafting flips
15. Shopping lists
16. Posting advisor
17. Smart price
18. Weekday pattern
19. Post and cancel checks
20. Undercut check
21. Seller view
22. Portfolio, bags, tooltip verdict, watch alerts
23. The pipeline board: index, movers, breakout, squeeze, drain, flips, crafts
24. Every threshold in one table
25. Known biases and things never measured
26. Open decision: should momentum use sales?

---

## 0. Conventions

- Money is copper. Prices are per unit unless a line says total.
- **Cut.** The auction house keeps 5%. `Cut(p) = floor(p * 0.95)` (`Tools.I.Cut`).
  "Net" always means after the cut.
- **Undercut.** `Undercut(p) = max(1, p - max(1, floor(p * 0.01)))`: 1% under, at least
  1 copper under, never below 1 (`Tools.I.Undercut`).
- **Scan-day.** `day = floor((time() - DAY0) / 86400)`, `DAY0 = 1577836800`
  (2020-01-01T00:00:00Z), or Auctionator's `SCAN_DAY_0` when Auctionator is loaded. History
  has one point per day a scan happened. Days without a scan are gaps; nothing fills them.
- **Recorded day** = a scan-day that has a stored price.
- **Window** = recorded days newer than `today - 14`.
- Percent changes are true percentages, `(new - old) / old * 100`, except where a log
  rate is named (momentum, weekday pattern, the pipeline's z-score).
- Median of an even count is the mean of the two middle values. Percentile is nearest
  rank: `sorted[max(1, min(n, ceil(p * n)))]`.

---

## 1. Sold, or fell off? What can and cannot be known

| Whose auction | What the client gives | What the addon can say |
| --- | --- | --- |
| Your own | `GetOwnedAuctionInfo(i).status`: 0 active, 1 sold | **Sold** is certain. An auction that leaves the list without a sold row is **gone**: expired or cancelled, the ledger cannot tell which (`Ledger.lua`) |
| Everyone else's | Full-scan rows carry no auction ID and no owner | Nothing per auction. Only an estimate per item from how its price ladder changed between two scans (section 5) |

For other players' auctions the estimate separates the three ways a unit can leave:

- **Expired**: excluded. A price level is only counted when the time between the two
  scans is shorter than the minimum time left on the auctions at that level, so none of
  them could have expired.
- **Cancelled**: not excluded. It is counted as a sale. The estimate reads high where
  people cancel and repost.
- **Sold and replaced**: missed. Units posted between scans hide units that sold. The
  estimate reads low where people relist heavily.

Momentum (section 7) does **not** use any of this. It is price speed, acceleration and
streak, plus a 10% term for the change in listed quantity. "Fewer listed" is all that
term knows; it does not know why.

---

## 2. From a scan to one record per item

`Scan.lua`. One `C_AuctionHouse.ReplicateItems()` request, rows read in batches.

**Per row.** `GetReplicateItemInfo(i)`, 0-based: count = return 3, buyout = return 10
(whole stack), itemID = return 17. Rows with buyout 0 (bid only) are skipped.

```
unit = floor(buyout / count)
```

**Per item** (`Add`). Keyed by itemID alone, so random-suffix gear shares one record.

```
min  = lowest unit price seen
qty  = sum of count            (units listed)
n    = number of rows          (auctions listed)
P,Q,T = the 40 cheapest distinct unit prices, units at each, and the shortest
        time-left band among the auctions at each
tail  = everything dearer than those 40: tailq units, tailmin lowest price, tailt band
```

Time-left bands: 0 = under 30 min, 1 = 30 min to 2 h, 2 = 2 to 12 h, 3 = over 12 h.

**Market price** (`Scan.MarketPrice`). The unit price at which the cheapest 15% of listed
quantity is covered:

```
target = max(1, ceil(qty * 0.15))
walk P cheapest first, summing Q; market = the first P[i] where the sum >= target
if the 40 kept levels never reach target: market = P[last]   (a lower bound)
```

One bait listing cannot move it unless it holds 15% of the units. This is the price every
ranking, trend and tool is measured against. The single cheapest listing is only shown.

**Ladder** (`Store.BuildLadder`), saved per item as `ld`, at most 12 levels:

- The first 8 distinct prices are kept exactly (`lx` = how many).
- Everything dearer is summed into premium buckets by `price / min`:

| Bucket | Range over the cheapest price |
| --- | --- |
| 1 | up to +2% ("at market") |
| 2 | up to +10% |
| 3 | up to +25% |
| 4 | up to +50% |
| 5 | beyond |

  A dearer level that would fall in bucket 1 is filed in bucket 2 (bucket 1 is covered by
  the exact levels). Each bucket is stored as its lowest price, its total units and its
  shortest band. The tail goes into the bucket of `tailmin`.
- `lt` = the epoch second the ladder was read.

`Data.UnitsAtOrBelow(id, price)`: sum of units at levels with `price <= X`. A bucket
counts whole once its lowest price is within reach, so past the exact levels the answer
is an upper bound and is flagged inexact.

**Scanner pacing** (not a market calculation, listed for completeness): 15 min cooldown,
batches start at 500 rows and scale by x0.8 / x1.2 between 100 and 8,000 to hold about
5 ms per batch, one batch every 0.03 s, results committed 600 items at a time.

---

## 3. The price store (daily history)

`Store.SetPrice`. Auctionator's layout plus our own fields. Per item:

| Field | Meaning | Update rule on each scan |
| --- | --- | --- |
| `m` | current cheapest unit price | replaced |
| `h[day]` | highest of the day's cheapest prices | `max` |
| `l[day]` | lowest of the day's cheapest prices | `min`; stored only when below `h[day]` |
| `a[day]` | most units listed that day | `max` |
| `n` | auctions listed | replaced |
| `mk` | market price | replaced |
| `k[day]` | market price at the last scan of the day | last scan wins |
| `c[day]` | auction count at the last scan of the day | last scan wins |
| `ld`, `lx`, `lt` | ladder | replaced |
| `sr` | sell-through accumulators | section 5 |

A day's price everywhere below is `l[day] or h[day]` ("the day's low": what a buyer could
have paid). History older than 35 days is pruned; an item with no days left is dropped.

---

## 4. Price source and imported history

`Data.PriceTable`, setting `priceSource`.

- `auto` (default): per item, whichever of our entry and Auctionator's has the later
  latest recorded day. Ours on a tie. Auctionator fills items we never scanned.
- `own`: only items we scanned, only our prices.
- `auctionator`: Auctionator's table only.

**Imported history** (`Data.CarryOver`, in `auto` and `own`). For an item we hold, every
day Auctionator recorded that we lack, and that is not later than our own latest day, is
added to a read-only view: its `h`, `l` and `a`. Our store is never written and a day we
recorded is never replaced. Imported days have no `k` or `c`, so any series containing
them is measured on the day's low (section 6, "Which price").

---

## 5. Sell-through and time to sell

`Store.UpdateSell`, `Data.SellRate`, `Data.TimeToSell`, `Tools.I.SellHours`.

### 5.1 Updating the accumulators (once per item per scan)

Compare the item's stored ladder (previous scan) with the new scan. `dt` = hours between.

1. `dt > 12` -> discard ("far"). `dt < 0.1` -> discard ("near"). No previous ladder ->
   "noprev".
2. For each previous level:
   - bucket `b` = `BucketOf(level price, previous cheapest price)` (table in section 2).
   - `tot[b] += qty`.
   - The level is **testable** when `dt < lower bound of its band`: 0 / 0.5 / 2 / 12 hours
     for bands 0 / 1 / 2 / 3. So band 0 is never testable, and a band-3 level is testable
     for any pair that survived step 1.
   - If testable, find `curq`, the units at the same place in the new scan:
     - exact level: units at exactly that price now (0 when the price is gone). Unknown
       when the price lies beyond the new scan's 40 kept levels and a tail exists.
     - bucket level: units in the new scan above the previous top exact price whose
       bucket (against the previous cheapest) is `b`. Unknown when the new scan has a
       tail and either `b` is 5 or the bucket's upper bound is past the new scan's
       deepest kept price.
   - When `curq` is known: `valid[b] += qty`, and if `qty > curq`,
     `gone[b] += qty - curq`. Units that appeared are never subtracted.
3. For each bucket with `tot[b] > 0` and `valid[b] * 2 >= tot[b]` (at least half its
   units testable):
   - once per pair, first fade all old evidence: `s[k] *= 0.95`, `h[k] *= 0.95` for all k.
   - `s[b] += gone[b] * tot[b] / valid[b]`   (scaled up to the whole bucket)
   - `h[b] += dt`
   - `c[b] += 1`
4. If any bucket counted, `n += 1` ("used"); otherwise "expiring".

`s` = estimated units sold, `h` = hours observed, `c` = pairs, per bucket; `n` = pairs
overall. The reason each pair was or was not used is tallied per realm.

### 5.2 Rate and grade

```
rate[b] = s[b] / h[b]                       units per hour, nil when h[b] = 0
item: pairs = n, hours = max over buckets of h[b], sold = sum of s[b]
```

| Grade | Pairs | Hours | Units sold |
| --- | --- | --- | --- |
| good | >= 8 | >= 24 | >= 30 |
| fair | >= 4 | >= 8 | >= 10 |
| low | >= 2 | >= 2 | >= 3 |
| none | otherwise | | |

### 5.3 Time to sell at a price

`Data.TimeToSell(id, price)`:

```
b      = BucketOf(price, cheapest listing of the latest scan)
rate   = sum of s[k]/h[k] for k = 1..b with h[k] > 0
sold   = sum of s[k] over the same k;  hours = max h[k];  pairs = max c[k]
grade  = Grade(pairs, hours, sold)
queue  = UnitsAtOrBelow(id, price)
hours to sell = max(1, queue) / rate
```

Buyers take the cheapest units first, so a unit waits behind everything at or below its
price. Returns no number, with a reason, when the grade is none or the rate is 0.

`Tools.I.SellHours(id, price, queueAhead, extra)` is the same rate with the caller's own
queue: `hours = (max(1, queueAhead) + max(0, extra - 1)) / rate`. The tools use it to
price "my N units behind what is already there".

### 5.4 The card's price points (`Data.SellPlan`)

Time to sell is shown at: 1% under the cheapest (`cheapest - max(1, floor(cheapest*0.01))`),
the cheapest, the market price (only when it is more than 2% above the cheapest),
`cheapest * 1.10` and `cheapest * 1.25`, each floored, duplicates dropped.

Wording (`Data.EtaText`): under 0.75 h "under an hour"; under 1.5 h "about 1h"; under 20 h
"about Nh"; under 36 h "about a day"; under 14 days "N-(N+1) days"; under 60 days "about
N weeks"; else "over 2 months".

---

## 6. Trend, timeframes, range

`Data.lua`. One classification feeds every arrow, word and percentage.

### 6.1 Which price (`Data.BasisOf`)

- **Market basis**: the item's latest `minRun` consecutive recorded days in the window
  all have `k[day]` (and the newest of them is the item's latest day). The series is those
  days' market prices and "now" is the newest of them. `minRun` is 3 for trend and
  timeframes, 4 for momentum.
- **Min basis**: otherwise. The series is every window day's low and "now" is `m`.

The two kinds of number are never mixed in one series.

### 6.2 Classification (`Data.Classify(days, price)`)

```
n < 1 or no price            -> "Not enough history"
lo, hi  = lowest low / highest high across the days and the current price
avg     = mean of the days' lows
pos     = (price - lo) / (hi - lo), clamped 0..1     (0.5 when hi = lo)
spread  = (hi - lo) / lo
n < 3                        -> "Not enough history" (lo/hi/pos still returned)
prev    = mean of the lows of all days EXCEPT the newest
pct     = (price - prev) / prev * 100
```

| `abs(pct)` | Strength | Direction |
| --- | --- | --- |
| under 3 | 0 | flat |
| 3 to under 8 | 1 drifting | sign of pct |
| 8 to under 20 | 2 moving | sign of pct |
| 20 or more | 3 spiking | sign of pct |

Word, first match wins:

1. strength 3 -> "Spiking" (up) / "Crashing" (down)
2. `spread >= 0.10` and `pos <= 0.20` and direction is not down -> "Bottoming"
3. `spread >= 0.10` and `pos >= 0.80` and direction is not up -> "Topping"
4. strength 0 -> "Flat"
5. up: "Drifting up" (1) / "Rising" (2); down: "Drifting down" / "Falling"

`Data.Trend(id)` = Classify on the item's basis series, plus `age` (scan-days since last
seen). An arrow for a price more than 3 scan-days old is drawn faded.

`Data.Stats(id).trend` is an older three-way field (`up` at `changePct >= 5`, `down` at
`<= -5`) kept for the dump. Screens use Classify.

### 6.3 Change over 1, 3, 7, 14 days (`Data.ChangesFor`)

```
target = latest recorded day - N
ref    = the latest recorded day d with d <= target and target - d <= tolerance
pct    = (price now - low of ref day) / low of ref day * 100
```

| N | Tolerance (days earlier) |
| --- | --- |
| 1 | 0 |
| 3 | 1 |
| 7 | 2 |
| 14 | 3 |

A later day is never used. No qualifying day means no value. On the market basis only
days in the market series qualify.

### 6.4 Range bar (`Data.Range`)

`lo`, `hi`, `avg`, `pos` from Classify on the window's lows at the current cheapest
price. "Cheap right now" at `pos <= 0.20`, "Expensive right now" at `pos >= 0.80`.

---

## 7. Momentum

`Data.MomentumFor(days)`, `Data.Momentum(id)`. Answers "is the move gaining or losing
force", on the same basis as the trend with `minRun = 4`.

**Series.** Points with a positive price, oldest first. Only the trailing run counts: walk
back from the newest point while the gap to the previous one is 3 days or less. Fewer than
4 points in the run -> "Not enough history", no score. Latest point more than 3 scan-days
old -> the same.

**Rates.** With `a` = first point of the run, `b` = third from last, `z` = last:

```
recent  = ln(z.price / b.price) / (z.day - b.day)      per day, last two steps
earlier = ln(b.price / a.price) / (b.day - a.day)      per day, everything before
moving  = abs(rate) >= 0.01                            (1% a day)
```

**State.**

| Recent | Earlier | State |
| --- | --- | --- |
| still | still | Quiet |
| still | moving | Fading |
| moving | still | Building |
| moving | moving, opposite sign | Reversing |
| moving | moving, same sign | `abs(recent)/abs(earlier)` >= 1.25 Building; <= 0.8 Fading; else Steady |

**Streak.** Walking back from the newest step, count consecutive steps
`ln(p[i] / p[i-1])` of at least 0.005 in size with the same sign as the newest one. A
newest step under 0.005 gives streak 0.

**Score**, -100 to +100, positive = upward force gaining:

```
spd = clamp(recent / 0.05, -1, 1)                    5% a day = full
acc = clamp((recent - earlier) / 0.04, -1, 1)        4% a day = full
stk = sign of streak * min(streak, 5) / 5            5 steps = full
sum = 0.4*spd + 0.3*acc + 0.2*stk          weight 0.9

if units listed are known and positive at both b and z:
    qtyChange = ln(z.qty / b.qty)
    sum += 0.1 * -clamp(qtyChange / 0.25, -1, 1)     weight 1.0
    (quantity down 25% = full bullish, up 25% = full bearish)

score = round(100 * sum / weight), clamped to -100..100
```

`qty` is `a[day]`, the most units listed that day. The quantity term changes the score
only, never the state word. Direction = sign of `recent` when moving, else the streak's.

**Reported evidence**: run length, span in days, `recent` and `earlier` as true percent a
day (`(e^rate - 1) * 100`), quantity change in percent, how many days were imported.

**Sentence** (`Data.MomentumText`): mentions listings only when the quantity change is
at least 10% either way.

---

## 8. Movers, breadth, market median

`Data.lua`, `Compute()`: one walk over the price table.

**Mover.** For each item, `d1` = latest recorded day, `d2` = the one before.

```
considered only when d2 exists and today - d1 <= 3
if k[d1] and k[d2] both exist:  cur = k[d1], prev = k[d2]         (market basis)
else:                           cur = m,     prev = low of d2     (min basis)
skip when prev < 50 or cur < 50 copper
"thin" (left out, counted) when: a[d1] or a[d2] is unknown or under 10 units,
                                 or c[d1] or c[d2] is recorded and under 3 auctions
pct = (cur - prev) / prev * 100
```

Gainers sorted by `pct` descending, losers ascending, item id breaks ties.

**Breadth** = count of gainers against count of losers (`Data.Index().up / .down`), with
`thin` and `history` (items with at least two recorded days) beside them.

**Market median** = median of `m` over all priced items (even count: floor of the mean of
the two middle values). `fresh` = items whose latest day is today.

**Movers tab sorts** (`Data.MomentumMovers`), over the top 400 movers each way:
"momentum" sorts gainers by highest score and losers by lowest; "just starting" keeps
only rows whose state is Building in the list's direction.

**Snapshots** (`Dump.lua`): at most one per hour per realm, 90 kept:
`{ t, median, count, bagValue }`.

---

## 9. Categories and the heatmap

`Data.Categories`, `CatStep`, `CatFinish`.

**Grouping** by `classID, subClassID` from `GetItemInfoInstant`: class 7 trade goods
(sub 9 herbs, 7 ore and metal, 5 cloth, 6 leather, 12 enchanting, 8 food and cooking,
other "other trade goods"); class 3 gems; class 0 consumables (sub 5 -> food and cooking);
class 2 and 4 weapons and armor; anything else "everything else"; unknown id
"uncategorised". 400 items classified per window paint.

**Per item.**

```
unit  = mk when present, else m
qty   = a[latest recorded day], 1 when unknown
value = unit * qty
pct   = (m - mean of the window lows except the newest) / that mean * 100   (2+ window days)
```

**Category index.** For each item with 2+ window days, each day's low divided by the
item's first window day's low. Per day, the median of those ratios x100, kept when at
least 3 items have that day. Category trend = `Classify` on that series; category
momentum = `MomentumFor` on it; sparkline = last 7 points.

**Tile area.**

```
cap    = the 99th percentile of all item values (when 20+ items exist)
weight = max(1, min(value, cap))                 per item
category value = sum of its items' weights
share  = category value / total
tile weight = max(share, 0.025)
```

A squarified treemap lays the tiles out, deterministically.

**Tile colour** = the category index `pct` on a diverging red / slate / green scale that
saturates at +-10% for a category and +-25% for one item. No data = slate.

**Drill-down** (`Data.CategoryHeat`): the 60 largest items by weight, the rest as one
tile coloured by the median of their `pct`.

---

## 10. Shared helpers used by the tools

`Tools.lua`.

- **Book** (`I.Book`): the item's ladder with `cheapest` (level 1), `market` (`mk`),
  `units` (total listed), `auctions` (`n`), `exact` (`lx`), `ageH`.
- **Buyable** (`I.Buyable(book, price, want)`): units and cost of buying cheapest first at
  or under `price`, exact levels only, up to `want`. Stops at the first bucket and flags
  "more may exist".
- **History** (`I.Hist`): the last 14 recorded days. `medianLow`, `p75Low` (75th
  percentile of lows). `supplyPerDay = max(0, (qty[last] - qty[prev]) / gap days)`, only
  when the two days are 4 or fewer apart, both have quantity, and neither is imported.
- **Deposit** (`I.Deposit(id, qty, duration)`): the client's
  `CalculateCommodityDeposit(id, duration, qty)`. If the client gives nothing:
  `floor(vendor sell price * qty * 0.30)`, flagged inexact. Duration defaults to the
  longest auction.
- **Durations** (`I.Durations`): builds 16000 to 19999: 2 h, 8 h, 24 h. Others: 12, 24,
  48. "Longest auction" below means the last of these.
- **Scan age**: tools that buy refuse a scan older than 12 h.

---

## 11. Sniper

`ToolsBook.lua`, `Tools.Sniper`. Listings far enough under market to resell.

Per item with a ladder:

```
pre-filter:  m <= mk * 0.95 / (1 + minRoi)           minRoi default 0.25
skip (thin):     auctions < 5 or units < 10
skip (history):  fewer than 3 recorded days
ref       = market price, capped at floor(medianLow * 1.5)   (a spike rarely holds)
netAh     = Cut(ref)
exit      = vendor when the vendor sell price >= netAh, else AH
resaleNet = vendor price or netAh
maxBuy    = floor(resaleNet / (1 + minRoi))
units, cost = Buyable(book, maxBuy)                  (trimmed to opts.maxSpend if given)
profit    = units * resaleNet - cost
keep when units > 0 and profit >= 2000 and profit / units >= 20
roi       = profit / cost
AH exit only: sell hours = SellHours(ref, queue = max(0, unitsAtOrBelow(ref) - units), extra = units)
              slow when over 72 h
```

Sorted by profit. 100 rows.

---

## 12. Reset finder

`ToolsBook.lua`, `Tools.Reset`, `EvaluateItem`, `RankRow`. Thin books where buying the
cheap end and re-listing higher pays. Considered: books of 1,500 units or fewer (and
within the mode's `maxUnits`), scan under 12 h old.

### 12.1 One candidate per "clear to level j"

For `j = 1 .. min(8, exact levels, levels - 1)`:

```
N        = units in levels 1..j           cost = their total price
last     = price of level j
remUnits = book units - N                 require remUnits > 0 and N >= 2

re-list price:
  target = max(1, ceil(remUnits * 0.15))
  mkRest = price of the level (from j+1 up) where the running units reach target
  raw    = mkRest; if the mode has histCap and p75Low * histCap < raw, raw = that ("history" cap)
  relist = floor(raw * mode.relistFactor)

netUnit    = Cut(relist)
profitFull = N * netUnit - cost
require relist > last, relist <= last * 50, profitFull >= 500

ahead   = units remaining at or under relist
deposit = Deposit(id, N)
rate    = summed sell-through rate of buckets 1..BucketOf(relist, cheapest)
allRate = summed rate of all 5 buckets

supply    = (supplyPerDay if known, else 0.25 * remUnits) + allRate * 24      units per day
tUndercut = 24 * 0.10 * remUnits / supply                                      hours

if rate > 0 and grade is low or better:
    hoursToSell = (ahead + N) / rate
    soldFrac    = clamp((rate * tUndercut - ahead) / N, 0, 1)        "modelled"
else:
    soldFrac    = mode.noEvidenceSold                                 "assumed"

netFall        = Cut(last)          what the unsold part fetches after an undercut
expected(f)    = N * (f * netUnit + (1 - f) * netFall) - cost - deposit * (1 - f)
profitExpected = expected(soldFrac)
profitStress   = expected(soldFrac / 2)
breakEven      = (cost / N - netFall) / (netUnit - netFall)     nil when over 1
roi = profitExpected / cost        roiFull = profitFull / cost
```

### 12.2 Risk points

| Condition | Points |
| --- | --- |
| No sell-through evidence / low / fair | 3 / 2 / 1 |
| `hoursToSell` over 72 / over 24 | 2 / 1 |
| `ahead > 3 * N` | 1 |
| Supply growth unknown | 1 |
| `supplyPerDay / remUnits` >= 0.5 / >= 0.2 | 2 / 1 |
| `relist >= 3 * last` / `>= 2 * last` | 2 / 1 |
| Fewer than 3 recorded days | 1 |
| A remembered seller lookup (under 24 h old) shows one seller with 50%+ of the book | 1 |
| `soldFrac < 0.5` | 1 |

Grade: 0-1 low, 2-3 medium, 4-5 high, 6+ extreme.

### 12.3 Modes (filter and assumed re-list only; the maths is the same)

| | conservative | standard | aggressive |
| --- | --- | --- | --- |
| `relistFactor` | 0.90 | 0.97 | 1.00 |
| `histCap` (x p75 low) | 1.10 | 1.25 | none |
| min recorded days | 4 | 3 | 0 |
| min evidence grade | fair | low | none |
| max book units | 300 | 600 | 1,500 |
| max risk points | 3 | 5 | 99 |
| min ROI | 30% expected | 15% expected | 5% of full |
| needs `profitExpected >= 1` | yes | yes | no |
| `noEvidenceSold` | 0.25 | 0.50 | 0.70 |

A candidate also fails when `hoursToSell > 72` (except aggressive). Per item, `best` =
the passing, affordable candidate with the highest expected profit; `bestRoi` likewise by
ROI. Budget defaults to the player's gold.

### 12.4 Ranking (`RankRow`)

```
tier 3 "backed"   : modelled share, fair or good evidence, 3+ recorded days
tier 2 "partial"  : modelled share otherwise
tier 1 "unproven" : assumed share
weight = good 1.0 / fair 0.8 / low 0.4 / none 0.15
markup = relist / last
sanity = min(1, 3 / max(1, markup)) * min(1, 100 / max(1, N))
score  = profitExpected * weight * sanity
suspect = tier 1 and sanity < 0.5
```

Default order: tier, then score. Other sorts: expected, full, roi, risk. 60 rows.
"Watch" sets a below-alert at `relist - 1`.

---

## 13. Vendor flips

`ToolsFlips.lua`, `Tools.VendorFlips`. For every item with a known merchant price:

```
buy    = Data.VendorBuy(id)             what the merchant charges
sell   = Undercut(cheapest)             or the plain price when there is no ladder
net    = Cut(sell)
profit = net - buy                      roi = profit / buy
drop when profit < 5 or roi < 0.15
drop (outlier) when sell > buy * 50 and auctions < 3
thin   = auctions < 3

with sell-through evidence at `sell`:
    perDay    = rate * 24
    dayProfit = profit * perDay
    rank      = dayProfit * (good 1.0 / fair 0.8 / low 0.4)
```

Default sort: rows with a rank first by rank, the rest by ROI. 150 rows.

---

## 14. Crafting: profit, cost, crafting flips

`ToolsCraft.lua`.

### 14.1 Craft profit (`CraftRow`, `Tools.Craft`)

Material cost for `qty` units (`ReagentCost`):

```
ah     = Buyable(book, any price, qty); if the exact levels hold fewer than qty, the rest
         is priced at the market price (or cheapest) and the row is flagged approx
vendor = VendorBuy * qty
use the cheaper of ah and vendor; with no ladder and no merchant, plain price * qty (approx)
```

```
cost   = sum of material costs
sale   = product's market price (or cheapest; or plain price when it has no ladder)
net    = Cut(sale) * outQty
profit = net - cost                  roi = profit / cost
vendorBetter = vendor sell * outQty > net
profitableNow = product book has 3+ auctions, nothing approximated, profit >= 1000,
                roi >= 0.10, scan no older than 24 h
```

Sorted profitable-now first, then profit. 100 rows.

### 14.2 Craft cost (`Tools.CraftCost`)

`cost / outQty` of the cheapest recipe that makes the item with every material priced.
Used as a floor by the smart price only when nothing in it was approximated.

### 14.3 Crafting flips (`Tools.CraftFlips`): craft it, sell it to a vendor

Materials for `n` crafts (`Materials`, `Batch`): exact listings cheapest first while a
level does not cost more than the merchant, the merchant for the rest. With no merchant,
units past the exact listings are estimated at `max(market or cheapest, dearest price
already taken)` and the row is flagged approx.

```
value  = vendor sell price of the product * outQty        (no cut, no deposit)
cost   = Batch(recipe, 1)
profit = value - cost; keep when profit >= 1
roi    = profit / cost

crafts = the largest n for which the n-th craft still pays:
         Batch(n) - Batch(n-1) < value, with every unit priced exactly
         (found by doubling then halving; capped at 2,000)
spend  = Batch(crafts)             total = value * crafts - spend
more   = the count stopped at the cap, or where the exact listings end
ahBetter = Cut(product market price) > vendor sell price
```

Sorted by total, then profit. 300 rows.

---

## 15. Shopping lists

`ToolsLists.lua`. Per line `{ item, want, max unit price }`:

```
available = Buyable(book, max)                units at or under the cap, exact levels
buy, cost = Buyable(book, max, want)          cheapest first
short     = want - buy                        avgUnit = cost / buy
state     = "buy" | "none under cap" | "no data"
```

---

## 16. Posting advisor

`ToolsSell.lua`, `Tools.Post`. `c` = cheapest, `baseMk = max(market, c)`.

| Option | Price |
| --- | --- |
| Undercut (quick) | `Undercut(c)` |
| Market / Match the cheapest | `baseMk` |
| Patient | `floor(baseMk * 1.10)` |
| Premium | `floor(baseMk * 1.25)` |

Per option: `ahead = UnitsAtOrBelow(price)`, `hours = SellHours(price, ahead, count)`,
`netUnit = Cut(price)`, `netTotal = netUnit * count`, deposit, and
`vsVendor = netTotal - vendor sell * count`.

**Recommendation.**

```
with any sell-time estimate:
    score = netTotal / (1 + hours / 48)       hours = 168 for an option with no estimate
    highest score wins
without:
    Undercut when c >= market * 0.9 (tight book), else Market
vendor instead, when vendor sell * count >= the Undercut option's netTotal
```

Flags: `thinBook` = fewer than 5 auctions; `baitBelow` = `c < market * 0.75`.

---

## 17. Smart price

`ToolsPrice.lua`, `Tools.SmartPrice(id, { count })`. `c` = cheapest,
`mk = max(market, c)`, `L` = hours of the longest auction.

### 17.1 Anchor

```
with fair or good sell-through evidence:
    rate  = TimeToSell(id, mk).rate
    reach = rate * L * 0.5
    cap   = floor(mk * 1.25)
    pick  = the dearest exact level, not above cap, with (units below it + count) <= reach
    anchor = Undercut(pick price) when pick is above c, else Undercut(c)
without, when c >= mk * 0.9 and the cheapest is not bait:
    anchor = Undercut(c)
otherwise:
    anchor = mk
```

### 17.2 Adjustments (summed, then clamped to +10% / -4%)

`horizonDays = clamp(SellHours(anchor) / 24, 0.25, 2)`, 1 when there is no estimate.

| Factor | Rule |
| --- | --- |
| Momentum | `drift = recent rate per day * horizonDays * 0.5 * weight`. Weight 1 for Building or Steady, 0.5 for Fading or Reversing, 0 for Quiet. Clamped to +-6%; to +2% when the trend word is Spiking. Applied when `abs(drift) >= 0.2%` |
| Trend (only when there is no momentum reading) | Strength 1 or 2: `1% * strength * direction`. Strength 3: 0 when up, -2% when down |
| Range | Needs 3+ days and a range 10%+ wide. `pos <= 0.2` and not falling: +2%. `pos >= 0.8` and not rising: -1% |
| Supply | Momentum's quantity change `<= -15%`: +1.5%. `>= +15%`: -1.5% |
| Weekday | Only when `L >= 24`, the pattern is not flat, and today and tomorrow each have 2+ weeks: `w = (f[tomorrow] - f[today]) * 0.5`, clamped to +-3%, applied when `abs(w) >= 0.3%` |

```
adj    = clamp(sum, -0.04, +0.10)
target = max(1, round(anchor * (1 + adj)))
```

### 17.3 Snap onto the book

```
if target >= c:
    next = the first exact level priced above target
    if Undercut(next) > target and Undercut(next) <= target * 1.03: price = Undercut(next)
    if price equals an existing exact level: price = Undercut(that level)
else:
    price = min(target, Undercut(c))
```

### 17.4 Guards, in order

1. **Expiry** (any evidence grade but none): if `SellHours(price) > L`, walk the exact
   levels below the price from dearest to cheapest and take the first `Undercut(level)`
   whose estimate fits in `L`; failing that, `Undercut(cheapest level)`.
2. **Own auction**: if your cheapest active auction is at or under `c` and the price is
   below it, price = your auction's price.
3. **Vendor floor**: `floor = floor(vendor sell / 0.95) + 1`. If the posting advisor said
   vendor, or `floor > mk * 1.25`: action = VENDOR. Else if `price < floor`: price = floor.
4. **Craft floor** (exact craft cost only): `breakEven = floor(craftCost / 0.95) + 1`. If
   `breakEven > mk * 1.25`: price unchanged, and a note states the shortfall
   `craftCost - Cut(price)`. Else if `price < breakEven`: price = breakEven.

### 17.5 Beside the price

- **HOLD** when momentum is up, Building or Steady, `recent >= 3%` a day and the trend is
  not Spiking. Or when a weekday within the next 3 days runs 4%+ above today's, the
  pattern's grade is better than low, and the price is not flagged as falling.
- **Falling** note when `recent <= -1%` a day and the state is Building or Steady.
- **Quantity**: `room = floor(rate * L - ahead)`. Suggest `room` when `rate > 0`,
  `room >= 1` and `count > room * 1.5`.
- **Duration**: the shortest with `hours >= 2 * estimated time to sell`; the longest when
  there is no estimate. The deposit shown is for that duration.
- **Confidence**: one point each for 5+ auctions, a momentum reading, a scan 6 h old or
  less; two points for fair or good evidence, one for low. 5+ = good, 3-4 = fair, else low.

Every factor is returned with its size and a sentence. The window shows each one.

---

## 18. Weekday pattern

`ToolsPrice.lua`, `Tools.Season`. Market-wide, not per item.

```
window = the last 35 scan-days
per item (stop after 2,500 items): its recorded days in the window, v = ln(day's low)
    needs 6+ days;  r[d] = clamp(v[d] - mean(v), -0.5, +0.5)
per date: value[d] = mean of r[d] over items;  a date needs 20+ items
blocks: five 7-day blocks counted back from the latest qualifying date
    a block needs 4+ qualifying dates;  blockMean = mean of its dates' values
per weekday w: f[w] = exp(mean over blocks of (value[d] - blockMean)) - 1
weeks  = number of qualifying blocks
grade  = 4+ weeks good, 3 fair, 2 low, else none
spread = f[best] - f[worst];  flat when spread < 0.01
```

Measuring each date against its own week keeps a market that is simply rising from
looking like a weekday effect. Weekday of a scan-day = `date("%w", DAY0 + day*86400 + 43200)`.

---

## 19. Post and cancel checks

`ToolsPost.lua`.

- Price must be a whole number of copper, 1 or more. Quantity 1 to what the house will
  take (`GetAvailablePostCount`).
- Deposit from `CalculateCommodityDeposit` / `CalculateItemDeposit` for the chosen
  duration; refused when the player's gold is under it.
- `total = price * qty`, `net = Cut(price) * qty`.
- **Second click** when `vendor sell >= Cut(price)`, or `price < market * 0.5`.
- Cancel: always a second click. Cost shown = the lost deposit plus `GetCancelCost` when
  the client has it; refused when gold is under that fee.

---

## 20. Undercut check

`ToolsSell.lua`, `Tools.Undercuts`. For each active owned auction at unit price `u`:

```
walk the ladder's levels priced below u
    others = level units, minus your own units at that exact price when that auction was
             posted before the scan (the ledger knows when); buckets cannot be adjusted
    count levels where others > 0
cheapest = the lowest such level;  unitsBelow = their sum
by = u - cheapest;  byPct = by / u;  suggest = Undercut(cheapest)
state = undercut | ok | nobook (item not in the last scan) | sold
```

Sorted by `byPct`. Owned unit price (`I.OwnedUnit`): `buyoutAmount` as given while
active; `floor(buyoutAmount / quantity)` on a sold row.

---

## 21. Seller view

`ToolsSellers.lua`. From one item search (rows with price, units, up to 4 owner names).

```
soleUnits   = units in rows this seller alone owns
sharedUnits = units in rows shared with others where this name appears
shareMin = soleUnits / total          shareMax = min(1, (sole + shared) / total)
concentrated = top seller's shareMin >= 0.50
cheap end = the cheapest rows covering max(1, ceil(total * 0.15)) units
    oneSeller = yes (one named owner) | no (two names, or a shared row) | unknown (names missing)
```

A lookup is "fresh" for 24 h.

---

## 22. Portfolio, bags, tooltip verdict, watch alerts

**Bags** (`Data.Bags`). Per item: `ahUnit = floor(price * 0.95)`. Best exit = AH when
`ahUnit >= vendor sell` (or no vendor price), else vendor. Totals: all on AH, all to
vendor, best exit each. Soulbound units are excluded from posting advice.

**Portfolio point** (`ToolsPortfolio.lua`), at most one an hour, 200 kept:

```
bag    = sum of best exits            bagAh, vendor = the two alternatives
auc    = sum of Cut(your unit price) * qty over active auctions
aucMkt = the same units at Cut(market price)
sold   = sum of Cut(unit) * qty over sold, uncollected auctions
total  = gold + bag + auc + sold
change vs a day / a week: against the latest saved point at least 0.8 x that old
```

**Realised** (all time, 7 days, 30 days), from the ledger: posted, sold, gone, active
counts and units; `soldNet = sum of Cut(price) * qty`;
`sellRate = sold units / (sold units + gone units)`. "Gone" mixes expired and cancelled.

**Tooltip verdict** (`Tooltip.lua`): `net = price * 0.95`. "Vendor it" when
`vendor sell >= net`. "Sell on AH" when `net >= vendor sell * 1.5`. Nothing in between,
and nothing when the vendor price is unknown.

**Watch alerts** (`Watch.lua`): hit when `price <= below` or `price >= above`, announced
once per crossing.

---

## 23. The pipeline board

The Python tools (`tools/`), run each cycle by `tools/wowd.py`:
`ingest_wow.py` -> `analyze_wow.py` -> `trend_wow.py` -> `craft_wow.py` ->
`export_decades.py`. They feed `DecadesMarketBoard` (Movers > Pipeline and the tooltip
signals). Fee is 5%.

**These are weaker than the addon and are not the specification.** The pipeline ingests
no market price, so every "price" below is the cheapest listing (`m`), and one bait
listing moves it. They are documented here because the board shows their output.

Tables: `prices(item, kind, duration, price)` holds the latest state: kind `m` (duration
0), and `l`, `h`, `a` per scan-day (duration = the scan-day number). `price_history`
holds the same rows for every ingest, stamped `snap_ts`.

### 23.1 Index, breadth, regime (`trend_wow.market_index`)

```
index[snapshot] = median of ln(m) over items;  shown as exp(index)
return since first = (exp(index[last]) - exp(index[first])) / exp(index[first]) * 100
breadth: compare the last snapshot with the one closest to 24 h earlier, over items in both
    up_pct = items with m higher / items compared * 100;  down_pct likewise
regime = BULL when up_pct >= 55, BEAR when up_pct <= 45, else CHOP
```

### 23.2 Movers (`trend_wow.momentum_alpha`)

```
t1 = latest snapshot; t0 = the earlier snapshot closest to t1 - 24 h; needs 50+ items in both
r     = 100 * ln(m[t1] / m[t0])
mret  = median of r over all common items
skip (thin) when either price < 50 copper or units listed at either snapshot < 10
alpha = r - mret
mu = median of r over the kept items;  sd = population standard deviation of them
z  = (r - mu) / sd
board = the 20 largest abs(z);  ret_pct = (e^(r/100) - 1) * 100
signal logged when abs(z) >= 3   (same item and direction suppressed for 6 h)
```

The export keeps up to 10 with `ret_pct > 0` and 10 with `ret_pct < 0`.

### 23.3 Breakout (`trend_wow.breakout_scanner`) — the board's "Breaks"

```
box = over the item's scan-days EXCLUDING the newest scan-day in the table:
      lo = lowest l,  hi = highest h   (a missing side borrows the other)
up-break    when m > hi:  break_pct = (m - hi) / hi * 100;  width = (hi - lo) / hi
down-break  when m < lo:  break_pct = (lo - m) / lo * 100;  width = (hi - lo) / lo
tight = width < 0.10
```

The newest day is left out because `m` is that day's last observation, so `h[today] >= m`
always and an up-break could never show. **The board lists tight breaks only**, the 10
largest `break_pct` each way.

### 23.4 Squeeze (`analyze_wow.squeeze_drain`)

```
days = scan-days on which the item has BOTH an l row and an h row; needs 2+
d0 = the earliest such day, d1 = the latest
band0 = (h[d0] - l[d0]) / l[d0]
band1 = (h[d1] - l[d1]) / l[d1]
squeeze when band0 > 0.10 and band1 < band0 * 0.5
sorted by band1 / band0 ascending; top 25 kept, the board shows 10
```

Only the first and last qualifying day are compared; nothing in between is looked at. An
`l` row exists only where the dump carried one, so days with a single price can be absent
from `days`.

### 23.5 Drain (`analyze_wow.squeeze_drain`)

```
adays = scan-days with an a row (units listed); needs 3+
a0 = units on the earliest, aN = units on the latest
drain when a0 >= 10 and aN <= a0 * 0.4
sorted by aN / a0 ascending; top 25 kept, the board shows 10
```

Price is not checked, although the docstring says "with stable price".

### 23.6 Vendor flip (`analyze_wow.vendor_flip`)

```
net = m * 0.95;  flip when net > vendor price
roi_pct = (net - vendor) / vendor * 100;  margin = net - vendor
sorted by roi_pct; top 25 kept, the board shows 15
```

No listing floor, no outlier guard. The in-game Vendor flips page (section 13) replaces it.

### 23.7 Percentile (`analyze_wow.percentile_positioner`)

Share of items with a lower `ln(m)`, over the whole market, and over the item's "cohort"
(other items in the same power-of-ten price band, 5+ needed).

### 23.8 Craft board (`craft_wow.build_board`)

```
cost = sum of m[material] * qty      (every material must have a price)
net  = m[product] * 0.95 - cost
net_per_unit = net / out_qty;  roi = net / cost
sorted by net_per_unit
```

Note that `out_qty` does not multiply the sale price here. The addon's craft profit
(section 14.1) does, and prices materials off the ladder.

### 23.9 Board fields that are always empty

`export_decades.py` copies these names, but the tools above do not produce them:
`squeeze[].box_pct`, `drain[].stock`, `flips[].net`. Readers must treat them as absent.
The useful fields are `band_first` / `band_last`, `qty_first` / `qty_last`, and
`buy` / `sell`.

Each board row also gets `s` (the item's last 12 snapshot prices), `now`, `lo`, `qty`,
`v` (vendor), `d` (days of history), `n` (snapshots).

---

## 24. Every threshold in one table

Addon values are in `Tools.K` (`Tools.lua`) unless a file is named. All are first
guesses; none has been tuned against real sales.

| Name | Value | Used for |
| --- | --- | --- |
| `AH_CUT` | 0.95 | the cut |
| `UNDERCUT_PCT` | 0.01 | undercut step |
| `MARKET_FRACTION` (Scan) | 0.15 | market price |
| `KEEP` (Scan) | 40 | price levels kept while scanning |
| `EXACT` (Store) | 8 | exact ladder levels |
| `PREMIUM` (Store) | 1.02, 1.10, 1.25, 1.50 | bucket bounds |
| `HISTORY_DAYS` (Store) | 35 | history kept |
| `MIN_DT`, `MAX_DT` (Store) | 0.1 h, 12 h | usable scan pairs |
| `DECAY` (Store) | 0.95 | fading of old sell-through evidence |
| Band lower bounds (Store) | 0, 0.5, 2, 12 h | testable levels |
| `WINDOW_DAYS` (Data) | 14 | trend window |
| `TREND_MIN_DAYS` (Data) | 3 | trend |
| `FLAT_PCT`, `DRIFT_PCT`, `MOVE_PCT` (Data) | 3, 8, 20 | trend strength |
| `EXTREME_SPREAD`, `LOW_ZONE`, `HIGH_ZONE` (Data) | 0.10, 0.20, 0.80 | bottoming / topping |
| `MOM_MIN_DAYS`, `MOM_MAX_GAP` (Data) | 4, 3 | momentum run |
| `MOM_MOVING`, `MOM_STEP` (Data) | 0.01, 0.005 | moving / streak step |
| `MOM_FASTER`, `MOM_SLOWER` (Data) | 1.25, 0.8 | building / fading |
| Momentum full scale (Data) | 0.05, 0.04, 5 steps, 0.25 | speed, acceleration, streak, supply |
| Momentum weights (Data) | 0.4, 0.3, 0.2, 0.1 | the same four |
| `MOVER_MIN_PRICE`, `MOVER_MAX_AGE`, `MOVER_MIN_QTY`, `MOVER_MIN_LISTINGS` (Data) | 50c, 3 days, 10, 3 | movers |
| `HEAT_MIN_SHARE`, `HEAT_CAP_Q`, `CAT_MIN_SAMPLES` (Data) | 0.025, 0.99, 3 | heatmap |
| `DEPOSIT_FALLBACK_PCT` | 0.30 | deposit estimate |
| `SNIPE_MIN_ROI`, `SNIPE_MIN_PROFIT`, `SNIPE_MIN_UNIT_PROFIT` | 0.25, 2000c, 20c | sniper |
| `SNIPE_MIN_AUCTIONS`, `SNIPE_MIN_UNITS`, `SNIPE_MIN_DAYS` | 5, 10, 3 | sniper |
| `SNIPE_SPIKE_CAP`, `SNIPE_MAX_SCAN_AGE_H`, `SNIPE_SLOW_HOURS` | 1.5, 12, 72 | sniper |
| `POST_MARKET_STEPS` | 1.10, 1.25 | patient, premium |
| `POST_BAIT_RATIO`, `POST_THIN_AUCTIONS` | 0.75, 5 | bait, thin book |
| `POST_HOURS_SCALE`, `POST_UNKNOWN_HOURS` | 48, 168 | advisor score |
| `POST_CONFIRM_LOW` | 0.5 | second click |
| `SMART_MAX_UP`, `SMART_MAX_DOWN` | 0.10, 0.04 | adjustment clamp |
| `SMART_DRIFT_SHARE`, `SMART_DRIFT_CAP`, `SMART_SPIKE_CAP` | 0.5, 0.06, 0.02 | momentum factor |
| `SMART_TREND_STEP` | 0.01 | trend factor |
| `SMART_RANGE_LOW_UP`, `SMART_RANGE_HIGH_DOWN` | 0.02, 0.01 | range factor |
| `SMART_SUPPLY`, `SMART_SUPPLY_MOVE` | 0.015, 15% | supply factor |
| `SMART_WEEKDAY_SHARE`, `SMART_WEEKDAY_CAP` | 0.5, 0.03 | weekday factor |
| `SMART_SNAP` | 0.03 | snap |
| `SMART_TARGET_SHARE`, `SMART_ANCHOR_MAX` | 0.5, 1.25 | anchor |
| `SMART_HOLD_RATE`, `SMART_HOLD_WEEKDAY` | 3% a day, 0.04 | hold |
| `SMART_QTY_SLACK` | 1.5 | quantity |
| `SEASON_WINDOW_DAYS`, `SEASON_MIN_ITEM_DAYS`, `SEASON_MIN_ITEMS` | 35, 6, 20 | weekday pattern |
| `SEASON_MIN_BLOCK_DAYS`, `SEASON_MAX_ITEMS`, `SEASON_FLAT` | 4, 2500, 0.01 | weekday pattern |
| `FLIP_MIN_UNIT_PROFIT`, `FLIP_MIN_ROI`, `FLIP_MAX_MULT`, `FLIP_THIN_AUCTIONS` | 5c, 0.15, 50, 3 | vendor flips |
| `CFLIP_MIN_PROFIT`, `CFLIP_MAX_CRAFTS` | 1c, 2000 | crafting flips |
| `CRAFT_MIN_PROFIT`, `CRAFT_MIN_ROI`, `CRAFT_MIN_AUCTIONS` | 1000c, 0.10, 3 | craft profit |
| `RESET_MAX_LEVELS`, `RESET_MARKET_FRACTION` | 8, 0.15 | reset |
| `RESET_SANITY_MIN_UNITS`, `RESET_SANITY_MAX_MULT`, `RESET_SANITY_MIN_PROFIT` | 2, 50, 500c | reset |
| `RESET_UNDERCUT_SHARE`, `RESET_DEFAULT_SUPPLY_PER_DAY`, `RESET_FALLBACK_HOURS` | 0.10, 0.25, 72 | reset |
| `RESET_RANK_MARKUP`, `RESET_RANK_UNITS`, `RESET_RANK_WEIGHT` | 3, 100, 1.0/0.8/0.4/0.15 | reset ranking |
| `SELLER_CONCENTRATED`, `SELLER_CHEAP_FRACTION`, `SELLER_FRESH_H` | 0.50, 0.15, 24 | sellers |
| `VERDICT_MARGIN` (Tooltip) | 1.5 | "Sell on AH" |
| `PORTFOLIO_GAP`, `PORTFOLIO_CAP` | 3600 s, 200 | portfolio |
| Pipeline: breadth regime | 55 / 45 | BULL / BEAR |
| Pipeline: mover floors, z signal | 50c, 10 units, abs(z) >= 3 | movers |
| Pipeline: tight box | width < 0.10 | breaks |
| Pipeline: squeeze | band0 > 0.10, band1 < 0.5 x band0 | squeeze |
| Pipeline: drain | a0 >= 10, aN <= 0.4 x a0, 3+ days | drain |

---

## 25. Known biases and things never measured

- Sell-through counts cancels as sales (reads high) and cannot see units sold and
  replaced between scans (reads low). It needs scans a few hours apart; with one scan a
  day most pairs are discarded as "far".
- Past the 8 exact levels, units-at-or-below is an upper bound, so time to sell at a high
  price reads long.
- The market price is a lower bound for an item with more than 40 distinct prices whose
  cheapest 40 hold under 15% of its units.
- Momentum's quantity is the day's largest listed total, so within-day resupply is
  invisible.
- The min basis (Auctionator days, imported days, days before 2.2.0) follows the cheapest
  listing and can be moved by one bait listing. The market basis cannot.
- Reset: `supplyPerDay` is growth between two scan-days only, and never negative.
- Vendor flips only cover merchants the player has opened. Crafting flips price off one
  scan while other players buy the same materials.
- No threshold in section 24 has been measured against real sales. The post log
  (`DecadesMarketDB.tools.postLog`) exists so the smart price can be; that check has not
  been done.

---

## 26. Open decision: should momentum use sales?

Today it does not. Two ways to add it, both using the sell-through rate from section 5:

1. **Replace the 10% supply term** with a demand term: the item's sell-through rate
   against its own listed units, e.g. `turnover = rate[1..2] * 24 / units listed`, scaled
   so a set daily turnover is full bullish. "Units are leaving at the cheap end" is a
   stronger signal than "fewer are listed".
2. **Keep supply, add demand** as a fifth term and re-weight.

Either way: most items scanned once a day have no usable pair and would fall back to the
price-only score, so the score would mean slightly different things for different items
unless the window marks which ones carry the demand term (as `evidence.hasQty` does now).
With guild scans pooled on the server this limit mostly goes away, which is a reason to
decide it together with the server work.

Not built. The choice belongs to whoever owns the momentum rule.
