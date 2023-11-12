[![CI](../../actions/workflows/ci.yml/badge.svg)](../../actions/workflows/ci.yml)

# Changes
```
[8] 2023-11-12 eh2k:
    * removed hardcoded compile flag '-fno-inline', added '-fno-rtti' for cpp files
[8] 2023-11-10 eh2k:
    * udynlink_cpp_init - support C++ global constructors, __init_array etc.
[7] 2023-11-08 eh2k:
    * mkmodule --public-symbols, write only public symbols to bin
[6] 2023-11-08 eh2k:
    * mkmodule --bin-name arg (--name can be used separately)
    * udynlink_get_module_name2 (get module name from base_addr)
[5] 2023-11-06 eh2k:
    * refactoring udynlink_load_module + tests
    * udynlink_error_msg added
    * mkmodule --public-symbols arg (do not wrapp all global functions)
    * platformio library.json
    * import in platformio.ini eg: lib_deps = https://github.com/????/udynlink.git
[4] 2023-11-02 eh2k:
    * O3 opt 
    * (string table issue) R_ARM_ABS32 data relocation
    * extra script/tests output
[3] 2023-11-01 eh2k:
    * no module reusing (multiple instances of one module alowed)
    * replaced udynlink_get_lot_base by a "fixed location" 
        * set *(0x20000000) = p_mod->ram_base, before function call
[2] 2023-11-01 eh2k:
    * Compile C++ code without errors (-fno-exceptions) + hello-world test
    * removed -fno-section-anchors
    * support --build_flags args (e.g. --build_flags=-I.)
[1] 2023-11-01 eh2k:
    * python3 migration (based on https://github.com/bogdanm/udynlink/pull/7) 
    * cycle.yml to gh_actions migration 
    * updating to latest qemu-system-gnuarmeclipse
```

# Overview

`udynlink` is a dynamic linker for ARM Cortex-M MCUs. It compiles code into a binary blob that can be loaded and executed at runtime on the MCU. The binary blob can be either loaded into RAM or executed directly from the flash memory (execute in place). The dynamic linker can be used for various purposes, such as:

- Use code that can run only from RAM (for example, code that manipulates the internal MCU flash memory, like a bootloader).
- Speed up the execution of some of your code by running it from RAM instead of Flash.
- Apply partial patches to your application (replace a part of the code by dynamically loading a newer version of that code).
- Implement loaders for modules written in C for higher-level languages (like Lua, Python or JavaScript).
- Since it it a dynamic loader, you can **probably** take advantage of the LGPL licensing terms. Keep in mind though that I am not a lawyer and I haven't verified this with a lawyer, so you'll likely want to double-check this.

# Status

- The code is very much work in progress, and likely quite buggy. Consider it to be pre-alpha quality.
- The code can only be compiled with [GCC ARM Embedded](https://launchpad.net/gcc-arm-embedded).
- The code was only tested for C code, not C++. Although it could work for some C++ code, assume that it'll fail for most C++ code.

# License

The dynamic linker code is licensed under the Apache 2.0 license.
The test suite uses the [GNU MCU Eclipse](https://gnu-mcu-eclipse.github.io/) project, which in turn uses code from the [micro-os-plus-iii](https://github.com/micro-os-plus/micro-os-plus-iii), which is licensed under the MIT license.

# How it works

In short, `udynlink` uses the compiler to generate code that can run from any address (position-independent code) and have its data anywhere in memory (position-independent data), then transforms this code into a loadable module by appending various information to it.

## Position-independent code and data

Let's consider this very simple code:

```
volatile int i = 1;

int main() {
    return i;
}
```

The code generated by the compiler when passed some commonly used options (`arm-none-eabi-gcc -O0 -c -mcpu=cortex-m4 -mthumb`) looks like this:

```
 1 00000000 <main>:
 2   0: b480        push  {r7}
 3   2: af00        add r7, sp, #0
 4   4: 4b03        ldr r3, [pc, #12] ; (14 <main+0x14>)
 5   6: 681b        ldr r3, [r3, #0]
 6   8: 4618        mov r0, r3
 7   a: 46bd        mov sp, r7
 8   c: f85d 7b04   ldr.w r7, [sp], #4
 9  10: 4770        bx  lr
10  12: bf00        nop
11  14: 00000000  .word 0x00000000
```
Line 4 reads the address of variable `i` using PC-relative addressing from address 0x14 (line 11) and line 5 reads the actual value of the variable. When linking this code, the linker will change the value in line 11 (which resides in the code section) to point to the actual address of the variable `i` in memory. This code can't easily be loaded and executed at runtime, since the address of `i` is fixed and it might conflict with the runtime memory map of the MCU. To fix this issue, the address of `i` (and thus the value at line 11 above) needs to be changed at runtime. Changing values in the code section is not difficult if the code section is located in RAM, but it's generally more difficult to do if the code section is located in flash.

Now let's compile the same code using the same options used by `udynlink` (`arm-none-eabi-gcc -O0 -c -mcpu=cortex-m4 -mthumb -msingle-pic-base -fno-inline -fPIE a.c -mno-pic-data-is-text-relative`):

```
 1 00000000 <main>:
 2   0: b480        push  {r7}
 3   2: af00        add r7, sp, #0
 4   4: 4b04        ldr r3, [pc, #16] ; (18 <main+0x18>)
 5   6: f859 3003   ldr.w r3, [r9, r3]
 6   a: 681b        ldr r3, [r3, #0]
 7   c: 4618        mov r0, r3
 8   e: 46bd        mov sp, r7
 9  10: f85d 7b04   ldr.w r7, [sp], #4
10  14: 4770        bx  lr
11  16: bf00        nop
12  18: 00000000  .word 0x00000000
```

Line 4 still reads a value using PC-relative addressing. However, that value is used in line 5 as an index into another table (which base is kept in `r9`) to get the actual address of `i`, which is then read in line 6. By changing the value in the table pointed to by `r9`, the variable `i` can be relocated in memory without changing any value in the code section.

This is a very simple example. In practice, there are other types of relocations (for both data and code) that need to be handled. But the basic idea is still the same: compile the code in such a way that it can run from anywhere in memory (RAM or flash), and have its data reside anywhere in RAM.

# Compiling a module image

To compile C code into a loadable module image, use the `scripts/mkmodule` script. The script runs a number of distinct steps:

1. It compiles the C source(s) in position-independent code and data mode (see the previous section for details). In this steps, it also generates wrappers for exported functions (see below for details).
2. It links the resulting code using a special linker script.
3. It reads the ELF file generated in step 2 above, looking for symbols and relocations.
4. It generates the loadable image. The image starts with a header and continues with the .text and .data sections. This image can be loaded using the dynamic linker.

## Step 1: compile source files

The `mkmodule` script receives a number of C source file names as arguments, and compiles all of them to object files. The compilation flags are chosen to generate a position-independent code and data object file. When compiling a source file, there's another step involved: each public (non-static) function is wrapped in code that loads the correct value of the `r9` register. Remember from the previous section that the addresses of the variables in the code are read from a table which base is kept in the `r9` register. Since each module has its own memory region, the value of `r9` is different for each module, and needs to be loaded before running any code that needs to access `r9`. The code that loads `r9` looks like this:

```
push    {r9, lr}
push    {r0, r1}
mov     r1, #0x1c
ldr     r1, [r1]
mov     r0, pc
blx     r1
mov     r9, r0
pop     {r0, r1}
bl      {{actname}}
pop     {r9, pc}
```

The code calls a function that receives the current value of the PC register and returns the value of `r9` for the code running at this address. The address of this function is kept in a fixed location in memory (`0x1c`). After setting the value of `r9`, the code branches to the original function.

> **Note** 
> 0x1c replaced by 0x20000000 (e.g. linkerscript: KEEP(*(.udynlink_lot_base)))
> * set *(0x20000000) = p_mod->ram_base, before function call
> ```
>     push    {r9, lr}
>     push    {r1}
>     ldr     r1, .L1{{s}}
>     ldr     r9, [r1]
>     pop     {r1}
>     bl      {{actname}}
>     pop     {r9, pc}
> .L1{{s}}:
>     .word   0x20000000
>     .size   {{s}}, . - {{s}}
> ```

## Step 2: link

The object files compiled in step 1 are linked using a special linker script (`scripts/code_before_data.ld`). The linker script defines a single memory area that starts at address 0 and contains the .text, .data and .bss sections (in this order). The code is linked using a special flag (`--unresolved-symbols=ignore-in-object-files`) that prevents the linker from exiting with an error when it doesn't find a symbol that needs to be linked. These symbols will be resolved when the dynamic linker loads the module (see below for details).

## Step 3: read symbols and relocations

In this step, `mkmodule` reads the .text, .data and .bss section generated in the previous step, building a list with the symbols found in the ELF file. It also reads the relocation section in the ELF file and processes each relocation in turn.

## Step 4: generate the module image

The binary image of the loadable module is built in this step. The image begins with a header that contains various information about the module, including:

- Exported symbols: these are the public symbols in your module's code. Symbols are both functions and non-static global variables.
- Foreign symbols: these are symbols needed by the module to run. Specifically, these are the symbols that were not found when linking the module ELF, but ignored because of the `--unresolved-symbols` linker flag (explained above).
- List of relocations that need to be applied when loading the module.

The module's .text and .data sections follow the header.

The picture below shows the memory layout of the module image:

```
+------------+ Offset 0
+ Header     +
+------------+
+ .text      +
+------------+
+ .data      +
+------------+
```

To load the module, a pointer to this module image needs to be passed to the dynamic linker running on the MCU. Note that the memory map of the module **after** it is loaded is different (see below for details).

# The dynamic linker

The dynamic linker is the code running on the MCU that's responsible with loading the modules created by `mkmodule`. Its interface can be found in `udynlink/udynlink.h`. To load a module, you need to call `udynlink_load_module` with the image of the module and a load mode:

- `UDYNLINK_LOAD_MODE_COPY_ALL`: the whole module image (header, text and data) is copied into RAM.
- `UDYNLINK_LOAD_MODE_COPY_CODE`: the .text and .data sections are copied into RAM, without the header. Note that the dynamic linker needs access to the module's header even after the module is loaded, so the header needs to remain accessible.
- `UDYNLINK_LOAD_MODE_XIP`: only .data is copied into RAM. The same observations related to the header apply.

Note that a module generally needs more RAM than the memory required by the load mode above. In particular, "execute in place" (`UDYNLINK_LOAD_MODE_XIP`) isn't the same as "no RAM required", it just means that the actual code runs directly from the module's image, without being copied anywhere. Even in XIP mode, the module likely needs RAM for its .data and .bss sections; even if it those sections are empty, the module likely needs RAM for its relocations. Modules that don't require any RAM at all to work can exist, but are quite rare.

Speaking of relocations, the dynamic linker uses an array called `LOT` (Linker Offset Table) that keeps a list of the relocations that need to be applied to the module's image in RAM (this is similar in concept with the usual GOT mechanism, but different in implementation, hence the different name). The LOT occupies the first region of the module's image in RAM.  The LOT is the table to which `r9` must point to when executing code in this module.

Besides applying relocations, the linker needs to resolve the module's foreign symbols. These are the symbols that are needed for the module to run, but were not found during linking. A simple example:

```
#include <stdio.h>

void greet(void) {
    printf("Hello, world!\n");
}
```

When compiling this code as module, `mkmodule` won't be able to find the definition of the `printf` function (since we're not linking with the C library), and `printf` will eventually make it into the module's foreign (unresolved) symbols list. When loading the module, the dynamic linker will see that `printf` is a foreign symbol and will attempt to resolve this by calling the `udynlink_external_resolve_symbol` function with the name of the function (`printf`). The function returns either the address of the `printf` function, or NULL if `printf` is not present in the firmware running on the MCU. If the return result is not NULL, the symbol is resolved and the module is ready to be used. Note that `udynlink_external_resolve_symbol` is free to return either the address of a symbol in the static on-chip MCU firmware, or the address of a symbol in another module. This, in turn, makes it possible to implement hierarchies of modules that depend on each other.

After the module is loaded, you can use the `udynlink_lookup_symbol` function to get the (relocated) value of a symbol. As an example, suppose that you compiled the `greet` function above into a module that you loaded using `udynlink_load_module`. To actually run the function, you need to do this:

```
udynlink_sym_t sym; // symbol structure
udynlink_lookup_symbol(p_mod, "greet", &sym); // p_mod is a handle to the module (returned by udynlink_load_module)
void (*p_greet)(void) = (void (*)(void))sym.val; // convert the symbol value to the proper function type
p_greet(); // call the function
```

Alternatively, you can use `udynlink_get_symbol_value` to get the value of the symbol directly:

```
void (*p_greet)(void) = (void (*)(void))udynlink_get_symbol_value(p_mod, "greet");
p_greet();
```

