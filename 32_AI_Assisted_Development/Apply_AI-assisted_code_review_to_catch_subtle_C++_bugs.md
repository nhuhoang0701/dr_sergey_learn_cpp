# Apply AI-assisted code review to catch subtle C++ bugs

**Category:** AI-Assisted C++ Development

---

## Topic Overview

AI-assisted code review augments human reviewers by systematically checking for patterns that are easy to miss: **lifetime issues**, **exception safety gaps**, **integer promotion bugs**, **incorrect const usage**, and **thread-safety violations**. The AI reviewer is tireless and consistent, while the human reviewer focuses on design, architecture, and business logic correctness.

### AI vs Human Review Strengths

| Bug Category | Human | AI | Best Approach |
| --- | --- | --- | --- |
| Undefined behavior | Sometimes misses | Good pattern matching | AI first, human verifies |
| Logic errors | Good | Medium | Human primary |
| Thread safety | Often misses | Good at data race patterns | AI + TSan |
| Design quality | Excellent | Weak | Human primary |
| Missing error handling | Sometimes misses | Systematic | AI primary |
| Performance issues | Context-dependent | Pattern-based | Both |
| API misuse | Context-dependent | Good if trained | AI primary |

---

## Self-Assessment

### Q1: Structure AI code review prompts for C++ pull requests

**Answer:**

```cpp

=== SYSTEMATIC C++ REVIEW PROMPT ===

"Review this C++ code change. For each finding, provide:

- Line number
- Severity: CRITICAL / WARNING / SUGGESTION
- Category: correctness / safety / performance / style
- Description of the issue
- Suggested fix with code

Check specifically for:

LIFETIME ISSUES:

- Dangling references/pointers (returning ref to local,

  ref to temporary, iterator invalidation)

- Use after move
- Pointers to stack variables escaping scope

UNDEFINED BEHAVIOR:

- Signed integer overflow
- Null pointer dereference on any code path
- Out-of-bounds access
- Uninitialized variable reads
- Strict aliasing violations (reinterpret_cast misuse)

RESOURCE MANAGEMENT:

- Raw new/delete without RAII wrapper
- File handles, sockets, mutexes not RAII-managed
- Exception safety: what leaks if line X throws?

CONCURRENCY:

- Shared mutable state without synchronization
- Lock ordering (potential deadlock)
- Condition variable spurious wakeup not handled
- atomic operations with wrong memory ordering

[paste code diff here]"

```

```cpp

// === Example: AI review catches subtle bugs ===

// Pull request code:
class ConnectionPool {
    std::vector<std::unique_ptr<Connection>> idle_;
    std::mutex mtx_;
public:
    Connection* acquire() {
        std::lock_guard lock(mtx_);
        if (idle_.empty()) {
            return new Connection(config_);  // AI: raw new, leak risk
        }
        auto conn = std::move(idle_.back());
        idle_.pop_back();
        return conn.release();  // AI: caller must delete, error-prone
    }

    void release(Connection* conn) {
        std::lock_guard lock(mtx_);
        idle_.emplace_back(conn);
    }
};

// AI review output:
// CRITICAL [line 8]: Raw new without RAII. If caller forgets
//   to call release(), connection leaks.
//   Fix: Return std::unique_ptr<Connection> with custom deleter
//   that returns to pool.
//
// WARNING [line 13]: Connection* ownership unclear.
//   No documentation on who owns the pointer.
//   Fix: Use unique_ptr with pool-returning deleter.
//
// SUGGESTION: Consider returning a RAII guard:

class PooledConnection {
public:
    PooledConnection(std::unique_ptr<Connection> conn,
                     ConnectionPool& pool)
        : conn_(std::move(conn)), pool_(pool) {}

    ~PooledConnection() {
        if (conn_) pool_.release(std::move(conn_));
    }

    Connection& operator*() { return *conn_; }
    Connection* operator->() { return conn_.get(); }

private:
    std::unique_ptr<Connection> conn_;
    ConnectionPool& pool_;
};

```

### Q2: Detect thread-safety issues with AI review

**Answer:**

```cpp

// === Code submitted for review ===
class MetricsCollector {
    struct Counter {
        std::string name;
        int64_t value = 0;
    };

    std::vector<Counter> counters_;  // Shared state
    std::mutex mtx_;

public:
    void increment(const std::string& name) {
        std::lock_guard lock(mtx_);
        for (auto& c : counters_) {
            if (c.name == name) {
                c.value++;
                return;
            }
        }
        counters_.push_back({name, 1});
    }

    // BUG: No lock! Data race with increment()
    int64_t get(const std::string& name) const {
        for (const auto& c : counters_) {
            if (c.name == name)
                return c.value;
        }
        return 0;
    }

    // BUG: Returns copy of internal state, but the copy
    // itself is not atomic - could see partial updates
    std::vector<Counter> snapshot() const {
        return counters_;  // No lock!
    }
};

// AI review catches:
// CRITICAL [line 23]: get() reads counters_ without lock.
//   Data race with increment() which modifies counters_.
//   Fix: Add std::lock_guard lock(mtx_);
//   Note: mtx_ must be declared mutable for const methods.
//
// CRITICAL [line 30]: snapshot() copies vector without lock.
//   Data race: another thread may be inserting while copying.
//   Fix: Lock, then copy.
//
// WARNING: Linear search O(n) per increment.
//   Consider std::unordered_map for O(1) lookup.
//
// SUGGESTION: Use std::shared_mutex for read-heavy workload:
//   - get()/snapshot(): shared_lock (multiple concurrent reads)
//   - increment(): unique_lock (exclusive write)

// FIXED VERSION:
class MetricsCollector {
    std::unordered_map<std::string, std::atomic<int64_t>> counters_;
    mutable std::shared_mutex mtx_;

public:
    void increment(const std::string& name) {
        std::shared_lock lock(mtx_);
        auto it = counters_.find(name);
        if (it != counters_.end()) {
            it->second.fetch_add(1, std::memory_order_relaxed);
            return;
        }
        lock.unlock();
        std::unique_lock wlock(mtx_);
        counters_[name].fetch_add(1, std::memory_order_relaxed);
    }

    int64_t get(const std::string& name) const {
        std::shared_lock lock(mtx_);
        auto it = counters_.find(name);
        return it != counters_.end()
            ? it->second.load(std::memory_order_relaxed) : 0;
    }
};

```

### Q3: Pre-commit AI review hook

**Answer:**

```cpp

=== PRE-COMMIT REVIEW CHECKLIST (automated) ===

For each changed .cpp/.hpp file, verify:

1. OWNERSHIP CLARITY

   [ ] Every raw pointer has clear ownership documentation
   [ ] No raw new/delete (use make_unique/make_shared)
   [ ] All resources have RAII wrappers

2. CONST CORRECTNESS

   [ ] All methods that don't modify state are const
   [ ] Parameters passed by const reference where appropriate
   [ ] Return const reference for non-owning access

3. EXCEPTION SAFETY

   [ ] Constructors: no partial construction possible
   [ ] Destructors: noexcept (implicit or explicit)
   [ ] Swap operations: noexcept
   [ ] Move operations: noexcept

4. THREAD SAFETY

   [ ] All shared mutable state protected by mutex
   [ ] No lock held during callbacks or user code
   [ ] Condition variables use predicate form (while loop)
   [ ] No TOCTOU (time-of-check-time-of-use) patterns

5. MODERN C++ USAGE

   [ ] Using structured bindings where appropriate
   [ ] Using auto for complex iterator types
   [ ] Using string_view for non-owning string parameters
   [ ] Using [[nodiscard]] for functions returning values

```

---

## Notes

- AI review is a **supplement**, not replacement, for human review
- The **diff-only** review ("review only the changes") is more effective than reviewing entire files
- AI excels at **systematic checking** that humans find tedious (const, nodiscard, error paths)
- Always validate AI findings — false positives are common, especially for "thread-safety" warnings
- Best workflow: AI review first → author addresses findings → human review for design/logic
- Thread-safety bugs found by AI review should be **confirmed with TSan**
- Consider integrating AI review as a CI check on pull requests
