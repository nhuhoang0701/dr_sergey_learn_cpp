# Use vcpkg or Conan for C++ dependency management in CMake projects

**Category:** Tooling & Debugging  
**Item:** #425  
**Reference:** <https://vcpkg.io>  

---

## Topic Overview

This guide focuses on the **CMake integration** patterns for vcpkg and Conan, including presets, CI workflows, and cross-compilation.

```cpp

Dependency flow in CMake:

vcpkg:  vcpkg.json -> vcpkg install -> find_package() -> target_link_libraries()
Conan:  conanfile.txt -> conan install -> find_package() -> target_link_libraries()

Both produce CMake targets that work with standard find_package().

```

---

## Self-Assessment

### Q1: Full vcpkg + CMake project with presets

```json

// vcpkg.json
{
    "name": "webapp",
    "version-string": "1.0.0",
    "dependencies": [
        "boost-beast",
        "nlohmann-json",
        "openssl",
        "sqlite3"
    ]
}

```

```json

// CMakePresets.json — team-wide configuration
{
    "version": 6,
    "configurePresets": [
        {
            "name": "default",
            "generator": "Ninja",
            "binaryDir": "${sourceDir}/build",
            "cacheVariables": {
                "CMAKE_TOOLCHAIN_FILE": "$env{VCPKG_ROOT}/scripts/buildsystems/vcpkg.cmake",
                "CMAKE_CXX_STANDARD": "20",
                "CMAKE_EXPORT_COMPILE_COMMANDS": "ON"
            }
        },
        {
            "name": "release",
            "inherits": "default",
            "cacheVariables": {
                "CMAKE_BUILD_TYPE": "Release"
            }
        }
    ],
    "buildPresets": [
        {
            "name": "default",
            "configurePreset": "default"
        }
    ]
}

```

```cmake

# CMakeLists.txt
cmake_minimum_required(VERSION 3.20)
project(WebApp LANGUAGES CXX)

find_package(Boost REQUIRED COMPONENTS system)
find_package(nlohmann_json CONFIG REQUIRED)
find_package(OpenSSL REQUIRED)
find_package(unofficial-sqlite3 CONFIG REQUIRED)

add_executable(webapp src/main.cpp src/server.cpp src/db.cpp)
target_link_libraries(webapp PRIVATE
    Boost::system
    nlohmann_json::nlohmann_json
    OpenSSL::SSL OpenSSL::Crypto
    unofficial::sqlite3::sqlite3
)

```

```bash

# Build using presets:
$ cmake --preset default
$ cmake --build --preset default
# vcpkg automatically installs all dependencies

```

### Q2: Conan 2.x CMake integration

```ini

# conanfile.txt (simple projects)
[requires]
boost/1.84.0
fmt/10.2.1
spdlog/1.13.0

[generators]
CMakeDeps
CMakeToolchain

[layout]
cmake_layout

```

```bash

# Install dependencies:
$ conan install . --output-folder=build --build=missing

# This generates:
#   build/conan_toolchain.cmake     (compiler settings)
#   build/FindBoost.cmake           (CMake find modules)
#   build/fmt-config.cmake
#   build/spdlog-config.cmake

# Configure and build:
$ cmake -B build -DCMAKE_TOOLCHAIN_FILE=build/conan_toolchain.cmake
$ cmake --build build

```

```cmake

# CMakeLists.txt (same as vcpkg — uses standard find_package):
cmake_minimum_required(VERSION 3.20)
project(MyApp LANGUAGES CXX)

find_package(Boost REQUIRED)
find_package(fmt REQUIRED)
find_package(spdlog REQUIRED)

add_executable(app main.cpp)
target_link_libraries(app PRIVATE
    Boost::boost
    fmt::fmt
    spdlog::spdlog
)

```

```python

# conanfile.py (advanced: with options and settings)
from conan import ConanFile

class MyAppConan(ConanFile):
    settings = "os", "compiler", "build_type", "arch"
    requires = (
        "boost/1.84.0",
        "fmt/10.2.1",
    )
    generators = "CMakeDeps", "CMakeToolchain"

    def configure(self):
        # Only need Boost headers (no compiled libs):
        self.options["boost"].header_only = True

```

### Q3: CI workflow with dependency caching

```yaml

# .github/workflows/build.yml — vcpkg with binary caching
name: Build
on: [push, pull_request]

env:
  VCPKG_BINARY_SOURCES: 'clear;nuget,GitHub,readwrite'

jobs:
  build:
    runs-on: ubuntu-latest
    steps:

      - uses: actions/checkout@v4

        with:
          submodules: true  # if vcpkg is a submodule

      - name: Setup vcpkg

        uses: lukka/run-vcpkg@v11
        with:
          vcpkgGitCommitId: 'abc123...'  # pin version

      - name: Cache vcpkg packages

        uses: actions/cache@v4
        with:
          path: build/vcpkg_installed
          key: vcpkg-${{ hashFiles('vcpkg.json') }}-${{ runner.os }}

      - name: Configure

        run: cmake --preset default

      - name: Build

        run: cmake --build --preset default

      - name: Test

        run: ctest --test-dir build --output-on-failure

```

```yaml

# Conan CI with caching:

- name: Install Conan

  run: pip install conan

- name: Cache Conan packages

  uses: actions/cache@v4
  with:
    path: ~/.conan2
    key: conan-${{ hashFiles('conanfile.txt') }}-${{ runner.os }}

- name: Install dependencies

  run: conan install . --output-folder=build --build=missing

- name: Build

  run: |
    cmake -B build -DCMAKE_TOOLCHAIN_FILE=build/conan_toolchain.cmake
    cmake --build build

```

---

## Notes

- Use CMakePresets.json with vcpkg for team-wide reproducible builds.
- vcpkg binary caching reduces CI time dramatically (skip rebuilding deps).
- Conan profiles handle cross-compilation: `conan install . -pr:h=arm64`.
- Both tools support lockfiles for fully reproducible builds.
- Prefer manifest mode (vcpkg.json / conanfile.txt) over global installs.
