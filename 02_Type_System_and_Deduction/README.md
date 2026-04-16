# Type System & Deduction

Type deduction rules (auto, decltype), type traits, concepts, CTAD, and type-safe vocabulary types (optional, variant, expected).

**Topics:** 31

## Contents

- [Know all the ways a type can be incomplete and the rules around using incomplete](Know_all_the_ways_a_type_can_be_incomplete_and_the_rules_around_using_incomplete.md)
- [Know how to use stdcommon reference t in range iterator requirements](Know_how_to_use_stdcommon_reference_t_in_range_iterator_requirements.md)
- [Know stdin place t and stdin place type t for in-place construction](Know_stdin_place_t_and_stdin_place_type_t_for_in-place_construction.md)
- [Know the standard type traits and how to use them in template constraints](Know_the_standard_type_traits_and_how_to_use_them_in_template_constraints.md)
- [Know when and how to use stdexpected C++23](Know_when_and_how_to_use_stdexpected_C++23.md)
- [Master auto type deduction rules and when they differ from template deduction](Master_auto_type_deduction_rules_and_when_they_differ_from_template_deduction.md)
- [Understand CTAD deduction guides for user-defined class templates](Understand_CTAD_deduction_guides_for_user-defined_class_templates.md)
- [Understand Concepts C++20 as named type constraints](Understand_Concepts_C++20_as_named_type_constraints.md)
- [Understand class template argument deduction CTAD C++17](Understand_class_template_argument_deduction_CTAD_C++17.md)
- [Understand decltype and decltypeauto](Understand_decltype_and_decltypeauto.md)
- [Understand overload resolution ranking of conversion sequences](Understand_overload_resolution_ranking_of_conversion_sequences.md)
- [Understand stdany and when to prefer it over void or variant](Understand_stdany_and_when_to_prefer_it_over_void_or_variant.md)
- [Understand stdcommon reference and its role in ranges](Understand_stdcommon_reference_and_its_role_in_ranges.md)
- [Understand stdconditional t for selecting types at compile time](Understand_stdconditional_t_for_selecting_types_at_compile_time.md)
- [Understand stdis nothrow move constructible and its effect on vector](Understand_stdis_nothrow_move_constructible_and_its_effect_on_vector.md)
- [Understand the Concepts standard library stdregular stdsemiregular stdcopyable](Understand_the_Concepts_standard_library_stdregular_stdsemiregular_stdcopyable.md)
- [Understand the difference between stddecay and stdremove reference](Understand_the_difference_between_stddecay_and_stdremove_reference.md)
- [Understand the interaction between auto and initializer list deduction](Understand_the_interaction_between_auto_and_initializer_list_deduction.md)
- [Use stdconvertible to and stdconstructible from concepts](Use_stdconvertible_to_and_stdconstructible_from_concepts.md)
- [Use stdis invocable and stdinvoke result for callable trait introspection](Use_stdis_invocable_and_stdinvoke_result_for_callable_trait_introspection.md)
- [Use stdis nothrow move constructible to guide safe generic code](Use_stdis_nothrow_move_constructible_to_guide_safe_generic_code.md)
- [Use stdis trivially copyable to validate memcpy-safe types](Use_stdis_trivially_copyable_to_validate_memcpy-safe_types.md)
- [Use stdoptional correctly as a nullable value type](Use_stdoptional_correctly_as_a_nullable_value_type.md)
- [Use stdremove pointer and stdadd pointer in pointer type transformations](Use_stdremove_pointer_and_stdadd_pointer_in_pointer_type_transformations.md)
- [Use stdspan with static extent for compile-time size checking](Use_stdspan_with_static_extent_for_compile-time_size_checking.md)
- [Use stdtype identity to prevent deduction in template arguments](Use_stdtype_identity_to_prevent_deduction_in_template_arguments.md)
- [Use stdtype index for runtime type keys in maps](Use_stdtype_index_for_runtime_type_keys_in_maps.md)
- [Use stdunderlying type to work with enum underlying types](Use_stdunderlying_type_to_work_with_enum_underlying_types.md)
- [Use stdvariant and stdvisit for type-safe unions](Use_stdvariant_and_stdvisit_for_type-safe_unions.md)
- [Use type aliases with using instead of typedef](Use_type_aliases_with_using_instead_of_typedef.md)
- [Use variable templates as type trait shorthands](Use_variable_templates_as_type_trait_shorthands.md)

## Notes

- uto deduction strips references and cv-qualifiers — use decltype(auto) to preserve them
- CTAD (Class Template Argument Deduction) reduces verbosity but can produce surprising results
- decltype(expr) vs decltype((expr)) — the extra parentheses change the result to a reference
- Template argument deduction and uto follow almost identical rules (with minor exceptions)
- Type traits (std::is_same_v, std::decay_t) are essential for writing correct generic code
- Forwarding references (T&& in template context) are not rvalue references — know the difference
- std::common_type is used by the standard library to resolve mixed-type expressions
- Deduction guides help CTAD work correctly for user-defined types
- uto in return types can silently change API when implementation changes — be cautious
- Use static_assert with type traits to enforce type constraints at compile time
