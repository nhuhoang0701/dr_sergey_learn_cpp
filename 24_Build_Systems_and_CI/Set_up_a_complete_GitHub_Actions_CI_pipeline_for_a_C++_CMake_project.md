# Set up a complete GitHub Actions CI pipeline for a C++ CMake project

**Category:** Build Systems & CI  
**Item:** #663  
**Reference:** <https://docs.github.com/en/actions>  

---

## Topic Overview

GitHub Actions provides free CI/CD for open-source C++ projects. A well-structured pipeline builds on all three major platforms (Linux/macOS/Windows), runs tests with sanitizers, and caches builds for speed.

### Pipeline Architecture

```cpp

Push / PR
  ↓
┌─────────────────────────────────────────────┐
│ Matrix Strategy                              │
│  ├── ubuntu-latest   + GCC 13               │
│  ├── ubuntu-latest   + Clang 17             │
│  ├── macos-latest    + Apple Clang          │
│  └── windows-latest  + MSVC 2022           │
├─────────────────────────────────────────────┤
│ Sanitizer Jobs (Linux only)                  │
│  ├── ASAN + UBSAN                           │
│  └── TSAN                                   │
├─────────────────────────────────────────────┤
│ Static Analysis (clang-tidy, cppcheck)      │
└─────────────────────────────────────────────┘

```

### Key GitHub Actions Concepts

| Concept | Purpose |
| --- | --- |
| `strategy.matrix` | Run same steps across multiple OS/compiler combos |
| `actions/cache` | Persist build dir / vcpkg between runs |
| `cmake --preset` | Reproducible configure from CMakePresets.json |
| `ctest --output-on-failure` | Show test output only on failures |

### Minimal Workflow File

```yaml

# .github/workflows/ci.yml
name: CI
on: [push, pull_request]

jobs:
  build:
    runs-on: ${{ matrix.os }}
    strategy:
      fail-fast: false
      matrix:
        os: [ubuntu-latest, macos-latest, windows-latest]
    steps:

      - uses: actions/checkout@v4
      - name: Configure

        run: cmake -B build -DCMAKE_BUILD_TYPE=Release

      - name: Build

        run: cmake --build build --config Release -j $(nproc 2>/dev/null || echo 2)

      - name: Test

        run: ctest --test-dir build --output-on-failure -C Release

```

---

## Self-Assessment

### Q1: Write a matrix workflow that builds on ubuntu-latest (GCC), macos-latest (Clang), and windows-latest (MSVC)

**Answer:**

```yaml

# .github/workflows/ci.yml
name: C++ CI

on:
  push:
    branches: [main, develop]
  pull_request:
    branches: [main]

jobs:
  build:
    name: ${{ matrix.os }} / ${{ matrix.compiler }}
    runs-on: ${{ matrix.os }}
    strategy:
      fail-fast: false      # Don't cancel other jobs on first failure
      matrix:
        include:
          # ── Linux GCC ──

          - os: ubuntu-latest

            compiler: gcc-13
            cc: gcc-13
            cxx: g++-13
            install: sudo apt-get install -y gcc-13 g++-13

          # ── Linux Clang ──

          - os: ubuntu-latest

            compiler: clang-17
            cc: clang-17
            cxx: clang++-17
            install: |
              wget https://apt.llvm.org/llvm.sh
              chmod +x llvm.sh
              sudo ./llvm.sh 17
          
          # ── macOS Apple Clang ──

          - os: macos-latest

            compiler: apple-clang
            cc: clang
            cxx: clang++
            install: ""

          # ── Windows MSVC ──

          - os: windows-latest

            compiler: msvc
            cc: cl
            cxx: cl
            install: ""

    steps:

      - uses: actions/checkout@v4

      - name: Install compiler

        if: matrix.install != ''
        run: ${{ matrix.install }}

      - name: Configure CMake

        env:
          CC: ${{ matrix.cc }}
          CXX: ${{ matrix.cxx }}
        run: >
          cmake -B build
          -DCMAKE_BUILD_TYPE=Release
          -DCMAKE_CXX_STANDARD=20
          -DBUILD_TESTING=ON

      - name: Build

        run: cmake --build build --config Release --parallel 2

      - name: Run tests

        working-directory: build
        run: ctest --output-on-failure -C Release --timeout 120

```

**Explanation:**

- `fail-fast: false` ensures all platforms report results even if one fails
- `matrix.include` gives explicit control over each configuration
- `CC`/`CXX` environment variables tell CMake which compiler to use
- `--parallel 2` matches the 2-vCPU GitHub-hosted runners
- `ctest --output-on-failure` only prints test output when something fails

### Q2: Add sanitizer jobs: one with ASAN+UBSAN and one with TSAN on Linux

**Answer:**

```yaml

  # Add this job alongside the build matrix job above
  sanitizers:
    name: ${{ matrix.sanitizer }}
    runs-on: ubuntu-latest
    strategy:
      fail-fast: false
      matrix:
        include:

          - sanitizer: ASAN+UBSAN

            cmake_flags: >
              -DCMAKE_CXX_FLAGS="-fsanitize=address,undefined
              -fno-sanitize-recover=all -fno-omit-frame-pointer"
              -DCMAKE_EXE_LINKER_FLAGS="-fsanitize=address,undefined"
            env_vars: "ASAN_OPTIONS=detect_leaks=1:detect_stack_use_after_return=1"

          - sanitizer: TSAN

            cmake_flags: >
              -DCMAKE_CXX_FLAGS="-fsanitize=thread -fno-omit-frame-pointer"
              -DCMAKE_EXE_LINKER_FLAGS="-fsanitize=thread"
            env_vars: "TSAN_OPTIONS=second_deadlock_stack=1"

    steps:

      - uses: actions/checkout@v4

      - name: Install Clang 17

        run: |
          wget https://apt.llvm.org/llvm.sh
          chmod +x llvm.sh
          sudo ./llvm.sh 17

      - name: Configure

        env:
          CC: clang-17
          CXX: clang++-17
        run: >
          cmake -B build
          -DCMAKE_BUILD_TYPE=Debug
          -DCMAKE_CXX_STANDARD=20
          -DBUILD_TESTING=ON
          ${{ matrix.cmake_flags }}

      - name: Build

        run: cmake --build build --parallel 2

      - name: Test with ${{ matrix.sanitizer }}

        env:
          ${{ matrix.env_vars }}: ""
        working-directory: build
        run: ctest --output-on-failure --timeout 300

```

**Key details:**

- ASAN and UBSAN are combined (compatible); TSAN must be separate (incompatible with ASAN)
- `-fno-sanitize-recover=all` makes the sanitizer abort on first error (fail-fast in tests)
- `-fno-omit-frame-pointer` gives better stack traces
- Debug build type is required for useful sanitizer line info
- `detect_leaks=1` enables the LeakSanitizer component of ASAN
- TSAN timeout is longer (300s) because thread sanitizer adds ~5-15x overhead

### Q3: Cache the build directory and vcpkg packages between CI runs to reduce build time

**Answer:**

```yaml

  build-cached:
    name: Cached Build (${{ matrix.os }})
    runs-on: ${{ matrix.os }}
    strategy:
      matrix:
        os: [ubuntu-latest, macos-latest, windows-latest]

    steps:

      - uses: actions/checkout@v4

      # ── Cache CMake build directory ──

      - name: Cache build directory

        uses: actions/cache@v4
        with:
          path: build
          key: build-${{ matrix.os }}-${{ hashFiles('CMakeLists.txt', 'src/**', 'include/**') }}
          restore-keys: |
            build-${{ matrix.os }}-

      # ── Cache vcpkg installed packages ──

      - name: Cache vcpkg

        uses: actions/cache@v4
        with:
          path: |
            ~/vcpkg
            build/vcpkg_installed
          key: vcpkg-${{ matrix.os }}-${{ hashFiles('vcpkg.json', 'vcpkg-configuration.json') }}
          restore-keys: |
            vcpkg-${{ matrix.os }}-

      # ── Setup vcpkg ──

      - name: Setup vcpkg

        uses: lukka/run-vcpkg@v11
        with:
          vcpkgGitCommitId: "a1a1cbc975abf909a6c8985a6a2b8fe20bbd9bd6"  # Pin!

      # ── Configure + Build ──

      - name: Configure

        run: >
          cmake -B build
          -DCMAKE_BUILD_TYPE=Release
          -DCMAKE_TOOLCHAIN_FILE=$VCPKG_ROOT/scripts/buildsystems/vcpkg.cmake
          -DCMAKE_CXX_STANDARD=20

      - name: Build (incremental)

        run: cmake --build build --config Release --parallel 2

      - name: Test

        run: ctest --test-dir build --output-on-failure -C Release

```

**Cache strategy explained:**

- **Build cache key** uses `hashFiles` on source files — rebuilds when code changes, reuses when only CI config changes
- **`restore-keys` fallback** restores the most recent cache for the same OS, enabling incremental builds (CMake only recompiles changed files)
- **vcpkg cache key** hashes `vcpkg.json` manifest — only reinstalls packages when dependencies change
- **Pin vcpkg commit** — without this, vcpkg updates silently break builds
- Cache saves ~3-8 minutes on typical projects (vcpkg install is the biggest cost)

---

## Notes

- **`ccache`** can further speed up builds — use `lukka/run-cmake@v10` with `CMAKE_CXX_COMPILER_LAUNCHER=ccache`
- **Artifacts:** Use `actions/upload-artifact@v4` to upload test reports, coverage, or built binaries
- **Branch protection:** Require the CI jobs to pass before merging PRs
- **Cost:** GitHub-hosted runners are free for public repos; private repos get 2000 min/month free
- **Self-hosted runners** are needed for specialized hardware (GPUs, ARM) or to avoid rate limits on large projects
- **`concurrency`** key can cancel in-progress runs when a new push arrives: `concurrency: { group: ci-${{ github.ref }}, cancel-in-progress: true }`
