# Phase 4 — Benchmark Results

## Experiment Setup
- **Dataset:** flights.csv - US Airline On-Time Performance 2015
- **Rows:** 5,819,079
- **Query:** Aggregation of delayed flights (DEPARTURE_DELAY > 15) grouped by AIRLINE
- **Hardware:** Windows 11, x86-64, MSVC 19.50
- **Method:** DuckDB CLI shell with `.timer on`

## Default Engine (STANDARD_VECTOR_SIZE = 2048U)

```
Run Time (s): real 1.530   user 6.810000   sys 0.000000
```

Result:
```
┌─────────┬─────────────────┬────────────────────┐
│ AIRLINE │ delayed_flights │     avg_delay      │
├─────────┼─────────────────┼────────────────────┤
│ WN      │          254138 │   52.415...        │
│ AA      │          119456 │   64.377...        │
│ DL      │          118136 │   62.633...        │
│ UA      │          116153 │   64.743...        │
│ EV      │           94024 │   68.570...        │
│ OO      │           93665 │   66.414...        │
│ B6      │           55779 │   63.278...        │
│ MQ      │           54572 │   64.134...        │
│ NK      │           30641 │   66.447...        │
│ US      │           28053 │   56.249...        │
│ F9      │           20154 │   72.138...        │
│ AS      │           17923 │   54.904...        │
│ VX      │           10630 │   59.373...        │
│ HA      │            5234 │   48.657...        │
└─────────┴─────────────────┴────────────────────┘
```

## Modified Engine (STANDARD_VECTOR_SIZE = 2U)

```
Run Time (s): real 3.337   user 19.875000   sys 0.750000
```

Same result set produced (correctness preserved).

## Comparison Table

| Metric | Default (2048U) | Modified (2U) | Ratio |
|---|---|---|---|
| Wall-clock time | 1.530s | 3.337s | **2.18×** |
| CPU user time | 6.810s | 19.875s | **2.92×** |
| CPU sys time | 0.000s | 0.750s | **∞ (new)** |
| Chunk operations (approx.) | ~2,842 | ~2,909,540 | **1024×** |

## Key Observation

The results are **identical** - both engines produce the same 14 rows with the same values. The modification degrades performance without compromising correctness. This confirms the vector size controls execution efficiency, not execution logic.

The 2.92× CPU overhead with only 2.18× wall-clock slowdown indicates the additional CPU work is partially masked by I/O wait time. In a memory-resident scenario (pre-loaded data), the CPU overhead ratio would be even more pronounced.
