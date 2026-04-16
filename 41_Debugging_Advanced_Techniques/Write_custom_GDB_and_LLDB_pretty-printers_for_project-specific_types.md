# Write custom GDB and LLDB pretty-printers for project-specific types

**Category:** Debugging Advanced Techniques  
**Standard:** C++17  
**Reference:** <https://sourceware.org/gdb/current/onlinedocs/gdb.html/Pretty-Printing.html>  

---

## Topic Overview

Raw debugger output for `std::map` or custom types is unreadable. Pretty-printers format types in a human-friendly way.

### GDB Pretty-Printer (Python)

```python

# my_printers.py — load with: (gdb) source my_printers.py
import gdb
import gdb.printing

class Vec3Printer:
    """Pretty-print a Vec3 struct."""
    def __init__(self, val):
        self.val = val

    def to_string(self):
        x = float(self.val['x'])
        y = float(self.val['y'])
        z = float(self.val['z'])
        return f"Vec3({x:.2f}, {y:.2f}, {z:.2f})"

class MatrixPrinter:
    """Pretty-print a Matrix class."""
    def __init__(self, val):
        self.val = val

    def to_string(self):
        rows = int(self.val['rows_'])
        cols = int(self.val['cols_'])
        return f"Matrix({rows}x{cols})"

    def children(self):
        """Yield (name, value) pairs for expandable children."""
        rows = int(self.val['rows_'])
        cols = int(self.val['cols_'])
        data = self.val['data_']
        for r in range(min(rows, 5)):
            for c in range(min(cols, 5)):
                yield f'[{r},{c}]', data[r * cols + c]

def build_printer():
    pp = gdb.printing.RegexpCollectionPrettyPrinter("myproject")
    pp.add_printer('Vec3', '^Vec3$', Vec3Printer)
    pp.add_printer('Matrix', '^Matrix$', MatrixPrinter)
    return pp

gdb.printing.register_pretty_printer(gdb.current_objfile(), build_printer())

```

### LLDB Pretty-Printer (Python)

```python

# my_formatters.py — load with: (lldb) command script import my_formatters.py
import lldb

def vec3_summary(valobj, internal_dict):
    x = valobj.GetChildMemberWithName('x').GetValueAsFloat()
    y = valobj.GetChildMemberWithName('y').GetValueAsFloat()
    z = valobj.GetChildMemberWithName('z').GetValueAsFloat()
    return f"Vec3({x:.2f}, {y:.2f}, {z:.2f})"

def __lldb_init_module(debugger, internal_dict):
    debugger.HandleCommand(
        'type summary add -F my_formatters.vec3_summary Vec3'
    )
    print("Loaded myproject pretty-printers")

```

### Auto-loading via .gdbinit

```bash

# project-root/.gdbinit
set auto-load safe-path /path/to/project

# Or system-wide ~/.gdbinit
add-auto-load-safe-path /home/user/projects

# The pretty-printer file is auto-loaded if named:
# objfile-gdb.py (e.g., libmylib.so-gdb.py)

```

---

## Self-Assessment

### Q1: How do pretty-printers improve debugging productivity

Without a printer: `$1 = {_M_elems = {0x7fff5fbff8a0, 0x7fff5fbff8b0, ...}}`. With a printer: `$1 = Vec3(1.00, 2.00, 3.00)`. This saves minutes of mental parsing per debugging session and reduces error in interpreting complex data structures.

### Q2: Show a printer for a custom hash map

```python

class FlatMapPrinter:
    def __init__(self, val):
        self.val = val
    def to_string(self):
        size = int(self.val['size_'])
        return f"FlatMap(size={size})"
    def children(self):
        buckets = self.val['buckets_']
        size = int(self.val['size_'])
        for i in range(size):
            entry = buckets[i]
            key = entry['key']
            value = entry['value']
            yield f'[{key}]', value
    def display_hint(self):
        return 'map'  # GDB displays as key: value pairs

```

### Q3: How to debug the pretty-printer itself

```bash

# In GDB:
(gdb) python import traceback; traceback.print_exc()
(gdb) python print(gdb.parse_and_eval("my_var").type)

# In LLDB:
(lldb) script import my_formatters
(lldb) script print(lldb.frame.FindVariable("my_var").GetType())

```

---

## Notes

- Most IDE debuggers (CLion, VS Code) use GDB/LLDB underneath and benefit from pretty-printers.
- libstdc++ and libc++ ship with comprehensive pretty-printers for standard library types.
- Put pretty-printer scripts in your project repo — they help the whole team.
- `display_hint()` returning `'map'`, `'array'`, or `'string'` controls GDB formatting.
