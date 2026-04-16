# Know out-of-line allocation strategies for coroutine frames

**Category:** Memory and Ownership  
**Standard:** C++20  
**Reference:** <https://en.cppreference.com/w/cpp/language/coroutines>  

---

## Topic Overview

Coroutine frames are heap-allocated by default. For performance-critical code, you can control this allocation via the promise type's `operator new`/`operator delete`.

### Default Allocation

```cpp

#include <coroutine>

// By default, the compiler calls ::operator new to allocate the coroutine frame:
// Frame contains: promise, parameters, local variables, suspension point index
// Size is determined by the compiler

struct Task {
    struct promise_type {
        Task get_return_object() { return {}; }
        std::suspend_always initial_suspend() { return {}; }
        std::suspend_always final_suspend() noexcept { return {}; }
        void return_void() {}
        void unhandled_exception() {}
        // Compiler calls ::operator new(frame_size) for the frame
    };
};

```

### Custom Allocation via Promise

```cpp

#include <coroutine>
#include <cstddef>
#include <iostream>

// Pool allocator for coroutine frames
class FramePool {
    static constexpr size_t BLOCK_SIZE = 4096;
    static constexpr size_t MAX_BLOCKS = 1024;
    static inline void* blocks_[MAX_BLOCKS];
    static inline size_t count_ = 0;
public:
    static void* allocate(size_t size) {
        if (count_ > 0 && size <= BLOCK_SIZE)
            return blocks_[--count_];
        return ::operator new(size);
    }
    static void deallocate(void* ptr, size_t size) {
        if (count_ < MAX_BLOCKS && size <= BLOCK_SIZE)
            blocks_[count_++] = ptr;
        else
            ::operator delete(ptr, size);
    }
};

struct PooledTask {
    struct promise_type {
        // Custom allocation — called instead of ::operator new
        void* operator new(size_t size) {
            return FramePool::allocate(size);
        }
        void operator delete(void* ptr, size_t size) {
            FramePool::deallocate(ptr, size);
        }
        PooledTask get_return_object() { return {}; }
        std::suspend_always initial_suspend() { return {}; }
        std::suspend_always final_suspend() noexcept { return {}; }
        void return_void() {}
        void unhandled_exception() {}
    };
};

```

### Heap Allocation Elision (HALO)

```cpp

// The compiler MAY elide the heap allocation if:
// 1. The coroutine's lifetime is bounded by the caller
// 2. The frame size is known at the call site
// 3. The coroutine doesn't escape the caller

Task simple_coroutine() {  // May be stack-allocated (HALO)
    co_return;
}

void caller() {
    auto t = simple_coroutine();  // Compiler can put frame on caller's stack
    // t doesn't escape this scope → HALO eligible
}

```

---

## Self-Assessment

### Q1: When does HALO (Heap Allocation eLision Optimization) apply

HALO applies when the compiler can prove the coroutine frame's lifetime is bounded by the caller's scope. The coroutine handle must not escape (stored in container, passed to another thread). In practice, HALO is limited and only Clang implements it.

### Q2: How to pass allocator arguments to coroutine frame allocation

```cpp

struct AllocTask {
    struct promise_type {
        // If the first coroutine parameter is an allocator, the compiler passes it:
        template<typename Alloc, typename... Args>
        void* operator new(size_t size, Alloc& alloc, Args&&...) {
            return alloc.allocate(size);
        }
    };
};

AllocTask my_coro(MyAllocator& alloc, int param) {
    co_return;  // alloc is forwarded to promise_type::operator new
}

```

### Q3: What is the `get_return_object_on_allocation_failure` pattern

```cpp

struct NullableTask {
    struct promise_type {
        static NullableTask get_return_object_on_allocation_failure() {
            return NullableTask{nullptr};  // Return null task instead of throwing
        }
        void* operator new(size_t sz) noexcept { return ::malloc(sz); }
        void operator delete(void* p) { ::free(p); }
    };
};
// If operator new returns nullptr, get_return_object_on_allocation_failure is called
// instead of throwing std::bad_alloc

```

---

## Notes

- Custom `operator new`/`delete` in the promise type controls frame allocation.
- HALO is the ideal optimization but is limited in practice (only simple cases, Clang only).
- For latency-critical code, use pool allocators for coroutine frames.
- `get_return_object_on_allocation_failure` enables non-throwing coroutine creation.
