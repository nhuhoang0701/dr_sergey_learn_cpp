# Use AI to generate cross-platform compatibility layers in C++

**Category:** AI-Assisted C++ Development

---

## Topic Overview

Writing cross-platform C++ code requires handling OS-specific APIs for file systems, networking, threading, process management, and GUI. AI assistants can generate **abstraction layers** that wrap platform-specific APIs behind a unified interface, generate platform detection macros, and translate between Windows/Linux/macOS APIs.

### Cross-Platform Areas and AI Assistance

| Area | Platform Differences | AI Help Level |
| --- | --- | --- |
| File system paths | `/` vs `\\`, case sensitivity | Excellent (use `<filesystem>`) |
| Shared libraries | dlopen vs LoadLibrary | Excellent |
| Networking sockets | POSIX vs Winsock | Good |
| Thread/process | pthreads vs Win32 threads | Good (use `<thread>`) |
| Memory mapping | mmap vs CreateFileMapping | Good |
| Console/terminal | ANSI vs Win32 Console API | Good |
| High-res timers | clock_gettime vs QueryPerformanceCounter | Excellent (use `<chrono>`) |

---

## Self-Assessment

### Q1: Generate a cross-platform shared library loader

**Answer:**

```cpp

=== PROMPT ===

"Generate a cross-platform dynamic library loader for
C++17 that works on Windows, Linux, and macOS.
Wrap dlopen/LoadLibrary, dlsym/GetProcAddress, etc.
Use RAII. Type-safe function pointer retrieval."

```

```cpp

// === AI-generated cross-platform shared library loader ===

#ifdef _WIN32
    #define WIN32_LEAN_AND_MEAN
    #include <Windows.h>
    using LibHandle = HMODULE;
#else
    #include <dlfcn.h>
    using LibHandle = void*;
#endif

class SharedLibrary {
    LibHandle handle_ = nullptr;
    std::string path_;

public:
    explicit SharedLibrary(const std::string& path)
        : path_(path) {
#ifdef _WIN32
        handle_ = LoadLibraryA(path.c_str());
#else
        handle_ = dlopen(path.c_str(), RTLD_NOW | RTLD_LOCAL);
#endif
        if (!handle_) {
            throw std::runtime_error(
                "Failed to load library: " + path + " - " + error());
        }
    }

    ~SharedLibrary() {
        if (handle_) {
#ifdef _WIN32
            FreeLibrary(handle_);
#else
            dlclose(handle_);
#endif
        }
    }

    SharedLibrary(const SharedLibrary&) = delete;
    SharedLibrary& operator=(const SharedLibrary&) = delete;
    SharedLibrary(SharedLibrary&& o) noexcept
        : handle_(std::exchange(o.handle_, nullptr))
        , path_(std::move(o.path_)) {}

    // Type-safe function retrieval
    template<typename Func>
    Func get_function(const std::string& name) const {
        static_assert(std::is_function_v<std::remove_pointer_t<Func>>,
            "Func must be a function pointer type");

        void* sym = nullptr;
#ifdef _WIN32
        sym = reinterpret_cast<void*>(
            GetProcAddress(handle_, name.c_str()));
#else
        sym = dlsym(handle_, name.c_str());
#endif
        if (!sym) {
            throw std::runtime_error(
                "Symbol not found: " + name + " in " + path_);
        }
        return reinterpret_cast<Func>(sym);
    }

    static std::string library_name(const std::string& base) {
#ifdef _WIN32
        return base + ".dll";
#elif defined(__APPLE__)
        return "lib" + base + ".dylib";
#else
        return "lib" + base + ".so";
#endif
    }

private:
    static std::string error() {
#ifdef _WIN32
        DWORD err = GetLastError();
        char* msg = nullptr;
        FormatMessageA(FORMAT_MESSAGE_ALLOCATE_BUFFER |
            FORMAT_MESSAGE_FROM_SYSTEM, nullptr, err, 0,
            reinterpret_cast<char*>(&msg), 0, nullptr);
        std::string result(msg ? msg : "Unknown error");
        LocalFree(msg);
        return result;
#else
        const char* err = dlerror();
        return err ? err : "Unknown error";
#endif
    }
};

// Usage:
auto lib = SharedLibrary(SharedLibrary::library_name("plugin"));
auto init = lib.get_function<int(*)(const char*)>("plugin_init");
int result = init("/path/to/config");

```

### Q2: Generate cross-platform memory-mapped file I/O

**Answer:**

```cpp

=== PROMPT ===

"Generate a cross-platform memory-mapped file class
for C++17. Support: read-only, read-write, and
copy-on-write modes. RAII with proper cleanup."

```

```cpp

// === AI-generated memory-mapped file ===

enum class MapMode { ReadOnly, ReadWrite, CopyOnWrite };

class MemoryMappedFile {
    void* data_ = nullptr;
    size_t size_ = 0;
#ifdef _WIN32
    HANDLE file_ = INVALID_HANDLE_VALUE;
    HANDLE mapping_ = nullptr;
#else
    int fd_ = -1;
#endif

public:
    MemoryMappedFile(const std::filesystem::path& path,
                     MapMode mode = MapMode::ReadOnly) {
        size_ = std::filesystem::file_size(path);
        if (size_ == 0) throw std::runtime_error("Empty file");

#ifdef _WIN32
        DWORD access = (mode == MapMode::ReadOnly)
            ? GENERIC_READ : (GENERIC_READ | GENERIC_WRITE);
        DWORD share = FILE_SHARE_READ;
        file_ = CreateFileW(path.c_str(), access, share,
            nullptr, OPEN_EXISTING, FILE_ATTRIBUTE_NORMAL, nullptr);
        if (file_ == INVALID_HANDLE_VALUE)
            throw std::system_error(GetLastError(),
                std::system_category(), "CreateFile failed");

        DWORD protect = PAGE_READONLY;
        DWORD map_access = FILE_MAP_READ;
        if (mode == MapMode::ReadWrite) {
            protect = PAGE_READWRITE;
            map_access = FILE_MAP_WRITE;
        } else if (mode == MapMode::CopyOnWrite) {
            protect = PAGE_WRITECOPY;
            map_access = FILE_MAP_COPY;
        }

        mapping_ = CreateFileMappingW(file_, nullptr, protect,
            0, 0, nullptr);
        if (!mapping_) {
            CloseHandle(file_);
            throw std::system_error(GetLastError(),
                std::system_category(), "CreateFileMapping");
        }
        data_ = MapViewOfFile(mapping_, map_access, 0, 0, 0);
#else
        int flags = O_RDONLY;
        int prot = PROT_READ;
        int mmap_flags = MAP_PRIVATE;

        if (mode == MapMode::ReadWrite) {
            flags = O_RDWR;
            prot = PROT_READ | PROT_WRITE;
            mmap_flags = MAP_SHARED;
        } else if (mode == MapMode::CopyOnWrite) {
            prot = PROT_READ | PROT_WRITE;
            mmap_flags = MAP_PRIVATE;  // Copy-on-write
        }

        fd_ = open(path.c_str(), flags);
        if (fd_ < 0)
            throw std::system_error(errno, std::generic_category());

        data_ = mmap(nullptr, size_, prot, mmap_flags, fd_, 0);
        if (data_ == MAP_FAILED) {
            close(fd_);
            throw std::system_error(errno, std::generic_category());
        }
#endif
    }

    ~MemoryMappedFile() {
#ifdef _WIN32
        if (data_) UnmapViewOfFile(data_);
        if (mapping_) CloseHandle(mapping_);
        if (file_ != INVALID_HANDLE_VALUE) CloseHandle(file_);
#else
        if (data_ && data_ != MAP_FAILED) munmap(data_, size_);
        if (fd_ >= 0) close(fd_);
#endif
    }

    MemoryMappedFile(const MemoryMappedFile&) = delete;
    MemoryMappedFile& operator=(const MemoryMappedFile&) = delete;

    std::span<const std::byte> data() const {
        return {static_cast<const std::byte*>(data_), size_};
    }
    std::span<std::byte> writable_data() {
        return {static_cast<std::byte*>(data_), size_};
    }
    size_t size() const { return size_; }
};

```

### Q3: Platform detection and feature macros

**Answer:**

```cpp

=== PROMPT ===

"Generate a platform_defs.hpp header with:

- OS detection (Windows, Linux, macOS, FreeBSD)
- Compiler detection (GCC, Clang, MSVC)
- Architecture detection (x86_64, ARM64, RISC-V)
- C++ standard detection
- Feature macros for SIMD, coroutines, modules

Use standard predefined macros only."

```

```cpp

// === AI-generated platform_defs.hpp ===
#pragma once

// --- OS Detection ---
#if defined(_WIN32) || defined(_WIN64)
    #define PLATFORM_WINDOWS 1
    #define PLATFORM_NAME "Windows"
#elif defined(__APPLE__) && defined(__MACH__)
    #define PLATFORM_MACOS 1
    #define PLATFORM_NAME "macOS"
#elif defined(__linux__)
    #define PLATFORM_LINUX 1
    #define PLATFORM_NAME "Linux"
#elif defined(__FreeBSD__)
    #define PLATFORM_FREEBSD 1
    #define PLATFORM_NAME "FreeBSD"
#else
    #define PLATFORM_UNKNOWN 1
    #define PLATFORM_NAME "Unknown"
#endif

// --- Compiler Detection ---
#if defined(_MSC_VER)
    #define COMPILER_MSVC 1
    #define COMPILER_VERSION _MSC_VER
#elif defined(__clang__)
    #define COMPILER_CLANG 1
    #define COMPILER_VERSION (__clang_major__ * 10000 + \
        __clang_minor__ * 100 + __clang_patchlevel__)
#elif defined(__GNUC__)
    #define COMPILER_GCC 1
    #define COMPILER_VERSION (__GNUC__ * 10000 + \
        __GNUC_MINOR__ * 100 + __GNUC_PATCHLEVEL__)
#endif

// --- Architecture ---
#if defined(__x86_64__) || defined(_M_X64)
    #define ARCH_X86_64 1
#elif defined(__aarch64__) || defined(_M_ARM64)
    #define ARCH_ARM64 1
#elif defined(__riscv) && __riscv_xlen == 64
    #define ARCH_RISCV64 1
#endif

// --- C++ Standard ---
#if __cplusplus >= 202302L
    #define CPP_STANDARD 23
#elif __cplusplus >= 202002L
    #define CPP_STANDARD 20
#elif __cplusplus >= 201703L
    #define CPP_STANDARD 17
#elif __cplusplus >= 201402L
    #define CPP_STANDARD 14
#endif

// --- Feature Detection ---
#if defined(__cpp_concepts) && __cpp_concepts >= 202002L
    #define HAS_CONCEPTS 1
#endif
#if defined(__cpp_coroutines) || defined(__cpp_impl_coroutine)
    #define HAS_COROUTINES 1
#endif
#if defined(__cpp_modules)
    #define HAS_MODULES 1
#endif

// --- SIMD Detection ---
#if defined(__SSE4_2__)
    #define HAS_SSE42 1
#endif
#if defined(__AVX2__)
    #define HAS_AVX2 1
#endif
#if defined(__AVX512F__)
    #define HAS_AVX512 1
#endif
#if defined(__ARM_NEON)
    #define HAS_NEON 1
#endif

```

---

## Notes

- Prefer **C++ standard library** (`<filesystem>`, `<thread>`, `<chrono>`) over OS-specific APIs when possible
- AI can translate between **Win32 and POSIX** APIs effectively — paste one platform, ask for the other
- Always test AI-generated platform code on **all target platforms** — AI often misses subtle differences
- For networking, consider **Boost.Asio** or **standalone Asio** over raw socket wrappers
- AI can generate **CMake platform detection** to complement C++ macros
- Use `std::filesystem::path` for path handling — it handles `/` vs `\\` automatically
- Ask AI for **CI/CD matrix** configurations to test across platforms
