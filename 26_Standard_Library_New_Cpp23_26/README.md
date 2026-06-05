# Standard Library — New in C++23/26

C++23 and C++26 added a lot of genuinely useful library features - not just language tweaks, but new containers, new formatting power, new concurrency tools, and new ways to write safer abstractions. This folder covers the ones worth knowing.

**Topics:** 25

## Contents

- [Understand stdhazard pointer for lock-free memory reclamation](Understand_stdhazard_pointer_for_lock-free_memory_reclamation.md)
- [Understand stdtext encoding for portable character encoding](Understand_stdtext_encoding_for_portable_character_encoding.md)
- [Use stddebugging utilities C++26 breakpoint and is debugger present](Use_stddebugging_utilities_C++26_breakpoint_and_is_debugger_present.md)
- [Use stddebugging utilities stdbreakpoint and stdis debugger present](Use_stddebugging_utilities_stdbreakpoint_and_stdis_debugger_present.md)
- [Use stdflat map and flat set in depth construction merge and extract](Use_stdflat_map_and_flat_set_in_depth_construction_merge_and_extract.md)
- [Use stdhive C++26 in depth stable addresses iteration and erasure](Use_stdhive_C++26_in_depth_stable_addresses_iteration_and_erasure.md)
- [Use stdindirect and stdpolymorphic C++26 for value-semantic polymorphism](Use_stdindirect_and_stdpolymorphic_C++26_for_value-semantic_polymorphism.md)
- [Use stdindirect and stdpolymorphic for value-semantic polymorphism](Use_stdindirect_and_stdpolymorphic_for_value-semantic_polymorphism.md)
- [Use stdinplace vector C++26 for fixed-capacity stack-allocated sequences](Use_stdinplace_vector_C++26_for_fixed-capacity_stack-allocated_sequences.md)
- [Use stdinplace vector for fixed-capacity stack allocation](Use_stdinplace_vector_for_fixed-capacity_stack_allocation.md)
- [Use stdlinalg C++26 for portable linear algebra on mdspan](Use_stdlinalg_C++26_for_portable_linear_algebra_on_mdspan.md)
- [Use stdlinalg for linear algebra without external dependencies](Use_stdlinalg_for_linear_algebra_without_external_dependencies.md)
- [Use stdrcu Read-Copy-Update for scalable read-heavy data structures](Use_stdrcu_Read-Copy-Update_for_scalable_read-heavy_data_structures.md)
- [Use stdtext encoding C++26 for portable encoding detection](Use_stdtext_encoding_C++26_for_portable_encoding_detection.md)
- [Understand compile-time format checking with stdbasic_format_string](Understand_compile-time_format_checking_with_stdbasic_format_string.md)
- [Use stdchunk_by stride and cartesian_product views C++23](Use_stdchunk_by_stride_and_cartesian_product_views_C++23.md)
- [Use stdcopyable_function C++26 as a std_function replacement](Use_stdcopyable_function_C++26_as_a_std_function_replacement.md)
- [Use stdforward_like C++23 to propagate owner value category](Use_stdforward_like_C++23_to_propagate_owner_value_category.md)
- [Use stdfunction_ref C++26 as a lightweight non-owning callable reference](Use_stdfunction_ref_C++26_as_a_lightweight_non-owning_callable_reference.md)
- [Use stdosyncstream C++20 for synchronized concurrent output](Use_stdosyncstream_C++20_for_synchronized_concurrent_output.md)
- [Use stdranges_to C++23 for materializing views into containers](Use_stdranges_to_C++23_for_materializing_views_into_containers.md)
- [Use stdto_underlying C++23 for safe enum to integer conversion](Use_stdto_underlying_C++23_for_safe_enum_to_integer_conversion.md)
- [Use stdunreachable C++23 to mark impossible code paths](Use_stdunreachable_C++23_to_mark_impossible_code_paths.md)
- [Use stdviewsenumerate C++23 for indexed range iteration](Use_stdviewsenumerate_C++23_for_indexed_range_iteration.md)
- [Use formatted ranges output with stdformat C++23](Use_formatted_ranges_output_with_stdformat_C++23.md)

## Notes

- `std::expected` (C++23) is the standard way to return a value or an error without exceptions.
- `std::flat_map` / `std::flat_set` (C++23) store elements in contiguous memory for better cache performance.
- `std::generator` (C++23) is the standard coroutine-based lazy sequence generator.
- `std::print` / `std::println` (C++23) provide Python-like formatted output.
- `std::ranges::to` (C++23) materializes lazy ranges into concrete containers.
- `std::mdspan` (C++23) provides multi-dimensional non-owning views over contiguous data.
- `std::stacktrace` (C++23) captures call stacks programmatically for diagnostics.
- `import std;` (C++23) imports the entire standard library as a named module.
- `std::views::zip` and `std::views::enumerate` (C++23) add missing range adaptors.
- Static reflection and `std::execution` are the headline features coming in C++26.
