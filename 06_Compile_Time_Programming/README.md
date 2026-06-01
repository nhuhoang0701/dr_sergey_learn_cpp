# Compile-Time Programming

constexpr, consteval, static_assert, template metaprogramming, and compile-time computation techniques.

**Topics:** 29

## Contents

- [Build a compile-time finite state machine using template specialization](Build_a_compile-time_finite_state_machine_using_template_specialization.md)
- [Generate dispatch tables at compile time using parameter pack expansion](Generate_dispatch_tables_at_compile_time_using_parameter_pack_expansion.md)
- [Know constexpr exception support status and workarounds](Know_constexpr_exception_support_status_and_workarounds.md)
- [Understand SFINAE with enable if vs concepts choose concepts](Understand_SFINAE_with_enable_if_vs_concepts_choose_concepts.md)
- [Understand expansion statements for Cpp26](Understand_expansion_statements_for_Cpp26.md)
- [Understand how to use if consteval C++23 to branch on constant evaluation contex](Understand_how_to_use_if_consteval_C++23_to_branch_on_constant_evaluation_contex.md)
- [Understand pack indexing C++26 for accessing parameter pack elements by index](Understand_pack_indexing_C++26_for_accessing_parameter_pack_elements_by_index.md)
- [Understand requires expressions and the requires clause C++20](Understand_requires_expressions_and_the_requires_clause_C++20.md)
- [Understand static assert and its role in template constraints](Understand_static_assert_and_its_role_in_template_constraints.md)
- [Use   cpp  feature test macros to detect standard library features](Use___cpp__feature_test_macros_to_detect_standard_library_features.md)
- [Use consteval for functions that must only run at compile time C++20](Use_consteval_for_functions_that_must_only_run_at_compile_time_C++20.md)
- [Use constexpr containers and algorithms C++20](Use_constexpr_containers_and_algorithms_C++20.md)
- [Use constexpr dynamic memory allocation with transient allocation Cpp20](Use_constexpr_dynamic_memory_allocation_with_transient_allocation_Cpp20.md)
- [Use constexpr lambdas C++17 in compile-time algorithms](Use_constexpr_lambdas_C++17_in_compile-time_algorithms.md)
- [Use constexpr lambdas as compile-time predicates](Use_constexpr_lambdas_as_compile-time_predicates.md)
- [Use if constexpr to select code branches at compile time](Use_if_constexpr_to_select_code_branches_at_compile_time.md)
- [Use static assert with concept-based diagnostics for better error messages](Use_static_assert_with_concept-based_diagnostics_for_better_error_messages.md)
- [Use stdarray as a compile-time lookup table](Use_stdarray_as_a_compile-time_lookup_table.md)
- [Use stdarray with constexpr to generate lookup tables at compile time](Use_stdarray_with_constexpr_to_generate_lookup_tables_at_compile_time.md)
- [Use stdbit width stdcountl zero and other bit utilities C++20](Use_stdbit_width_stdcountl_zero_and_other_bit_utilities_C++20.md)
- [Use stdintegral constant and compile-time value wrappers](Use_stdintegral_constant_and_compile-time_value_wrappers.md)
- [Use stdis constant evaluated C++20 for dual compileruntime paths](Use_stdis_constant_evaluated_C++20_for_dual_compileruntime_paths.md)
- [Use stdmake integer sequence and stdindex sequence for compile-time iteration](Use_stdmake_integer_sequence_and_stdindex_sequence_for_compile-time_iteration.md)
- [Use stdto array C++20 to deduce array size from initializer](Use_stdto_array_C++20_to_deduce_array_size_from_initializer.md)
- [Write a compile-time prime sieve as a constexpr stdarray](Write_a_compile-time_prime_sieve_as_a_constexpr_stdarray.md)
- [Write a compile-time string parser using consteval](Write_a_compile-time_string_parser_using_consteval.md)
- [Write compile-time hash maps using constexpr arrays perfect hashing](Write_compile-time_hash_maps_using_constexpr_arrays_perfect_hashing.md)
- [Write compile-time string parsing using consteval and stdstring view](Write_compile-time_string_parsing_using_consteval_and_stdstring_view.md)
- [Write constexpr functions that work both at compile time and runtime](Write_constexpr_functions_that_work_both_at_compile_time_and_runtime.md)

## Notes

- `constexpr` functions can run at both compile time and runtime - the compiler decides based on context.
- `consteval` (C++20) forces compile-time evaluation - use it for immediate functions that must never run at runtime.
- `constexpr` containers (`std::vector`, `std::string`) arrived in C++20 for transient allocations.
- Template metaprogramming is being replaced by `constexpr` and `consteval` for most use cases.
- `static_assert` validates compile-time conditions with clear error messages.
- `std::is_constant_evaluated()` lets a function detect whether it's running at compile time.
- Compile-time string processing is possible with `constexpr std::string` in C++20.
- `constexpr` virtual functions work since C++20 for compile-time polymorphism.
- Fold expressions (C++17) replace recursive template patterns for variadic operations.
- `std::source_location` provides compile-time file/line info without macros.
