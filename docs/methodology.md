# Methodology

This document walks through every calculation in the model, in the order they depend on each other. DAX is quoted from `Freight Elasticity.SemanticModel/model.bim`, shortened where noted.

- [1. Input signals](#1-input-signals)
- [2. Data-sufficiency rules](#2-data-sufficiency-rules)
- [3. Elasticity variants](#3-elasticity-variants)
- [4. Elasticity clusters](#4-elasticity-clusters)
- [5. Window / Shadow / Blind (WSB)](#5-window--shadow--blind-wsb)
- [6. Market Position](#6-market-position)
- [7. Supporting metrics](#7-supporting-metrics)
- [8. Implementation notes & caveats](#8-implementation-notes--caveats)

---

## 1. Input signals

All elasticity measures correlate two weekly series. The observations are the weeks (`year-week = year*100 + week`) visible in the current filter context.

### Markup over ECOH %

ECOH (*estimated cost of haul*) is the brokerage's estimate of the carrier cost of a load. Markup over ECOH is the margin the brokerage put on top of it.

```dax
Markup Over ECOH % =
CALCULATE(
    DIVIDE(SUM([markup_ecoh]), SUM([original_ecoh])),
    'Loads (Main Set)'[original_ecoh] >= 300      -- ignore tiny/partial loads
)
```

### Markup over ECOH (wo ol) %: outliers removed

Outlier fences are computed **once, across the whole dataset**, from every *(week, KMA lane)* point. Any value outside them is blanked:

```
IPR = P80 − P20
low fence  = P20 − 1.25 · IPR
high fence = P80 + 1.75 · IPR
```

The fences are an inter-percentile variant of Tukey's fences. They are **asymmetric** on purpose: markup is bounded below (deep negative markups are rare errors) but has a long, legitimate right tail. The `OL Fences` and `OL Counts` DAX queries show the fence values and how many points each one removes.

### Conversion %

```
Conversion = MIN(loads ÷ quotes, 100 %)
```

Quotes are matched to loads on a composite key, using an **inactive relationship** activated with `USERELATIONSHIP`. That way the same `Quotes` table can be joined at two grains:

| Measure | Matching key |
|---|---|
| `# Quotes (Lane Level)` / `Conversion (Lane Level) %` | year-week : city lane : equipment : quoting method |
| `# Quotes (KMA Level)` / `Conversion (KMA Level) %` | year-week : KMA lane : equipment : quoting method |

All elasticity measures use **KMA-level** conversion. City-to-city lanes are too sparse week to week to correlate.

## 2. Data-sufficiency rules

### LPQ Threshold: minimum weeks

```dax
LPQ Threshold = ROUND(COUNTROWS(ALL weeks in dataset) / 10, 0)    -- 135 weeks → 14
```

A context is only scored when the number of weeks with **both** a markup and a conversion value (`C`) is at least this threshold. Otherwise every elasticity measure returns blank.

### ZDT Fence: zero-deflation scale

This fence is used only by SRCC. It ranks weekly markup and weekly conversion for the **whole portfolio** (one point per week), takes the rank difference `D` for each week, and returns its inter-quartile range:

```
ZDT Fence = P75(D) − P25(D)
```

It sets the scale of the "missing weeks" penalty described below. The `ZTD Fence` and `SRCC` DAX queries are the working versions of this calculation.

## 3. Elasticity variants

Notation: `T` is the set of weeks in context, `N = |T|` and `C` is the number of weeks where both signals exist. *X* is weekly markup (outliers removed) and *Y* is weekly KMA-level conversion.

**Sign convention for all variants:** a negative value means higher markup goes with lower conversion (**elastic**). A positive value means conversion holds up or rises (**inelastic**).

### Elasticity (raw): Pearson *r*

The textbook computational formula, built from running sums over `T`:

```
r = (C·ΣXY − ΣX·ΣY) / √[(C·ΣX² − (ΣX)²)·(C·ΣY² − (ΣY)²)]
```

If the radicand is negative (floating-point edge cases), the result is blank.

### Elasticity %: normalised Pearson

```
Elasticity % = 1 − 2 / (1 + e^r)      ( = tanh(r / 2) )
```

A logistic squashing that keeps the sign and order of *r* and maps [−1, 1] onto ≈ [−0.46, 0.46]. **This is the headline elasticity in most visuals.**

### Elasticity % SRCC: Spearman with zero-deflation

```
ρ        = 1 − 6·Σd² / (N³ − N)          d = dense rank(X) − dense rank(Y), ranked descending
LF       = ZDT Fence
SF       = MAX(LF / 10, 5)
penalty  = MIN((N − C) / LF, 1 / SF)     -- at most 0.2
SRCC     = ρ − penalty
```

Spearman captures any **monotonic** relationship, not just linear ones, and is robust to the occasional extreme week. The **zero-deflation term** subtracts a penalty that grows with the number of weeks where the lane had activity but a missing markup or conversion value (`N − C`). Thin, gappy lanes are pulled toward "more elastic / less certain" instead of showing a confident correlation built on few points. The penalty is capped at 1 / SF ≤ 0.2, so it can only shift the score, never dominate it.

### Elasticity % KTC: Kendall τ

For every pair of weeks, count concordant pairs (both ranks move in the same direction) and discordant pairs:

```
τ = (concordant − discordant) / C(N, 2)
```

This is τ-a: ties aren't corrected. It's less sensitive than Spearman to a single large rank gap, so it works as a robustness check. The DAX is O(N²) per evaluated cell. With weekly grain and ~135 weeks it stays fast.

### ξ Correl. %: ξ-style dependence

Inspired by [Chatterjee's ξ coefficient (2021)](https://arxiv.org/abs/1909.10140), which can detect non-monotonic dependence:

```
D  = Σ |R(i+1) − R(i)|            R = rank of conversion, i+1 = the following week
L  = Σ l(i)·(N − l(i))            l(i) = # weeks with markup ≥ markup(i)
ξ  = 1 − N·D / (2·L)
ξ Correl. % = 1 − 2 / (1 + e^ξ)
```

See the [caveats](#8-implementation-notes--caveats) for how this differs from the textbook estimator.

## 4. Elasticity clusters

`Elasticity Clusters` combines the two main variants, **Elasticity %** (Pearson, `D1`) and **Elasticity % SRCC** (`D2`). It uses a neutrality threshold:

```dax
NC Threshold = − MEDIAN over all (KMA lane : equipment) keys of [Elasticity %]
```

The typical lane is mildly elastic (negative median), so `NC Threshold` is a small positive number. Lanes no further from zero than a typical lane are treated as neutral.

| Rule (evaluated in order) | Cluster |
|---|---|
| both blank | *(blank)* |
| \|D1\| < NC and \|D2\| < NC | **3. Neutral** |
| D1 < 0 and D2 < 0 | **1. True Elastic**: both methods agree price hurts conversion |
| D1 > 0 and D2 > 0 | **5. True Inelastic**: both agree there's pricing headroom |
| D1 < 0 and D2 > 0 | **2. Semi Elastic**: linear trend elastic, rank trend not |
| otherwise | **4. Semi Inelastic** |

The numeric prefixes order the clusters from most to least price-sensitive in slicers and legends.

### Poisson clusters (alternative segmentation)

The `Poisson` table holds elasticities from a **Poisson regression** of loads on markup. It was fitted outside Power BI for 130 dry-van KMA lanes:

- `poisson_raw = elasticity / 100` (coefficient); `Poisson % = AVERAGE(poisson_raw) / 25` rescales it to a range comparable with Elasticity %.
- `ClusterMappingTable` is the calculated table Power BI generates when you use **Automatically find clusters** on a scatter chart. It runs `KMeansClustering` with **k = 5** over *(Elasticity %, Poisson %, total loads)* per lane.
- `Poisson Clusters` maps the five k-means IDs onto the same five labels as above. The *Clusterization [Poisson]* page puts the regression-based and correlation-based views side by side.

## 5. Window / Shadow / Blind (WSB)

WSB measures **how visible a lane is**: how much data exists to price it confidently. It's only defined for a single KMA lane in context. Otherwise it's blank.

| Source | Measure | WINDOW | SHADOW | BLIND |
|---|---|---|---|---|
| Brokerage's own loads | `WSB` | > 12 loads **and** > 2 shippers | exactly one of the two | neither |
| Market Intel A | `WSB (Market Intel A)` | lane volume > P99 | lane volume > P97.5 | otherwise |
| Market Intel B | `WSB (Market Intel B)` | volume > P75 **and** contributors > P90 | exactly one of the two | neither |

Market percentiles (`Market Intel A P99`, `Market Intel A P98`, `Market Intel B P75`, `Market Intel B P90`) are computed over all lanes, ignoring report filters. The measure is named `Market Intel A P98`, but it uses the 97.5th percentile.

## 6. Market Position

Each WSB level is mapped to an ordinal (WINDOW = 3, SHADOW = 2, BLIND = 1), and the brokerage's level is compared with the market's:

| Condition | Market Position |
|---|---|
| brokerage blank, market has a level | **Not participating** |
| market blank | *(blank)* |
| brokerage = market | **At Market** |
| brokerage > market | **Ahead Of Market** |
| brokerage < market | **Behind Market** |

`Market Position` uses Market Intel A and `Market Position (Market Intel B)` uses Market Intel B. The calculated table `dim_loads_clusters` takes a snapshot of both positions, the elasticity cluster and the three WSB levels per *KMA lane : equipment*, so they can be used as **slicers** and axes. A measure can't be placed on a slicer directly.

Combined with elasticity, this gives the pricing playbook. For example, a *True Inelastic* lane where the brokerage is *Behind Market* has room to price up. A *True Elastic* lane in a market *Window* calls for sharp, benchmark-driven pricing.

## 7. Supporting metrics

| Measure | Definition |
|---|---|
| `RPM` | bookings ÷ mileage, blanked outside $1–$30 per mile (data-quality guard) |
| `RPM (Market Intel A)` | Market Intel A average rate ÷ mileage, for benchmarking `RPM` |
| `Sourcing Spread Base %` | gross sourcing spread ÷ bookings |
| `Avg. LPW` | loads per week |
| `# Unique Carriers` | distinct carriers per week (rounded up) |
| `# Unique Shippers`, `# Unique Lanes`, `# Unique KMA Lanes`, `# Unique Cluster Keys`, `# Weeks` | distinct counts |

`dim_lane_activity` flags each *KMA lane : equipment : quoting method* as active in 2023 and/or 2024, so dormant lanes can be filtered out.

## 8. Implementation notes & caveats

- **Correlation is not causation.** Markup and conversion move together for reasons other than price sensitivity too, such as seasonality, capacity crunches and shipper mix. The clusters are a prioritisation tool, not a demand curve.
- **Dense ranks** are used for Spearman and Kendall. With ties, the Spearman shortcut formula is an approximation, and Kendall is τ-a (no tie correction).
- **ξ variant:** the textbook ξ sorts observations by *X* (markup) and sums rank jumps of *Y* between consecutive *X*. This implementation takes consecutive **weeks** instead, and builds `l(i)` from *X*. It behaves as a time-ordered dependence score, so read it as experimental.
- **Sigmoid normalisation** (`tanh(r/2)`) compresses the scale, so `Elasticity %` never exceeds ≈ ±0.46 and isn't on the same scale as SRCC or KTC. Compare variants by sign and rank, not by magnitude.
- Outlier fences and the ZDT fence are **global**: they don't change with slicers. That's intended, because every lane is judged against the same yardstick.
- The calculated tables (`dim_loads_clusters`, `ClusterMappingTable`, `dim_lane_activity`) are computed **at refresh time** and don't react to report filters.
