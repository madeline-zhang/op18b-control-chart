# DAX reference

Every calculation in the report, in the order it was built. All of them live on the single `op18b` table.

## Calculated columns

Calculated columns are computed once per row and stored in the table.

### PeriodIndex
Numbers each hospital's reporting periods 1–7 in date order. `EARLIER` refers back to the row being calculated, so each row is ranked only against rows from the same hospital.

```dax
PeriodIndex =
RANKX (
    FILTER ( op18b, op18b[hospital] = EARLIER ( op18b[hospital] ) ),
    op18b[period_end], , ASC, DENSE
)
```

### PrevMinutes
The same hospital's OP-18b value from the period before. Blank for period 1.

```dax
PrevMinutes =
VAR h = op18b[hospital]
VAR i = op18b[PeriodIndex]
RETURN
    MAXX (
        FILTER ( op18b, op18b[hospital] = h && op18b[PeriodIndex] = i - 1 ),
        op18b[median_minutes]
    )
```

### MovingRange
The absolute change from the previous period: the "mR" in XmR. Blank for a hospital's first period, because there is nothing before it to compare with.

```dax
MovingRange =
IF (
    ISBLANK ( op18b[PrevMinutes] ),
    BLANK (),
    ABS ( op18b[median_minutes] - op18b[PrevMinutes] )
)
```

## Measures

Measures are evaluated live in whatever filter context the visual supplies. `ALLEXCEPT ( op18b, op18b[hospital] )` drops the period filter that comes from the chart's x-axis but keeps the hospital chosen in the slicer. That is why the center line and the limits stay flat across every point.

```dax
Median Minutes = AVERAGE ( op18b[median_minutes] )
```
Each point on the chart is a single row, so this returns that period's value.

```dax
Center Line =
CALCULATE ( AVERAGE ( op18b[median_minutes] ), ALLEXCEPT ( op18b, op18b[hospital] ) )
```
X̄, the hospital's mean across all periods.

```dax
Avg Moving Range =
CALCULATE ( AVERAGE ( op18b[MovingRange] ), ALLEXCEPT ( op18b, op18b[hospital] ) )
```
mR̄. `AVERAGE` skips the blank first period, so 7 periods give 6 moving ranges.

```dax
UCL = [Center Line] + 2.66 * [Avg Moving Range]

LCL = MAX ( 0, [Center Line] - 2.66 * [Avg Moving Range] )
```
2.66 = 3 ÷ d₂ (1.128 for subgroups of size 2). It converts mR̄ into a 3-sigma estimate.

```dax
Signal Point =
IF ( [Median Minutes] > [UCL] || [Median Minutes] < [LCL], [Median Minutes] )
```
Returns a value only for points outside the limits. On the chart it is drawn as red markers with no line, a simpler approach than conditional formatting.

```dax
Latest Minutes =
VAR lastIdx = CALCULATE ( MAX ( op18b[PeriodIndex] ), ALLEXCEPT ( op18b, op18b[hospital] ) )
RETURN
    CALCULATE (
        MAX ( op18b[median_minutes] ),
        ALLEXCEPT ( op18b, op18b[hospital] ),
        op18b[PeriodIndex] = lastIdx
    )
```
The most recent value for the selected hospital, shown in a card.

## Expected values

These are checked against an independent pandas calculation:

| Hospital | Center line (min) | Avg moving range | UCL | LCL |
| --- | --- | --- | --- | --- |
| University Medical Center | 263.4 | 13.50 | 299.3 | 227.5 |
| Wilson County Hospital | 174.1 | 5.17 | 187.9 | 160.4 |
| Tullahoma-Harton Hospital | 172.3 | 5.67 | 187.4 | 157.2 |
| Bedford Hospital | 125.6 | 2.67 | 132.7 | 118.5 |
