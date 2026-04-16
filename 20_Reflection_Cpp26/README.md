# Reflection (C++26)

Compile-time reflection: meta-objects, type introspection, and code generation facilities.

**Topics:** 22

## Contents

- [Enumerate enum values and names at compile time using reflection](Enumerate_enum_values_and_names_at_compile_time_using_reflection.md)
- [Enumerate struct members at compile time using reflection](Enumerate_struct_members_at_compile_time_using_reflection.md)
- [Enumerate struct members at compile time using stdmetamembers of](Enumerate_struct_members_at_compile_time_using_stdmetamembers_of.md)
- [Generate code from reflections using template splicing ](Generate_code_from_reflections_using_template_splicing_.md)
- [Generate switch dispatch tables from enum reflections](Generate_switch_dispatch_tables_from_enum_reflections.md)
- [Implement compile-time ORM and query generation via reflection](Implement_compile-time_ORM_and_query_generation_via_reflection.md)
- [Implement compile-time struct serialization using reflection](Implement_compile-time_struct_serialization_using_reflection.md)
- [Understand static reflection with the operator and stdmeta](Understand_static_reflection_with_the_operator_and_stdmeta.md)
- [Understand the C++26 static reflection model with the operator](Understand_the_C++26_static_reflection_model_with_the_operator.md)
- [Understand the C++26 static reflection model with the splice operator](Understand_the_C++26_static_reflection_model_with_the_splice_operator.md)
- [Understand the difference between compile-time and runtime reflection in C++](Understand_the_difference_between_compile-time_and_runtime_reflection_in_C++.md)
- [Understand the limits of C++26 reflection what is and is not reflectable](Understand_the_limits_of_C++26_reflection_what_is_and_is_not_reflectable.md)
- [Understand the splice operator for injecting reflections back into code](Understand_the_splice_operator_for_injecting_reflections_back_into_code.md)
- [Understand token injection and code injection proposals](Understand_token_injection_and_code_injection_proposals.md)
- [Use reflection for automatic debug printing and logging](Use_reflection_for_automatic_debug_printing_and_logging.md)
- [Use reflection for compile-time interface checking beyond concepts](Use_reflection_for_compile-time_interface_checking_beyond_concepts.md)
- [Use reflection for enum-to-string conversion without macros](Use_reflection_for_enum-to-string_conversion_without_macros.md)
- [Use reflection to generate comparison and hash operators automatically](Use_reflection_to_generate_comparison_and_hash_operators_automatically.md)
- [Use reflection to implement automatic JSON serialization](Use_reflection_to_implement_automatic_JSON_serialization.md)
- [Use reflection to implement automatic enum-to-string conversion](Use_reflection_to_implement_automatic_enum-to-string_conversion.md)
- [Use reflection to implement automatic struct serialization](Use_reflection_to_implement_automatic_struct_serialization.md)
- [Use stdmetadefine class to synthesize new types at compile time](Use_stdmetadefine_class_to_synthesize_new_types_at_compile_time.md)

## Notes

- C++26 static reflection (P2996) provides compile-time introspection of types and declarations
- ^^T (the reflection operator) produces a meta-object representing type T
- Splicing ([:refl:]) converts a meta-object back into code — the inverse of reflection
- Reflection enables automatic serialization, ORM, and enum-to-string without macros
- std::meta::members_of enumerates class members at compile time
- Template-for loops iterate over reflected members — replacing recursive metaprogramming
- Reflection works within consteval functions — all introspection happens at compile time
- Annotations (proposed) may allow attaching metadata to declarations for reflection consumption
- Reflection obsoletes many uses of macros, X-macros, and code generation tools
- Library support (std::meta namespace) provides building blocks for reflection algorithms
