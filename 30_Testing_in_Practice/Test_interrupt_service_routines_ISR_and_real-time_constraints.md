# Test interrupt service routines (ISR) and real-time constraints

**Category:** Testing in Practice

---

## Topic Overview

**Interrupt Service Routines (ISRs)** are among the hardest embedded code to test. They execute asynchronously, have strict timing requirements, can't use blocking operations, and interact with hardware registers. Testing strategies: extract logic from ISRs into testable functions, simulate interrupt-driven behavior on host, and verify timing constraints.

### ISR Testing Challenges

| Challenge | Problem | Testing Strategy |
| --- | --- | --- |
| Asynchronous execution | Non-deterministic timing | Simulate with direct function calls |
| No blocking allowed | Can't use mutex, malloc, iostream | Static analysis + review |
| Shared state with main | Data races | Test with TSan, atomic verification |
| Hardware registers | Memory-mapped I/O | HAL abstraction + mocks |
| Timing constraints | Must complete in µs | Benchmark critical sections |
| Reentrance | Nested interrupts | Priority-based test scenarios |

---

## Self-Assessment

### Q1: Extract testable logic from ISRs

**Answer:**

```cpp

// === BAD: Untestable ISR with embedded logic ===
// Can't test this on host — depends on hardware registers
// void TIM2_IRQHandler() {
//     if (TIM2->SR & TIM_SR_UIF) {
//         TIM2->SR &= ~TIM_SR_UIF;
//         static int count = 0;
//         count++;
//         if (count >= 1000) {
//             GPIOA->ODR ^= (1 << 5);  // Toggle LED
//             count = 0;
//         }
//     }
// }

// === GOOD: Separate logic from hardware ===

// Pure logic, fully testable on host
class BlinkController {
public:
    explicit BlinkController(uint32_t period_ticks)
        : period_(period_ticks) {}

    // Returns true when LED should toggle
    bool tick() {
        if (++count_ >= period_) {
            count_ = 0;
            return true;
        }
        return false;
    }

    void reset() { count_ = 0; }
    uint32_t count() const { return count_; }

private:
    uint32_t period_;
    uint32_t count_ = 0;
};

// === Thin ISR wrapper (not unit-tested, but trivially correct) ===
// In production:
// BlinkController blink_ctrl(1000);
// void TIM2_IRQHandler() {
//     timer_hal.clear_interrupt();
//     if (blink_ctrl.tick()) {
//         gpio_hal.toggle(LED_PIN);
//     }
// }

```

```cpp

// === Tests for extracted logic ===
#include <gtest/gtest.h>

TEST(BlinkControllerTest, TogglesAtPeriod) {
    BlinkController ctrl(100);

    // First 99 ticks: no toggle
    for (int i = 0; i < 99; ++i)
        EXPECT_FALSE(ctrl.tick()) << "Tick " << i;

    // 100th tick: toggle!
    EXPECT_TRUE(ctrl.tick());

    // Counter resets
    EXPECT_EQ(ctrl.count(), 0);
}

TEST(BlinkControllerTest, Period1TogglesEveryTick) {
    BlinkController ctrl(1);
    EXPECT_TRUE(ctrl.tick());
    EXPECT_TRUE(ctrl.tick());
    EXPECT_TRUE(ctrl.tick());
}

TEST(BlinkControllerTest, ResetClearsCounter) {
    BlinkController ctrl(100);
    for (int i = 0; i < 50; ++i) ctrl.tick();
    ctrl.reset();
    EXPECT_EQ(ctrl.count(), 0);
}

```

### Q2: Test ISR-to-main communication (ring buffers, flags)

**Answer:**

```cpp

#include <gtest/gtest.h>
#include <atomic>
#include <thread>
#include <array>
#include <cstdint>

// === Lock-free ring buffer for ISR-to-main communication ===
template<typename T, size_t N>
class RingBuffer {
    static_assert((N & (N - 1)) == 0, "N must be power of 2");
public:
    bool push(const T& item) {
        auto head = head_.load(std::memory_order_relaxed);
        auto next = (head + 1) & (N - 1);
        if (next == tail_.load(std::memory_order_acquire))
            return false;  // Full
        buffer_[head] = item;
        head_.store(next, std::memory_order_release);
        return true;
    }

    bool pop(T& item) {
        auto tail = tail_.load(std::memory_order_relaxed);
        if (tail == head_.load(std::memory_order_acquire))
            return false;  // Empty
        item = buffer_[tail];
        tail_.store((tail + 1) & (N - 1), std::memory_order_release);
        return true;
    }

    bool empty() const {
        return head_.load(std::memory_order_acquire)
            == tail_.load(std::memory_order_acquire);
    }

    size_t size() const {
        auto h = head_.load(std::memory_order_acquire);
        auto t = tail_.load(std::memory_order_acquire);
        return (h - t) & (N - 1);
    }

private:
    std::array<T, N> buffer_{};
    std::atomic<size_t> head_{0};
    std::atomic<size_t> tail_{0};
};

// === Unit tests ===
TEST(RingBufferTest, EmptyOnCreation) {
    RingBuffer<int, 8> rb;
    EXPECT_TRUE(rb.empty());
    EXPECT_EQ(rb.size(), 0);
}

TEST(RingBufferTest, PushPopRoundtrip) {
    RingBuffer<int, 8> rb;
    EXPECT_TRUE(rb.push(42));
    EXPECT_EQ(rb.size(), 1);

    int val;
    EXPECT_TRUE(rb.pop(val));
    EXPECT_EQ(val, 42);
    EXPECT_TRUE(rb.empty());
}

TEST(RingBufferTest, FullReturnsFalse) {
    RingBuffer<int, 4> rb;  // Capacity = 3 (one slot reserved)
    EXPECT_TRUE(rb.push(1));
    EXPECT_TRUE(rb.push(2));
    EXPECT_TRUE(rb.push(3));
    EXPECT_FALSE(rb.push(4));  // Full!
}

TEST(RingBufferTest, PopEmptyReturnsFalse) {
    RingBuffer<int, 4> rb;
    int val;
    EXPECT_FALSE(rb.pop(val));
}

TEST(RingBufferTest, FIFOOrder) {
    RingBuffer<int, 8> rb;
    rb.push(1); rb.push(2); rb.push(3);
    int val;
    rb.pop(val); EXPECT_EQ(val, 1);
    rb.pop(val); EXPECT_EQ(val, 2);
    rb.pop(val); EXPECT_EQ(val, 3);
}

// === Stress test simulating ISR producer + main consumer ===
TEST(RingBufferTest, ConcurrentProducerConsumer) {
    RingBuffer<uint32_t, 256> rb;
    constexpr uint32_t COUNT = 100000;
    std::atomic<uint32_t> consumed{0};
    std::atomic<bool> done{false};

    // Simulated ISR (producer thread)
    auto producer = std::thread([&] {
        for (uint32_t i = 0; i < COUNT; ++i) {
            while (!rb.push(i))
                std::this_thread::yield();  // Simulate ISR retry
        }
        done.store(true);
    });

    // Main loop (consumer)
    uint32_t expected = 0;
    while (expected < COUNT) {
        uint32_t val;
        if (rb.pop(val)) {
            ASSERT_EQ(val, expected) << "Out-of-order at " << expected;
            expected++;
        }
    }

    producer.join();
    EXPECT_TRUE(rb.empty());
}

```

### Q3: Verify ISR timing constraints on host

**Answer:**

```cpp

#include <gtest/gtest.h>
#include <benchmark/benchmark.h>
#include <chrono>

// === Measure worst-case execution time of ISR logic ===
// Note: Host timing != target timing, but relative comparisons are valid

class UartRxHandler {
public:
    // This is the hot path called from ISR
    void on_byte_received(uint8_t byte) {
        if (byte == FRAME_START) {
            state_ = State::Header;
            frame_pos_ = 0;
            return;
        }

        switch (state_) {
            case State::Idle:
                break;
            case State::Header:
                if (frame_pos_ < 2) {
                    header_[frame_pos_++] = byte;
                    if (frame_pos_ == 2) {
                        payload_len_ = header_[1];
                        state_ = State::Payload;
                        frame_pos_ = 0;
                    }
                }
                break;
            case State::Payload:
                if (frame_pos_ < payload_len_ && frame_pos_ < MAX_PAYLOAD) {
                    payload_[frame_pos_++] = byte;
                    if (frame_pos_ == payload_len_) {
                        state_ = State::Checksum;
                    }
                }
                break;
            case State::Checksum:
                frame_complete_ = verify_checksum(byte);
                state_ = State::Idle;
                break;
        }
    }

    bool is_frame_complete() const { return frame_complete_; }
    void clear_frame() { frame_complete_ = false; }

private:
    static constexpr uint8_t FRAME_START = 0x7E;
    static constexpr size_t MAX_PAYLOAD = 64;

    enum class State { Idle, Header, Payload, Checksum };
    State state_ = State::Idle;
    uint8_t header_[2]{};
    uint8_t payload_[MAX_PAYLOAD]{};
    uint8_t payload_len_ = 0;
    size_t frame_pos_ = 0;
    bool frame_complete_ = false;

    bool verify_checksum(uint8_t checksum) {
        uint8_t calc = header_[0] ^ header_[1];
        for (size_t i = 0; i < payload_len_; ++i)
            calc ^= payload_[i];
        return calc == checksum;
    }
};

// === Functional test ===
TEST(UartRxHandlerTest, ParsesCompleteFrame) {
    UartRxHandler handler;

    // Frame: START, type=0x01, len=3, payload={0x10, 0x20, 0x30}, checksum
    handler.on_byte_received(0x7E);  // Start
    handler.on_byte_received(0x01);  // Type
    handler.on_byte_received(0x03);  // Length
    handler.on_byte_received(0x10);  // Payload
    handler.on_byte_received(0x20);
    handler.on_byte_received(0x30);
    handler.on_byte_received(0x01 ^ 0x03 ^ 0x10 ^ 0x20 ^ 0x30);  // Checksum

    EXPECT_TRUE(handler.is_frame_complete());
}

TEST(UartRxHandlerTest, RejectsBadChecksum) {
    UartRxHandler handler;
    handler.on_byte_received(0x7E);
    handler.on_byte_received(0x01);
    handler.on_byte_received(0x01);  // 1 byte payload
    handler.on_byte_received(0xAA);
    handler.on_byte_received(0xFF);  // Wrong checksum

    EXPECT_FALSE(handler.is_frame_complete());
}

// === Timing benchmark (use Google Benchmark) ===
static void BM_UartRxPerByte(benchmark::State& state) {
    UartRxHandler handler;
    uint8_t byte = 0x42;

    for (auto _ : state) {
        handler.on_byte_received(byte);
        benchmark::DoNotOptimize(handler);
    }
}
BENCHMARK(BM_UartRxPerByte);

```

---

## Notes

- **Extract, don't mock ISRs** — move logic into plain C++ classes, call them from thin ISR wrappers
- ISR logic must be O(1) or bounded — no dynamic allocation, no unbounded loops
- Ring buffers are the standard ISR-to-main communication primitive — test them thoroughly
- Use ThreadSanitizer on producer-consumer tests to verify lock-free correctness
- Host benchmarks show relative timing; use real hardware profiling for absolute µs measurements
- Verify that ISR handlers never call blocking functions (static analysis can check this)
- Power-of-2 buffer sizes enable fast modulo via bitwise AND
