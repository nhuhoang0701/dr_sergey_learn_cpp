# Know ELF and hex binary layout and linker script customization

**Category:** Embedded & Constrained Systems  
**Standard:** C++17  
**Reference:** <https://sourceware.org/binutils/docs/ld/Scripts.html>  

---

## Topic Overview

### Why Linker Scripts Matter for Embedded C++

On a desktop, the OS loads your program into virtual memory and you never think about addresses. On bare-metal embedded systems, **you** control exactly where every byte of code and data lives in physical flash and RAM. The linker script defines:

- Where code (`.text`) lives in flash
- Where initialized data (`.data`) lives in RAM (but is stored in flash and copied at startup)
- Where zeroed data (`.bss`) lives in RAM
- The stack location and size
- Interrupt vector table placement
- Special sections for DMA buffers, backup RAM, etc.

### ELF File Structure

The compiler produces **ELF** (Executable and Linkable Format) files. Key sections:

| Section | Contents | Resides in |
| --- | --- | --- |
| `.text` | Machine code (instructions) | Flash |
| `.rodata` | `const` data, string literals, vtables | Flash |
| `.data` | Initialized global/static variables | RAM (initialized from flash copy) |
| `.bss` | Zero-initialized global/static variables | RAM (zeroed at startup) |
| `.init_array` | C++ global constructor pointers | Flash |
| `.fini_array` | C++ global destructor pointers | Flash |
| `.ARM.exidx` | Exception unwind tables (if exceptions enabled) | Flash |

### A Complete Linker Script for Cortex-M

```ld

/* STM32F411 — 512 KB flash, 128 KB RAM */
MEMORY
{
    FLASH (rx)  : ORIGIN = 0x08000000, LENGTH = 512K
    RAM   (rwx) : ORIGIN = 0x20000000, LENGTH = 128K
}

/* Entry point — the Reset_Handler from startup code */
ENTRY(Reset_Handler)

/* Minimum stack and heap sizes — linker error if they don't fit */
_Min_Stack_Size = 0x1000;  /* 4 KB */
_Min_Heap_Size  = 0x0;     /* No heap in most embedded */

SECTIONS
{
    /* Interrupt vector table — must be at flash start */
    .isr_vector : {
        . = ALIGN(4);
        KEEP(*(.isr_vector))
        . = ALIGN(4);
    } >FLASH

    /* Code */
    .text : {
        . = ALIGN(4);
        *(.text)
        *(.text*)
        *(.glue_7)         /* ARM/Thumb interworking */
        *(.glue_7t)

        KEEP(*(.init))
        KEEP(*(.fini))

        . = ALIGN(4);
        _etext = .;
    } >FLASH

    /* Read-only data */
    .rodata : {
        . = ALIGN(4);
        *(.rodata)
        *(.rodata*)
        . = ALIGN(4);
    } >FLASH

    /* C++ global constructors */
    .init_array : {
        . = ALIGN(4);
        PROVIDE_HIDDEN(__init_array_start = .);
        KEEP(*(SORT(.init_array.*)))
        KEEP(*(.init_array))
        PROVIDE_HIDDEN(__init_array_end = .);
        . = ALIGN(4);
    } >FLASH

    /* C++ global destructors (rarely used in embedded) */
    .fini_array : {
        . = ALIGN(4);
        PROVIDE_HIDDEN(__fini_array_start = .);
        KEEP(*(SORT(.fini_array.*)))
        KEEP(*(.fini_array))
        PROVIDE_HIDDEN(__fini_array_end = .);
        . = ALIGN(4);
    } >FLASH

    /* ARM exception unwind tables */
    .ARM.exidx : {
        __exidx_start = .;
        *(.ARM.exidx*)
        __exidx_end = .;
    } >FLASH

    /* Used by startup code to initialize .data */
    _sidata = LOADADDR(.data);

    /* Initialized data — stored in flash, copied to RAM at startup */
    .data : {
        . = ALIGN(4);
        _sdata = .;
        *(.data)
        *(.data*)
        . = ALIGN(4);
        _edata = .;
    } >RAM AT> FLASH

    /* Zero-initialized data */
    .bss : {
        . = ALIGN(4);
        _sbss = .;
        __bss_start__ = _sbss;
        *(.bss)
        *(.bss*)
        *(COMMON)
        . = ALIGN(4);
        _ebss = .;
        __bss_end__ = _ebss;
    } >RAM

    /* Heap grows up from end of BSS */
    ._heap : {
        . = ALIGN(8);
        PROVIDE(end = .);
        PROVIDE(_end = .);
        . = . + _Min_Heap_Size;
        . = ALIGN(8);
    } >RAM

    /* Stack grows down from top of RAM */
    ._stack : {
        . = ALIGN(8);
        . = . + _Min_Stack_Size;
        . = ALIGN(8);
        _estack = .;
    } >RAM

    /* Verify everything fits */
    ASSERT((_estack <= ORIGIN(RAM) + LENGTH(RAM)), "RAM overflow!")
}

```

### Startup Code — Initializing C++ from Bare Metal

The startup code (`Reset_Handler`) must initialize `.data`, `.bss`, and call C++ global constructors before `main()`:

```cpp

extern "C" {

// Symbols defined in linker script
extern uint32_t _sidata;  // Source of .data in flash
extern uint32_t _sdata;   // Start of .data in RAM
extern uint32_t _edata;   // End of .data in RAM
extern uint32_t _sbss;    // Start of .bss
extern uint32_t _ebss;    // End of .bss
extern uint32_t _estack;  // Initial stack pointer

// C++ constructor table
extern void (*__init_array_start[])();
extern void (*__init_array_end[])();

void Reset_Handler() {
    // 1. Copy .data from flash to RAM
    uint32_t* src = &_sidata;
    uint32_t* dst = &_sdata;
    while (dst < &_edata) {
        *dst++ = *src++;
    }

    // 2. Zero .bss
    dst = &_sbss;
    while (dst < &_ebss) {
        *dst++ = 0;
    }

    // 3. Call C++ global constructors
    for (auto fn = __init_array_start; fn < __init_array_end; ++fn) {
        (*fn)();
    }

    // 4. Call main
    main();

    // 5. Should never return — halt if it does
    while (true) { __WFI(); }
}

} // extern "C"

```

### Custom Sections for DMA and Special Memory

```cpp

// Place a buffer in a specific RAM region for DMA
__attribute__((section(".dma_buffer")))
alignas(32) uint8_t dma_rx_buffer[1024];

// Corresponding linker script addition:
// .dma_buffer (NOLOAD) : {
//     . = ALIGN(32);
//     *(.dma_buffer)
// } >RAM

```

### Inspecting the Binary

```bash

# Show section sizes
arm-none-eabi-size -A firmware.elf

# Show section layout
arm-none-eabi-objdump -h firmware.elf

# Disassemble with source
arm-none-eabi-objdump -d -S firmware.elf

# Generate binary for flashing
arm-none-eabi-objcopy -O binary firmware.elf firmware.bin

# Generate Intel HEX
arm-none-eabi-objcopy -O ihex firmware.elf firmware.hex

# Show symbol table sorted by size
arm-none-eabi-nm --size-sort -S firmware.elf

```

### Intel HEX Format

The `.hex` file is a text representation of binary data with addresses. Each line:

```cpp

:LLAAAATT[DD...]CC

```

- `LL` = byte count
- `AAAA` = 16-bit address
- `TT` = record type (00=data, 01=EOF, 04=extended address)
- `DD` = data bytes
- `CC` = checksum

---

## Self-Assessment

### Q1: Why does `.data` appear in both flash and RAM in the linker script

Initialized globals like `int x = 42;` must have their initial values stored somewhere non-volatile (flash), but at runtime they must be writable (RAM). The linker script specifies:

- **VMA (Virtual Memory Address)**: `>RAM` — where the CPU accesses `x` at runtime
- **LMA (Load Memory Address)**: `AT> FLASH` — where the initial values are stored

The startup code (`Reset_Handler`) copies from LMA to VMA before `main()` runs. The symbols `_sidata`, `_sdata`, `_edata` delimit the source and destination of this copy.

### Q2: What happens if you forget to call `__init_array` constructors

Global C++ objects (static variables with constructors) will **not be initialized**. Their memory in `.bss` will be zero or contain whatever `.data` provides, but the constructor side effects — opening files, configuring peripherals, registering callbacks — will not execute. This leads to subtle bugs where code "works" on the debugger (which may auto-initialize) but fails on the real target.

### Q3: How do you verify that your firmware fits in flash

```bash

# Quick size summary
arm-none-eabi-size firmware.elf
#   text    data     bss     dec     hex filename
#  48320    1024    8192   57536    e100 firmware.elf
# text + data = flash usage (49344 bytes)
# data + bss  = RAM usage (9216 bytes)

```

The linker script `ASSERT` validates at link time. You can also add CI checks:

```bash

# Fail if flash usage exceeds 480 KB (leaving 32 KB for bootloader)
max_flash=491520
actual=$(arm-none-eabi-size -B firmware.elf | tail -1 | awk '{print $1+$2}')
[ "$actual" -le "$max_flash" ] || exit 1

```

---

## Notes

- Use `KEEP()` for sections the linker might discard (vector table, constructor arrays)
- `NOLOAD` sections occupy no flash space — useful for DMA buffers and stack
- Use `ALIGN(4)` for ARM, `ALIGN(2)` for 16-bit architectures
- Enable `-Wl,--print-memory-usage` in GCC to see flash/RAM usage at link time
- Use `arm-none-eabi-nm --size-sort` to find the largest symbols when optimizing size
