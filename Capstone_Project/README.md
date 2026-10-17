# Low-Latency Order Book — Capstone

**Student:** Shlok Gupta | **SST Roll Number:** 24BCS10322
**Course:** Scaler HFT C++ 2028  
**Deadline:** 17th October 2026, 11:59 PM  
**Language:** C++17

---

## Table of Contents
1. [Project Overview](#project-overview)
2. [How to Build](#how-to-build)
3. [How to Run](#how-to-run)
4. [Interface Changes](#interface-changes)
5. [Data Structure Choices](#data-structure-choices)
6. [Complexity Table](#complexity-table)
7. [Correctness Results](#correctness-results)
8. [Performance Results](#performance-results)
9. [perf Analysis](#perf-analysis)
10. [Performance Report](#performance-report)

---

## Project Overview

This project implements a single-instrument, single-threaded resting order book in three genuinely different ways, benchmarks them against a deterministic 1 000 000-operation workload, and analyses their performance using `std::chrono` and Linux `perf`.

---

## How to Build

### Option 1 — CMake (recommended)

```bash
# From the project root
mkdir build && cd build
cmake .. -DCMAKE_BUILD_TYPE=Release
make -j$(nproc)
# Binary: build/bench
```

### Option 2 — Single g++ command

```bash
g++ -std=c++17 -O2 -march=native -funroll-loops -Wall -Wextra \
    -I include \
    src/bench.cpp \
    -o bench
```

---

## How to Run

### Correctness check

```bash
./bench --verify
```

Expected output:

```
Running correctness test...

Implementation A: PASS
Implementation B: PASS
Implementation C: PASS

All implementations produce identical results.
```

### Benchmarks

```bash
./bench --impl A
./bench --impl B
./bench --impl C
```

Each run executes the workload **5 times** and reports the **median** result.

### perf profiling (Linux only)

```bash
# perf stat for each implementation
perf stat -r 5 \
  -e task-clock,cycles,instructions,branches,branch-misses,\
cache-misses,L1-dcache-load-misses,dTLB-load-misses,page-faults \
  ./bench --impl A

perf stat -r 5 \
  -e task-clock,cycles,instructions,branches,branch-misses,\
cache-misses,L1-dcache-load-misses,dTLB-load-misses,page-faults \
  ./bench --impl B

perf stat -r 5 \
  -e task-clock,cycles,instructions,branches,branch-misses,\
cache-misses,L1-dcache-load-misses,dTLB-load-misses,page-faults \
  ./bench --impl C

# perf record + report
perf record -g ./bench --impl A && perf report
perf record -g ./bench --impl B && perf report
perf record -g ./bench --impl C && perf report
```

---

## Interface Changes

The public API matches the assignment specification exactly.  
One deliberate internal change:

| Change | Reason |
|--------|--------|
| Prices stored as **integer ticks** internally (`int tick = round(price * 100)`) | Avoids floating-point comparison issues; tick arithmetic is exact and fast |
| `add_order` / `cancel_order` still accept `double price` externally | Fully backward-compatible with the specified API |

All three implementations share the same external interface so the benchmark can test them interchangeably.

---

## Data Structure Choices

### Implementation A — `std::map` (Red-Black Tree)

| Aspect | Detail |
|--------|--------|
| Level storage | `std::map<int, long>` (tick → aggregate qty), one per side |
| Order storage | `std::unordered_map<uint64_t, Order>` |
| Best bid | `bids_.rbegin()` — **O(1)** (std::map maintains begin/end in O(1)) |
| Best ask | `asks_.begin()` — **O(1)** (std::map maintains begin/end in O(1)) |
| Find order for cancel | `orders_.find(id)` — O(1) average |

The red-black tree keeps price levels sorted automatically. Its main cost is pointer-chasing (poor cache locality) and O(log N) insertion/deletion. Iterator access to begin/rbegin is O(1) — the tree keeps a pointer to the min and max nodes.

---

### Implementation B — Sorted `std::vector`

| Aspect | Detail |
|--------|--------|
| Level storage | `std::vector<PriceLevel>` sorted descending (bids) or ascending (asks) |
| Order storage | `std::unordered_map<uint64_t, Order>` |
| Best bid | `bids_.front()` — O(1) |
| Best ask | `asks_.front()` — O(1) |
| Find order for cancel | `orders_.find(id)` O(1) + binary search O(log N) + shift O(N) |

Vectors are contiguous in memory — excellent cache behaviour for reads, but insertion/deletion must shift elements. With ≤ 10 000 levels, this is fast in practice.

---

### Implementation C — Fixed-Size Tick-Indexed Array (Custom Design)

| Aspect | Detail |
|--------|--------|
| Level storage | `std::array<long, 10001>` indexed by `tick - TICK_MIN` |
| Order storage | `std::unordered_map<uint64_t, Order>` |
| Best bid | Cached `best_bid_` cursor — O(1) read |
| Best ask | Cached `best_ask_` cursor — O(1) read |
| Find order for cancel | `orders_.find(id)` O(1) + direct array slot update O(1) + cursor repair O(N) amortised O(1) |

The level array is ~80 KB and fits comfortably in L2 cache. All level updates are a single array write. BBO is always O(1). The cursor repair walk only happens when the best level empties, which is rare in typical markets.

---

## Complexity Table

| Operation | Impl A | Impl B | Impl C |
|-----------|--------|--------|--------|
| `add_order` | O(log N) | O(log N) search + O(N) shift | **O(1)** |
| `cancel_order` | O(log N) | O(log N) search + O(N) shift | **O(1) amort.** |
| `get_bbo` | **O(1)** | **O(1)** | **O(1)** |
| `get_bids(D)` | O(D) | O(D) | O(D) |
| `get_asks(D)` | O(D) | O(D) | O(D) |

N = number of active price levels (max 10 000)  
D = requested depth

---

## Correctness Results

The `--verify` mode runs two test stages:

**Stage 1 — Unit tests (10 groups, 56 assertions)**

| # | Test | Scenario |
|---|------|----------|
| 1 | Empty book | `get_bbo`, `get_bids`, `get_asks` on empty book — no crash, correct flags |
| 2 | Cancel unknown | `cancel_order(999)` never added → `false` |
| 3 | Cancel already-cancelled | cancel twice → first `true`, second `false` |
| 4 | Aggregate qty at same price | two orders at 99.50 → qty sums; cancelling one reduces aggregate |
| 5 | BBO updates — bid | cancel best bid → BBO falls to next level |
| 6 | BBO updates — ask | cancel best ask → BBO rises to next level |
| 7 | Price-band boundaries | orders at 50.00, 99.99, 100.01, 150.00 — all correctly handled |
| 8 | Insufficient depth | 3 levels exist, `get_bids(10)` returns exactly 3 |
| 9 | BBO with multi levels each side | correct best is selected from both sides |
| 10 | Cancel mid-book level | BBO unchanged after non-best level is removed |

**Stage 2 — Differential test: 200 000 ops, seed=42**  
All three implementations replay the same workload and are compared at every BBO query, every depth query, and every cancellation.

```
Running correctness test...

--- Unit tests ---
Unit tests: 56/56 passed

--- Differential test (200 000 ops, seed=42) ---

Implementation A: PASS
Implementation B: PASS
Implementation C: PASS

All implementations produce identical results.
Exit code: 0
```

`./bench --verify` exits with code **0** on success and **1** on any failure (CI-safe).

---

## Performance Results

> Measured on: Windows 11, g++ 16.2.0 (MSYS2 UCRT64), compiled with `-O2`  
> Each run = 1 000 000 ops (seed=42). 5 runs, median reported.

| Metric | Implementation A | Implementation B | Implementation C |
|--------|-----------------|-----------------|-----------------|
| Total time (ms) | **389.10** | **326.91** | **226.24** |
| ns / add | **333.75** | **252.59** | **156.70** |
| ns / cancel | **560.53** | **541.36** | **386.91** |
| ns / BBO | **34.09** | **34.56** | **31.71** |
| IPC | _(Linux perf required)_ | _(Linux perf required)_ | _(Linux perf required)_ |
| Branch misses (%) | _(Linux perf required)_ | _(Linux perf required)_ | _(Linux perf required)_ |
| L1-dcache-load-misses | _(Linux perf required)_ | _(Linux perf required)_ | _(Linux perf required)_ |
| Cache misses | _(Linux perf required)_ | _(Linux perf required)_ | _(Linux perf required)_ |
| Page faults | _(Linux perf required)_ | _(Linux perf required)_ | _(Linux perf required)_ |

---

## perf Analysis

> Paste your `perf stat` outputs here after running on Linux.

### Implementation A — perf stat
```
<paste perf stat output for A>
```

### Implementation B — perf stat
```
<paste perf stat output for B>
```

### Implementation C — perf stat
```
<paste perf stat output for C>
```

---

## Performance Report

### 11.1 Which implementation is fastest?

**Expected hypothesis** (to be confirmed by your measurements):

Implementation **C** (fixed-size array) should be fastest overall:
- `add_order`: C wins — O(1) direct array write vs. O(log N) tree walk (A) or binary-search + shift (B).
- `cancel_order`: C wins — O(1) slot decrement + O(1) cursor read vs. O(log N) (A) and O(N) shift (B).
- `get_bbo`: B and C tie at O(1); A is slower at O(log N).

**Why C is faster:** The flat array is ~80 KB and likely fits in L2 cache. Every operation is a single array index — no pointer chasing, no tree navigation, no element shifting. The CPU's hardware prefetcher can predict accesses efficiently.

---

### 11.2 Branch Prediction

**Expected:** Implementation A has the most branch misses because red-black tree traversal follows unpredictable pointer paths (left/right child, rotation decisions) whose outcomes depend on the data distribution. The tree's conditional branches are hard for the branch predictor to learn.

Implementation C has the fewest: the dominant path is `levels_[slot] += qty` with no branches.

---

### 11.3 Cache Behavior

| Impl | Memory access pattern |
|------|-----------------------|
| A (`std::map`) | Pointer-chasing through tree nodes scattered on the heap → many L1/L2 misses |
| B (sorted vector) | Contiguous array — great spatial locality for reads; insert/erase touches contiguous memory |
| C (fixed array) | Single large contiguous array; accessed by direct index → prefetcher-friendly, few cache misses |

Implementation A is expected to have the highest cache miss rate due to tree pointer chasing. Each node is a separate heap allocation at an unpredictable address.

Implementation C accesses the same ~80 KB array for every operation. After a warm-up period the entire array can reside in L2 or L3 cache.

---

### 11.4 Page Faults

Implementation A will likely have the most page faults because `std::map` allocates a separate heap node for each new price level. Over 1 000 000 operations touching up to 10 000 distinct levels, many allocator pages are touched for the first time.

Implementation C allocates its level array **once** at construction (80 KB). After that, no further page faults occur from level access. Orders are still heap-allocated (via `unordered_map`) but that is identical across all three.

---

### 11.5 Production Choice

**Choice: Implementation C**

In a real HFT system, I would deploy Implementation C for the following reasons:

| Factor | Reasoning |
|--------|-----------|
| **Latency** | O(1) add and cancel; O(1) BBO — lowest tail latency because there is no tree traversal or element shifting |
| **Predictability** | The fixed array means memory layout is known at startup; no GC pauses, no allocator jitter. The only variability is cursor repair, which is rare |
| **Memory usage** | The 80 KB level array is small and fits in cache. Unlike Implementation A, there is no per-level heap allocation |
| **Cache behaviour** | Contiguous array → hardware prefetcher works well; few cache misses after warm-up |
| **Complexity** | Slightly more complex than A to reason about (cursor repair), but the design is explainable and auditable |
| **Maintainability** | The code is ~120 lines, self-contained, and the cursor invariant is easy to test |

Implementation B has the advantage of O(1) BBO and cache-friendly vector reads, but its O(N) insert/erase is unacceptable for HFT where microseconds matter and cancel bursts are common.

Implementation A is the simplest to reason about but has the worst cache behaviour. It is a fine choice for a first prototype but not for production.

---

### 11.6 What Would You Try Next?

1. **Integer ticks everywhere** — Already done in this implementation, but can be propagated to the public API to avoid `tick_to_price()` calls on hot paths.
2. **Intrusive linked list for orders per level** — Instead of `unordered_map`, use a free-list pool of order nodes so cancellation touches only cache-warm memory.
3. **Object pool for `unordered_map` buckets** — The hash map bucket allocation can cause cache misses; a pre-allocated pool fixes this.
4. **Branch reduction in `get_bids`/`get_asks`** — Use SIMD to scan the 80 KB level array for non-zero entries faster than scalar loops.
5. **Sorted index set for Implementation C** — Add a small `std::set<int>` of active ticks to make `get_bids(D)` exactly O(D) without scanning gaps in the array.
6. **`__builtin_expect`** — Annotate the `best_bid_` repair branch as unlikely so the compiler lays out the hot path in a straight line.
7. **Cache-line alignment** — Align `levels_[]` to 64 bytes and pad the `best_bid_`/`best_ask_` cursors to their own cache line to avoid false sharing (useful if we add a reader thread later).
8. **Lock-free snapshot** — For the BBO, a `std::atomic<uint64_t>` packing bid+ask ticks allows a reader to get the BBO without any lock in a multi-threaded extension.
