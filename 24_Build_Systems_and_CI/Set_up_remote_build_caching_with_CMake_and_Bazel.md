# Set up remote build caching with CMake and Bazel

**Category:** Build Systems & CI  
**Item:** #746  
**Reference:** <https://github.com/mozilla/sccache>  

---

## Topic Overview

Remote build caching stores compilation results on a shared server so that multiple CI jobs, developers, or machines can reuse previous build outputs. Instead of recompiling unchanged source files, the build system fetches the cached object files. For large C++ projects, this can reduce CI build times from 30+ minutes to 2-5 minutes.

### Caching Architecture

```cpp

Developer A (laptop)          CI Runner 1          CI Runner 2
     │                             │                     │
     │  compile foo.cpp            │                     │
     ├──→ hash(flags + source) ──→ Cache Server ←────────┤
     │       cache miss!           (sccache/remote)      │
     │  ← upload foo.o             │                     │
     │                             │  compile foo.cpp    │
     │                             │  hash → cache HIT!  │
     │                             │  ← download foo.o   │
     │                             │  (no compilation!)  │

```

### Caching Tools Comparison

| Tool | Build System | Backend | Protocol |
| --- | --- | --- | --- |
| sccache | CMake/Make (any) | S3, GCS, Azure, Redis, local | HTTP |
| ccache | CMake/Make (any) | Local disk, remote via Redis | File/Redis |
| Bazel remote cache | Bazel | HTTP, gRPC (Remote Execution API) | HTTP/gRPC |
| BuildKit cache | Docker | Registry, S3, local | OCI |

---

## Self-Assessment

### Q1: Configure CMake with sccache as a compiler launcher to cache compilation results remotely

**Answer:**

```bash

# ═══════════ Install sccache ═══════════
# macOS
brew install sccache

# Linux (from GitHub releases)
curl -L https://github.com/mozilla/sccache/releases/download/v0.8.1/sccache-v0.8.1-x86_64-unknown-linux-musl.tar.gz \
  | tar xz -C /usr/local/bin --strip-components=1

# Verify
sccache --version
# sccache 0.8.1

```

```cmake

# ═══════════ Method 1: CMake variable ═══════════
# CMakeLists.txt
find_program(SCCACHE sccache)
if(SCCACHE)
    set(CMAKE_C_COMPILER_LAUNCHER ${SCCACHE})
    set(CMAKE_CXX_COMPILER_LAUNCHER ${SCCACHE})
    message(STATUS "Using sccache: ${SCCACHE}")
endif()

# ═══════════ Method 2: Command line ═══════════
# cmake -B build \
#   -DCMAKE_C_COMPILER_LAUNCHER=sccache \
#   -DCMAKE_CXX_COMPILER_LAUNCHER=sccache \
#   -DCMAKE_BUILD_TYPE=Release

```

```bash

# ═══════════ Configure S3 remote backend ═══════════
export SCCACHE_BUCKET=my-build-cache
export SCCACHE_REGION=us-east-1
export AWS_ACCESS_KEY_ID=AKIA...
export AWS_SECRET_ACCESS_KEY=...

# Or GCS:
# export SCCACHE_GCS_BUCKET=my-build-cache
# export SCCACHE_GCS_KEY_PATH=/path/to/service-account.json

# Or Redis:
# export SCCACHE_REDIS=redis://cache-server:6379

# Start sccache server
sccache --start-server

# Build with CMake
cmake -B build \
    -DCMAKE_C_COMPILER_LAUNCHER=sccache \
    -DCMAKE_CXX_COMPILER_LAUNCHER=sccache \
    -DCMAKE_BUILD_TYPE=Release

cmake --build build --parallel $(nproc)

# Check cache stats
sccache --show-stats
# Compile requests                 150
# Compile requests executed         150
# Cache hits                        120 (80%)
# Cache misses                       30
# Cache timeouts                      0
# Non-cacheable calls                 0

```

```yaml

# GitHub Actions CI integration
jobs:
  build:
    runs-on: ubuntu-latest
    steps:

      - uses: actions/checkout@v4

      - name: Install sccache

        run: |
          curl -L https://github.com/mozilla/sccache/releases/download/v0.8.1/sccache-v0.8.1-x86_64-unknown-linux-musl.tar.gz \
            | sudo tar xz -C /usr/local/bin --strip-components=1

      - name: Configure sccache

        env:
          SCCACHE_BUCKET: ${{ secrets.SCCACHE_BUCKET }}
          AWS_ACCESS_KEY_ID: ${{ secrets.AWS_ACCESS_KEY_ID }}
          AWS_SECRET_ACCESS_KEY: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
        run: sccache --start-server

      - name: Configure CMake

        run: cmake -B build -DCMAKE_CXX_COMPILER_LAUNCHER=sccache -DCMAKE_BUILD_TYPE=Release

      - name: Build

        run: cmake --build build -j2

      - name: Show cache stats

        run: sccache --show-stats

```

**How sccache works:** Before compiling each `.cpp` file, sccache hashes the preprocessed source + compiler flags + compiler version. If the hash matches a cached entry (locally or on S3/GCS), it returns the cached `.o` file without invoking the compiler. First build is cold (all misses), subsequent builds are hot (80-95% hit rate).

### Q2: Set up Bazel remote cache with --remote_cache pointing to an HTTP/gRPC cache server

**Answer:**

```python

# ═══════════ .bazelrc — Bazel configuration ═══════════
# Remote cache via HTTP (simplest setup)
build --remote_cache=https://cache.example.com/bazel-cache
build --remote_upload_local_results=true

# Authentication (if required)
build --remote_header=Authorization=Bearer <token>

# For Google Cloud Storage:
# build --remote_cache=https://storage.googleapis.com/my-bazel-cache
# build --google_default_credentials

# For gRPC remote execution (Buildfarm, BuildBuddy, etc.):
# build --remote_executor=grpc://buildfarm.example.com:8980
# build --remote_instance_name=default

```

```bash

# ═══════════ Simple HTTP cache with nginx ═══════════

# nginx.conf for a Bazel cache server:
# server {
#     listen 8080;
#     location /bazel-cache/ {
#         root /var/cache;
#         dav_methods PUT DELETE;
#         create_full_put_path on;
#         client_max_body_size 100m;
#     }
# }

# Start: docker run -p 8080:8080 -v /var/cache:/var/cache <nginx-image>

# Point Bazel at it:
bazel build //... --remote_cache=http://cache-server:8080/bazel-cache

# First build: all cache misses (uploads results)
# INFO: 200 actions, 0 cache hits

# Second build (same or different machine):
# INFO: 200 actions, 195 cache hits, 5 changed
# Build time: 45min → 3min

```

```python

# BUILD file — Bazel caches at the action level
cc_library(
    name = "mylib",
    srcs = ["mylib.cpp"],
    hdrs = ["mylib.h"],
    deps = ["@abseil-cpp//absl/strings"],
)

cc_binary(
    name = "myapp",
    srcs = ["main.cpp"],
    deps = [":mylib"],
)

cc_test(
    name = "mylib_test",
    srcs = ["mylib_test.cpp"],
    deps = [":mylib", "@googletest//:gtest_main"],
)

```

**How Bazel caching works:** Bazel computes a hash of each action (inputs + command + environment). If the hash exists in the remote cache, the outputs are downloaded instead of executing the action. This works for compilation, linking, test execution, and any custom rule.

### Q3: Measure cache hit ratio and build time improvement for a CI pipeline with remote caching

**Answer:**

```bash

# ═══════════ Measuring sccache performance ═══════════

# Reset stats before a build
sccache --zero-stats

# Build
cmake --build build --parallel $(nproc)

# Get stats
sccache --show-stats
# Compile requests                 250
# Compile requests executed         250
# Cache hits                        230 (92.0%)
# Cache misses                       20
# Cache hit rate                   92.0%
# Avg cache retrieval time         0.05s
# Avg compilation time             2.3s

# Calculate time savings:
# 230 cache hits × 2.3s avg compile = 529s saved
# 20 cache misses × 2.3s = 46s compilation
# Without cache: 250 × 2.3s = 575s (9.6 min)
# With cache: 46s + 230 × 0.05s = 57.5s (1 min)
# Speedup: ~10x

# ═══════════ Measuring Bazel cache performance ═══════════

# Bazel shows cache hits automatically:
bazel build //... 2>&1 | tail -5
# INFO: Elapsed time: 45.2s, Critical Path: 12.3s
# INFO: 200 processes: 195 remote cache hit, 5 internal

# For detailed stats:
bazel build //... --build_event_json_file=build_events.json
# Parse the JSON for action cache statistics

# ═══════════ CI pipeline timing comparison ═══════════
# Track in CI with timestamps:

```

```yaml

# GitHub Actions with timing
jobs:
  build-with-cache:
    runs-on: ubuntu-latest
    steps:

      - uses: actions/checkout@v4

      - name: Setup sccache

        run: |
          curl -L https://github.com/mozilla/sccache/releases/download/v0.8.1/sccache-v0.8.1-x86_64-unknown-linux-musl.tar.gz \
            | sudo tar xz -C /usr/local/bin --strip-components=1
          sccache --start-server
        env:
          SCCACHE_BUCKET: ${{ secrets.SCCACHE_BUCKET }}
          AWS_ACCESS_KEY_ID: ${{ secrets.AWS_ACCESS_KEY_ID }}
          AWS_SECRET_ACCESS_KEY: ${{ secrets.AWS_SECRET_ACCESS_KEY }}

      - name: Build

        run: |
          START=$(date +%s)
          cmake -B build -DCMAKE_CXX_COMPILER_LAUNCHER=sccache -DCMAKE_BUILD_TYPE=Release
          cmake --build build -j2
          END=$(date +%s)
          echo "Build time: $((END - START)) seconds" >> $GITHUB_STEP_SUMMARY

      - name: Cache statistics

        run: |
          sccache --show-stats | tee -a $GITHUB_STEP_SUMMARY
          # Write stats to job summary for easy viewing

```

**Expected results after cache is warm:**

| Metric | Without cache | With sccache | With Bazel remote |
| --- | --- | --- | --- |
| Full rebuild | 30 min | 30 min (cold) | 30 min (cold) |
| Incremental (5 files changed) | 5 min | 1 min | 30 sec |
| No-change rebuild | 3 min (ccache local) | 30 sec | 10 sec |
| Cache hit rate | N/A | 80-95% | 85-99% |

---

## Notes

- **sccache vs ccache:** sccache supports remote backends natively; ccache requires third-party solutions for remote caching (Redis via `ccache --set-config=remote_storage=redis://...` in ccache 4.8+)
- **Cache invalidation:** changing compiler version, flags, or system headers invalidates all cache entries — always pin compiler versions
- **S3 lifecycle policy:** set a 30-day expiration on the cache bucket to prevent unbounded growth
- **Security:** cache servers should be read-only for untrusted PRs (prevent cache poisoning). Use `--remote_upload_local_results=false` for forked PR builds
- **sccache limitations:** doesn't cache linking, only compilation. For link-time caching, use Bazel or incremental linking
