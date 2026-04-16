# Test memory-constrained embedded code with custom allocators

**Category:** Testing in Practice

---

## Topic Overview

Embedded systems often have severe memory constraints (KB of RAM, no heap). Testing memory-constrained code requires: **custom allocators** that model real hardware limits, **monitoring allocators** that track usage, and **tests that verify allocation patterns** don't exceed budgets.

### Memory Testing Strategies

| Strategy | What It Validates | Approach |
| --- | --- | --- |
| **Bounded allocator** | Code stays within RAM budget | Allocator that fails at limit |
| **Counting allocator** | No unexpected heap use | Track alloc/free count |
| **Leak detector** | All memory freed | Compare alloc vs free at test end |
| **Fragmentation test** | Pool doesn't fragment | Stress alloc/free patterns |
| **Stack analysis** | No stack overflow | Static analysis + runtime check |

---

## Self-Assessment

### Q1: Build monitoring and bounded allocators for testing

**Answer:**

```cpp

#include <cstddef>
#include <cstdlib>
#include <cassert>
#include <map>
#include <stdexcept>
#include <memory>
#include <vector>

// === Monitoring allocator: tracks all allocations ===
class MonitoringAllocator {
public:
    struct Stats {
        size_t total_allocated = 0;
        size_t total_freed = 0;
        size_t current_usage = 0;
        size_t peak_usage = 0;
        size_t alloc_count = 0;
        size_t free_count = 0;
    };

    void* allocate(size_t size) {
        void* ptr = std::malloc(size);
        if (!ptr) throw std::bad_alloc();

        allocations_[ptr] = size;
        stats_.total_allocated += size;
        stats_.current_usage += size;
        stats_.alloc_count++;
        if (stats_.current_usage > stats_.peak_usage)
            stats_.peak_usage = stats_.current_usage;
        return ptr;
    }

    void deallocate(void* ptr) {
        auto it = allocations_.find(ptr);
        if (it == allocations_.end())
            throw std::runtime_error("Double free or invalid free");

        stats_.total_freed += it->second;
        stats_.current_usage -= it->second;
        stats_.free_count++;
        allocations_.erase(it);
        std::free(ptr);
    }

    const Stats& stats() const { return stats_; }
    bool has_leaks() const { return !allocations_.empty(); }
    size_t leak_count() const { return allocations_.size(); }

    void reset() {
        // Free any leaked memory
        for (auto& [ptr, size] : allocations_)
            std::free(ptr);
        allocations_.clear();
        stats_ = {};
    }

private:
    std::map<void*, size_t> allocations_;
    Stats stats_;
};

// === Bounded allocator: simulates limited RAM ===
class BoundedAllocator {
public:
    explicit BoundedAllocator(size_t max_bytes)
        : max_bytes_(max_bytes) {}

    void* allocate(size_t size) {
        if (used_ + size > max_bytes_)
            return nullptr;  // Out of memory (no throw — embedded style)

        void* ptr = std::malloc(size);
        if (!ptr) return nullptr;

        allocations_[ptr] = size;
        used_ += size;
        return ptr;
    }

    void deallocate(void* ptr) {
        auto it = allocations_.find(ptr);
        if (it != allocations_.end()) {
            used_ -= it->second;
            allocations_.erase(it);
        }
        std::free(ptr);
    }

    size_t used() const { return used_; }
    size_t available() const { return max_bytes_ - used_; }

private:
    size_t max_bytes_;
    size_t used_ = 0;
    std::map<void*, size_t> allocations_;
};

```

### Q2: Test embedded data structures within memory budgets

**Answer:**

```cpp

#include <gtest/gtest.h>
#include <array>
#include <cstdint>

// === Fixed-capacity static vector (no heap) ===
template<typename T, size_t Capacity>
class StaticVector {
public:
    bool push_back(const T& value) {
        if (size_ >= Capacity) return false;
        data_[size_++] = value;
        return true;
    }

    bool pop_back() {
        if (size_ == 0) return false;
        size_--;
        return true;
    }

    T& operator[](size_t i) { return data_[i]; }
    const T& operator[](size_t i) const { return data_[i]; }

    size_t size() const { return size_; }
    size_t capacity() const { return Capacity; }
    bool empty() const { return size_ == 0; }
    bool full() const { return size_ == Capacity; }

    T* begin() { return data_.data(); }
    T* end() { return data_.data() + size_; }

private:
    std::array<T, Capacity> data_{};
    size_t size_ = 0;
};

// === Pool allocator (fixed-size blocks, O(1) alloc/free) ===
template<size_t BlockSize, size_t NumBlocks>
class PoolAllocator {
public:
    PoolAllocator() {
        // Build free list
        for (size_t i = 0; i < NumBlocks; ++i) {
            auto* block = reinterpret_cast<FreeNode*>(&pool_[i * BlockSize]);
            block->next = free_list_;
            free_list_ = block;
        }
    }

    void* allocate() {
        if (!free_list_) return nullptr;
        auto* block = free_list_;
        free_list_ = block->next;
        allocated_++;
        return block;
    }

    void deallocate(void* ptr) {
        auto* node = static_cast<FreeNode*>(ptr);
        node->next = free_list_;
        free_list_ = node;
        allocated_--;
    }

    size_t allocated() const { return allocated_; }
    size_t available() const { return NumBlocks - allocated_; }

private:
    struct FreeNode { FreeNode* next; };
    alignas(std::max_align_t) uint8_t pool_[BlockSize * NumBlocks]{};
    FreeNode* free_list_ = nullptr;
    size_t allocated_ = 0;
};

// === Tests ===
TEST(StaticVectorTest, RespectsCapacity) {
    StaticVector<int, 4> vec;
    EXPECT_TRUE(vec.push_back(1));
    EXPECT_TRUE(vec.push_back(2));
    EXPECT_TRUE(vec.push_back(3));
    EXPECT_TRUE(vec.push_back(4));
    EXPECT_FALSE(vec.push_back(5));  // Full!
    EXPECT_EQ(vec.size(), 4);
}

TEST(StaticVectorTest, NoHeapAllocations) {
    MonitoringAllocator monitor;
    // StaticVector is fully stack-allocated;
    // verify sizeof is exactly what we expect
    StaticVector<int, 10> vec;
    constexpr size_t expected = sizeof(std::array<int, 10>) + sizeof(size_t);
    EXPECT_EQ(sizeof(vec), expected);
}

TEST(PoolAllocatorTest, AllocatesUpToCapacity) {
    PoolAllocator<64, 8> pool;
    std::vector<void*> ptrs;

    for (int i = 0; i < 8; ++i) {
        void* p = pool.allocate();
        ASSERT_NE(p, nullptr) << "Failed at allocation " << i;
        ptrs.push_back(p);
    }

    EXPECT_EQ(pool.allocate(), nullptr);  // Exhausted
    EXPECT_EQ(pool.allocated(), 8);

    // Free all
    for (auto* p : ptrs) pool.deallocate(p);
    EXPECT_EQ(pool.allocated(), 0);
}

TEST(PoolAllocatorTest, ReuseFreedBlocks) {
    PoolAllocator<32, 2> pool;
    void* p1 = pool.allocate();
    void* p2 = pool.allocate();
    EXPECT_EQ(pool.allocate(), nullptr);

    pool.deallocate(p1);
    void* p3 = pool.allocate();
    EXPECT_NE(p3, nullptr);  // Reused p1's block
}

TEST(BoundedAllocatorTest, EnforcesLimit) {
    BoundedAllocator alloc(1024);  // 1KB limit

    void* p1 = alloc.allocate(512);
    EXPECT_NE(p1, nullptr);
    EXPECT_EQ(alloc.used(), 512);

    void* p2 = alloc.allocate(600);
    EXPECT_EQ(p2, nullptr);  // Over budget!

    void* p3 = alloc.allocate(500);
    EXPECT_NE(p3, nullptr);  // Fits

    alloc.deallocate(p1);
    alloc.deallocate(p3);
}

```

### Q3: Detect memory leaks and verify peak usage in tests

**Answer:**

```cpp

#include <gtest/gtest.h>

// === Test fixture that checks for leaks on teardown ===
class MemoryAwareTest : public ::testing::Test {
protected:
    void TearDown() override {
        EXPECT_FALSE(monitor_.has_leaks())
            << "Memory leak: " << monitor_.leak_count() << " allocations not freed";

        auto& stats = monitor_.stats();
        EXPECT_EQ(stats.current_usage, 0)
            << "Leaked " << stats.current_usage << " bytes";

        // Optionally verify peak doesn't exceed budget
        if (max_peak_bytes_ > 0) {
            EXPECT_LE(stats.peak_usage, max_peak_bytes_)
                << "Peak usage " << stats.peak_usage
                << " exceeds budget " << max_peak_bytes_;
        }

        monitor_.reset();
    }

    MonitoringAllocator monitor_;
    size_t max_peak_bytes_ = 0;  // 0 = no limit
};

class FirmwareBufferManager {
public:
    explicit FirmwareBufferManager(MonitoringAllocator& alloc)
        : alloc_(alloc) {}

    void* acquire(size_t size) {
        return alloc_.allocate(size);
    }

    void release(void* ptr) {
        alloc_.deallocate(ptr);
    }

private:
    MonitoringAllocator& alloc_;
};

class BufferManagerTest : public MemoryAwareTest {
protected:
    void SetUp() override {
        max_peak_bytes_ = 2048;  // Budget: 2KB peak
        mgr_ = std::make_unique<FirmwareBufferManager>(monitor_);
    }

    std::unique_ptr<FirmwareBufferManager> mgr_;
};

TEST_F(BufferManagerTest, AcquireAndReleaseNoLeaks) {
    auto* buf = mgr_->acquire(256);
    ASSERT_NE(buf, nullptr);
    mgr_->release(buf);
    // TearDown verifies no leaks
}

TEST_F(BufferManagerTest, StaysWithinPeakBudget) {
    std::vector<void*> buffers;
    for (int i = 0; i < 4; ++i)
        buffers.push_back(mgr_->acquire(256));  // 4 * 256 = 1024 < 2048

    for (auto* p : buffers)
        mgr_->release(p);
    // TearDown checks peak <= 2048
}

```

---

## Notes

- **Static containers** (StaticVector, static_string) avoid heap entirely — preferred for embedded
- **Pool allocators** provide O(1) alloc/free with zero fragmentation for fixed-size objects
- Use `MonitoringAllocator` in test fixtures to enforce memory budgets and detect leaks
- On real hardware, use `-fstack-usage` (GCC) and link-time stack analysis to verify stack sizes
- `new` and `delete` can be overridden globally in test builds to route through monitoring allocators
- Compile with `-fno-exceptions -fno-rtti` in embedded builds; test both configurations
- Memory fragmentation tests: allocate/free in random patterns, verify pool still works
- `sizeof()` tests verify that data structures match expected memory layout
