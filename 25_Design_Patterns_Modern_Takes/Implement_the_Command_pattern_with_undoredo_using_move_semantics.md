# Implement the Command pattern with undo/redo using move semantics

**Category:** Design Patterns — Modern Takes  
**Item:** #571  
**Standard:** C++11  
**Reference:** <https://en.wikipedia.org/wiki/Command_pattern>  

---

## Topic Overview

The **Command pattern** encapsulates an action as an object with `execute()` and `undo()` methods. Combined with **move semantics**, commands are transferred between undo/redo stacks without copying, enabling efficient history management for editors, drawing tools, and transaction systems.

### Architecture

```cpp

User action
    │
    ▼
 Command (execute + undo)
    │ std::move
    ▼
 undo_stack: vector<unique_ptr<Command>>
    │ undo() + std::move
    ▼
 redo_stack: vector<unique_ptr<Command>>
    │ redo() + std::move
    ▼
 back to undo_stack

```

---

## Self-Assessment

### Q1: Define a Command interface with execute() and undo(), stored as unique_ptr<Command> in a history stack

**Answer:**

```cpp

#include <iostream>
#include <string>
#include <memory>
#include <vector>

// ═══════════ Command interface ═══════════
class Command {
public:
    virtual ~Command() = default;
    virtual void execute() = 0;
    virtual void undo() = 0;
    virtual std::string description() const = 0;
};

// ═══════════ Document model ═══════════
class Document {
    std::string text_;
public:
    void insert(size_t pos, const std::string& s) {
        text_.insert(pos, s);
    }
    void erase(size_t pos, size_t len) {
        text_.erase(pos, len);
    }
    const std::string& text() const { return text_; }
};

// ═══════════ InsertCommand ═══════════
class InsertCommand : public Command {
    Document& doc_;
    size_t pos_;
    std::string text_;
public:
    InsertCommand(Document& doc, size_t pos, std::string text)
        : doc_(doc), pos_(pos), text_(std::move(text)) {}

    void execute() override { doc_.insert(pos_, text_); }
    void undo() override    { doc_.erase(pos_, text_.size()); }
    std::string description() const override {
        return "Insert '" + text_ + "' at " + std::to_string(pos_);
    }
};

// ═══════════ History stack ═══════════
class History {
    std::vector<std::unique_ptr<Command>> undo_stack_;
public:
    void execute(std::unique_ptr<Command> cmd) {
        cmd->execute();
        undo_stack_.push_back(std::move(cmd));  // move into stack
    }
    void undo() {
        if (undo_stack_.empty()) return;
        undo_stack_.back()->undo();
        undo_stack_.pop_back();
    }
};

int main() {
    Document doc;
    History history;

    history.execute(std::make_unique<InsertCommand>(doc, 0, "Hello"));
    history.execute(std::make_unique<InsertCommand>(doc, 5, " World"));
    std::cout << doc.text() << '\n';  // Hello World

    history.undo();
    std::cout << doc.text() << '\n';  // Hello

    history.undo();
    std::cout << doc.text() << '\n';  // (empty)
}

```

### Q2: Use std::move to efficiently push commands onto the undo stack without copying

**Answer:**

```cpp

#include <iostream>
#include <memory>
#include <vector>
#include <string>

class Command {
public:
    virtual ~Command() = default;
    virtual void execute() = 0;
    virtual void undo() = 0;
};

// ═══════════ Command with large captured state ═══════════
class DrawCommand : public Command {
    std::vector<float> pixels_;  // Could be megabytes of data
    int layer_;
public:
    DrawCommand(std::vector<float> pixels, int layer)
        : pixels_(std::move(pixels))   // Move-construct: O(1), no copy
        , layer_(layer) {}

    void execute() override {
        std::cout << "Draw " << pixels_.size() << " pixels on layer "
                  << layer_ << '\n';
    }
    void undo() override {
        std::cout << "Erase " << pixels_.size() << " pixels from layer "
                  << layer_ << '\n';
    }
};

int main() {
    std::vector<std::unique_ptr<Command>> undo_stack;

    // Create command with large pixel data
    std::vector<float> big_data(1'000'000, 1.0f);
    auto cmd = std::make_unique<DrawCommand>(std::move(big_data), 0);
    // big_data is now empty — ownership transferred to DrawCommand

    cmd->execute();
    undo_stack.push_back(std::move(cmd));
    // cmd is now nullptr — ownership transferred to undo_stack

    std::cout << "Stack size: " << undo_stack.size() << '\n';

    // No copies at any point — only moves
    // Without move semantics: 2 copies of 1M floats = 8MB wasted
    // With move semantics: 0 copies, just pointer transfers
}

```

### Q3: Implement redo by maintaining a separate redo stack and moving commands between stacks

**Answer:**

```cpp

#include <iostream>
#include <memory>
#include <vector>
#include <string>

class Command {
public:
    virtual ~Command() = default;
    virtual void execute() = 0;
    virtual void undo() = 0;
    virtual std::string name() const = 0;
};

// ═══════════ Full Undo/Redo Manager ═══════════
class UndoRedoManager {
    std::vector<std::unique_ptr<Command>> undo_stack_;
    std::vector<std::unique_ptr<Command>> redo_stack_;
public:
    void execute(std::unique_ptr<Command> cmd) {
        cmd->execute();
        undo_stack_.push_back(std::move(cmd));
        redo_stack_.clear();  // New action invalidates redo history
    }

    void undo() {
        if (undo_stack_.empty()) { std::cout << "Nothing to undo\n"; return; }
        auto cmd = std::move(undo_stack_.back());  // Move out of undo
        undo_stack_.pop_back();
        cmd->undo();
        redo_stack_.push_back(std::move(cmd));      // Move into redo
    }

    void redo() {
        if (redo_stack_.empty()) { std::cout << "Nothing to redo\n"; return; }
        auto cmd = std::move(redo_stack_.back());   // Move out of redo
        redo_stack_.pop_back();
        cmd->execute();
        undo_stack_.push_back(std::move(cmd));      // Move back into undo
    }
};

// Simple counter command for demonstration
class AddCommand : public Command {
    int& value_;
    int amount_;
public:
    AddCommand(int& value, int amount) : value_(value), amount_(amount) {}
    void execute() override { value_ += amount_; }
    void undo() override    { value_ -= amount_; }
    std::string name() const override { return "Add " + std::to_string(amount_); }
};

int main() {
    int counter = 0;
    UndoRedoManager mgr;

    mgr.execute(std::make_unique<AddCommand>(counter, 10));
    mgr.execute(std::make_unique<AddCommand>(counter, 20));
    std::cout << counter << '\n';    // 30

    mgr.undo();                       // Undo +20
    std::cout << counter << '\n';    // 10

    mgr.undo();                       // Undo +10
    std::cout << counter << '\n';    // 0

    mgr.redo();                       // Redo +10
    std::cout << counter << '\n';    // 10

    mgr.redo();                       // Redo +20
    std::cout << counter << '\n';    // 30

    mgr.undo();                       // Undo +20
    mgr.execute(std::make_unique<AddCommand>(counter, 5));  // New action
    std::cout << counter << '\n';    // 15
    mgr.redo();                       // Nothing — redo cleared by new action
}

```

---

## Notes

- `unique_ptr` enforces single ownership — commands cannot be accidentally shared between stacks
- `redo_stack_.clear()` on new execute is essential — otherwise redo history is inconsistent
- Move semantics make this pattern practical even with commands capturing large state (images, buffers)
- Consider `std::deque` instead of `vector` if you need to limit history depth with efficient front removal
