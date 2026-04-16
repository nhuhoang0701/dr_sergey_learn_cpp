# Set up cross-compilation with CMake toolchain files

**Category:** Build Systems & CI  
**Item:** #742  
**Reference:** <https://cmake.org/cmake/help/latest/manual/cmake-toolchains.7.html>  

---

## Topic Overview

This third cross-compilation file focuses on the **end-to-end workflow**: writing a toolchain file, cross-compiling a full project, running it under QEMU, and understanding why `CMAKE_FIND_ROOT_PATH_MODE_*` must be configured correctly.

### Cross-Compilation Workflow

```cpp

1. Write toolchain file  →  arm-linux-gnueabihf.cmake
2. Configure             →  cmake -B build --toolchain arm-linux-gnueabihf.cmake
3. Build                 →  cmake --build build
4. Test via QEMU         →  ctest --test-dir build  (uses CMAKE_CROSSCOMPILING_EMULATOR)
5. Deploy                →  scp build/app user@target-device:/opt/

```

### CMAKE_FIND_ROOT_PATH_MODE Summary

| Mode | `NEVER` | `ONLY` | `BOTH` |
| --- | --- | --- | --- |
| **PROGRAM** | Search host only (tools) | Search target only | Search everywhere |
| **LIBRARY** | Search host only (danger!) | Search target only ✅ | Search everywhere |
| **INCLUDE** | Search host only (danger!) | Search target only ✅ | Search everywhere |
| **PACKAGE** | Search host only | Search target only ✅ | Search everywhere |

The convention: `PROGRAM = NEVER`, everything else `= ONLY`. This ensures build tools come from the host but libraries/headers come from the target sysroot.

---

## Self-Assessment

### Q1: Write a toolchain file that sets CMAKE_SYSTEM_NAME, CMAKE_C_COMPILER, and sysroot for ARM Linux

**Answer:**

```cmake

# cmake/arm-linux-gnueabihf.cmake
# ARM 32-bit hard-float toolchain for Raspberry Pi / Debian armhf

# ═══════════ Target platform ═══════════
set(CMAKE_SYSTEM_NAME Linux)           # Triggers cross-compilation mode
set(CMAKE_SYSTEM_PROCESSOR armv7l)     # ARM 32-bit

# ═══════════ Compilers ═══════════
set(CMAKE_C_COMPILER   arm-linux-gnueabihf-gcc)
set(CMAKE_CXX_COMPILER arm-linux-gnueabihf-g++)
set(CMAKE_ASM_COMPILER arm-linux-gnueabihf-gcc)

# Optional: specify exact version if multiple installed
# set(CMAKE_C_COMPILER   /usr/bin/arm-linux-gnueabihf-gcc-12)
# set(CMAKE_CXX_COMPILER /usr/bin/arm-linux-gnueabihf-g++-12)

# ═══════════ Sysroot ═══════════
# Contains target headers (/usr/include/) and libraries (/usr/lib/)
# Create:
#   sudo debootstrap --arch=armhf bullseye /opt/armhf-sysroot
#   OR: mount a Raspberry Pi SD card image
set(CMAKE_SYSROOT /opt/armhf-sysroot)

# ═══════════ Search path control ═══════════
set(CMAKE_FIND_ROOT_PATH_MODE_PROGRAM NEVER)  # Host tools (protoc, etc.)
set(CMAKE_FIND_ROOT_PATH_MODE_LIBRARY ONLY)   # Target libraries
set(CMAKE_FIND_ROOT_PATH_MODE_INCLUDE ONLY)   # Target headers
set(CMAKE_FIND_ROOT_PATH_MODE_PACKAGE ONLY)   # Target CMake packages

# ═══════════ Emulator for testing ═══════════
set(CMAKE_CROSSCOMPILING_EMULATOR "qemu-arm;-L;${CMAKE_SYSROOT}")

# ═══════════ CPU-specific flags ═══════════
set(CMAKE_C_FLAGS_INIT   "-march=armv7-a -mfpu=neon-vfpv4 -mfloat-abi=hard")
set(CMAKE_CXX_FLAGS_INIT "-march=armv7-a -mfpu=neon-vfpv4 -mfloat-abi=hard")

```

**Variables explained:**

- `CMAKE_SYSTEM_NAME` → any value other than the host OS triggers cross-compilation mode
- `CMAKE_SYSTEM_PROCESSOR` → used by CMake checks and feature tests
- `CMAKE_SYSROOT` → passed to GCC as `--sysroot=` flag; all standard include/lib paths are relative to this
- `CMAKE_C_FLAGS_INIT` → flags set once during initial configuration (not overridden on reconfigure)

### Q2: Cross-compile a CMake project for aarch64 on an x86_64 host and run it under QEMU

**Answer:**

```bash

# ═══════════ Full aarch64 cross-compilation walkthrough ═══════════

# Step 1: Install tools
sudo apt-get update
sudo apt-get install -y \
    gcc-aarch64-linux-gnu g++-aarch64-linux-gnu \
    qemu-user qemu-user-binfmt

# Step 2: Create minimal sysroot (or use an existing ARM64 rootfs)
sudo debootstrap --arch=arm64 jammy /opt/arm64-sysroot

# Step 3: Write the toolchain file (aarch64-linux.cmake)
cat > cmake/aarch64-linux.cmake << 'EOF'
set(CMAKE_SYSTEM_NAME Linux)
set(CMAKE_SYSTEM_PROCESSOR aarch64)
set(CMAKE_C_COMPILER   aarch64-linux-gnu-gcc)
set(CMAKE_CXX_COMPILER aarch64-linux-gnu-g++)
set(CMAKE_SYSROOT /opt/arm64-sysroot)
set(CMAKE_FIND_ROOT_PATH_MODE_PROGRAM NEVER)
set(CMAKE_FIND_ROOT_PATH_MODE_LIBRARY ONLY)
set(CMAKE_FIND_ROOT_PATH_MODE_INCLUDE ONLY)
set(CMAKE_FIND_ROOT_PATH_MODE_PACKAGE ONLY)
set(CMAKE_CROSSCOMPILING_EMULATOR "qemu-aarch64;-L;${CMAKE_SYSROOT}")
EOF

# Step 4: Configure
cmake -B build-arm64 \
    --toolchain cmake/aarch64-linux.cmake \
    -DCMAKE_BUILD_TYPE=Release \
    -DCMAKE_CXX_STANDARD=20 \
    -DBUILD_TESTING=ON

# Step 5: Build
cmake --build build-arm64 --parallel $(nproc)

# Step 6: Verify it's an ARM64 binary
file build-arm64/myapp
# myapp: ELF 64-bit LSB pie executable, ARM aarch64, version 1 (SYSV)

# Step 7: Run manually under QEMU
qemu-aarch64 -L /opt/arm64-sysroot build-arm64/myapp
# Output: Hello from C++20 on aarch64

# Step 8: Run tests via ctest (uses QEMU automatically)
cd build-arm64 && ctest --output-on-failure --timeout 120
# ctest internally runs: qemu-aarch64 -L /opt/arm64-sysroot ./test_mylib
# All 15 tests passed

# Step 9: Deploy to real hardware
scp build-arm64/myapp user@arm-device:/opt/bin/
ssh user@arm-device /opt/bin/myapp

```

**QEMU performance note:** `qemu-user` translates ARM64 instructions to x86_64 at runtime with ~5-15x overhead. This is acceptable for correctness testing but not for benchmarks. For performance testing, use the actual target hardware or a VM.

### Q3: Explain CMAKE_FIND_ROOT_PATH_MODE_* settings and why they are critical for cross builds

**Answer:**

```cmake

# ═══════════ What happens WITHOUT proper MODE settings ═══════════
#
# Default (no toolchain file settings):
#   find_library(ZLIB z)
#     Searches: /usr/lib/x86_64-linux-gnu/libz.so  ← HOST library
#     Found: /usr/lib/x86_64-linux-gnu/libz.so
#     Links x86_64 libz into ARM binary → CRASH at runtime
#     Error: "wrong ELF class: ELFCLASS64" (expected ELFCLASS32 for armhf)
#
# ═══════════ What happens WITH proper MODE settings ═══════════
#
# set(CMAKE_FIND_ROOT_PATH_MODE_LIBRARY ONLY)
# set(CMAKE_SYSROOT /opt/armhf-sysroot)
#
#   find_library(ZLIB z)
#     Searches: /opt/armhf-sysroot/usr/lib/arm-linux-gnueabihf/libz.so  ← TARGET
#     Found: correct ARM library
#     Links correctly → works on ARM device

# ═══════════ Why PROGRAM must be NEVER ═══════════
#
# Build tools (protoc, moc, flex, bison) run on the HOST during build
# If MODE_PROGRAM = ONLY:
#   find_program(PROTOC protoc)
#     Searches: /opt/armhf-sysroot/usr/bin/protoc  ← ARM binary
#     Can't execute ARM binary on x86_64 host → build fails
#
# With MODE_PROGRAM = NEVER:
#   find_program(PROTOC protoc)
#     Searches: /usr/bin/protoc  ← HOST binary
#     Executes on host → generates .pb.cc files → cross-compiled into ARM

# ═══════════ Debugging with --debug-find ═══════════

```

```bash

# See exactly where CMake searches for each library/package:
cmake -B build --toolchain arm.cmake --debug-find 2>&1 | grep "find_library"

# Example output with MODE_LIBRARY ONLY:
# find_library(ZLIB_LIBRARY) searching:
#   /opt/armhf-sysroot/usr/lib/arm-linux-gnueabihf    ← sysroot
#   /opt/armhf-sysroot/usr/local/lib                    ← sysroot
# find_library(ZLIB_LIBRARY) result: /opt/armhf-sysroot/usr/lib/arm-linux-gnueabihf/libz.so

# Without ONLY mode, you'd also see:
#   /usr/lib/x86_64-linux-gnu    ← HOST (wrong!)
#   /usr/local/lib                ← HOST (wrong!)

```

```cmake

# ═══════════ Special case: CMAKE_FIND_ROOT_PATH for extra libs ═══════════

# If you installed custom libraries outside the sysroot:
set(CMAKE_FIND_ROOT_PATH
    /opt/armhf-sysroot         # Standard target libs
    /opt/custom-arm-libs       # Extra cross-compiled libraries
)

# Now find_library also searches:
#   /opt/custom-arm-libs/lib/
#   /opt/custom-arm-libs/usr/lib/

# ═══════════ Per-call override ═══════════
# Override the mode for a single find_* call:
find_library(HOST_TOOL tool
    NO_CMAKE_FIND_ROOT_PATH     # Ignore sysroot, search host
)

find_library(TARGET_LIB foo
    ONLY_CMAKE_FIND_ROOT_PATH   # Force sysroot search only
)

```

**Summary of the three settings for cross-compilation:**

| Setting | Value | Why |
| --- | --- | --- |
| `CMAKE_FIND_ROOT_PATH_MODE_PROGRAM` | `NEVER` | Build tools (protoc, moc) must run on the host |
| `CMAKE_FIND_ROOT_PATH_MODE_LIBRARY` | `ONLY` | Target libraries must match the target architecture |
| `CMAKE_FIND_ROOT_PATH_MODE_INCLUDE` | `ONLY` | Target headers may differ from host headers |

Without these, every `find_library` and `find_package` call risks pulling in host libraries, causing cryptic linker errors or runtime crashes on the target device.

---

## Notes

- **`CMAKE_FIND_ROOT_PATH_MODE_PACKAGE ONLY`** is also recommended — it restricts `find_package` config-file searches to the sysroot
- **CMake presets** can encode toolchain files: `"toolchainFile": "cmake/aarch64-linux.cmake"` in `CMakePresets.json`
- **Docker-based cross-compilation** often uses multiarch packages instead of a sysroot: `dpkg --add-architecture arm64 && apt-get install libboost-all-dev:arm64`
- **`NO_CMAKE_FIND_ROOT_PATH`** on individual `find_*` calls overrides the global mode — useful for special cases
