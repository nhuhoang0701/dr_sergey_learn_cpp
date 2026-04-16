# Use stdexec::let_value and let_error for monadic sender chaining

**Category:** std::execution & Senders/Receivers  
**Item:** #704  
**Standard:** C++20  
**Reference:** <https://github.com/NVIDIA/stdexec>  

---

## Topic Overview

`let_value` and `let_error` enable **dependent** sender chaining — where the next async step depends on the result of the previous one.

| Adaptor | Input | Callback returns | Purpose |
| --- | --- | --- | --- |
| `then(f)` | value | value (not a sender) | Simple transform |
| `let_value(f)` | value | **sender** | Dependent async step |
| `let_error(f)` | error | **sender** | Error recovery with async work |
| `let_stopped(f)` | stopped | **sender** | Cancellation recovery |

```cpp

then(f):       value ─→ f(value) ─→ new_value
let_value(f):  value ─→ f(value) ─→ sender ─→ (execute sender) ─→ result

```

---

## Self-Assessment

### Q1: `let_value` for dependent sender chaining

```cpp

#include <stdexec/execution.hpp>
#include <exec/static_thread_pool.hpp>
#include <iostream>
#include <string>

// Simulate async operations that return senders:
auto fetch_user_id(const std::string& token) {
    return stdexec::just(42);  // user_id = 42
}

auto fetch_user_profile(int user_id) {
    return stdexec::just(std::string("Alice"));
}

auto fetch_user_posts(const std::string& username) {
    return stdexec::just(10);  // 10 posts
}

int main() {
    // let_value chains DEPENDENT senders:
    // Each step needs the RESULT of the previous step to decide what to do next.
    auto pipeline = stdexec::just(std::string("auth_token_xyz"))
        | stdexec::let_value([](const std::string& token) {
            // Returns a SENDER (not a value!):
            return fetch_user_id(token);
        })
        | stdexec::let_value([](int user_id) {
            std::cout << "User ID: " << user_id << '\n';
            return fetch_user_profile(user_id);
        })
        | stdexec::let_value([](const std::string& username) {
            std::cout << "Username: " << username << '\n';
            return fetch_user_posts(username);
        })
        | stdexec::then([](int post_count) {
            std::cout << "Posts: " << post_count << '\n';
            return post_count;
        });

    auto [result] = stdexec::sync_wait(std::move(pipeline)).value();
    std::cout << "Final: " << result << '\n';
}
// Output:
// User ID: 42
// Username: Alice
// Posts: 10
// Final: 10

```

**Why not `then`?** `then` expects `f(value)` to return a **value**. But `fetch_user_id` returns a **sender**. `let_value` unwraps the sender and executes it.

### Q2: `let_error` for error recovery with a fallback sender

```cpp

#include <stdexec/execution.hpp>
#include <iostream>
#include <string>

auto fetch_from_primary() {
    // Simulate primary server failure:
    return stdexec::just_error(
        std::make_exception_ptr(std::runtime_error("primary down")));
}

auto fetch_from_backup() {
    return stdexec::just(std::string("data from backup"));
}

int main() {
    // let_error: receives the error, returns a SENDER for recovery
    auto pipeline = fetch_from_primary()
        | stdexec::let_error([](std::exception_ptr ep) {
            try { std::rethrow_exception(ep); }
            catch (const std::exception& e) {
                std::cout << "Primary failed: " << e.what()
                          << ", trying backup...\n";
            }
            // Return a fallback SENDER:
            return fetch_from_backup();
        })
        | stdexec::then([](const std::string& data) {
            std::cout << "Got: " << data << '\n';
            return data;
        });

    auto [result] = stdexec::sync_wait(std::move(pipeline)).value();
    std::cout << "Result: " << result << '\n';
}
// Output:
// Primary failed: primary down, trying backup...
// Got: data from backup
// Result: data from backup

```

**`upon_error` vs `let_error`:**

- `upon_error(f)`: `f` returns a **value** (simple transformation).
- `let_error(f)`: `f` returns a **sender** (can do async recovery).

### Q3: `let_value` vs coroutine `co_await`

```cpp

// With coroutines (sequential, readable):
exec::task<int> coroutine_version() {
    auto token = co_await get_auth_token();
    auto user_id = co_await fetch_user_id(token);
    auto profile = co_await fetch_user_profile(user_id);
    co_return profile.post_count;
}

// With let_value (pipeline, composable):
auto sender_version() {
    return get_auth_token()
        | stdexec::let_value([](auto token) {
            return fetch_user_id(token);
        })
        | stdexec::let_value([](auto user_id) {
            return fetch_user_profile(user_id);
        })
        | stdexec::then([](auto profile) {
            return profile.post_count;
        });
}

```

| Aspect | `co_await` (Coroutines) | `let_value` (Senders) |
| --- | --- | --- |
| Readability | More natural for sequential | Pipeline-style |
| Parallel fan-out | Manual | `when_all` + `let_value` |
| Scheduler control | Implicit | Explicit (`starts_on`, `continues_on`) |
| Allocation | Coroutine frame (heap) | Operation state (stack possible) |
| Error handling | try/catch + co_await | `let_error` + error channel |
| Cancellation | Manual stop_token | Automatic via stopped channel |

```cpp

// Best of both: combine them!
exec::task<int> combined() {
    // Sequential part (coroutine):
    auto config = co_await load_config();

    // Fan-out part (senders):
    auto [a, b] = co_await stdexec::when_all(
        fetch_a(config.url_a),
        fetch_b(config.url_b)
    );

    // Sequential again:
    co_return merge(a, b);
}

```

---

## Notes

- `let_value` is the sender equivalent of monadic `flatMap` / `>>=` (bind).
- The callback's return value MUST be a sender. Returning a non-sender is a compile error.
- `let_value` keeps the value alive for the lifetime of the returned sender.
- Use `then` for simple transforms; use `let_value` only when you need to return a sender.
- `let_stopped` is available for cancellation recovery returning a sender.
