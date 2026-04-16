# Design callback and extension interfaces

**Category:** API & Library Design  
**Standard:** C++17/20  
**Reference:** <https://en.cppreference.com/w/cpp/utility/functional/function>  

---

## Topic Overview

Libraries need to let users inject behavior. Three approaches exist with different tradeoffs.

### Comparison

| Approach | Overhead | Inlinable | Copyable | Type-erased |
| --- | --- | --- | --- | --- |
| `std::function` | Heap alloc possible | No | Yes | Yes |
| Template parameter | Zero | Yes | Depends | No |
| Type-erased interface | Virtual call | No | Depends | Yes |

### std::function — Simple, Dynamic

```cpp

#include <functional>
#include <iostream>
#include <vector>

class EventSystem {
    std::vector<std::function<void(int)>> handlers_;
public:
    void on_event(std::function<void(int)> handler) {
        handlers_.push_back(std::move(handler));
    }
    void fire(int event_id) {
        for (auto& h : handlers_) h(event_id);
    }
};

// Usage:
EventSystem sys;
sys.on_event([](int id) { std::cout << "Event: " << id << "\n"; });
sys.fire(42);

```

### Template — Zero Overhead, Compile-Time

```cpp

#include <concepts>
#include <iostream>

template<typename Handler>
requires std::invocable<Handler, int>
class EventProcessor {
    Handler handler_;
public:
    explicit EventProcessor(Handler h) : handler_(std::move(h)) {}
    void process(int event) { handler_(event); }
};

// Usage — handler is inlined
auto proc = EventProcessor([](int id) { /* inlined */ });
proc.process(42);

```

### Type Erasure — Runtime Polymorphism Without Inheritance

```cpp

#include <memory>
#include <iostream>

class AnyHandler {
    struct Concept {
        virtual void call(int) = 0;
        virtual ~Concept() = default;
    };
    template<typename T>
    struct Model : Concept {
        T handler;
        Model(T h) : handler(std::move(h)) {}
        void call(int v) override { handler(v); }
    };
    std::unique_ptr<Concept> impl_;
public:
    template<typename T>
    AnyHandler(T handler) : impl_(std::make_unique<Model<T>>(std::move(handler))) {}
    void operator()(int v) { impl_->call(v); }
};

```

---

## Self-Assessment

### Q1: When to choose each approach

- **`std::function`**: when callbacks are stored, heterogeneous callable types
- **Template**: hot paths, library internals, when inlining matters
- **Type erasure**: when you need runtime polymorphism without forcing inheritance on users

### Q2: What is `std::move_only_function` (C++23) and when to prefer it

`std::move_only_function` doesn't require copyability — works with move-only callables (like those capturing `unique_ptr`). Use it when callbacks own resources and don't need to be copied.

### Q3: Show the performance difference

```cpp

std::function call:     ~5-10 ns (indirect call + possible heap)
Template/inlined call:  ~0-1 ns (direct call, often inlined)
Virtual type erasure:   ~3-5 ns (indirect call, no heap per call)

```

---

## Notes

- `std::function` can allocate on the heap for large captures — use `std::function_ref` (C++26) for non-owning references.
- Templates are best for library-internal callbacks; `std::function` for stored callbacks in APIs.
- `std::move_only_function` (C++23) replaces `std::function` when copyability isn't needed.
