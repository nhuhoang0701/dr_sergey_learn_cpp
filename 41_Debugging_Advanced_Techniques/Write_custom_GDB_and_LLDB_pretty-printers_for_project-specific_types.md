# Write custom GDB and LLDB pretty-printers for project-specific types

**Category:** Debugging Advanced Techniques  
**Standard:** C++17  
**Reference:** <https://sourceware.org/gdb/current/onlinedocs/gdb.html/Pretty-Printing.html>  

---

## Topic Overview

When you `print my_vec` in GDB on a raw `std::map` or your own custom type, you get something like `{_M_elems = {0x7fff5fbff8a0, 0x7fff5fbff8b0, ...}}` - internal representation all the way down. Reading that for a single variable already takes mental effort; doing it for twenty variables in a debugging session compounds into a real productivity drain.

Pretty-printers solve this by teaching the debugger how to display your types in a human-friendly format. They're Python scripts loaded by GDB or LLDB that intercept the display of matching types and format them however you want. The standard library types you use every day (`std::vector`, `std::map`, `std::string`) already have pretty-printers shipped with libstdc++ and libc++. Writing your own for project-specific types brings the same quality of display to your entire codebase.

### GDB Pretty-Printer (Python)

A GDB pretty-printer is a Python class with a `to_string()` method (for a one-line summary) and optionally a `children()` method (for expandable sub-elements). You register a collection of printers keyed by type name regexes, and GDB calls the right one automatically:

```python
# my_printers.py - load with: (gdb) source my_printers.py
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

The `children()` method on `MatrixPrinter` is what makes the type expandable in the debugger - when you drill into the `Matrix` value, GDB calls `children()` and shows each `[row,col]` entry as a named sub-element. Without `children()`, the printer only shows the summary line.

### LLDB Pretty-Printer (Python)

LLDB uses a slightly different API. Instead of a printer class per type, you write a summary function that returns a string, and register it for a type name using `type summary add`. The `__lldb_init_module` function is the entry point that LLDB calls when you import your script:

```python
# my_formatters.py - load with: (lldb) command script import my_formatters.py
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

The `valobj` parameter is an `lldb.SBValue` representing the variable being printed. `GetChildMemberWithName('x')` navigates to the `x` member, and `GetValueAsFloat()` extracts its value as a Python float. The resulting string is what LLDB shows when it displays a `Vec3`.

### Auto-loading via .gdbinit

Manually typing `source my_printers.py` every time you start GDB is tedious. GDB has an auto-loading mechanism: if you name your printer file `<objfile>-gdb.py` (e.g., `libmylib.so-gdb.py`), GDB loads it automatically when it loads that shared library. For project-local setups, a `.gdbinit` in the project root is simpler:

```bash
# project-root/.gdbinit
set auto-load safe-path /path/to/project

# Or system-wide ~/.gdbinit
add-auto-load-safe-path /home/user/projects

# The pretty-printer file is auto-loaded if named:
# objfile-gdb.py (e.g., libmylib.so-gdb.py)
```

The `set auto-load safe-path` line is necessary because GDB's auto-loading security policy refuses to load scripts from arbitrary paths by default. You have to explicitly tell GDB which paths are trusted.

---

## Self-Assessment

### Q1: How do pretty-printers improve debugging productivity

The difference is stark. Without a printer: `$1 = {_M_elems = {0x7fff5fbff8a0, 0x7fff5fbff8b0, ...}}`. With a printer: `$1 = Vec3(1.00, 2.00, 3.00)`. That's the difference between reading raw internal state and reading the actual logical value. For complex types like matrices, trees, or hash maps, the unformatted output can be essentially unreadable without the printer. Over a debugging session where you inspect dozens of variables, a few minutes of saved mental parsing per variable adds up quickly.

### Q2: Show a printer for a custom hash map

This example shows the `children()` pattern used to display key-value pairs, and the `display_hint()` method that tells GDB to format the output as a map (alternating key/value pairs displayed with colons):

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

The `display_hint()` returning `'map'` is the detail that makes GDB format the output as `{key1: value1, key2: value2}` instead of a flat numbered list. The other valid hints are `'array'` (numbered elements) and `'string'` (display as a string).

### Q3: How to debug the pretty-printer itself

Pretty-printers break in ways that can be hard to diagnose if the error is silently swallowed. GDB and LLDB both give you ways to inspect what's happening:

```bash
# In GDB:
(gdb) python import traceback; traceback.print_exc()
(gdb) python print(gdb.parse_and_eval("my_var").type)

# In LLDB:
(lldb) script import my_formatters
(lldb) script print(lldb.frame.FindVariable("my_var").GetType())
```

If a printer crashes, GDB falls back to showing the raw value silently. Running the `traceback.print_exc()` line will show you the Python exception that was suppressed. The `parse_and_eval("my_var").type` call lets you inspect the exact type name that GDB is seeing, which is useful when your regex doesn't match because the type name includes a namespace or template arguments you didn't account for.

---

## Notes

- Most IDE debuggers (CLion, VS Code) use GDB/LLDB underneath and benefit from pretty-printers.
- libstdc++ and libc++ ship with comprehensive pretty-printers for standard library types.
- Put pretty-printer scripts in your project repo - they help the whole team.
- `display_hint()` returning `'map'`, `'array'`, or `'string'` controls GDB formatting.
