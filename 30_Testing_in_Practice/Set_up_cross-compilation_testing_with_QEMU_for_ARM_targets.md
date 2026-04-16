# Set up cross-compilation testing with QEMU for ARM targets

**Category:** Testing in Practice

---

## Topic Overview

**QEMU user-mode emulation** lets you run ARM (or other architecture) binaries on an x86 host. Combined with cross-compilation, this enables running embedded test binaries in CI without physical hardware. QEMU system emulation can even emulate full boards with peripherals.

### QEMU Modes

| Mode | Emulates | Speed | Use Case |
| --- | --- | --- | --- |
| **User-mode** (`qemu-arm`) | CPU + syscalls | Fast (10-50x native) | Run test binaries |
| **System** (`qemu-system-arm`) | Full board | Slow (100x+) | Boot firmware, test peripherals |
| **Docker multiarch** | Transparent via binfmt | Moderate | CI containers |

---

## Self-Assessment

### Q1: Set up ARM cross-compilation and QEMU test execution

**Answer:**

```bash

# === Install cross-compilation toolchain + QEMU ===
# Ubuntu/Debian:
sudo apt-get install -y \
    gcc-arm-linux-gnueabihf \
    g++-arm-linux-gnueabihf \
    qemu-user \
    qemu-user-static \
    libc6-armhf-cross

# For AArch64:
sudo apt-get install -y \
    gcc-aarch64-linux-gnu \
    g++-aarch64-linux-gnu \
    qemu-user

# Verify
arm-linux-gnueabihf-g++ --version
qemu-arm --version

```

```cmake

# === arm-toolchain.cmake ===
set(CMAKE_SYSTEM_NAME Linux)
set(CMAKE_SYSTEM_PROCESSOR arm)

set(CMAKE_C_COMPILER arm-linux-gnueabihf-gcc)
set(CMAKE_CXX_COMPILER arm-linux-gnueabihf-g++)

# Sysroot for cross-compiled libraries
set(CMAKE_SYSROOT /usr/arm-linux-gnueabihf)
set(CMAKE_FIND_ROOT_PATH /usr/arm-linux-gnueabihf)

set(CMAKE_FIND_ROOT_PATH_MODE_PROGRAM NEVER)
set(CMAKE_FIND_ROOT_PATH_MODE_LIBRARY ONLY)
set(CMAKE_FIND_ROOT_PATH_MODE_INCLUDE ONLY)

# Tell CTest to run tests via QEMU
set(CMAKE_CROSSCOMPILING_EMULATOR
    "qemu-arm;-L;/usr/arm-linux-gnueabihf")

```

```cmake

# === CMakeLists.txt ===
cmake_minimum_required(VERSION 3.20)
project(embedded_tests)

set(CMAKE_CXX_STANDARD 20)

add_library(firmware_logic
    src/protocol_parser.cpp
    src/state_machine.cpp
    src/crc_calculator.cpp
)

enable_testing()
find_package(GTest REQUIRED)

add_executable(firmware_tests
    tests/test_protocol.cpp
    tests/test_state_machine.cpp
    tests/test_crc.cpp
)
target_link_libraries(firmware_tests firmware_logic GTest::gtest_main)
add_test(NAME firmware_tests COMMAND firmware_tests)
# CMAKE_CROSSCOMPILING_EMULATOR automatically prepends qemu-arm

```

```bash

# === Build and test ===
# Cross-compile for ARM
cmake -B build-arm --toolchain arm-toolchain.cmake
cmake --build build-arm

# Verify it's an ARM binary
file build-arm/firmware_tests
# Output: ELF 32-bit LSB executable, ARM, EABI5

# Run via QEMU (automatic with CTest)
ctest --test-dir build-arm --output-on-failure

# Or run manually
qemu-arm -L /usr/arm-linux-gnueabihf ./build-arm/firmware_tests

```

### Q2: Build Google Test for ARM cross-compilation

**Answer:**

```cmake

# === Cross-compile GTest as part of the project ===
include(FetchContent)
FetchContent_Declare(
    googletest
    GIT_REPOSITORY https://github.com/google/googletest.git
    GIT_TAG        v1.14.0
)
# Prevent GTest from being installed with the project
set(INSTALL_GTEST OFF CACHE BOOL "" FORCE)
FetchContent_MakeAvailable(googletest)

# GTest is automatically cross-compiled with the project's toolchain
add_executable(arm_tests tests/test_all.cpp)
target_link_libraries(arm_tests gtest_main gmock)
add_test(NAME arm_tests COMMAND arm_tests)

```

```bash

# === aarch64-toolchain.cmake (64-bit ARM) ===
set(CMAKE_SYSTEM_NAME Linux)
set(CMAKE_SYSTEM_PROCESSOR aarch64)

set(CMAKE_C_COMPILER aarch64-linux-gnu-gcc)
set(CMAKE_CXX_COMPILER aarch64-linux-gnu-g++)

set(CMAKE_SYSROOT /usr/aarch64-linux-gnu)
set(CMAKE_FIND_ROOT_PATH /usr/aarch64-linux-gnu)

set(CMAKE_FIND_ROOT_PATH_MODE_PROGRAM NEVER)
set(CMAKE_FIND_ROOT_PATH_MODE_LIBRARY ONLY)
set(CMAKE_FIND_ROOT_PATH_MODE_INCLUDE ONLY)

set(CMAKE_CROSSCOMPILING_EMULATOR
    "qemu-aarch64;-L;/usr/aarch64-linux-gnu")

```

### Q3: CI pipeline with QEMU cross-compilation testing

**Answer:**

```yaml

# === .github/workflows/cross-test.yml ===
name: Cross-Platform Tests

on: [push, pull_request]

jobs:
  cross-test:
    strategy:
      matrix:
        arch:

          - name: arm32

            toolchain: arm-toolchain.cmake
            packages: gcc-arm-linux-gnueabihf g++-arm-linux-gnueabihf qemu-user

          - name: aarch64

            toolchain: aarch64-toolchain.cmake
            packages: gcc-aarch64-linux-gnu g++-aarch64-linux-gnu qemu-user

    runs-on: ubuntu-latest
    name: test-${{ matrix.arch.name }}

    steps:

      - uses: actions/checkout@v4

      - name: Install cross toolchain

        run: |
          sudo apt-get update
          sudo apt-get install -y ${{ matrix.arch.packages }} ninja-build

      - name: Cross-compile

        run: |
          cmake -B build -G Ninja \
            --toolchain cmake/${{ matrix.arch.toolchain }} \
            -DCMAKE_BUILD_TYPE=Release
          cmake --build build

      - name: Verify binary architecture

        run: file build/firmware_tests

      - name: Run tests via QEMU

        run: ctest --test-dir build --output-on-failure --parallel 4
        timeout-minutes: 10

  # === Docker multiarch (alternative approach) ===
  docker-arm:
    runs-on: ubuntu-latest
    steps:

      - uses: actions/checkout@v4

      - name: Set up QEMU for Docker

        uses: docker/setup-qemu-action@v3

      - name: Build and test in ARM container

        run: |
          docker run --rm --platform linux/arm/v7 \
            -v ${{ github.workspace }}:/src -w /src \
            gcc:13 \
            bash -c "
              apt-get update && \
              apt-get install -y cmake ninja-build && \
              cmake -B build -G Ninja && \
              cmake --build build && \
              ctest --test-dir build --output-on-failure
            "

```

---

## Notes

- **`CMAKE_CROSSCOMPILING_EMULATOR`** is the key — CTest automatically prepends QEMU to all test commands
- Use `-L /path/to/sysroot` with QEMU to resolve dynamic libraries for cross-compiled binaries
- User-mode QEMU handles CPU emulation well but does **not** emulate peripherals (timers, GPIO, DMA)
- For peripheral-dependent code, use QEMU system mode with a machine definition (much more complex)
- Static linking (`-static`) avoids sysroot issues: `set(CMAKE_EXE_LINKER_FLAGS "-static")`
- QEMU user-mode runs at ~10-50x slowdown vs native — fast enough for test suites
- Docker multiarch with `setup-qemu-action` is simpler but slower than direct cross-compilation
- Test both 32-bit ARM and AArch64 if your firmware targets both
