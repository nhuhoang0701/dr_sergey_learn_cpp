# Implement the Observer pattern with modern C++ (signals and slots)

**Category:** OOP Design

---

## Topic Overview

The **Observer pattern** decouples event sources (subjects) from event handlers (observers). Modern C++ replaces the classic GoF approach with:

| Approach | Pros | Cons |
| --- | --- | --- |
| Virtual interface | Type-safe, clear contract | Coupling, inheritance required |
| `std::function` slots | Flexible, any callable | Lifetime management needed |
| Signal/slot library | Auto-disconnect, thread-safe | External dependency |
| `std::weak_ptr` observers | Auto-cleanup on destroy | Shared ownership required |

---

## Self-Assessment

### Q1: Implement a modern signal/slot system with auto-disconnect

**Answer:**

```cpp

#include <functional>
#include <vector>
#include <algorithm>
#include <memory>
#include <iostream>

// Connection handle for disconnect
class Connection {
    std::shared_ptr<bool> alive_;
public:
    Connection() : alive_(std::make_shared<bool>(true)) {}
    void disconnect() { *alive_ = false; }
    bool connected() const { return *alive_; }
    std::weak_ptr<bool> token() const { return alive_; }
};

// Type-safe signal with auto-cleanup
template<typename... Args>
class Signal {
    struct Slot {
        std::function<void(Args...)> callback;
        std::weak_ptr<bool> token;
    };
    std::vector<Slot> slots_;

public:
    Connection connect(std::function<void(Args...)> cb) {
        Connection conn;
        slots_.push_back({std::move(cb), conn.token()});
        return conn;
    }

    void emit(Args... args) {
        // Remove dead connections and invoke live ones
        slots_.erase(
            std::remove_if(slots_.begin(), slots_.end(),
                [](const Slot& s) { return s.token.expired(); }),
            slots_.end());

        for (auto& slot : slots_) {
            if (auto lock = slot.token.lock()) {
                slot.callback(args...);
            }
        }
    }

    size_t slot_count() const { return slots_.size(); }
};

// Example: button click system
class Button {
public:
    Signal<int, int> on_click;      // x, y
    Signal<> on_hover;

    void click(int x, int y) { on_click.emit(x, y); }
    void hover() { on_hover.emit(); }
};

int main() {
    Button btn;
    auto c1 = btn.on_click.connect([](int x, int y) {
        std::cout << "Clicked at (" << x << ", " << y << ")\n";
    });

    auto c2 = btn.on_click.connect([](int x, int y) {
        std::cout << "Logger: click (" << x << ", " << y << ")\n";
    });

    btn.click(10, 20);  // Both handlers fire

    c1.disconnect();     // Remove first handler
    btn.click(30, 40);   // Only logger fires
    return 0;
}

```

### Q2: Show an observer that auto-unregisters when destroyed

**Answer:**

```cpp

#include <functional>
#include <vector>
#include <memory>
#include <iostream>
#include <unordered_map>

template<typename... Args>
class Event {
    struct Observer {
        std::weak_ptr<void> lifetime;  // Observer's lifetime token
        std::function<void(Args...)> handler;
    };
    std::vector<Observer> observers_;
public:
    // Observer passes shared_ptr to itself as lifetime token
    template<typename T>
    void subscribe(std::shared_ptr<T> observer,
                   std::function<void(Args...)> handler) {
        observers_.push_back({observer, std::move(handler)});
    }

    void notify(Args... args) {
        // Auto-remove dead observers
        observers_.erase(
            std::remove_if(observers_.begin(), observers_.end(),
                [](const Observer& o) { return o.lifetime.expired(); }),
            observers_.end());
        for (auto& o : observers_)
            o.handler(args...);
    }
};

// Model with change notification
class DataModel {
public:
    Event<const std::string&, int> on_change;  // key, new_value

    void set(const std::string& key, int value) {
        data_[key] = value;
        on_change.notify(key, value);
    }
private:
    std::unordered_map<std::string, int> data_;
};

class DashboardWidget : public std::enable_shared_from_this<DashboardWidget> {
public:
    void bind(DataModel& model) {
        model.on_change.subscribe(
            shared_from_this(),
            [this](const std::string& key, int val) {
                std::cout << "Dashboard: " << key << " = " << val << "\n";
            });
    }
};

int main() {
    DataModel model;
    {
        auto widget = std::make_shared<DashboardWidget>();
        widget->bind(model);
        model.set("temperature", 42);  // Widget notified
    }
    // widget destroyed — auto-unregistered
    model.set("temperature", 50);  // No crash, no notification
    return 0;
}

```

### Q3: Build a thread-safe observable property

**Answer:**

```cpp

#include <functional>
#include <vector>
#include <mutex>
#include <shared_mutex>
#include <iostream>

template<typename T>
class ObservableProperty {
public:
    using ChangeHandler = std::function<void(const T& old_val, const T& new_val)>;

    explicit ObservableProperty(T initial)
        : value_(std::move(initial)) {}

    // Thread-safe read
    T get() const {
        std::shared_lock lock(mtx_);
        return value_;
    }

    // Thread-safe write with notification
    void set(T new_val) {
        std::vector<ChangeHandler> handlers_copy;
        T old_val;
        {
            std::unique_lock lock(mtx_);
            if (value_ == new_val) return;  // No change
            old_val = std::move(value_);
            value_ = std::move(new_val);
            handlers_copy = handlers_;  // Copy handlers under lock
        }
        // Notify OUTSIDE lock to prevent deadlocks
        for (auto& h : handlers_copy)
            h(old_val, value_);
    }

    void on_change(ChangeHandler handler) {
        std::unique_lock lock(mtx_);
        handlers_.push_back(std::move(handler));
    }

private:
    T value_;
    std::vector<ChangeHandler> handlers_;
    mutable std::shared_mutex mtx_;
};

int main() {
    ObservableProperty<int> counter(0);
    counter.on_change([](int old_v, int new_v) {
        std::cout << old_v << " -> " << new_v << "\n";
    });

    counter.set(1);   // 0 -> 1
    counter.set(5);   // 1 -> 5
    counter.set(5);   // No output (same value)
    return 0;
}

```

---

## Notes

- Use `std::weak_ptr` for automatic observer cleanup — no manual unregister needed
- **Always notify outside the lock** to prevent deadlocks (observer callbacks may re-enter)
- `Connection` objects give callers explicit control over their subscription lifetime
- For production: consider libraries like Boost.Signals2 (thread-safe, connection management)
- Prefer signals/slots over virtual observer interfaces — more flexible, less coupling
- Be careful with lambda captures: if the observer is destroyed, captured `this` dangles
