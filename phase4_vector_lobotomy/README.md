# Phase 4 - The Vector Lobotomy

## Objective
Empirically prove the necessity of vectorized execution by modifying DuckDB's core vector size constant at the C++ source level and recompiling the engine.

## Why Source-Level Modification Was Necessary
Phase 2 (Experiment 2B) demonstrated that `STANDARD_VECTOR_SIZE` cannot be changed at runtime via the Python API's `SET` pragma - it is a compile-time constant baked into the binary. The only valid way to test this architectural pillar was to modify and recompile the C++ source.

## The Modification

**File:** `src/include/duckdb/common/vector_size.hpp`

```cpp
// BEFORE (default engine)
#define STANDARD_VECTOR_SIZE 2048U

// AFTER (modified engine — "lobotomised")
#define STANDARD_VECTOR_SIZE 2U
```

**Effect:** Forces the execution engine to process data in 2-row chunks instead of 2048-row chunks, destroying SIMD (Single Instruction, Multiple Data) capability. With 5.8M rows, this creates approximately 2.9 million individual chunk-processing operations instead of ~2,842.

## Reproduction Steps

### Prerequisites
- CMake 3.x or higher
- C++ compiler: MSVC 19.x (Windows) or GCC/Clang (Linux/Mac)
- ~4GB free disk space for build artifacts
- ~15–20 minutes build time

### Steps

**1. Clone DuckDB source**
```bash
git clone https://github.com/duckdb/duckdb.git
cd duckdb
```

**2. Apply the patch**
```bash
git apply vector_size.hpp.patch
```

Or manually edit the file:
```bash
# Windows
notepad src\include\duckdb\common\vector_size.hpp

# Linux/Mac
nano src/include/duckdb/common/vector_size.hpp
```

**3. Verify the change**
```bash
git diff src/include/duckdb/common/vector_size.hpp
```

**4. Compile (Windows — Developer Command Prompt)**
```bash
mkdir build
cd build
cmake .. -DCMAKE_BUILD_TYPE=Release
cmake --build . --config Release
```

**4. Compile (Linux/Mac)**
```bash
mkdir build && cd build
cmake .. -DCMAKE_BUILD_TYPE=Release
make -j$(nproc)
```

**5. Verify the binary**
```bash
# Windows
.\build\Release\duckdb.exe

# Linux/Mac
./build/duckdb
```

**6. Run the benchmark**
```sql
.timer on
SELECT AIRLINE, count(*) as delayed_flights, avg(DEPARTURE_DELAY) as avg_delay
FROM read_csv_auto('/path/to/flights.csv')
WHERE DEPARTURE_DELAY > 15
GROUP BY AIRLINE
ORDER BY delayed_flights DESC;
```

## Benchmark Results

Dataset: US Airline On-Time Performance, 5,819,079 rows

| Metric | Default Engine (2048U) | Modified Engine (2U) | Impact |
|---|---|---|---|
| Wall-clock time (real) | 1.53s | 3.337s | **2.18× slower** |
| CPU user time | 6.81s | 19.875s | **2.92× overhead** |
| CPU sys time | 0.00s | 0.75s | **New kernel overhead** |

## Interpretation

### Why User Time is the Critical Metric
The wall-clock doubling (2.18×) understates the actual damage. The CPU user time (2.92× overhead) is what reveals the mechanism:

- **Default (2048U):** ~2,842 chunk operations. Each chunk is large enough for SIMD instructions to process 2048 rows in parallel per CPU instruction. L1 cache is kept warm between operations.

- **Modified (2U):** ~2,909,540 chunk operations. Each chunk is too small for SIMD. The CPU spends most of its time on C++ function call overhead and cache misses rather than mathematical aggregations.

### The sys Time Finding
The appearance of sys time (0.00s → 0.75s) is an additional finding: with 2.9 million tiny chunks, the engine began making OS-level kernel calls for memory management that never occurred with 2048-row chunks. This overhead does not exist in the default engine.

### Connection to Phase 2
This experiment directly validates what Phase 2 (Experiment 2B) hypothesised - that `STANDARD_VECTOR_SIZE` is a compile-time constant controlling SIMD batch size. The Python API cannot expose this control. Phase 4 is the only empirical proof of this pillar's value.
