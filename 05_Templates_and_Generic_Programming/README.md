# Templates & Generic Programming

Function/class templates, SFINAE, variadic templates, template metaprogramming, and fold expressions.

**Topics:** 38

## Contents

- [Know the Curiously Recurring Template Pattern CRTP and its uses](Know_the_Curiously_Recurring_Template_Pattern_CRTP_and_its_uses.md)
- [Understand SFINAE and know when to replace it with Concepts](Understand_SFINAE_and_know_when_to_replace_it_with_Concepts.md)
- [Understand class template specialization for traits detection idiom](Understand_class_template_specialization_for_traits_detection_idiom.md)
- [Understand concept subsumption and overload ordering](Understand_concept_subsumption_and_overload_ordering.md)
- [Understand explicit instantiation and prevent implicit instantiation](Understand_explicit_instantiation_and_prevent_implicit_instantiation.md)
- [Understand how to detect whether a type is a specialization of a template](Understand_how_to_detect_whether_a_type_is_a_specialization_of_a_template.md)
- [Understand partial and full template specialization](Understand_partial_and_full_template_specialization.md)
- [Understand stdinitializer list interaction with template deduction](Understand_stdinitializer_list_interaction_with_template_deduction.md)
- [Understand substitution failure vs hard error in SFINAE](Understand_substitution_failure_vs_hard_error_in_SFINAE.md)
- [Understand template argument substitution order and its effect on errors](Understand_template_argument_substitution_order_and_its_effect_on_errors.md)
- [Understand template instantiation and its compile-time cost](Understand_template_instantiation_and_its_compile-time_cost.md)
- [Understand template specialization ordering and partial ordering rules](Understand_template_specialization_ordering_and_partial_ordering_rules.md)
- [Understand template template parameters](Understand_template_template_parameters.md)
- [Understand the interaction between templates and exceptions in stack unwinding](Understand_the_interaction_between_templates_and_exceptions_in_stack_unwinding.md)
- [Use abbreviated function templates with auto parameters C++20](Use_abbreviated_function_templates_with_auto_parameters_C++20.md)
- [Use concepts to constrain return types and not just parameters](Use_concepts_to_constrain_return_types_and_not_just_parameters.md)
- [Use fold expressions C++17 to operate on parameter packs](Use_fold_expressions_C++17_to_operate_on_parameter_packs.md)
- [Use non-type template parameters NTTPs including class types C++20](Use_non-type_template_parameters_NTTPs_including_class_types_C++20.md)
- [Use stdapply and stdmake from tuple for tuple-based generic invocation](Use_stdapply_and_stdmake_from_tuple_for_tuple-based_generic_invocation.md)
- [Use stdcommon type to deduce the common type of a set of types](Use_stdcommon_type_to_deduce_the_common_type_of_a_set_of_types.md)
- [Use stdconjunction stddisjunction and stdnegation for compound type predicates](Use_stdconjunction_stddisjunction_and_stdnegation_for_compound_type_predicates.md)
- [Use stddeclval to refer to type expressions without constructing objects](Use_stddeclval_to_refer_to_type_expressions_without_constructing_objects.md)
- [Use stdfunction ref C++26 as a lightweight non-owning callable](Use_stdfunction_ref_C++26_as_a_lightweight_non-owning_callable.md)
- [Use stdtuple as a type-level cons cell for compile-time type lists](Use_stdtuple_as_a_type-level_cons_cell_for_compile-time_type_lists.md)
- [Use stdtype traits to build a generic JSON serializer](Use_stdtype_traits_to_build_a_generic_JSON_serializer.md)
- [Use tag dispatch for selecting overloads based on type properties](Use_tag_dispatch_for_selecting_overloads_based_on_type_properties.md)
- [Use template specialization to customize standard library traits](Use_template_specialization_to_customize_standard_library_traits.md)
- [Use the hidden-friend idiom to improve overload resolution](Use_the_hidden-friend_idiom_to_improve_overload_resolution.md)
- [Use variadic templates and parameter packs correctly](Use_variadic_templates_and_parameter_packs_correctly.md)
- [Write a constexpr string view command dispatcher using template metaprogramming](Write_a_constexpr_string_view_command_dispatcher_using_template_metaprogramming.md)
- [Write a generic event bus using templates and type erasure](Write_a_generic_event_bus_using_templates_and_type_erasure.md)
- [Write a generic scope exit RAII guard using templates and lambdas](Write_a_generic_scope_exit_RAII_guard_using_templates_and_lambdas.md)
- [Write a type list and implement compile-time operations on it](Write_a_type_list_and_implement_compile-time_operations_on_it.md)
- [Write expression templates to defer computation and eliminate temporaries](Write_expression_templates_to_defer_computation_and_eliminate_temporaries.md)
- [Write expression templates to eliminate temporaries in arithmetic chains](Write_expression_templates_to_eliminate_temporaries_in_arithmetic_chains.md)
- [Write function templates with multiple type parameters](Write_function_templates_with_multiple_type_parameters.md)
- [Write recursive type lists and compile-time type manipulation](Write_recursive_type_lists_and_compile-time_type_manipulation.md)
- [Write type-erased wrappers using templates](Write_type-erased_wrappers_using_templates.md)

## Notes

- SFINAE (Substitution Failure Is Not An Error) enables compile-time overload selection
- Prefer C++20 concepts over SFINAE — they give clearer error messages and intent
- Template instantiation happens at use site — put definitions in headers (or use explicit instantiation)
- Variadic templates use parameter packs and fold expressions (C++17) for clean recursive patterns
- CRTP (Curiously Recurring Template Pattern) provides static polymorphism without virtual dispatch
- Type erasure combines templates and runtime polymorphism — see std::function, std::any
- Non-type template parameters (NTTP) can be floating-point and class types since C++20
- Template specialization (full and partial) customizes behavior for specific types
- Two-phase lookup: non-dependent names resolve at definition, dependent names at instantiation
- if constexpr (C++17) enables compile-time branching without SFINAE tricks
