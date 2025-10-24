# Function Pointers in C++ - Comprehensive Guide

## Table of Contents
1. [Function Pointers Basics](#1-function-pointers-basics)
2. [Pointers to Member Functions](#2-pointers-to-member-functions)
3. [Function Pointers vs Object Pointers](#3-function-pointers-vs-object-pointers)
4. [Callbacks with Function Pointers](#4-callbacks-with-function-pointers)
5. [Function Pointers with Default Parameters](#5-function-pointers-with-default-parameters)
6. [Function Pointers in STL Containers](#6-function-pointers-in-stl-containers)
7. [Passing Function Pointers as Parameters](#7-passing-function-pointers-as-parameters)
8. [std::function and std::bind](#8-stdfunction-and-stdbind)
9. [Function Pointers with Variable Parameters](#9-function-pointers-with-variable-parameters)
10. [Performance Comparison](#10-performance-comparison)

---

## 1. Function Pointers Basics

### What is a Function Pointer?

A function pointer is a variable that stores the address of a function. Unlike regular pointers that point to data, function pointers point to executable code. They allow you to call functions indirectly and enable dynamic function selection at runtime.

### Syntax

```cpp
return_type (*pointer_name)(parameter_types);
```

### Example 1: Basic Function Pointer

```cpp
#include <iostream>

// Simple functions
int add(int a, int b) {
    return a + b;
}

int subtract(int a, int b) {
    return a - b;
}

int multiply(int a, int b) {
    return a * b;
}

int main() {
    // Declare a function pointer
    int (*operation)(int, int);
    
    // Assign function address to pointer
    operation = add;
    std::cout << "Add: " << operation(10, 5) << std::endl;  // Output: 15
    
    // Reassign to different function
    operation = subtract;
    std::cout << "Subtract: " << operation(10, 5) << std::endl;  // Output: 5
    
    operation = multiply;
    std::cout << "Multiply: " << operation(10, 5) << std::endl;  // Output: 50
    
    return 0;
}
```

### Example 2: Using typedef for Cleaner Syntax

```cpp
#include <iostream>

typedef int (*MathOperation)(int, int);

int divide(int a, int b) {
    return (b != 0) ? a / b : 0;
}

void executeOperation(MathOperation op, int x, int y) {
    std::cout << "Result: " << op(x, y) << std::endl;
}

int main() {
    MathOperation op = divide;
    executeOperation(op, 20, 4);  // Output: Result: 5
    return 0;
}
```

---

## 2. Pointers to Member Functions

### Declaration and Usage

Member function pointers are more complex because they need to be called with an object instance.

### Example 1: Basic Member Function Pointer

```cpp
#include <iostream>

class Calculator {
public:
    int add(int a, int b) {
        return a + b;
    }
    
    int multiply(int a, int b) {
        return a * b;
    }
    
    static int staticAdd(int a, int b) {
        return a + b;
    }
};

int main() {
    // Pointer to member function
    int (Calculator::*memberFuncPtr)(int, int);
    
    // Assign member function
    memberFuncPtr = &Calculator::add;
    
    // Create object
    Calculator calc;
    
    // Call through pointer (using .* operator)
    std::cout << "Member function result: " 
              << (calc.*memberFuncPtr)(10, 5) << std::endl;  // Output: 15
    
    // Using pointer to object (->* operator)
    Calculator* pCalc = &calc;
    memberFuncPtr = &Calculator::multiply;
    std::cout << "Via pointer: " 
              << (pCalc->*memberFuncPtr)(10, 5) << std::endl;  // Output: 50
    
    // Static member functions work like regular functions
    int (*staticFuncPtr)(int, int) = &Calculator::staticAdd;
    std::cout << "Static function: " 
              << staticFuncPtr(10, 5) << std::endl;  // Output: 15
    
    return 0;
}
```

### Example 2: Member Function Pointer with typedef

```cpp
#include <iostream>
#include <string>

class Vehicle {
public:
    std::string name;
    
    Vehicle(const std::string& n) : name(n) {}
    
    void start() {
        std::cout << name << " is starting..." << std::endl;
    }
    
    void stop() {
        std::cout << name << " is stopping..." << std::endl;
    }
    
    void accelerate(int speed) {
        std::cout << name << " accelerating to " << speed << " km/h" << std::endl;
    }
};

// typedef for member function pointer
typedef void (Vehicle::*VehicleAction)();
typedef void (Vehicle::*VehicleSpeedAction)(int);

int main() {
    Vehicle car("BMW");
    
    VehicleAction action = &Vehicle::start;
    (car.*action)();  // Output: BMW is starting...
    
    action = &Vehicle::stop;
    (car.*action)();  // Output: BMW is stopping...
    
    VehicleSpeedAction speedAction = &Vehicle::accelerate;
    (car.*speedAction)(120);  // Output: BMW accelerating to 120 km/h
    
    return 0;
}
```

---

## 3. Function Pointers vs Object Pointers

### Key Differences

| Aspect | Function Pointer | Object Pointer |
|--------|-----------------|----------------|
| **Points to** | Function code in memory | Object data in memory |
| **Syntax** | `return_type (*ptr)(params)` | `Type* ptr` |
| **Dereferencing** | `(*ptr)(args)` or `ptr(args)` | `*ptr` or `ptr->member` |
| **Use case** | Dynamic function selection | Dynamic data access |
| **Memory location** | Code segment | Heap or Stack |

### Example: Comparison

```cpp
#include <iostream>

class DataProcessor {
public:
    int value;
    
    DataProcessor(int v) : value(v) {}
    
    void display() {
        std::cout << "Value: " << value << std::endl;
    }
};

int processData(int x) {
    return x * 2;
}

int main() {
    // Object pointer
    DataProcessor* objPtr = new DataProcessor(42);
    objPtr->display();  // Output: Value: 42
    std::cout << "Object address: " << objPtr << std::endl;
    std::cout << "Object value: " << objPtr->value << std::endl;
    
    // Function pointer
    int (*funcPtr)(int) = processData;
    std::cout << "Function result: " << funcPtr(21) << std::endl;  // Output: 42
    std::cout << "Function address: " << (void*)funcPtr << std::endl;
    
    // Member function pointer (combines both concepts)
    void (DataProcessor::*memberPtr)() = &DataProcessor::display;
    (objPtr->*memberPtr)();  // Requires object to call
    
    delete objPtr;
    return 0;
}
```

---

## 4. Callbacks with Function Pointers

### Concept

Callbacks allow you to pass a function as a parameter to another function, enabling customizable behavior.

### Example 1: Event Handler Callback

```cpp
#include <iostream>
#include <vector>

// Callback function type
typedef void (*EventCallback)(const std::string& event);

// Event handler that uses callbacks
class EventManager {
private:
    std::vector<EventCallback> callbacks;
    
public:
    void registerCallback(EventCallback cb) {
        callbacks.push_back(cb);
    }
    
    void triggerEvent(const std::string& eventName) {
        std::cout << "Event triggered: " << eventName << std::endl;
        for (auto& callback : callbacks) {
            callback(eventName);
        }
    }
};

// Callback implementations
void onButtonClick(const std::string& event) {
    std::cout << "  Button clicked handler: " << event << std::endl;
}

void onMouseMove(const std::string& event) {
    std::cout << "  Mouse moved handler: " << event << std::endl;
}

void logEvent(const std::string& event) {
    std::cout << "  [LOG] Event logged: " << event << std::endl;
}

int main() {
    EventManager manager;
    
    // Register callbacks
    manager.registerCallback(onButtonClick);
    manager.registerCallback(logEvent);
    
    // Trigger events
    manager.triggerEvent("CLICK");
    std::cout << std::endl;
    
    manager.registerCallback(onMouseMove);
    manager.triggerEvent("MOUSE_MOVE");
    
    return 0;
}
```

### Example 2: Array Processing Callback

```cpp
#include <iostream>

typedef bool (*FilterCallback)(int);

void filterArray(int arr[], int size, FilterCallback filter) {
    std::cout << "Filtered values: ";
    for (int i = 0; i < size; i++) {
        if (filter(arr[i])) {
            std::cout << arr[i] << " ";
        }
    }
    std::cout << std::endl;
}

// Filter functions
bool isEven(int n) { return n % 2 == 0; }
bool isPositive(int n) { return n > 0; }
bool isGreaterThan10(int n) { return n > 10; }

int main() {
    int numbers[] = {-5, 2, 15, -8, 23, 4, 30, -1};
    int size = sizeof(numbers) / sizeof(numbers[0]);
    
    filterArray(numbers, size, isEven);          // Output: 2 -8 4 30
    filterArray(numbers, size, isPositive);      // Output: 2 15 23 4 30
    filterArray(numbers, size, isGreaterThan10); // Output: 15 23 30
    
    return 0;
}
```

---

## 5. Function Pointers with Default Parameters

### Important Notes

- Function pointers **cannot** store default parameter values
- When calling through a function pointer, all parameters must be explicitly provided
- Default parameters are resolved at compile-time, but function pointers are resolved at runtime

---

## 4.5. Common Pitfalls: SIGILL (Signal 4) Crashes with Function Pointers

### Understanding SIGILL (Illegal Instruction)

**SIGILL (Signal 4)** is one of the most critical errors when working with function pointers in embedded systems. It indicates the CPU attempted to execute an illegal or invalid instruction, often caused by:

1. **NULL or uninitialized function pointer**
2. **Dangling pointer to deleted/destroyed function**
3. **Memory corruption overwriting function pointer**
4. **ABI mismatch or calling convention issues**
5. **Jumping to data instead of code**
6. **Corrupted vtable in C++ objects**

### Example 1: NULL Function Pointer (Most Common Cause)

```cpp
#include <iostream>
#include <csignal>
#include <cstdlib>

typedef void (*CallbackFunc)(int);

void signalHandler(int signum) {
    std::cout << "\n[CRASH] Signal " << signum << " (SIGILL) caught!" << std::endl;
    std::cout << "Illegal instruction - likely NULL function pointer" << std::endl;
    exit(signum);
}

void validCallback(int value) {
    std::cout << "Valid callback called with: " << value << std::endl;
}

// WRONG: Dangerous code that will crash
void unsafeExample() {
    CallbackFunc callback = nullptr;  // Uninitialized or explicitly NULL
    
    // This will cause SIGILL!
    // callback(42);  // CRASH: Attempting to call NULL pointer
}

// CORRECT: Safe version with validation
void safeExample() {
    CallbackFunc callback = nullptr;
    
    // Always check before calling
    if (callback != nullptr) {
        callback(42);
    } else {
        std::cout << "Callback is NULL - skipping call" << std::endl;
    }
    
    // Now assign valid function
    callback = validCallback;
    if (callback != nullptr) {
        callback(42);  // Safe to call
    }
}

int main() {
    // Install signal handler for debugging
    signal(SIGILL, signalHandler);
    
    std::cout << "=== Safe Example ===" << std::endl;
    safeExample();
    
    std::cout << "\n=== Unsafe Example (commented out to prevent crash) ===" << std::endl;
    // unsafeExample();  // Uncomment to see crash
    
    return 0;
}
```

### Example 2: Dangling Function Pointer (Dynamic Libraries/Plugins)

```cpp
#include <iostream>
#include <memory>
#include <csignal>

typedef int (*OperationFunc)(int, int);

class PluginManager {
private:
    OperationFunc operation;
    bool isLoaded;
    
public:
    PluginManager() : operation(nullptr), isLoaded(false) {}
    
    // Simulates loading a plugin
    void loadPlugin() {
        // In real scenario, this would be dlopen() or LoadLibrary()
        operation = [](int a, int b) -> int { return a + b; };
        isLoaded = true;
        std::cout << "Plugin loaded successfully" << std::endl;
    }
    
    // Simulates unloading a plugin
    void unloadPlugin() {
        // In real scenario: dlclose() or FreeLibrary()
        operation = nullptr;  // Function no longer valid!
        isLoaded = false;
        std::cout << "Plugin unloaded" << std::endl;
    }
    
    // WRONG: Doesn't check if plugin is loaded
    int executeUnsafe(int a, int b) {
        return operation(a, b);  // CRASH if plugin unloaded!
    }
    
    // CORRECT: Validates before calling
    int executeSafe(int a, int b) {
        if (isLoaded && operation != nullptr) {
            return operation(a, b);
        } else {
            std::cerr << "ERROR: Plugin not loaded!" << std::endl;
            return -1;
        }
    }
};

int main() {
    PluginManager manager;
    
    // Safe usage
    std::cout << "=== Safe Usage ===" << std::endl;
    manager.loadPlugin();
    std::cout << "Result: " << manager.executeSafe(10, 5) << std::endl;
    manager.unloadPlugin();
    std::cout << "After unload: " << manager.executeSafe(10, 5) << std::endl;
    
    // Unsafe would crash here:
    // std::cout << "Result: " << manager.executeUnsafe(10, 5) << std::endl;  // SIGILL!
    
    return 0;
}
```

### Example 3: Memory Corruption Overwriting Function Pointer

```cpp
#include <iostream>
#include <cstring>
#include <csignal>

typedef void (*HandlerFunc)();

void validHandler() {
    std::cout << "Valid handler executed" << std::endl;
}

void demonstrateMemoryCorruption() {
    HandlerFunc handlers[3];
    handlers[0] = validHandler;
    handlers[1] = validHandler;
    handlers[2] = validHandler;
    
    std::cout << "Original function pointers:" << std::endl;
    std::cout << "handlers[0] = " << (void*)handlers[0] << std::endl;
    std::cout << "handlers[1] = " << (void*)handlers[1] << std::endl;
    std::cout << "handlers[2] = " << (void*)handlers[2] << std::endl;
    
    // WRONG: Buffer overflow corrupts function pointers
    char buffer[8];
    // This overflow will corrupt handlers array!
    // strcpy(buffer, "This is a very long string that will overflow");
    
    // Simulating corruption
    std::cout << "\n=== Simulating memory corruption ===" << std::endl;
    memset(&handlers[1], 0xFF, sizeof(HandlerFunc));  // Corrupt pointer
    
    std::cout << "After corruption:" << std::endl;
    std::cout << "handlers[1] = " << (void*)handlers[1] << std::endl;
    
    // Attempting to call corrupted pointer
    std::cout << "\nCalling handlers:" << std::endl;
    handlers[0]();  // OK
    
    // Check before calling corrupted pointer
    if (handlers[1] == validHandler) {
        handlers[1]();  // Would be OK
    } else {
        std::cout << "handlers[1] corrupted - skipping!" << std::endl;
    }
}

int main() {
    demonstrateMemoryCorruption();
    return 0;
}
```

### Example 4: Type Mismatch / ABI Issues

```cpp
#include <iostream>

// Function with different calling conventions
extern "C" {
    typedef int (*CStyleFunc)(int, int);
}

typedef int (*CppStyleFunc)(int, int);

int add(int a, int b) {
    return a + b;
}

// WRONG: Potential ABI mismatch
void demonstrateTypeMismatch() {
    // Dangerous: Casting between incompatible function types
    typedef void (*VoidFunc)();
    typedef int (*IntFunc)(int, int);
    
    IntFunc intFunc = add;
    
    // VERY DANGEROUS: Wrong signature cast!
    // VoidFunc voidFunc = (VoidFunc)intFunc;
    // voidFunc();  // SIGILL or undefined behavior!
    
    std::cout << "Type safety is critical with function pointers!" << std::endl;
}

int main() {
    demonstrateTypeMismatch();
    return 0;
}
```

### Example 5: Comprehensive Safety Wrapper

```cpp
#include <iostream>
#include <functional>
#include <stdexcept>
#include <csignal>
#include <csetjmp>

// Safe function pointer wrapper
template<typename Ret, typename... Args>
class SafeFunctionPointer {
private:
    typedef Ret (*FuncPtr)(Args...);
    FuncPtr funcPtr;
    bool isValid;
    std::string name;
    
    // Magic number for validation (simple integrity check)
    static constexpr uint32_t MAGIC = 0xDEADBEEF;
    uint32_t magic;
    
public:
    SafeFunctionPointer(const std::string& funcName = "Unknown") 
        : funcPtr(nullptr), isValid(false), name(funcName), magic(MAGIC) {}
    
    void setFunction(FuncPtr ptr) {
        if (ptr == nullptr) {
            std::cerr << "WARNING: Attempting to set NULL pointer for " << name << std::endl;
            funcPtr = nullptr;
            isValid = false;
        } else {
            funcPtr = ptr;
            isValid = true;
        }
    }
    
    bool validate() const {
        // Check magic number (detects memory corruption)
        if (magic != MAGIC) {
            std::cerr << "CRITICAL: Memory corruption detected in " << name << std::endl;
            return false;
        }
        
        // Check if function pointer is set
        if (!isValid || funcPtr == nullptr) {
            std::cerr << "ERROR: Function pointer " << name << " not initialized" << std::endl;
            return false;
        }
        
        return true;
    }
    
    Ret call(Args... args) {
        if (!validate()) {
            throw std::runtime_error("Invalid function pointer: " + name);
        }
        
        try {
            return funcPtr(args...);
        } catch (const std::exception& e) {
            std::cerr << "Exception in " << name << ": " << e.what() << std::endl;
            throw;
        }
    }
    
    // Safe call with default return value
    Ret callSafe(Ret defaultValue, Args... args) {
        if (!validate()) {
            return defaultValue;
        }
        
        try {
            return funcPtr(args...);
        } catch (...) {
            std::cerr << "Exception caught in " << name << std::endl;
            return defaultValue;
        }
    }
    
    bool isSet() const {
        return isValid && funcPtr != nullptr && magic == MAGIC;
    }
};

// Example usage
int multiply(int a, int b) {
    return a * b;
}

int divide(int a, int b) {
    if (b == 0) throw std::runtime_error("Division by zero");
    return a / b;
}

int main() {
    std::cout << "=== Safe Function Pointer Example ===" << std::endl;
    
    SafeFunctionPointer<int, int, int> mathOp("MultiplyOperation");
    
    // Try to call before initialization
    std::cout << "\n1. Calling uninitialized function:" << std::endl;
    int result = mathOp.callSafe(-1, 10, 5);
    std::cout << "Result (should be default -1): " << result << std::endl;
    
    // Set valid function
    std::cout << "\n2. Setting and calling valid function:" << std::endl;
    mathOp.setFunction(multiply);
    if (mathOp.isSet()) {
        result = mathOp.call(10, 5);
        std::cout << "10 * 5 = " << result << std::endl;
    }
    
    // Change to division
    std::cout << "\n3. Changing to division function:" << std::endl;
    mathOp.setFunction(divide);
    result = mathOp.callSafe(0, 20, 4);
    std::cout << "20 / 4 = " << result << std::endl;
    
    // Try division by zero with exception handling
    std::cout << "\n4. Division by zero (safe call):" << std::endl;
    result = mathOp.callSafe(-999, 20, 0);
    std::cout << "Result (should be default -999): " << result << std::endl;
    
    // Simulate memory corruption
    std::cout << "\n5. Simulating memory corruption:" << std::endl;
    SafeFunctionPointer<int, int, int>* pOp = &mathOp;
    // Corrupt the magic number (in real scenario this could happen via buffer overflow)
    // *((uint32_t*)((char*)pOp + sizeof(void*) + sizeof(bool) + sizeof(std::string))) = 0x00000000;
    
    std::cout << "Is still valid? " << (mathOp.isSet() ? "Yes" : "No") << std::endl;
    
    return 0;
}
```

### Example 6: Real-World Automotive Example - CAN Message Handler

```cpp
#include <iostream>
#include <map>
#include <vector>
#include <cstdint>
#include <csignal>

// Simulating CAN message structure
struct CANMessage {
    uint32_t id;
    uint8_t data[8];
    uint8_t length;
};

typedef void (*CANMessageHandler)(const CANMessage& msg);

class CANMessageDispatcher {
private:
    std::map<uint32_t, CANMessageHandler> handlers;
    std::vector<uint32_t> validMessageIds;
    
public:
    // Register handler for specific CAN ID
    bool registerHandler(uint32_t canId, CANMessageHandler handler) {
        if (handler == nullptr) {
            std::cerr << "ERROR: Cannot register NULL handler for CAN ID 0x" 
                      << std::hex << canId << std::dec << std::endl;
            return false;
        }
        
        handlers[canId] = handler;
        validMessageIds.push_back(canId);
        std::cout << "Registered handler for CAN ID 0x" << std::hex << canId << std::dec << std::endl;
        return true;
    }
    
    // Unregister handler
    void unregisterHandler(uint32_t canId) {
        auto it = handlers.find(canId);
        if (it != handlers.end()) {
            handlers.erase(it);
            std::cout << "Unregistered handler for CAN ID 0x" << std::hex << canId << std::dec << std::endl;
        }
    }
    
    // WRONG: Unsafe dispatch
    void dispatchUnsafe(const CANMessage& msg) {
        handlers[msg.id](msg);  // CRASH if handler not registered!
    }
    
    // CORRECT: Safe dispatch with validation
    bool dispatchSafe(const CANMessage& msg) {
        auto it = handlers.find(msg.id);
        
        if (it == handlers.end()) {
            std::cerr << "WARNING: No handler for CAN ID 0x" << std::hex << msg.id << std::dec << std::endl;
            return false;
        }
        
        if (it->second == nullptr) {
            std::cerr << "CRITICAL: Handler for CAN ID 0x" << std::hex << msg.id 
                      << std::dec << " is NULL!" << std::endl;
            return false;
        }
        
        try {
            it->second(msg);
            return true;
        } catch (const std::exception& e) {
            std::cerr << "Exception in handler for CAN ID 0x" << std::hex << msg.id 
                      << std::dec << ": " << e.what() << std::endl;
            return false;
        }
    }
};

// Example CAN message handlers
void handleDoorStatus(const CANMessage& msg) {
    std::cout << "Door status handler - CAN ID: 0x" << std::hex << msg.id << std::dec << std::endl;
}

void handleEngineRPM(const CANMessage& msg) {
    uint16_t rpm = (msg.data[0] << 8) | msg.data[1];
    std::cout << "Engine RPM: " << rpm << std::endl;
}

int main() {
    CANMessageDispatcher dispatcher;
    
    // Register handlers
    dispatcher.registerHandler(0x100, handleDoorStatus);
    dispatcher.registerHandler(0x200, handleEngineRPM);
    
    // Create test messages
    CANMessage doorMsg = {0x100, {0x01, 0x00}, 2};
    CANMessage rpmMsg = {0x200, {0x0F, 0xA0}, 2};  // 4000 RPM
    CANMessage unknownMsg = {0x999, {0x00}, 1};
    
    std::cout << "\n=== Dispatching Messages ===" << std::endl;
    dispatcher.dispatchSafe(doorMsg);
    dispatcher.dispatchSafe(rpmMsg);
    dispatcher.dispatchSafe(unknownMsg);  // No handler - safe handling
    
    // Unregister and try again
    std::cout << "\n=== After Unregistering Door Handler ===" << std::endl;
    dispatcher.unregisterHandler(0x100);
    dispatcher.dispatchSafe(doorMsg);  // Safe - won't crash
    
    return 0;
}
```

### Best Practices to Prevent SIGILL

#### 1. **Always Initialize Function Pointers**
```cpp
// WRONG
void (*callback)();  // Uninitialized!

// CORRECT
void (*callback)() = nullptr;
```

#### 2. **Always Check Before Calling**
```cpp
// CORRECT
if (callback != nullptr) {
    callback();
}
```

#### 3. **Use Smart Wrappers**
```cpp
// Use std::function which handles nullptr checks better
std::function<void()> safeCallback = nullptr;
if (safeCallback) {  // Built-in validity check
    safeCallback();
}
```

#### 4. **Validate in Debug Builds**
```cpp
#ifdef DEBUG
    #define CALL_FUNC_PTR(ptr, ...) \
        do { \
            if (ptr == nullptr) { \
                fprintf(stderr, "NULL function pointer at %s:%d\n", __FILE__, __LINE__); \
                abort(); \
            } \
            ptr(__VA_ARGS__); \
        } while(0)
#else
    #define CALL_FUNC_PTR(ptr, ...) ptr(__VA_ARGS__)
#endif
```

#### 5. **Use RAII for Plugin Management**
```cpp
class PluginHandle {
    void* handle;
    OperationFunc func;
public:
    PluginHandle(const char* path) {
        handle = dlopen(path, RTLD_LAZY);
        if (handle) {
            func = (OperationFunc)dlsym(handle, "operation");
        }
    }
    
    ~PluginHandle() {
        if (handle) {
            dlclose(handle);
            func = nullptr;  // Prevent dangling pointer
        }
    }
    
    bool isValid() const { return func != nullptr; }
    int execute(int a, int b) {
        if (!isValid()) throw std::runtime_error("Invalid plugin");
        return func(a, b);
    }
};
```

### Debugging SIGILL Crashes

#### Using GDB to Debug Function Pointer Crashes

```bash
# Run program in GDB
gdb ./your_program

# Set breakpoint before suspected crash
(gdb) break main
(gdb) run

# When SIGILL occurs:
(gdb) where          # Show call stack
(gdb) info registers # Check instruction pointer
(gdb) x/i $pc        # Examine instruction at crash point
(gdb) print callback # Print function pointer value
```

#### Common SIGILL Patterns

```cpp
// Pattern 1: NULL pointer (address 0x00000000)
// Pattern 2: Invalid address (0xDEADBEEF, 0xFFFFFFFF)
// Pattern 3: Stack/Heap address (0x7fff..., 0x5555...)
// Pattern 4: Very low address (0x00000004, 0x00000008) - often NULL + offset
```

### Summary: SIGILL Prevention Checklist

- [ ] **Initialize all function pointers to nullptr**
- [ ] **Check for nullptr before every indirect call**
- [ ] **Validate function pointers after plugin load/unload**
- [ ] **Use type-safe wrappers (std::function)**
- [ ] **Add bounds checking for function pointer arrays**
- [ ] **Implement magic number validation for critical pointers**
- [ ] **Use RAII for resource management**
- [ ] **Enable compiler warnings (-Wall -Wextra)**
- [ ] **Use address sanitizer during testing (-fsanitize=address)**
- [ ] **Add defensive logging before function pointer calls**
- [ ] **Test error paths and edge cases**
- [ ] **Review code for buffer overflows near function pointers**

---

### Example 1: Demonstrating the Limitation

```cpp
#include <iostream>

// Function with default parameters
void greet(const std::string& name, const std::string& greeting = "Hello") {
    std::cout << greeting << ", " << name << "!" << std::endl;
}

int main() {
    // Direct call - default parameter works
    greet("Alice");           // Output: Hello, Alice!
    greet("Bob", "Hi");       // Output: Hi, Bob!
    
    // Function pointer declaration
    void (*greetPtr)(const std::string&, const std::string&);
    greetPtr = greet;
    
    // Must provide all parameters when calling through pointer
    // greetPtr("Alice");  // ERROR: Won't compile
    greetPtr("Alice", "Hello");  // OK: Output: Hello, Alice!
    greetPtr("Charlie", "Hey");  // Output: Hey, Charlie!
    
    return 0;
}
```

### Example 2: Workaround Using Wrapper Functions

```cpp
#include <iostream>

void processData(int value, int multiplier = 2, int offset = 0) {
    int result = (value * multiplier) + offset;
    std::cout << "Result: " << result << std::endl;
}

// Wrapper functions to simulate default parameters
void processDataDefault(int value) {
    processData(value, 2, 0);  // Apply defaults manually
}

void processDataWithMultiplier(int value, int multiplier) {
    processData(value, multiplier, 0);
}

int main() {
    // Function pointer
    void (*processor)(int, int, int) = processData;
    
    // Must provide all parameters
    processor(10, 2, 0);   // Output: Result: 20
    processor(10, 3, 5);   // Output: Result: 35
    
    // Using wrapper for "default" behavior
    void (*defaultProcessor)(int) = processDataDefault;
    defaultProcessor(10);  // Output: Result: 20
    
    return 0;
}
```

---

## 6. Function Pointers in STL Containers

### Example 1: Vector of Function Pointers

```cpp
#include <iostream>
#include <vector>
#include <string>

typedef void (*CommandHandler)(const std::string&);

void handleStart(const std::string& param) {
    std::cout << "Starting with parameter: " << param << std::endl;
}

void handleStop(const std::string& param) {
    std::cout << "Stopping with parameter: " << param << std::endl;
}

void handlePause(const std::string& param) {
    std::cout << "Pausing with parameter: " << param << std::endl;
}

void handleResume(const std::string& param) {
    std::cout << "Resuming with parameter: " << param << std::endl;
}

int main() {
    // Vector of function pointers
    std::vector<CommandHandler> commands;
    
    commands.push_back(handleStart);
    commands.push_back(handleStop);
    commands.push_back(handlePause);
    commands.push_back(handleResume);
    
    // Execute all commands
    for (size_t i = 0; i < commands.size(); i++) {
        commands[i]("param_" + std::to_string(i));
    }
    
    return 0;
}
```

### Example 2: Map of Function Pointers (Command Pattern)

```cpp
#include <iostream>
#include <map>
#include <string>

typedef int (*MathOp)(int, int);

int add(int a, int b) { return a + b; }
int subtract(int a, int b) { return a - b; }
int multiply(int a, int b) { return a * b; }
int divide(int a, int b) { return b != 0 ? a / b : 0; }

int main() {
    // Map string commands to function pointers
    std::map<std::string, MathOp> operations;
    
    operations["add"] = add;
    operations["sub"] = subtract;
    operations["mul"] = multiply;
    operations["div"] = divide;
    
    // Use the map to execute commands
    std::cout << "10 + 5 = " << operations["add"](10, 5) << std::endl;
    std::cout << "10 - 5 = " << operations["sub"](10, 5) << std::endl;
    std::cout << "10 * 5 = " << operations["mul"](10, 5) << std::endl;
    std::cout << "10 / 5 = " << operations["div"](10, 5) << std::endl;
    
    // Dynamic command execution
    std::string cmd = "mul";
    if (operations.find(cmd) != operations.end()) {
        std::cout << "Dynamic: " << operations[cmd](7, 3) << std::endl;
    }
    
    return 0;
}
```

### Example 3: Array of Member Function Pointers

```cpp
#include <iostream>
#include <array>

class StateMachine {
public:
    void stateIdle() { std::cout << "State: IDLE" << std::endl; }
    void stateRunning() { std::cout << "State: RUNNING" << std::endl; }
    void statePaused() { std::cout << "State: PAUSED" << std::endl; }
    void stateStopped() { std::cout << "State: STOPPED" << std::endl; }
};

typedef void (StateMachine::*StateHandler)();

int main() {
    StateMachine machine;
    
    // Array of member function pointers
    std::array<StateHandler, 4> states = {
        &StateMachine::stateIdle,
        &StateMachine::stateRunning,
        &StateMachine::statePaused,
        &StateMachine::stateStopped
    };
    
    // Execute all states
    for (auto state : states) {
        (machine.*state)();
    }
    
    return 0;
}
```

---

## 7. Passing Function Pointers as Parameters

### Example 1: Function Pointer as Parameter

```cpp
#include <iostream>
#include <vector>

typedef int (*TransformFunc)(int);

int square(int x) { return x * x; }
int cube(int x) { return x * x * x; }
int doubleValue(int x) { return x * 2; }

// Function that accepts function pointer as parameter
void transformArray(std::vector<int>& arr, TransformFunc transform) {
    for (auto& val : arr) {
        val = transform(val);
    }
}

void printArray(const std::vector<int>& arr) {
    for (const auto& val : arr) {
        std::cout << val << " ";
    }
    std::cout << std::endl;
}

int main() {
    std::vector<int> numbers = {1, 2, 3, 4, 5};
    
    std::cout << "Original: ";
    printArray(numbers);
    
    transformArray(numbers, square);
    std::cout << "Squared: ";
    printArray(numbers);  // Output: 1 4 9 16 25
    
    transformArray(numbers, doubleValue);
    std::cout << "Doubled: ";
    printArray(numbers);  // Output: 2 8 18 32 50
    
    return 0;
}
```

### Example 2: Multiple Function Pointers as Parameters

```cpp
#include <iostream>

typedef int (*BinaryOp)(int, int);

int add(int a, int b) { return a + b; }
int multiply(int a, int b) { return a * b; }
int subtract(int a, int b) { return a - b; }

// Function accepting multiple function pointers
int complexCalculation(int a, int b, int c, 
                       BinaryOp op1, BinaryOp op2) {
    int temp = op1(a, b);
    return op2(temp, c);
}

int main() {
    int result1 = complexCalculation(5, 3, 2, add, multiply);
    // (5 + 3) * 2 = 16
    std::cout << "Result 1: " << result1 << std::endl;
    
    int result2 = complexCalculation(10, 4, 2, multiply, subtract);
    // (10 * 4) - 2 = 38
    std::cout << "Result 2: " << result2 << std::endl;
    
    int result3 = complexCalculation(15, 5, 3, subtract, add);
    // (15 - 5) + 3 = 13
    std::cout << "Result 3: " << result3 << std::endl;
    
    return 0;
}
```

### Example 3: Template with Function Pointer

```cpp
#include <iostream>
#include <vector>
#include <algorithm>

// Generic apply function using templates
template<typename T, typename Func>
void applyToAll(std::vector<T>& container, Func operation) {
    for (auto& item : container) {
        operation(item);
    }
}

void printInt(int& x) {
    std::cout << x << " ";
}

void incrementInt(int& x) {
    x++;
}

void doubleInt(int& x) {
    x *= 2;
}

int main() {
    std::vector<int> numbers = {1, 2, 3, 4, 5};
    
    std::cout << "Original: ";
    applyToAll(numbers, printInt);
    std::cout << std::endl;
    
    applyToAll(numbers, incrementInt);
    std::cout << "After increment: ";
    applyToAll(numbers, printInt);
    std::cout << std::endl;
    
    applyToAll(numbers, doubleInt);
    std::cout << "After double: ";
    applyToAll(numbers, printInt);
    std::cout << std::endl;
    
    return 0;
}
```

---

## 8. std::function and std::bind

### Modern C++ Alternative to Function Pointers

`std::function` is a general-purpose polymorphic function wrapper. It's more flexible than raw function pointers.

### Example 1: Basic std::function

```cpp
#include <iostream>
#include <functional>

int add(int a, int b) {
    return a + b;
}

class Multiplier {
public:
    int operator()(int a, int b) const {
        return a * b;
    }
};

int main() {
    // std::function can hold regular functions
    std::function<int(int, int)> func = add;
    std::cout << "Function: " << func(10, 5) << std::endl;
    
    // Can hold functors
    func = Multiplier();
    std::cout << "Functor: " << func(10, 5) << std::endl;
    
    // Can hold lambdas
    func = [](int a, int b) { return a - b; };
    std::cout << "Lambda: " << func(10, 5) << std::endl;
    
    return 0;
}
```

### Example 2: std::bind for Partial Application

```cpp
#include <iostream>
#include <functional>

int multiply(int a, int b, int c) {
    return a * b * c;
}

class Calculator {
public:
    int divide(int a, int b) {
        return b != 0 ? a / b : 0;
    }
};

int main() {
    using namespace std::placeholders;
    
    // Bind first parameter
    auto multiplyBy2 = std::bind(multiply, 2, _1, _2);
    std::cout << "2 * 3 * 4 = " << multiplyBy2(3, 4) << std::endl;
    
    // Bind first two parameters
    auto multiplyBy2And3 = std::bind(multiply, 2, 3, _1);
    std::cout << "2 * 3 * 5 = " << multiplyBy2And3(5) << std::endl;
    
    // Bind member function
    Calculator calc;
    auto divideBy2 = std::bind(&Calculator::divide, &calc, _1, 2);
    std::cout << "10 / 2 = " << divideBy2(10) << std::endl;
    
    // Reorder parameters
    auto subtractReversed = std::bind(std::minus<int>(), _2, _1);
    std::cout << "Reversed 10 - 3 (3 - 10) = " 
              << subtractReversed(10, 3) << std::endl;
    
    return 0;
}
```

### Example 3: std::function in Containers

```cpp
#include <iostream>
#include <functional>
#include <vector>
#include <map>

class TaskManager {
private:
    std::vector<std::function<void()>> tasks;
    
public:
    void addTask(std::function<void()> task) {
        tasks.push_back(task);
    }
    
    void executeTasks() {
        for (auto& task : tasks) {
            task();
        }
    }
};

void task1() {
    std::cout << "Executing task 1" << std::endl;
}

class Worker {
public:
    void doWork(int id) {
        std::cout << "Worker doing task " << id << std::endl;
    }
};

int main() {
    TaskManager manager;
    
    // Add regular function
    manager.addTask(task1);
    
    // Add lambda
    manager.addTask([]() {
        std::cout << "Executing lambda task" << std::endl;
    });
    
    // Add bound member function
    Worker worker;
    manager.addTask(std::bind(&Worker::doWork, &worker, 42));
    
    // Add lambda with capture
    int counter = 0;
    manager.addTask([&counter]() {
        std::cout << "Counter task: " << ++counter << std::endl;
    });
    
    manager.executeTasks();
    
    return 0;
}
```

### Example 4: Comparison - Function Pointer vs std::function

```cpp
#include <iostream>
#include <functional>

int operation(int a, int b) {
    return a + b;
}

int main() {
    // Traditional function pointer
    int (*funcPtr)(int, int) = operation;
    std::cout << "Function pointer: " << funcPtr(5, 3) << std::endl;
    
    // std::function - more flexible
    std::function<int(int, int)> stdFunc = operation;
    std::cout << "std::function: " << stdFunc(5, 3) << std::endl;
    
    // std::function can hold lambda (function pointer cannot)
    stdFunc = [](int a, int b) { return a * b; };
    std::cout << "std::function with lambda: " << stdFunc(5, 3) << std::endl;
    
    // Function pointer advantages: simpler, faster
    // std::function advantages: flexible, can hold lambdas, functors, bound functions
    
    return 0;
}
```

---

## 9. Function Pointers with Variable Parameters

### Handling Different Parameter Counts

### Example 1: Using Variadic Templates

```cpp
#include <iostream>

// Variadic template function
template<typename... Args>
void print(Args... args) {
    (std::cout << ... << args) << std::endl;
}

template<typename Func, typename... Args>
void callFunction(Func func, Args... args) {
    func(args...);
}

void func1(int a) {
    std::cout << "func1: " << a << std::endl;
}

void func2(int a, int b) {
    std::cout << "func2: " << a << ", " << b << std::endl;
}

void func3(int a, int b, int c) {
    std::cout << "func3: " << a << ", " << b << ", " << c << std::endl;
}

int main() {
    callFunction(func1, 10);
    callFunction(func2, 10, 20);
    callFunction(func3, 10, 20, 30);
    
    return 0;
}
```

### Example 2: Using std::function with Variable Arguments

```cpp
#include <iostream>
#include <functional>
#include <vector>

class FunctionRegistry {
private:
    std::vector<std::function<void()>> tasks;
    
public:
    // Register function with no parameters
    void registerTask(std::function<void()> task) {
        tasks.push_back(task);
    }
    
    // Register function with parameters using bind
    template<typename Func, typename... Args>
    void registerTask(Func func, Args... args) {
        tasks.push_back(std::bind(func, args...));
    }
    
    void executeTasks() {
        for (auto& task : tasks) {
            task();
        }
    }
};

void noParams() {
    std::cout << "Function with no parameters" << std::endl;
}

void oneParam(int a) {
    std::cout << "Function with one parameter: " << a << std::endl;
}

void twoParams(int a, int b) {
    std::cout << "Function with two parameters: " << a << ", " << b << std::endl;
}

void threeParams(int a, int b, int c) {
    std::cout << "Function with three parameters: " 
              << a << ", " << b << ", " << c << std::endl;
}

int main() {
    FunctionRegistry registry;
    
    registry.registerTask(noParams);
    registry.registerTask(oneParam, 10);
    registry.registerTask(twoParams, 20, 30);
    registry.registerTask(threeParams, 40, 50, 60);
    
    registry.executeTasks();
    
    return 0;
}
```

### Example 3: Function Wrapper for Different Signatures

```cpp
#include <iostream>
#include <functional>
#include <map>
#include <string>

class CommandDispatcher {
private:
    std::map<std::string, std::function<void()>> commands;
    
public:
    // Register command with no arguments
    void registerCommand(const std::string& name, void (*func)()) {
        commands[name] = func;
    }
    
    // Register command with arguments (using lambda wrapper)
    template<typename Func, typename... Args>
    void registerCommand(const std::string& name, Func func, Args... args) {
        commands[name] = [=]() { func(args...); };
    }
    
    void executeCommand(const std::string& name) {
        if (commands.find(name) != commands.end()) {
            commands[name]();
        } else {
            std::cout << "Command not found: " << name << std::endl;
        }
    }
};

void cmdExit() {
    std::cout << "Executing EXIT command" << std::endl;
}

void cmdPrint(const std::string& msg) {
    std::cout << "Print: " << msg << std::endl;
}

void cmdAdd(int a, int b) {
    std::cout << "Add result: " << (a + b) << std::endl;
}

int main() {
    CommandDispatcher dispatcher;
    
    dispatcher.registerCommand("exit", cmdExit);
    dispatcher.registerCommand("print", cmdPrint, "Hello World");
    dispatcher.registerCommand("add", cmdAdd, 15, 25);
    
    dispatcher.executeCommand("exit");
    dispatcher.executeCommand("print");
    dispatcher.executeCommand("add");
    
    return 0;
}
```

---

## 10. Performance Comparison: Function Pointers vs std::function

### Benchmark Overview

Function pointers are generally faster due to less overhead, while std::function provides more flexibility at the cost of performance.

### Example 1: Simple Performance Test

```cpp
#include <iostream>
#include <functional>
#include <chrono>
#include <vector>

int simpleAdd(int a, int b) {
    return a + b;
}

// Performance measurement helper
template<typename Func>
long long measureTime(Func func, int iterations) {
    auto start = std::chrono::high_resolution_clock::now();
    
    volatile int result = 0;  // Prevent optimization
    for (int i = 0; i < iterations; i++) {
        result = func(i, i + 1);
    }
    
    auto end = std::chrono::high_resolution_clock::now();
    return std::chrono::duration_cast<std::chrono::microseconds>(end - start).count();
}

int main() {
    const int ITERATIONS = 10000000;
    
    // Test 1: Direct function call
    auto start = std::chrono::high_resolution_clock::now();
    volatile int result = 0;
    for (int i = 0; i < ITERATIONS; i++) {
        result = simpleAdd(i, i + 1);
    }
    auto end = std::chrono::high_resolution_clock::now();
    auto directTime = std::chrono::duration_cast<std::chrono::microseconds>(end - start).count();
    
    // Test 2: Function pointer
    int (*funcPtr)(int, int) = simpleAdd;
    start = std::chrono::high_resolution_clock::now();
    for (int i = 0; i < ITERATIONS; i++) {
        result = funcPtr(i, i + 1);
    }
    end = std::chrono::high_resolution_clock::now();
    auto funcPtrTime = std::chrono::duration_cast<std::chrono::microseconds>(end - start).count();
    
    // Test 3: std::function
    std::function<int(int, int)> stdFunc = simpleAdd;
    start = std::chrono::high_resolution_clock::now();
    for (int i = 0; i < ITERATIONS; i++) {
        result = stdFunc(i, i + 1);
    }
    end = std::chrono::high_resolution_clock::now();
    auto stdFuncTime = std::chrono::duration_cast<std::chrono::microseconds>(end - start).count();
    
    // Test 4: Lambda
    auto lambda = [](int a, int b) { return a + b; };
    start = std::chrono::high_resolution_clock::now();
    for (int i = 0; i < ITERATIONS; i++) {
        result = lambda(i, i + 1);
    }
    end = std::chrono::high_resolution_clock::now();
    auto lambdaTime = std::chrono::duration_cast<std::chrono::microseconds>(end - start).count();
    
    std::cout << "Performance Results (" << ITERATIONS << " iterations):" << std::endl;
    std::cout << "----------------------------------------" << std::endl;
    std::cout << "Direct call:      " << directTime << " μs (baseline)" << std::endl;
    std::cout << "Function pointer: " << funcPtrTime << " μs ("
              << (float)funcPtrTime / directTime << "x)" << std::endl;
    std::cout << "std::function:    " << stdFuncTime << " μs ("
              << (float)stdFuncTime / directTime << "x)" << std::endl;
    std::cout << "Lambda:           " << lambdaTime << " μs ("
              << (float)lambdaTime / directTime << "x)" << std::endl;
    
    return 0;
}
```

### Example 2: Memory Overhead Comparison

```cpp
#include <iostream>
#include <functional>

int sampleFunction(int a, int b) {
    return a + b;
}

int main() {
    int (*funcPtr)(int, int) = sampleFunction;
    std::function<int(int, int)> stdFunc = sampleFunction;
    auto lambda = [](int a, int b) { return a + b; };
    
    std::cout << "Memory Size Comparison:" << std::endl;
    std::cout << "----------------------------------------" << std::endl;
    std::cout << "Function pointer:  " << sizeof(funcPtr) << " bytes" << std::endl;
    std::cout << "std::function:     " << sizeof(stdFunc) << " bytes" << std::endl;
    std::cout << "Lambda (no capture): " << sizeof(lambda) << " bytes" << std::endl;
    
    // Lambda with capture
    int x = 10;
    auto lambdaCapture = [x](int a, int b) { return a + b + x; };
    std::cout << "Lambda (with capture): " << sizeof(lambdaCapture) << " bytes" << std::endl;
    
    // std::function with lambda
    std::function<int(int, int)> stdFuncLambda = lambdaCapture;
    std::cout << "std::function (lambda): " << sizeof(stdFuncLambda) << " bytes" << std::endl;
    
    return 0;
}
```

### Example 3: Complex Performance Scenario

```cpp
#include <iostream>
#include <functional>
#include <chrono>
#include <vector>
#include <random>

class PerformanceTest {
public:
    static int compute(int a, int b, int c) {
        return (a * b) + (c % 100);
    }
    
    int memberCompute(int a, int b, int c) {
        return (a * b) - (c % 50);
    }
};

int main() {
    const int ITERATIONS = 5000000;
    PerformanceTest testObj;
    
    std::cout << "Complex Performance Test (" << ITERATIONS << " iterations)" << std::endl;
    std::cout << "============================================" << std::endl;
    
    // Test 1: Static member via function pointer
    auto start = std::chrono::high_resolution_clock::now();
    int (*staticPtr)(int, int, int) = &PerformanceTest::compute;
    volatile int result = 0;
    for (int i = 0; i < ITERATIONS; i++) {
        result = staticPtr(i, i + 1, i + 2);
    }
    auto end = std::chrono::high_resolution_clock::now();
    auto staticTime = std::chrono::duration_cast<std::chrono::microseconds>(end - start).count();
    
    // Test 2: Member function via std::function
    start = std::chrono::high_resolution_clock::now();
    std::function<int(int, int, int)> memberFunc = 
        std::bind(&PerformanceTest::memberCompute, &testObj, 
                  std::placeholders::_1, std::placeholders::_2, std::placeholders::_3);
    for (int i = 0; i < ITERATIONS; i++) {
        result = memberFunc(i, i + 1, i + 2);
    }
    end = std::chrono::high_resolution_clock::now();
    auto memberTime = std::chrono::duration_cast<std::chrono::microseconds>(end - start).count();
    
    // Test 3: Lambda with capture
    start = std::chrono::high_resolution_clock::now();
    int multiplier = 2;
    auto lambdaCapture = [multiplier](int a, int b, int c) {
        return (a * b * multiplier) + (c % 100);
    };
    for (int i = 0; i < ITERATIONS; i++) {
        result = lambdaCapture(i, i + 1, i + 2);
    }
    end = std::chrono::high_resolution_clock::now();
    auto lambdaTime = std::chrono::duration_cast<std::chrono::microseconds>(end - start).count();
    
    std::cout << "Static function pointer: " << staticTime << " μs" << std::endl;
    std::cout << "Member via std::function: " << memberTime << " μs" << std::endl;
    std::cout << "Lambda with capture: " << lambdaTime << " μs" << std::endl;
    
    std::cout << "\nRelative Performance:" << std::endl;
    std::cout << "Member function overhead: +" 
              << ((float)memberTime / staticTime - 1) * 100 << "%" << std::endl;
    std::cout << "Lambda overhead: +" 
              << ((float)lambdaTime / staticTime - 1) * 100 << "%" << std::endl;
    
    return 0;
}
```

### Performance Summary

| Method | Speed | Memory | Flexibility | Use Case |
|--------|-------|--------|-------------|----------|
| **Function Pointer** | Fastest | Smallest | Limited | Performance-critical, simple callbacks |
| **std::function** | Slower | Larger | High | General-purpose, complex scenarios |
| **Lambda** | Fast (if not captured) | Variable | High | Modern C++, inline logic |
| **Direct Call** | Fastest | N/A | N/A | When function is known at compile time |

### When to Use What

**Use Function Pointers when:**
- Maximum performance is critical
- Simple callback mechanism needed
- Working with C APIs
- Memory overhead is a concern

**Use std::function when:**
- Need to store different callable types
- Working with lambdas and bind
- Flexibility is more important than raw performance
- Building plugin systems or event handlers

**Use Lambdas when:**
- Need inline function definitions
- Require closures (capturing variables)
- Modern C++ codebase
- Short, localized functionality

---

## Conclusion

Function pointers are a powerful feature in C++ that enable:
- Dynamic function selection
- Callback mechanisms
- Plugin architectures
- Event-driven programming

While `std::function` and lambdas provide more modern alternatives with additional flexibility, traditional function pointers still have their place in performance-critical code and when interfacing with C libraries.

### Best Practices

1. **Prefer modern alternatives** (`std::function`, lambdas) unless performance is critical
2. **Use typedef/using** for cleaner function pointer syntax
3. **Consider type safety** - templates can help
4. **Profile before optimizing** - measure actual performance impact
5. **Document function pointer parameters** clearly
6. **Be careful with member function pointers** - they're more complex
7. **Use nullptr** instead of NULL for function pointers in modern C++

### Further Reading

- C++ Standard Library documentation on `<functional>`
- Effective Modern C++ by Scott Meyers
- C++ Concurrency in Action by Anthony Williams
- Design Patterns (Gang of Four) - Strategy, Command patterns

---

*Document created for senior software engineers working with automotive embedded systems and C++ development.*
