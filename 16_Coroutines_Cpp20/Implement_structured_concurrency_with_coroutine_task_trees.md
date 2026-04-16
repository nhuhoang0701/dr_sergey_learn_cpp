# Implement structured concurrency with coroutine task trees

**Category:** Coroutines (C++20)  
**Standard:** C++20  
**Reference:** <https://en.cppreference.com/w/cpp/language/coroutines>  

---

## Topic Overview

### What Is Structured Concurrency

**Structured concurrency** ensures that concurrent tasks follow the same scoping rules as structured programming: a parent task does not complete until all its child tasks have completed (or been cancelled). This prevents:

- Fire-and-forget tasks that leak resources.
- Dangling references from parent scope to child tasks.
- Unhandled exceptions in detached tasks.

### The Problem with Unstructured Concurrency

```cpp

// Unstructured: parent returns before child completes
task<void> parent() {
    auto child = spawn(some_work());  // fire and forget
    co_return;  // parent done — but child is still running!
    // Who owns child? Who catches its exceptions?
}

```

### Structured Concurrency with Task Trees

In a structured model, the parent **awaits** all children:

```cpp

task<void> parent() {
    // when_all waits for ALL children before parent continues
    co_await when_all(
        child_task_1(),
        child_task_2(),
        child_task_3()
    );
    // Here: all children are done, exceptions propagated
    co_return;
}

```

### Implementing a Basic Task Nursery / Scope

```cpp

#include <coroutine>
#include <vector>
#include <exception>
#include <iostream>

// Forward declarations (simplified)
template<typename T> struct task;

struct task_scope {
    struct scope_promise;
    using handle_type = std::coroutine_handle<scope_promise>;

    std::vector<std::coroutine_handle<>> children_;
    std::exception_ptr first_error_;

    // Spawn a child task into this scope
    template<typename T>
    void spawn(task<T> child) {
        children_.push_back(child.release_handle());
    }

    // Wait for all children — called by the parent coroutine
    auto join() {
        struct join_awaiter {
            task_scope& scope;

            bool await_ready() { return scope.children_.empty(); }

            void await_suspend(std::coroutine_handle<> parent) {
                // Resume each child, set continuation to parent
                for (auto& child : scope.children_) {
                    child.resume();
                }
            }

            void await_resume() {
                if (scope.first_error_)
                    std::rethrow_exception(scope.first_error_);
            }
        };
        return join_awaiter{*this};
    }
};

```

### Cancellation in Structured Concurrency

When one child fails, cancel siblings:

```cpp

task<void> robust_parent() {
    auto scope = make_task_scope();

    scope.spawn(download_file("a.txt"));
    scope.spawn(download_file("b.txt"));
    scope.spawn(download_file("c.txt"));

    try {
        co_await scope.join();
        // All downloads succeeded
    } catch (const std::exception& e) {
        // One child failed — scope automatically cancelled others
        std::cerr << "Download failed: " << e.what() << "\n";
        // All children are guaranteed completed/cancelled here
    }
    // Scope destructor verifies all children are done
}

```

### Library Support

| Library | Structured Concurrency Primitive |
| --- | --- |
| folly::coro | `AsyncScope`, `collectAll`, `collectAny` |
| libunifex | `when_all`, `let_value`, `async_scope` |
| stdexec | `when_all`, `async_scope` (P2300) |
| cppcoro | `when_all` (basic) |

### Example with cppcoro-style `when_all`

```cpp

task<void> process_batch(std::vector<std::string> urls) {
    // Structured: parent waits for ALL children
    std::vector<task<Response>> tasks;
    for (auto& url : urls) {
        tasks.push_back(fetch(url));
    }

    // when_all ensures all tasks complete before we continue
    auto results = co_await when_all(std::move(tasks));

    for (auto& response : results) {
        process(response);
    }
    // All tasks done — no leaks, no dangling references
}

```

---

## Self-Assessment

### Q1: Explain why structured concurrency prevents resource leaks

In structured concurrency, every spawned task is **owned by a scope** that guarantees completion before the scope exits. This means:

1. **No orphaned tasks** — every task has a parent that waits for it.
2. **Exception propagation** — child exceptions are caught by the parent scope.
3. **Deterministic cleanup** — resources are released in reverse order of acquisition.
4. **No "fire and forget"** — you cannot accidentally ignore a task's result.

```cpp

task<void> no_leak() {
    auto scope = make_scope();
    scope.spawn(open_connection());     // child 1
    scope.spawn(allocate_resource());   // child 2
    co_await scope.join();              // both done — resources cleaned up
}

```

### Q2: Implement `when_all` for two tasks

```cpp

template<typename T, typename U>
task<std::pair<T, U>> when_all(task<T> a, task<U> b) {
    // Start both tasks
    auto handle_a = a.start();
    auto handle_b = b.start();

    // Await both (simplified — real impl uses continuations)
    T result_a = co_await handle_a;
    U result_b = co_await handle_b;

    co_return std::make_pair(std::move(result_a), std::move(result_b));
}

// Usage:
task<void> example() {
    auto [count, name] = co_await when_all(
        get_count(),     // task<int>
        get_name()       // task<std::string>
    );
    std::cout << name << ": " << count << "\n";
}

```

### Q3: How does cancellation work in a task tree

```cpp

Parent
├── Child A (running)
├── Child B (FAILED with exception)
└── Child C (running)

```

When **Child B** fails:

1. The scope catches the exception.
2. The scope sends a **cancellation request** (via stop tokens) to Child A and Child C.
3. Child A and Child C check the stop token at their next `co_await` and exit early.
4. The scope waits for A and C to acknowledge cancellation.
5. The scope re-throws Child B's exception to the parent.

---

## Notes

- Structured concurrency is a core principle of P2300 (`std::execution`).
- Think of task scopes like lexical scopes: child tasks cannot outlive their parent scope.
- Always prefer `when_all` over manual spawn-and-forget patterns.
- C++26's `std::execution` will provide a standard `async_scope` for structured concurrency.
