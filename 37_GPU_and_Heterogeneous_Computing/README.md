# GPU & Heterogeneous Computing

C++ for GPU and accelerator programming: CUDA, SYCL/oneAPI, OpenCL, std::execution GPU backends, data transfer optimization, unified memory, GPU-friendly data structures, and offloading with OpenMP.

**Topics:** 13

## Contents

- [Bridge stdexecution with GPU backends](Bridge_stdexecution_with_GPU_backends.md)
- [Design GPU-friendly data structures for coalesced access](Design_GPU-friendly_data_structures_for_coalesced_access.md)
- [Implement CPU-GPU task graphs for heterogeneous pipelines](Implement_CPU-GPU_task_graphs_for_heterogeneous_pipelines.md)
- [Integrate CUDA with modern C++ for GPU kernel development](Integrate_CUDA_with_modern_C++_for_GPU_kernel_development.md)
- [Offload computation with OpenMP target directives](Offload_computation_with_OpenMP_target_directives.md)
- [Optimize host-device data transfer patterns](Optimize_host-device_data_transfer_patterns.md)
- [Profile GPU kernels with NSight and ROCProfiler](Profile_GPU_kernels_with_NSight_and_ROCProfiler.md)
- [Understand GPU memory hierarchy registers shared memory L1 L2 global](Understand_GPU_memory_hierarchy_registers_shared_memory_L1_L2_global.md)
- [Use CUDA Cooperative Groups for flexible thread synchronization](Use_CUDA_Cooperative_Groups_for_flexible_thread_synchronization.md)
- [Use OpenCL C++ bindings for vendor-neutral GPU programming](Use_OpenCL_C++_bindings_for_vendor-neutral_GPU_programming.md)
- [Use SYCL and oneAPI for portable heterogeneous computing](Use_SYCL_and_oneAPI_for_portable_heterogeneous_computing.md)
- [Use Vulkan Compute for portable GPU computing from Cpp](Use_Vulkan_Compute_for_portable_GPU_computing_from_Cpp.md)
- [Use unified shared memory for simplified GPU programming](Use_unified_shared_memory_for_simplified_GPU_programming.md)

## Notes

- CUDA extends C++ for GPU programming — kernel functions run on thousands of GPU threads simultaneously
- SYCL provides standard C++ heterogeneous computing — single-source GPU code without language extensions
- GPU memory management (device vs host) is the primary complexity — use unified memory for prototyping
- Occupancy optimization (balancing threads, registers, shared memory) is key to GPU performance
- std::execution (C++26) aims to unify CPU and GPU execution under one programming model
- GPU workloads suit data-parallel operations — matrix math, image processing, simulations
- Memory coalescing (adjacent threads accessing adjacent memory) is critical for GPU throughput
- Shared memory is fast but limited (~48KB per block) — use it for data reuse within thread blocks
- Vulkan Compute and OpenCL are cross-vendor GPU compute APIs — less C++-friendly than CUDA/SYCL
- Profile GPU code with NSight (NVIDIA) or RGP (AMD) — bottlenecks differ from CPU code
