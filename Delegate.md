# Delegate Pattern - Complete Design Documentation

## Table of Contents
1. [Overview](#overview)
2. [Purpose and Problem Solved](#purpose-and-problem-solved)
3. [Design Architecture](#design-architecture)
4. [Implementation Details](#implementation-details)
5. [Usage Examples](#usage-examples)
6. [Performance Considerations](#performance-considerations)
7. [std::forward Usage Analysis](#stdforward-usage-analysis)
8. [Best Practices](#best-practices)

---

## Overview

The `Delegate` class is a type-safe, thread-safe callback mechanism that implements the **Observer Pattern** (also known as Event/Listener pattern). It allows multiple functions (callbacks) to be registered and invoked together, enabling loose coupling between components.

### Key Features:
- ✅ **Multi-cast**: Multiple callbacks can subscribe to one delegate
- ✅ **Type-safe**: Compile-time type checking via templates
- ✅ **Thread-safe**: Protected by `std::recursive_mutex`
- ✅ **Perfect forwarding**: Efficient argument passing without copies
- ✅ **Return value collection**: Non-void delegates collect all return values
- ✅ **Zero-overhead abstraction**: No runtime overhead when not used

---

## Purpose and Problem Solved

### Problems in Traditional Callback Systems

#### Problem 1: Tight Coupling
```cpp
// ❌ BAD: Tight coupling
class NetworkManager {
    Application* app;  // Direct dependency
public:
    void onDataReceived(Data data) {
        app->handleData(data);  // Must know about Application
    }
};
```

#### Problem 2: Single Callback Limitation
```cpp
// ❌ BAD: Only one callback allowed
class NetworkManager {
    std::function<void(Data)> callback;  // Only ONE subscriber
public:
    void onDataReceived(Data data) {
        if (callback) callback(data);
    }
};
```

#### Problem 3: Manual Memory Management
```cpp
// ❌ BAD: Manual callback management
class NetworkManager {
    std::vector<void(*)(Data)> callbacks;  // Raw function pointers
public:
    void addCallback(void(*func)(Data)) {
        callbacks.push_back(func);  // No RAII, no type safety
    }
};
```

### Solution: Delegate Pattern

```cpp
// ✅ GOOD: Loose coupling with Delegate
class NetworkManager {
    Delegate<void(Data)> onDataReceived;  // Multiple subscribers, type-safe
public:
    template<typename F>
    void addOnDataReceived(F&& callback) {
        onDataReceived += std::forward<F>(callback);
    }
    
    void processData(Data data) {
        onDataReceived(data);  // Notify all subscribers
    }
};
```

**Benefits:**
1. **Loose Coupling**: NetworkManager doesn't know about subscribers
2. **Multiple Subscribers**: Any number of callbacks can register
3. **Type Safety**: Compile-time checking of callback signatures
4. **Automatic Cleanup**: RAII-based memory management
5. **Thread Safety**: Built-in mutex protection

---

## Design Architecture

### Class Hierarchy

```
Delegate<ReturnType(Args...)>
    │
    ├── Uses: DelegateImpl::Invoker<ReturnType, Args...>  // Primary template
    │         └── Specialization: Invoker<void, Args...>   // void specialization
    │
    └── Stores: std::list<shared_ptr<std::function<ReturnType(Args...)>>>
```

### Component Breakdown

#### 1. Delegate Class (Public Interface)
```cpp
template <typename TReturnType, typename... Args>
class Delegate<TReturnType(Args...)> {
    // Public API for registering and invoking callbacks
};
```

#### 2. Invoker Helper (Implementation Detail)
```cpp
namespace DelegateImpl {
    template <typename TReturnType, typename... Args>
    struct Invoker {
        // Handles invocation logic and return value collection
    };
}
```

### Data Structure Choice: `std::list`

**Why `std::list` instead of `std::vector`?**

```cpp
std::list<std::shared_ptr<functionType>> mFunctionList;
```

**Advantages:**
- ✅ **Iterator Stability**: Iterators remain valid after insertions/deletions
- ✅ **No Reallocation**: No memory moves when adding callbacks
- ✅ **Easy Removal**: O(1) removal if we add disconnect functionality later
- ✅ **Cache-Friendly for Small Lists**: Typical use has few callbacks

**Trade-offs:**
- ❌ No random access (not needed for our use case)
- ❌ More memory overhead per element (acceptable for few callbacks)

---

## Implementation Details

### Thread Safety Strategy

```cpp
mutable std::recursive_mutex mMutex;

// Why recursive_mutex?
// Allows same thread to lock multiple times (e.g., callback calling another delegate)
```

**Lock Locations:**
1. **During Registration** (`Connect`/`operator+=`)
2. **During Invocation** (`Invoke`/`operator()`)

**Example Scenario Requiring Recursive Lock:**
```cpp
Delegate<void()> delegate1;
Delegate<void()> delegate2;

// Callback that triggers another delegate
delegate1 += []() {
    delegate2();  // If delegate2 is same thread, needs recursive lock
};
```

### Return Value Collection

#### Non-Void Return Type
```cpp
template <typename TReturnType, typename... Args>
struct Invoker {
    using ReturnType = std::vector<TReturnType>;
    
    static ReturnType Invoke(...) {
        std::vector<TReturnType> returnValues;
        for (const auto& func : delegate.mFunctionList) {
            returnValues.push_back((*func)(std::forward<Args>(params)...));
        }
        return returnValues;  // All return values collected
    }
};
```

#### Void Return Type (Specialization)
```cpp
template <typename... Args>
struct Invoker<void, Args...> {
    static void Invoke(...) {
        for (const auto& func : delegate.mFunctionList) {
            (*func)(std::forward<Args>(params)...);  // No return value
        }
    }
};
```

### Perfect Forwarding Chain

```cpp
// 1. User calls with arguments
delegate(arg1, arg2);

// 2. operator() forwards to Invoker
operator()(Args&&... args) {
    return Invoker::Invoke(*this, std::forward<Args>(args)...);
}

// 3. Invoker forwards to each callback
(*functionPtr)(std::forward<Args>(params)...)
```

**Why Perfect Forwarding?**
- Preserves lvalue/rvalue nature of arguments
- Avoids unnecessary copies
- Enables move semantics for heavy objects

---

## Usage Examples

### Example 1: Simple Event Notification (Void Return)

```cpp
#include "Delegate.hpp"
#include <iostream>
#include <string>

class Button {
private:
    Delegate<void()> onClick;
    
public:
    // Register click handlers
    template<typename F>
    void addOnClick(F&& handler) {
        onClick += std::forward<F>(handler);
    }
    
    void click() {
        std::cout << "Button clicked!\n";
        onClick();  // Notify all handlers
    }
};

int main() {
    Button button;
    
    // Multiple handlers can subscribe
    button.addOnClick([]() { 
        std::cout << "Handler 1: Logging click\n"; 
    });
    
    button.addOnClick([]() { 
        std::cout << "Handler 2: Playing sound\n"; 
    });
    
    button.addOnClick([]() { 
        std::cout << "Handler 3: Updating UI\n"; 
    });
    
    button.click();
    // Output:
    // Button clicked!
    // Handler 1: Logging click
    // Handler 2: Playing sound
    // Handler 3: Updating UI
}
```

### Example 2: Network Data Reception (Real-world Use Case)

```cpp
#include "Delegate.hpp"
#include <memory>

// Message structure
struct ReceiveMessage {
    std::vector<uint8_t> data;
    uint32_t timestamp;
    int32_t AppID;
};

class NetworkService {
private:
    Delegate<void(std::shared_ptr<ReceiveMessage>&)> onRawDataReceived;
    
public:
    template<typename F>
    void addOnRawDataReceived(F&& handler) {
        onRawDataReceived += std::forward<F>(handler);
    }
    
    void handleIncomingData(std::shared_ptr<ReceiveMessage>& msg) {
        // Notify all subscribers
        onRawDataReceived(msg);
    }
};

class EncodeProtocolHandler {
public:
    void handleRawData(std::shared_ptr<ReceiveMessage>& msg) {
        std::cout << "Encode: Processing message from server " 
                  << msg->AppID << std::endl;
        // Parse Encode protocol
    }
};

class LoggingService {
public:
    void logReceivedData(std::shared_ptr<ReceiveMessage>& msg) {
        std::cout << "Logger: Received " << msg->data.size() 
                  << " bytes at timestamp " << msg->timestamp << std::endl;
    }
};

int main() {
    NetworkService network;
    EncodeProtocolHandler EncodeHandler;
    LoggingService logger;
    
    // Subscribe multiple handlers
    network.addOnRawDataReceived([&](auto& msg) {
        EncodeHandler.handleRawData(msg);
    });
    
    network.addOnRawDataReceived([&](auto& msg) {
        logger.logReceivedData(msg);
    });
    
    // Simulate receiving data
    auto message = std::make_shared<ReceiveMessage>();
    message->data = {0x01, 0x02, 0x03};
    message->timestamp = 12345;
    message->AppID = 1;
    
    network.handleIncomingData(message);
    // Both handlers are called automatically
}
```

### Example 3: Timer Callbacks

```cpp
#include "Delegate.hpp"

enum class TimerTypeId {
    HEARTBEAT,
    RECONNECT,
    DATA_SYNC
};

class TimerManager {
private:
    Delegate<void(const TimerTypeId&)> onTimerTimeout;
    
public:
    template<typename F>
    void addOnTimerTimeout(F&& handler) {
        onTimerTimeout += std::forward<F>(handler);
    }
    
    void simulateTimeout(TimerTypeId timerId) {
        std::cout << "Timer expired: " << static_cast<int>(timerId) << std::endl;
        onTimerTimeout(timerId);
    }
};

class HeartbeatService {
public:
    void onTimeout(const TimerTypeId& timerId) {
        if (timerId == TimerTypeId::HEARTBEAT) {
            std::cout << "HeartbeatService: Sending heartbeat\n";
        }
    }
};

class ReconnectService {
public:
    void onTimeout(const TimerTypeId& timerId) {
        if (timerId == TimerTypeId::RECONNECT) {
            std::cout << "ReconnectService: Attempting reconnection\n";
        }
    }
};

int main() {
    TimerManager timerMgr;
    HeartbeatService heartbeat;
    ReconnectService reconnect;
    
    // Register multiple timer handlers
    timerMgr.addOnTimerTimeout([&](const TimerTypeId& id) {
        heartbeat.onTimeout(id);
    });
    
    timerMgr.addOnTimerTimeout([&](const TimerTypeId& id) {
        reconnect.onTimeout(id);
    });
    
    // Simulate different timer events
    timerMgr.simulateTimeout(TimerTypeId::HEARTBEAT);
    timerMgr.simulateTimeout(TimerTypeId::RECONNECT);
}
```

### Example 4: Delegates with Return Values

```cpp
#include "Delegate.hpp"
#include <vector>
#include <numeric>

class DataValidator {
private:
    Delegate<bool(int)> validators;
    
public:
    template<typename F>
    void addValidator(F&& validator) {
        validators += std::forward<F>(validator);
    }
    
    bool validate(int value) {
        // Get all validation results
        std::vector<bool> results = validators(value);
        
        // All validators must pass
        return std::all_of(results.begin(), results.end(), 
                          [](bool result) { return result; });
    }
};

int main() {
    DataValidator validator;
    
    // Add multiple validation rules
    validator.addValidator([](int value) {
        std::cout << "Range check: ";
        bool result = value >= 0 && value <= 100;
        std::cout << (result ? "PASS" : "FAIL") << std::endl;
        return result;
    });
    
    validator.addValidator([](int value) {
        std::cout << "Even check: ";
        bool result = value % 2 == 0;
        std::cout << (result ? "PASS" : "FAIL") << std::endl;
        return result;
    });
    
    validator.addValidator([](int value) {
        std::cout << "Multiple of 10 check: ";
        bool result = value % 10 == 0;
        std::cout << (result ? "PASS" : "FAIL") << std::endl;
        return result;
    });
    
    std::cout << "\nValidating value: 50\n";
    bool isValid = validator.validate(50);
    std::cout << "Overall result: " << (isValid ? "VALID" : "INVALID") << std::endl;
    
    std::cout << "\nValidating value: 45\n";
    isValid = validator.validate(45);
    std::cout << "Overall result: " << (isValid ? "VALID" : "INVALID") << std::endl;
}
```

### Example 5: State Change Notifications (SharedData Pattern)

```cpp
#include "Delegate.hpp"
#include <memory>

// State structures
struct ApnState {
    int32_t teleState;
    int32_t mainState;
    std::string ifname;
};

class NetworkStateManager {
private:
    Delegate<void(std::shared_ptr<ApnState>&)> onApnStateUpdated;
    std::shared_ptr<ApnState> currentState;
    
public:
    NetworkStateManager() : currentState(std::make_shared<ApnState>()) {}
    
    template<typename F>
    void addonApnStateUpdated(F&& handler) {
        onApnStateUpdated += std::forward<F>(handler);
    }
    
    void updateApnState(int32_t realValue, int32_t mValue) {
        currentState->teleState = realValue;
        currentState->mainState = mValue;
        
        // Notify all listeners
        onApnStateUpdated(currentState);
    }
};

class UIService {
public:
    void onStateChanged(std::shared_ptr<ApnState>& state) {
        std::cout << "UI: Network state changed - Tele:" 
                  << state->teleState << " Main:" << state->mainState << std::endl;
        // Update UI indicators
    }
};

class LoggingService {
public:
    void onStateChanged(std::shared_ptr<ApnState>& state) {
        std::cout << "Logger: Recording state change to file\n";
        // Write to log file
    }
};

class AnalyticsService {
public:
    void onStateChanged(std::shared_ptr<ApnState>& state) {
        std::cout << "Analytics: Sending state metrics to server\n";
        // Send analytics data
    }
};

int main() {
    NetworkStateManager stateMgr;
    UIService ui;
    LoggingService logger;
    AnalyticsService analytics;
    
    // Multiple services subscribe to state changes
    stateMgr.addonApnStateUpdated([&](auto& state) {
        ui.onStateChanged(state);
    });
    
    stateMgr.addonApnStateUpdated([&](auto& state) {
        logger.onStateChanged(state);
    });
    
    stateMgr.addonApnStateUpdated([&](auto& state) {
        analytics.onStateChanged(state);
    });
    
    // State change triggers all handlers
    stateMgr.updateApnState(1, 2);
    std::cout << std::endl;
    stateMgr.updateApnState(2, 3);
}
```

### Example 6: Command Pattern Integration

```cpp
#include "Delegate.hpp"

enum class CommandId {
    CONNECT,
    DISCONNECT,
    SEND_DATA,
    RECEIVE_DATA
};

struct Command {
    CommandId id;
    void* data;
};

class CommandDispatcher {
private:
    Delegate<void(std::shared_ptr<Command>&)> onCommandReceived;
    
public:
    template<typename F>
    void addCommandHandler(F&& handler) {
        onCommandReceived += std::forward<F>(handler);
    }
    
    void dispatch(std::shared_ptr<Command> cmd) {
        std::cout << "Dispatching command: " << static_cast<int>(cmd->id) << std::endl;
        onCommandReceived(cmd);
    }
};

class ConnectionHandler {
public:
    void handleCommand(std::shared_ptr<Command>& cmd) {
        if (cmd->id == CommandId::CONNECT || cmd->id == CommandId::DISCONNECT) {
            std::cout << "ConnectionHandler: Processing connection command\n";
        }
    }
};

class DataHandler {
public:
    void handleCommand(std::shared_ptr<Command>& cmd) {
        if (cmd->id == CommandId::SEND_DATA || cmd->id == CommandId::RECEIVE_DATA) {
            std::cout << "DataHandler: Processing data command\n";
        }
    }
};

class AuditHandler {
public:
    void handleCommand(std::shared_ptr<Command>& cmd) {
        std::cout << "AuditHandler: Logging command for audit trail\n";
    }
};

int main() {
    CommandDispatcher dispatcher;
    ConnectionHandler connHandler;
    DataHandler dataHandler;
    AuditHandler auditHandler;
    
    // Register handlers
    dispatcher.addCommandHandler([&](auto& cmd) {
        connHandler.handleCommand(cmd);
    });
    
    dispatcher.addCommandHandler([&](auto& cmd) {
        dataHandler.handleCommand(cmd);
    });
    
    dispatcher.addCommandHandler([&](auto& cmd) {
        auditHandler.handleCommand(cmd);
    });
    
    // Dispatch different commands
    auto connectCmd = std::make_shared<Command>();
    connectCmd->id = CommandId::CONNECT;
    dispatcher.dispatch(connectCmd);
    
    std::cout << std::endl;
    
    auto sendCmd = std::make_shared<Command>();
    sendCmd->id = CommandId::SEND_DATA;
    dispatcher.dispatch(sendCmd);
}
```

---

## Performance Considerations

### Memory Overhead

```cpp
sizeof(Delegate<void()>)  ≈  sizeof(std::recursive_mutex) + sizeof(std::list<...>)
                           ≈  40 bytes + 24 bytes = ~64 bytes
```

**Per Callback:**
```cpp
sizeof(shared_ptr<std::function<void()>>)  ≈  16 bytes (pointer) + 
                                              24 bytes (control block) +
                                              sizeof(function object)
                                           ≈  40+ bytes per callback
```

### Time Complexity

| Operation | Time Complexity | Notes |
|-----------|----------------|-------|
| Registration (`operator+=`) | O(1) | List append |
| Invocation | O(n) | n = number of callbacks |
| Lock acquisition | O(1) amortized | Mutex contention possible |

### Optimization Tips

#### 1. Minimize Callback Count
```cpp
// ❌ BAD: Too many granular callbacks
delegate.addCallback([]() { log("step1"); });
delegate.addCallback([]() { log("step2"); });
delegate.addCallback([]() { log("step3"); });

// ✅ GOOD: Combine related operations
delegate.addCallback([]() { 
    log("step1");
    log("step2");
    log("step3");
});
```

#### 2. Use References for Heavy Objects
```cpp
// ❌ BAD: Copies large object
Delegate<void(LargeData)> delegate;

// ✅ GOOD: Pass by reference
Delegate<void(const LargeData&)> delegate;

// ✅ BETTER: Move semantics for temporary objects
Delegate<void(std::shared_ptr<LargeData>&)> delegate;
```

#### 3. Avoid Recursive Delegate Calls
```cpp
// ⚠️ WARNING: Can cause performance issues
Delegate<void()> delegate1;
Delegate<void()> delegate2;

delegate1 += [&]() {
    delegate2();  // Triggers nested lock acquisition
};
```

---

## std::forward Usage Analysis

### Correct Usage Locations

#### ✅ Location 1: Registration (`operator+=`)
```cpp
template<typename F>
inline Delegate& operator+=(F&& function) noexcept
{
    return Connect(std::forward<F>(function));  // ✅ CORRECT
}
```

**Why:** Preserves value category when storing the function.

#### ✅ Location 2: Connect Function
```cpp
template<typename F>
Delegate& Connect(F&& function) noexcept
{
    mFunctionList.push_back(
        std::make_shared<functionType>(std::forward<F>(function))  // ✅ CORRECT
    );
}
```

**Why:** Forwards once into `std::function` constructor.

#### ✅ Location 3: Argument Forwarding in Invoke
```cpp
static void Invoke(const Delegate<void(Args...)>& delegate, Args&&... params)
{
    for (const auto& functionPtr : delegate.mFunctionList)
    {
        (*functionPtr)(std::forward<Args>(params)...);  // ✅ CORRECT
    }
}
```

**Why:** Forwards arguments to each callback, preserving their value category.

### Incorrect Usage (What to Avoid)

#### ❌ WRONG: Forwarding Stored Functors
```cpp
// ❌ DO NOT DO THIS
for (auto&& functionPtr : delegate.mFunctionList)
{
    std::forward<decltype(functionPtr)>(functionPtr)(...);  // ❌ WRONG
}
```

**Problem:** Would move from stored function, breaking subsequent calls.

#### ❌ WRONG: Multiple Forwards of Same Argument
```cpp
// ❌ DO NOT DO THIS
template<typename T>
void process(T&& arg) {
    func1(std::forward<T>(arg));  // OK first time
    func2(std::forward<T>(arg));  // ❌ WRONG - arg might be moved-from
}

// ✅ CORRECT: Only forward once
template<typename T>
void process(T&& arg) {
    auto& argRef = arg;  // Create reference
    func1(argRef);
    func2(argRef);
}
```

### std::forward Decision Tree

```
Is it a function parameter with T&& or Args&&?
├─ YES: Is it used only once?
│  ├─ YES: Use std::forward<T>(param) ✅
│  └─ NO: Use param directly (or const auto&) ❌
└─ NO: Is it a stored member variable?
   └─ Use it directly, never forward ❌
```

---

## Best Practices

### 1. Naming Convention

```cpp
// ✅ GOOD: Clear, descriptive names
class NetworkService {
    Delegate<void(Data&)> onDataReceived;      // Event name
    
public:
    template<typename F>
    void addOnDataReceived(F&& handler) {      // add + EventName
        onDataReceived += std::forward<F>(handler);
    }
};
```

### 2. Template Parameter for Registration

```cpp
// ✅ GOOD: Template parameter for flexibility
template<typename F>
void addCallback(F&& func) {
    delegate += std::forward<F>(func);
}

// ❌ BAD: Forces std::function conversion early
void addCallback(std::function<void()> func) {
    delegate += func;
}
```

### 3. Const Correctness

```cpp
class Service {
    Delegate<void()> onChange;
    
public:
    // ✅ GOOD: Const method for invocation (doesn't modify logical state)
    void notifyChange() const {
        onChange();  // Invoking callbacks doesn't change Service state
    }
    
    // ✅ GOOD: Non-const for registration (modifies callback list)
    template<typename F>
    void addOnChange(F&& handler) {
        onChange += std::forward<F>(handler);
    }
};
```

### 4. Avoid Callback Hell

```cpp
// ❌ BAD: Nested callbacks are hard to read
service1.addCallback([&]() {
    service2.addCallback([&]() {
        service3.addCallback([&]() {
            // Too deep!
        });
    });
});

// ✅ GOOD: Flat structure with named functions
auto handleService3 = []() { /* ... */ };
auto handleService2 = [&]() { service3.addCallback(handleService3); };
auto handleService1 = [&]() { service2.addCallback(handleService2); };
service1.addCallback(handleService1);
```

### 5. Document Thread Safety Expectations

```cpp
/**
 * @brief Network data reception delegate
 * @note Callbacks are invoked on the network thread
 * @warning Callbacks must be thread-safe if accessing shared state
 */
Delegate<void(std::shared_ptr<Data>&)> onDataReceived;
```

### 6. Consider Disconnect Functionality

```cpp
// Future enhancement: Return handle for disconnection
template<typename F>
size_t addCallback(F&& func) {
    size_t handle = nextHandle++;
    callbacks[handle] = std::forward<F>(func);
    return handle;
}

void removeCallback(size_t handle) {
    callbacks.erase(handle);
}
```

### 7. Error Handling in Callbacks

```cpp
// ✅ GOOD: Catch exceptions in individual callbacks
static void Invoke(...) {
    for (const auto& func : delegate.mFunctionList) {
        try {
            (*func)(std::forward<Args>(params)...);
        } catch (const std::exception& e) {
            // Log error but continue with other callbacks
            LOG("Callback exception: %s", e.what());
        }
    }
}
```

---

## Common Pitfalls and Solutions

### Pitfall 1: Lifetime Issues

```cpp
// ❌ DANGER: Capturing local variables by reference
void setupCallbacks() {
    int localCounter = 0;
    delegate += [&]() {
        localCounter++;  // ❌ Dangling reference when setupCallbacks() returns
    };
}

// ✅ SOLUTION: Capture by value or use shared_ptr
void setupCallbacks() {
    auto counter = std::make_shared<int>(0);
    delegate += [counter]() {
        (*counter)++;  // ✅ Safe
    };
}
```

### Pitfall 2: Recursive Invocation

```cpp
// ❌ DANGER: Infinite recursion
Delegate<void()> delegate;
delegate += [&]() {
    delegate();  // ❌ Calls itself infinitely
};

// ✅ SOLUTION: Guard against recursion
std::atomic<bool> isInvoking{false};
void safeInvoke() {
    if (isInvoking.exchange(true)) return;  // Already invoking
    delegate();
    isInvoking = false;
}
```

### Pitfall 3: Order Dependency

```cpp
// ⚠️ WARNING: Callback order matters
delegate += []() { std::cout << "A"; };
delegate += []() { std::cout << "B"; };
delegate += []() { std::cout << "C"; };
delegate();  // Output: ABC (order is guaranteed)

// ✅ SOLUTION: Document if order matters
/**
 * @note Callbacks are invoked in registration order
 * @warning Do not rely on execution order for correctness
 */
```

---

## Comparison with Alternatives

### vs. `std::function` alone

| Feature | Delegate | std::function |
|---------|----------|---------------|
| Multiple callbacks | ✅ Yes | ❌ No (single callback) |
| Thread-safe | ✅ Yes | ❌ No |
| Return value collection | ✅ Yes | ✅ Yes (single) |
| Memory overhead | Medium | Low |

### vs. Signal/Slot (Qt, Boost.Signals2)

| Feature | Delegate | Qt Signals | Boost.Signals2 |
|---------|----------|------------|----------------|
| No external dependency | ✅ Yes | ❌ Requires Qt | ❌ Requires Boost |
| Compile-time type safety | ✅ Yes | ⚠️ MOC-based | ✅ Yes |
| Thread-safe | ✅ Yes | ⚠️ Queued only | ✅ Yes |
| Disconnect support | ❌ No* | ✅ Yes | ✅ Yes |
| Performance | High | Medium | Medium |

*Can be added if needed

### vs. Observer Pattern (Manual)

```cpp
// Manual Observer Pattern
class Observer {
public:
    virtual void update() = 0;
};

class Subject {
    std::vector<Observer*> observers;
public:
    void attach(Observer* obs) { observers.push_back(obs); }
    void notify() {
        for (auto* obs : observers) obs->update();
    }
};

// Delegate Pattern (cleaner, no inheritance required)
class Subject {
    Delegate<void()> onUpdate;
public:
    template<typename F>
    void attach(F&& callback) {
        onUpdate += std::forward<F>(callback);
    }
    void notify() { onUpdate(); }
};
```

---

## Real-World problem in  Application Flow

### Complete Example: Socket Message Reception Flow

```cpp
// 1. SerializeServices receives raw socket data
class SerializeServices {
    Delegate<void(std::shared_ptr<ReceiveMessage>&)> onReceiveRawMessage;
    
public:
    void handleSockReceive(uint8_t* buffer, size_t length) {
        auto message = std::make_shared<ReceiveMessage>();
        message->data.assign(buffer, buffer + length);
        
        // Trigger delegate
        onReceiveRawMessage(message);
    }
    
    template<typename F>
    void addOnReceiveRawMessage(F&& handler) {
        onReceiveRawMessage += std::forward<F>(handler);
    }
};

// 2. MessageProcessor handles the raw data
class MessageProcessor {
    Delegate<void(std::shared_ptr<ReceiveMessage>&)> onReceiveSocketMessage;
    
public:
    void handleReceiver(std::shared_ptr<ReceiveMessage>& msg) {
        // Process/validate message
        unpackRawData(msg);
        decodeDataToPacket(msg);
        
        // Forward to application layer
        onReceiveSocketMessage(msg);
    }
    
    template<typename F>
    void addOnReceiveSocketMessage(F&& handler) {
        onReceiveSocketMessage += std::forward<F>(handler);
    }
    
private:
    void unpackRawData(std::shared_ptr<ReceiveMessage>& msg) { /* ... */ }
    void decodeDataToPacket(std::shared_ptr<ReceiveMessage>& msg) { /* ... */ }
};

// 3. Application layer consumes processed data
class Application {
public:
    void onSocketMessageReceived(std::shared_ptr<ReceiveMessage>& msg) {
        switch(msg->getMessageType()) {
            case MessageType::SERVER_LOGIN_RESPONSE:
                handleLoginResponse(msg);
                break;
            case MessageType::SERVER_HEARTBEAT:
                handleHeartbeat(msg);
                break;
            case MessageType::REPORT_MESSAGE:
                handleRealtimeData(msg);
                break;
        }
    }
    
private:
    void handleLoginResponse(std::shared_ptr<ReceiveMessage>& msg) { /* ... */ }
    void handleHeartbeat(std::shared_ptr<ReceiveMessage>& msg) { /* ... */ }
    void handleRealtimeData(std::shared_ptr<ReceiveMessage>& msg) { /* ... */ }
};

// 4. Setup the delegate chain
int main() {
    SerializeServices SerializeServices;
    MessageProcessor processor;
    Application app;
    
    // Connect the chain
    SerializeServices.addOnReceiveRawMessage([&](auto& msg) {
        processor.handleReceiver(msg);
    });
    
    processor.addOnReceiveSocketMessage([&](auto& msg) {
        app.onSocketMessageReceived(msg);
    });
    
    // Simulate receiving data
    uint8_t buffer[] = {0x01, 0x02, 0x03, 0x04};
    SerializeServices.handleSockReceive(buffer, sizeof(buffer));
    
    // Data flows through the delegate chain automatically
}
```

---

## Conclusion

The `Delegate` pattern provides a robust, type-safe, and efficient mechanism for implementing callbacks in C++. It successfully decouples components while maintaining performance and thread safety.

### Key Takeaways:
1. ✅ Use delegates for event-driven architectures
2. ✅ Perfect forwarding ensures zero-copy argument passing
3. ✅ Thread-safe by design with `std::recursive_mutex`
4. ✅ Supports both void and non-void return types
5. ✅ Minimal memory overhead for typical use cases
6. ✅ No external dependencies (pure C++17)

### When to Use:
- Multiple listeners for single event
- Loose coupling between modules
- Plugin/extension architectures
- Event-driven systems (data flow, timers, network events)

### When NOT to Use:
- Single callback sufficient (`std::function` is simpler)
- Performance-critical hot paths (virtual dispatch might be faster)
- Need automatic disconnect (requires enhancement)

---

## References

- C++17 Standard: Perfect Forwarding (§17.6.3.3)
- Effective Modern C++ by Scott Meyers - Item 25: Use std::move on rvalue references

---

