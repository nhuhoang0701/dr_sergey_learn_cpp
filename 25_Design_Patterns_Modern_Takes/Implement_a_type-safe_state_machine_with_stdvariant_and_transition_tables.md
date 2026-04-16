# Implement a type-safe state machine with std::variant and transition tables

**Category:** Design Patterns — Modern Takes  
**Item:** #669  
**Standard:** C++17  
**Reference:** <https://en.cppreference.com/w/cpp/utility/variant>  

---

## Topic Overview

This second state machine file focuses on **transition tables as data** — encoding transitions in a constexpr array rather than inline in `std::visit` lambdas. It also demonstrates how adding a new state creates compile errors in incomplete visitors.

### Data-Driven vs Code-Driven Transitions

```cpp

Code-driven (visit lambdas):
  Transitions are spread across lambda bodies

  + Easy for complex transition logic (guards, actions)
  - Hard to visualize all transitions at a glance

Data-driven (transition table):
  constexpr array of {from, event, to, guard, action}

  + All transitions visible in one table
  + Easy to add/remove transitions
  - Less flexible for complex actions

```

---

## Self-Assessment

### Q1: Encode states as variant alternatives and transitions as overloaded visit lambdas

**Answer:**

```cpp

#include <iostream>
#include <variant>
#include <string>

// ═══════════ States ═══════════
struct Idle     { };
struct Running  { int speed; };
struct Paused   { int saved_speed; };
struct Stopped  { std::string reason; };

using State = std::variant<Idle, Running, Paused, Stopped>;

// ═══════════ Events ═══════════
struct Start { int speed; };
struct Pause {};
struct Resume {};
struct Stop  { std::string reason; };

using Event = std::variant<Start, Pause, Resume, Stop>;

// ═══════════ Overload helper ═══════════
template<class... Ts> struct overloaded : Ts... { using Ts::operator()...; };
template<class... Ts> overloaded(Ts...) -> overloaded<Ts...>;

// Transitions encoded as overloaded visit lambdas
State next_state(const State& state, const Event& event) {
    return std::visit(overloaded{
        [](const Idle&, const Start& e) -> State {
            std::cout << "Starting at speed " << e.speed << "\n";
            return Running{e.speed};
        },
        [](const Running& s, const Pause&) -> State {
            std::cout << "Pausing (was speed " << s.speed << ")\n";
            return Paused{s.speed};
        },
        [](const Running&, const Stop& e) -> State {
            std::cout << "Stopping: " << e.reason << "\n";
            return Stopped{e.reason};
        },
        [](const Paused& s, const Resume&) -> State {
            std::cout << "Resuming at speed " << s.saved_speed << "\n";
            return Running{s.saved_speed};
        },
        [](const Paused&, const Stop& e) -> State {
            std::cout << "Stopping from pause: " << e.reason << "\n";
            return Stopped{e.reason};
        },
        // Catch-all for invalid transitions
        [](const auto& s, const auto&) -> State {
            std::cout << "Invalid transition\n";
            return s;
        }
    }, state, event);
}

int main() {
    State state = Idle{};

    state = next_state(state, Start{100});    // → Running{100}
    state = next_state(state, Pause{});        // → Paused{100}
    state = next_state(state, Start{50});      // Invalid (can't start from paused)
    state = next_state(state, Resume{});       // → Running{100}
    state = next_state(state, Stop{"done"});   // → Stopped{"done"}
}

```

### Q2: Build a transition table as a constexpr array of (from_state, event, to_state) triples

**Answer:**

```cpp

#include <iostream>
#include <variant>
#include <array>
#include <optional>
#include <functional>
#include <string_view>

// States as enum indices (for the table) and types (for the variant)
enum class StateID { Locked, Unlocked, Open, COUNT };
enum class EventID { Lock, Unlock, Push, Close, COUNT };

// ═══════════ Transition Table (compile-time data) ═══════════
struct Transition {
    StateID from;
    EventID event;
    StateID to;
    std::string_view action;  // Description of what happens
};

constexpr auto transition_table = std::array{
    Transition{StateID::Locked,   EventID::Unlock, StateID::Unlocked, "Key turns, door unlocks"},
    Transition{StateID::Unlocked, EventID::Lock,   StateID::Locked,   "Door locks"},
    Transition{StateID::Unlocked, EventID::Push,   StateID::Open,     "Door swings open"},
    Transition{StateID::Open,     EventID::Close,  StateID::Unlocked, "Door closes"},
};

// Lookup: O(N) scan, but N is tiny and constexpr-friendly
constexpr std::optional<Transition> find_transition(StateID from, EventID event) {
    for (const auto& t : transition_table) {
        if (t.from == from && t.event == event)
            return t;
    }
    return std::nullopt;
}

// ═══════════ Runtime state machine using the table ═══════════
constexpr std::string_view state_name(StateID s) {
    constexpr std::string_view names[] = {"Locked", "Unlocked", "Open"};
    return names[static_cast<int>(s)];
}

constexpr std::string_view event_name(EventID e) {
    constexpr std::string_view names[] = {"Lock", "Unlock", "Push", "Close"};
    return names[static_cast<int>(e)];
}

class DoorFSM {
    StateID state_ = StateID::Locked;
public:
    bool process(EventID event) {
        auto t = find_transition(state_, event);
        if (t) {
            std::cout << state_name(state_) << " + " << event_name(event)
                      << " → " << state_name(t->to) << " (" << t->action << ")\n";
            state_ = t->to;
            return true;
        }
        std::cout << state_name(state_) << " + " << event_name(event)
                  << " → INVALID\n";
        return false;
    }
};

int main() {
    DoorFSM door;
    door.process(EventID::Push);     // Locked + Push → INVALID
    door.process(EventID::Unlock);   // Locked + Unlock → Unlocked
    door.process(EventID::Push);     // Unlocked + Push → Open
    door.process(EventID::Close);    // Open + Close → Unlocked
    door.process(EventID::Lock);     // Unlocked + Lock → Locked

    // Compile-time validation:
    static_assert(find_transition(StateID::Locked, EventID::Unlock).has_value());
    static_assert(!find_transition(StateID::Locked, EventID::Push).has_value());
}

```

### Q3: Show that adding a new state forces a compile error in any incomplete visitor

**Answer:**

```cpp

#include <iostream>
#include <variant>

struct Off {};
struct On  {};
struct Standby {};  // NEW STATE — forces all visitors to be updated

using DeviceState = std::variant<Off, On, Standby>;

template<class... Ts> struct overloaded : Ts... { using Ts::operator()...; };
template<class... Ts> overloaded(Ts...) -> overloaded<Ts...>;

// This visitor is COMPLETE — handles all 3 states:
void print_state(const DeviceState& s) {
    std::visit(overloaded{
        [](const Off&)     { std::cout << "Device is OFF\n"; },
        [](const On&)      { std::cout << "Device is ON\n"; },
        [](const Standby&) { std::cout << "Device is in STANDBY\n"; },
        // If Standby line is removed: COMPILE ERROR
        // "no matching function for call to 'overloaded<...>::operator()(const Standby&)'"
    }, s);
}

// This visitor uses auto&& catch-all — no error but loses exhaustiveness:
void print_state_loose(const DeviceState& s) {
    std::visit(overloaded{
        [](const Off&) { std::cout << "OFF\n"; },
        [](const On&)  { std::cout << "ON\n"; },
        [](const auto&) { std::cout << "OTHER\n"; },  // Catches Standby silently
        // ⚠️ auto&& suppresses the compile error — use with caution
    }, s);
}

int main() {
    DeviceState s = Standby{};
    print_state(s);       // Device is in STANDBY
    print_state_loose(s); // OTHER
}

```

**Key insight:** avoid `auto&&` catch-all in visitors if you want the compiler to force updates when new states are added.

---

## Notes

- **Transition tables as data** are great for visualization and code generation — you can auto-generate state diagrams from the table
- **Guards:** add a `bool(*guard)()` field to the Transition struct for conditional transitions
- **Actions:** add `void(*action)()` to execute side effects on transitions
- **`static_assert`** with `find_transition()` validates transition existence at compile time
