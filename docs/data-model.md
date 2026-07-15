# Data model

The Azure Cost Analyzer lakehouse follows a **medallion** layout. Everything downstream is derived
from a single silver table, `focus_partitioned`, which holds normalized **FOCUS** billing rows.

```
FOCUS export (parquet)  ──▶  focus_partitioned (silver)  ──▶  gold_* facts + dim_* dimensions  ──▶  AzureCostAnalyzer_SM
```

- **Grain conventions**: *daily* facts join `dim_date[Date]`; *monthly* facts join
  `dim_month[YearMonth]`.
- **Cost columns**: `EffectiveCost` (amortized/actual), `BilledCost` (invoiced), `ListCost`
  (public list price), `ContractedCost` (negotiated on-demand-equivalent).
- **Window**: gold facts keep the **last 12 months** (partition-pruned on `Year`/`Month`).

---

## Silver

### `focus_partitioned`

Normalized FOCUS rows loaded by `01_Load_CostManagement_Focus_Data`, partitioned by `Year`,
`Month`. Carries the full FOCUS column set plus Azure `x_*` extensions (`x_ResourceGroupName`,
`x_CostCenter`, `x_EffectiveCostInUsd`, `x_SkuDetails`, …) and these **derived** columns added
during enrichment:

| Column | Type | Source |
|---|---|---|
| `Date` | date | `to_date(ChargePeriodStart)` |
| `YearMonth` | string `yyyy-MM` | `date_format(ChargePeriodStart)` |
| `ResourceGroupName` | string | `x_ResourceGroupName` |
| `ResourceType` | string | ARM provider extracted from `ResourceId` (`providers/<ns>/<type>`), else `(unknown)` |
| `Year`, `Month` | int | partition keys (from `BillingPeriodStart`) |

Key FOCUS columns used downstream: `ChargePeriodStart/End`, `BillingPeriodStart`, `ChargeCategory`,
`PricingCategory`, `CommitmentDiscountStatus/Type/Id/Name/Category`, `SubAccountName`,
`ServiceCategory`, `ServiceName`, `RegionName`, `ResourceId`, `ResourceName`, `Tags` (JSON),
`EffectiveCost`, `BilledCost`, `ListCost`, `ContractedCost`, `ConsumedQuantity`.

---

## Dimensions

### `dim_date` — one row per day (`02_Gold_Calendar`)

| Column | Type | Notes |
|---|---|---|
| `Date` | date | key |
| `Year`, `Month`, `Quarter`, `WeekOfYear`, `DayOfMonth`, `DayOfWeek` | int | |
| `YearMonth` | string `yyyy-MM` | joins `dim_month` |
| `QuarterLabel` | string | e.g. `Q3 2026` |
| `YearWeek` | string | e.g. `2026-W07` |
| `DayName`, `MonthName` | string | |
| `DateKey` | int | `yyyyMMdd` |
| `IsPast`, `IsCurrentMonth` | bool | |

Marked as the model's **date table** on `Date`.

### `dim_month` — one row per month (`02_Gold_Calendar`)

| Column | Type | Notes |
|---|---|---|
| `YearMonth` | string `yyyy-MM` | key |
| `MonthStart` | date | first day (used for MoM time-intelligence) |
| `Year` | int | |
| `MonthNum` | int | |
| `MonthName` | string | |
| `Value` | int | constant `1` (auto-date compatibility) |

### `dim_tag_key` — your tag universe (`05_Gold_ByTag`)

Lets the app / report discover the **dynamic** tag columns at runtime. **No relationship** — it is
evaluated directly.

| Column | Type | Notes |
|---|---|---|
| `TagKey` | string | normalized (lower/trim) tag key from the FOCUS `Tags` JSON |
| `TagKeyDisplay` | string | friendly title-cased label |
| `ColumnName` | string | the PascalCase column that key becomes in `gold_chargeback_by_tag` |
| `Rank` | int | rank by total effective cost (1 = highest-spend tag) |
| `EffectiveCost` | double | total cost carrying that tag |

---

## Gold facts

### `gold_cost_summary_daily` (`04`) — daily grain

`Date × SubAccountName × ServiceCategory × ServiceName × RegionName`

| Column | Type |
|---|---|
| `Date` | date |
| `SubAccountName`, `ServiceCategory`, `ServiceName`, `RegionName` | string |
| `EffectiveCost`, `BilledCost`, `ListCost`, `SavingsAmount` | double |

> `SavingsAmount = ListCost − EffectiveCost`. Filtered to `ChargeCategory = 'Usage'`.

### `gold_cost_summary_monthly` (`04`) — monthly grain

`YearMonth × SubAccountName × ServiceCategory × ServiceName`

| Column | Type |
|---|---|
| `YearMonth` | string `yyyy-MM` |
| `SubAccountName`, `ServiceCategory`, `ServiceName` | string |
| `EffectiveCost`, `BilledCost`, `ListCost`, `SavingsAmount` | double |

### `gold_cost_focus_monthly` (`03`) — enriched monthly fact

Adds FOCUS + `x_*` dimensions and a **hybrid savings baseline** (list price for on-demand,
contracted rate for commitments). Includes **all** `ChargeCategory` values.

| Column | Type | Notes |
|---|---|---|
| `YearMonth` | string | |
| `SubAccountName`, `ResourceGroupName`, `ServiceCategory`, `ServiceName`, `RegionName` | string | |
| `ChargeCategory` | string | Usage / Purchase / Tax / Adjustment / Credit |
| `PricingCategory` | string | Standard / Committed / Spot / … |
| `CommitmentDiscountStatus` | string | Used / Unused / N/A |
| `CostCenter` | string | `x_CostCenter` |
| `EffectiveCost`, `BilledCost`, `ListCost`, `ContractedCost` | double | |
| `SavingsBaseline` | double | list (non-commitment) or contracted (commitment) |
| `SavingsAmount` | double | `SavingsBaseline − EffectiveCost` |
| `EffectiveCostUSD` | double | `x_EffectiveCostInUsd` |
| `ResourceCount` | long | distinct `ResourceId` |

### `gold_chargeback_by_tag` (`05`) — WIDE dynamic-tag fact

**One row per resource-month**, so costs are counted exactly once and the app/report can group or
filter by **several tags at the same time with no double counting**.

| Column | Type | Notes |
|---|---|---|
| `YearMonth` | string | |
| `SubAccountName`, `ServiceCategory`, `ServiceName`, `RegionName`, `ResourceGroupName` | string | |
| `ResourceId`, `ResourceName`, `ResourceType` | string | |
| **`<TagColumn>`** (one per discovered key) | string | tag value, or `"Untagged"` if absent |
| `EffectiveCost`, `BilledCost` | double | |
| `TagCount` | int | number of tags on the resource (`0` = fully untagged) |

> The `<TagColumn>` set is **data-dependent** — it is discovered from the FOCUS `Tags` JSON and
> named in PascalCase (`cost center` → `CostCenter`). `dim_tag_key` maps keys → columns → display.

### `gold_cost_by_resource` (`08`) — resource-grain fact (tag-free)

Same resource grain as the WIDE table but **without** per-tag columns; keeps `TagCount` so
governance measures work.

| Column | Type |
|---|---|
| `YearMonth` | string |
| `SubAccountName`, `ResourceGroupName`, `ResourceId`, `ResourceName`, `ResourceType`, `ServiceCategory`, `ServiceName`, `RegionName` | string |
| `EffectiveCost`, `BilledCost`, `ListCost`, `SavingsAmount` | double |
| `TagCount` | int |

### `gold_reservations_coverage` (`06`)

`YearMonth × SubAccountName`

| Column | Type | Notes |
|---|---|---|
| `ReservedCost` | double | committed (`PricingCategory = 'Committed'`) cost |
| `TotalCost` | double | all usage cost |
| `CoveragePercent` | double | `ReservedCost / TotalCost × 100` |

### `gold_reservations_waste` (`06`)

`YearMonth × SubAccountName`

| Column | Type | Notes |
|---|---|---|
| `WasteCost` | double | `EffectiveCost` where `CommitmentDiscountStatus = 'Unused'` |
| `UtilizationPercent` | double | `Used / (Used + Unused) × 100`, null when no commitment |

### `gold_reservations_detail` (`06`) — per-commitment

`YearMonth × SubAccountName × CommitmentDiscount(Id/Name/Type/Category) × ServiceName × RegionName`

| Column | Type | Notes |
|---|---|---|
| `CommitmentDiscountId/Name/Type/Category` | string | which reservation / savings plan |
| `UsedCost`, `WasteCost` | double | used vs idle commitment cost |
| `TotalCommitmentCost` | double | `Used + Unused` |
| `ContractedCostUsed` | double | on-demand-equivalent of used capacity |
| `SavingsUsed` | double | `ContractedCostUsed − UsedCost` |
| `UtilizationPercent` | double | per-commitment utilization |

### `gold_cost_anomalies` (`07`)

`Date × SubAccountName × ServiceCategory × ServiceName × RegionName`

| Column | Type | Notes |
|---|---|---|
| `EffectiveCost` | double | |
| `RollingMean`, `RollingStdDev` | double | trailing window stats |
| `ZScore` | double | `(cost − mean) / stddev` |
| `IsAnomaly` | bool | `abs(ZScore) > 3` |
| `Severity` | string | derived from z-score magnitude |

---

## Semantic model — `AzureCostAnalyzer_SM`

**Direct Lake on SQL** over the Lakehouse SQL analytics endpoint. Built by
`09_Build_SemanticModel` using `semantic-link-labs`.

### 13 bound tables

`focus_partitioned`, `gold_cost_summary_daily`, `gold_cost_summary_monthly`,
`gold_cost_focus_monthly`, `gold_chargeback_by_tag`, `dim_tag_key`, `gold_cost_by_resource`,
`gold_reservations_coverage`, `gold_reservations_waste`, `gold_reservations_detail`,
`gold_cost_anomalies`, `dim_date`, `dim_month`.

### Relationships (Many→One, single-direction)

| From | Column | To | Column |
|---|---|---|---|
| `focus_partitioned` | `Date` | `dim_date` | `Date` |
| `gold_cost_summary_daily` | `Date` | `dim_date` | `Date` |
| `gold_cost_anomalies` | `Date` | `dim_date` | `Date` |
| `gold_cost_summary_monthly` | `YearMonth` | `dim_month` | `YearMonth` |
| `gold_cost_focus_monthly` | `YearMonth` | `dim_month` | `YearMonth` |
| `gold_chargeback_by_tag` | `YearMonth` | `dim_month` | `YearMonth` |
| `gold_cost_by_resource` | `YearMonth` | `dim_month` | `YearMonth` |
| `gold_reservations_coverage` | `YearMonth` | `dim_month` | `YearMonth` |
| `gold_reservations_waste` | `YearMonth` | `dim_month` | `YearMonth` |
| `gold_reservations_detail` | `YearMonth` | `dim_month` | `YearMonth` |
| `dim_date` | `YearMonth` | `dim_month` | `YearMonth` |

`dim_tag_key` is bound but **not related** — the app/report evaluates it to discover the dynamic tag
columns. `dim_date` is the marked **date table**.

### Measures (by display folder)

| Folder | Measure | Home table | DAX (simplified) |
|---|---|---|---|
| Cost | `Total Effective Cost` | gold_cost_summary_monthly | `SUM(gold_cost_summary_monthly[EffectiveCost])` |
| Cost | `Total Billed Cost` | gold_cost_summary_monthly | `SUM(gold_cost_summary_monthly[BilledCost])` |
| Cost | `Total List Cost` | gold_cost_summary_monthly | `SUM(gold_cost_summary_monthly[ListCost])` |
| Cost | `Total Savings` | gold_cost_summary_monthly | `SUM(gold_cost_summary_monthly[SavingsAmount])` |
| Cost | `Savings %` | gold_cost_summary_monthly | `DIVIDE([Total Savings], [Total List Cost])` |
| Cost | `Real Savings` | gold_cost_focus_monthly | `SUM(gold_cost_focus_monthly[SavingsAmount])` |
| Cost | `Total Baseline Cost` | gold_cost_focus_monthly | `SUM(gold_cost_focus_monthly[SavingsBaseline])` |
| FOCUS | `FX Effective Cost` | focus_partitioned | `SUM(focus_partitioned[EffectiveCost])` |
| FOCUS | `FX Consumed Quantity` | focus_partitioned | `SUM(focus_partitioned[ConsumedQuantity])` |
| Time Intelligence | `Last Month Cost` | gold_cost_summary_monthly | `CALCULATE([Total Effective Cost], REMOVEFILTERS(dim_month), dim_month[MonthStart] = EDATE(MAX(dim_month[MonthStart]), -1))` |
| Time Intelligence | `MoM Cost %` | gold_cost_summary_monthly | `DIVIDE([Total Effective Cost] - [Last Month Cost], [Last Month Cost])` |
| Anomalies | `Anomaly Count` | gold_cost_anomalies | `CALCULATE(COUNTROWS(gold_cost_anomalies), gold_cost_anomalies[IsAnomaly] = TRUE())` |
| Anomalies | `Anomaly Count (KPI)` | gold_cost_anomalies | `[Anomaly Count]` |
| Reservations | `RI Coverage %` | gold_reservations_coverage | `DIVIDE(SUM(gold_reservations_coverage[ReservedCost]), SUM(gold_reservations_coverage[TotalCost]))` |
| Reservations | `RI Waste Cost` | gold_reservations_waste | `SUM(gold_reservations_waste[WasteCost])` |
| Reservations | `Net Reservation Savings` | gold_cost_focus_monthly | `CALCULATE(SUM(gold_cost_focus_monthly[SavingsAmount]), gold_cost_focus_monthly[PricingCategory] = "Committed")` |
| Reservations | `Used Reservation Savings` | gold_reservations_waste | `SUM(gold_reservations_detail[SavingsUsed])` |
| Reservations | `Used Reservation Cost` | gold_reservations_detail | `SUM(gold_reservations_detail[UsedCost])` |
| Reservations | `Wasted Reservation Cost` | gold_reservations_detail | `SUM(gold_reservations_detail[WasteCost])` |
| Reservations | `Reservation Utilization %` | gold_reservations_detail | `DIVIDE(SUM(gold_reservations_detail[UsedCost]), SUM(gold_reservations_detail[UsedCost]) + SUM(gold_reservations_detail[WasteCost]))` |
| Governance | `Untagged Cost` | gold_cost_by_resource | `CALCULATE(SUM(gold_cost_by_resource[EffectiveCost]), gold_cost_by_resource[TagCount] = 0)` |
| Governance | `Untagged %` | gold_cost_by_resource | `DIVIDE([Untagged Cost], SUM(gold_cost_by_resource[EffectiveCost]))` |
| Governance | `Untagged Resource Count` | gold_cost_by_resource | `CALCULATE(DISTINCTCOUNT(gold_cost_by_resource[ResourceId]), gold_cost_by_resource[TagCount] = 0)` |
| Governance | `Tag Count` | gold_cost_by_resource | `SUM(gold_cost_by_resource[TagCount])` |
| Governance | `Tags In Filter` | gold_cost_by_resource | `CALCULATE(DISTINCTCOUNT(gold_cost_by_resource[ResourceId]), gold_cost_by_resource[TagCount] > 0)` |
| Resource Detail | `Resource Effective Cost` | gold_cost_by_resource | `SUM(gold_cost_by_resource[EffectiveCost])` |
| Resource Detail | `Resource Count` | gold_cost_by_resource | `DISTINCTCOUNT(gold_cost_by_resource[ResourceId])` |
| Tags | `Tag Effective Cost` | gold_chargeback_by_tag | `SUM(gold_chargeback_by_tag[EffectiveCost])` |
| Tags | `Tagged Resource Count` | gold_chargeback_by_tag | `CALCULATE(DISTINCTCOUNT(gold_chargeback_by_tag[ResourceId]), gold_chargeback_by_tag[TagCount] > 0)` |
| Tags | `Untagged Resource Count (Tags)` | gold_chargeback_by_tag | `CALCULATE(DISTINCTCOUNT(gold_chargeback_by_tag[ResourceId]), gold_chargeback_by_tag[TagCount] = 0)` |

> DAX shown is simplified — the format string (currency / percent / number) of each measure is omitted.

