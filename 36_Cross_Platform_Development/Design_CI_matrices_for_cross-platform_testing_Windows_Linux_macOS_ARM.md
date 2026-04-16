# Design CI matrices for cross-platform testing

**Category:** Cross-Platform Development  
**Standard:** C++17  
**Reference:** <https://docs.github.com/en/actions/using-jobs/using-a-matrix-strategy-for-your-jobs>  

---

## Topic Overview

### Why CI Matrices Matter

A single compiler on one platform catches only a fraction of bugs. Different compilers have different:

- **Warning sets** (Clang catches things GCC misses and vice versa)
- **Standard library implementations** (libstdc++, libc++, MSVC STL)
- **ABI and padding** differences
- **Platform-specific behavior** (path separators, endianness, `wchar_t` size)

A CI matrix runs the same tests across multiple configurations automatically.

### GitHub Actions Matrix — Complete Example

```yaml

name: CI
on: [push, pull_request]

jobs:
  build-and-test:
    strategy:
      fail-fast: false  # Don't cancel other jobs if one fails
      matrix:
        include:
          # Linux GCC

          - os: ubuntu-24.04

            compiler: gcc-13
            cxx: g++-13
            cc: gcc-13
            build_type: Release
            sanitizer: ""

          # Linux Clang

          - os: ubuntu-24.04

            compiler: clang-17
            cxx: clang++-17
            cc: clang-17
            build_type: Release
            sanitizer: ""

          # Linux ASan + UBSan

          - os: ubuntu-24.04

            compiler: clang-17-asan
            cxx: clang++-17
            cc: clang-17
            build_type: Debug
            sanitizer: "-fsanitize=address,undefined -fno-omit-frame-pointer"

          # Linux TSan

          - os: ubuntu-24.04

            compiler: clang-17-tsan
            cxx: clang++-17
            cc: clang-17
            build_type: Debug
            sanitizer: "-fsanitize=thread"

          # macOS

          - os: macos-14

            compiler: apple-clang
            cxx: clang++
            cc: clang
            build_type: Release
            sanitizer: ""

          # Windows MSVC

          - os: windows-2022

            compiler: msvc
            cxx: cl
            cc: cl
            build_type: Release
            sanitizer: ""

    runs-on: ${{ matrix.os }}
    name: ${{ matrix.compiler }}

    steps:

      - uses: actions/checkout@v4

      - name: Configure

        run: >
          cmake -B build
          -DCMAKE_BUILD_TYPE=${{ matrix.build_type }}
          -DCMAKE_CXX_COMPILER=${{ matrix.cxx }}
          -DCMAKE_C_COMPILER=${{ matrix.cc }}
          -DCMAKE_CXX_FLAGS="${{ matrix.sanitizer }}"
          -DCMAKE_EXE_LINKER_FLAGS="${{ matrix.sanitizer }}"

      - name: Build

        run: cmake --build build --config ${{ matrix.build_type }} -j $(nproc 2>/dev/null || echo 4)

      - name: Test

        run: ctest --test-dir build --build-config ${{ matrix.build_type }} --output-on-failure -j4

```

### ARM Cross-Compilation in CI

```yaml

  arm64-cross:
    runs-on: ubuntu-24.04
    name: ARM64 Cross-Compile

    steps:

      - uses: actions/checkout@v4

      - name: Install cross-compiler

        run: sudo apt-get install -y gcc-aarch64-linux-gnu g++-aarch64-linux-gnu qemu-user

      - name: Configure

        run: cmake -B build -DCMAKE_TOOLCHAIN_FILE=cmake/aarch64-linux.cmake

      - name: Build

        run: cmake --build build -j$(nproc)

      - name: Test under QEMU

        run: |
          cd build
          ctest --output-on-failure
        env:
          QEMU_LD_PREFIX: /usr/aarch64-linux-gnu

```

### Multi-Standard Matrix

Test with multiple C++ standard versions:

```yaml

    strategy:
      matrix:
        cxx_standard: [17, 20, 23]
        compiler: [g++-13, clang++-17]

    steps:

      - name: Configure

        run: >
          cmake -B build
          -DCMAKE_CXX_STANDARD=${{ matrix.cxx_standard }}
          -DCMAKE_CXX_COMPILER=${{ matrix.compiler }}

```

### Caching Dependencies

```yaml

      - name: Cache vcpkg

        uses: actions/cache@v4
        with:
          path: build/vcpkg_installed
          key: vcpkg-${{ matrix.os }}-${{ hashFiles('vcpkg.json') }}

      - name: Cache build

        uses: actions/cache@v4
        with:
          path: build
          key: build-${{ matrix.compiler }}-${{ hashFiles('CMakeLists.txt', 'src/**') }}

```

### What Each Config Catches

| Configuration | Catches |
| --- | --- |
| GCC Release | Warnings GCC-specific, optimization bugs |
| Clang Release | Warnings Clang-specific, different code gen |
| ASan + UBSan | Buffer overflows, use-after-free, UB, signed overflow |
| TSan | Data races, deadlocks, lock-order violations |
| MSVC | Windows-specific issues, different ABI, permissive mode violations |
| macOS | Apple Clang quirks, framework compatibility |
| ARM / QEMU | Endianness, alignment, weak memory model bugs |
| C++17/20/23 | Feature availability, deprecated features, breaking changes |

---

## Self-Assessment

### Q1: Why use `fail-fast: false`

By default, GitHub Actions cancels all matrix jobs when one fails. With `fail-fast: false`, all configurations run to completion. This is important because:

- A bug might only reproduce on one platform — you need all results to diagnose
- Different sanitizers catch different bugs — you want all sanitizer results
- A platform-specific fix might break another platform — you need the full picture

### Q2: What is the minimum CI matrix for a cross-platform C++ library

At minimum: **GCC + Clang + MSVC** (3 compilers) with **ASan+UBSan** (1 sanitizer config). This gives you 4 jobs total and catches the vast majority of portability issues. Add TSan if you have concurrent code, and ARM cross-compilation if you target embedded.

### Q3: How do you test on ARM without ARM hardware

Use `qemu-user` (user-mode QEMU) which translates ARM instructions to x86 on the fly. Cross-compile with `aarch64-linux-gnu-g++`, then run the test binary with `qemu-aarch64`. Set `QEMU_LD_PREFIX` to point to the ARM sysroot for shared libraries. This is slower than native but catches alignment, endianness, and ABI issues.

---

## Notes

- Cache build artifacts to speed up CI — `actions/cache@v4` can cut build time by 50%+
- Run sanitizers in Debug mode (`-O0` or `-O1`) for better error messages
- Use `ccache` in CI to cache compilation across runs
- Consider using `act` (https://github.com/nektos/act) to test GitHub Actions locally
- Add a coverage job with `--coverage` and upload to Codecov or Coveralls
