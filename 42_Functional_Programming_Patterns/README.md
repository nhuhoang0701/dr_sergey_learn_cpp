# Functional Programming Patterns in C++

Applying functional programming concepts in modern C++: monadic error chaining, algebraic data types, persistent data structures, expression templates, continuation-passing style, and pattern matching.

**Topics:** 9

## Contents

- [Apply the expression template pattern for lazy evaluation](Apply_the_expression_template_pattern_for_lazy_evaluation.md)
- [Design pure functions and referential transparency for testability](Design_pure_functions_and_referential_transparency_for_testability.md)
- [Implement monadic error chaining with std::expected and std::optional](Implement_monadic_error_chaining_with_stdstdexpected_and_stdstdoptional.md)
- [Implement pattern matching manually with std::visit overload sets](Implement_pattern_matching_manually_with_stdstdvisit_overload_sets.md)
- [Implement persistent and immutable data structures in C++](Implement_persistent_and_immutable_data_structures_in_Cpp.md)
- [Implement railway-oriented programming for error handling](Implement_railway-oriented_programming_for_error_handling.md)
- [Use algebraic data types with std::variant as sum type and std::tuple as product type](Use_algebraic_data_types_with_stdstdvariant_as_sum_type_and_stdstdtuple_as_produ.md)
- [Use continuation-passing style CPS with coroutines and senders](Use_continuation-passing_style_CPS_with_coroutines_and_senders.md)
- [Use std::views pipeline as a functional transformation chain](Use_stdstdviews_pipeline_as_a_functional_transformation_chain.md)

## Notes

- Monadic chaining with std::optional::and_then / std::expected::and_then (C++23) enables composable error handling
- Pure functions (no side effects, same input → same output) are easier to test and parallelize
- std::variant + std::visit implements algebraic data types (sum types) in C++
- Expression templates defer computation — used in linear algebra libraries (Eigen, Blaze)
- Persistent (immutable) data structures share unchanged parts — efficient for undo/version history
- Railway-oriented programming chains operations that may fail, short-circuiting on first error
- std::views pipelines (C++20) provide lazy functional transformations over ranges
- Continuation-passing style (CPS) maps naturally to coroutines and sender/receiver patterns
- Pattern matching proposals (C++26) aim to add inspect expressions for functional-style dispatch
- Avoid mutable shared state — the core principle of functional programming applies directly to concurrent C++
