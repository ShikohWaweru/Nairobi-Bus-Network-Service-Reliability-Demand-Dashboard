# DAX Measures & Model Logic

Every measure in the model, with the reasoning behind it. Written against a star schema where `Trips` is the fact table and `Fleet`, `Routes` and `Calendar` filter into it one-to-many.

---

## Service Reliability

### Total Trips

```dax
Total Trips =
COUNTROWS ( Trips )
```

Counts **scheduled** service, not delivered service. Trips with no logged actual time are deliberately retained — see *Design note 1*.

---

### Average Start Delay

```dax
Average Start Delay =
AVERAGEX (
    FILTER ( Trips, NOT ISBLANK ( Trips[ActualStartTime] ) ),
    DATEDIFF ( Trips[ScheduledStartTime], Trips[ActualStartTime], MINUTE )
)
```

`AVERAGEX` over a filtered table rather than `AVERAGE` on a calculated column, so blank telemetry doesn't drag the average toward zero. Returns minutes; negative values (early departures) are kept, because an early bus is also a schedule adherence failure.

---

### On-Time Performance %

```dax
On-Time Performance % =
VAR Tolerance = 5
VAR OnTime =
    CALCULATE (
        COUNTROWS ( Trips ),
        FILTER (
            Trips,
            NOT ISBLANK ( Trips[ActualStartTime] )
                && DATEDIFF ( Trips[ScheduledStartTime], Trips[ActualStartTime], MINUTE ) <= Tolerance
        )
    )
RETURN
    DIVIDE ( OnTime, [Total Trips] )
```

The **5-minute tolerance** follows standard urban transit practice. `DIVIDE` rather than `/` so an empty slicer selection returns blank instead of an error.

The denominator is `[Total Trips]` — *all* scheduled trips. An unlogged trip counts against punctuality, because from the passenger's point of view a bus that never arrived was not on time.

---

## Demand & Utilization

### Total Passengers

```dax
Total Passengers =
SUM ( Passengers[PassengerCount] )
```

---

### Passenger Load Factor

```dax
Passenger Load Factor =
DIVIDE (
    [Total Passengers],
    SUMX ( Trips, RELATED ( Fleet[Capacity] ) )
)
```

The important measure in the model. It is a **weighted ratio** — total passengers over total seats *offered* — not an average of per-trip percentages. Averaging percentages would give a 25-seat mini bus the same weight as a 60-seat articulated bus and quietly distort the number.

`SUMX` with `RELATED` pulls capacity across the `Fleet → Trips` relationship at row level, so the measure stays correct under any slicer combination.

---

### Overcrowded Trips

```dax
Overcrowded Trips =
VAR ComfortThreshold = 0.8
RETURN
    COUNTROWS (
        FILTER (
            Trips,
            DIVIDE (
                RELATED ( Passengers[PassengerCount] ),
                RELATED ( Fleet[Capacity] )
            ) > ComfortThreshold
        )
    )
```

Overcrowding is set at **80% of capacity**, not 100%. A bus at 95% of seated capacity is already a poor passenger experience — dwell times rise, boarding slows, and delay compounds. Measuring at the comfort threshold surfaces the problem *before* it becomes a delay statistic.

---

### % Trips Overcrowded

```dax
% Trips Overcrowded =
DIVIDE ( [Overcrowded Trips], [Total Trips] )
```

Paired with the absolute count in the routes table so a high-volume route and a low-volume route can be compared fairly.

---

## Derived columns (Power Query / DAX)

### Departure Hour Label

```dax
Departure Hour Label =
FORMAT ( HOUR ( Trips[ScheduledStartTime] ), "00" ) & ":00"
```

Buckets schedules into hourly bands for the demand-profile column chart. Sorted by a hidden numeric `Departure Hour` column so `06:00` precedes `12:00` rather than sorting alphabetically.

---

### Start Delay (Minutes)

```dax
Start Delay (Minutes) =
DATEDIFF ( Trips[ScheduledStartTime], Trips[ActualStartTime], MINUTE )
```

Materialised as a column rather than recalculated inside every measure — cheaper at this grain and it makes the delay distribution directly inspectable.

---

## Design notes

**1 — Blank telemetry is signal, not noise.**
120 trips (3.0%) have a scheduled departure and no actual times. The reflex is to filter them out. This model keeps them: they are counted in `Total Trips`, excluded from `Average Start Delay` (where they'd be meaningless), and counted as *not* on time in `On-Time Performance %` (where their absence is exactly the point). Three different treatments, each chosen to match what the measure is trying to say.

**2 — No bidirectional relationships.**
Every relationship is single-direction, many-to-one from `Trips` outward. `Passengers` sits one-to-one on `TripID`. The model has zero ambiguous filter paths, which means no `CROSSFILTER` patches and no measures that silently break when a new visual is added.

**3 — `Calendar` is a marked date table.**
182 continuous days with no gaps, marked as the model's date table so time intelligence functions resolve correctly and the monthly trend line sorts chronologically rather than alphabetically.

**4 — Thresholds are variables, not literals.**
`Tolerance` and `ComfortThreshold` are declared as `VAR`s at the top of each measure. If the operator's SLA changes from 5 to 3 minutes, it's a one-line edit in one place — and the assumption is visible to anyone reading the code instead of buried in a filter expression.
