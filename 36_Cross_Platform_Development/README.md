# Cross-Platform Development

Writing portable C++ across operating systems and compilers: platform abstraction layers, endianness handling, filesystem differences, Unicode, SIMD portability, feature detection, and compiler extension management.

**Topics:** 14

## Contents

- [Abstract platform-specific atomic and memory model differences](Abstract_platform-specific_atomic_and_memory_model_differences.md)
- [Design CI matrices for cross-platform testing Windows Linux macOS ARM](Design_CI_matrices_for_cross-platform_testing_Windows_Linux_macOS_ARM.md)
- [Design platform abstraction layers in C++](Design_platform_abstraction_layers_in_C++.md)
- [Detect and adapt to library availability differences](Detect_and_adapt_to_library_availability_differences.md)
- [Handle DLL and SO loading differences LoadLibrary vs dlopen](Handle_DLL_and_SO_loading_differences_LoadLibrary_vs_dlopen.md)
- [Handle Unicode and text encoding across platforms](Handle_Unicode_and_text_encoding_across_platforms.md)
- [Handle endianness detection and byte swapping portably](Handle_endianness_detection_and_byte_swapping_portably.md)
- [Handle wchar t size differences between Windows and Linux](Handle_wchar_t_size_differences_between_Windows_and_Linux.md)
- [Manage filesystem path differences across Windows and POSIX](Manage_filesystem_path_differences_across_Windows_and_POSIX.md)
- [Understand threading model differences and stdthread portability](Understand_threading_model_differences_and_stdthread_portability.md)
- [Use CMake toolchain files and presets for multi-platform builds](Use_CMake_toolchain_files_and_presets_for_multi-platform_builds.md)
- [Use compiler-specific extensions safely with abstraction macros](Use_compiler-specific_extensions_safely_with_abstraction_macros.md)
- [Use conditional compilation and feature detection macros](Use_conditional_compilation_and_feature_detection_macros.md)
- [Write portable SIMD code with Highway and stdsimd](Write_portable_SIMD_code_with_Highway_and_stdsimd.md)

## Notes

- Test on all target platforms in CI - behavior varies across compilers and standard libraries.
- Use `#ifdef _WIN32`, `__linux__`, `__APPLE__` for platform detection, or prefer CMake feature tests when possible.
- Filesystem paths differ (`/` vs `\\`) - use `std::filesystem::path` for portable path handling instead of raw strings.
- Windows uses UTF-16 (`wchar_t`); Unix uses UTF-8 - normalize to UTF-8 internally and convert only at the boundary.
- Socket APIs differ - use Asio or a platform-abstraction library for networking rather than wrapping them yourself.
- Endianness matters for binary protocols - use `std::byteswap` (C++23) or a manual conversion helper.
- Dynamic linking differs: `.so` (Linux), `.dylib` (macOS), `.dll` (Windows) - abstract the difference behind CMake targets.
- Align on standard C++ features rather than compiler extensions for maximum portability across toolchains.
- Use `__has_include` and `__has_cpp_attribute` for feature detection at compile time instead of guessing by compiler version.
- CMake presets (`CMakePresets.json`) standardize cross-platform build configurations and make them shareable with the team.
