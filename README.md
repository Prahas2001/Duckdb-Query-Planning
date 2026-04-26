# DuckDB — Query Planning & Systems Analysis
### Big Data Engineering | Course Project

A five-phase empirical investigation of DuckDB's internal architecture. This project moves beyond basic querying to reverse-engineer the execution path, identify core design decisions, and **empirically prove** their value through source-code-level modification, controlled benchmarks, and deliberate failure injection.

---

## Repository Structure

```
duckdb-systems-analysis/
│
├── README.md
├── requirements.txt
│
├── phase1_execution_path/
│   └── Duckdb_Phase_1.ipynb
│
├── phase2_design_decisions/
│   └── Duckdb_Phase_2.ipynb
│
├── phase3_benchmarking/
│   └── Duckdb_Phase_3.ipynb
│
├── phase4_vector_lobotomy/
│   ├── README.md
│   ├── vector_size.hpp.patch
│   └── benchmark_results.md
│
└── phase5_failure_analysis/
    └── Duckdb_Phase_5.ipynb
```

---

## Dataset

All experiments use the **US Airline On-Time Performance** dataset (`flights.csv`):
- **5,819,079 rows** across 31 columns
- Covers departure delays, carriers, airports, and flight metadata
- Available on Kaggle: [2015 Flight Delays and Cancellations](https://www.kaggle.com/datasets/usdot/flight-delays)

> The dataset file is not included in this repository due to size. Download `flights.csv` and place it in the root directory before running any notebook.

---

## Phase 1 - Execution Path Mapping

**Notebook:** `phase1_execution_path/Duckdb_Phase_1.ipynb`

Traced how DuckDB processes a raw SQL query down to the physical execution layer.

| Sub-experiment | What was tested | Evidence produced |
|---|---|---|
| 1A — Parser | Intentional syntax error (`SELCT`) | `ParserException` thrown immediately |
| 1B — Binder | Wrong column names (`CARRIER`, `DEP_DELAY`) | `BinderException` with candidate suggestions |
| 1C — Physical Planner | `EXPLAIN` on aggregation query | Full DAG: `READ_CSV → FILTER → HASH_GROUP_BY → ORDER_BY` |
| 1D — Execution Engine | Full query on 5.8M rows | Result in ~1.86s with timing |

**Key finding:** The pipeline has distinct, ordered stages. Syntax is validated before the catalog is consulted, and the catalog is consulted before data is touched.

---

## Phase 2 - Core Design Decisions

**Notebook:** `phase2_design_decisions/Duckdb_Phase_2.ipynb`

Identified and empirically probed three architectural pillars. For each: where it lives in code, what problem it solves, and what tradeoff it introduces.

---

### 2A - PostgreSQL Parser

**Code location:** DuckDB embeds the PostgreSQL parser as a submodule under `third_party/libpg_query/`. It is invoked as the first stage of every query before any catalog or data access occurs.

**Problem it solves:** Writing a robust, production-grade SQL parser from scratch is a multi-year effort. By reusing PostgreSQL's battle-tested parser, DuckDB gets full SQL syntax validation - including deeply nested expressions, CTEs, and window functions - for free.

**Tradeoff:** DuckDB inherits PostgreSQL's SQL dialect and cannot diverge from it without forking and maintaining the parser independently. Any syntax that PostgreSQL does not support, DuckDB cannot support through this layer alone.

**Empirical proof - three exception types proving three ordered pipeline stages:**

```
ParserException  → "SELCT" syntax error       (Stage 1: caught before catalog lookup)
CatalogException → nonexistent_table          (Stage 2: parser passed, table checked)
BinderException  → "DEP_DELAY" column missing (Stage 3: table found, column resolved)
```

Three structurally similar queries, three distinct exception classes - proving the pipeline has independent, ordered validation stages. The binder also suggested `DEPARTURE_DELAY` as a candidate column, showing active metadata consultation.

---

### 2B - Vectorized Execution (Morsel-Driven)

**Code location:** `src/include/duckdb/common/vector_size.hpp` defines `STANDARD_VECTOR_SIZE 2048U` — the number of rows processed per execution chunk. This is a compile-time constant, not a runtime setting.

**Problem it solves:** Row-at-a-time execution (the Volcano model used by older databases) causes a function call per row. On 5.8M rows, that is 5.8 million individual dispatch operations. Vectorized execution batches 2048 rows per call, enabling SIMD (Single Instruction, Multiple Data) CPU instructions that process multiple values in a single clock cycle and keeping the L1 cache warm between operations.

**Tradeoff:** Fixed-width columnar vectors require data to be laid out contiguously in memory. This is ideal for analytical (OLAP) workloads with uniform data types, but is a poor fit for highly variable, nested, or semi-structured data where row layouts differ significantly between records.

**Empirical proof:** Attempting to vary `STANDARD_VECTOR_SIZE` via the Python API's `SET` pragma produced no measurable effect - confirming it is a compile-time constant. The only valid test was source-level modification (Phase 4), which produced a **2.92× CPU overhead** when the vector size was reduced from 2048 to 2.

---

### 2C - Buffer Manager (Out-of-Core Spill)

**Code location:** DuckDB's buffer manager is implemented in `src/storage/buffer/` and `src/storage/buffer_manager.cpp`. It intercepts memory allocations during query execution and automatically redirects overflow to disk when the configured `memory_limit` is exceeded.

**Problem it solves:** Analytical queries over large datasets frequently require intermediate state (e.g., hash tables for GROUP BY, sort buffers for ORDER BY) that exceeds available RAM. Without a buffer manager, these queries would crash. DuckDB's buffer manager transparently spills these intermediate states to `.tmp` files, allowing queries to complete on datasets larger than physical memory.

**Tradeoff:** Spilling to disk introduces significant latency - disk I/O is orders of magnitude slower than RAM. Empirically measured at **2.66–3.69× overhead** depending on OS I/O scheduling. If both RAM and disk are exhausted simultaneously, the buffer manager throws an `OutOfMemoryException` and aborts — there is no third tier of storage to fall back to.

**Empirical proof - memory constrained to 300MB on 5.8M row aggregation:**

| Condition | Time | Spill volume |
|---|---|---|
| No memory limit | 1.562s | None |
| 300MB cap | 4.149s | **138 MB** written to system temp |

On Windows, `SET temp_directory` was silently overridden — DuckDB used `C:\Users\...\AppData\Local\Temp` instead of the specified path. The 138MB of `.tmp` files were identified in the system temp directory, confirming active spill. This platform-specific behaviour is a production deployment consideration.

---

### Design Decision Tradeoff Summary

| Decision | Benefit | Tradeoff |
|---|---|---|
| PostgreSQL parser | Battle-tested SQL parsing, zero maintenance | Locked to PostgreSQL dialect, cannot diverge without forking |
| Vectorized execution (2048U) | SIMD throughput, L1 cache efficiency | Requires fixed-width columnar layout — poor fit for nested/variable data |
| Buffer manager spill | Handles datasets larger than RAM | 2.66–3.69× latency penalty on spill; hard abort if disk also exhausted |
| In-process architecture* | Zero network latency (0.056s queries) | No fault isolation — host crash kills database with no recovery |

*The in-process tradeoff is the architectural thread connecting Phase 2 to Phase 5.

---

## Phase 3 - Big Data Pipeline Context

**Notebook:** `phase3_benchmarking/Duckdb_Phase_3.ipynb`

Same aggregation query on the same 5.8M row dataset across three engines.

| Sub-experiment | What was tested | Evidence produced |
|---|---|---|
| 3A — Engine Setup | Load flights.csv into SQLite `.db` file | 5,819,079 rows confirmed loaded |
| 3A (Fix) — SQLite Rebuild | Detected empty DB from failed prior run, forced rebuild | Row count verified before timing |
| 3B — Three-Engine Benchmark | DuckDB (CSV) vs SQLite (pre-loaded) vs Pandas (CSV) | Initial timing with methodological anomaly identified |
| 3C — Fair Comparison | DuckDB pre-loaded `.duckdb` vs SQLite pre-loaded `.db` | Pure execution model comparison on equal footing |

### 3A - Engine Setup

SQLite was chosen as the row-based comparison engine. To eliminate CSV parsing from its benchmark time, all 5,819,079 rows were pre-loaded into a binary `.db` file as a one-time setup step. A DB rebuild was required after an initial empty-DB error was detected (3A Fix) - this is documented transparently in the notebook.

### 3B - Three-Engine Benchmark (Initial Run)

| Engine | Time | vs DuckDB | Architecture |
|---|---|---|---|
| DuckDB (from raw CSV) | 2.862s | 1.0× baseline | Columnar vectorized, in-process |
| SQLite (pre-loaded) | 2.186s | 0.76× | Row-based scan, no vectorization |
| Pandas (from CSV) | 91.857s | **32.1× slower** | Full RAM load + Python interpreter |

**Anomaly identified:** SQLite appeared faster than DuckDB. This was flagged as a structural confound — SQLite skipped CSV parsing (data was pre-loaded), while DuckDB parsed raw CSV on every run. The comparison was not on equal footing.

### 3C - Fair Comparison (DuckDB Pre-loaded vs SQLite Pre-loaded)

To isolate pure execution model performance, DuckDB was benchmarked against a pre-loaded `.duckdb` file — eliminating CSV parse cost from both engines.

| Engine | Time | vs DuckDB | What it isolates |
|---|---|---|---|
| **DuckDB** (pre-loaded) | **0.056s** | 1.0× baseline | Pure columnar vectorized execution |
| SQLite (pre-loaded) | 2.186s | **39.3× slower** | Pure row-based scan execution |

**Key finding:** On equal footing, DuckDB is **39.3× faster** than SQLite. The gap is entirely attributable to execution model - SQLite performs a full row scan reading all 31 columns per row to retrieve 2, while DuckDB's columnar storage reads only the projected columns (`AIRLINE`, `DEPARTURE_DELAY`) in SIMD chunks.

### Why Pandas is 32× Slower Than DuckDB (from CSV)

Two compounding costs DuckDB eliminates entirely:
1. Full materialisation of all 31 columns × 5.8M rows into RAM before any filtering occurs
2. Aggregation executed in Python's interpreted runtime with no SIMD instructions

DuckDB projects only the required columns during streaming parse — never materialising unused data. Against pre-loaded DuckDB (0.056s), Pandas is **1,640× slower**.

---

## Phase 4 - The Vector Lobotomy *(Source-Level Experiment)*

**Directory:** `phase4_vector_lobotomy/`

The core experiment of this project. To empirically prove the necessity of vectorized execution, DuckDB's engine was modified at the C++ source level and recompiled.

### The Modification

**File:** `src/include/duckdb/common/vector_size.hpp`

```cpp
// Before
#define STANDARD_VECTOR_SIZE 2048U

// After
#define STANDARD_VECTOR_SIZE 2U
```

This forces the execution engine to process data in 2-row chunks instead of 2048-row chunks, destroying SIMD capability.

### Benchmark Results

| Metric | Default Engine (2048U) | Modified Engine (2U) | Impact |
|---|---|---|---|
| Wall-clock time (real) | 1.53s | 3.337s | **2.18× slower** |
| CPU user time | 6.81s | 19.875s | **2.92× overhead** |
| CPU sys time | 0.00s | 0.75s | **New kernel overhead** |

### Interpretation

The wall-clock doubling understates the damage. The critical metric is **CPU user time** - 19.875s vs 6.81s proves the CPU spent extra cycles on function call overhead and cache misses across 2.9 million tiny 2-row chunks, rather than mathematical aggregations. The appearance of `sys` time (0 → 0.75s) reveals kernel-level memory management calls that never occurred with the default 2048-row chunks.

### Reproduction Steps

See `phase4_vector_lobotomy/README.md` for full steps. Requires CMake 3.x and MSVC (Windows) or GCC/Clang (Linux/Mac).

---

## Phase 5 - Failure Analysis

**Notebook:** `phase5_failure_analysis/Duckdb_Phase_5.ipynb`

Three distinct failure modes empirically demonstrated.

### 5A - OOM Hard Abort (memory_limit = 100MB)

```
duckdb.OutOfMemoryException:
Out of Memory Error: could not allocate block of size 30.5 MiB (66.9 MiB/95.3 MiB used)
```

The buffer manager attempted to manage the query, calculated exactly how much memory was available, and threw a clean exception when a required allocation was impossible. **No partial result, no recovery.**

### 5B - OOM Graceful Spill (memory_limit = 300MB)

| Run | Time |
|---|---|
| Baseline (no limit) | 1.877s |
| Spill run 1 | 6.280s |
| Spill run 2 | 5.158s |
| Spill run 3 | 4.765s |
| **Average overhead** | **2.88×** |

138 MB spilled to disk. Query completed successfully. Variance across runs (1.515s range) is attributable to Windows I/O scheduling - disk spill benchmarks on a non-real-time OS are inherently noisy.

### 5C - Host Process Interference (In-Process Crash)

```python
# Query running in background thread
# Host force-closes the connection mid-query
con.close()

# Query result:
('error', 'Error', 'bad_weak_ptr')
```

`bad_weak_ptr` is a C++ memory signal — the connection object was destroyed in host memory while the query thread still held a reference to it. This is not a clean "connection closed" error. It is the raw consequence of shared memory: **the host killed the object; the query thread found a corpse.**

### Failure Mode Summary

| Mode | Trigger | Evidence | Behaviour |
|---|---|---|---|
| 5A: Hard OOM | 100MB cap | `OutOfMemoryException` | Violent abort, no recovery |
| 5B: Soft OOM | 300MB cap | 138MB spill, 2.88× overhead | Graceful degradation, completes |
| 5C: Host crash | `con.close()` mid-query | `bad_weak_ptr` at C++ level | Immediate failure, no HA |

**Architecture conclusion:** DuckDB has a two-tier failure response. Memory pressure it can manage → graceful spill. Memory pressure it cannot manage → clean exception. Host-level failure → no defence, shared fate. This is the direct consequence of the in-process architecture — the same design that delivers 0.056s query execution eliminates the fault-tolerance boundary that Spark's master/worker model provides.

---

## Setup & Requirements

```bash
pip install -r requirements.txt
```

**requirements.txt:**
```
duckdb>=0.10.0
pandas>=2.0.0
psutil>=5.9.0
jupyter>=1.0.0
```

> PySpark is listed as optional. Phase 3 uses Pandas as the distributed-framework comparison due to Java/JVM dependency complexity. If you have Java 11+ installed, PySpark can be substituted.

### Phase 4 Requirements (Source Compilation)
- CMake 4.x
- MSVC 19.x (Windows) / GCC or Clang (Linux/Mac)
- ~20 minutes build time
- ~4GB disk space for build artifacts

---

## Concept Mapping

Relating DuckDB's architecture to core systems concepts taught in the course.

| Concept | Where it appears in DuckDB | Evidence from this project |
|---|---|---|
| **Columnar Storage** | Data laid out column-by-column in memory vectors, not row-by-row | Phase 3: 39.3× faster than SQLite (row-based) on same query |
| **DAG Execution** | Physical planner produces a Directed Acyclic Graph of operators (`READ_CSV → FILTER → HASH_GROUP_BY → ORDER_BY`) | Phase 1C: `EXPLAIN` output shows the full operator DAG |
| **Fault Tolerance** | DuckDB has none at the host level — in-process means shared fate | Phase 5C: `bad_weak_ptr` on host interference; no WAL, no replication |
| **Reliability under resource pressure** | Buffer manager provides graceful degradation via out-of-core spill | Phase 5B: 138MB spill, query completes; Phase 5A: hard abort when spill impossible |
| **Partitioning / Parallelism** | Morsel-driven parallelism breaks data into dynamic chunks distributed across threads | Phase 4: vector size controls chunk granularity; reducing it 1024× causes 2.92× CPU overhead |
| **Storage vs Execution tradeoff** | OLAP columnar layout optimised for read-heavy aggregation, not write-heavy OLTP | Phase 3: DuckDB 39.3× faster than SQLite for analytical query; SQLite remains better for point lookups |

---

## Key Findings Summary

| Phase | Finding | Metric |
|---|---|---|
| 1 | Three distinct pipeline stages confirmed | 3 exception types |
| 2 | Buffer manager spills 138MB gracefully | 2.66× slowdown |
| 3 | DuckDB 39.3× faster than SQLite on equal footing | 0.056s vs 2.186s |
| 3 | Pandas 1,640× slower due to full materialisation | 0.056s vs 91.857s |
| 4 | Vector size 2U causes 2.92× CPU overhead | 19.875s vs 6.81s user time |
| 5 | In-process architecture has no host-crash defence | `bad_weak_ptr` |

---

## Tools & Environment

- **Language:** Python 3.10.11
- **DuckDB:** pip-installed library (Phases 1–3, 5) + compiled from source (Phase 4)
- **OS:** Windows 11
- **Hardware:** x86-64, MSVC compiler v19.50
- **Dataset:** 5,819,079 rows, 31 columns, US airline flights 2015
