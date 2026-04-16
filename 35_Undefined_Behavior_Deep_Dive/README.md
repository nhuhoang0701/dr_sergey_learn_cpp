# Undefined Behavior Deep Dive

Comprehensive understanding of undefined, unspecified, and implementation-defined behavior in C++: UB catalog, compiler exploitation of UB, strict aliasing, pointer provenance, sequencing rules, and detection tooling.

**Topics:** 14

## Contents

- [Distinguish implementation-defined unspecified and undefined behavior](Distinguish_implementation-defined_unspecified_and_undefined_behavior.md)
- [Identify common UB in multi-threaded code beyond data races](Identify_common_UB_in_multi-threaded_code_beyond_data_races.md)
- [Know dangling reference UB in range-based for with temporaries](Know_dangling_reference_UB_in_range-based_for_with_temporaries.md)
- [Know null pointer dereference UB nuances passing vs dereferencing](Know_null_pointer_dereference_UB_nuances_passing_vs_dereferencing.md)
- [Know the complete catalog of undefined behavior in C++](Know_the_complete_catalog_of_undefined_behavior_in_C++.md)
- [Know union type-punning rules and C vs C++ differences](Know_union_type-punning_rules_and_C_vs_C++_differences.md)
- [Master strict aliasing rules with practical examples](Master_strict_aliasing_rules_with_practical_examples.md)
- [Understand how compilers exploit UB for optimization](Understand_how_compilers_exploit_UB_for_optimization.md)
- [Understand pointer provenance and stdlaunder](Understand_pointer_provenance_and_stdlaunder.md)
- [Understand restrict and restrict semantics and UB implications](Understand_restrict_and_restrict_semantics_and_UB_implications.md)
- [Understand sequencing rules and evaluation order since C++17](Understand_sequencing_rules_and_evaluation_order_since_C++17.md)
- [Understand signed integer overflow UB and why it enables optimizations](Understand_signed_integer_overflow_UB_and_why_it_enables_optimizations.md)
- [Use UBSan ASan MSan and static analyzers for UB detection](Use_UBSan_ASan_MSan_and_static_analyzers_for_UB_detection.md)
- [Use stdbit cast as the safe alternative to reinterpret cast](Use_stdbit_cast_as_the_safe_alternative_to_reinterpret_cast.md)

## Notes

- Signed integer overflow is UB — compilers exploit this for optimization (e.g., assuming i + 1 > i)
- Null pointer dereference is UB — the compiler may optimize away null checks that follow dereference
- Use-after-free, double-free, and buffer overflow are UB — sanitizers (ASan) catch these at runtime
- Strict aliasing violations (type-punning through incompatible pointers) are UB — use std::bit_cast
- Accessing an uninitialized variable of any type (except unsigned char) is UB
- Data races on non-atomic shared variables are UB — even a single concurrent read/write pair
- Infinite loops without side effects are UB in C++11+ — the compiler may remove them
- Shifting a signed value by its bit-width or more is UB — also shifting into the sign bit (pre-C++20)
- UB can manifest as "working correctly" — then break silently with a different compiler or optimization level
- -fsanitize=undefined is the single best tool for detecting UB during development
