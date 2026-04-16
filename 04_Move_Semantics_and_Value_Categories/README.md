# Move Semantics & Value Categories

Rvalue references, std::move, std::forward, perfect forwarding, copy elision, and value category rules.

**Topics:** 22

## Contents

- [Know ref-qualified member functions and overloading on this value category](Know_ref-qualified_member_functions_and_overloading_on_this_value_category.md)
- [Know that stdinitializer list always copies and never moves elements](Know_that_stdinitializer_list_always_copies_and_never_moves_elements.md)
- [Know the stdstdrelocate proposals and trivial relocatability Cpp26 direction](Know_the_stdstdrelocate_proposals_and_trivial_relocatability_Cpp26_direction.md)
- [Know when return stdmovex is harmful and prevents NRVO](Know_when_return_stdmovex_is_harmful_and_prevents_NRVO.md)
- [Know when to mark functions noexcept and understand its performance implications](Know_when_to_mark_functions_noexcept_and_understand_its_performance_implications.md)
- [Master perfect forwarding with stdforward and universal references](Master_perfect_forwarding_with_stdforward_and_universal_references.md)
- [Understand conditional noexcept and its impact on stdvector reallocation](Understand_conditional_noexcept_and_its_impact_on_stdvector_reallocation.md)
- [Understand deducing-this interaction with move semantics Cpp23](Understand_deducing-this_interaction_with_move_semantics_Cpp23.md)
- [Understand forwarding references vs rvalue references syntactically](Understand_forwarding_references_vs_rvalue_references_syntactically.md)
- [Understand guaranteed copy elision prvalue materialization C++17](Understand_guaranteed_copy_elision_prvalue_materialization_C++17.md)
- [Understand how move semantics interact with virtual functions](Understand_how_move_semantics_interact_with_virtual_functions.md)
- [Understand implicit move from return statements C++23 clarification](Understand_implicit_move_from_return_statements_C++23_clarification.md)
- [Understand moved-from state guarantees in the standard library](Understand_moved-from_state_guarantees_in_the_standard_library.md)
- [Understand sink parameters and by-value vs const-reference trade-offs](Understand_sink_parameters_and_by-value_vs_const-reference_trade-offs.md)
- [Understand sink parameters when to take by value and when by rvalue reference](Understand_sink_parameters_when_to_take_by_value_and_when_by_rvalue_reference.md)
- [Understand stdmove and why it does not actually move anything](Understand_stdmove_and_why_it_does_not_actually_move_anything.md)
- [Understand stdstdswap vs memberwise swap performance trade-offs](Understand_stdstdswap_vs_memberwise_swap_performance_trade-offs.md)
- [Understand the five value categories lvalue rvalue xvalue prvalue glvalue](Understand_the_five_value_categories_lvalue_rvalue_xvalue_prvalue_glvalue.md)
- [Understand trivially relocatable types and memcpy-based relocation](Understand_trivially_relocatable_types_and_memcpy-based_relocation.md)
- [Use stdforward like C++23 for forwarding with ownership propagation](Use_stdforward_like_C++23_for_forwarding_with_ownership_propagation.md)
- [Use stdmove only function C++23 for non-copyable callable wrappers](Use_stdmove_only_function_C++23_for_non-copyable_callable_wrappers.md)
- [Write correct move constructors and move assignment operators](Write_correct_move_constructors_and_move_assignment_operators.md)

## Notes

- std::move does not move — it casts to an rvalue reference, enabling move constructors
- Moved-from objects must be in a valid but unspecified state — never assume their value
- std::forward preserves value category in perfect forwarding — use only with forwarding references
- Value categories (lvalue, xvalue, prvalue) determine which overload gets called
- The copy-and-swap idiom provides strong exception guarantee for assignment operators
- Guaranteed copy elision (C++17) means prvalues are never materialized unless needed
- Return by value — compilers apply NRVO/RVO, and move semantics handles the rest
- 
oexcept on move constructors is critical — containers like std::vector require it for strong exception safety
- Don't std::move in return statements with local variables — it prevents NRVO
- Rvalue reference members in classes are almost always a design mistake
