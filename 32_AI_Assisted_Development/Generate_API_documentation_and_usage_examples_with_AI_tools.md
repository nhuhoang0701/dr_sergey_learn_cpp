# Generate API documentation and usage examples with AI tools

**Category:** AI-Assisted C++ Development

---

## Topic Overview

AI assistants can generate **comprehensive API documentation** from C++ header files, including function descriptions, parameter documentation, return value explanations, usage examples, thread-safety notes, and exception specifications. This is one of the highest-ROI uses of AI for C++ — documentation is essential but tedious to write.

### AI Documentation Generation Quality

| Documentation Type | AI Quality | Key Consideration |
| --- | --- | --- |
| Function/method docs (Doxygen) | Excellent | Verify parameter descriptions |
| Usage examples | Excellent | Test that examples compile |
| Class overview documentation | Good | May need domain context |
| Thread-safety annotations | Medium | Verify actual thread safety |
| Exception specifications | Good | Check against actual throws |
| Migration guides | Good | Needs version context |
| README generation | Excellent | Review for accuracy |

---

## Self-Assessment

### Q1: Generate Doxygen documentation from headers

**Answer:**

```cpp

=== PROMPT ===

"Generate Doxygen documentation for every public method
in this header. Include:

- Brief and detailed description
- @param with direction ([in], [out], [in,out])
- @return with all possible return values
- @throws with conditions
- @note for thread safety and performance
- @code ... @endcode with a usage example
- @pre and @post conditions

Style: imperative mood ('Returns' not 'This returns')."

```

```cpp

// Input header:
template<typename T, typename Alloc = std::allocator<T>>
class ConcurrentQueue {
public:
    ConcurrentQueue();
    explicit ConcurrentQueue(size_t initial_capacity);
    ~ConcurrentQueue();

    bool push(const T& item);
    bool push(T&& item);
    bool try_pop(T& out);
    std::optional<T> try_pop();
    bool try_pop_for(T& out, std::chrono::milliseconds timeout);
    size_t size() const;
    bool empty() const;
    void clear();
};

// === AI-generated Doxygen ===

/**

 * @brief Thread-safe, lock-free multi-producer multi-consumer queue.

 *

 * Provides a bounded concurrent queue suitable for inter-thread
 * communication. Uses a ring buffer internally with atomic operations
 * for lock-free push/pop in the common case.

 *

 * @tparam T     Element type. Must be nothrow move-constructible.
 * @tparam Alloc Allocator type for internal storage (default: std::allocator).

 *

 * @note Thread-safe: all methods can be called concurrently from
 *       multiple threads without external synchronization.
 * @note Lock-free for push/try_pop when queue is neither full nor empty.

 *

 * @code
 * ConcurrentQueue<WorkItem> queue(1024);

 *

 * // Producer thread:
 * queue.push(WorkItem{"task-1", payload});

 *

 * // Consumer thread:
 * WorkItem item;
 * if (queue.try_pop(item)) {
 *     process(item);
 * }
 * @endcode

 */

/**

 * @brief Constructs an empty queue with default capacity (1024).
 * @post empty() == true && size() == 0

 */

/**

 * @brief Constructs an empty queue with the specified capacity.
 * @param[in] initial_capacity Maximum number of elements.
 *            Rounded up to the next power of 2 internally.
 * @pre initial_capacity > 0
 * @throws std::invalid_argument if initial_capacity is 0.
 * @throws std::bad_alloc if memory allocation fails.

 */

/**

 * @brief Enqueues an element by copy.
 * @param[in] item Element to enqueue.
 * @return true if successfully enqueued, false if queue is full.
 * @note Non-blocking. Returns immediately if queue is full.
 * @note Thread-safe: can be called from multiple producer threads.

 */

/**

 * @brief Attempts to dequeue an element without blocking.
 * @param[out] out Receives the dequeued element if successful.
 *             Not modified if queue is empty.
 * @return true if an element was dequeued, false if queue is empty.
 * @note Non-blocking. Returns immediately if queue is empty.

 */

/**

 * @brief Attempts to dequeue with a timeout.
 * @param[out] out Receives the dequeued element if successful.
 * @param[in] timeout Maximum time to wait for an element.
 * @return true if an element was dequeued within the timeout.
 * @note Blocking: may sleep up to @p timeout duration.
 * @note Uses condition variable internally (not lock-free).

 */

```

### Q2: Generate usage examples and tutorials

**Answer:**

```cpp

=== PROMPT ===

"Generate a comprehensive usage guide for this library
with code examples. Cover:

- Quick start (minimal working example)
- Common patterns
- Error handling
- Advanced usage
- Performance tips
- FAQ with code answers"

```

```cpp

// === AI-generated usage guide ===

// ## Quick Start
//
// ```cpp
// #include <my_lib/http_client.hpp>
//
// int main() {
//     http::Client client;
//     auto response = client.get("https://api.example.com/data");
//     if (response.ok()) {
//         std::cout << response.body() << '\n';
//     }
// }
// ```

// ## Common Patterns

// Pattern: POST with JSON body
http::Client client;
auto response = client.post("https://api.example.com/users",
    http::json_body({{"name", "Alice"}, {"age", 30}})
);

// Pattern: Add headers and authentication
client.set_default_header("Authorization", "Bearer " + token);
client.set_default_header("Content-Type", "application/json");

// Pattern: Timeout and retry
auto response = client.get("https://slow-api.example.com",
    http::Options{}
        .timeout(std::chrono::seconds{5})
        .retries(3)
        .retry_delay(std::chrono::milliseconds{500})
);

// ## Error Handling

auto response = client.get(url);
if (!response) {
    // Network error (DNS, connection refused, timeout)
    std::cerr << "Error: " << response.error().message() << '\n';
} else if (response.status() >= 400) {
    // HTTP error (4xx, 5xx)
    std::cerr << "HTTP " << response.status()
              << ": " << response.body() << '\n';
}

// ## FAQ

// Q: How to make concurrent requests?
auto [r1, r2, r3] = http::when_all(
    client.async_get("https://api1.example.com"),
    client.async_get("https://api2.example.com"),
    client.async_get("https://api3.example.com")
);

// Q: How to stream large responses?
client.get_stream(url, [](std::span<const char> chunk) {
    process_chunk(chunk);
    return true;  // Continue receiving
});

```

### Q3: Generate API changelog and migration docs

**Answer:**

```cpp

=== PROMPT ===

"Compare these two header versions and generate:

1. Changelog (what changed, added, removed, deprecated)
2. Migration guide with before/after code examples
3. Breaking changes list"

```

```markdown

## AI-Generated Changelog: v2.0 -> v3.0

### Breaking Changes

- `Client::get()` now returns `Expected<Response, Error>`

  instead of `Response` (was throwing on error)

- `Response::body()` returns `std::string_view` instead

  of `std::string` (zero-copy, but don't store it!)

- Removed: `Client::set_timeout(int ms)` ->

  use `Options::timeout(chrono::milliseconds)`

### Migration Guide

#### Error handling (BREAKING)

```cpp

// v2.0 (exceptions)
try {
    auto resp = client.get(url);
    use(resp.body());
} catch (const http::Error& e) {
    handle_error(e);
}

// v3.0 (Expected)
auto resp = client.get(url);
if (resp) {
    use(resp->body());
} else {
    handle_error(resp.error());
}

```

#### Timeout configuration (BREAKING)

```cpp

// v2.0
client.set_timeout(5000);  // Removed

// v3.0
client.get(url, http::Options{}
    .timeout(std::chrono::seconds{5}));

```

```text

---

## Notes

- Always **paste the actual header** — AI needs the signatures to generate accurate docs
- Ask AI to generate **compilable examples** and test them before publishing
- For Doxygen, specify the **style** (brief descriptions, parameter directions, etc.)
- AI is excellent at generating **README.md** files with badges, install instructions, and quick start
- For large APIs, process **one class at a time** for better quality
- Ask AI to identify **missing documentation** by comparing header declarations against doc comments
- Use AI to maintain docs: "update these docs for the changes in this diff"
