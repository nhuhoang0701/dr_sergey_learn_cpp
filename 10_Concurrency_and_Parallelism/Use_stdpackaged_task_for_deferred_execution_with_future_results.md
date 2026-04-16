# Use std::packaged_task for deferred execution with future results

**Category:** Concurrency & Parallelism  
**Item:** #373  
**Standard:** C++11  
**Reference:** <https://en.cppreference.com/w/cpp/thread/packaged_task>  

---

## Topic Overview

`std::packaged_task<R(Args...)>` wraps a callable and connects it to a `std::future<R>`. Unlike `std::async`, you control **when and where** the task executes.

### How It Works

```cpp

1. Create:     packaged_task<int(int)> task(compute);
2. Get future: auto fut = task.get_future();
3. Execute:    task(42);         // can be on any thread, at any time
4. Retrieve:   int r = fut.get(); // blocks until task completes

```

### async vs packaged_task vs promise

```cpp

┌──────────────┬────────────────────────────────────────┐
│ std::async   │ Creates task + runs it immediately     │
│              │ (either on new thread or deferred)      │
│              │ Most convenient, least control          │
├──────────────┼────────────────────────────────────────┤
│ packaged_task│ Creates task, YOU decide when/where     │
│              │ to run it. Perfect for thread pools.    │
│              │ medium convenience, medium control      │
├──────────────┼────────────────────────────────────────┤
│ promise      │ YOU manually set the value/exception.   │
│              │ No callable — raw control.              │
│              │ Least convenient, most control          │
└──────────────┴────────────────────────────────────────┘

```

---

## Self-Assessment

### Q1: Wrap a callable in std::packaged_task and retrieve the result via its future after execution

**Answer:**

```cpp

#include <future>
#include <thread>
#include <iostream>
#include <functional>

int compute(int a, int b) {
    return a * a + b * b;
}

int main() {
    // === Basic: wrap a function ===
    std::packaged_task<int(int, int)> task(compute);
    auto fut = task.get_future();

    // Task hasn't run yet! We have full control.
    std::cout << "Task created, not yet executed\n";

    // Execute on current thread:
    task(3, 4); // calls compute(3, 4), stores result in shared state

    std::cout << "Result: " << fut.get() << "\n"; // 25

    // === Execute on another thread ===
    std::packaged_task<int(int, int)> task2(compute);
    auto fut2 = task2.get_future();

    // packaged_task is move-only (no copy)
    std::thread t(std::move(task2), 5, 12);

    std::cout << "Task2 running on thread...\n";
    std::cout << "Result2: " << fut2.get() << "\n"; // 169
    t.join();

    // === Wrap a lambda ===
    std::packaged_task<std::string()> task3([] {
        return std::string("Hello from packaged_task!");
    });
    auto fut3 = task3.get_future();
    task3();
    std::cout << fut3.get() << "\n";

    // Output:
    // Task created, not yet executed
    // Result: 25
    // Task2 running on thread...
    // Result2: 169
    // Hello from packaged_task!
}

```

### Q2: Post a packaged_task to a thread pool and collect the future for later retrieval

**Answer:**

```cpp

#include <future>
#include <thread>
#include <queue>
#include <mutex>
#include <condition_variable>
#include <functional>
#include <iostream>
#include <vector>

// Simple thread pool using packaged_task
class ThreadPool {
    std::vector<std::jthread> workers_;
    std::queue<std::function<void()>> tasks_;
    std::mutex mtx_;
    std::condition_variable cv_;
    bool stop_ = false;

public:
    explicit ThreadPool(size_t n_threads) {
        for (size_t i = 0; i < n_threads; ++i) {
            workers_.emplace_back([this](std::stop_token) {
                while (true) {
                    std::function<void()> task;
                    {
                        std::unique_lock lock(mtx_);
                        cv_.wait(lock, [&] { return stop_ || !tasks_.empty(); });
                        if (stop_ && tasks_.empty()) return;
                        task = std::move(tasks_.front());
                        tasks_.pop();
                    }
                    task();
                }
            });
        }
    }

    // Submit work, get a future back
    template<typename F, typename... Args>
    auto submit(F&& f, Args&&... args) -> std::future<decltype(f(args...))> {
        using R = decltype(f(args...));

        auto task = std::make_shared<std::packaged_task<R()>>(
            [f = std::forward<F>(f), ...args = std::forward<Args>(args)]() mutable {
                return f(args...);
            });

        auto fut = task->get_future();

        {
            std::lock_guard lock(mtx_);
            tasks_.push([task] { (*task)(); });
            // task is wrapped in shared_ptr because std::function requires copyable
            // but packaged_task is move-only
        }
        cv_.notify_one();
        return fut;
    }

    ~ThreadPool() {
        {
            std::lock_guard lock(mtx_);
            stop_ = true;
        }
        cv_.notify_all();
        // jthreads auto-join in destructor
    }
};

int main() {
    ThreadPool pool(4);

    // Submit tasks and collect futures
    std::vector<std::future<int>> futures;
    for (int i = 0; i < 10; ++i) {
        futures.push_back(pool.submit([](int x) {
            std::this_thread::sleep_for(std::chrono::milliseconds(20));
            return x * x;
        }, i));
    }

    // Collect results
    for (int i = 0; i < 10; ++i) {
        std::cout << i << "² = " << futures[i].get() << "\n";
    }

    // Output:
    // 0² = 0
    // 1² = 1
    // 2² = 4
    // 3² = 9
    // 4² = 16
    // 5² = 25
    // 6² = 36
    // 7² = 49
    // 8² = 64
    // 9² = 81
}

```

**Key insight:** `packaged_task` is the bridge between a thread pool and `std::future`. The pool stores tasks in a queue; each task is a `packaged_task` that automatically fills the associated future when executed.

### Q3: Explain the difference between packaged_task and std::async in terms of execution control

**Answer:**

```cpp

EXECUTION CONTROL COMPARISON
═════════════════════════════

                    std::async                  std::packaged_task
────────────────────────────────────────────────────────────────────
When it runs:       Immediately (async) or      When YOU invoke operator()
                    on get() (deferred)         
Where it runs:      New thread or caller thread  Any thread you choose
Thread pool:        NO (creates new threads)     YES (queue + reuse threads)
Deferred execution: Only deferred policy         Natural — just delay the call
Exception:          Stored in future             Stored in future
Destructor blocks:  YES (async future joins)     NO
Move-only:          N/A (returns future)         YES (must std::move)

```

```cpp

#include <future>
#include <thread>
#include <iostream>
#include <chrono>

using namespace std::chrono_literals;

int compute() {
    std::this_thread::sleep_for(100ms);
    return 42;
}

int main() {
    // === std::async: immediate execution, no control ===
    {
        auto start = std::chrono::steady_clock::now();
        auto fut = std::async(std::launch::async, compute);
        //                                        ^^^^^^^ runs NOW on new thread

        // Discarding the future BLOCKS! (destructor joins the thread)
        // std::async(std::launch::async, compute); // ← blocks here!

        auto result = fut.get();
        auto ms = std::chrono::duration_cast<std::chrono::milliseconds>(
            std::chrono::steady_clock::now() - start);
        std::cout << "async: " << result << " in " << ms.count() << "ms\n";
    }

    // === packaged_task: deferred execution, full control ===
    {
        auto start = std::chrono::steady_clock::now();
        std::packaged_task<int()> task(compute);
        auto fut = task.get_future();

        // Task NOT running yet — we control when it starts
        std::this_thread::sleep_for(200ms);

        // Run it now, on a specific thread
        std::thread t(std::move(task));
        t.join();

        auto result = fut.get();
        auto ms = std::chrono::duration_cast<std::chrono::milliseconds>(
            std::chrono::steady_clock::now() - start);
        std::cout << "packaged_task: " << result << " in " << ms.count() << "ms\n";
        // ~300ms (200ms delay + 100ms compute)
    }

    // === packaged_task: reset() to reuse ===
    {
        std::packaged_task<int()> task(compute);
        auto fut1 = task.get_future();
        task(); // first execution
        std::cout << "First: " << fut1.get() << "\n";

        task.reset(); // reset shared state — can get a new future
        auto fut2 = task.get_future();
        task(); // second execution
        std::cout << "Second: " << fut2.get() << "\n";
    }

    // Output:
    // async: 42 in 100ms
    // packaged_task: 42 in 300ms
    // First: 42
    // Second: 42
}

```

---

## Notes

- **Move-only:** `packaged_task` cannot be copied. Use `std::move` to pass into threads or queues. For `std::function` (which requires copyability), wrap in `shared_ptr`.
- **`reset()`:** Resets the shared state, allowing the task to be invoked again with a new future. The old future becomes broken (throws `broken_promise` if `.get()` not yet called).
- **Exception handling:** If the wrapped callable throws, the exception is stored in the shared state and rethrown on `fut.get()`.
- **`void` return:** `packaged_task<void(int)>` works for tasks with no return value. The future is `future<void>`.
- **Thread pool essential:** `packaged_task` is the standard building block for thread pools — `std::async` can't be queued or deferred.
- Compile with `-std=c++20 -O2 -pthread`.
