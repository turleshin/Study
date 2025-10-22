# C++ Multi-Threading Basics

## 1. Benefits and Risks of Multi-Threading
Benefits:
- Parallelism: Utilize multiple CPU cores.
- Responsiveness: Keep UI/reactive parts alive while background work runs.
- Throughput: Pipeline tasks (producer/consumer).
- Resource sharing: Threads share address space (no IPC serialization).
Risks:
- Race conditions and data corruption.
- Deadlocks / livelocks / starvation.
- Increased complexity and debugging difficulty.
- False sharing and cache thrashing.
- Oversubscription causing context switch overhead.

## 2. Race Condition and Resolution
Race condition: Outcome depends on non-deterministic timing of reads/writes to shared data.
Example:
```cpp
int counter = 0;
void work() { for(int i=0;i<100000;i++) ++counter; }
```
Two threads produce incorrect total.
Resolutions:
- Mutual exclusion (std::mutex).
- Atomic operations (std::atomic<int> counter; ++counter;).
- Higher-level patterns (message passing, task queues).

## 3. Using Mutex in C++
Mutex protects a critical section so only one thread enters at a time.
```cpp
#include <mutex>
std::mutex m;
int sharedValue = 0;
void increment() {
    std::lock_guard<std::mutex> lock(m); // RAII
    ++sharedValue;
}
```
Variants: std::unique_lock (manual lock/unlock), std::recursive_mutex (avoid unless required), std::timed_mutex.

## 4. Deadlock Concept and Avoidance
Deadlock: Set of threads each waiting for resources held by others, none proceed.
Common causes: Circular wait, holding locks while acquiring additional locks.
Avoidance:
- Impose global lock ordering.
- Use std::scoped_lock(m1, m2) for multiple locks.
- Minimize lock scope and never call user callbacks while holding locks.
- Prefer lock-free or single ownership designs.

## 5. Creating and Managing Threads in C++
Use std::thread; join or detach before destruction.
```cpp
#include <thread>
#include <vector>
#include <iostream>

void task(int id) { std::cout << "Task " << id << "\n"; }

int main() {
    std::vector<std::thread> threads;
    for(int i=0;i<4;i++) threads.emplace_back(task, i);
    for(auto& t: threads) t.join(); // ensure completion
}
```
Use std::jthread (C++20) for automatic join on destruction and cooperative cancellation via stop_token.
```cpp
#include <thread>
void work(std::stop_token st) { while(!st.stop_requested()) { /*...*/ } }
int main(){ std::jthread t(work); /* request stop if needed */ }
```

## 6. Thread-Safety Concept
Thread-safe code functions correctly when accessed concurrently from multiple threads.
Requires:
- Proper synchronization of shared mutable state.
- Avoiding data races (defined by the standard: unsynchronized conflicting accesses).
- Using atomic operations or locks around shared writes/reads.
Thread-safety levels: not thread-safe, thread-compatible (each thread owns instance), fully thread-safe (shared instance).

## 7. Using Condition Variable in C++
Condition variable blocks threads until a predicate becomes true.
```cpp
#include <mutex>
#include <condition_variable>
#include <queue>

std::mutex m;
std::condition_variable cv;
std::queue<int> q;
bool finished = false;

void producer() {
    for(int i=0;i<5;i++) {
        {
            std::lock_guard<std::mutex> lock(m);
            q.push(i);
        }
        cv.notify_one();
    }
    {
        std::lock_guard<std::mutex> lock(m);
        finished = true;
    }
    cv.notify_all();
}

void consumer() {
    while(true) {
        std::unique_lock<std::mutex> lock(m);
        cv.wait(lock, []{ return !q.empty() || finished; });
        if(!q.empty()) { int v = q.front(); q.pop(); /* process v */ }
        else if(finished) break;
    }
}
```
Always wait with predicate to handle spurious wakeups.

## 8. Multi-Threading vs Multi-Processing
| Aspect | Threads | Processes |
|--------|---------|-----------|
| Memory Space | Shared | Separate |
| IPC Overhead | Low | Higher |
| Isolation | Weak | Strong |
| Crash Impact | May crash entire process | Usually isolated |
| Security | Less isolation | More isolation |
| Creation Cost | Lower | Higher |
| Use Cases | Parallel tasks sharing data | Heavy isolation, scaling across cores with strong boundaries |

## 9. False Sharing and Avoidance
False sharing: Performance degradation when threads modify distinct variables that reside on the same CPU cache line. Although the variables are logically independent, they share a physical line; each write causes cache coherence traffic (invalidations) forcing other cores to reload the line.

### 9.1 Hardware Background
- Cache line: Smallest transfer unit between cache levels (typically 64 bytes on x86_64, 64/128 on some ARM). All bytes in a line move together.
- Coherence protocol (MESI/MOESI): A core writing to a line transitions its state to Modified/Owned and invalidates copies in other cores.
- Independent counters placed adjacently can "ping-pong" a line between cores on every increment, drastically increasing latency.

### 9.2 Symptoms
- Code scales poorly as thread count increases (e.g., throughput flattens or declines).
- High CPU usage but low useful progress.
- Profiling shows time in atomic increments or lock regions with minimal contention logically.
- Performance improves when adding artificial spacing or grouping by thread.

### 9.3 Minimal Demonstration
```cpp
#include <atomic>
#include <chrono>
#include <iostream>
#include <thread>

struct BadCounters { // adjacent atomics likely share a line
    std::atomic<long> a{0};
    std::atomic<long> b{0};
};

struct GoodCounters { // separate lines via alignment & padding
    alignas(64) std::atomic<long> a{0};
    alignas(64) std::atomic<long> b{0};
};

constexpr long ITERS = 10'000'000;

template <typename C> void run(C& c) {
    auto start = std::chrono::high_resolution_clock::now();
    std::thread t1([&]{ for(long i=0;i<ITERS;i++) c.a.fetch_add(1, std::memory_order_relaxed); });
    std::thread t2([&]{ for(long i=0;i<ITERS;i++) c.b.fetch_add(1, std::memory_order_relaxed); });
    t1.join(); t2.join();
    auto end = std::chrono::high_resolution_clock::now();
    std::cout << "Elapsed ms: " << std::chrono::duration_cast<std::chrono::milliseconds>(end-start).count() << "\n";
}

int main() {
    BadCounters bad; run(bad);
    GoodCounters good; run(good);
}
```
Typical output: GoodCounters significantly faster.

### 9.4 Common Sources
- Arrays of per-thread counters (counter[thread_id]) tightly packed.
- Structs with multiple frequently updated fields.
- Vector<bool>/bitsets where bits from different threads share same line.
- Shared flags in lock-free algorithms placed adjacently.

### 9.5 Mitigation Techniques
1. Padding / Alignment:
```cpp
struct Counter { std::atomic<long> v; char pad[64 - sizeof(std::atomic<long>)]; };
```
Ensure size rounds to line boundary.
2. alignas:
```cpp
struct Counter { alignas(64) std::atomic<long> v; };
```
Group each Counter separately (beware of packing in arrays; the compiler may still place them contiguously—add padding or use std::array<Counter, N> where Counter size >= 64).
3. Array Striding:
Allocate N * stride elements and index by thread_id * stride where stride >= cache line size.
4. Per-Thread Local Aggregation:
Each thread accumulates locally then merges (reduces false sharing on hot updates).
5. Data-Oriented Partitioning:
Place write-heavy data in dedicated structures separate from read-mostly fields.
6. Avoid Atomic on Shared Line Unless Needed:
Sometimes a regular non-atomic thread-local variable suffices until merge.

### 9.6 Verification Strategies
- Inspect addresses: Print & reinterpret_cast<uintptr_t>(&obj.field) differences; ensure >= 64.
- Use performance counters: Tools like perf (Linux) with events (cache-misses, LLC-store-misses) or VTune's false sharing analysis.
- Micro-benchmark variants (with/without padding) to quantify impact.

### 9.7 Over-Padding Trade-Off
Excess padding increases memory footprint and may reduce cache utilization if too many large padded objects thrash higher level caches. Strike balance: only pad hot independent write locations.

### 9.8 High-Level Containers
Boost.Interprocess / concurrent containers may internally mitigate false sharing, but verify for custom patterns.

### 9.9 C++20 std::hardware_destructive_interference_size
Use standard constants for portability:
```cpp
#include <new>
constexpr std::size_t CLS = std::hardware_destructive_interference_size; // typical 64
struct Counter { alignas(CLS) std::atomic<long> v; };
```
Also: std::hardware_constructive_interference_size for grouping fields that benefit from sharing.

### 9.10 Summary Checklist
- Are two hot write locations within same 64B? Separate them.
- Can local accumulation reduce shared writes? Implement it.
- Use alignas(CLS) + verify object layout.
- Profile to confirm improvement; avoid premature padding everywhere.

## 10. Using std::atomic in C++
std::atomic provides lock-free (where possible) operations with defined memory ordering.
```cpp
#include <atomic>
std::atomic<int> counter{0};
void work() {
    for(int i=0;i<1000;i++) counter.fetch_add(1, std::memory_order_relaxed);
}
```
Ordering basics:
- memory_order_relaxed: no ordering guarantees beyond atomicity.
- memory_order_acquire: subsequent reads/writes cannot move before.
- memory_order_release: prior reads/writes cannot move after.
- memory_order_acq_rel: both acquire and release.
- memory_order_seq_cst: strongest total ordering.
Common pattern:
```cpp
// Publish data
payload = value; // normal write
flag.store(true, std::memory_order_release);
// Consume data
if(flag.load(std::memory_order_acquire)) { use(payload); }
```
Check lock-free property: counter.is_lock_free().

---
Next: Advanced topics (thread pools, work stealing, lock-free structures)
