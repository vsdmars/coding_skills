# High-Performance C++ Coding Skill & Engineering Reference
*Based on "The Art of Writing Efficient Programs" (2nd Edition) by Fedor G. Pikus*

This skill specification defines the principles, patterns, hardware realities, and code-level rules for writing high-performance, hardware-aware, and scalable C++ code. It serves as an actionable engineering reference, software architecture guide, and system prompt for C++ performance engineering.

---

## 1. Core Operating Directives & Philosophy

### 1.1 The Cardinal Rule of Performance
* **Never guess about performance.** Human intuition regarding code execution speed, compiler optimizations, and hardware behavior is frequently incorrect. Every performance claim, optimization attempt, or architectural change must be driven and validated by empirical measurement.
* **Measure, Understand, Improve, Verify, Repeat:** Follow an iterative performance engineering lifecycle. Establish baseline benchmarks before modifying code, profile to locate true bottlenecks, apply targeted transformations, and re-benchmark to verify speedup.

### 1.2 Performance vs. Efficiency
* **Efficiency** means using available hardware resources optimally without waste: keeping computational units busy, avoiding unnecessary memory allocations, eliminating redundant calculations, and preventing idle CPU cycles.
* **Performance** is the quantitative achievement of specific metrics tailored to application requirements:
  * **Throughput:** Maximizing operations or items processed per unit of time (critical for data processing, HPC, financial backends).
  * **Latency / Turnaround Time:** Minimizing time-to-answer for individual operations (critical for interactive systems).
  * **Tail Latency (95th / 99th Percentile):** Ensuring predictable response times without extreme spikes (critical for real-time systems).
  * **Power Consumption:** Minimizing energy draw per computation (critical for mobile and embedded devices).

### 1.3 Design for Performance
* Performance cannot be grafted onto a slow system as an afterthought; it must be an explicit design goal from inception.
* Avoid **premature optimization** (spending time tweaking non-critical code paths), but strictly avoid **intentional pessimization** (choosing data structures, interfaces, or ownership models that structurally impede performance).

---

## 2. Measurement, Profiling & Benchmarking Protocols

### 2.1 Micro-Benchmarking Standards (Google Benchmark)
* **Prevent Compiler Optimization of Benchmarks:** Compilers discard code whose outputs do not impact observable program behavior. Use benchmark harnesses to ensure computations run without being elided:
  * Use `benchmark::DoNotOptimize(value)` to force the compiler to treat `value` as read/written.
  * Use `benchmark::ClobberMemory()` to prevent the compiler from caching memory reads across loop iterations.
* **Avoid Volatile for Loop Variables:** Do not declare benchmark loop variables as `volatile`, as this prevents legitimate compiler optimizations (like register allocation) that would occur in production code.
* **Compiling Benchmarks:** Benchmark code fragments in separate compilation units with optimization enabled (`-O3`) to prevent the compiler from performing unrepresentative full-program optimizations based on constant benchmark inputs.

### 2.2 Profiling Workflows
* **Coarse-to-Fine Granularity:** Start with whole-program or module-level timing (using `std::chrono::high_resolution_clock`), proceed to CPU profilers (`perf`, Intel VTune) to identify hot functions and lines, and finish with micro-benchmarks to iterate on isolated critical loops.
* **Hardware Event Counters (`perf stat`):**
  * Measure key hardware metrics: `cycles`, `instructions` (evaluating Instructions Per Cycle - IPC), `branches`, `branch-misses`, `cache-references`, `cache-misses`.
  * Low IPC (< 1.0) indicates the CPU is stalled on memory accesses or branch mispredictions.
* **Call Graphs & Inlining Analysis:** Use stack sampling (`perf record -g` or VTune Hotspots with stack collection) to determine call chains. Be aware that compiler function inlining will scatter function assembly across caller sites.

---

## 3. CPU Architecture & Instruction-Level Optimization

### 3.1 Exploiting Instruction-Level Parallelism (ILP)
* Modern CPUs feature multiple execution units (ALUs, vector units, floating-point pipelines) capable of executing multiple instructions per cycle in parallel.
* Structure independent arithmetic operations together so the CPU out-of-order execution engine can schedule them simultaneously.

### 3.2 Managing Data Dependencies & Pipelining
* **Data Dependencies:** When an operation depends on the output of an immediately preceding operation, execution is serialized, preventing full ILP utilization.
* **Pipelining:** Interleave computations across independent iterations (e.g., processing multiple accumulators or loop streams) to keep hardware execution units saturated despite multi-cycle instruction latencies.

### 3.3 Branch Prediction & Branchless Computing
* **Branch Misprediction Penalty:** Unpredictable conditional branches (`if`/`else`, loops) break CPU pipelines, forcing pipeline flushes that cost 10–20+ CPU cycles.
* **Predictable vs. Unpredictable Branches:** Highly predictable branches (e.g., loop boundaries `i < N`, rare error checks) carry near-zero cost due to speculative execution. Random or 50/50 branches incur heavy penalties.
* **Branchless Selection:** Replace unpredictable branches with arithmetic, bitwise operations, or conditional moves (`CMOVE` / ternary operators `a ? b : c`):
  ```cpp
  // Branched (unpredictable condition b1[i]):
  if (b1[i]) { a1 += p1[i] - p2[i]; } else { a2 += p1[i] * p2[i]; }

  // Branchless (always evaluates arithmetic, uses boolean mask/indexing):
  unsigned long diff = p1[i] - p2[i];
  unsigned long prod = p1[i] * p2[i];
  a1 += diff * b1[i];
  a2 += prod * (!b1[i]);
  ```
* **Short-Circuiting Optimization:** Logical operators (`&&`, `||`) evaluate sequentially via conditional jumps. When evaluating non-side-effecting boolean conditions that are poorly predicted, replace short-circuiting logic with arithmetic or bitwise evaluation (`&`, `|`, `+`) to yield a single, predictable condition check.

---

## 4. Memory Architecture & Cache Optimization

### 4.1 The CPU–Memory Gap
* Accessing main memory (DRAM) requires tens to hundreds of nanoseconds (~200+ CPU cycles), whereas L1 cache access takes ~1 ns (4–5 cycles).
* High performance requires maximizing cache hit rates and tailoring data structures to the memory hierarchy (L1, L2, L3 caches).

### 4.2 Access Patterns: Sequential vs. Pointer Chasing
* **Sequential Access:** Optimal. Hardware prefetchers detect contiguous memory streams (e.g., `std::vector`, flat arrays) and fetch upcoming cache lines into L1/L2 automatically, hiding memory latency.
* **Random Access & Pointer Chasing:** Highly inefficient. Traversing linked lists, node-based trees, or graph pointers (`node = node->next`) creates serial dependency chains where each load address is unknown until the previous load finishes, completely defeating Memory-Level Parallelism (MLP).
* **Indexed Array Access:** When random access is required, prefer pre-calculated array indexing over pointer chasing. Indexed lookups allow out-of-order CPUs to issue multiple load requests in parallel.

### 4.3 Cache Line & Alignment Standards
* **Cache Line Size:** Main memory is transferred to caches in 64-byte lines.
* **Spatial Locality:** Pack related fields together in memory so that fetching a single 64-byte cache line brings in data that will all be used together.
* **TLB (Translation Lookaside Buffer):** Minimize stride sizes across large memory regions. Accessing memory across 4 KB page boundaries triggers TLB misses; process data in contiguous, page-friendly blocks or consider huge pages for massive datasets.

---

## 5. C++ Language Efficiency & Idioms

### 5.1 Eliminating Unnecessary Copying
* **Unnecessary Copying is C++'s #1 Inefficiency:**
  * Pass large, read-only parameters by `const&` or raw pointer/reference.
  * For small scalar types (integers, floats, pointers), pass by value—indirection via reference adds pointer-dereference overhead.
* **Move Semantics & RVO:**
  * Ensure custom classes implement efficient move constructors and move assignment operators.
  * Structure functions to leverage **Return Value Optimization (RVO)** and Named RVO (NRVO): return local variables by value directly, allowing the compiler to construct the object in-place in the caller's stack frame. Do not explicitly call `std::move()` on returned local variables, as this disables RVO.
* **Mark Non-Copyable Types:** Delete copy constructors and copy assignment operators for resource-heavy objects (`= delete`), forcing explicit ownership transfer via move semantics or explicit `clone()` methods.

### 5.2 Optimizing Memory Allocation
* **Heap Allocation Overhead:** Calls to `new`/`malloc` incur operating system allocation overhead, thread lock synchronization, and heap fragmentation.
* **Pre-Allocation & Buffer Reuse:**
  * Pre-allocate container capacity outside tight loops using `vector::reserve()`.
  * Reuse buffers across loop iterations rather than allocating and deleting memory inside loops.
* **Custom Allocators & Memory Pools:** Use arena allocators, block allocators, or memory-mapped files (`mmap`) for frequent, fixed-size allocations or temporary high-throughput processing.

---

## 6. Compiler Symbiosis & Undefined Behavior

### 6.1 Assisting Compiler Proofs
* Compilers optimize strictly under the **As-If Rule**: code transformations must not alter observable program behavior. If the compiler cannot prove an optimization is safe under all possible legal runtime states, it will skip it.
* **Eliminate Global State:** Avoid global variables and shared mutable state inside performance-critical loops, as the compiler must assume external function calls might modify them.
* **Basic Blocks & Inlining:** Keep functions short and modular. Small functions enable the compiler to inline them, forming larger basic blocks where optimizations (dead code elimination, constant propagation, SIMD vectorization) can be proven valid.

### 6.2 Aliasing Control & Compiler Hints
* **Pointer Aliasing:** The compiler must assume that distinct pointers of compatible types might point to overlapping memory locations, forcing redundant memory re-loads.
* **Strict Aliasing Rule:** Rely on strict aliasing rules or supply non-aliasing guarantees using platform restrict qualifiers (`__restrict` in GCC/Clang/MSVC) when pointers are guaranteed not to overlap.
* **Invariants & Assertions:** Provide compile-time invariants or explicit unreachable markers (e.g., `std::unreachable()` in C++23) to inform the compiler about restricted value ranges or impossible execution branches.

### 6.3 Exploiting Undefined Behavior (UB) Safely
* Compilers assume UB never occurs in valid code. This assumption allows significant optimizations:
  * Signed integer overflow is UB, enabling the compiler to prove loop counters like `for (int i = 0; i < N; ++i)` increase monotonically and optimize loop bounds.
  * Unreachable code paths based on narrow contract preconditions allow stripping dead branch checks.
* **Contract Specification:** Define explicit narrow contracts for low-level internal APIs (where the caller guarantees valid inputs) to strip runtime validation checks on hot paths. Validate contracts using Sanitizers (`UBSan`, `ASan`) during testing and debug builds.

---

## 7. High-Performance Concurrency & Parallelism

### 7.1 Realities of Multi-Core Execution
* Adding threads increases throughput only if execution units are idle and work can be partitioned without synchronization contention.
* Memory bandwidth is a shared, finite resource across cores; multi-threaded programs frequently become memory-bound before exhausting CPU execution units.
* **C++17 Parallel STL:** Use parallel execution policies (`std::execution::par`, `std::execution::par_unseq`) for data-parallel algorithm execution over large contiguous datasets.
* **C++20 Coroutines:** Utilize stackless coroutines (`co_await`, `co_yield`) for asynchronous control flow, lazy generators, and work-stealing schedulers without the overhead of OS thread context switching.

### 7.2 Synchronization Costs & Lock Selection
* **Lock-Based vs. Lock-Free:**
  * `std::mutex`: Incurs OS context switches when contended; suitable for long-duration critical sections.
  * **Spinlock:** Keeps the CPU core spinning in user space. Superior throughput under low-to-medium contention for extremely brief critical sections, but consumes power and degrades performance if preempted by the OS.
  * **Lock-Free (Atomic CAS Loops):** Uses CPU hardware atomic primitives (`compare_exchange_weak`). Highly performant when contention is low or dispersed, but can cause extreme cache line bouncing and coherency traffic under high contention.

### 7.3 Eliminating False Sharing
* **False Sharing:** Occurs when two separate threads modify logically distinct variables that happen to reside on the same 64-byte cache line. The cache coherence protocol invalidates the entire line back and forth across core caches ("cache line bouncing"), severely degrading performance.
* **Prevention:** Align per-thread data, accumulators, or atomic variables on 64-byte boundaries (or platform cache line size) using `alignas(64)` or `std::hardware_destructive_interference_size`.

### 7.4 C++ Memory Model & Memory Ordering
* Use explicit, relaxed memory orders when sequential consistency (`std::memory_order_seq_cst`) is not required:
  * `std::memory_order_relaxed`: Guarantees atomicity of the single operation without issuing synchronization barriers across other memory operations.
  * `std::memory_order_release` / `std::memory_order_acquire`: Establishes a happens-before relationship between publishing threads and reading threads without the global cost of full memory fences.

### 7.5 NUMA Awareness (Non-Uniform Memory Access)
* On multi-socket or multi-domain hardware, memory access latency to local socket RAM is significantly lower than remote socket RAM.
* Bind threads to specific CPU cores/sockets and allocate memory locally on the same NUMA node. Minimize cross-node data sharing to prevent interconnect saturation and TLB shootdown stalls.

---

## 8. Concurrent Data Structures & Patterns

### 8.1 Transactional & Minimal Interfaces
* Concurrent data structures must offer minimal, transactional interfaces. Avoid check-then-act patterns like `if (!q.empty()) { val = q.pop(); }` which create inherent data races between operations. Instead, combine operations into single atomic transactions (e.g., `bool q.try_pop(val)`).

### 8.2 Publishing Protocol
* Safely reveal newly constructed objects to other threads in two distinct steps:
  1. **Private Preparation:** Construct and fully initialize the data structure on a thread-local stack or private allocation without exposure.
  2. **Atomic Publication:** Publish the fully initialized pointer to shared visibility using a single store with `memory_order_release`. Readers consume the pointer using `memory_order_acquire`.

### 8.3 Safe Memory Reclamation & The ABA Problem
* **The ABA Problem:** In lock-free node-based structures, a CAS operation checking pointer address `A` can succeed even if node `A` was deleted, freed, and re-allocated at the same address, corrupting structural links.
* **Reclamation Strategies:**
  * **Arena / Append-Only Allocation:** Defer all physical deallocations until the entire container or processing phase finishes.
  * **Hazard Pointers & Epoch-Based Reclamation:** Track active reader references to guarantee a node is never freed while any thread holds an active reference to it.
  * **Atomic Smart Pointers (`std::atomic<std::shared_ptr<T>>`):** Provide safe reference-counted reclamation at the cost of atomic increment/decrement overhead on every read.

---

## 9. High-Performance API & Architectural Principles

### 9.1 The Minimum Information Principle (For Component Implementations)
* A component fulfilling a request should disclose as little internal operational detail as possible in its contract.
* Returning boolean status (e.g., `q.empty()`) rather than exact size (`q.size()`) leaves the implementer free to optimize internal state representation without breaking caller expectations.

### 9.2 The Maximum Information Principle (For Component Callers)
* An API should allow the caller to communicate full operational intent to the component.
* Provide bulk, range, or specialized member functions (e.g., `vector::insert(range)`, custom `find()`) so the internal implementation can perform batch processing, vectorization, or temporary invariant violations during processing.

### 9.3 Data Flow Design
* Classify all system data flows into three distinct concurrency buckets:
  1. **Single-Threaded / Exclusive:** Data owned by a single thread; zero locking overhead.
  2. **Read-Only / Immutable:** Shared across threads without mutation; zero locking overhead.
  3. **Shared Mutable:** Shared data modified concurrently; strictly isolate and minimize using transactional building blocks or local thread accumulation followed by batch reconciliation.

---

## 10. Quick-Reference Performance Engineering Checklist

| Category | Optimization Rule | Verification / Method |
| :--- | :--- | :--- |
| **Benchmarking** | Establish baseline before optimizing; use `DoNotOptimize()` and `ClobberMemory()`. | Google Benchmark / `-O3` separate translation units |
| **Profiling** | Identify bottlenecks via hardware performance counters (`IPC`, branch-misses, cache-misses). | Linux `perf stat`, `perf record`, Intel VTune |
| **CPU Execution** | Interleave arithmetic streams to exploit ILP and pipelining; eliminate unpredictable branches. | Branchless selection, CMOVE, bitwise masks |
| **Memory Access** | Prefer contiguous arrays (`std::vector`) over pointer chasing (`std::list`, trees). | Sequential prefetching, indexed arrays |
| **Alignment** | Align per-thread accumulators and atomics to 64 bytes to prevent false sharing. | `alignas(64)` / `std::hardware_destructive_interference_size` |
| **Copy Prevention** | Pass large objects by `const&`; leverage RVO and move semantics; mark resource objects non-copyable. | Delete copy ops (`= delete`), NRVO |
| **Allocations** | Pre-allocate capacity outside tight loops; reuse buffers; use arena allocators. | `vector::reserve()`, memory pools |
| **Compiler Symbiosis** | Keep functions small for inlining; avoid global variables in loops; use alias qualifiers. | `__restrict`, `std::unreachable()`, optimization remarks |
| **Concurrency** | Use `relaxed` or `acquire-release` memory ordering where `seq_cst` is not required. | `std::memory_order_relaxed`, `acquire`/`release` |
| **API Design** | Design minimal, transactional thread-safe APIs; enforce Minimum & Maximum Information Principles. | `try_pop()`, range operations |
