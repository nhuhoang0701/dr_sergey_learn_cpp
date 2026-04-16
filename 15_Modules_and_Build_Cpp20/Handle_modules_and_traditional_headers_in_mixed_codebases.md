# Handle Modules and Traditional Headers in Mixed Codebases

**Category:** Modules & Build (C++20)  
**Standard:** C++20  
**Reference:** [cppreference — Modules](https://en.cppreference.com/w/cpp/language/modules)  

---

## Topic Overview

Real-world C++ projects will exist in a mixed module/header state for years. Team-owned code migrates incrementally, third-party libraries remain header-based, and platform SDKs may never modularize. Mastering the patterns for mixed codebases is not a transitional skill — it is a permanent requirement for modern C++ development.

The three main bridging mechanisms are: (1) the **global module fragment** for consuming headers inside modules, (2) **header units** for importing headers without modification, and (3) **conditional compilation** with feature-test macros for code that must work both as a header and as a module. Each has distinct trade-offs around macro handling, build system integration, and portability.

| Mechanism | Modifies Original Header | Exports Macros | Build System Complexity |
| --- | --- | --- | --- |
| Global module fragment | No | No (trapped in fragment) | Low (just `#include`) |
| Header unit (`import <h>`) | No | Yes | Medium (needs HU scanning) |
| Dual-mode header/module | Yes (conditional) | Conditionally | High (two code paths) |
| Wrapper module | No (separate file) | No (converted) | Medium |

```cpp

┌───────────────────────────────────────────────────────────┐
│            Mixed Codebase Architecture                    │
│                                                           │
│  ┌──────────────┐  ┌──────────────┐  ┌────────────────┐  │
│  │  Your Modules │  │  Wrapper     │  │  Header-only   │  │
│  │  (native)     │  │  Modules     │  │  (unchanged)   │  │
│  │               │  │              │  │                │  │
│  │ export module │  │ module;      │  │ #pragma once   │  │
│  │   mylib;      │  │ #include ... │  │ #include ...   │  │
│  │               │  │ export module│  │                │  │
│  │               │  │   wrapper;   │  │                │  │
│  └───────┬───────┘  └───────┬──────┘  └───────┬────────┘  │
│          │                  │                  │           │
│          ▼                  ▼                  ▼           │
│  ┌─────────────────────────────────────────────────────┐  │
│  │              Application Code                       │  │
│  │  import mylib;                                      │  │
│  │  import wrapper;                                    │  │
│  │  #include "legacy_header.h"                         │  │
│  └─────────────────────────────────────────────────────┘  │
└───────────────────────────────────────────────────────────┘

```

The critical rule: **`#include` and `import` cannot be freely mixed for the same entity in the same TU**. A TU either includes a header or imports it (as a header unit or through a wrapper module). Mixing both for the same header causes ODR issues and is typically ill-formed in practice.

---

## Self-Assessment

### Q1: How do you wrap a header-only library (e.g., a math library) in a module while preserving full functionality

```cpp

// ---------- Third-party: glm/glm.hpp (header-only, not modifiable) ----------
// Contains: glm::vec3, glm::mat4, glm::translate(), macros like GLM_FORCE_RADIANS

// ---------- glm_module.cppm (wrapper module) ----------
module;                                   // global module fragment

// Set configuration macros BEFORE including
#define GLM_FORCE_RADIANS
#define GLM_FORCE_DEPTH_ZERO_TO_ONE
#include <glm/glm.hpp>
#include <glm/gtc/matrix_transform.hpp>
#include <glm/gtc/type_ptr.hpp>

export module glm_wrapper;

// Re-export the namespace and key types
export namespace glm {
    // Types
    using glm::vec2;
    using glm::vec3;
    using glm::vec4;
    using glm::mat3;
    using glm::mat4;
    using glm::quat;

    // Functions
    using glm::translate;
    using glm::rotate;
    using glm::scale;
    using glm::perspective;
    using glm::lookAt;
    using glm::normalize;
    using glm::cross;
    using glm::dot;
    using glm::value_ptr;
}

// Expose macro-configured constants as constexpr
export constexpr bool glm_force_radians = true;
export constexpr bool glm_depth_zero_to_one = true;

// ---------- consumer.cpp ----------
import glm_wrapper;

void setup_camera() {
    auto view = glm::lookAt(
        glm::vec3(0, 0, 5),
        glm::vec3(0, 0, 0),
        glm::vec3(0, 1, 0)
    );
    auto proj = glm::perspective(
        glm::radians(45.0f), 16.0f/9.0f, 0.1f, 100.0f
    );
    // GLM_FORCE_RADIANS  // NOT available — macro stayed in global module fragment
}

```

**Key points:** The global module fragment contains all `#include` and `#define` directives. After `export module glm_wrapper;`, you selectively re-export with `using` declarations. Macros configured before the include affect the library's behavior but do not leak to importers.

---

### Q2: How do you write a header that works both as a traditional `#include` and behind a module interface

```cpp

// ---------- mylib.h — dual-mode header ----------
#pragma once

// Feature-test macro: detect module context
#if defined(__cpp_modules) && defined(MYLIB_USE_MODULES)
// When used inside a module wrapper — no include guards needed,
// no includes (module handles them)
#define MYLIB_EXPORT export
#else
// Traditional header mode
#include <string>
#include <vector>
#include <memory>
#define MYLIB_EXPORT
#endif

namespace mylib {

MYLIB_EXPORT struct Config {
    std::string name;
    int max_connections = 100;
    bool verbose = false;
};

MYLIB_EXPORT class Service {
public:
    explicit Service(Config cfg);
    void start();
    void stop();
    bool is_running() const;
private:
    struct Impl;
    std::unique_ptr<Impl> impl_;
};

MYLIB_EXPORT std::vector<Config> load_configs(const std::string& path);

} // namespace mylib

// ---------- mylib_module.cppm — module wrapper when modules are available ----------
module;

#include <string>
#include <vector>
#include <memory>

#define MYLIB_USE_MODULES 1
#include "mylib.h"

export module mylib;

// Everything marked MYLIB_EXPORT in the header is now exported

// ---------- consumer_header.cpp — traditional usage ----------
#include "mylib.h"

void use_service() {
    mylib::Config cfg{"test", 50, true};
    mylib::Service svc(cfg);
    svc.start();
}

// ---------- consumer_module.cpp — module usage ----------
import mylib;

void use_service() {
    mylib::Config cfg{"test", 50, true};
    mylib::Service svc(cfg);
    svc.start();
}

```

**Dual-mode pattern summary:**

| Aspect | Header Mode | Module Mode |
| --- | --- | --- |
| `MYLIB_EXPORT` expands to | (empty) | `export` |
| Includes | In the header itself | In global module fragment |
| Macros from header | Available to consumer | Not available |
| ODR safety | Relies on include guards | Enforced by module system |

---

### Q3: How do you handle a dependency that uses macros for its API (e.g., assertion macros, logging macros) in a modular codebase

```cpp

// ---------- Problem: logging library uses macros ----------
// spdlog/spdlog.h defines: SPDLOG_INFO, SPDLOG_ERROR, SPDLOG_DEBUG, etc.
// These macros capture __FILE__, __LINE__ — cannot be replaced by functions

// ---------- Strategy: isolate macro usage in implementation units ----------

// logger.cppm (module interface — clean, no macros)
export module logger;

export enum class LogLevel { Debug, Info, Warn, Error };

export void log(LogLevel level, const char* file, int line, const char* msg);
export void log_fmt(LogLevel level, const char* file, int line, const char* fmt, ...);

// Provide a header with macros for call-site info capture
// logger_macros.h (companion header — NOT part of the module)
#pragma once
#include <utility>

// These macros must live in a header — modules cannot export macros
#define LOG_DEBUG(msg) ::logger_detail::log_debug(__FILE__, __LINE__, msg)
#define LOG_INFO(msg)  ::logger_detail::log_info(__FILE__, __LINE__, msg)
#define LOG_ERROR(msg) ::logger_detail::log_error(__FILE__, __LINE__, msg)

// Thin inline wrappers declared in header (not in module)
namespace logger_detail {
    // Forward-declared — defined in module implementation
    void log_debug(const char* file, int line, const char* msg);
    void log_info(const char* file, int line, const char* msg);
    void log_error(const char* file, int line, const char* msg);
}

// logger_impl.cpp (module implementation — contains spdlog)
module;
#include <spdlog/spdlog.h>       // macros contained in global module fragment
module logger;

void log(LogLevel level, const char* file, int line, const char* msg) {
    switch (level) {
        case LogLevel::Info:  SPDLOG_INFO("[{}:{}] {}", file, line, msg); break;
        case LogLevel::Error: SPDLOG_ERROR("[{}:{}] {}", file, line, msg); break;
        case LogLevel::Debug: SPDLOG_DEBUG("[{}:{}] {}", file, line, msg); break;
        default: break;
    }
}

// Also define the header-declared wrappers (external linkage)
namespace logger_detail {
    void log_debug(const char* f, int l, const char* m) { log(LogLevel::Debug, f, l, m); }
    void log_info(const char* f, int l, const char* m)  { log(LogLevel::Info, f, l, m); }
    void log_error(const char* f, int l, const char* m) { log(LogLevel::Error, f, l, m); }
}

// ---------- consumer.cpp ----------
import logger;                    // module: gets LogLevel, log()
#include "logger_macros.h"        // header: gets LOG_INFO etc.

void process() {
    log(LogLevel::Info, __FILE__, __LINE__, "explicit call");  // module API
    LOG_INFO("macro call");       // macro automatically captures file/line
}

```

**Pattern: module + companion header.** The module provides the core API; a thin companion header provides macros that need source-location capture. This is the accepted pattern for `__FILE__`/`__LINE__` dependent APIs. C++20's `std::source_location` can eventually eliminate the need for the companion header.

```cpp

// Future improvement with std::source_location (no macros needed):
export void log(LogLevel level, const char* msg,
                std::source_location loc = std::source_location::current());

```

---

## Notes

- **Global module fragment** (`module; ... export module M;`) is the primary tool for consuming headers inside modules. Everything before `export module` is in the fragment.
- Never `#include` a header and `import` it (or its module) in the same TU — this causes ODR ambiguity.
- **Macro-heavy APIs** (logging, assertions, serialization) need a **companion header** alongside the module until `std::source_location` and reflection eliminate macro needs.
- The dual-mode header pattern (`MYLIB_EXPORT` macro) allows gradual consumer migration — some consumers `#include`, others `import`.
- `__cpp_modules` is the feature-test macro for module support. Combine with a project-specific macro (e.g., `MYLIB_USE_MODULES`) for dual-mode control.
- Wrapper modules around third-party headers should re-export only what your project uses — this produces a smaller BMI and faster compilation.
- Implementation units (`module M;`) can freely `#include` any headers in the global module fragment — this does not affect the module's importers.
- Test your mixed codebase on all target compilers — module/header interaction is an area where compiler behavior still varies (as of 2024-2025).
- When wrapping template-heavy header-only libraries, ensure all needed specializations are instantiated or visible through the wrapper module's interface.
- Keep a clear naming convention: `.cppm` for module interface units, `.cpp` for implementation, `.h` for legacy headers and companion macro headers.
