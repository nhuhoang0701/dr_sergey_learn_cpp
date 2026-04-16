# Apply the type-state pattern to prevent operations on invalid states

**Category:** Best Practices & Idioms  
**Item:** #501  
**Reference:** <https://en.cppreference.com/w/cpp/language/types>  

---

## Topic Overview

The **type-state pattern** encodes an object's state in the type system, so invalid operations are **compile-time errors** rather than runtime crashes.

### Runtime vs Compile-Time State Checking

```cpp

Runtime (enum + assert):            Compile-time (type-state):
socket.send(data);                  // auto connected = socket.connect(addr);
// throws if not connected!         connected.send(data);  // only exists on Connected!
// discovered at runtime             // wrong state = compiler error

```

### Type-State Transition Diagram

```cpp

 SocketClosed  ─── connect() ───▶  SocketConnected
                                    │
                    send()/recv()   │  close()
                    (available)     │
                                    ▼
                               SocketClosed

 SocketClosed  ─── listen()  ───▶  SocketListening
                                    │
                    accept()        │
                    (available)     │

```

---

## Self-Assessment

### Q1: Implement a Socket with different types for Unconnected, Connected, and Listening states

```cpp

#include <iostream>
#include <string>

// Forward declarations
class SocketConnected;
class SocketListening;

// State 1: Closed (initial state)
class SocketClosed {
    int fd_;
public:
    explicit SocketClosed(int fd) : fd_(fd) {
        std::cout << "Socket created (fd=" << fd_ << ")\n";
    }

    // Transitions to Connected state
    SocketConnected connect(const std::string& addr);

    // Transitions to Listening state
    SocketListening listen(int port);

    // send() and recv() do NOT exist here — compile error if called!
};

// State 2: Connected (can send/recv)
class SocketConnected {
    int fd_;
    std::string addr_;
public:
    SocketConnected(int fd, std::string addr) : fd_(fd), addr_(std::move(addr)) {
        std::cout << "Connected to " << addr_ << "\n";
    }

    void send(const std::string& data) {
        std::cout << "Sending " << data.size() << " bytes to " << addr_ << "\n";
    }

    std::string recv() {
        std::cout << "Receiving from " << addr_ << "\n";
        return "response_data";
    }

    // Transition back to Closed
    SocketClosed close() {
        std::cout << "Closing connection to " << addr_ << "\n";
        return SocketClosed(fd_);
    }

    // listen() does NOT exist — compile error!
};

// State 3: Listening (can accept)
class SocketListening {
    int fd_;
    int port_;
public:
    SocketListening(int fd, int port) : fd_(fd), port_(port) {
        std::cout << "Listening on port " << port_ << "\n";
    }

    SocketConnected accept() {
        std::cout << "Accepted connection on port " << port_ << "\n";
        return SocketConnected(fd_, "client:" + std::to_string(port_));
    }

    // send() and recv() do NOT exist — compile error!
};

// Implementations
SocketConnected SocketClosed::connect(const std::string& addr) {
    return SocketConnected(fd_, addr);
}

SocketListening SocketClosed::listen(int port) {
    return SocketListening(fd_, port);
}

int main() {
    // Valid usage: follows state transitions
    SocketClosed sock(3);
    auto connected = sock.connect("server:8080");
    connected.send("Hello");
    auto response = connected.recv();
    auto closed = connected.close();

    // Server path
    SocketClosed server(4);
    auto listening = server.listen(9090);
    auto client = listening.accept();
    client.send("Welcome");
}
// Expected output:
// Socket created (fd=3)
// Connected to server:8080
// Sending 5 bytes to server:8080
// Receiving from server:8080
// Closing connection to server:8080
// Socket created (fd=3)
// Socket created (fd=4)
// Listening on port 9090
// Accepted connection on port 9090
// Connected to client:9090
// Sending 7 bytes to client:9090

```

### Q2: Show that calling `send()` on an Unconnected socket is a compile error

```cpp

// These lines would NOT compile:

// SocketClosed sock(1);
// sock.send("data");   // ERROR: 'SocketClosed' has no member named 'send'
// sock.recv();         // ERROR: 'SocketClosed' has no member named 'recv'

// SocketListening listener(2, 80);
// listener.send("data");  // ERROR: 'SocketListening' has no member named 'send'

// The compiler catches invalid state transitions:
// auto connected = sock.connect("addr");
// connected.listen(80);  // ERROR: 'SocketConnected' has no member named 'listen'

```

**Compiler errors (GCC):**

```cpp

error: 'class SocketClosed' has no member named 'send'
error: 'class SocketListening' has no member named 'send'
error: 'class SocketConnected' has no member named 'listen'

```

Every invalid operation is caught **at compile time**, not at runtime.

### Q3: Compare type-state encoding vs runtime state enum + assert

```cpp

// RUNTIME approach (traditional)
class RuntimeSocket {
    enum class State { Closed, Connected, Listening };
    State state_ = State::Closed;
public:
    void send(const std::string& data) {
        assert(state_ == State::Connected);  // runtime check!
        // ... send data
    }
    void connect(const std::string& addr) {
        assert(state_ == State::Closed);
        state_ = State::Connected;
    }
};
// Problem: assert only fires in debug builds!
// In release: UB or silent corruption

```

**Comparison:**

| Aspect | Type-State (compile-time) | Enum+Assert (runtime) |
| --- | --- | --- |
| Error detection | Compile error | Runtime crash (debug only) |
| Release builds | Still safe | assert disabled (⚠️ UB) |
| Code verbosity | More types (3+ classes) | Simpler (1 class) |
| Flexibility | Must know state at compile time | Can change state dynamically |
| Polymorphism | Harder (different types) | Easy (one type) |
| Discoverability | IDE shows valid methods | Must read docs |

**When to use type-state:**

- Protocol states (TCP, HTTP, file handles)
- Builder pattern (must call required setters)
- State machines with clear transitions

**When runtime state is better:**

- State determined by user input or network
- Many possible states (combinatorial explosion of types)
- Need to store heterogeneous states in containers

---

## Notes

- Type-state is a form of **making illegal states unrepresentable** — a key principle in reliable software.
- Consuming `this` (move semantics) prevents using the old state after transition.
- C++ doesn't have Rust's `move` semantics for ownership transfer, so the moved-from object still exists (but is in a valid-but-unspecified state).
- Combine with `[[nodiscard]]` on transition methods to prevent ignoring the new state.
