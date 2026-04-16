# Implement copy-on-write (COW) semantics for expensive-to-copy types

**Category:** OOP Design

---

## Topic Overview

**Copy-on-Write (COW)** delays expensive deep copies until a write actually occurs. Copies share the same data; only when one copy is modified does it detach and create its own copy.

```cpp

COW lifecycle:
  A = original        →  A.data refcount=1
  B = A  (cheap copy)  →  A.data refcount=2 (shared)
  B.modify()           →  B detaches → B.data refcount=1 (new copy)
                          A.data refcount=1 (unchanged)

```

| Aspect | Eager Copy | COW |
| --- | :---: | :---: |
| Copy cost | O(n) always | O(1) until write |
| Write cost | O(1) | O(n) on first write (detach) |
| Memory usage | Higher (always copy) | Lower (shared) |
| Thread safety | Inherent | **Needs atomic refcount** |
| Cache locality | Better | May be worse (shared heap) |

---

## Self-Assessment

### Q1: Implement a basic COW string class

**Answer:**

```cpp

#include <memory>
#include <cstring>
#include <iostream>
#include <utility>

class CowString {
public:
    CowString() : data_(std::make_shared<std::string>()) {}
    explicit CowString(const char* s) : data_(std::make_shared<std::string>(s)) {}

    // Copy is cheap: just share the pointer
    CowString(const CowString&) = default;
    CowString& operator=(const CowString&) = default;

    // Read access: no copy needed
    char operator[](size_t i) const {
        return (*data_)[i];
    }

    // Write access: detach if shared
    char& at(size_t i) {
        detach();
        return (*data_)[i];
    }

    void append(const char* s) {
        detach();
        data_->append(s);
    }

    const char* c_str() const { return data_->c_str(); }
    size_t size() const { return data_->size(); }
    long use_count() const { return data_.use_count(); }

private:
    void detach() {
        if (data_.use_count() > 1) {
            data_ = std::make_shared<std::string>(*data_);  // Deep copy
        }
    }

    std::shared_ptr<std::string> data_;
};

int main() {
    CowString a("Hello, World!");
    CowString b = a;  // Cheap! Shared data
    std::cout << "After copy: a.refs=" << a.use_count()
              << " b.refs=" << b.use_count() << "\n";
    // After copy: a.refs=2 b.refs=2

    b.append(" Modified");  // Detach! b gets its own copy
    std::cout << "After modify: a.refs=" << a.use_count()
              << " b.refs=" << b.use_count() << "\n";
    // After modify: a.refs=1 b.refs=1

    std::cout << "a: " << a.c_str() << "\n";  // "Hello, World!"
    std::cout << "b: " << b.c_str() << "\n";  // "Hello, World! Modified"
    return 0;
}

```

### Q2: Build a thread-safe COW buffer with atomic reference counting

**Answer:**

```cpp

#include <atomic>
#include <cstring>
#include <algorithm>
#include <iostream>

class CowBuffer {
    struct Data {
        std::atomic<int> refcount{1};
        size_t size;
        size_t capacity;
        char payload[];  // Flexible array member (C-style, common in practice)

        static Data* create(size_t cap) {
            auto* d = static_cast<Data*>(::operator new(
                sizeof(Data) + cap));
            d->refcount.store(1, std::memory_order_relaxed);
            d->size = 0;
            d->capacity = cap;
            return d;
        }

        Data* clone() const {
            auto* d = create(capacity);
            d->size = size;
            std::memcpy(d->payload, payload, size);
            return d;
        }

        void release() {
            if (refcount.fetch_sub(1, std::memory_order_acq_rel) == 1)
                ::operator delete(this);
        }

        void acquire() {
            refcount.fetch_add(1, std::memory_order_relaxed);
        }
    };

    Data* data_;

    void detach() {
        if (data_->refcount.load(std::memory_order_acquire) > 1) {
            Data* new_data = data_->clone();
            data_->release();
            data_ = new_data;
        }
    }

public:
    explicit CowBuffer(size_t cap = 64)
        : data_(Data::create(cap)) {}

    ~CowBuffer() { data_->release(); }

    CowBuffer(const CowBuffer& o) : data_(o.data_) {
        data_->acquire();  // Cheap copy!
    }

    CowBuffer& operator=(const CowBuffer& o) {
        if (this != &o) {
            o.data_->acquire();
            data_->release();
            data_ = o.data_;
        }
        return *this;
    }

    // Read: no detach
    const char* data() const { return data_->payload; }
    size_t size() const { return data_->size; }

    // Write: detach first
    void write(const char* src, size_t n) {
        detach();
        std::memcpy(data_->payload + data_->size, src, n);
        data_->size += n;
    }
};

```

### Q3: Show COW for an immutable-externally, mutable-internally data structure

**Answer:**

```cpp

#include <memory>
#include <vector>
#include <iostream>

// COW vector: snapshots are cheap, mutations detach
template<typename T>
class CowVector {
    std::shared_ptr<std::vector<T>> data_;

    void detach() {
        if (data_.use_count() > 1)
            data_ = std::make_shared<std::vector<T>>(*data_);
    }

public:
    CowVector() : data_(std::make_shared<std::vector<T>>()) {}

    // Read-only access — always safe, no detach
    const T& operator[](size_t i) const { return (*data_)[i]; }
    size_t size() const { return data_->size(); }
    auto begin() const { return data_->cbegin(); }
    auto end() const { return data_->cend(); }

    // Write access — detach on modify
    void push_back(const T& val) { detach(); data_->push_back(val); }
    void set(size_t i, const T& val) { detach(); (*data_)[i] = val; }
    void clear() { detach(); data_->clear(); }

    // Snapshot: O(1) copy for read-only sharing
    CowVector snapshot() const { return *this; }

    long ref_count() const { return data_.use_count(); }
};

int main() {
    CowVector<int> v;
    for (int i = 0; i < 1000; ++i) v.push_back(i);

    // Cheap snapshots for readers
    auto snap1 = v.snapshot();  // O(1)
    auto snap2 = v.snapshot();  // O(1)
    std::cout << "Refs: " << v.ref_count() << "\n";  // 3

    v.push_back(1000);  // Detaches v from snapshots
    std::cout << "After write: v.refs=" << v.ref_count()
              << " snap1.refs=" << snap1.ref_count() << "\n";
    // v.refs=1, snap1.refs=2 (snap1, snap2 still share)
    return 0;
}

```

---

## Notes

- **COW was removed from `std::string` in C++11** — SSO (Small String Optimization) replaced it
- `shared_ptr` provides atomic refcounting out of the box — simplest COW implementation
- COW is excellent for: configuration snapshots, undo history, functional-style data structures
- **Thread safety caveat:** `shared_ptr` refcount is atomic, but the string data is not — detach must be exclusive
- The `detach()` check must use `use_count() > 1`, not `unique()` (deprecated in C++17, removed C++20)
- Qt's `QString`, `QByteArray`, `QVector` all use implicit sharing (COW) via `QSharedDataPointer`
