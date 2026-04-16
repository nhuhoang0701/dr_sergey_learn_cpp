# Use package managers (Conan, vcpkg) in project architecture

**Category:** Project Architecture

---

## Topic Overview

**Package managers** automate dependency installation, version resolution, and build integration. **vcpkg** (Microsoft) and **Conan** (JFrog) are the two dominant C++ package managers. Both solve the "dependency hell" problem but with different philosophies: vcpkg integrates tightly with CMake; Conan is more flexible and works with any build system.

### vcpkg vs Conan

| Aspect | vcpkg | Conan |
| --- | --- | --- |
| Integration | CMake-native (toolchain file) | Generators for CMake, Meson, etc. |
| Version mgmt | manifest mode (vcpkg.json) | conanfile.txt/conanfile.py |
| Binary caching | Built-in | Built-in + remote |
| Custom packages | Overlay ports | conanfile.py recipes |
| CI integration | Simple | Flexible |
| Package count | ~2500 | ~1700 (ConanCenter) |

---

## Self-Assessment

### Q1: Set up vcpkg with manifest mode in a project

**Answer:**

```json

// === vcpkg.json (project manifest) ===
{
  "name": "my-project",
  "version-semver": "1.0.0",
  "dependencies": [
    "fmt",
    "spdlog",
    {
      "name": "boost-asio",
      "version>=": "1.83.0"
    },
    {
      "name": "gtest",
      "host": true
    },
    {
      "name": "protobuf",
      "features": ["zlib"]
    }
  ],
  "overrides": [
    { "name": "fmt", "version": "10.1.1" }
  ],
  "builtin-baseline": "a1a1cbc975528dd2758c143091a4b3246af5e221"
}

```

```cmake

# === CMakeLists.txt ===
cmake_minimum_required(VERSION 3.25)
project(my-project)

# vcpkg integrates via toolchain file:
# cmake -B build -DCMAKE_TOOLCHAIN_FILE=$VCPKG_ROOT/scripts/buildsystems/vcpkg.cmake

# Or set in CMakePresets.json:
# "toolchainFile": "$env{VCPKG_ROOT}/scripts/buildsystems/vcpkg.cmake"

find_package(fmt CONFIG REQUIRED)
find_package(spdlog CONFIG REQUIRED)
find_package(Boost REQUIRED COMPONENTS system)
find_package(GTest CONFIG REQUIRED)
find_package(Protobuf CONFIG REQUIRED)

add_executable(app src/main.cpp)
target_link_libraries(app PRIVATE
    fmt::fmt
    spdlog::spdlog
    Boost::system
    protobuf::libprotobuf
)

```

```json

// === CMakePresets.json ===
{
  "version": 6,
  "configurePresets": [
    {
      "name": "default",
      "generator": "Ninja",
      "binaryDir": "build",
      "toolchainFile": "$env{VCPKG_ROOT}/scripts/buildsystems/vcpkg.cmake",
      "cacheVariables": {
        "CMAKE_BUILD_TYPE": "Release"
      }
    }
  ]
}

```

### Q2: Set up Conan 2.x with CMake integration

**Answer:**

```ini

# === conanfile.txt (simple) ===
[requires]
fmt/10.1.1
spdlog/1.12.0
boost/1.83.0
protobuf/3.21.12

[generators]
CMakeDeps
CMakeToolchain

[options]
boost/*:without_test=True
boost/*:without_python=True

```

```python

# === conanfile.py (advanced) ===
from conan import ConanFile
from conan.tools.cmake import CMake, cmake_layout

class MyProject(ConanFile):
    name = "my-project"
    version = "1.0.0"
    settings = "os", "compiler", "build_type", "arch"
    generators = "CMakeDeps", "CMakeToolchain"

    def requirements(self):
        self.requires("fmt/10.1.1")
        self.requires("spdlog/1.12.0")
        self.requires("boost/1.83.0")
        if self.settings.os != "baremetal":
            self.requires("protobuf/3.21.12")

    def build_requirements(self):
        self.test_requires("gtest/1.14.0")

    def layout(self):
        cmake_layout(self)

    def build(self):
        cmake = CMake(self)
        cmake.configure()
        cmake.build()

    def package(self):
        cmake = CMake(self)
        cmake.install()

```

```bash

# Build with Conan 2.x:
conan install . --output-folder=build --build=missing
cmake --preset conan-release
cmake --build --preset conan-release

```

### Q3: Create a custom package/port

**Answer:**

```python

# === Conan: custom recipe for internal library ===
# recipes/mylib/conanfile.py
from conan import ConanFile
from conan.tools.cmake import CMake, cmake_layout
from conan.tools.files import copy

class MyLibConan(ConanFile):
    name = "mylib"
    version = "2.0.0"
    settings = "os", "compiler", "build_type", "arch"
    generators = "CMakeDeps", "CMakeToolchain"
    exports_sources = "CMakeLists.txt", "src/*", "include/*"

    def requirements(self):
        self.requires("fmt/10.1.1")

    def layout(self):
        cmake_layout(self)

    def build(self):
        cmake = CMake(self)
        cmake.configure()
        cmake.build()

    def package(self):
        copy(self, "*.h", src=os.path.join(self.source_folder, "include"),
             dst=os.path.join(self.package_folder, "include"))
        copy(self, "*.a", src=self.build_folder,
             dst=os.path.join(self.package_folder, "lib"), keep_path=False)
        copy(self, "*.so", src=self.build_folder,
             dst=os.path.join(self.package_folder, "lib"), keep_path=False)

    def package_info(self):
        self.cpp_info.libs = ["mylib"]

# Upload to private remote:
# conan create recipes/mylib/
# conan upload mylib/2.0.0 -r=my-artifactory --all

```

```cmake

# === vcpkg: custom overlay port ===
# ports/mylib/portfile.cmake
vcpkg_from_github(
    OUT_SOURCE_PATH SOURCE_PATH
    REPO myorg/mylib
    REF v2.0.0
    SHA512 abc123...
)

vcpkg_cmake_configure(SOURCE_PATH "${SOURCE_PATH}")
vcpkg_cmake_install()
vcpkg_cmake_config_fixup(CONFIG_PATH lib/cmake/mylib)
file(REMOVE_RECURSE "${CURRENT_PACKAGES_DIR}/debug/include")

# ports/mylib/vcpkg.json
# { "name": "mylib", "version": "2.0.0",
#   "dependencies": ["fmt"] }

# Use: vcpkg install mylib --overlay-ports=./ports

```

---

## Notes

- **vcpkg manifest mode** (vcpkg.json) is preferred over classic mode — version-locks dependencies per project
- **Conan 2.x** is a major redesign; don't follow Conan 1.x tutorials
- Both support **binary caching**: avoid rebuilding deps in CI (vcpkg: `VCPKG_BINARY_SOURCES`, Conan: `conan remote`)
- Pin dependency versions — never rely on "latest" in production
- For internal libraries, use Conan private remotes or vcpkg overlay ports
- `CMakePresets.json` works seamlessly with both vcpkg and Conan for reproducible builds
- Consider `FetchContent` for small, header-only dependencies where a full package manager is overkill
