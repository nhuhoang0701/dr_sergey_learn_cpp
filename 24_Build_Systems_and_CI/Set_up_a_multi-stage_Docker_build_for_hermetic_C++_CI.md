# Set up a multi-stage Docker build for hermetic C++ CI

**Category:** Build Systems & CI  
**Item:** #568  
**Reference:** <https://docs.docker.com/build/guide/multi-stage/>  

---

## Topic Overview

A **hermetic build** means every dependency — compiler, libraries, system headers — is locked inside a container. Multi-stage Docker builds separate the heavy toolchain image (builder) from a tiny runtime image (runner), producing small, reproducible artifacts.

### Multi-Stage Build Architecture

```cpp

┌───────────────────────────────────────────────────────┐
│ Stage 1: builder (ubuntu:22.04 + GCC + vcpkg + deps)  │
│  ├── Install compiler toolchain                        │
│  ├── Install vcpkg + project dependencies             │
│  ├── cmake --build → produces binary                  │
│  └── Image size: ~2-4 GB                              │
├───────────────────────────────────────────────────────┤
│ Stage 2: runner (ubuntu:22.04-minimal or distroless)   │
│  ├── COPY --from=builder /app/build/myapp /app/       │
│  ├── Only runtime libs (libstdc++, etc.)              │
│  └── Image size: ~50-150 MB                           │
└───────────────────────────────────────────────────────┘

```

### Why Hermetic Builds

| Benefit | Explanation |
| --- | --- |
| Reproducibility | Same compiler, same libs, same result — on any machine |
| No "works on my machine" | CI and dev use identical images |
| Small deployments | Runner image has only the binary + runtime deps |
| Security | Smaller attack surface (no compiler, no build tools in prod) |
| Caching | Docker layer cache speeds up rebuilds |

---

## Self-Assessment

### Q1: Write a Dockerfile with a builder stage (toolchain) and a runner stage (minimal binary)

**Answer:**

```dockerfile

# ═══════════ Stage 1: Builder ═══════════
FROM ubuntu:22.04 AS builder

# Prevent apt interactive prompts
ENV DEBIAN_FRONTEND=noninteractive

# Install build essentials + CMake + Ninja
RUN apt-get update && apt-get install -y --no-install-recommends \
    build-essential \
    gcc-12 g++-12 \
    cmake \
    ninja-build \
    git \
    ca-certificates \
    pkg-config \
    && rm -rf /var/lib/apt/lists/*

# Set GCC 12 as default
RUN update-alternatives --install /usr/bin/gcc gcc /usr/bin/gcc-12 100 \
    && update-alternatives --install /usr/bin/g++ g++ /usr/bin/g++-12 100

# Copy source code
WORKDIR /src
COPY CMakeLists.txt .
COPY src/ src/
COPY include/ include/
COPY tests/ tests/

# Configure and build
RUN cmake -B build \
    -G Ninja \
    -DCMAKE_BUILD_TYPE=Release \
    -DCMAKE_CXX_STANDARD=20 \
    -DBUILD_TESTING=ON

RUN cmake --build build --parallel $(nproc)

# Run tests inside the builder (fail build if tests fail)
RUN cd build && ctest --output-on-failure

# ═══════════ Stage 2: Runner ═══════════
FROM ubuntu:22.04 AS runner

RUN apt-get update && apt-get install -y --no-install-recommends \
    libstdc++6 \
    && rm -rf /var/lib/apt/lists/*

# Create non-root user
RUN useradd --create-home --shell /bin/bash appuser

WORKDIR /app
# Copy only the built binary from builder
COPY --from=builder /src/build/myapp .

# Run as non-root
USER appuser

ENTRYPOINT ["./myapp"]

```

```bash

# Build the image
docker build -t myapp:latest .

# Layer sizes:
# builder stage: ~2 GB (not shipped)
# runner stage:  ~80 MB (shipped to production)

# Run the app
docker run --rm myapp:latest --help

```

**Explanation:** The builder stage contains the full toolchain (~2 GB) and runs the build + tests. The runner stage uses `COPY --from=builder` to extract only the compiled binary. The toolchain is never shipped to production, reducing image size by ~95% and attack surface.

### Q2: Pin the compiler version and vcpkg baseline hash for a fully reproducible CI image

**Answer:**

```dockerfile

# ═══════════ Pinned, Reproducible Builder ═══════════
FROM ubuntu:22.04@sha256:abc123... AS builder
# ↑ Pin the base image by SHA256 digest, not just tag
# Tags like "22.04" can change when Ubuntu pushes updates

ENV DEBIAN_FRONTEND=noninteractive

# Pin exact compiler packages
RUN apt-get update && apt-get install -y --no-install-recommends \
    gcc-12=12.3.0-1ubuntu1~22.04 \
    g++-12=12.3.0-1ubuntu1~22.04 \
    cmake=3.22.1-1ubuntu1.22.04.2 \
    ninja-build=1.10.1-1 \
    git \
    ca-certificates \
    curl \
    zip unzip tar \
    pkg-config \
    && rm -rf /var/lib/apt/lists/*

# ═══════════ Pin vcpkg to exact commit ═══════════
ENV VCPKG_ROOT=/opt/vcpkg
ENV VCPKG_DEFAULT_BINARY_CACHE=/opt/vcpkg-cache

RUN git clone https://github.com/microsoft/vcpkg.git $VCPKG_ROOT \
    && cd $VCPKG_ROOT \
    && git checkout a1a1cbc975abf909a6c8985a6a2b8fe20bbd9bd6 \
    && ./bootstrap-vcpkg.sh -disableMetrics

RUN mkdir -p $VCPKG_DEFAULT_BINARY_CACHE

# Copy manifest first (for layer caching)
WORKDIR /src
COPY vcpkg.json vcpkg-configuration.json ./

# Install dependencies (cached unless vcpkg.json changes)
RUN $VCPKG_ROOT/vcpkg install --triplet x64-linux

# Copy source and build
COPY CMakeLists.txt .
COPY src/ src/
COPY include/ include/

RUN cmake -B build -G Ninja \
    -DCMAKE_BUILD_TYPE=Release \
    -DCMAKE_TOOLCHAIN_FILE=$VCPKG_ROOT/scripts/buildsystems/vcpkg.cmake \
    -DCMAKE_CXX_STANDARD=20

RUN cmake --build build --parallel $(nproc)

```

```json

// vcpkg-configuration.json — pin the baseline
{
  "default-registry": {
    "kind": "git",
    "baseline": "a1a1cbc975abf909a6c8985a6a2b8fe20bbd9bd6",
    "repository": "https://github.com/microsoft/vcpkg"
  }
}

```

**Every version is locked:**

- Base image: pinned by `@sha256:` digest
- Compiler: exact `apt` package versions (e.g., `gcc-12=12.3.0-...`)
- CMake: exact version
- vcpkg: pinned by git commit hash
- vcpkg packages: pinned by `vcpkg-configuration.json` baseline

### Q3: Use BuildKit cache mounts to cache vcpkg installs between CI runs

**Answer:**

```dockerfile

# syntax=docker/dockerfile:1.4
# ↑ Required for BuildKit features

FROM ubuntu:22.04 AS builder
ENV DEBIAN_FRONTEND=noninteractive

RUN apt-get update && apt-get install -y --no-install-recommends \
    build-essential gcc-12 g++-12 cmake ninja-build git curl \
    ca-certificates zip unzip tar pkg-config \
    && rm -rf /var/lib/apt/lists/*

ENV VCPKG_ROOT=/opt/vcpkg
RUN git clone https://github.com/microsoft/vcpkg.git $VCPKG_ROOT \
    && cd $VCPKG_ROOT \
    && git checkout a1a1cbc975abf909a6c8985a6a2b8fe20bbd9bd6 \
    && ./bootstrap-vcpkg.sh -disableMetrics

WORKDIR /src
COPY vcpkg.json vcpkg-configuration.json ./

# ═══════════ BuildKit cache mount for vcpkg ═══════════
# --mount=type=cache persists the directory across builds
# Even if the Docker layer cache is cleared, the mount survives
RUN --mount=type=cache,target=/opt/vcpkg-cache \
    VCPKG_DEFAULT_BINARY_CACHE=/opt/vcpkg-cache \
    $VCPKG_ROOT/vcpkg install --triplet x64-linux

COPY CMakeLists.txt .
COPY src/ src/
COPY include/ include/

# ═══════════ BuildKit cache mount for CMake build ═══════════
# Cache the build directory for incremental compilation
RUN --mount=type=cache,target=/src/build-cache \
    cmake -B build-cache -G Ninja \
    -DCMAKE_BUILD_TYPE=Release \
    -DCMAKE_TOOLCHAIN_FILE=$VCPKG_ROOT/scripts/buildsystems/vcpkg.cmake \
    && cmake --build build-cache --parallel $(nproc) \
    && cp build-cache/myapp /src/myapp

FROM ubuntu:22.04 AS runner
RUN apt-get update && apt-get install -y --no-install-recommends libstdc++6 \
    && rm -rf /var/lib/apt/lists/*
COPY --from=builder /src/myapp /app/myapp
ENTRYPOINT ["/app/myapp"]

```

```bash

# Build with BuildKit enabled
DOCKER_BUILDKIT=1 docker build -t myapp:latest .

# Or in CI (GitHub Actions):
# - name: Build Docker image
#   env:
#     DOCKER_BUILDKIT: 1
#   run: docker build -t myapp:latest

# Cache mount benefits:
# First build:  vcpkg install → 5 min (downloads + builds)
# Second build:  vcpkg install → 10 sec (binary cache hit)
# CMake:         Only recompiles changed files

```

**BuildKit cache mount vs layer cache:**

- **Layer cache** is invalidated when ANY earlier layer changes (e.g., new apt package)
- **Cache mount** persists independently — survives layer invalidation
- `--mount=type=cache,target=/path` mounts a persistent volume at that path during the RUN command
- vcpkg binary cache + CMake object cache together save 5-10 minutes per CI run

---

## Notes

- **Distroless runner** (`gcr.io/distroless/cc-debian12`) has no shell at all — smallest possible image but harder to debug
- **Security scanning:** Run `docker scout cves myapp:latest` or Trivy to check for CVEs in the runner image
- **`.dockerignore`** is critical — exclude `.git/`, `build/`, `node_modules/` to keep the build context small
- **Layer ordering:** Put rarely-changing steps (apt install) first, frequently-changing (COPY source) last to maximize cache reuse
- **Multi-platform:** Use `docker buildx build --platform linux/amd64,linux/arm64` for cross-platform images
