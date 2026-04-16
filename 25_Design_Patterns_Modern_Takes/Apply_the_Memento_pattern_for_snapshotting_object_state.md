# Apply the Memento pattern for snapshotting object state

**Category:** Design Patterns — Modern Takes  
**Item:** #677  
**Reference:** <https://en.wikipedia.org/wiki/Memento_pattern>  

---

## Topic Overview

The **Memento pattern** captures an object's internal state as an opaque snapshot so the object can be restored to that state later — without exposing its internals. Modern C++ makes this elegant with move semantics, `std::unique_ptr`, and value types.

### Pattern Structure

```cpp

┌──────────────┐     creates      ┌────────────┐
│  Originator  │────────────────►│  Memento   │
│ (TextEditor) │                  │ (snapshot) │
│              │◄────────────────│            │
│              │    restores      └────────────┘
└──────────────┘                       │
                                       │ stored in
                                       ▼
                                ┌──────────────┐
                                │  Caretaker   │
                                │ (History)    │
                                │ vector<Mem>  │
                                └──────────────┘

```

**Key principle:** The Memento is opaque — the Caretaker stores it but cannot read or modify the snapshot contents.

---

## Self-Assessment

### Q1: Implement a TextEditor with save_state() returning a Memento and restore_state(Memento)

**Answer:**

```cpp

#include <iostream>
#include <string>
#include <vector>
#include <chrono>

class TextEditor {
public:
    // ═══════════ Opaque Memento (nested class) ═══════════
    class Memento {
        friend class TextEditor;  // Only TextEditor can read internals
        std::string text_;
        int cursor_pos_;
        std::chrono::system_clock::time_point timestamp_;

        // Private constructor — only TextEditor can create
        Memento(std::string text, int cursor)
            : text_(std::move(text)), cursor_pos_(cursor),
              timestamp_(std::chrono::system_clock::now()) {}
    public:
        // Public: caretaker can check when snapshot was taken
        auto timestamp() const { return timestamp_; }
    };

    // Editor operations
    void type(const std::string& s) {
        text_.insert(cursor_, s);
        cursor_ += static_cast<int>(s.size());
    }

    void move_cursor(int pos) { cursor_ = std::clamp(pos, 0, static_cast<int>(text_.size())); }
    void backspace() { if (cursor_ > 0) { text_.erase(--cursor_, 1); } }

    // ═══════════ Memento operations ═══════════
    Memento save_state() const {
        return Memento{text_, cursor_};
    }

    void restore_state(const Memento& m) {
        text_ = m.text_;
        cursor_ = m.cursor_pos_;
    }

    void print() const {
        std::cout << "\"" << text_ << "\" (cursor at " << cursor_ << ")\n";
    }

private:
    std::string text_;
    int cursor_ = 0;
};

// ═══════════ Caretaker: manages undo history ═══════════
class History {
    std::vector<TextEditor::Memento> snapshots_;
public:
    void save(TextEditor::Memento m) {
        snapshots_.push_back(std::move(m));
    }

    TextEditor::Memento undo() {
        auto m = std::move(snapshots_.back());
        snapshots_.pop_back();
        return m;
    }

    bool can_undo() const { return !snapshots_.empty(); }
};

int main() {
    TextEditor editor;
    History history;

    history.save(editor.save_state());  // snapshot: ""
    editor.type("Hello");
    editor.print();  // "Hello" (cursor at 5)

    history.save(editor.save_state());  // snapshot: "Hello"
    editor.type(" World");
    editor.print();  // "Hello World" (cursor at 11)

    // Undo
    editor.restore_state(history.undo());
    editor.print();  // "Hello" (cursor at 5)

    editor.restore_state(history.undo());
    editor.print();  // "" (cursor at 0)
}

```

**Output:**

```text

"Hello" (cursor at 5)
"Hello World" (cursor at 11)
"Hello" (cursor at 5)
"" (cursor at 0)

```

### Q2: Use move semantics to transfer memento ownership cheaply

**Answer:**

```cpp

#include <iostream>
#include <memory>
#include <string>
#include <vector>

class Document {
public:
    // Memento holding potentially large data → move, don't copy
    struct Memento {
        std::string content;          // Could be megabytes
        std::vector<int> formatting;  // Large auxiliary data

        // Enable move, disable copy for large mementos
        Memento(std::string c, std::vector<int> f)
            : content(std::move(c)), formatting(std::move(f)) {}
        Memento(Memento&&) noexcept = default;
        Memento& operator=(Memento&&) noexcept = default;
        Memento(const Memento&) = delete;             // No accidental copies
        Memento& operator=(const Memento&) = delete;
    };

    void set_content(std::string s) { content_ = std::move(s); }

    // save_state: moves internal state into memento (O(1) for strings > SSO)
    Memento save_state() {
        return Memento{content_, formatting_};  // Copies here — we keep our data
    }

    // restore_state: moves memento data back in (cheap transfer)
    void restore_state(Memento m) {
        content_ = std::move(m.content);       // O(1) move
        formatting_ = std::move(m.formatting); // O(1) move
    }

    void print() const { std::cout << "Content: " << content_ << '\n'; }

private:
    std::string content_;
    std::vector<int> formatting_;
};

// Caretaker with unique_ptr for heap-allocated mementos
class UndoStack {
    std::vector<std::unique_ptr<Document::Memento>> stack_;
public:
    void push(Document::Memento m) {
        // Move memento onto heap
        stack_.push_back(std::make_unique<Document::Memento>(std::move(m)));
    }

    Document::Memento pop() {
        auto m = std::move(*stack_.back());  // Move out of unique_ptr
        stack_.pop_back();
        return m;                            // NRVO or move
    }

    bool empty() const { return stack_.empty(); }
};

int main() {
    Document doc;
    UndoStack undo;

    doc.set_content("Version 1 — lots of text here...");
    undo.push(doc.save_state());   // Snapshot saved (copy of content)

    doc.set_content("Version 2 — different text");
    doc.print();  // Content: Version 2 — different text

    doc.restore_state(undo.pop()); // Move back — no copies!
    doc.print();  // Content: Version 1 — lots of text here...
}

```

### Q3: Show how the Memento encapsulates state without exposing internals (opaque snapshot)

**Answer:**

```cpp

#include <iostream>
#include <string>
#include <vector>

class SpreadsheetCell {
public:
    // Memento is a public type but has PRIVATE data
    // Only SpreadsheetCell (friend) can access internals
    class Snapshot {
        friend class SpreadsheetCell;  // ← ONLY the originator can read

        double value_;
        std::string formula_;
        bool is_bold_;
        
        Snapshot(double v, std::string f, bool b)
            : value_(v), formula_(std::move(f)), is_bold_(b) {}
    public:
        // Caretaker can ONLY see metadata, not the actual state
        std::size_t estimated_size() const {
            return sizeof(*this) + formula_.size();
        }
        // No getters for value_, formula_, is_bold_ — truly opaque!
    };

    void set_formula(std::string f) { formula_ = std::move(f); value_ = 42.0; }
    void set_bold(bool b) { is_bold_ = b; }

    Snapshot save() const { return Snapshot{value_, formula_, is_bold_}; }

    void restore(const Snapshot& s) {
        value_ = s.value_;        // ← OK: SpreadsheetCell is friend
        formula_ = s.formula_;
        is_bold_ = s.is_bold_;
    }

    void print() const {
        std::cout << "Cell: " << formula_ << " = " << value_
                  << (is_bold_ ? " [BOLD]" : "") << '\n';
    }

private:
    double value_ = 0.0;
    std::string formula_;
    bool is_bold_ = false;
};

int main() {
    SpreadsheetCell cell;
    cell.set_formula("=A1+B1");
    cell.set_bold(true);
    cell.print();  // Cell: =A1+B1 = 42 [BOLD]

    auto snapshot = cell.save();
    // snapshot.value_;    ← COMPILE ERROR: private member
    // snapshot.formula_;  ← COMPILE ERROR: private member
    // Only metadata is accessible:
    std::cout << "Snapshot size: " << snapshot.estimated_size() << " bytes\n";

    cell.set_formula("=C1*2");
    cell.set_bold(false);
    cell.print();  // Cell: =C1*2 = 42 [BOLD=false]

    cell.restore(snapshot);
    cell.print();  // Cell: =A1+B1 = 42 [BOLD]
}

```

**Output:**

```text

Cell: =A1+B1 = 42 [BOLD]
Snapshot size: 55 bytes
Cell: =C1*2 = 42
Cell: =A1+B1 = 42 [BOLD]

```

---

## Notes

- **friend class** is the standard C++ technique for opaque mementos — the caretaker holds the snapshot but can't inspect it
- **Move semantics** make memento creation/restoration cheap for large objects (strings, vectors)
- **Consider memory limits** — store only the last N snapshots, or compress old mementos
- **Incremental mementos** (delta/diff) save memory for large objects that change slightly between snapshots**

- The Memento encapsulates state without exposing internals (opaque snapshot).

---

## Notes

_Add your own notes, examples, and observations here._

```cpp

// Your practice code

```
