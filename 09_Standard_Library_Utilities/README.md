# Standard Library — Utilities

Utility types and functions: tuple, pair, chrono, formatting, string operations, and more.

**Topics:** 37

## Contents

- [Know stdfunction and its performance characteristics](Know_stdfunction_and_its_performance_characteristics.md)
- [Understand stdstdinteger sequence patterns for index tricks](Understand_stdstdinteger_sequence_patterns_for_index_tricks.md)
- [Use stdaddressof to safely get an objects address despite operator overloads](Use_stdaddressof_to_safely_get_an_objects_address_despite_operator_overloads.md)
- [Use stdbit cast C++20 for safe type punning](Use_stdbit_cast_C++20_for_safe_type_punning.md)
- [Use stdbit floor and bit ceil C++20 for power-of-two alignment](Use_stdbit_floor_and_bit_ceil_C++20_for_power-of-two_alignment.md)
- [Use stdbyte for type-safe raw memory manipulation](Use_stdbyte_for_type-safe_raw_memory_manipulation.md)
- [Use stdbyteswap C++23 and stdendian for portable endianness handling](Use_stdbyteswap_C++23_and_stdendian_for_portable_endianness_handling.md)
- [Use stdchrono for time measurement and duration arithmetic](Use_stdchrono_for_time_measurement_and_duration_arithmetic.md)
- [Use stdfilesystem for portable file system operations C++17](Use_stdfilesystem_for_portable_file_system_operations_C++17.md)
- [Use stdformat C++20 for type-safe string formatting](Use_stdformat_C++20_for_type-safe_string_formatting.md)
- [Use stdgenerator C++23 as a standard coroutine range](Use_stdgenerator_C++23_as_a_standard_coroutine_range.md)
- [Use stdhash and define custom hash specializations](Use_stdhash_and_define_custom_hash_specializations.md)
- [Use stdinitializer list correctly in constructors and assignment](Use_stdinitializer_list_correctly_in_constructors_and_assignment.md)
- [Use stdlinalg C++26 for linear algebra operations](Use_stdlinalg_C++26_for_linear_algebra_operations.md)
- [Use stdmdspan C++23 for multi-dimensional array views](Use_stdmdspan_C++23_for_multi-dimensional_array_views.md)
- [Use stdmidpoint and stdlerp for safe numeric interpolation](Use_stdmidpoint_and_stdlerp_for_safe_numeric_interpolation.md)
- [Use stdnumeric limits to write portable numeric code](Use_stdnumeric_limits_to_write_portable_numeric_code.md)
- [Use stdoptional monadic operations transform and then or else C++23](Use_stdoptional_monadic_operations_transform_and_then_or_else_C++23.md)
- [Use stdprint and stdprintln C++23 for formatted output](Use_stdprint_and_stdprintln_C++23_for_formatted_output.md)
- [Use stdrandom device and mt19937 for high-quality random number generation](Use_stdrandom_device_and_mt19937_for_high-quality_random_number_generation.md)
- [Use stdratio for compile-time rational arithmetic](Use_stdratio_for_compile-time_rational_arithmetic.md)
- [Use stdreference wrapper for storing references in containers](Use_stdreference_wrapper_for_storing_references_in_containers.md)
- [Use stdsource location C++20 for zero-overhead logging metadata](Use_stdsource_location_C++20_for_zero-overhead_logging_metadata.md)
- [Use stdspanstream C++23 for in-memory IO on raw buffers](Use_stdspanstream_C++23_for_in-memory_IO_on_raw_buffers.md)
- [Use stdssize C++20 to get a signed size from a container](Use_stdssize_C++20_to_get_a_signed_size_from_a_container.md)
- [Use stdstacktrace C++23 for diagnostic stack capture](Use_stdstacktrace_C++23_for_diagnostic_stack_capture.md)
- [Use stdstdchrono calendar and timezone features Cpp20](Use_stdstdchrono_calendar_and_timezone_features_Cpp20.md)
- [Use stdstdcopyable function Cpp26 vs stdstdmove only function Cpp23 vs stdstdfun](Use_stdstdcopyable_function_Cpp26_vs_stdstdmove_only_function_Cpp23_vs_stdstdfun.md)
- [Use stdstdexpected vs stdstdoptional decision matrix](Use_stdstdexpected_vs_stdstdoptional_decision_matrix.md)
- [Use stdstdis within lifetime Cpp26 for constexpr lifetime queries](Use_stdstdis_within_lifetime_Cpp26_for_constexpr_lifetime_queries.md)
- [Use stdstrings from chars and to chars for locale-independent conversion](Use_stdstrings_from_chars_and_to_chars_for_locale-independent_conversion.md)
- [Use stdstrong ordering weak ordering partial ordering as return types](Use_stdstrong_ordering_weak_ordering_partial_ordering_as_return_types.md)
- [Use stdtie for lexicographic comparison of multiple struct members](Use_stdtie_for_lexicographic_comparison_of_multiple_struct_members.md)
- [Use stdto chars and stdfrom chars for allocation-free conversions](Use_stdto_chars_and_stdfrom_chars_for_allocation-free_conversions.md)
- [Use stdtuple and stdget for heterogeneous value grouping](Use_stdtuple_and_stdget_for_heterogeneous_value_grouping.md)
- [Use stdviewsas const C++23 to create const views of mutable ranges](Use_stdviewsas_const_C++23_to_create_const_views_of_mutable_ranges.md)
- [Use stdviewskeys and viewsvalues to iterate map keys or values](Use_stdviewskeys_and_viewsvalues_to_iterate_map_keys_or_values.md)

## Notes

- std::optional represents a value that may not exist — prefer over sentinel values and raw pointers
- std::variant is a type-safe union — use std::visit with overload sets for pattern matching
- std::any provides type-erased storage — use sparingly, prefer std::variant when types are known
- std::expected (C++23) combines error handling with return values — better than std::optional for errors
- std::format (C++20) replaces printf and << streams with type-safe formatting
- std::chrono provides type-safe time handling — avoid raw integers for durations
- std::filesystem (C++17) handles paths, directory traversal, and file operations portably
- std::string_view avoids copies but beware dangling — it does not own the string
- Monadic operations on std::optional (.and_then, .transform) enable functional chaining (C++23)
- std::to_underlying (C++23) safely converts scoped enums to their underlying type
