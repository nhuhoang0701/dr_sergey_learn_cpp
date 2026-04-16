# Implement an event bus using std::variant and type-indexed dispatch

**Category:** Design Patterns — Modern Takes  
**Item:** #675  
**Standard:** C++17  
**Reference:** <https://en.cppreference.com/w/cpp/types/type_index>  

---

## Topic Overview

An **event bus** decouples publishers from subscribers — components fire events without knowing who listens. Using `std::type_index` as the key and `std::any`-wrapped handlers, we get a fully type-safe event system with no macros or inheritance hierarchies.

### Architecture

```cpp

Publisher:                        EventBus:                       Subscribers:
  bus.publish(MouseClick{x,y})      handlers[type_index(MouseClick)]     handler1(MouseClick)
                                    → [handler1, handler2]               handler2(MouseClick)
  bus.publish(KeyPress{'A'})        handlers[type_index(KeyPress)]       handler3(KeyPress)
                                    → [handler3]

```

---

## Self-Assessment

### Q1: Define EventBus that maps event types to lists of handlers using std::unordered_map<type_index, vector<any>>

**Answer:**

```cpp

#include <iostream>
#include <any>
#include <vector>
#include <unordered_map>
#include <typeindex>
#include <functional>

class EventBus {
    // Map: event type → list of handlers (type-erased as std::any)
    std::unordered_map<std::type_index, std::vector<std::any>> handlers_;

public:
    // Subscribe a handler for event type T
    template<typename T>
    void subscribe(std::function<void(const T&)> handler) {
        auto key = std::type_index(typeid(T));
        handlers_[key].push_back(std::move(handler));
    }

    // Publish an event — calls all registered handlers for type T
    template<typename T>
    void publish(const T& event) {
        auto key = std::type_index(typeid(T));
        auto it = handlers_.find(key);
        if (it == handlers_.end()) return;  // No subscribers — no-op

        for (auto& handler_any : it->second) {
            auto& handler = std::any_cast<std::function<void(const T&)>&>(handler_any);
            handler(event);
        }
    }

    // Clear all handlers for a specific event type
    template<typename T>
    void unsubscribe_all() {
        handlers_.erase(std::type_index(typeid(T)));
    }
};

// ═══════════ Event types (plain structs, no base class needed) ═══════════
struct MouseClick  { int x, y; int button; };
struct KeyPress    { char key; bool shift; };
struct WindowClose {};

int main() {
    EventBus bus;

    // Subscribe handlers
    bus.subscribe<MouseClick>([](const MouseClick& e) {
        std::cout << "Click at (" << e.x << ", " << e.y << ")\n";
    });

    bus.subscribe<MouseClick>([](const MouseClick& e) {
        if (e.button == 1)
            std::cout << "Left click detected\n";
    });

    bus.subscribe<KeyPress>([](const KeyPress& e) {
        std::cout << "Key pressed: " << e.key
                  << (e.shift ? " +Shift" : "") << '\n';
    });

    bus.subscribe<WindowClose>([](const WindowClose&) {
        std::cout << "Window closing — saving state...\n";
    });

    // Publish events
    bus.publish(MouseClick{100, 200, 1});
    bus.publish(KeyPress{'A', true});
    bus.publish(WindowClose{});
}

```

**Output:**

```text

Click at (100, 200)
Left click detected
Key pressed: A +Shift
Window closing — saving state...

```

### Q2: Subscribe and publish events with type safety: bus.subscribe<MouseEvent>(handler)

**Answer:**

```cpp

#include <iostream>
#include <any>
#include <vector>
#include <unordered_map>
#include <typeindex>
#include <functional>
#include <string>

// ═══════════ Enhanced EventBus with subscription IDs ═══════════
class EventBus {
    using HandlerID = std::size_t;
    struct HandlerEntry {
        HandlerID id;
        std::any handler;
    };
    
    std::unordered_map<std::type_index, std::vector<HandlerEntry>> handlers_;
    HandlerID next_id_ = 0;

public:
    // Returns a subscription ID for later unsubscription
    template<typename T>
    HandlerID subscribe(std::function<void(const T&)> handler) {
        HandlerID id = next_id_++;
        auto key = std::type_index(typeid(T));
        handlers_[key].push_back({id, std::move(handler)});
        return id;
    }

    // Lambda overload (auto-wraps in std::function)
    template<typename T, typename F>
    HandlerID subscribe(F&& func) {
        return subscribe<T>(std::function<void(const T&)>(std::forward<F>(func)));
    }

    template<typename T>
    void publish(const T& event) {
        auto it = handlers_.find(std::type_index(typeid(T)));
        if (it == handlers_.end()) return;

        for (auto& entry : it->second) {
            auto& fn = std::any_cast<std::function<void(const T&)>&>(entry.handler);
            fn(event);
        }
    }

    // Unsubscribe a specific handler by ID
    void unsubscribe(HandlerID id) {
        for (auto& [type, entries] : handlers_) {
            std::erase_if(entries, [id](const HandlerEntry& e) { return e.id == id; });
        }
    }
};

// Event types
struct PlayerDamaged { std::string player; int amount; };
struct PlayerHealed  { std::string player; int amount; };
struct GameOver      { std::string winner; };

int main() {
    EventBus bus;

    // Type-safe subscriptions
    auto id1 = bus.subscribe<PlayerDamaged>([](const PlayerDamaged& e) {
        std::cout << e.player << " took " << e.amount << " damage\n";
    });

    bus.subscribe<PlayerDamaged>([](const PlayerDamaged& e) {
        if (e.amount > 50) std::cout << "CRITICAL HIT!\n";
    });

    bus.subscribe<PlayerHealed>([](const PlayerHealed& e) {
        std::cout << e.player << " healed for " << e.amount << "\n";
    });

    // Type-safe publishing
    bus.publish(PlayerDamaged{"Alice", 75});  // Both damage handlers fire
    bus.publish(PlayerHealed{"Alice", 30});   // Only heal handler fires

    // Unsubscribe first damage handler
    bus.unsubscribe(id1);
    bus.publish(PlayerDamaged{"Bob", 60});    // Only "CRITICAL HIT!" handler fires
}

```

**Output:**

```text

Alice took 75 damage
CRITICAL HIT!
Alice healed for 30
CRITICAL HIT!

```

### Q3: Show that publishing an unsubscribed event type is a no-op, not an error

**Answer:**

```cpp

#include <iostream>
#include <any>
#include <vector>
#include <unordered_map>
#include <typeindex>
#include <functional>

class EventBus {
    std::unordered_map<std::type_index, std::vector<std::any>> handlers_;
public:
    template<typename T>
    void subscribe(std::function<void(const T&)> handler) {
        handlers_[std::type_index(typeid(T))].push_back(std::move(handler));
    }

    template<typename T>
    void publish(const T& event) {
        auto it = handlers_.find(std::type_index(typeid(T)));
        if (it == handlers_.end()) {
            // No handlers registered for this type — silent no-op
            // This is intentional: publishers shouldn't know/care if anyone listens
            return;
        }
        for (auto& h : it->second) {
            std::any_cast<std::function<void(const T&)>&>(h)(event);
        }
    }
};

struct Click    { int x, y; };
struct Hover    { int x, y; };
struct Scroll   { int delta; };
struct Resize   { int w, h; };  // Nobody subscribes to this

int main() {
    EventBus bus;

    // Only subscribe to Click
    bus.subscribe<Click>([](const Click& e) {
        std::cout << "Click handler: (" << e.x << ", " << e.y << ")\n";
    });

    // Publish various events:
    bus.publish(Click{10, 20});    // ✅ Handler called
    bus.publish(Hover{30, 40});    // ✅ No crash, no error — just a no-op
    bus.publish(Scroll{-3});       // ✅ No crash, no error — just a no-op
    bus.publish(Resize{800, 600}); // ✅ No crash, no error — just a no-op

    std::cout << "All events published safely\n";
    // Only Click produced output — all others were silent no-ops
}

```

**Output:**

```text

Click handler: (10, 20)
All events published safely

```

---

## Notes

- **Thread safety:** wrap `handlers_` access in `std::shared_mutex` for concurrent publish/subscribe
- **Event queuing:** instead of immediate dispatch, store events and process them in batch
- **Alternatives:** `std::function<void(const std::variant<Events...>&)>` for a closed set of event types (no type erasure overhead)
- **Performance:** `std::any_cast` has overhead — for hot paths, consider a template-based bus with `std::tuple` of handler vectors
