# Implement the Memento pattern for state snapshots with value semantics

**Category:** Design Patterns — Modern Takes  
**Item:** #754  
**Standard:** C++11  
**Reference:** <https://en.wikipedia.org/wiki/Memento_pattern>  

---

## Topic Overview

This second Memento file focuses on **value semantics**, **copy-on-write optimization**, and **move semantics** for efficient snapshots. Unlike the first Memento file (which covered basic snapshot/restore with friend classes), this one addresses performance and memory considerations for large state objects.

### Snapshot Strategies

| Strategy | Cost to Save | Cost to Restore | Memory | Best For |
| --- | --- | --- | --- | --- |
| Deep copy | O(n) | O(n) | Full copy per snapshot | Small state |
| Copy-on-write (CoW) | O(1) share | O(n) on first modify | Shared until modified | Many snapshots, rare modifications |
| Move capture | O(1) | O(1) | Original moved out | One-shot save (no further use of original) |

---

## Self-Assessment

### Q1: Use deep copy to implement a Memento that captures the complete state of an object

**Answer:**

```cpp

#include <iostream>
#include <string>
#include <vector>
#include <memory>

// ═══════════ Canvas with large pixel data ═══════════
class Canvas {
    std::vector<uint8_t> pixels_;
    int width_, height_;
    std::string label_;

public:
    Canvas(int w, int h, const std::string& label)
        : pixels_(w * h * 4, 0), width_(w), height_(h), label_(label) {}

    void draw_pixel(int x, int y, uint8_t r, uint8_t g, uint8_t b) {
        int i = (y * width_ + x) * 4;
        pixels_[i] = r; pixels_[i+1] = g; pixels_[i+2] = b; pixels_[i+3] = 255;
    }

    // ═══════════ Memento = deep copy of entire state ═══════════
    struct Memento {
        std::vector<uint8_t> pixels;
        int width, height;
        std::string label;
    };

    Memento save() const {
        return Memento{pixels_, width_, height_, label_};  // Deep copy
    }

    void restore(const Memento& m) {
        pixels_ = m.pixels;  // Deep copy back
        width_ = m.width;
        height_ = m.height;
        label_ = m.label;
    }

    size_t pixel_count() const { return pixels_.size(); }
    std::string info() const {
        return label_ + " (" + std::to_string(width_) + "x" + std::to_string(height_) + ")";
    }
};

int main() {
    Canvas canvas(100, 100, "Drawing1");
    canvas.draw_pixel(50, 50, 255, 0, 0);

    // Save complete state (deep copy: 40KB of pixel data copied)
    auto snapshot = canvas.save();
    std::cout << "Saved: " << snapshot.pixels.size() << " bytes\n";

    canvas.draw_pixel(0, 0, 0, 255, 0);  // Modify after save

    // Restore to saved state
    canvas.restore(snapshot);
    std::cout << "Restored: " << canvas.info() << '\n';
}

```

### Q2: Optimize by using copy-on-write (shared_ptr + lazy copy) for large state objects

**Answer:**

```cpp

#include <iostream>
#include <string>
#include <vector>
#include <memory>

// ═══════════ Copy-on-Write (CoW) buffer ═══════════
class CowBuffer {
    struct Data {
        std::vector<int> values;
    };
    std::shared_ptr<Data> data_;

    // Detach: make a private copy if shared
    void detach() {
        if (data_.use_count() > 1) {
            data_ = std::make_shared<Data>(*data_);  // Copy only when modifying
        }
    }

public:
    CowBuffer() : data_(std::make_shared<Data>()) {}

    // Read: no copy needed — return const reference
    const std::vector<int>& values() const { return data_->values; }

    // Write: detach first (copy if shared), then modify
    void push(int val) {
        detach();
        data_->values.push_back(val);
    }

    void set(size_t i, int val) {
        detach();
        data_->values[i] = val;
    }

    // ═══════════ Memento = share, not copy ═══════════
    struct Memento {
        std::shared_ptr<Data> snapshot;  // Shared ownership — O(1)!
    };

    Memento save() const {
        return Memento{data_};  // Just increment ref count — O(1)
    }

    void restore(const Memento& m) {
        data_ = m.snapshot;  // Point to shared data — O(1)
    }

    size_t size() const { return data_->values.size(); }
    long use_count() const { return data_.use_count(); }
};

int main() {
    CowBuffer buf;
    for (int i = 0; i < 1'000'000; ++i) buf.push(i);

    // Save: O(1) — just shares the pointer
    auto snap1 = buf.save();
    std::cout << "After save, ref count: " << buf.use_count() << '\n';  // 2

    // Modify: triggers copy (detach)
    buf.set(0, 999);
    std::cout << "After modify, ref count: " << buf.use_count() << '\n';  // 1

    // Restore: O(1) — just share pointer again
    buf.restore(snap1);
    std::cout << "Restored buf[0] = " << buf.values()[0] << '\n';  // 0

    // Multiple snapshots share same data until any is modified
    auto snap2 = buf.save();
    auto snap3 = buf.save();
    std::cout << "3 snapshots, ref count: " << buf.use_count() << '\n';  // 4
}

```

### Q3: Show that move semantics enables efficient Memento capture without unnecessary copying

**Answer:**

```cpp

#include <iostream>
#include <string>
#include <vector>

// ═══════════ Move-based Memento: zero-copy snapshot ═══════════
class GameState {
    std::vector<float> world_data_;
    std::string level_name_;

public:
    GameState(std::string level, size_t data_size)
        : world_data_(data_size, 1.0f), level_name_(std::move(level)) {}

    // Move constructor/assignment for efficient transfer
    GameState(GameState&&) = default;
    GameState& operator=(GameState&&) = default;
    GameState(const GameState&) = default;
    GameState& operator=(const GameState&) = default;

    bool empty() const { return world_data_.empty(); }
    size_t size() const { return world_data_.size(); }
    const std::string& level() const { return level_name_; }

    void modify(float val) { world_data_[0] = val; }
};

// ═══════════ Checkpoint system using move ═══════════
class Checkpoint {
    GameState saved_state_;
public:
    // Move the state INTO the checkpoint (originator loses it)
    explicit Checkpoint(GameState&& state)
        : saved_state_(std::move(state)) {}

    // Move the state OUT (one-shot restore)
    GameState restore() && {
        return std::move(saved_state_);
    }
};

int main() {
    GameState game("Level 5", 1'000'000);
    std::cout << "Before save: " << game.size() << " floats\n";

    // Save via move — O(1), no copy of 4MB data
    Checkpoint cp(std::move(game));
    std::cout << "After move-save: game empty? " << game.empty() << '\n';  // true

    // Restore via move — O(1)
    game = std::move(cp).restore();
    std::cout << "After restore: " << game.size() << " floats, level: "
              << game.level() << '\n';

    // For undo history: use deep copy for save, move for temporary captures
    std::vector<GameState> history;
    history.push_back(game);           // Deep copy for persistent snapshot
    game.modify(42.0f);               // Modify original
    history.push_back(game);           // Another deep copy
    
    // Restore from history
    game = std::move(history.back());  // Move from history (destroys that snapshot)
    history.pop_back();
    std::cout << "Restored from history: " << game.size() << " floats\n";
}

```

---

## Notes

- **Deep copy** is simplest and safest — use when state is small (< few KB)
- **Copy-on-write** shines when taking many snapshots but rarely restoring — common in version control
- **Move-based save** is O(1) but destructive — the originator loses its state until restore
- Combine strategies: use CoW for persistent snapshots (undo history) and move for temporary checkpoints (quick save/load)
