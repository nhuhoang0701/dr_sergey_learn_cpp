# Implement the Pipeline pattern for data processing tasks

**Category:** Project Architecture

---

## Topic Overview

The **Pipeline pattern** chains processing stages where the output of one stage feeds the input of the next. Each stage runs independently and can be parallelized. Pipelines are ideal for ETL, image processing, network packet handling, and any sequential data transformation. In C++, pipelines can be implemented with function composition, coroutines, or thread-per-stage designs.

### Pipeline Variants

| Variant | Concurrency | Latency | Throughput | Use Case |
| --- | --- | --- | --- | --- |
| **Synchronous** | None | Low (single item) | Low | Simple transforms |
| **Thread-per-stage** | High | Higher (queue overhead) | High | Streaming data |
| **Coroutine-based** | Cooperative | Low | Medium | Lazy evaluation |
| **SIMD/batch** | Data-parallel | Low per batch | Highest | Image/signal processing |

---

## Self-Assessment

### Q1: Implement a type-safe synchronous pipeline

**Answer:**

```cpp

#include <functional>
#include <vector>
#include <string>
#include <iostream>

// === Pipeline with type-safe chaining ===
template<typename In, typename Out>
class Stage {
public:
    using Fn = std::function<Out(In)>;
    explicit Stage(Fn fn) : fn_(std::move(fn)) {}

    Out process(In input) const { return fn_(input); }

    // Chain: Stage<A,B> | Stage<B,C> -> Stage<A,C>
    template<typename Next>
    auto then(Stage<Out, Next> next) const {
        auto current = fn_;
        return Stage<In, Next>([current, next](In input) {
            return next.process(current(input));
        });
    }

private:
    Fn fn_;
};

// Convenience: make_stage
template<typename In, typename Out>
auto make_stage(std::function<Out(In)> fn) {
    return Stage<In, Out>(std::move(fn));
}

// === Example: Text processing pipeline ===
auto pipeline =
    Stage<std::string, std::string>([](std::string s) {
        // Stage 1: Trim whitespace
        auto start = s.find_first_not_of(" \t\n");
        auto end = s.find_last_not_of(" \t\n");
        return s.substr(start, end - start + 1);
    })
    .then(Stage<std::string, std::string>([](std::string s) {
        // Stage 2: Lowercase
        std::transform(s.begin(), s.end(), s.begin(), ::tolower);
        return s;
    }))
    .then(Stage<std::string, std::vector<std::string>>([](std::string s) {
        // Stage 3: Tokenize
        std::vector<std::string> tokens;
        std::istringstream iss(s);
        std::string word;
        while (iss >> word) tokens.push_back(word);
        return tokens;
    }))
    .then(Stage<std::vector<std::string>, int>([](std::vector<std::string> tokens) {
        // Stage 4: Count
        return static_cast<int>(tokens.size());
    }));

// Usage:
// int count = pipeline.process("  Hello World  FOO  ");
// count == 3

```

### Q2: Build a concurrent thread-per-stage pipeline

**Answer:**

```cpp

#include <queue>
#include <thread>
#include <mutex>
#include <condition_variable>
#include <optional>
#include <functional>

// === Bounded blocking queue ===
template<typename T>
class BoundedQueue {
public:
    explicit BoundedQueue(size_t capacity) : capacity_(capacity) {}

    void push(T item) {
        std::unique_lock lock(mutex_);
        cv_full_.wait(lock, [this] {
            return queue_.size() < capacity_ || done_;
        });
        if (done_) return;
        queue_.push(std::move(item));
        cv_empty_.notify_one();
    }

    std::optional<T> pop() {
        std::unique_lock lock(mutex_);
        cv_empty_.wait(lock, [this] {
            return !queue_.empty() || done_;
        });
        if (queue_.empty()) return std::nullopt;
        T item = std::move(queue_.front());
        queue_.pop();
        cv_full_.notify_one();
        return item;
    }

    void close() {
        std::lock_guard lock(mutex_);
        done_ = true;
        cv_empty_.notify_all();
        cv_full_.notify_all();
    }

private:
    std::queue<T> queue_;
    size_t capacity_;
    std::mutex mutex_;
    std::condition_variable cv_empty_, cv_full_;
    bool done_ = false;
};

// === Concurrent pipeline ===
template<typename T>
class ConcurrentPipeline {
public:
    using StageFunc = std::function<T(T)>;

    void add_stage(StageFunc fn, size_t queue_size = 64) {
        stages_.push_back(std::move(fn));
        // One queue between each pair of stages
        if (queues_.size() < stages_.size())
            queues_.push_back(
                std::make_shared<BoundedQueue<T>>(queue_size));
    }

    void start() {
        // Add output queue
        queues_.push_back(std::make_shared<BoundedQueue<T>>(64));

        for (size_t i = 0; i < stages_.size(); ++i) {
            threads_.emplace_back([this, i] {
                auto& in_q = *queues_[i];
                auto& out_q = *queues_[i + 1];
                while (auto item = in_q.pop()) {
                    out_q.push(stages_[i](std::move(*item)));
                }
                out_q.close();
            });
        }
    }

    void push(T item) { queues_.front()->push(std::move(item)); }
    std::optional<T> pop() { return queues_.back()->pop(); }

    void finish() {
        queues_.front()->close();
        for (auto& t : threads_) t.join();
    }

private:
    std::vector<StageFunc> stages_;
    std::vector<std::shared_ptr<BoundedQueue<T>>> queues_;
    std::vector<std::jthread> threads_;
};

// Usage:
// ConcurrentPipeline<ImageData> pipe;
// pipe.add_stage(decode);    // Thread 1: decode
// pipe.add_stage(resize);    // Thread 2: resize
// pipe.add_stage(compress);  // Thread 3: compress
// pipe.start();
// for (auto& file : files) pipe.push(load(file));
// pipe.finish();

```

### Q3: C++20 coroutine-based lazy pipeline

**Answer:**

```cpp

#include <coroutine>
#include <ranges>

// === Generator for lazy pipeline stages ===
template<typename T>
class Generator {
public:
    struct promise_type {
        T value;
        Generator get_return_object() {
            return {std::coroutine_handle<promise_type>::from_promise(*this)};
        }
        std::suspend_always initial_suspend() { return {}; }
        std::suspend_always final_suspend() noexcept { return {}; }
        std::suspend_always yield_value(T v) {
            value = std::move(v);
            return {};
        }
        void return_void() {}
        void unhandled_exception() { std::terminate(); }
    };

    bool next() {
        handle_.resume();
        return !handle_.done();
    }
    T value() const { return handle_.promise().value; }

    ~Generator() { if (handle_) handle_.destroy(); }

private:
    explicit Generator(std::coroutine_handle<promise_type> h) : handle_(h) {}
    std::coroutine_handle<promise_type> handle_;
};

// === Pipeline stages as coroutines ===
Generator<int> read_numbers(const std::vector<int>& data) {
    for (int n : data)
        co_yield n;
}

Generator<int> filter_even(Generator<int>& source) {
    while (source.next()) {
        int val = source.value();
        if (val % 2 == 0)
            co_yield val;
    }
}

Generator<int> multiply(Generator<int>& source, int factor) {
    while (source.next())
        co_yield source.value() * factor;
}

// === Usage: lazy, pulls on demand ===
void process() {
    std::vector<int> data = {1, 2, 3, 4, 5, 6, 7, 8};
    auto stage1 = read_numbers(data);
    auto stage2 = filter_even(stage1);
    auto stage3 = multiply(stage2, 10);

    while (stage3.next()) {
        std::cout << stage3.value() << " ";  // 20 40 60 80
    }
}

// Or with C++20 ranges (similar concept):
// auto result = data
//     | std::views::filter([](int n) { return n % 2 == 0; })
//     | std::views::transform([](int n) { return n * 10; });

```

---

## Notes

- **Synchronous pipelines**: compose functions, ideal when processing is fast
- **Thread-per-stage**: best when stages have different speeds — bounded queues provide backpressure
- **Coroutine pipelines**: lazy evaluation, no extra threads, pull-based
- **C++20 ranges**: the standard library's built-in pipeline mechanism
- Bounded queues prevent fast producers from overwhelming slow consumers
- Monitor queue sizes at runtime to identify pipeline bottlenecks
- For batch processing, accumulate items and process in bulk for better cache utilization
