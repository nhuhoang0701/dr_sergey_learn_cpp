# Use LLM assistants effectively for C++ code generation and review

**Category:** AI-Assisted C++ Development

---

## Topic Overview

LLM assistants can dramatically accelerate C++ development when used strategically. They excel at **boilerplate generation**, **code review**, **explaining complex code**, and **exploring design alternatives**. They struggle with **subtle UB**, **complex template errors**, **memory model correctness**, and **ABI compatibility**. Knowing when to trust and when to verify is the key skill.

### LLM Strengths and Weaknesses for C++

| Task | LLM Effectiveness | Trust Level |
| --- | --- | --- |
| Boilerplate (RAII, operators) | Excellent | High (still verify) |
| Code explanation | Excellent | High |
| Algorithm implementation | Good | Medium (test thoroughly) |
| Design pattern application | Good | Medium |
| Concurrency code | Weak | Low (verify with TSan) |
| Template metaprogramming | Weak | Low (compile and test) |
| UB detection | Medium | Low (use sanitizers) |
| Platform-specific (SIMD, syscalls) | Medium | Low (test on target) |

---

## Self-Assessment

### Q1: Effective workflows for C++ code generation

**Answer:**

```cpp

=== WORKFLOW 1: GENERATE + REVIEW + REFINE ===

Step 1 - Generate with constraints:
  "Implement a connection pool for PostgreSQL using libpq.
   C++20, max 16 connections, thread-safe, RAII.
   Return std::expected<Connection, Error>."

Step 2 - Self-review prompt:
  "Review the code you just generated. Check for:

   - Thread safety: are all shared resources protected?
   - Exception safety: what happens if libpq throws?
   - Resource leaks: is every PGconn* freed on all paths?
   - Performance: is there unnecessary lock contention?"

Step 3 - Address findings:
  "Fix issue #2: the PGconn* leaks if connection
   validation fails after acquisition. Use a scope guard."


=== WORKFLOW 2: EXPLAIN BEFORE GENERATE ===

  "I need to choose between:
   a) std::shared_mutex for read-heavy concurrent map
   b) Lock-free robin hood hash map
   c) Per-shard std::unordered_map with striped locks

   My workload: 95% reads, 5% writes, 4 threads, 100K entries.
   Which approach and why? Then implement your recommendation."


=== WORKFLOW 3: TEST-FIRST ===

  "Write GoogleTest test cases for a CircularBuffer<T, N>:

   - push/pop FIFO ordering
   - full buffer overwrites oldest
   - empty buffer pop returns nullopt
   - capacity() returns N
   - works with move-only types

   Then implement CircularBuffer to pass these tests."

```

```cpp

// === Code review prompt that catches real bugs ===
// Given this code, ask the LLM:

class SessionManager {
    std::unordered_map<int, std::shared_ptr<Session>> sessions_;
    std::mutex mtx_;
public:
    void add(int id, std::shared_ptr<Session> s) {
        std::lock_guard lock(mtx_);
        sessions_[id] = std::move(s);
    }

    std::shared_ptr<Session> get(int id) {
        std::lock_guard lock(mtx_);
        auto it = sessions_.find(id);
        return it != sessions_.end() ? it->second : nullptr;
    }

    void remove_expired() {
        std::lock_guard lock(mtx_);
        for (auto it = sessions_.begin(); it != sessions_.end(); ) {
            if (it->second->expired())
                it = sessions_.erase(it);
            else
                ++it;
        }
    }
};

// Prompt: "Review for thread safety, performance, and correctness.
//  Specifically: is the lock granularity correct? Any performance
//  issues with the mutex? What about the shared_ptr ref counting?"

// Good LLM response would identify:
// 1. remove_expired() holds lock while calling expired() - if
//    expired() is slow, all other threads are blocked
// 2. get() returns shared_ptr while lock is held - OK, but
//    the Session could expire right after return
// 3. Consider shared_mutex for read-heavy workload
// 4. remove_expired() could be called from a background thread
//    on a snapshot to reduce lock contention

```

### Q2: Code review with LLMs - what to check

**Answer:**

```text

=== STRUCTURED CODE REVIEW PROMPT ===

"Review this C++ code for the following categories.
For each issue, rate severity (Critical/Warning/Info)
and show the fix:

1. CORRECTNESS
   - Undefined behavior (lifetime, aliasing, overflow)
   - Logic errors
   - Off-by-one errors
   - Integer truncation/promotion issues

2. SAFETY
   - Resource leaks
   - Exception safety guarantee (basic/strong/nothrow)
   - Null pointer dereference paths
   - Buffer overflows

3. CONCURRENCY (if applicable)
   - Data races
   - Deadlock potential
   - Memory ordering correctness
   - TOCTOU (time-of-check-time-of-use)

4. PERFORMANCE
   - Unnecessary copies
   - Missing move semantics
   - Suboptimal container choice
   - Cache-unfriendly access patterns

5. STYLE / MAINTAINABILITY
   - Missing const
   - Missing [[nodiscard]]
   - Unclear ownership semantics
   - Magic numbers

[paste code here]"


=== DIFFERENTIAL REVIEW (code change) ===

"Here's a diff for a pull request. Review the CHANGES ONLY
(not the entire file). Focus on:

- Does the change introduce bugs?
- Are edge cases handled?
- Is the change consistent with surrounding code?
- Any ABI break if this is a public library?

[paste diff]"

```

### Q3: Knowing when NOT to trust LLM output

**Answer:**

```cpp

=== HIGH-RISK AREAS: ALWAYS VERIFY MANUALLY ===

1. LOCK-FREE / ATOMIC CODE:

   LLMs frequently get memory ordering wrong.
   -> Always use ThreadSanitizer (TSan)
   -> Run stress tests (millions of iterations)
   -> Verify on ARM (weaker memory model than x86)

2. TEMPLATE METAPROGRAMMING:

   Complex SFINAE/concept interactions often wrong.
   -> Must compile-test with multiple compilers
   -> Verify error messages for invalid types

3. UNDEFINED BEHAVIOR:

   LLMs may generate code that works on one platform
   but is technically UB (strict aliasing, signed overflow).
   -> Always compile with -fsanitize=undefined
   -> Test with optimization levels -O0 and -O3

4. PLATFORM-SPECIFIC CODE:

   System calls, SIMD intrinsics, compiler extensions.
   -> Test on the actual target platform
   -> Cross-reference with man pages / Intel intrinsics guide

5. SECURITY-SENSITIVE CODE:

   Crypto, input validation, privilege handling.
   -> Never trust AI-generated crypto
   -> Use established libraries (OpenSSL, libsodium)
   -> Fuzz-test all input parsing

=== VERIFICATION PIPELINE ===

Generate → Compile (-Wall -Wextra -Werror)
        → Static analysis (clang-tidy, cppcheck)
        → Sanitizers (ASan + UBSan + TSan)
        → Unit tests (generated or manual)
        → Code review (human for high-risk areas)

```

---

## Notes

- Use LLMs for the **first draft**, not the **final version** — always review and test
- **Explain-then-implement** prompts produce better results than direct "write X" prompts
- For complex code, **generate tests first** then implementation — tests constrain the LLM
- LLMs are excellent at **code explanation** — use them to understand unfamiliar codebases
- Always verify with **sanitizers** (`-fsanitize=address,undefined,thread`)
- **Few-shot examples** from your codebase teach style better than verbal descriptions
- The best use of LLMs is **pair programming**: you design, LLM implements, you verify
