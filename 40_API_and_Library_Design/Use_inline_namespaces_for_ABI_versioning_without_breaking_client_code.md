# Use inline namespaces for ABI versioning without breaking client code

**Category:** API & Library Design  
**Standard:** C++11/17  
**Reference:** <https://en.cppreference.com/w/cpp/language/namespace#Inline_namespaces>  

---

## Topic Overview

Inline namespaces let a library have multiple ABI versions coexist while keeping client code unchanged. The linker catches ABI mismatches as unresolved symbols instead of silent corruption.

### How It Works

```cpp

// mylib/version.hpp
namespace mylib {
    inline namespace v2 {  // "inline" makes v2 the default
        struct Widget {
            int x;
            int y;
            int z;  // Added in v2 — changes sizeof(Widget)!
        };
        void process(Widget& w);
    }

    namespace v1 {  // Old ABI — still available explicitly
        struct Widget {
            int x;
            int y;
        };
        void process(Widget& w);
    }
}

// Client code — no changes needed:
// mylib::Widget w;           // Resolves to mylib::v2::Widget
// mylib::process(w);         // Resolves to mylib::v2::process
//
// Client compiled against v1 looking for mylib::v1::Widget
// will get a LINKER ERROR (not silent ABI mismatch!)

```

### Name Mangling Makes It Safe

```cpp

// The mangled symbol names include the namespace:
// mylib::v1::process → _ZN5mylib2v17processERNS0_6WidgetE
// mylib::v2::process → _ZN5mylib2v27processERNS0_6WidgetE
//
// A v1-compiled client calls the v1 symbol.
// If linked against v2 library (which only exports v2 symbol):
// → LINKER ERROR: undefined reference to mylib::v1::process
// This is MUCH better than silently using the wrong struct layout!

```

### Practical Pattern

```cpp

// config.hpp — choose ABI version at build time
#if MYLIB_ABI_VERSION == 1
  #define MYLIB_ABI_NS v1
#elif MYLIB_ABI_VERSION == 2
  #define MYLIB_ABI_NS v2
#else
  #define MYLIB_ABI_NS v3
#endif

// api.hpp
namespace mylib {
    inline namespace MYLIB_ABI_NS {
        class Connection {
            // Layout changes between ABI versions
        public:
            void send(const char* data, size_t len);
            void close();
        };
    }
}
// Clients see: mylib::Connection
// Linker sees: mylib::v3::Connection

```

---

## Self-Assessment

### Q1: Show what happens when a client compiled against v1 links with a v2 library

The client's object files reference mangled symbols containing `v1` (e.g., `mylib::v1::Widget`). The v2 library exports symbols containing `v2`. The linker fails with "undefined reference" — a clear error. Without inline namespaces, the linker would succeed, but the v1 client would use the wrong struct layout, causing memory corruption.

### Q2: Can you have multiple inline namespaces active simultaneously

No. Only one version should be `inline` at a time (the "current" version). Other versions exist as non-inline namespaces that clients can use explicitly if they need backward compatibility: `mylib::v1::Widget`.

### Q3: How do the standard library implementations use inline namespaces

```cpp

// libstdc++ uses inline namespace __cxx11 for ABI-breaking changes:
namespace std {
    inline namespace __cxx11 {
        class string { /* SSO implementation */ };
    }
    // Old ABI: std::string
    // New ABI: std::__cxx11::string
    // -D_GLIBCXX_USE_CXX11_ABI=0 switches to old ABI
}

// libc++ uses inline namespace __1: std::__1::string
// MSVC STL uses inline namespaces for ABI version tracking

```

---

## Notes

- **Every shared library should use inline namespaces** for ABI versioning — it turns silent corruption into linker errors.
- The `inline` keyword on a namespace is unrelated to `inline` on functions.
- libstdc++'s `__cxx11` ABI break in GCC 5 was managed via inline namespaces.
- Combine with symbol visibility control for clean shared library interfaces.
