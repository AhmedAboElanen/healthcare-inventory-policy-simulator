# DAX Tables and Measures

The model imports one table, `Policy_Results`, from the Excel workbook (`Policy_Results` sheet, one row per item per policy). Policy values are `"P1 Baseline"` and `"P2 Optimized"`. Helper tables and What-If parameters are created inside Power BI.

## 1. Core measures (Policy Comparison page)

```dax
Avg Inventory Units = SUM(Policy_Results[AvgInventoryUnits])
Inventory Value = SUM(Policy_Results[InventoryValueEGP])
Holding Cost = SUM(Policy_Results[HoldingCostEGP])
Safety Stock Units = SUM(Policy_Results[SafetyStockUnits])
Avg Orders per Item = AVERAGE(Policy_Results[OrdersPerYear])
Item Count = DISTINCTCOUNT(Policy_Results[ItemID])
```

## 2. Policy-specific measures

Same pattern for each of the five measures above (shown for Inventory Value):

```dax
P1 Inventory Value = CALCULATE([Inventory Value], Policy_Results[Policy] = "P1 Baseline")
P2 Inventory Value = CALCULATE([Inventory Value], Policy_Results[Policy] = "P2 Optimized")
```

Equivalent measures exist for Avg Inventory Units, Holding Cost, Safety Stock Units and Avg Orders per Item (`P1 Avg Inventory Units`, `P2 Holding Cost`, and so on).

## 3. Change percentages

```dax
Avg Inventory Change % = DIVIDE([P2 Avg Inventory Units] - [P1 Avg Inventory Units], [P1 Avg Inventory Units])
Inventory Value Change % = DIVIDE([P2 Inventory Value] - [P1 Inventory Value], [P1 Inventory Value])
Holding Cost Change % = DIVIDE([P2 Holding Cost] - [P1 Holding Cost], [P1 Holding Cost])
Safety Stock Change % = DIVIDE([P2 Safety Stock Units] - [P1 Safety Stock Units], [P1 Safety Stock Units])
Avg Orders Change % = DIVIDE([P2 Avg Orders per Item] - [P1 Avg Orders per Item], [P1 Avg Orders per Item])
```

## 4. Items moved to 3-month coverage

```dax
Items at 3-Month Cover =
CALCULATE(DISTINCTCOUNT(Policy_Results[ItemID]),
    Policy_Results[Policy] = "P2 Optimized", Policy_Results[P2TargetMonths] = 3)

P1 Value (3-Month Items) =
CALCULATE([Inventory Value], Policy_Results[Policy] = "P1 Baseline", Policy_Results[P2TargetMonths] = 3)

P2 Value (3-Month Items) =
CALCULATE([Inventory Value], Policy_Results[Policy] = "P2 Optimized", Policy_Results[P2TargetMonths] = 3)

Value Change % (3-Month Items) =
DIVIDE([P2 Value (3-Month Items)] - [P1 Value (3-Month Items)], [P1 Value (3-Month Items)])

P1 Orders (3-Month Items) =
CALCULATE([Avg Orders per Item], Policy_Results[Policy] = "P1 Baseline", Policy_Results[P2TargetMonths] = 3)

P2 Orders (3-Month Items) =
CALCULATE([Avg Orders per Item], Policy_Results[Policy] = "P2 Optimized", Policy_Results[P2TargetMonths] = 3)

Orders Change % (3-Month Items) =
DIVIDE([P2 Orders (3-Month Items)] - [P1 Orders (3-Month Items)], [P1 Orders (3-Month Items)])
```

## 5. Time Gap page

```dax
Gap_Phases =
DATATABLE(
    "Phase", STRING,
    "PhaseOrder", INTEGER,
    "Policy", STRING,
    "Days", INTEGER,
    {
        {"1. Information lag", 1, "P1 Baseline", 30},
        {"1. Information lag", 1, "P2 Optimized", 0},
        {"2. HQ approval", 2, "P1 Baseline", 40},
        {"2. HQ approval", 2, "P2 Optimized", 15},
        {"3. Supplier delivery", 3, "P1 Baseline", 20},
        {"3. Supplier delivery", 3, "P2 Optimized", 15}
    }
)

Total Gap Days = SUM(Gap_Phases[Days])
P1 Gap Days = CALCULATE([Total Gap Days], Gap_Phases[Policy] = "P1 Baseline")
P2 Gap Days = CALCULATE([Total Gap Days], Gap_Phases[Policy] = "P2 Optimized")
Gap Change % = DIVIDE([P2 Gap Days] - [P1 Gap Days], [P1 Gap Days])

P1 Request-to-Delivery =
CALCULATE([Total Gap Days], Gap_Phases[Policy] = "P1 Baseline", Gap_Phases[PhaseOrder] >= 2)

P2 Request-to-Delivery =
CALCULATE([Total Gap Days], Gap_Phases[Policy] = "P2 Optimized", Gap_Phases[PhaseOrder] >= 2)
```

Consumption surge: a What-If parameter named `Surge` (Modeling, New parameter, Numeric range, 0 to 1, increment 0.05, default 0.25). It generates the table `Surge` and the measure `Surge Value`.

```dax
P1 Extra Consumption (months) = [Surge Value] * [P1 Gap Days] / 30
P2 Extra Consumption (months) = [Surge Value] * [P2 Gap Days] / 30
```

## 6. Simulator page

Seven What-If parameters (Numeric range, decimal). Each generates a table and a measure named `<Name> Value`.

| Parameter | Min | Max | Increment | Default |
|---|---|---|---|---|
| Consumption | 10 | 1000 | 10 | 50 |
| P1 Lead | 0.5 | 4 | 0.5 | 2 |
| P1 Safety | 0.5 | 4 | 0.5 | 2 |
| P1 Max | 2 | 8 | 0.5 | 6 |
| P2 Lead | 0.5 | 4 | 0.5 | 1 |
| P2 Safety | 0.5 | 4 | 0.5 | 1 |
| P2 Max | 2 | 8 | 0.5 | 3 |

Policy measures (shown for P1; P2 uses `P2 Safety Value`, `P2 Lead Value`, `P2 Max Value`):

```dax
Sim SS P1 = [Consumption Value] * [P1 Safety Value]
Sim ROP P1 = [Consumption Value] * ([P1 Safety Value] + [P1 Lead Value])
Sim Max P1 = [Consumption Value] * [P1 Max Value]
Sim Order Qty P1 = [Consumption Value] * MAX([P1 Max Value] - [P1 Safety Value] - [P1 Lead Value], 0)
Sim Peak P1 = [Sim Max P1] - [Consumption Value] * [P1 Lead Value]
Sim Avg Inventory P1 = ([Sim Peak P1] + [Sim SS P1]) / 2
Sim Orders per Year P1 = DIVIDE(12 * [Consumption Value], [Sim Order Qty P1], 0)
```

Comparison:

```dax
Sim Avg Inventory Change % = DIVIDE([Sim Avg Inventory P2] - [Sim Avg Inventory P1], [Sim Avg Inventory P1])
Sim Orders Change % = DIVIDE([Sim Orders per Year P2] - [Sim Orders per Year P1], [Sim Orders per Year P1])
```

Stock-level chart (steady-state sawtooth over 12 months):

```dax
Sim_Time = SELECTCOLUMNS(GENERATESERIES(0, 12, 0.05), "Month", ROUND([Value], 2))

Sim Stock P1 =
VAR t = MAX(Sim_Time[Month])
VAR cycle = DIVIDE([Sim Order Qty P1], [Consumption Value])
RETURN IF(cycle > 0, [Sim Peak P1] - [Consumption Value] * MOD(t, cycle), BLANK())
```

`Sim Stock P2` is identical, using `[Sim Order Qty P2]` and `[Sim Peak P2]`.

Results table (one row per metric, one column per policy):

```dax
Sim_Metrics =
DATATABLE(
    "Metric", STRING,
    "MetricOrder", INTEGER,
    {
        {"Safety stock (units)", 1},
        {"Reorder point (units)", 2},
        {"Max level (units)", 3},
        {"Order quantity (units)", 4},
        {"Peak stock (units)", 5},
        {"Average inventory (units)", 6},
        {"Orders per year", 7}
    }
)

Sim Table Baseline =
VAR m = SELECTEDVALUE(Sim_Metrics[MetricOrder])
VAR v = SWITCH(m,
    1, [Sim SS P1],
    2, [Sim ROP P1],
    3, [Sim Max P1],
    4, [Sim Order Qty P1],
    5, [Sim Peak P1],
    6, [Sim Avg Inventory P1],
    7, [Sim Orders per Year P1])
RETURN IF(ISBLANK(m), BLANK(), FORMAT(v, IF(m = 7, "0.0", "#,0")))
```

`Sim Table Optimized` is identical, using the P2 measures. The `Metric` column is sorted by `MetricOrder`.
