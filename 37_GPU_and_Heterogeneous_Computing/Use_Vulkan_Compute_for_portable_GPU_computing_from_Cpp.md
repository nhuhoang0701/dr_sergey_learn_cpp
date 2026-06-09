# Use Vulkan Compute for portable GPU computing from C++

**Category:** GPU and Heterogeneous Computing  
**Standard:** C++17  
**Reference:** <https://www.khronos.org/vulkan/>  

---

## Topic Overview

### Why Vulkan Compute

Vulkan Compute provides several advantages over CUDA - but the honest answer is that it comes at a significant verbosity cost, so it's worth being clear about when that trade-off is worth making.

- **Vendor-neutral** - runs on NVIDIA, AMD, Intel, Qualcomm, Apple (via MoltenVK)
- **No proprietary compiler** - uses SPIR-V intermediate representation
- **Same API as graphics** - share resources between compute and rendering pipelines in the same application
- **Fine-grained control** - explicit memory management, synchronization, and queue submission

The tradeoff is real: Vulkan is significantly more verbose than CUDA. A CUDA vector addition is about 10 lines. The equivalent Vulkan code is around 200 lines of boilerplate. You pay that cost upfront to gain portability and graphics integration.

### Architecture Overview

Before writing any code, it helps to understand how all the Vulkan objects relate to each other. The hierarchy is strictly nested - each object depends on the ones above it:

```cpp
C++ Application
 └── Vulkan Instance
      └── Physical Device (GPU)
           └── Logical Device
                ├── Command Pool -> Command Buffer
                ├── Descriptor Set Layout -> Pipeline Layout -> Compute Pipeline
                ├── Shader Module (SPIR-V bytecode)
                └── Buffers (device memory)
```

You always work top-down: create the instance, pick a physical device, create a logical device, then create the resources (pipelines, buffers, command buffers) that you need for your computation.

### Writing a Compute Shader (GLSL -> SPIR-V)

Vulkan shaders are written in GLSL (or HLSL), then compiled to SPIR-V binary offline. The SPIR-V binary is what your application loads at runtime - there's no runtime shader compilation in the style of OpenCL. This is a deliberate design choice that eliminates startup latency and makes shader behavior fully deterministic across drivers.

```glsl
// vector_add.comp -- compiled to SPIR-V with glslangValidator
#version 450

layout(local_size_x = 256) in;  // Workgroup size

layout(set = 0, binding = 0) buffer InputA  { float a[]; };
layout(set = 0, binding = 1) buffer InputB  { float b[]; };
layout(set = 0, binding = 2) buffer Output  { float result[]; };

layout(push_constant) uniform PushConstants {
    uint count;
};

void main() {
    uint idx = gl_GlobalInvocationID.x;
    if (idx < count) {
        result[idx] = a[idx] + b[idx];
    }
}
```

Compile this shader to SPIR-V before running your application:

```bash
glslangValidator -V vector_add.comp -o vector_add.spv
```

The `.spv` file is a binary intermediate representation. Your C++ application will read this file and load it into a `VkShaderModule`.

### C++ Vulkan Compute Setup (with vulkan.hpp)

The `vulkan.hpp` C++ bindings give you RAII wrappers (`vk::Unique*` handles) and type-safe flags, making the code substantially less error-prone than the raw C API. Here's the initialization and buffer creation:

```cpp
#include <vulkan/vulkan.hpp>
#include <fstream>
#include <vector>

class VulkanCompute {
    vk::UniqueInstance instance_;
    vk::PhysicalDevice physical_device_;
    vk::UniqueDevice device_;
    vk::Queue compute_queue_;
    uint32_t queue_family_index_ = 0;

public:
    void init() {
        // 1. Create instance
        vk::ApplicationInfo app_info("ComputeApp", 1, "Engine", 1, VK_API_VERSION_1_2);
        vk::InstanceCreateInfo inst_info({}, &app_info);
        instance_ = vk::createInstanceUnique(inst_info);

        // 2. Select physical device with compute capability
        auto devices = instance_->enumeratePhysicalDevices();
        physical_device_ = devices[0];  // In production, score and select

        // 3. Find compute queue family
        auto queue_props = physical_device_.getQueueFamilyProperties();
        for (uint32_t i = 0; i < queue_props.size(); ++i) {
            if (queue_props[i].queueFlags & vk::QueueFlagBits::eCompute) {
                queue_family_index_ = i;
                break;
            }
        }

        // 4. Create logical device
        float priority = 1.0f;
        vk::DeviceQueueCreateInfo queue_info({}, queue_family_index_, 1, &priority);
        vk::DeviceCreateInfo dev_info({}, queue_info);
        device_ = physical_device_.createDeviceUnique(dev_info);
        compute_queue_ = device_->getQueue(queue_family_index_, 0);
    }

    // Create a GPU buffer
    struct GpuBuffer {
        vk::UniqueBuffer buffer;
        vk::UniqueDeviceMemory memory;
        vk::DeviceSize size;
    };

    GpuBuffer create_buffer(vk::DeviceSize size, vk::BufferUsageFlags usage) {
        GpuBuffer buf;
        buf.size = size;

        // Create buffer
        vk::BufferCreateInfo buf_info({}, size, usage);
        buf.buffer = device_->createBufferUnique(buf_info);

        // Allocate memory
        auto mem_reqs = device_->getBufferMemoryRequirements(*buf.buffer);
        auto mem_props = physical_device_.getMemoryProperties();

        uint32_t mem_type = 0;
        for (uint32_t i = 0; i < mem_props.memoryTypeCount; ++i) {
            if ((mem_reqs.memoryTypeBits & (1 << i)) &&
                (mem_props.memoryTypes[i].propertyFlags &
                 (vk::MemoryPropertyFlagBits::eHostVisible |
                  vk::MemoryPropertyFlagBits::eHostCoherent))) {
                mem_type = i;
                break;
            }
        }

        vk::MemoryAllocateInfo alloc_info(mem_reqs.size, mem_type);
        buf.memory = device_->allocateMemoryUnique(alloc_info);
        device_->bindBufferMemory(*buf.buffer, *buf.memory, 0);

        return buf;
    }

    // Map, write, unmap
    void upload(GpuBuffer& buf, const void* data, size_t size) {
        void* mapped = device_->mapMemory(*buf.memory, 0, size);
        std::memcpy(mapped, data, size);
        device_->unmapMemory(*buf.memory);
    }
};
```

The memory type search loop deserves attention. Vulkan exposes multiple memory heaps with different properties - you need to find a type that satisfies your buffer's alignment requirements (`memoryTypeBits`) AND has the properties you need (host-visible and host-coherent for CPU-accessible buffers). This is more explicit than CUDA's `cudaMalloc`, but it also gives you control over exactly what kind of memory you're getting.

### Running a Compute Dispatch

Once the pipeline is set up, dispatching compute work looks like this. The key concepts are descriptor sets (which bind your buffers to the shader's binding slots), push constants (small per-dispatch values passed directly to the shader), and command buffers (recorded lists of GPU commands):

```cpp
void dispatch_vector_add(VulkanCompute& ctx, size_t n) {
    // Load SPIR-V shader
    auto spv = read_file("vector_add.spv");
    vk::ShaderModuleCreateInfo shader_info({}, spv.size(),
                                            reinterpret_cast<const uint32_t*>(spv.data()));
    auto shader = ctx.device().createShaderModuleUnique(shader_info);

    // Create descriptor set layout (3 storage buffers)
    std::array bindings = {
        vk::DescriptorSetLayoutBinding(0, vk::DescriptorType::eStorageBuffer,
                                        1, vk::ShaderStageFlagBits::eCompute),
        vk::DescriptorSetLayoutBinding(1, vk::DescriptorType::eStorageBuffer,
                                        1, vk::ShaderStageFlagBits::eCompute),
        vk::DescriptorSetLayoutBinding(2, vk::DescriptorType::eStorageBuffer,
                                        1, vk::ShaderStageFlagBits::eCompute),
    };

    // Create compute pipeline
    vk::PushConstantRange push_range(vk::ShaderStageFlagBits::eCompute, 0, sizeof(uint32_t));
    // ... pipeline layout, descriptor pool, descriptor sets, command buffer ...

    // Dispatch: ceil(n / 256) workgroups
    uint32_t groups = (static_cast<uint32_t>(n) + 255) / 256;
    cmd.dispatch(groups, 1, 1);

    // Submit and wait
    ctx.compute_queue().submit(submit_info);
    ctx.device().waitIdle();
}
```

The `cmd.dispatch(groups, 1, 1)` call is the Vulkan equivalent of a CUDA `<<<blocks, threads>>>` launch. The first argument is the number of workgroups, and the shader's `local_size_x = 256` determines the threads per workgroup.

### Vulkan vs CUDA Comparison

Here's the honest trade-off summary. If you just need fast GPU code on NVIDIA hardware, CUDA wins on every practical metric except portability:

| Aspect | Vulkan Compute | CUDA |
| --- | --- | --- |
| Vendor support | All GPUs | NVIDIA only |
| Setup complexity | ~200 lines boilerplate | ~10 lines |
| Shader language | GLSL/HLSL -> SPIR-V | CUDA C++ |
| Memory management | Manual (allocate, bind, map) | `cudaMalloc`/`cudaMemcpy` |
| Synchronization | Explicit barriers, fences | Stream-based |
| Debug tools | RenderDoc, Vulkan Validation | NSight, cuda-memcheck |
| Best for | Portable compute, graphics+compute | Maximum NVIDIA perf |

---

## Self-Assessment

### Q1: Why choose Vulkan Compute over CUDA for a project

Choose Vulkan when you need **vendor independence** (AMD, Intel, mobile GPUs), when your application already uses Vulkan for graphics, or when targeting platforms where CUDA is unavailable (Android, macOS via MoltenVK). CUDA offers simpler APIs and better NVIDIA-specific performance, so choose CUDA if you only target NVIDIA GPUs.

### Q2: What is SPIR-V and why does Vulkan use it

SPIR-V (Standard Portable Intermediate Representation) is a binary intermediate format for GPU shaders. Vulkan uses it instead of text-based shaders because:

- **Portable** - the same SPIR-V binary runs on any Vulkan implementation
- **Pre-compiled** - no shader compilation at application startup
- **Language-agnostic** - can be generated from GLSL, HLSL, or custom languages
- **Optimizable** - the driver can still optimize the SPIR-V for specific hardware

### Q3: What is the local_size_x in compute shaders

`local_size_x` defines the number of threads in a **workgroup** (similar to a CUDA thread block). A `dispatch(N, 1, 1)` call launches `N` workgroups, each with `local_size_x` threads. Typical values are 64, 128, or 256. The optimal size depends on GPU architecture - 256 is a safe default for most GPUs.

---

## Notes

- Use `vulkan-hpp` (C++ bindings) instead of the raw C API for RAII and type safety - the raw C API requires manual handle cleanup and lacks the `vk::Unique*` lifetime management.
- RenderDoc can capture and debug Vulkan compute dispatches, letting you inspect buffer contents and replay individual dispatches.
- Enable Vulkan Validation Layers during development to catch API misuse - they are comprehensive and catch errors that would otherwise produce silent corruption or crashes.
- Consider Kompute (<https://github.com/KomputeFramework/kompute>) for a higher-level Vulkan compute wrapper that eliminates most of the boilerplate while keeping the portability benefits.
- For simple compute tasks, SYCL or OpenCL may be easier alternatives - reach for Vulkan when you specifically need graphics integration or the absolute minimum runtime overhead.
