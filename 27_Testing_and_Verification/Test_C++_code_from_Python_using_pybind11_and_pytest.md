# Test C++ code from Python using pybind11 and pytest

**Category:** Testing & Verification  
**Item:** #589  
**Standard:** C++20  
**Reference:** <https://pybind11.readthedocs.io/>  

---

## Topic Overview

This file focuses on **property-based testing with Python's Hypothesis library** against C++ code and **understanding pybind11 call overhead**. (See files #767 and #687 for basic pybind11 bindings, parametrize, and embedded interpreter usage.)

### pybind11 Call Overhead

```cpp

Python  ──call──▶  pybind11 wrapper  ──call──▶  C++ function
        ~100-300ns        type conversion         native speed
        overhead          + GIL release

```

| Call Pattern             | Overhead per Call | Acceptable For |
| --- | :---: | :---: |
| Simple scalar return     | ~100 ns | ✅ any test |
| Vector conversion        | ~1 μs / 1000 elems | ✅ unit tests |
| Large NumPy buffer copy  | ~10 μs / 1M elems | ✅ integration tests |
| Thousands of tiny calls  | measurable | ⚠️ batch if possible |

---

## Self-Assessment

### Q1: Write a pybind11 module that exposes a C++ function and write a pytest test for it

**Answer:**

```cpp

// ═══════════ string_utils.hpp ═══════════
#pragma once
#include <string>
#include <algorithm>
#include <cctype>
#include <sstream>
#include <vector>

namespace strutil {

std::string to_upper(const std::string& s) {
    std::string result = s;
    std::transform(result.begin(), result.end(), result.begin(),
                   [](unsigned char c) { return std::toupper(c); });
    return result;
}

std::string to_lower(const std::string& s) {
    std::string result = s;
    std::transform(result.begin(), result.end(), result.begin(),
                   [](unsigned char c) { return std::tolower(c); });
    return result;
}

bool is_palindrome(const std::string& s) {
    auto clean = to_lower(s);
    clean.erase(std::remove_if(clean.begin(), clean.end(),
                [](unsigned char c) { return !std::isalnum(c); }),
                clean.end());
    return std::equal(clean.begin(), clean.begin() + clean.size() / 2,
                      clean.rbegin());
}

std::vector<std::string> split(const std::string& s, char delim) {
    std::vector<std::string> tokens;
    std::istringstream iss(s);
    std::string token;
    while (std::getline(iss, token, delim))
        if (!token.empty()) tokens.push_back(token);
    return tokens;
}

} // namespace strutil

```

```cpp

// ═══════════ bindings.cpp ═══════════
#include <pybind11/pybind11.h>
#include <pybind11/stl.h>
#include "string_utils.hpp"

namespace py = pybind11;

PYBIND11_MODULE(string_utils, m) {
    m.doc() = "C++ string utilities";
    m.def("to_upper", &strutil::to_upper);
    m.def("to_lower", &strutil::to_lower);
    m.def("is_palindrome", &strutil::is_palindrome);
    m.def("split", &strutil::split, py::arg("s"), py::arg("delim") = ' ');
}

```

```python

# ═══════════ test_string_utils.py ═══════════
import string_utils
import pytest

class TestToUpper:
    def test_basic(self):
        assert string_utils.to_upper("hello") == "HELLO"

    def test_mixed(self):
        assert string_utils.to_upper("Hello World") == "HELLO WORLD"

    def test_empty(self):
        assert string_utils.to_upper("") == ""

    def test_already_upper(self):
        assert string_utils.to_upper("ABC") == "ABC"

class TestPalindrome:
    @pytest.mark.parametrize("s", [
        "racecar", "A man a plan a canal Panama",
        "Was it a car or a cat I saw", "madam",
    ])
    def test_true_palindromes(self, s):
        assert string_utils.is_palindrome(s)

    @pytest.mark.parametrize("s", ["hello", "python", "ab"])
    def test_false_palindromes(self, s):
        assert not string_utils.is_palindrome(s)

class TestSplit:
    def test_default_space(self):
        assert string_utils.split("a b c") == ["a", "b", "c"]

    def test_custom_delimiter(self):
        assert string_utils.split("a,b,c", ',') == ["a", "b", "c"]

    def test_empty_string(self):
        assert string_utils.split("") == []

```

### Q2: Use Python's hypothesis library for property-based tests of the exposed C++ function

**Answer:**

```python

# ═══════════ test_hypothesis.py ═══════════
# Property-based testing: Hypothesis generates random inputs
# and checks that invariants always hold

import string_utils
from hypothesis import given, assume, settings, example
from hypothesis import strategies as st

# ── Property: to_upper(to_lower(s)) == to_upper(s) ──
@given(st.text(alphabet=st.characters(whitelist_categories=('L', 'N', 'Z'))))
def test_upper_lower_roundtrip(s):
    """to_upper is idempotent after to_lower."""
    assert string_utils.to_upper(string_utils.to_lower(s)) == string_utils.to_upper(s)


# ── Property: to_lower is idempotent ──
@given(st.text())
def test_lower_idempotent(s):
    result = string_utils.to_lower(s)
    assert string_utils.to_lower(result) == result


# ── Property: palindrome reversal ──
@given(st.text(alphabet=st.characters(whitelist_categories=('L',)), min_size=1))
def test_palindrome_of_mirror(s):
    """s + reverse(s) is always a palindrome."""
    mirrored = s + s[::-1]
    assert string_utils.is_palindrome(mirrored)


# ── Property: split then join roundtrip ──
@given(st.lists(st.text(
    alphabet=st.characters(blacklist_characters=','),
    min_size=1
), min_size=1))
def test_split_join_roundtrip(parts):
    """split(join(parts, ','), ',') == parts"""
    joined = ",".join(parts)
    result = string_utils.split(joined, ',')
    assert result == parts


# ── Property: to_upper preserves length ──
@given(st.text())
def test_upper_preserves_length(s):
    assert len(string_utils.to_upper(s)) == len(s)


# ── Targeted example: Hypothesis can shrink failing inputs ──
@given(st.text(min_size=0, max_size=1000))
@example("")             # Explicit edge case
@example("a")            # Single char
@example("\x00\xff")     # Null and high bytes
@settings(max_examples=500)
def test_lower_upper_length_always_preserved(s):
    assert len(string_utils.to_lower(s)) == len(s)
    assert len(string_utils.to_upper(s)) == len(s)

```

### Q3: Explain the round-trip cost of pybind11 calls and when it makes test latency acceptable

**Answer:**

```python

# ═══════════ Benchmark: measuring pybind11 overhead ═══════════
import string_utils
import time

def benchmark_call_overhead():
    """Measure per-call overhead of pybind11 dispatch."""
    N = 100_000
    s = "hello"

    # pybind11 calls
    start = time.perf_counter_ns()
    for _ in range(N):
        string_utils.to_upper(s)
    elapsed_cpp = (time.perf_counter_ns() - start) / N

    # Pure Python equivalent
    start = time.perf_counter_ns()
    for _ in range(N):
        s.upper()
    elapsed_py = (time.perf_counter_ns() - start) / N

    print(f"pybind11 to_upper: {elapsed_cpp:.0f} ns/call")
    print(f"Python   .upper(): {elapsed_py:.0f} ns/call")
    print(f"Overhead:          {elapsed_cpp - elapsed_py:.0f} ns/call")

# Typical results:
#   pybind11 to_upper: ~250 ns/call
#   Python   .upper(): ~80 ns/call
#   Overhead:          ~170 ns/call

```

**When the overhead is acceptable:**

| Scenario | Calls/Test | Overhead | Verdict |
| --- | :---: | :---: | :---: |
| Unit test: one function call | 1 | ~200 ns | ✅ Negligible |
| Parametrized: 100 inputs | 100 | ~20 μs | ✅ Negligible |
| Hypothesis: 500 examples | 500 | ~100 μs | ✅ Fine |
| Stress test: 1M calls | 1M | ~200 ms | ⚠️ Noticeable |
| Data pipeline: large buffers | 1 | ~1 μs (buffer copy) | ✅ Amortized |

**Key insight:** The overhead is per-call, not per-byte. For compute-heavy C++ functions (matrix operations, parsing, compression), the pybind11 dispatch cost is negligible compared to the work inside. The overhead only matters when you make **many tiny calls** — in that case, batch operations into a single C++ call that processes a vector.

```python

# ── Bad: many tiny calls ──
# for x in big_list:
#     result.append(string_utils.to_upper(x))  # 200ns overhead × N

# ── Good: batch call ──
# results = string_utils.to_upper_batch(big_list)  # One call, C++ loops internally

```

---

## Notes

- Hypothesis shrinks failing inputs to minimal reproducing cases — invaluable for finding edge cases in C++ parsers
- `pip install hypothesis` — no C++ changes needed
- For NumPy buffers, use `pybind11/numpy.h` for zero-copy access (no conversion overhead)
- Run Hypothesis with `--hypothesis-show-statistics` to see how many examples were generated
- Combine with sanitizers: build the C++ module with `-fsanitize=address` and Hypothesis will find memory bugs
