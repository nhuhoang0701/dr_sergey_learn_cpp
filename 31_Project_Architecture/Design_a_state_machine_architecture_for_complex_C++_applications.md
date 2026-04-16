# Design a state machine architecture for complex C++ applications

**Category:** Project Architecture

---

## Topic Overview

**State machines** model systems where behavior depends on current state and incoming events. In C++ projects they appear in protocol parsers, UI workflows, device drivers, and game logic. A well-designed state machine makes transitions explicit, eliminates impossible states, and simplifies testing. Modern C++ offers `std::variant`-based approaches that give compile-time exhaustiveness checking.

### State Machine Implementation Approaches

| Approach | Type Safety | Overhead | Extensibility |
| --- | --- | --- | --- |
| **switch/enum** | Low (no exhaustiveness) | Zero | Hard to extend |
| **State pattern (virtual)** | Medium | vtable | Open/closed |
| **std::variant + visit** | High (compile-time) | Small | Closed set |
| **Boost.SML** | Very high | Zero (compile-time) | DSL-defined |

---

## Self-Assessment

### Q1: Implement a variant-based state machine with compile-time safety

**Answer:**

```cpp

#include <variant>
#include <string>
#include <iostream>
#include <optional>

// === States (each state can carry data) ===
struct Idle {};
struct Connecting { std::string address; int retries = 0; };
struct Connected  { int session_id; };
struct Disconnecting { std::string reason; };
struct Error { std::string message; int code; };

using State = std::variant<Idle, Connecting, Connected,
                            Disconnecting, Error>;

// === Events ===
struct Connect     { std::string address; };
struct ConnectOk   { int session_id; };
struct ConnectFail { std::string error; };
struct Disconnect  { std::string reason; };
struct DisconnectOk {};
struct Timeout {};
struct Reset {};

using Event = std::variant<Connect, ConnectOk, ConnectFail,
                            Disconnect, DisconnectOk, Timeout, Reset>;

// === Transition function ===
struct Transition {
    // Idle + Connect -> Connecting
    State operator()(Idle&, Connect& e) {
        return Connecting{e.address};
    }

    // Connecting + ConnectOk -> Connected
    State operator()(Connecting&, ConnectOk& e) {
        return Connected{e.session_id};
    }

    // Connecting + ConnectFail -> Error or retry
    State operator()(Connecting& s, ConnectFail& e) {
        if (s.retries < 3)
            return Connecting{s.address, s.retries + 1};
        return Error{e.error, -1};
    }

    // Connecting + Timeout -> retry or Error
    State operator()(Connecting& s, Timeout&) {
        if (s.retries < 3)
            return Connecting{s.address, s.retries + 1};
        return Error{"Connection timeout", -2};
    }

    // Connected + Disconnect -> Disconnecting
    State operator()(Connected&, Disconnect& e) {
        return Disconnecting{e.reason};
    }

    // Disconnecting + DisconnectOk -> Idle
    State operator()(Disconnecting&, DisconnectOk&) {
        return Idle{};
    }

    // Any state + Reset -> Idle
    State operator()(auto&, Reset&) {
        return Idle{};
    }

    // Default: remain in current state (invalid transition)
    State operator()(auto& state, auto&) {
        return state;  // No transition
    }
};

// === State Machine ===
class ConnectionStateMachine {
public:
    void process(Event event) {
        state_ = std::visit(
            [](auto& s, auto& e) -> State {
                return Transition{}(s, e);
            },
            state_, event);
    }

    const State& current() const { return state_; }

    std::string state_name() const {
        return std::visit([](const auto& s) -> std::string {
            using T = std::decay_t<decltype(s)>;
            if constexpr (std::is_same_v<T, Idle>)           return "Idle";
            if constexpr (std::is_same_v<T, Connecting>)     return "Connecting";
            if constexpr (std::is_same_v<T, Connected>)      return "Connected";
            if constexpr (std::is_same_v<T, Disconnecting>)  return "Disconnecting";
            if constexpr (std::is_same_v<T, Error>)          return "Error";
        }, state_);
    }

private:
    State state_ = Idle{};
};

```

### Q2: Test state machine transitions exhaustively

**Answer:**

```cpp

#include <gtest/gtest.h>

TEST(ConnectionSM, IdleToConnecting) {
    ConnectionStateMachine sm;
    sm.process(Connect{"192.168.1.1"});
    EXPECT_EQ(sm.state_name(), "Connecting");
    EXPECT_EQ(std::get<Connecting>(sm.current()).address, "192.168.1.1");
}

TEST(ConnectionSM, ConnectingToConnected) {
    ConnectionStateMachine sm;
    sm.process(Connect{"host"});
    sm.process(ConnectOk{42});
    EXPECT_EQ(sm.state_name(), "Connected");
    EXPECT_EQ(std::get<Connected>(sm.current()).session_id, 42);
}

TEST(ConnectionSM, RetriesOnFailure) {
    ConnectionStateMachine sm;
    sm.process(Connect{"host"});

    for (int i = 0; i < 3; ++i) {
        sm.process(ConnectFail{"refused"});
        EXPECT_EQ(sm.state_name(), "Connecting");
        EXPECT_EQ(std::get<Connecting>(sm.current()).retries, i + 1);
    }

    // 4th failure -> Error
    sm.process(ConnectFail{"refused"});
    EXPECT_EQ(sm.state_name(), "Error");
}

TEST(ConnectionSM, ResetFromAnyState) {
    ConnectionStateMachine sm;
    sm.process(Connect{"host"});
    sm.process(ConnectOk{1});
    EXPECT_EQ(sm.state_name(), "Connected");

    sm.process(Reset{});
    EXPECT_EQ(sm.state_name(), "Idle");
}

TEST(ConnectionSM, InvalidTransitionIgnored) {
    ConnectionStateMachine sm;  // Idle
    sm.process(ConnectOk{1});   // Invalid: Idle + ConnectOk
    EXPECT_EQ(sm.state_name(), "Idle");  // Unchanged
}

```

### Q3: Add entry/exit actions and transition guards

**Answer:**

```cpp

// === State machine with actions and guards ===
template<typename StateMachine>
class WithActions {
public:
    void process(Event event) {
        auto old_state = sm_.current();

        // Exit action for old state
        std::visit([this](const auto& s) { on_exit(s); }, old_state);

        sm_.process(std::move(event));

        // Entry action for new state (only if state changed)
        if (old_state.index() != sm_.current().index()) {
            std::visit([this](const auto& s) { on_enter(s); },
                       sm_.current());
        }
    }

    const State& current() const { return sm_.current(); }

private:
    void on_enter(const Connecting& s) {
        std::cout << "Starting connection to " << s.address << "\n";
        // Start connection timer
    }
    void on_enter(const Connected& s) {
        std::cout << "Connected, session=" << s.session_id << "\n";
    }
    void on_enter(const Error& s) {
        std::cout << "Error: " << s.message << "\n";
        // Log, alert, trigger recovery
    }
    void on_enter(const auto&) {}  // Default: no action

    void on_exit(const Connected&) {
        std::cout << "Leaving connected state\n";
        // Flush buffers, cancel pending requests
    }
    void on_exit(const auto&) {}  // Default: no action

    StateMachine sm_;
};

// Usage:
// WithActions<ConnectionStateMachine> sm;
// sm.process(Connect{"host"});  // Prints: Starting connection to host
// sm.process(ConnectOk{1});     // Prints: Connected, session=1

```

---

## Notes

- `std::variant` + `std::visit` gives **compile-time exhaustiveness** — missing transition = compile error
- Each state struct can carry state-specific data (address, retries, session ID)
- The `auto& state, auto& event` catch-all handles invalid transitions gracefully
- **Test every valid transition AND every invalid one** — the catch-all must preserve state
- Entry/exit actions handle resource acquisition and cleanup at state boundaries
- For large state machines (>15 states), consider Boost.SML for a declarative DSL
- Hierarchical state machines: a state can contain a nested state machine (HSM pattern)
