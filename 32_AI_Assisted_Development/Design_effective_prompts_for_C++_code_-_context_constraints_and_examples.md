# Design effective prompts for C++ code - context, constraints, and examples

**Category:** AI-Assisted C++ Development

---

## Topic Overview

Effective C++ prompts follow a **CCoE** structure: **Context** (standard, compiler, project constraints), **Constraints** (what to avoid, resource limits), and **Examples** (expected input/output, interface signatures). This structured approach produces dramatically better results than freeform requests, especially for C++ where correctness depends on subtle details.

### Prompt Component Impact

| Component | Without It | With It |
| --- | --- | --- |
| **C++ version** | May use deprecated features | Targets exact standard |
| **Compiler** | Generic code | Compiler-specific attrs/intrinsics |
| **Error handling** | Random mix of exceptions/codes | Consistent strategy |
| **Interface spec** | AI invents its own API | Matches existing codebase |
| **Examples** | May misunderstand intent | Nails the format/behavior |
| **Anti-requirements** | Includes unnecessary deps | Clean, focused output |

---

## Self-Assessment

### Q1: Build prompts for different C++ development scenarios

**Answer:**

```cpp

=== SCENARIO: New class implementation ===

CONTEXT:
  "C++20, GCC 13, Linux, project uses:

   - std::expected for error handling (no exceptions)
   - RAII for all resources
   - GoogleTest for testing
   - snake_case naming convention"

TASK:
  "Implement a memory-mapped file wrapper class."

INTERFACE (show exactly what you want):
  "class MappedFile {
   public:
     static std::expected<MappedFile, std::error_code>
       open(const std::filesystem::path& path, AccessMode mode);

     std::span<const std::byte> data() const;
     std::span<std::byte> mutable_data();  // only if writable
     size_t size() const;

     // Move-only (owns mmap handle)
     MappedFile(MappedFile&&) noexcept;
     MappedFile& operator=(MappedFile&&) noexcept;
     ~MappedFile();  // munmap in destructor
   };"

CONSTRAINTS:
  "- No copy construction/assignment

   - Use mmap()/munmap() directly, no Boost
   - Handle files >4GB (use size_t, not int)
   - Return std::error_code on failure (not throw)"

EXAMPLE USAGE:
  "auto file = MappedFile::open("data.bin", AccessMode::ReadOnly);
   if (file) {
     auto bytes = file->data();
     process(bytes);
   } else {
     log_error(file.error());
   }"


=== SCENARIO: Performance-critical code ===

CONTEXT:
  "C++17, Clang 16, x86-64. Hot path in trading system.
   Latency budget: <100ns per call. Called 10M times/sec."

TASK:
  "Implement a lock-free timestamp cache that stores the
   last N timestamps and returns the median."

CONSTRAINTS:
  "- No heap allocation in the hot path

   - No mutexes or locks
   - Cache-friendly: all data in one cache line if possible
   - Use std::atomic with relaxed ordering where safe
   - Compile with -O3 -march=native"

BENCHMARK TARGET:
  "Include a benchmark using Google Benchmark that proves
   <100ns per median() call with N=16."


=== SCENARIO: Template library ===

CONTEXT:
  "Header-only C++20 library. Must work with GCC 12+,
   Clang 15+, MSVC 19.34+."

TASK:
  "Implement a type-safe printf replacement using
   variadic templates and concepts."

CONSTRAINTS:
  "- Compile-time format string validation (consteval)

   - Concepts to constrain printable types
   - No <format> dependency (not available on all targets)
   - Support: int, double, string_view, pointer, bool
   - Reject unsupported types at compile time"

EXPECTED:
  "safe_printf("Name: {}, Age: {}", name, age);  // OK
   safe_printf("Value: {}", my_mutex);  // Compile error"

```

### Q2: Iterative prompt refinement workflow

**Answer:**

```cpp

=== FOUR-STEP REFINEMENT WORKFLOW ===

STEP 1: EXPLORE (for unfamiliar topics)
  "What are the approaches for implementing a work-stealing
   thread pool in C++? Compare:

   - std::deque per thread with lock
   - Lock-free Chase-Lev deque
   - std::jthread with stop_token

   List trade-offs: complexity, performance, correctness."

  [Review response, choose approach]

STEP 2: SCAFFOLD
  "Implement a work-stealing thread pool with Chase-Lev deques.
   Show only the class declaration and key method signatures.
   Include comments for thread-safety guarantees.
   Don't implement the bodies yet."

  [Review interface, adjust before full implementation]

STEP 3: IMPLEMENT
  "Now implement the full class based on this interface:
   [paste refined interface from Step 2]
   Focus on correctness:

   - Memory ordering for the Chase-Lev deque
   - Proper shutdown sequence
   - Exception handling for tasks

   Add inline comments for non-obvious synchronization."

  [Review implementation]

STEP 4: VERIFY
  "Review this implementation for:

   1. Data races (any shared mutable state without sync?)
   2. ABA problems in the lock-free deque
   3. Memory ordering correctness (too weak? too strong?)
   4. Exception safety (what if a task throws?)
   5. Shutdown correctness (can tasks be lost?)

   For each issue found, show the fix."


=== REFINEMENT PROMPTS ===

"The code you generated uses std::mutex. Refactor to use
std::atomic<> with the minimum memory ordering that's correct.
Explain why each ordering was chosen."

"The solution allocates in the hot path (line 42, new Task{}).
Refactor to use a pre-allocated object pool instead.
Capacity fixed at construction time."

"Add [[nodiscard]], [[likely]]/[[unlikely]], and
__builtin_expect hints where they'd help. Annotate noexcept
on all functions that don't throw."

```

### Q3: Few-shot prompting for consistent code style

**Answer:**

```cpp

=== FEW-SHOT: teach the LLM your project style ===

"My project follows this pattern for RAII wrappers.
Here's an existing example:

class FileDescriptor {
public:
    static std::expected<FileDescriptor, std::error_code>
    open(const char* path, int flags) {
        int fd = ::open(path, flags);
        if (fd < 0)
            return std::unexpected(std::error_code(
                errno, std::system_category()));
        return FileDescriptor(fd);
    }

    FileDescriptor(FileDescriptor&& other) noexcept
        : fd_(std::exchange(other.fd_, -1)) {}

    FileDescriptor& operator=(FileDescriptor&& other) noexcept {
        if (this != &other) {
            close();
            fd_ = std::exchange(other.fd_, -1);
        }
        return *this;
    }

    ~FileDescriptor() { close(); }

    int get() const noexcept { return fd_; }
    explicit operator bool() const noexcept { return fd_ >= 0; }

private:
    explicit FileDescriptor(int fd) : fd_(fd) {}
    void close() noexcept {
        if (fd_ >= 0) { ::close(fd_); fd_ = -1; }
    }

    FileDescriptor(const FileDescriptor&) = delete;
    FileDescriptor& operator=(const FileDescriptor&) = delete;

    int fd_ = -1;
};

Now write a similar RAII wrapper for a POSIX shared memory
region (shm_open/shm_unlink + mmap/munmap).
Follow the EXACT same patterns:

- Factory method returning std::expected
- Move-only
- Private constructor
- noexcept where possible
- std::exchange in move operations"

```

---

## Notes

- **Context block** should be reusable across prompts — save it in a system prompt or template
- **Interface-first prompts** produce better results than "implement X" — you control the API
- **Anti-requirements** ("no Boost", "no exceptions") prevent unwanted dependencies
- **Few-shot examples** from your own codebase teach consistent style better than descriptions
- Iterative refinement (scaffold → implement → verify) beats one-shot for complex code
- For templates/concepts: show expected compile-time errors, not just successes
- Always include **example usage** — it disambiguates intent better than any description
