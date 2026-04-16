# Error Handling

Exceptions, error codes, std::expected, noexcept, and error handling best practices.

**Topics:** 26

## Contents

- [Design error domains for large-scale multi-library projects](Design_error_domains_for_large-scale_multi-library_projects.md)
- [Design error hierarchies with error domains for large codebases](Design_error_hierarchies_with_error_domains_for_large_codebases.md)
- [Design structured error propagation across library boundaries](Design_structured_error_propagation_across_library_boundaries.md)
- [Handle errors across coroutine boundaries correctly](Handle_errors_across_coroutine_boundaries_correctly.md)
- [Know stdstdexpected vs Boost.Outcome vs Boost.LEAF comparison](Know_stdstdexpected_vs_Boost.Outcome_vs_Boost.LEAF_comparison.md)
- [Know the difference between noexcept operator and noexcept specifier](Know_the_difference_between_noexcept_operator_and_noexcept_specifier.md)
- [Know when to use exceptions vs error codes vs stdexpected](Know_when_to_use_exceptions_vs_error_codes_vs_stdexpected.md)
- [Nest exceptions with stdthrow with nested for diagnostic chains](Nest_exceptions_with_stdthrow_with_nested_for_diagnostic_chains.md)
- [Understand Cpp26 Contracts in depth assertion levels build modes continuation](Understand_Cpp26_Contracts_in_depth_assertion_levels_build_modes_continuation.md)
- [Understand and use stdterminate stdabort and terminate handlers](Understand_and_use_stdterminate_stdabort_and_terminate_handlers.md)
- [Understand contract programming and assertion-based design](Understand_contract_programming_and_assertion-based_design.md)
- [Understand exception safety in move-only types](Understand_exception_safety_in_move-only_types.md)
- [Understand stderror condition vs stderror code and when to use each](Understand_stderror_condition_vs_stderror_code_and_when_to_use_each.md)
- [Understand the Herb Sutter deterministic exceptions proposal and its alternative](Understand_the_Herb_Sutter_deterministic_exceptions_proposal_and_its_alternative.md)
- [Understand the logic error vs runtime error hierarchy](Understand_the_logic_error_vs_runtime_error_hierarchy.md)
- [Use contracts C++26 preview and preconditionpostcondition annotations](Use_contracts_C++26_preview_and_preconditionpostcondition_annotations.md)
- [Use custom exception classes in a hierarchy with stdexception base](Use_custom_exception_classes_in_a_hierarchy_with_stdexception_base.md)
- [Use nested exceptions with stdnested exception for exception chaining](Use_nested_exceptions_with_stdnested_exception_for_exception_chaining.md)
- [Use stderror code and stderror category for system error handling](Use_stderror_code_and_stderror_category_for_system_error_handling.md)
- [Use stdexception ptr for cross-thread exception propagation](Use_stdexception_ptr_for_cross-thread_exception_propagation.md)
- [Use stdexpected monadic operations transform and then or else](Use_stdexpected_monadic_operations_transform_and_then_or_else.md)
- [Use stdsystem error and its integration with stderror code](Use_stdsystem_error_and_its_integration_with_stderror_code.md)
- [Use stdterminate handler and stdunexpected handler correctly](Use_stdterminate_handler_and_stdunexpected_handler_correctly.md)
- [Use trycatch at the right granularity not too fine not too coarse](Use_trycatch_at_the_right_granularity_not_too_fine_not_too_coarse.md)
- [Write correct exception-safe code using RAII and strongbasic guarantees](Write_correct_exception-safe_code_using_RAII_and_strongbasic_guarantees.md)
- [Write noexcept exception specifications correctly and propagate them](Write_noexcept_exception_specifications_correctly_and_propagate_them.md)

## Notes

- std::expected (C++23) is the modern way to return errors without exceptions
- Exception safety levels: no-throw, strong, basic, none — aim for at least basic guarantee
- 
oexcept on functions enables compiler optimizations and is required for move operations in containers
- Don't throw in destructors — it causes std::terminate if another exception is in flight
- std::error_code / std::error_category provide system-compatible error handling
- Catch exceptions by const reference (const std::exception&) to avoid slicing
- std::nested_exception allows wrapping lower-level errors while preserving original context
- Prefer returning error codes in performance-critical paths — exceptions have non-zero cost on the throw path
- Use static_assert and concepts to catch errors at compile time instead of runtime
- Function-try-blocks on constructors are the only way to catch exceptions from member initializer lists
