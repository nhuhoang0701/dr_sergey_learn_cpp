# Memory & Ownership

Smart pointers (unique_ptr, shared_ptr), RAII, allocators, PMR, memory layout, and ownership semantics.

**Topics:** 36

## Contents

- [Handle exceptions safely in constructors that manage multiple resources](Handle_exceptions_safely_in_constructors_that_manage_multiple_resources.md)
- [Implement a memory pool with free-list recycling](Implement_a_memory_pool_with_free-list_recycling.md)
- [Know how to use stdallocator traits to write allocator-aware code](Know_how_to_use_stdallocator_traits_to_write_allocator-aware_code.md)
- [Know out-of-line allocation strategies for coroutine frames](Know_out-of-line_allocation_strategies_for_coroutine_frames.md)
- [Know the difference between operator new and operator new](Know_the_difference_between_operator_new_and_operator_new.md)
- [Know the rules of the Rule of Zero Rule of Three and Rule of Five](Know_the_rules_of_the_Rule_of_Zero_Rule_of_Three_and_Rule_of_Five.md)
- [Know when shared ptr is appropriate and understand its overhead](Know_when_shared_ptr_is_appropriate_and_understand_its_overhead.md)
- [Know when to use intrusive reference counting vs stdshared ptr](Know_when_to_use_intrusive_reference_counting_vs_stdshared_ptr.md)
- [Understand RAII and apply it to all resource types](Understand_RAII_and_apply_it_to_all_resource_types.md)
- [Understand aligned alloc and over-aligned types](Understand_aligned_alloc_and_over-aligned_types.md)
- [Understand aligned storage aligned union and their C++23 deprecation](Understand_aligned_storage_aligned_union_and_their_C++23_deprecation.md)
- [Understand dangling pointers vs dangling references UB in both cases](Understand_dangling_pointers_vs_dangling_references_UB_in_both_cases.md)
- [Understand how deleter types affect unique ptr size](Understand_how_deleter_types_affect_unique_ptr_size.md)
- [Understand implicit object creation and stdstdstart lifetime as Cpp20 and Cpp23](Understand_implicit_object_creation_and_stdstdstart_lifetime_as_Cpp20_and_Cpp23.md)
- [Understand memory resource interface and how to chain fallback allocators](Understand_memory_resource_interface_and_how_to_chain_fallback_allocators.md)
- [Understand placement new and its valid use cases](Understand_placement_new_and_its_valid_use_cases.md)
- [Understand stack vs heap allocation tradeoffs and prefer stack allocation](Understand_stack_vs_heap_allocation_tradeoffs_and_prefer_stack_allocation.md)
- [Understand stddestroy at vs explicit destructor call and their guarantees](Understand_stddestroy_at_vs_explicit_destructor_call_and_their_guarantees.md)
- [Understand stdpmrmonotonic buffer resource and arena allocation](Understand_stdpmrmonotonic_buffer_resource_and_arena_allocation.md)
- [Understand stdstdowner less for comparing weak ptr in ordered containers](Understand_stdstdowner_less_for_comparing_weak_ptr_in_ordered_containers.md)
- [Understand strict aliasing rules and their impact on optimization](Understand_strict_aliasing_rules_and_their_impact_on_optimization.md)
- [Understand the dangers of dangling pointers and how ownership semantics eliminat](Understand_the_dangers_of_dangling_pointers_and_how_ownership_semantics_eliminat.md)
- [Understand the operator newdelete overloading and class-specific allocators](Understand_the_operator_newdelete_overloading_and_class-specific_allocators.md)
- [Understand the small string optimization SSO in stdstring](Understand_the_small_string_optimization_SSO_in_stdstring.md)
- [Use make unique and make shared instead of raw new](Use_make_unique_and_make_shared_instead_of_raw_new.md)
- [Use stdassume aligned C++20 to inform the optimizer of pointer alignment](Use_stdassume_aligned_C++20_to_inform_the_optimizer_of_pointer_alignment.md)
- [Use stddestroy stdconstruct at and related low-level memory utilities C++1720](Use_stddestroy_stdconstruct_at_and_related_low-level_memory_utilities_C++1720.md)
- [Use stdout ptr and stdinout ptr C++23 for smart pointer interop with C APIs](Use_stdout_ptr_and_stdinout_ptr_C++23_for_smart_pointer_interop_with_C_APIs.md)
- [Use stdpmrstring and PMR containers to avoid allocator template proliferation](Use_stdpmrstring_and_PMR_containers_to_avoid_allocator_template_proliferation.md)
- [Use stdpmrunsynchronized pool resource for fast thread-local pool allocation](Use_stdpmrunsynchronized_pool_resource_for_fast_thread-local_pool_allocation.md)
- [Use stdspan C++20 to pass non-owning views to contiguous memory](Use_stdspan_C++20_to_pass_non-owning_views_to_contiguous_memory.md)
- [Use stdstart lifetime as C++23 for safe reinterpretation of byte arrays](Use_stdstart_lifetime_as_C++23_for_safe_reinterpretation_of_byte_arrays.md)
- [Use stdstdmake shared for overwrite and stdstdmake unique for overwrite Cpp20](Use_stdstdmake_shared_for_overwrite_and_stdstdmake_unique_for_overwrite_Cpp20.md)
- [Use stdunique ptr as the default ownership smart pointer](Use_stdunique_ptr_as_the_default_ownership_smart_pointer.md)
- [Use stdunique ptr with custom deleters for non-heap resources](Use_stdunique_ptr_with_custom_deleters_for_non-heap_resources.md)
- [Use stdweak ptr to break cyclic ownership](Use_stdweak_ptr_to_break_cyclic_ownership.md)

## Notes

- std::unique_ptr is zero-overhead — prefer it as the default ownership tool
- std::shared_ptr has atomic reference counting overhead — don't use it unless sharing is truly needed
- std::weak_ptr breaks cyclic references and enables safe observer patterns
- Custom deleters on unique_ptr change its type; on shared_ptr they don't (type erasure)
- Always use std::make_unique / std::make_shared — they are exception-safe and efficient
- Placement new constructs objects in pre-allocated memory — pair it with explicit destructor calls
- The allocator model in C++ separates allocation from construction — understand std::pmr for arena patterns
- Stack vs heap: stack is faster but limited; prefer stack allocation unless lifetime exceeds scope
- RAII (Resource Acquisition Is Initialization) is the core C++ idiom for resource management
- std::span (C++20) provides safe, non-owning views over contiguous memory
