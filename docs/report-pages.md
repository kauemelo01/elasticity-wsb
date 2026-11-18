# Report pages

The report has 16 pages, in four groups. Most pages share two filter tables, **shipper list** and **quoting method** (manual vs. automated): click a row to scope the whole page.

## 1. Elasticity by grain

The same measures, sliced at different levels of detail.

| Page | What it shows |
|---|---|
| **Regions** | Matrix of origin region × destination region × distance bucket (under / over 500 miles) with Pearson and SRCC elasticity. Shows where in the country price sensitivity sits. |
| **Lanes** | City-to-city lanes: unique shippers and carriers, conversion, markup, sourcing spread, SRCC and elasticity cluster. A combo chart tracks weekly bookings against markup and conversion. |
| **KMAs** | The same view at KMA-lane level, where elasticity is actually computed, with a card counting lanes per cluster. |
| **Shippers** | Matrix of shipper → KMA lane with conversion, markup, sourcing spread, loads per week, SRCC and cluster: which customers tolerate price increases. |
| **Cluster Keys** | Grain of shipper × equipment × KMA lane, the most specific unit a pricing rule can target. |
| **Scatter Plot** | Weekly markup (outliers removed) vs. conversion for the selected lane(s). This is the raw picture each elasticity number summarises, with summary cards for Pearson, SRCC, markup, conversion and loads. |

## 2. Clusterization

Scatter charts that place each entity by its elasticity scores, coloured by `Elasticity Clusters`. Tooltips carry all four variants (Pearson, SRCC, KTC, ξ), loads per week and total loads.

| Page | One dot per |
|---|---|
| **Clusterization [Lane KMA]** | KMA lane |
| **Clusterization [Shipper]** | shipper |
| **Clusterization [Cluster Keys]** | shipper × equipment × KMA lane, plus a count of keys per cluster |
| **Clusterization [Poisson]** | Poisson lane: correlation elasticity vs. Poisson-regression elasticity, coloured by the k-means `Poisson Clusters` |

## 3. Window / Shadow / Blind

| Page | What it shows |
|---|---|
| **WSB (Sonar)** | Lane list with the brokerage's WSB next to SONAR's WSB, all elasticity variants, RPM vs. SONAR RPM and share of market volume. Pivot summaries count lanes by cluster × WSB, both internal and SONAR. A combo chart compares weekly RPM against SONAR RPM and volume. |
| **WSB (DAT)** | The same, benchmarked against DAT volumes and contributors. |

## 4. Market Position

| Page | What it shows |
|---|---|
| **Market Position (Sonar)** | Scatter of KMA lanes (Pearson vs. SRCC elasticity), coloured by Market Position against SONAR. Slicer on position, cards counting lanes per position. |
| **Market Position (Sonar) [Table]** | The same lanes as a sortable table, with WSB levels, volumes, carriers, shippers and sourcing spread. |
| **Market Position (DAT)** | Scatter as above, positioned against DAT. |
| **Market Position (DAT) [Table]** | Table version for DAT. |

## Reading the pages together

1. Start at **Regions** or **KMAs** to find clusters of elastic or inelastic lanes.
2. Check the **Scatter Plot** for a lane to make sure the correlation is real, not driven by a handful of weeks.
3. Use **Market Position** to see whether the brokerage is ahead of or behind the market on those lanes.
4. Drill into **Shippers** or **Cluster Keys** to find who is driving the behaviour, which is the level where a pricing action is actually taken.
