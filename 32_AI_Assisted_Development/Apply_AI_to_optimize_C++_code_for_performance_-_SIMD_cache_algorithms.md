# Apply AI to optimize C++ code for performance - SIMD, cache, algorithms

**Category:** AI-Assisted C++ Development

---

## Topic Overview

AI assistants can analyze C++ code for performance bottlenecks and suggest optimizations including **SIMD vectorization**, **cache-friendly data layouts**, **algorithm selection**, and **micro-optimizations**. AI is especially valuable for SIMD because intrinsics are hard to write correctly and vary across architectures.

### AI Performance Optimization Areas

| Optimization | AI Strength | Human Review Needed |
| --- | --- | --- |
| Algorithm complexity improvement | Excellent | Low |
| SIMD intrinsic generation | Good | Medium (verify correctness) |
| Cache-friendly struct layout | Good | Low |
| Branch prediction hints | Medium | Medium |
| Memory allocation strategy | Good | Medium |
| Lock-free data structures | Medium | High |
| Compiler-specific pragmas | Good | Medium |

---

## Self-Assessment

### Q1: Use AI to generate SIMD-optimized code

**Answer:**

```cpp

=== PROMPT ===

"Optimize this function using AVX2 intrinsics.
Target: x86-64 with AVX2 support.
Input: arrays of float, guaranteed 32-byte aligned,
length is a multiple of 8.
Also provide a scalar fallback."

```

```cpp

// Scalar version (slow):
void multiply_add(const float* a, const float* b,
                  const float* c, float* result, size_t n) {
    for (size_t i = 0; i < n; ++i) {
        result[i] = a[i] * b[i] + c[i];
    }
}

// === AI-generated AVX2 version ===
#include <immintrin.h>

void multiply_add_avx2(
    const float* __restrict a,
    const float* __restrict b,
    const float* __restrict c,
    float* __restrict result,
    size_t n) {

    size_t i = 0;

    // Process 8 floats per iteration (256 bits / 32 bits)
    for (; i + 8 <= n; i += 8) {
        __m256 va = _mm256_load_ps(a + i);   // Load 8 floats
        __m256 vb = _mm256_load_ps(b + i);
        __m256 vc = _mm256_load_ps(c + i);

        // FMA: result = a * b + c (single instruction)
        __m256 vr = _mm256_fmadd_ps(va, vb, vc);

        _mm256_store_ps(result + i, vr);     // Store 8 floats
    }

    // Scalar tail (if n not multiple of 8)
    for (; i < n; ++i) {
        result[i] = a[i] * b[i] + c[i];
    }
}

// With runtime dispatch:
void multiply_add_dispatch(
    const float* a, const float* b,
    const float* c, float* result, size_t n) {
#if defined(__AVX2__) && defined(__FMA__)
    multiply_add_avx2(a, b, c, result, n);
#else
    multiply_add(a, b, c, result, n);
#endif
}

```

### Q2: Use AI for cache-friendly optimizations

**Answer:**

```cpp

=== PROMPT ===

"This struct has poor cache performance.
Reorganize for cache-friendly access patterns.
The hot path reads: position, velocity, alive flag.
The cold path reads: name, creation_time, metadata.
Array of 100,000 entities iterated every frame (60fps)."

```

```cpp

// BAD: All data interleaved (cache-unfriendly)
struct Entity {
    std::string name;            // 32 bytes (cold)
    double position[3];          // 24 bytes (hot)
    double velocity[3];          // 24 bytes (hot)
    std::string metadata;        // 32 bytes (cold)
    time_point creation_time;    // 8 bytes (cold)
    bool alive;                  // 1 byte (hot)
    // 7 bytes padding
    // Total: ~128 bytes per entity
    // Cache line (64B) holds ~0.5 entities
};
std::vector<Entity> entities(100'000);
// Hot loop touches 128 * 100K = 12.8MB -> cache thrashing


// GOOD: SoA (Structure of Arrays) - AI-suggested
struct EntityData {
    // Hot data (accessed every frame)
    struct Hot {
        std::vector<std::array<float, 3>> positions;   // 12B each
        std::vector<std::array<float, 3>> velocities;  // 12B each
        std::vector<bool_packed> alive;                 // 1 bit each
    } hot;

    // Cold data (accessed rarely)
    struct Cold {
        std::vector<std::string> names;
        std::vector<std::string> metadata;
        std::vector<time_point> creation_times;
    } cold;

    size_t count = 0;
};

// Hot loop now touches: 12 + 12 = 24 bytes per entity
// 24 * 100K = 2.4MB -> fits in L2 cache!
// ~5x less memory bandwidth than AoS version


// BETTER: SoA with SIMD-friendly alignment
struct alignas(32) SimdPositions {
    // x, y, z stored separately for SIMD
    std::vector<float> x, y, z;  // Each contiguous
};

// Now we can process 8 entities at once with AVX:
void update_positions_simd(SimdPositions& pos,
                           const SimdPositions& vel,
                           float dt, size_t n) {
    __m256 vdt = _mm256_set1_ps(dt);
    for (size_t i = 0; i + 8 <= n; i += 8) {
        __m256 px = _mm256_load_ps(&pos.x[i]);
        __m256 vx = _mm256_load_ps(&vel.x[i]);
        px = _mm256_fmadd_ps(vx, vdt, px);  // pos += vel * dt
        _mm256_store_ps(&pos.x[i], px);
        // Same for y, z...
    }
}

```

### Q3: Use AI for algorithm optimization

**Answer:**

```cpp

=== PROMPT ===

"This function is called 10M times/sec in our hot path.
Profile shows it takes 40% of CPU time.
Suggest algorithmic and micro-optimizations.
Target: x86-64, GCC 13, -O3."

```

```cpp

// Original: O(n) linear search, called frequently
bool contains_any(const std::vector<int>& haystack,
                  const std::vector<int>& needles) {
    for (int needle : needles) {
        for (int h : haystack) {
            if (h == needle) return true;
        }
    }
    return false;
}

// === AI-suggested optimizations ===

// Option 1: Sort + binary search - O(n log n) once, O(m log n) per call
bool contains_any_sorted(std::span<const int> haystack,  // Pre-sorted!
                         std::span<const int> needles) {
    for (int needle : needles) {
        if (std::binary_search(haystack.begin(), haystack.end(), needle))
            return true;
    }
    return false;
}

// Option 2: Hash set - O(n) build, O(m) lookup
bool contains_any_hash(std::span<const int> haystack,
                       std::span<const int> needles) {
    // Use flat_hash_set for cache-friendly buckets
    absl::flat_hash_set<int> set(haystack.begin(), haystack.end());
    for (int needle : needles) {
        if (set.contains(needle)) return true;
    }
    return false;
}

// Option 3: Bloom filter for approximate pre-check
class BloomFilter {
    std::array<uint64_t, 4> bits_{};  // 256 bits
public:
    void insert(int val) {
        uint32_t h = std::hash<int>{}(val);
        bits_[0] |= 1ULL << (h & 63);
        bits_[1] |= 1ULL << ((h >> 6) & 63);
    }
    bool maybe_contains(int val) const {
        uint32_t h = std::hash<int>{}(val);
        return (bits_[0] & (1ULL << (h & 63))) &&
               (bits_[1] & (1ULL << ((h >> 6) & 63)));
    }
};

// Option 4: Branchless with SIMD (for small needles)
bool contains_any_simd(const int* haystack, size_t hlen,
                       int needle) {
    __m256i vneedle = _mm256_set1_epi32(needle);
    size_t i = 0;
    for (; i + 8 <= hlen; i += 8) {
        __m256i chunk = _mm256_loadu_si256(
            reinterpret_cast<const __m256i*>(&haystack[i]));
        __m256i cmp = _mm256_cmpeq_epi32(chunk, vneedle);
        if (_mm256_movemask_epi8(cmp) != 0) return true;
    }
    for (; i < hlen; ++i)
        if (haystack[i] == needle) return true;
    return false;
}

// AI recommendation:
// - If haystack changes rarely: pre-sort + binary search
// - If haystack changes often: flat_hash_set
// - If haystack small (<32): SIMD scan
// - If approximate OK: Bloom filter pre-check

```

---

## Notes

- Always **profile first** — ask AI to suggest profiling commands (`perf`, `VTune`, `Instruments`)
- AI-generated SIMD code needs **testing on target hardware** — it may use unavailable instructions
- For cache optimization, tell AI the **data size and access pattern** (random vs sequential)
- Ask AI to generate **benchmark code** (Google Benchmark) alongside optimizations
- SIMD intrinsics vary by architecture — specify the **target ISA** (SSE4.2, AVX2, AVX-512, NEON)
- AI is excellent at **AoS → SoA** transformations but may miss aliasing issues
- For `[[likely]]`/`[[unlikely]]`, ask AI to analyze **branch prediction** based on expected data distribution
