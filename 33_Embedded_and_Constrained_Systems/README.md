# Embedded & Constrained Systems

C++ development for bare-metal, RTOS, and resource-constrained environments: freestanding implementations, no-heap strategies, ISR patterns, MISRA/AUTOSAR compliance, and hardware abstractions.

**Topics:** 18

## Contents

- [Achieve deterministic execution and WCET analysis](Achieve_deterministic_execution_and_WCET_analysis.md)
- [Analyze stack usage and enforce static memory budgets](Analyze_stack_usage_and_enforce_static_memory_budgets.md)
- [Design bootloader and firmware update architecture in Cpp](Design_bootloader_and_firmware_update_architecture_in_Cpp.md)
- [Design hardware register abstractions with volatile and MMIO](Design_hardware_register_abstractions_with_volatile_and_MMIO.md)
- [Design interrupt-safe C++ patterns for ISR contexts](Design_interrupt-safe_C++_patterns_for_ISR_contexts.md)
- [Develop with fno-exceptions and fno-rtti constraints](Develop_with_fno-exceptions_and_fno-rtti_constraints.md)
- [Handle real-time scheduling and priority inversion in C++](Handle_real-time_scheduling_and_priority_inversion_in_C++.md)
- [Implement fixed-point arithmetic without floating point](Implement_fixed-point_arithmetic_without_floating_point.md)
- [Implement power management and low-power modes from Cpp](Implement_power_management_and_low-power_modes_from_Cpp.md)
- [Know ELF and hex binary layout and linker script customization](Know_ELF_and_hex_binary_layout_and_linker_script_customization.md)
- [Know MISRA C++ and AUTOSAR C++ coding standards](Know_MISRA_C++_and_AUTOSAR_C++_coding_standards.md)
- [Understand Cpp exception table overhead and when to use fno-exceptions](Understand_Cpp_exception_table_overhead_and_when_to_use_fno-exceptions.md)
- [Understand freestanding vs hosted C++ implementations](Understand_freestanding_vs_hosted_C++_implementations.md)
- [Use C++ on ARM Cortex-M and RISC-V microcontrollers](Use_C++_on_ARM_Cortex-M_and_RISC-V_microcontrollers.md)
- [Use Cpp with Zephyr RTOS and FreeRTOS](Use_Cpp_with_Zephyr_RTOS_and_FreeRTOS.md)
- [Use static allocation strategies without heap](Use_static_allocation_strategies_without_heap.md)
- [Use stdstdinplace vector Cpp26 for heap-free containers in embedded](Use_stdstdinplace_vector_Cpp26_for_heap-free_containers_in_embedded.md)
- [Write bare-metal startup code and linker scripts for C++](Write_bare-metal_startup_code_and_linker_scripts_for_C++.md)

## Notes

- Embedded C++ often prohibits dynamic allocation - use `std::array`, stack buffers, and placement new instead of `new`/`delete`.
- `-fno-exceptions` and `-fno-rtti` reduce binary size and overhead significantly in constrained environments.
- `volatile`-qualified peripheral registers must be accessed through `volatile` pointers - but `volatile` is not atomic and does not replace synchronization.
- Real-time systems require deterministic timing - avoid allocations, exceptions, and unbounded loops in time-critical paths.
- Interrupt-safe programming needs `volatile` and atomic operations - regular variables may be optimized away by the compiler across an ISR boundary.
- Static analysis and MISRA C++ compliance are common requirements in safety-critical embedded development.
- `constexpr` computing moves work to compile time - ideal for lookup tables, CRC generation, and configuration constants.
- Bit manipulation and register maps benefit from strongly-typed wrapper classes that prevent accidental register misuse.
- Memory-mapped I/O requires specific alignment and access size - use `reinterpret_cast` carefully and only where justified by a documented deviation.
- Cross-compilation toolchains (ARM GCC, IAR, Keil) have different standard library support - always verify which headers are available in your freestanding environment.
