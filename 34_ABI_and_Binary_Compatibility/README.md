# ABI & Binary Compatibility

Understanding and managing Application Binary Interface stability: Itanium/MSVC ABI layout, symbol visibility, shared library versioning, ABI-stable interface design, and binary compatibility across compiler versions.

**Topics:** 10

## Contents

- [Control symbol visibility for shared libraries](Control_symbol_visibility_for_shared_libraries.md)
- [Design ABI-stable C++ library interfaces](Design_ABI-stable_C++_library_interfaces.md)
- [Implement SO and DLL versioning strategies](Implement_SO_and_DLL_versioning_strategies.md)
- [Know MSVC ABI and class layout with reportSingleClassLayout](Know_MSVC_ABI_and_class_layout_with_reportSingleClassLayout.md)
- [Understand abi_tag attribute for symbol versioning](Understand_abi_tag_attribute_for_symbol_versioning.md)
- [Understand COM ABI implications on Windows](Understand_COM_ABI_implications_on_Windows.md)
- [Understand Itanium ABI layout rules for vtables RTTI and exceptions](Understand_Itanium_ABI_layout_rules_for_vtables_RTTI_and_exceptions.md)
- [Understand what changes break ABI in C++ libraries](Understand_what_changes_break_ABI_in_C++_libraries.md)
- [Use extern C wrappers for stable FFI boundaries](Use_extern_C_wrappers_for_stable_FFI_boundaries.md)
- [Use inline namespace versioning for ABI evolution](Use_inline_namespace_versioning_for_ABI_evolution.md)

## Notes

- ABI defines how code interacts at binary level: calling conventions, vtable layout, name mangling
- The Itanium ABI is used by GCC and Clang on most platforms; MSVC has its own ABI
- Changing virtual function order, adding virtual functions, or modifying class layout breaks ABI
- Inline namespaces allow ABI versioning — std::string changed ABI between libstdc++ versions
- Pimpl idiom preserves ABI — adding private members doesn't change class size visible to users
- extern "C" functions have stable ABI — use them for plugin interfaces and FFI boundaries
- std::function and std::any provide ABI-stable type erasure for library boundaries
- Symbol visibility (__attribute__((visibility("default")))) controls what's exported from shared libraries
- Mixing different C++ standard library versions (libstdc++ vs libc++) in one process is dangerous
- Use -fvisibility=hidden and explicit exports to minimize ABI surface area
