# Generate documentation with Doxygen, Standardese, or hdoc

**Category:** API & Library Design  
**Standard:** C++17  
**Reference:** <https://www.doxygen.nl/>  

---

## Topic Overview

API documentation should be generated from source code comments. This ensures docs stay in sync with the code.

### Doxygen (Industry Standard)

```cpp

/// @brief Calculate the area of a circle.
/// @param radius The radius of the circle. Must be non-negative.
/// @return The area as a double.
/// @throws std::invalid_argument if radius is negative.
/// @note Uses M_PI from <cmath>.
///
/// @code
/// double a = circle_area(5.0);  // Returns ~78.54
/// @endcode
///
/// @see square_area(), ellipse_area()
/// @since v2.1
double circle_area(double radius);

/// @brief A thread-safe connection pool.
/// @tparam Connection The connection type (must be DefaultConstructible).
///
/// Usage:
/// @code
/// Pool<TcpConnection> pool(10);
/// auto conn = pool.acquire();
/// conn->send("hello");
/// pool.release(std::move(conn));
/// @endcode
template<typename Connection>
class Pool {
public:
    /// @brief Create a pool with the specified capacity.
    /// @param capacity Maximum number of connections.
    explicit Pool(size_t capacity);

    /// @brief Acquire a connection from the pool.
    /// @return A unique_ptr to a connection, or nullptr if pool is exhausted.
    [[nodiscard]] std::unique_ptr<Connection> acquire();

    /// @brief Return a connection to the pool.
    void release(std::unique_ptr<Connection> conn);
};

```

### Doxyfile Configuration

```ini

# Doxyfile essentials
PROJECT_NAME     = "MyLib"
OUTPUT_DIRECTORY = docs/
INPUT            = include/ src/
RECURSIVE        = YES
EXTRACT_ALL      = NO        # Only document commented entities
GENERATE_HTML    = YES
GENERATE_LATEX   = NO
USE_MDFILE_AS_MAINPAGE = README.md
WARN_IF_UNDOCUMENTED = YES  # Enforce documentation coverage

```

---

## Self-Assessment

### Q1: What are the most important Doxygen tags for API documentation

`@brief` (one-line summary), `@param` (parameter docs), `@return` (return value), `@throws` (exceptions), `@pre`/`@post` (pre/postconditions), `@note` (important notes), `@deprecated` (deprecation notice).

### Q2: Compare Doxygen vs Standardese vs hdoc

| Tool | Language | Output | C++20 Support |
| --- | --- | --- | --- |
| Doxygen | C/C++/Java | HTML, LaTeX, XML | Partial |
| Standardese | C++ only | Markdown, HTML | Better |
| hdoc | C++ only | HTML (modern) | Good |

### Q3: How to enforce documentation coverage in CI

```bash

# Run Doxygen with WARN_IF_UNDOCUMENTED = YES
# Parse warning output and fail CI if any undocumented public API
doxygen Doxyfile 2>&1 | grep "warning:" && exit 1

```

---

## Notes

- **Document the contract**: preconditions, postconditions, exception guarantees, thread safety.
- Use `@code`/`@endcode` for usage examples in every public function.
- Standardese generates documentation that reads like the C++ standard — excellent for library authors.
- hdoc produces modern, searchable HTML with full C++20 support.
