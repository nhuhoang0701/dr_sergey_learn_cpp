# Use non-temporal stores for write-only large buffers to bypass cache

**Category:** Performance & CPU Architecture  
**Item:** #720  
**Reference:** <https://gcc.gnu.org/onlinedocs/gcc/Other-Builtins.html>  

---

## Topic Overview

Normal stores follow the cache hierarchy: read the cache line into L1, modify it, write back. For write-only buffers larger than the cache, this wastes bandwidth reading data we'll immediately overwrite. Non-temporal stores bypass the cache, writing directly to memory.

```cpp

Temporal store (normal):             Non-temporal store:

  1. Read cache line (64B from DRAM)   1. Write directly to write-combine buffer
  2. Modify bytes in cache             2. Flush full 64B line to DRAM
  3. Evict old cache line              (skips read + skips cache pollution)

  READ bandwidth: wasted!              READ bandwidth: saved!
  Cache: polluted with write-only data Cache: untouched!

```

| Feature | Temporal store | Non-temporal store |
| --- | --- | --- |
| Cache read-for-ownership | Yes (wastes bandwidth) | No |
| Cache pollution | Yes | No |
| Write ordering | Strongly ordered | Weakly ordered (need `_mm_sfence`) |
| Best for | Data reused soon | Write-once, large buffers |
| Throughput (memset 1GB) | ~12 GB/s | ~20 GB/s |

---

## Self-Assessment

### Q1: Non-temporal store with `_mm_stream_si128`

```cpp

#include <iostream>
#include <chrono>
#include <immintrin.h>  // SSE/AVX intrinsics
#include <cstring>
#include <cstdlib>

// Normal memset: reads cache lines then writes (wasted read bandwidth)
void normal_fill(int* data, int val, size_t count) {
    for (size_t i = 0; i < count; ++i)
        data[i] = val;
    // Compiler generates: mov [rdi + rcx*4], eax
    // CPU internally: read 64B into cache, modify 4B, may evict useful data
}

// Non-temporal memset: writes bypass cache entirely
void nt_fill(int* data, int val, size_t count) {
    __m128i v = _mm_set1_epi32(val);  // splat val to 4 ints
    size_t i = 0;
    for (; i + 4 <= count; i += 4) {
        _mm_stream_si128(reinterpret_cast<__m128i*>(data + i), v);
        // movntdq [rdi], xmm0   (non-temporal, bypasses cache)
    }
    // Handle remainder
    for (; i < count; ++i)
        data[i] = val;
    _mm_sfence();  // Ensure all NT stores are globally visible
    // sfence is REQUIRED: NT stores are weakly ordered!
}

// AVX2 version: 32 bytes per store (8 ints)
void nt_fill_avx2(int* data, int val, size_t count) {
    __m256i v = _mm256_set1_epi32(val);
    size_t i = 0;
    for (; i + 8 <= count; i += 8) {
        _mm256_stream_si256(reinterpret_cast<__m256i*>(data + i), v);
        // vmovntdq [rdi], ymm0
    }
    for (; i < count; ++i) data[i] = val;
    _mm_sfence();
}

int main() {
    constexpr size_t N = 256 * 1024 * 1024 / sizeof(int); // 256 MB
    // Must be 32-byte aligned for AVX2 streaming stores
    int* data = static_cast<int*>(std::aligned_alloc(32, N * sizeof(int)));

    auto bench = [&](auto fn, const char* label) {
        auto t0 = std::chrono::high_resolution_clock::now();
        for (int r = 0; r < 10; ++r) fn(data, 42, N);
        auto t1 = std::chrono::high_resolution_clock::now();
        auto ms = std::chrono::duration_cast<std::chrono::milliseconds>(t1 - t0);
        double gbps = (N * sizeof(int) * 10.0) / (ms.count() * 1e6);
        std::cout << label << ": " << ms.count() << " ms (" << gbps << " GB/s)\n";
    };

    bench(normal_fill,   "Normal   ");
    bench(nt_fill,       "NT SSE4  ");
    bench(nt_fill_avx2,  "NT AVX2  ");
    // Typical DDR4 results:
    //   Normal:  ~220ms (12 GB/s) - half bandwidth wasted on reads
    //   NT SSE4: ~130ms (20 GB/s)
    //   NT AVX2: ~125ms (21 GB/s)

    std::free(data);
}

```

### Q2: Why non-temporal stores avoid cache pollution

```cpp

#include <iostream>
#include <vector>
#include <chrono>
#include <immintrin.h>

// Scenario: process hot data in a loop, then write results to output buffer.
// If output buffer pollutes cache, hot data gets evicted!

void process_with_temporal(const float* hot, float* output, int n) {
    for (int i = 0; i < n; ++i) {
        // Read from hot array (should stay in cache)
        float val = hot[i % 1024] * 2.0f;  // reuses same 4KB

        // Temporal store to output: POLLUTES cache!
        output[i] = val;
        // This evicts hot[...] from L1 cache
        // Next iteration: hot[...] must be reloaded from L2/L3
    }
}

void process_with_nt(const float* hot, float* output, int n) {
    for (int i = 0; i < n; ++i) {
        float val = hot[i % 1024] * 2.0f;  // stays in L1!

        // Non-temporal store: BYPASSES cache entirely
        _mm_stream_si32(reinterpret_cast<int*>(output + i),
                        *reinterpret_cast<int*>(&val));
        // Output data never enters L1 -> hot data stays resident
    }
    _mm_sfence();
}

int main() {
    constexpr int N = 100'000'000;
    std::vector<float> hot(1024, 1.5f);
    float* output = static_cast<float*>(
        std::aligned_alloc(16, N * sizeof(float)));

    auto bench = [&](auto fn, const char* label) {
        auto t0 = std::chrono::high_resolution_clock::now();
        fn(hot.data(), output, N);
        auto t1 = std::chrono::high_resolution_clock::now();
        auto ms = std::chrono::duration_cast<std::chrono::milliseconds>(t1 - t0);
        std::cout << label << ": " << ms.count() << " ms\n";
    };

    bench(process_with_temporal, "Temporal    ");
    bench(process_with_nt,      "Non-temporal");
    // Temporal:     ~300ms (hot data evicted, constant L1->L2 traffic)
    // Non-temporal:  ~180ms (hot data stays in L1, output goes to RAM)

    std::free(output);
}

```

### Q3: Benchmark video frame clear: temporal vs non-temporal

```cpp

#include <iostream>
#include <chrono>
#include <immintrin.h>
#include <cstdlib>
#include <cstring>

// Video frame: 1920x1080 RGBA = 8.3 MB
// At 60 FPS, we clear 500 MB/s. Cache pollution matters!

struct Frame {
    static constexpr int W = 1920, H = 1080, BPP = 4;
    static constexpr size_t SIZE = W * H * BPP;  // ~8.3 MB
    uint8_t* data;

    Frame() : data(static_cast<uint8_t*>(std::aligned_alloc(32, SIZE))) {}
    ~Frame() { std::free(data); }
};

void clear_temporal(Frame& f, uint8_t color) {
    std::memset(f.data, color, Frame::SIZE);
    // Uses rep stosb or temporal stores -> pollutes L1/L2/L3 with frame data
}

void clear_nt(Frame& f, uint8_t color) {
    __m256i c = _mm256_set1_epi8(static_cast<char>(color));
    uint8_t* p = f.data;
    for (size_t i = 0; i < Frame::SIZE; i += 32) {
        _mm256_stream_si256(reinterpret_cast<__m256i*>(p + i), c);
    }
    _mm_sfence();
    // Frame data never enters cache -> cache available for game logic
}

int main() {
    Frame f;
    constexpr int FRAMES = 600; // 10 seconds at 60fps

    auto bench = [&](auto fn, const char* label) {
        auto t0 = std::chrono::high_resolution_clock::now();
        for (int i = 0; i < FRAMES; ++i) fn(f, 0);
        auto t1 = std::chrono::high_resolution_clock::now();
        auto ms = std::chrono::duration_cast<std::chrono::milliseconds>(t1 - t0);
        double gbps = (Frame::SIZE / 1e9) * FRAMES / (ms.count() / 1000.0);
        std::cout << label << ": " << ms.count() << " ms"
                  << " (" << gbps << " GB/s)\n";
    };

    bench(clear_temporal, "memset (temporal)    ");
    bench(clear_nt,      "streaming (non-temp) ");
    // memset:    ~250ms, 20 GB/s but pollutes 8MB of L3 per frame
    // streaming: ~200ms, 25 GB/s and cache stays clean
    //
    // Measure cache pollution:
    //   perf stat -e LLC-load-misses,LLC-store-misses ./test
    //   temporal:     LLC-store-misses: 130,000 per frame
    //   non-temporal: LLC-store-misses:       0 per frame
}

```

---

## Notes

- **Always use `_mm_sfence()`** after non-temporal stores before reading the data.
- Non-temporal stores require 16/32-byte aligned addresses (`aligned_alloc`).
- Only use for write-only buffers larger than L3 cache (or when avoiding cache pollution matters).
- `memset` in glibc often uses NT stores automatically for large buffers (>= 256KB).
- Clang's `-fno-builtin-memset` + manual NT loop gives more control.
