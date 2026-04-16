# Implement the Command pattern with undo/redo using move semantics

**Category:** Design Patterns — Modern Takes  
**Item:** #670  
**Standard:** C++11  
**Reference:** <https://en.wikipedia.org/wiki/Command_pattern>  

---

## Topic Overview

This second Command pattern file focuses on using `std::deque` for bounded history, `unique_ptr` polymorphism for type-erased commands, and the mechanics of transferring ownership between undo and redo stacks.

### Deque vs Vector for Command History

| Feature | `std::vector` | `std::deque` |
| --- | --- | --- |
| Push back | O(1) amortized | O(1) |
| Pop back | O(1) | O(1) |
| Pop front (limit history) | O(n) — expensive! | O(1) — ideal |
| Contiguous memory | Yes | No (chunked) |
| **Best for** | Unbounded history | Bounded history with max depth |

---

## Self-Assessment

### Q1: Define a Command interface with execute() and undo() and store commands in a std::deque

**Answer:**

```cpp

#include <iostream>
#include <string>
#include <memory>
#include <deque>

// ═══════════ Command interface ═══════════
class Command {
public:
    virtual ~Command() = default;
    virtual void execute() = 0;
    virtual void undo() = 0;
};

// ═══════════ Spreadsheet model ═══════════
class Spreadsheet {
    std::string cells_[3][3];  // 3x3 grid
public:
    void set(int r, int c, const std::string& val) { cells_[r][c] = val; }
    std::string get(int r, int c) const { return cells_[r][c]; }
    void print() const {
        for (int r = 0; r < 3; ++r) {
            for (int c = 0; c < 3; ++c)
                std::cout << "[" << (cells_[r][c].empty() ? " " : cells_[r][c]) << "] ";
            std::cout << '\n';
        }
    }
};

// ═══════════ SetCellCommand ═══════════
class SetCellCommand : public Command {
    Spreadsheet& sheet_;
    int row_, col_;
    std::string new_value_;
    std::string old_value_;  // Saved for undo
public:
    SetCellCommand(Spreadsheet& sheet, int r, int c, std::string val)
        : sheet_(sheet), row_(r), col_(c), new_value_(std::move(val)) {}

    void execute() override {
        old_value_ = sheet_.get(row_, col_);  // Save before overwriting
        sheet_.set(row_, col_, new_value_);
    }
    void undo() override {
        sheet_.set(row_, col_, old_value_);
    }
};

// ═══════════ Bounded history with deque ═══════════
class CommandHistory {
    std::deque<std::unique_ptr<Command>> history_;
    static constexpr size_t MAX_HISTORY = 50;
public:
    void execute(std::unique_ptr<Command> cmd) {
        cmd->execute();
        history_.push_back(std::move(cmd));
        if (history_.size() > MAX_HISTORY) {
            history_.pop_front();  // O(1) with deque!
        }
    }
    void undo() {
        if (history_.empty()) return;
        history_.back()->undo();
        history_.pop_back();
    }
    size_t size() const { return history_.size(); }
};

int main() {
    Spreadsheet sheet;
    CommandHistory history;

    history.execute(std::make_unique<SetCellCommand>(sheet, 0, 0, "A1"));
    history.execute(std::make_unique<SetCellCommand>(sheet, 1, 1, "B2"));
    history.execute(std::make_unique<SetCellCommand>(sheet, 2, 2, "C3"));
    sheet.print();
    // [A1] [ ] [ ]
    // [ ] [B2] [ ]
    // [ ] [ ] [C3]

    history.undo();
    history.undo();
    sheet.print();
    // [A1] [ ] [ ]
    // [ ] [ ] [ ]
    // [ ] [ ] [ ]
}

```

### Q2: Use unique_ptr<Command> to store commands by type and std::move to transfer ownership to the history stack

**Answer:**

```cpp

#include <iostream>
#include <memory>
#include <deque>
#include <string>

class Command {
public:
    virtual ~Command() = default;
    virtual void execute() = 0;
    virtual void undo() = 0;
};

// ═══════════ Multiple command types stored polymorphically ═══════════
class PrintCommand : public Command {
    std::string message_;
public:
    explicit PrintCommand(std::string msg) : message_(std::move(msg)) {}
    void execute() override { std::cout << ">> " << message_ << '\n'; }
    void undo() override    { std::cout << "<< undo: " << message_ << '\n'; }
};

class CompositeCommand : public Command {
    std::deque<std::unique_ptr<Command>> children_;
public:
    void add(std::unique_ptr<Command> cmd) {
        children_.push_back(std::move(cmd));  // Transfer ownership
    }
    void execute() override {
        for (auto& cmd : children_) cmd->execute();
    }
    void undo() override {
        // Undo in reverse order!
        for (auto it = children_.rbegin(); it != children_.rend(); ++it)
            (*it)->undo();
    }
};

int main() {
    // Build a composite command
    auto macro = std::make_unique<CompositeCommand>();
    macro->add(std::make_unique<PrintCommand>("Step 1"));
    macro->add(std::make_unique<PrintCommand>("Step 2"));
    macro->add(std::make_unique<PrintCommand>("Step 3"));

    // Execute the macro
    macro->execute();
    // >> Step 1
    // >> Step 2
    // >> Step 3

    // Undo the macro (reverse order)
    macro->undo();
    // << undo: Step 3
    // << undo: Step 2
    // << undo: Step 1

    // The composite is itself a Command — can be stored in history
    std::deque<std::unique_ptr<Command>> history;
    history.push_back(std::move(macro));  // macro is now nullptr
}

```

### Q3: Implement redo by re-executing commands from a redo stack, transferring ownership back

**Answer:**

```cpp

#include <iostream>
#include <memory>
#include <deque>
#include <string>

class Command {
public:
    virtual ~Command() = default;
    virtual void execute() = 0;
    virtual void undo() = 0;
};

class AppendCommand : public Command {
    std::string& target_;
    std::string text_;
public:
    AppendCommand(std::string& target, std::string text)
        : target_(target), text_(std::move(text)) {}
    void execute() override { target_ += text_; }
    void undo() override    { target_.erase(target_.size() - text_.size()); }
};

// ═══════════ Full undo/redo with deque ═══════════
class Editor {
    std::string buffer_;
    std::deque<std::unique_ptr<Command>> undo_stack_;
    std::deque<std::unique_ptr<Command>> redo_stack_;
public:
    void type(const std::string& text) {
        auto cmd = std::make_unique<AppendCommand>(buffer_, text);
        cmd->execute();
        undo_stack_.push_back(std::move(cmd));
        redo_stack_.clear();  // Invalidate redo on new action
    }

    void undo() {
        if (undo_stack_.empty()) return;
        auto cmd = std::move(undo_stack_.back());
        undo_stack_.pop_back();
        cmd->undo();
        redo_stack_.push_back(std::move(cmd));  // Move to redo
    }

    void redo() {
        if (redo_stack_.empty()) return;
        auto cmd = std::move(redo_stack_.back());
        redo_stack_.pop_back();
        cmd->execute();                          // Re-execute
        undo_stack_.push_back(std::move(cmd));   // Move back to undo
    }

    void show() const {
        std::cout << "Buffer: \"" << buffer_ << "\"\n";
    }
};

int main() {
    Editor ed;
    ed.type("Hello");    ed.show();  // "Hello"
    ed.type(" World");   ed.show();  // "Hello World"
    ed.undo();           ed.show();  // "Hello"
    ed.undo();           ed.show();  // ""
    ed.redo();           ed.show();  // "Hello"
    ed.redo();           ed.show();  // "Hello World"
    ed.undo();
    ed.type("!");        ed.show();  // "Hello!"  (redo cleared)
    ed.redo();           ed.show();  // "Hello!"  (nothing to redo)
}

```

---

## Notes

- `std::deque` is preferred for bounded history because `pop_front()` is O(1) vs vector's O(n)
- Composite commands (macros) are themselves `Command` — enables recording multi-step actions as a single undo unit
- Always undo composite commands in **reverse order** to maintain consistency
- Clearing redo on new execute prevents branching history (though some editors support undo trees)
