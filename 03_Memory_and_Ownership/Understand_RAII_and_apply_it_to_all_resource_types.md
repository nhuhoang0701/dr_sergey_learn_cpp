# Understand RAII and Apply It to All Resource Types

**Category:** Memory & Ownership  
**Item:** #29  
**Standard:** C++17  
**Reference:** <https://en.cppreference.com/w/cpp/language/raii>  

---

## Topic Overview

### What Is RAII

**RAII (Resource Acquisition Is Initialization)** is the foundational C++ idiom:

- **Constructor** acquires the resource
- **Destructor** releases the resource
- Resource lifetime is tied to object scope — guaranteed cleanup even on exceptions

### RAII Works for ALL Resources

| Resource Type | Acquire | Release | RAII Wrapper |
| --- | --- | --- | --- |
| Heap memory | `new` | `delete` | `unique_ptr`, `shared_ptr` |
| File handle | `fopen` | `fclose` | Custom wrapper or `unique_ptr<FILE, Closer>` |
| Mutex lock | `lock()` | `unlock()` | `lock_guard`, `unique_lock` |
| Network socket | `socket()` | `close()` | Custom wrapper |
| Database connection | `connect()` | `disconnect()` | Custom wrapper |
| OpenGL texture | `glGenTextures` | `glDeleteTextures` | Custom wrapper |
| Thread | `std::thread t(f)` | `t.join()` | `std::jthread` (C++20) |

### Why It Matters

Without RAII, you need manual cleanup at every exit point:

```cpp

// BAD: manual cleanup
void process() {
    FILE* f = fopen("data.txt", "r");
    int* buf = new int[100];
    
    if (error1) { delete[] buf; fclose(f); return; }  // cleanup
    if (error2) { delete[] buf; fclose(f); return; }  // cleanup again!
    if (error3) { delete[] buf; fclose(f); return; }  // and again!
    
    delete[] buf; fclose(f);  // normal exit cleanup
}

```

With RAII, cleanup is automatic:

```cpp

// GOOD: RAII — no manual cleanup needed
void process() {
    auto f = UniqueFile(fopen("data.txt", "r"));
    auto buf = std::make_unique<int[]>(100);
    
    if (error1) return;  // automatic cleanup
    if (error2) return;  // automatic cleanup
    if (error3) return;  // automatic cleanup
}   // automatic cleanup

```

---

## Self-Assessment

### Q1: Write an RAII wrapper for a C file handle (`FILE*`) that closes on destruction

```cpp

#include <iostream>
#include <cstdio>
#include <string>
#include <stdexcept>
#include <utility>

class File {
    FILE* handle_ = nullptr;

public:
    // Constructor acquires the resource
    File(const char* path, const char* mode) {
        handle_ = std::fopen(path, mode);
        if (!handle_) {
            throw std::runtime_error(std::string("Failed to open: ") + path);
        }
        std::cout << "  [File] Opened \"" << path << "\"\n";
    }

    // Destructor releases the resource
    ~File() {
        if (handle_) {
            std::fclose(handle_);
            std::cout << "  [File] Closed\n";
        }
    }

    // Move constructor (transfer ownership)
    File(File&& other) noexcept : handle_(other.handle_) {
        other.handle_ = nullptr;
    }

    // Move assignment
    File& operator=(File&& other) noexcept {
        if (this != &other) {
            if (handle_) std::fclose(handle_);
            handle_ = other.handle_;
            other.handle_ = nullptr;
        }
        return *this;
    }

    // No copying (file handle is unique)
    File(const File&) = delete;
    File& operator=(const File&) = delete;

    // Access
    FILE* get() const { return handle_; }
    explicit operator bool() const { return handle_ != nullptr; }

    // Convenience methods
    void write(const std::string& text) {
        std::fputs(text.c_str(), handle_);
    }

    std::string read_all() {
        std::string result;
        char buf[256];
        while (std::fgets(buf, sizeof(buf), handle_)) {
            result += buf;
        }
        return result;
    }
};

int main() {
    // RAII: file automatically closed at scope exit
    std::cout << "=== Simple RAII file usage ===\n";
    {
        File f("raii_test.txt", "w");
        f.write("Hello, RAII!\n");
        f.write("Automatic cleanup is beautiful.\n");
    }   // f.~File() called automatically — file closed

    // RAII: read it back
    {
        File f("raii_test.txt", "r");
        std::cout << "Contents:\n" << f.read_all();
    }   // closed automatically

    // RAII: exception safety
    std::cout << "\n=== Exception safety ===\n";
    try {
        File f("raii_test.txt", "r");
        // Simulate an exception
        throw std::runtime_error("Something failed!");
        // f.write("This never executes");
    } catch (const std::exception& e) {
        std::cout << "Caught: " << e.what() << "\n";
        // f is STILL properly closed! RAII guarantees it.
    }

    // Move semantics
    std::cout << "\n=== Move ownership ===\n";
    {
        File f1("raii_test.txt", "r");
        File f2 = std::move(f1);  // f1 gives up ownership
        std::cout << "f1 valid? " << (f1 ? "yes" : "no") << "\n";  // no
        std::cout << "f2 valid? " << (f2 ? "yes" : "no") << "\n";  // yes
    }   // only f2 closes the file

    std::remove("raii_test.txt");
    return 0;
}

```

### Q2: Explain why a destructor must never throw and how `std::terminate` protects the invariant

```cpp

#include <iostream>
#include <stdexcept>

// WHY destructors must not throw:
//
// 1. During stack unwinding (exception propagation), all local objects
//    are destroyed. If a destructor throws DURING unwinding, we have
//    two active exceptions. C++ calls std::terminate — game over.
//
// 2. Containers call destructors in loops. If one element's destructor
//    throws, remaining elements LEAK because the loop aborts.
//
// 3. RAII guarantee broken: if cleanup can fail, resources may leak.

struct BadResource {
    int id;
    BadResource(int i) : id(i) {
        std::cout << "  BadResource(" << id << ") acquired\n";
    }
    ~BadResource() noexcept(false) {   // DANGEROUS: destructor may throw
        std::cout << "  ~BadResource(" << id << ") releasing\n";
        if (id == 2) {
            throw std::runtime_error("destructor threw!");
            // If this happens during stack unwinding → std::terminate!
        }
    }
};

struct GoodResource {
    int id;
    GoodResource(int i) : id(i) {
        std::cout << "  GoodResource(" << id << ") acquired\n";
    }
    ~GoodResource() noexcept {  // CORRECT: noexcept (default since C++11)
        std::cout << "  ~GoodResource(" << id << ") releasing\n";
        // If cleanup can fail, swallow the error and log
        try {
            // risky_cleanup();
        } catch (...) {
            std::cerr << "  WARNING: cleanup failed for " << id << "\n";
            // Don't rethrow! Swallow it.
        }
    }
};

int main() {
    std::cout << "=== Good RAII (noexcept destructors) ===\n";
    try {
        GoodResource a(1);
        GoodResource b(2);
        throw std::runtime_error("exception!");
        // Stack unwinds: ~b() then ~a() called safely
    } catch (const std::exception& e) {
        std::cout << "Caught: " << e.what() << "\n";
    }

    std::cout << "\n=== Key rules ===\n";
    std::cout << "1. Destructors are implicitly noexcept since C++11\n";
    std::cout << "2. If a dtor throws during unwinding → std::terminate()\n";
    std::cout << "3. If cleanup can fail: catch internally, log, don't throw\n";
    std::cout << "4. Design resources so cleanup is infallible\n";

    return 0;
}

```

### Q3: Refactor a function with multiple early returns and manual cleanup into RAII-clean code

```cpp

#include <iostream>
#include <cstdio>
#include <memory>
#include <string>
#include <vector>
#include <cstdlib>

// ======== BEFORE: Manual cleanup nightmare ========
int process_bad(const char* filename) {
    FILE* f = fopen(filename, "r");
    if (!f) return -1;

    int* buffer = (int*)malloc(100 * sizeof(int));
    if (!buffer) {
        fclose(f);              // Cleanup point 1
        return -2;
    }

    char* name = (char*)malloc(256);
    if (!name) {
        free(buffer);           // Cleanup point 2
        fclose(f);
        return -3;
    }

    // Do work...
    // if (error) { free(name); free(buffer); fclose(f); return -4; }

    free(name);                 // Cleanup point 3
    free(buffer);
    fclose(f);
    return 0;
}

// ======== AFTER: Clean RAII version ========

// FILE* RAII wrapper
struct FileCloser {
    void operator()(FILE* f) const { if (f) fclose(f); }
};
using UniqueFile = std::unique_ptr<FILE, FileCloser>;

// malloc RAII wrapper
struct FreeDeleter {
    void operator()(void* p) const { free(p); }
};
template<typename T>
using MallocPtr = std::unique_ptr<T, FreeDeleter>;

int process_good(const char* filename) {
    // Each resource is wrapped — cleanup is automatic
    UniqueFile f(fopen(filename, "r"));
    if (!f) return -1;

    MallocPtr<int> buffer(static_cast<int*>(malloc(100 * sizeof(int))));
    if (!buffer) return -2;    // f automatically closed

    MallocPtr<char> name(static_cast<char*>(malloc(256)));
    if (!name) return -3;      // buffer freed, f closed automatically

    // Do work...
    // if (error) return -4;   // ALL resources cleaned up automatically

    return 0;
}   // ALL resources cleaned up in reverse order

// ======== Even better: use standard types ========
int process_best(const char* filename) {
    // No custom wrappers needed!
    UniqueFile f(fopen(filename, "r"));
    if (!f) return -1;

    std::vector<int> buffer(100);           // RAII vector
    std::string name(256, '\0');            // RAII string

    // if (error) return -4;   // automatic cleanup, always
    return 0;
}

int main() {
    std::cout << "=== RAII refactoring demo ===\n\n";

    // Create test file
    {
        UniqueFile f(fopen("raii_refactor_test.txt", "w"));
        fputs("test data\n", f.get());
    }

    int result;

    result = process_bad("raii_refactor_test.txt");
    std::cout << "process_bad:  " << result << "\n";

    result = process_good("raii_refactor_test.txt");
    std::cout << "process_good: " << result << "\n";

    result = process_best("raii_refactor_test.txt");
    std::cout << "process_best: " << result << "\n";

    std::remove("raii_refactor_test.txt");

    std::cout << "\n=== Benefits of RAII refactoring ===\n";
    std::cout << "1. No duplicate cleanup code\n";
    std::cout << "2. Exception-safe (cleanup on throw)\n";
    std::cout << "3. Can't forget to cleanup (compiler does it)\n";
    std::cout << "4. Reverse order destruction guaranteed\n";

    return 0;
}

```

---

## Notes

- **RAII is THE most important C++ idiom** — it makes resource management automatic and exception-safe.
- When you write a destructor, think "Rule of Five" — you probably also need copy/move operations.
- For C APIs, use `unique_ptr` with a custom deleter rather than writing a full wrapper class.
- `std::lock_guard`, `std::unique_lock`, `std::jthread`, `std::fstream` are all RAII wrappers.
- RAII + move semantics = zero-overhead resource transfer between scopes.
