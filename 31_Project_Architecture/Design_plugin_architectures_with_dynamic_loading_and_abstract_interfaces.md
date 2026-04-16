# Design plugin architectures with dynamic loading and abstract interfaces

**Category:** Project Architecture

---

## Topic Overview

**Plugin architecture** allows extending an application at runtime by loading shared libraries (`.so`/`.dll`) that implement a known interface. The host application defines abstract interfaces; plugins implement them and export a factory function. This enables third-party extensibility, modular deployment, and optional features without recompilation.

### Plugin Architecture Components

| Component | Role | Example |
| --- | --- | --- |
| **Host** | Loads plugins, manages lifecycle | Application executable |
| **Plugin Interface** | Abstract classes plugins must implement | `ICodecPlugin`, `IFilter` |
| **Plugin** | Shared library implementing the interface | `libmp3_codec.so` |
| **Factory Function** | C-linkage entry point for plugin creation | `create_plugin()` |
| **Plugin Manager** | Discovery, loading, version checking | Scans plugin directory |

---

## Self-Assessment

### Q1: Implement a cross-platform plugin loading system

**Answer:**

```cpp

#include <memory>
#include <string>
#include <vector>
#include <filesystem>
#include <iostream>

// === Plugin Interface (shared header between host and plugins) ===
// plugin_api.h
class IPlugin {
public:
    virtual ~IPlugin() = default;
    virtual const char* name() const = 0;
    virtual int version() const = 0;
    virtual bool initialize() = 0;
    virtual void shutdown() = 0;
};

// Every plugin exports these C functions:
extern "C" {
    using CreatePluginFn = IPlugin*(*)();
    using DestroyPluginFn = void(*)(IPlugin*);
}

// === Cross-platform dynamic library wrapper ===
class DynamicLibrary {
public:
    explicit DynamicLibrary(const std::filesystem::path& path) {
#ifdef _WIN32
        handle_ = LoadLibraryW(path.wstring().c_str());
#else
        handle_ = dlopen(path.c_str(), RTLD_LAZY);
#endif
        if (!handle_)
            throw std::runtime_error("Failed to load: " + path.string());
    }

    ~DynamicLibrary() {
        if (handle_) {
#ifdef _WIN32
            FreeLibrary(static_cast<HMODULE>(handle_));
#else
            dlclose(handle_);
#endif
        }
    }

    template<typename Fn>
    Fn get_function(const char* name) const {
#ifdef _WIN32
        auto fn = GetProcAddress(static_cast<HMODULE>(handle_), name);
#else
        auto fn = dlsym(handle_, name);
#endif
        return reinterpret_cast<Fn>(fn);
    }

    DynamicLibrary(DynamicLibrary&&) = default;
    DynamicLibrary& operator=(DynamicLibrary&&) = default;
    DynamicLibrary(const DynamicLibrary&) = delete;

private:
    void* handle_ = nullptr;
};

// === Plugin Manager ===
class PluginManager {
public:
    struct LoadedPlugin {
        DynamicLibrary library;
        IPlugin* instance;
        DestroyPluginFn destroy;
    };

    void scan_directory(const std::filesystem::path& dir) {
        for (const auto& entry : std::filesystem::directory_iterator(dir)) {
            auto ext = entry.path().extension();
            if (ext == ".so" || ext == ".dll" || ext == ".dylib") {
                load_plugin(entry.path());
            }
        }
    }

    void load_plugin(const std::filesystem::path& path) {
        try {
            DynamicLibrary lib(path);
            auto create = lib.get_function<CreatePluginFn>("create_plugin");
            auto destroy = lib.get_function<DestroyPluginFn>("destroy_plugin");

            if (!create || !destroy) {
                std::cerr << "Invalid plugin: " << path << "\n";
                return;
            }

            IPlugin* plugin = create();
            if (plugin->version() < MIN_API_VERSION) {
                destroy(plugin);
                return;
            }

            plugin->initialize();
            plugins_.push_back({std::move(lib), plugin, destroy});
            std::cout << "Loaded: " << plugin->name()
                      << " v" << plugin->version() << "\n";
        } catch (const std::exception& e) {
            std::cerr << "Error loading " << path << ": " << e.what() << "\n";
        }
    }

    ~PluginManager() {
        // Shutdown in reverse order
        for (auto it = plugins_.rbegin(); it != plugins_.rend(); ++it) {
            it->instance->shutdown();
            it->destroy(it->instance);
        }
    }

    static constexpr int MIN_API_VERSION = 1;
private:
    std::vector<LoadedPlugin> plugins_;
};

```

### Q2: Write a concrete plugin as a shared library

**Answer:**

```cpp

// === mp3_codec_plugin.cpp (compiled to libmp3_codec.so) ===
#include "plugin_api.h"
#include <iostream>

class Mp3CodecPlugin : public IPlugin {
public:
    const char* name() const override { return "MP3 Codec"; }
    int version() const override { return 2; }

    bool initialize() override {
        std::cout << "MP3 codec initialized\n";
        return true;
    }

    void shutdown() override {
        std::cout << "MP3 codec shutdown\n";
    }
};

// === C-linkage factory functions ===
extern "C" {

#ifdef _WIN32
__declspec(dllexport)
#else
__attribute__((visibility("default")))
#endif
IPlugin* create_plugin() {
    return new Mp3CodecPlugin();
}

#ifdef _WIN32
__declspec(dllexport)
#else
__attribute__((visibility("default")))
#endif
void destroy_plugin(IPlugin* p) {
    delete p;
}

}  // extern "C"

// Build: g++ -shared -fPIC -fvisibility=hidden mp3_codec_plugin.cpp
//        -o libmp3_codec.so

```

### Q3: Implement version negotiation and safe ABI boundaries

**Answer:**

```cpp

// === Safe ABI: use C-compatible structures at the boundary ===

// plugin_abi.h — frozen ABI, never change existing fields
struct PluginInfo {
    int struct_size;        // For forward compatibility
    int api_version;
    const char* name;
    const char* description;
    const char* author;
};

// Functions use C types only at the boundary
extern "C" {
    // Plugin fills in PluginInfo; host checks api_version
    using GetPluginInfoFn = bool(*)(PluginInfo* info);

    // Host passes opaque context; plugin stores it
    using InitPluginFn = bool(*)(void* host_context);
    using ShutdownPluginFn = void(*)();
}

// === Version negotiation in host ===
bool negotiate_version(DynamicLibrary& lib) {
    auto get_info = lib.get_function<GetPluginInfoFn>("get_plugin_info");
    if (!get_info) return false;

    PluginInfo info{};
    info.struct_size = sizeof(PluginInfo);
    if (!get_info(&info)) return false;

    // Check struct_size for forward compatibility
    if (info.struct_size < static_cast<int>(offsetof(PluginInfo, api_version)

                                          + sizeof(int)))

        return false;  // Too old, missing api_version field

    if (info.api_version < MIN_API_VERSION ||
        info.api_version > MAX_API_VERSION)
        return false;  // Incompatible version

    return true;
}

// Key rules for ABI safety:
// 1. NEVER pass std::string, std::vector across DLL boundary
// 2. Use C types (const char*, int, struct with C layout)
// 3. Allocate and free on the SAME side of the boundary
// 4. Use struct_size field for forward compatibility
// 5. Never remove or reorder fields in published structs

```

---

## Notes

- **C-linkage is mandatory** for factory functions: `extern "C"` prevents name mangling
- **Never pass C++ types across DLL boundaries**: `std::string`, `std::vector`, exceptions — ABI incompatible
- Memory rule: whoever allocates must deallocate (hence `destroy_plugin`)
- Use `-fvisibility=hidden` and explicitly export only factory symbols
- `struct_size` field enables forward compatibility without breaking old plugins
- Plugin shutdown must happen before `dlclose`/`FreeLibrary`
- Consider using `std::shared_ptr` with custom deleter to pair `create`/`destroy` automatically
