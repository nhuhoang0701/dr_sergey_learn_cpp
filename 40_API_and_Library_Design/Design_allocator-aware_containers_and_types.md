# Design allocator-aware containers and types

**Category:** API & Library Design  
**Standard:** C++17/20  
**Reference:** <https://en.cppreference.com/w/cpp/named_req/AllocatorAwareContainer>  

---

## Topic Overview

Allocator-aware types let users control memory allocation strategy without changing the container code. This is essential for arena allocation, PMR, and embedded systems.

### Making a Type Allocator-Aware

```cpp

#include <memory>
#include <memory_resource>
#include <vector>
#include <string>
#include <iostream>

template<typename T, typename Allocator = std::allocator<T>>
class SimpleBuffer {
public:
    using allocator_type = Allocator;
    using value_type = T;

    SimpleBuffer() = default;

    explicit SimpleBuffer(const Allocator& alloc)
        : alloc_(alloc) {}

    SimpleBuffer(size_t count, const T& value, const Allocator& alloc = Allocator())
        : alloc_(alloc) {
        data_ = std::allocator_traits<Allocator>::allocate(alloc_, count);
        size_ = count;
        for (size_t i = 0; i < count; ++i)
            std::allocator_traits<Allocator>::construct(alloc_, data_ + i, value);
    }

    ~SimpleBuffer() {
        for (size_t i = 0; i < size_; ++i)
            std::allocator_traits<Allocator>::destroy(alloc_, data_ + i);
        if (data_)
            std::allocator_traits<Allocator>::deallocate(alloc_, data_, size_);
    }

    allocator_type get_allocator() const { return alloc_; }
    size_t size() const { return size_; }
    T& operator[](size_t i) { return data_[i]; }

private:
    T* data_ = nullptr;
    size_t size_ = 0;
    [[no_unique_address]] Allocator alloc_{};
};

int main() {
    // With default allocator
    SimpleBuffer<int> buf1(5, 42);
    std::cout << "Default: " << buf1[0] << "\n";

    // With PMR allocator
    std::array<std::byte, 1024> storage;
    std::pmr::monotonic_buffer_resource resource(storage.data(), storage.size());
    SimpleBuffer<int, std::pmr::polymorphic_allocator<int>> buf2(
        5, 99, std::pmr::polymorphic_allocator<int>(&resource));
    std::cout << "PMR: " << buf2[0] << "\n";
}

```

---

## Self-Assessment

### Q1: Why use `std::allocator_traits` instead of calling allocator methods directly

`allocator_traits` provides default implementations for optional allocator methods. A minimal allocator only needs `allocate()` and `deallocate()` — `traits` fills in `construct`, `destroy`, `max_size`, etc. This makes custom allocators much simpler to write.

### Q2: What is `[[no_unique_address]]` and why use it for allocators

Stateless allocators (like `std::allocator`) have `sizeof == 1`. Without `[[no_unique_address]]`, they waste padding bytes in every container. With the attribute, the compiler can overlap the allocator with other members, making `sizeof(Container)` smaller — often saving 8 bytes.

### Q3: Show how PMR containers use a runtime-polymorphic allocator

```cpp

#include <memory_resource>
#include <vector>
#include <string>

void pmr_example() {
    std::byte buf[4096];
    std::pmr::monotonic_buffer_resource arena(buf, sizeof(buf));

    // All these use the arena — same allocator TYPE, different resource
    std::pmr::vector<int> nums(&arena);
    std::pmr::string name("hello", &arena);

    nums.push_back(42);  // Allocates from buf[], not heap
}

```

---

## Notes

- Always use `std::allocator_traits` — never call allocator methods directly.
- `[[no_unique_address]]` (C++20) enables empty base optimization for allocators.
- PMR (`std::pmr`) provides runtime allocator polymorphism without template proliferation.
- Allocator-aware types should propagate the allocator to sub-containers.
