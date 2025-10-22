# Shared Memory Basics

## 1. Concept of Shared Memory
Shared memory is a block of RAM that multiple running programs (processes) can access directly. Instead of sending copies of data (like with sockets or pipes), processes read and write to the same memory region, making data exchange very fast.

### Why It Exists
Inter‑process communication (IPC) methods like pipes or message queues move data from one process space to another. Shared memory avoids extra copying by letting processes map the same physical pages into their virtual address spaces.

### Core Idea
One process creates (or opens) a shared segment. Other processes attach (map) that segment into their own address space. All can then use normal pointer or array operations on it.

```
+------------------+        +------------------+
|  Process A       |        |  Process B       |
|  Virtual Memory  |        |  Virtual Memory  |
|  [shared region] |<------>|  [shared region] |
+------------------+        +------------------+
            \              /
             \            /
              +----------+
              | Physical |
              |  Memory  |
              +----------+
```

### Key Properties
- Fast: No kernel data copying after mapping.
- Explicit: You decide layout (structs, arrays, rings, etc.).
- Unstructured: OS does not enforce format or safety.
- Requires synchronization: Multiple writers need coordination (mutex, semaphore, atomic operations) to avoid race conditions.

### What Shared Memory Is NOT
- It is not automatically thread‑safe.
- It is not persistent storage (unless backed by a file and flushed).
- It is not a communication protocol by itself—you must define structure and synchronization.

### Typical Implementations
- POSIX: shm_open + ftruncate + mmap (e.g. /dev/shm on Linux).
- System V: shmget + shmat + shmdt.
- File-backed: open a file + mmap (allows persistence/sharing via file descriptor inheritance or name).
- Windows: CreateFileMapping + MapViewOfFile ("named shared memory").

### When To Use
- High-frequency data exchange (telemetry, sensor frames).
- Large data blocks (images, matrices) where copying is expensive.
- Producer/consumer designs with low latency constraints.

### When To Avoid
- Small, infrequent messages (simpler IPC may suffice).
- Distributed systems across different machines (shared memory is local only).
- Cases requiring security isolation (shared memory widens attack surface unless carefully controlled).

### Synchronization Reminder
Even though memory is shared, CPU caches can reorder/merge operations. Use proper primitives:
- Mutex/semaphore for mutual exclusion.
- Condition variable or event for notification.
- Atomic variables + memory barriers for lock-free structures.

### Simple Example Layout (Conceptual)
```
struct SharedBlock {
    std::atomic<uint32_t> writeIndex;
    std::atomic<uint32_t> readIndex;
    char buffer[BUFFER_SIZE];
};
```
Processes agree on this layout and update indices with atomics, protecting wrap-around.

### Advantages vs Message Passing
| Aspect        | Shared Memory              | Pipes/Queues            |
|---------------|----------------------------|-------------------------|
| Latency       | Very low (no copy after map) | Higher (kernel mediation) |
| Data Size     | Efficient for large blocks  | Large blocks costly      |
| Complexity    | Must design protocol        | Provided by API          |
| Safety        | Easy to corrupt data        | API enforces boundaries  |

### Mental Model
Think of shared memory as a blank whiteboard both processes can write on. Fast—but requires rules to avoid scribbling over each other.

### Minimal C++ Examples
Below are stripped-down examples showing one process writing a struct and another reading it. Omitted: full error checking and synchronization (needed for production).

#### POSIX (Linux/macOS) Producer
```cpp
// producer.cpp
#include <fcntl.h>      // O_CREAT, O_RDWR
#include <sys/mman.h>   // shm_open, mmap
#include <sys/stat.h>   // S_IRUSR, S_IWUSR
#include <unistd.h>     // ftruncate, close
#include <cstring>
#include <iostream>

struct SharedData { int value; char msg[32]; };

int main() {
    const char* name = "/demo_shm"; // POSIX requires leading slash
    int fd = shm_open(name, O_CREAT | O_RDWR, 0666);
    if (fd == -1) { perror("shm_open"); return 1; }
    if (ftruncate(fd, sizeof(SharedData)) == -1) { perror("ftruncate"); return 1; }

    void* ptr = mmap(nullptr, sizeof(SharedData), PROT_READ | PROT_WRITE, MAP_SHARED, fd, 0);
    if (ptr == MAP_FAILED) { perror("mmap"); return 1; }

    auto* data = static_cast<SharedData*>(ptr);
    data->value = 42;
    std::strncpy(data->msg, "Hello from producer", sizeof(data->msg));
    std::cout << "Wrote value=42" << std::endl;

    // Keep process alive to allow reader to attach (demo purpose)
    std::cout << "Press Enter to exit..."; std::cin.get();

    munmap(ptr, sizeof(SharedData));
    close(fd);
    // Optionally shm_unlink(name); // remove segment
    return 0;
}
```

#### POSIX Consumer
```cpp
// consumer.cpp
#include <fcntl.h>
#include <sys/mman.h>
#include <sys/stat.h>
#include <unistd.h>
#include <iostream>
#include <cstring>

struct SharedData { int value; char msg[32]; };

int main() {
    const char* name = "/demo_shm";
    int fd = shm_open(name, O_RDWR, 0666);
    if (fd == -1) { perror("shm_open"); return 1; }

    void* ptr = mmap(nullptr, sizeof(SharedData), PROT_READ | PROT_WRITE, MAP_SHARED, fd, 0);
    if (ptr == MAP_FAILED) { perror("mmap"); return 1; }

    auto* data = static_cast<SharedData*>(ptr);
    std::cout << "Read value=" << data->value << " msg=" << data->msg << std::endl;

    munmap(ptr, sizeof(SharedData));
    close(fd);
    return 0;
}
```
Build (example):
```
g++ -std=c++17 producer.cpp -o producer
g++ -std=c++17 consumer.cpp -o consumer
```
Run in two terminals: first ./producer then ./consumer.

#### Windows Producer
```cpp
// producer.cpp
#include <windows.h>
#include <iostream>
#include <cstring>

struct SharedData { int value; char msg[32]; };

int main() {
    const wchar_t* name = L"Local\\DemoSharedMemory"; // Local or Global scope
    HANDLE hMap = CreateFileMappingW(INVALID_HANDLE_VALUE, nullptr, PAGE_READWRITE, 0, sizeof(SharedData), name);
    if (!hMap) { std::cerr << "CreateFileMapping failed\n"; return 1; }

    void* ptr = MapViewOfFile(hMap, FILE_MAP_WRITE | FILE_MAP_READ, 0, 0, sizeof(SharedData));
    if (!ptr) { std::cerr << "MapViewOfFile failed\n"; CloseHandle(hMap); return 1; }

    auto* data = static_cast<SharedData*>(ptr);
    data->value = 99;
    std::strncpy(data->msg, "Hello Windows", sizeof(data->msg));
    std::cout << "Wrote value=99" << std::endl;

    std::cout << "Press Enter to exit..."; std::cin.get();
    UnmapViewOfFile(ptr);
    CloseHandle(hMap);
    return 0;
}
```

#### Windows Consumer
```cpp
// consumer.cpp
#include <windows.h>
#include <iostream>

struct SharedData { int value; char msg[32]; };

int main() {
    const wchar_t* name = L"Local\\DemoSharedMemory";
    HANDLE hMap = OpenFileMappingW(FILE_MAP_READ | FILE_MAP_WRITE, FALSE, name);
    if (!hMap) { std::cerr << "OpenFileMapping failed\n"; return 1; }

    void* ptr = MapViewOfFile(hMap, FILE_MAP_READ | FILE_MAP_WRITE, 0, 0, sizeof(SharedData));
    if (!ptr) { std::cerr << "MapViewOfFile failed\n"; CloseHandle(hMap); return 1; }

    auto* data = static_cast<SharedData*>(ptr);
    std::cout << "Read value=" << data->value << " msg=" << data->msg << std::endl;

    UnmapViewOfFile(ptr);
    CloseHandle(hMap);
    return 0;
}
```
Build (example):
```
cl /std:c++17 producer.cpp
cl /std:c++17 consumer.cpp
```
Run producer.exe then consumer.exe in separate terminals.

Notes:
- For real use add synchronization (e.g., a named semaphore or an atomic flag) before reading uninitialized data.
- Avoid busy waiting; use events or semaphores for notification.
- Consider alignment and padding for larger structures.

---
Next: 2. Creating a shared memory segment (POSIX & Windows examples)

## 2. Creating and Using Shared Memory in C++
This section shows concrete steps to create, attach, use, and clean up shared memory with minimal synchronization.

### 2.1 General Steps
1. Define a shared data structure (ensure standard layout, watch padding).
2. Create or open a named shared memory object (platform API).
3. Set its size (POSIX: ftruncate; Windows: size in CreateFileMapping).
4. Map it into the process address space (mmap / MapViewOfFile).
5. Optionally create/open synchronization primitives (semaphore, mutex, event).
6. Use the memory (read/write) under synchronization.
7. Unmap / close handles.
8. Cleanup creator-only resources (shm_unlink, CloseHandle, etc.).

### 2.2 Shared Structure Example
```cpp
struct SharedPacket {
    std::atomic<uint32_t> sequence; // updated by producer
    char message[64];               // payload
};
```
Initialize atomics only once by the creator; use memory barriers implicitly provided by atomic ops.

### 2.3 POSIX Implementation (Linux/macOS)
APIs: shm_open, ftruncate, mmap, munmap, close, shm_unlink, sem_open, sem_post, sem_wait, sem_close, sem_unlink.

Names:
- Shared memory name must start with '/': e.g. "/demo_packet".
- Semaphore name must start with '/': e.g. "/demo_sem".

#### Producer (Writer)
```cpp
// posix_producer.cpp
#include <fcntl.h>
#include <sys/mman.h>
#include <sys/stat.h>
#include <unistd.h>
#include <semaphore.h>
#include <cstring>
#include <iostream>
#include <atomic>

struct SharedPacket { std::atomic<uint32_t> sequence; char message[64]; };

int main() {
    const char* shmName = "/demo_packet";
    const char* semName = "/demo_sem";
    int fd = shm_open(shmName, O_CREAT | O_RDWR, 0666);
    if (fd == -1) { perror("shm_open"); return 1; }
    if (ftruncate(fd, sizeof(SharedPacket)) == -1) { perror("ftruncate"); return 1; }
    void* ptr = mmap(nullptr, sizeof(SharedPacket), PROT_READ | PROT_WRITE, MAP_SHARED, fd, 0);
    if (ptr == MAP_FAILED) { perror("mmap"); return 1; }

    auto* data = static_cast<SharedPacket*>(ptr);
    // Initialize once
    data->sequence.store(0, std::memory_order_relaxed);

    sem_t* sem = sem_open(semName, O_CREAT, 0666, 0); // initial value 0
    if (sem == SEM_FAILED) { perror("sem_open"); return 1; }

    for (uint32_t i = 1; i <= 5; ++i) {
        data->sequence.store(i, std::memory_order_release);
        std::snprintf(data->message, sizeof(data->message), "Packet #%u", i);
        sem_post(sem); // signal consumer new data available
        std::cout << "Produced: " << data->message << std::endl;
        sleep(1); // simulate work
    }

    // Cleanup (keep shared objects alive for consumer until after run)
    munmap(ptr, sizeof(SharedPacket));
    close(fd);
    sem_close(sem);
    // Optionally remove names after last process:
    // shm_unlink(shmName); sem_unlink(semName);
    return 0;
}
```

#### Consumer (Reader)
```cpp
// posix_consumer.cpp
#include <fcntl.h>
#include <sys/mman.h>
#include <sys/stat.h>
#include <unistd.h>
#include <semaphore.h>
#include <iostream>
#include <atomic>

struct SharedPacket { std::atomic<uint32_t> sequence; char message[64]; };

int main() {
    const char* shmName = "/demo_packet";
    const char* semName = "/demo_sem";
    int fd = shm_open(shmName, O_RDWR, 0666);
    if (fd == -1) { perror("shm_open"); return 1; }
    void* ptr = mmap(nullptr, sizeof(SharedPacket), PROT_READ | PROT_WRITE, MAP_SHARED, fd, 0);
    if (ptr == MAP_FAILED) { perror("mmap"); return 1; }

    auto* data = static_cast<SharedPacket*>(ptr);
    sem_t* sem = sem_open(semName, 0); // open existing
    if (sem == SEM_FAILED) { perror("sem_open"); return 1; }

    for (int i = 0; i < 5; ++i) {
        sem_wait(sem); // wait for producer
        uint32_t seq = data->sequence.load(std::memory_order_acquire);
        std::cout << "Consumed seq=" << seq << " msg=" << data->message << std::endl;
    }

    munmap(ptr, sizeof(SharedPacket));
    close(fd);
    sem_close(sem);
    // After both exit, one process can call shm_unlink & sem_unlink.
    return 0;
}
```
Build:
```
g++ -std=c++17 posix_producer.cpp -o posix_producer -pthread
g++ -std=c++17 posix_consumer.cpp -o posix_consumer -pthread
```
Run in two terminals: start consumer, then producer (or producer first; semaphore ensures proper blocking).

### 2.4 Windows Implementation
APIs: CreateFileMapping / OpenFileMapping, MapViewOfFile / UnmapViewOfFile, CreateMutex / OpenMutex, WaitForSingleObject, ReleaseMutex, CloseHandle.

Object naming: Use a consistent wide string (e.g. L"Local\\DemoPacketMap" and L"Local\\DemoPacketMutex"). Local confines to current session.

#### Producer (Writer)
```cpp
// win_producer.cpp
#include <windows.h>
#include <iostream>
#include <atomic>
#include <cstdio>

struct SharedPacket { std::atomic<uint32_t> sequence; char message[64]; };

int main() {
    const wchar_t* mapName = L"Local\\DemoPacketMap";
    const wchar_t* mutexName = L"Local\\DemoPacketMutex";

    HANDLE hMap = CreateFileMappingW(INVALID_HANDLE_VALUE, nullptr, PAGE_READWRITE, 0, sizeof(SharedPacket), mapName);
    if (!hMap) { std::cerr << "CreateFileMapping failed\n"; return 1; }
    auto* data = static_cast<SharedPacket*>(MapViewOfFile(hMap, FILE_MAP_WRITE | FILE_MAP_READ, 0, 0, sizeof(SharedPacket)));
    if (!data) { std::cerr << "MapViewOfFile failed\n"; CloseHandle(hMap); return 1; }

    HANDLE hMutex = CreateMutexW(nullptr, FALSE, mutexName);
    if (!hMutex) { std::cerr << "CreateMutex failed\n"; return 1; }

    data->sequence.store(0, std::memory_order_relaxed);

    for (uint32_t i = 1; i <= 5; ++i) {
        WaitForSingleObject(hMutex, INFINITE);
        data->sequence.store(i, std::memory_order_release);
        std::snprintf(data->message, sizeof(data->message), "Packet %u", i);
        ReleaseMutex(hMutex);
        std::cout << "Produced Packet " << i << std::endl;
        Sleep(1000);
    }

    UnmapViewOfFile(data);
    CloseHandle(hMutex);
    CloseHandle(hMap);
    return 0;
}
```

#### Consumer (Reader)
```cpp
// win_consumer.cpp
#include <windows.h>
#include <iostream>
#include <atomic>

struct SharedPacket { std::atomic<uint32_t> sequence; char message[64]; };

int main() {
    const wchar_t* mapName = L"Local\\DemoPacketMap";
    const wchar_t* mutexName = L"Local\\DemoPacketMutex";

    HANDLE hMap = OpenFileMappingW(FILE_MAP_READ | FILE_MAP_WRITE, FALSE, mapName);
    if (!hMap) { std::cerr << "OpenFileMapping failed\n"; return 1; }
    auto* data = static_cast<SharedPacket*>(MapViewOfFile(hMap, FILE_MAP_READ | FILE_MAP_WRITE, 0, 0, sizeof(SharedPacket)));
    if (!data) { std::cerr << "MapViewOfFile failed\n"; CloseHandle(hMap); return 1; }

    HANDLE hMutex = OpenMutexW(SYNCHRONIZE, FALSE, mutexName);
    if (!hMutex) { std::cerr << "OpenMutex failed\n"; return 1; }

    uint32_t last = 0;
    while (last < 5) {
        WaitForSingleObject(hMutex, INFINITE);
        uint32_t seq = data->sequence.load(std::memory_order_acquire);
        if (seq > last) {
            std::cout << "Consumed seq=" << seq << " msg=" << data->message << std::endl;
            last = seq;
        }
        ReleaseMutex(hMutex);
        Sleep(200); // polling delay
    }

    UnmapViewOfFile(data);
    CloseHandle(hMutex);
    CloseHandle(hMap);
    return 0;
}
```
Compile (Developer Command Prompt):
```
cl /std:c++17 win_producer.cpp
cl /std:c++17 win_consumer.cpp
```
Run producer then consumer (or vice versa; consumer waits by polling until sequence advances).

### 2.5 Cleanup & Lifetime
- Only unlink/delete shared objects after all processes detach.
- POSIX: shm_unlink removes the name; existing mappings persist until unmapped.
- Windows: last handle close frees unnamed mapping; named objects go away when no handles remain.

### 2.6 Common Pitfalls
| Issue | Cause | Mitigation |
|-------|-------|------------|
| Stale data | Missing synchronization | Use mutex/semaphore or atomics with ordering |
| Size mismatch | Different struct definitions | Centralize struct in shared header |
| Leaked objects | Forgot shm_unlink / handles | Perform cleanup in shutdown path |
| Partial write | Writer interrupted mid-update | Use atomic sequence + copy then publish |
| Cache coherence | Relaxed atomics misuse | Use acquire/release or stronger fences |

---
Next: 3. Designing data layout & synchronization patterns

## 3. Designing Data Layout & Synchronization Patterns
Design impacts performance and correctness. Keep structures simple, cache-friendly, and versioned.

### 3.1 Principles
- Fixed-size structs reduce fragmentation.
- Use POD / standard layout types for binary compatibility.
- Add a version/size field for forward compatibility.
- Align to cache lines (typically 64 bytes) to avoid false sharing.
- Separate frequently updated atomics from large payloads.

### 3.2 Common Layout Patterns
1. Header + Ring Buffer
```
struct Header { std::atomic<uint32_t> writeIdx; std::atomic<uint32_t> readIdx; };
struct SharedRegion { Header hdr; Item items[CAPACITY]; };
```
2. Header + Slots + Free List (for variable-sized objects)
3. Multiple producer queues -> single dispatcher region.

### 3.3 Publication Pattern (Single Producer)
- Fill payload first.
- Use memory_order_release when publishing index/state.
- Consumer reads index with memory_order_acquire then reads payload.

### 3.4 Avoiding False Sharing
Pad hot atomics:
```
struct AlignedAtomic { std::atomic<uint64_t> value; char pad[64 - sizeof(std::atomic<uint64_t>)]; };
```

### 3.5 Versioning
```
struct SharedHeader {
    uint32_t abiVersion; // increment when layout changes
    uint32_t totalSize;  // sizeof(SharedSegment)
};
```
Consumers validate before use.

---
## 4. Security Issues Related to Shared Memory
Shared memory bypasses structured IPC checks; improper use can expose data or allow tampering.

### 4.1 Risks
- Unauthorized Access: Weak permissions on POSIX shm objects (mode 0666) or global Windows objects.
- Name Collisions / Spoofing: Attacker creates object first with permissive contents.
- Data Poisoning: Malicious process writes invalid state leading to crashes or logic errors.
- Information Disclosure: Sensitive data remains in pages after detach.
- Race Conditions: TOCTOU bugs during initialization.
- Denial of Service: Filling buffers, holding locks, or leaving semaphore counts inconsistent.

### 4.2 Mitigations
- Restrict permissions (e.g., 0640) and proper ACLs on Windows.
- Use unpredictable names or incorporate UUIDs.
- Validate headers (magic, version, size, checksum) before trusting payload.
- Zero memory on creation and optionally on teardown (explicit bzero / SecureZeroMemory).
- Employ watchdog / timeouts for lock acquisitions.
- Use separate integrity fields (CRC32 / hash) for critical blocks.
- Drop privileges after creation if running as elevated user.

### 4.3 Least Privilege
Only processes needing access should have handle; close unused descriptors immediately.

---
## 5. Difference Between Shared Memory and Memory-Mapped Files
Memory-mapped files (mmap of regular file / CreateFileMapping with file handle) are a superset enabling persistence. Shared memory often refers to anonymous or named, non-persistent regions.

| Aspect | Shared Memory Object | Memory-Mapped File |
|--------|----------------------|--------------------|
| Backing Store | Anonymous kernel pages | File on disk |
| Persistence | Dies after last detach | Data persists on disk |
| Setup | shm_open + ftruncate | open + mmap / CreateFile + CreateFileMapping |
| Use Case | Fast transient IPC | Large file I/O, partial loading |
| Capacity | Limited by RAM | Limited by RAM + disk size |
| Security | Controlled by object perms | File system permissions |
| Lifetime Control | shm_unlink removes name | File deletion separate from mappings |

Memory-mapped files can also be shared between processes if both map the same file.

---
## 6. Managing Synchronization in Shared Memory
Synchronization preserves consistency when multiple readers/writers access regions.

### 6.1 Primitive Choices
- Mutex / Semaphore: Simplicity; higher overhead, possible contention.
- Condition Variable + Mutex: Wait for state changes without polling.
- Events (Windows) / Futex (Linux low-level) for wake-ups.
- Atomics (std::atomic) for counters, flags, indices.
- Spinlocks for very short critical sections (avoid if work inside uncertain).

### 6.2 Memory Ordering Guidelines
- Producer publish: write payload -> release store of sequence/state.
- Consumer consume: acquire load of sequence/state -> read payload.
- Use relaxed only for non-dependent counters.

### 6.3 Patterns
1. Single-Producer Single-Consumer Ring: indices with atomics, no locks.
2. Multi-Producer Single-Consumer: use atomic fetch_add for ticket, or a lock.
3. Multi-Producer Multi-Consumer: prefer lock per queue or work stealing design.
4. Double Buffer (Ping-Pong): producer flips active index; consumer reads inactive then waits.

### 6.4 Avoid Deadlocks
- Consistent lock ordering.
- Timeouts + fallback.
- Minimize work inside critical section.

### 6.5 Monitoring Health
Store heartbeat timestamp or sequence; consumer detects staleness.

---
## 7. POSIX Shared Memory Explained
POSIX defines portable APIs for shared objects:

### 7.1 Core Calls
- shm_open(name, flags, mode): returns file descriptor in special namespace (often /dev/shm).
- ftruncate(fd, size): sets region size (creator only).
- mmap(NULL, size, PROT_READ|PROT_WRITE, MAP_SHARED, fd, 0): maps into address space.
- munmap(ptr, size): unmap.
- close(fd): close descriptor.
- shm_unlink(name): remove name entry.

### 7.2 Naming Rules
- Name must begin with '/'; no other slashes.
- Namespace separate from regular filesystem.

### 7.3 Permissions
Mode bits behave like file permissions; honor umask.

### 7.4 Synchronization Complements
Named semaphores (sem_open) or robust pthread mutex (in shared memory with PTHREAD_PROCESS_SHARED attribute).

### 7.5 Example Mutex Initialization
```cpp
pthread_mutex_t* mtx;
// allocate inside mapped region
pthread_mutexattr_t attr; pthread_mutexattr_init(&attr);
pthread_mutexattr_setpshared(&attr, PTHREAD_PROCESS_SHARED);
pthread_mutex_init(mtx, &attr);
pthread_mutexattr_destroy(&attr);
```

### 7.6 Cleaning Up
Call shm_unlink once; existing mappings continue until unmapped by processes.

---
## 8. C++ Libraries Supporting Shared Memory
- Boost.Interprocess: High-level abstractions (managed segments, named mutex/semaphore, containers).
- Qt (QSharedMemory): Simple API for sharing byte array + QSystemSemaphore.
- Folly: Provides atomic and lock-free structures (can be used in shared memory with care).
- Intel TBB: Concurrent containers (not directly multi-process unless mapped carefully).
- mmap/System APIs: Direct low-level approach (POSIX, System V, Windows).
- ZeroMQ / nanomsg: Not pure shared memory, but alternative IPC with queues (for comparison).
- OpenMPI / Shared memory transport: HPC contexts.

Selection depends on portability, complexity tolerance, and need for ready-made containers.

---
## 9. Practical Applications of Shared Memory
- High-frequency market data feed distribution.
- Sensor fusion pipelines (camera frames, LiDAR point clouds).
- Multimedia streaming between capture and encode processes.
- Game engine subsystems (physics <-> AI) separation.
- Simulation / robotics frameworks exchanging state.
- Database buffer pools and caching layers.
- Real-time telemetry dashboards.
- IPC between sandboxed components where copying overhead is critical.

---
## 10. Concept and Usage of boost::interprocess
Boost.Interprocess wraps OS APIs, offering managed segments that perform allocation internally.

### 10.1 Core Concepts
- managed_shared_memory: segment manager + allocator.
- named_mutex / named_semaphore: synchronization primitives.
- Offset pointers: portable across process address spaces.
- Interprocess containers: vector, map, string variants using shared allocators.

### 10.2 Basic Example
Producer creates and populates shared objects; consumer opens and reads.

#### Producer
```cpp
// bi_producer.cpp
#include <boost/interprocess/managed_shared_memory.hpp>
#include <boost/interprocess/sync/named_mutex.hpp>
#include <boost/interprocess/sync/scoped_lock.hpp>
#include <iostream>

using namespace boost::interprocess;

struct SharedStruct { int value; char text[32]; };

int main() {
    // Remove old segment (optional safety)
    shared_memory_object::remove("DemoBoostSeg");
    managed_shared_memory segment(create_only, "DemoBoostSeg", 65536);

    // Construct object in segment
    SharedStruct* obj = segment.construct<SharedStruct>("Packet")( );
    obj->value = 123; std::snprintf(obj->text, sizeof(obj->text), "%s", "Boost IPC");

    // Create named mutex
    named_mutex::remove("DemoBoostMtx");
    named_mutex mtx(create_only, "DemoBoostMtx");

    {
        scoped_lock<named_mutex> lock(mtx);
        obj->value = 456; // protected update
    }

    std::cout << "Produced value=" << obj->value << std::endl;
    std::cout << "Press Enter to exit..."; std::cin.get();

    // Cleanup
    segment.destroy<SharedStruct>("Packet");
    named_mutex::remove("DemoBoostMtx");
    shared_memory_object::remove("DemoBoostSeg");
    return 0;
}
```

#### Consumer
```cpp
// bi_consumer.cpp
#include <boost/interprocess/managed_shared_memory.hpp>
#include <boost/interprocess/sync/named_mutex.hpp>
#include <boost/interprocess/sync/scoped_lock.hpp>
#include <iostream>

using namespace boost::interprocess;

struct SharedStruct { int value; char text[32]; };

int main() {
    managed_shared_memory segment(open_only, "DemoBoostSeg");
    SharedStruct* obj = segment.find<SharedStruct>("Packet").first;
    named_mutex mtx(open_only, "DemoBoostMtx");

    {
        scoped_lock<named_mutex> lock(mtx);
        std::cout << "Read value=" << obj->value << " text=" << obj->text << std::endl;
    }
    return 0;
}
```

### 10.3 Containers Example (Shared Vector)
```cpp
#include <boost/interprocess/managed_shared_memory.hpp>
#include <boost/interprocess/containers/vector.hpp>
#include <boost/interprocess/allocators/allocator.hpp>
#include <iostream>
using namespace boost::interprocess;

int main() {
    shared_memory_object::remove("DemoVecSeg");
    managed_shared_memory seg(create_only, "DemoVecSeg", 1 << 20);
    using ShmAllocator = allocator<int, managed_shared_memory::segment_manager>;
    using ShmVector = vector<int, ShmAllocator>;
    ShmAllocator alloc(seg.get_segment_manager());
    ShmVector* v = seg.construct<ShmVector>("Nums")(alloc);
    for (int i = 0; i < 10; ++i) v->push_back(i);
    std::cout << "Vector size=" << v->size() << std::endl;
}
```
Consumer can open_only and find "Nums" similarly.

### 10.4 Advantages
- Reduces boilerplate for allocation and synchronization.
- Provides RAII semantics for resources.
- Portable across platforms supported by Boost.

### 10.5 Considerations
- Segment size must anticipate peak allocation.
- Destroy constructed objects before removing segment.
- Address relocation handled via offset_ptr; avoid raw pointers across processes.

---
Next: 11. Performance optimization techniques
