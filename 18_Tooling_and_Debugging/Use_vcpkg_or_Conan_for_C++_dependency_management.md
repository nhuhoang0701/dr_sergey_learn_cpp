# Use vcpkg or Conan for C++ dependency management

**Category:** Tooling & Debugging  
**Item:** #171  
**Reference:** <https://vcpkg.io>  

---

## Topic Overview

C++ dependency management tools automate downloading, building, and linking third-party libraries.

| Feature | vcpkg | Conan |
| --- | --- | --- |
| Maintainer | Microsoft | JFrog |
| Package count | 2,500+ | 1,500+ |
| CMake integration | Native toolchain file | Generator-based |
| Binary caching | Yes | Yes |
| Custom recipes | Ports | conanfile.py |
| Corporate proxy | Supported | Supported |

---

## Self-Assessment

### Q1: Set up a CMake project with vcpkg to use Boost.Asio

```bash

# Step 1: Install vcpkg
$ git clone https://github.com/microsoft/vcpkg.git
$ cd vcpkg && ./bootstrap-vcpkg.sh  # Linux/macOS
# or .\bootstrap-vcpkg.bat          # Windows

```

```json

// vcpkg.json (manifest file — place in project root)
{
    "name": "my-server",
    "version-string": "1.0.0",
    "dependencies": [
        "boost-asio",
        "fmt",
        "spdlog"
    ]
}

```

```cmake

# CMakeLists.txt
cmake_minimum_required(VERSION 3.20)
project(MyServer LANGUAGES CXX)
set(CMAKE_CXX_STANDARD 20)

# vcpkg provides find_package support automatically:
find_package(Boost REQUIRED COMPONENTS system)
find_package(fmt CONFIG REQUIRED)
find_package(spdlog CONFIG REQUIRED)

add_executable(server main.cpp)
target_link_libraries(server PRIVATE
    Boost::system
    fmt::fmt
    spdlog::spdlog
)

```

```cpp

// main.cpp
#include <boost/asio.hpp>
#include <fmt/format.h>
#include <spdlog/spdlog.h>
#include <iostream>

int main() {
    boost::asio::io_context io;
    spdlog::info("Server starting...");
    fmt::print("Boost.Asio version: {}\n", BOOST_ASIO_VERSION);
}

```

```bash

# Build with vcpkg toolchain:
$ cmake -B build \
    -DCMAKE_TOOLCHAIN_FILE=<vcpkg-root>/scripts/buildsystems/vcpkg.cmake
$ cmake --build build
# vcpkg automatically installs dependencies from vcpkg.json

```

### Q2: vcpkg manifest mode vs classic mode

| Aspect | Classic Mode | Manifest Mode |
| --- | --- | --- |
| Install command | `vcpkg install boost-asio` | Automatic from `vcpkg.json` |
| Scope | Global (shared across projects) | Per-project |
| Version control | Not versioned | `vcpkg.json` checked into Git |
| Reproducibility | Depends on global state | Fully reproducible |
| CI friendly | Requires pre-install step | Self-contained |

```bash

# Classic mode (legacy):
$ vcpkg install boost-asio fmt spdlog
$ cmake -B build -DCMAKE_TOOLCHAIN_FILE=.../vcpkg.cmake
# Libraries installed globally in vcpkg/installed/

# Manifest mode (recommended):
# Just create vcpkg.json in project root:
$ cmake -B build -DCMAKE_TOOLCHAIN_FILE=.../vcpkg.cmake
# vcpkg reads vcpkg.json and installs to build/vcpkg_installed/
# Each project has its own dependency set

# Version pinning (vcpkg.json):
{
    "dependencies": [
        { "name": "fmt", "version>=": "10.0.0" }
    ],
    "builtin-baseline": "abc123..."  // pin vcpkg registry version
}

```

### Q3: Create a Conan recipe for a header-only library

```python

# conanfile.py — recipe for a header-only library
from conan import ConanFile
from conan.tools.files import copy
import os

class MyMathConan(ConanFile):
    name = "mymath"
    version = "1.0.0"
    license = "MIT"
    description = "A simple header-only math library"
    exports_sources = "include/*"
    no_copy_source = True  # header-only: no build needed

    def package(self):
        copy(self, "*.hpp",
             src=os.path.join(self.source_folder, "include"),
             dst=os.path.join(self.package_folder, "include"))

    def package_info(self):
        self.cpp_info.bindirs = []
        self.cpp_info.libdirs = []
        # Header-only: no libs to link

    def package_id(self):
        self.info.clear()  # Header-only: same package for all settings

```

```cpp

// include/mymath/math.hpp
#pragma once

namespace mymath {
    constexpr double pi = 3.14159265358979323846;

    constexpr int square(int x) { return x * x; }
    constexpr double circle_area(double r) { return pi * r * r; }
}

```

```bash

# Create and test the package:
$ conan create . --build=missing
# Exporting: mymath/1.0.0
# Package created: mymath/1.0.0

# Use in a consumer project:
# conanfile.txt:
[requires]
mymath/1.0.0

[generators]
CMakeDeps
CMakeToolchain

# Build:
$ conan install . --build=missing
$ cmake -B build -DCMAKE_TOOLCHAIN_FILE=build/conan_toolchain.cmake
$ cmake --build build

```

---

## Notes

- Use **manifest mode** for vcpkg (reproducible, version-controlled).
- Conan 2.x is the current major version; avoid Conan 1.x APIs.
- Both support binary caching for faster CI builds.
- vcpkg integrates more seamlessly with CMake's `find_package()`.
- Conan is more flexible for custom build systems and cross-compilation.
