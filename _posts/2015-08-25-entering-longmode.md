---
layout: post
title: 'Entering Long Mode'
category: 'rust-os'
---
In the [previous post] we created a minimal multiboot kernel. It just prints `OK` and hangs. Let's extend it! The goal is to call 64-bit [Rust] code. But the CPU is currently in [Protected Mode] and allows only 32-bit instructions and up to 4GiB memory. So we need to setup _Paging_ and switch to the 64-bit [Long Mode] first.

I tried to explain everything in detail and to keep the code as simple as possible. If you have any questions, suggestions or other issues, please leave a comment or [create an issue] on Github. The source code is available in a [repository][source code], too.

[previous post]: {{ site.url }}{{ page.previous.url }}
[Rust]: http://www.rust-lang.org/
[Real Mode]: http://wiki.osdev.org/Real_Mode
[Protected Mode]: https://en.wikipedia.org/wiki/Protected_mode
[Long Mode]: https://en.wikipedia.org/wiki/Long_mode
[create an issue]: https://github.com/phil-opp/phil-opp.github.io/issues
[source code]: https://github.com/phil-opp/blogOS/tree/entering_longmode

## Some Checks
To avoid bugs and strange errors on old CPUs we should test if the processor supports every needed feature. Let's start by writing an error function, that displays `ERR: ` and a given [ASCII] character. We add it at the end of our boot.asm:

```nasm
...
; Prints `ERR: ` and the given error code to screen and hangs.
; parameter: error code (in ascii) in al
error:
    mov dword [0xb8000], 0x4f524f45
    mov dword [0xb8004], 0x4f3a4f52
    mov dword [0xb8008], 0x4f204f20
    mov byte  [0xb800a], al
    hlt
```
At address `b8000` begins the so-called [VGA text buffer]. It's an array of screen characters that consists of a 8 byte color code and a 8 byte ASCII character. We used the color code `4f` which means white text on red background. `0x52` is an ASCII `R`, `0x45` is an `E`, `0x3a` is a `:`, and `0x20` is a space. The second space is overwritten by the given ASCII char, then the CPU is stopped.

Now we will add some check _functions_. A function is just a normal label with an `ret` (return) instruction at the end. The `call` instruction can be used to call it. Unlike the `jmp` instruction that just jumps to a memory address, the `call` instruction will push a return address to the stack (and the `ret` will jump to this address). But wait, we don't have a stack yet. The [stack pointer] in the esp register could point to some important data or even invalid memory. So we need to update it and point it to some valid stack memory. Let's create this memory by reserving some bytes at the end of our `boot.asm`:

```nasm
...
section .bss
stack_bottom:
    resb 64
stack_top:
```
A stack doesn't need to be initialized with data because we will `pop` only if we `pushed` before. By using the [.bss] section and the `resb` (reserve byte) command, we just declare the length of the uninitialized data (64 byte) and avoid storing these bytes in the executable. When loading the executable, GRUB will create the section and the stack in memory. To use this stack, we update the stack pointer register right after `start`:

```nasm
global start

section .text
bits 32
start:
    mov esp, stack_top

    ; print `OK` to screen
    ...
```
We use `stack_top` because the stack grows downwards: A `push eax` subtracts 4 from `esp` and does a `mov [esp], eax` afterwards (`eax` is a general purpose register). Now we have a valid stack pointer and are able to call functions.

The following check functions are just here for completeness and I won't explain details. Basically they all work the same: They will check for a feature and jump to `error` if it's not available.

[ASCII]: https://en.wikipedia.org/wiki/ASCII
[VGA text buffer]: https://en.wikipedia.org/wiki/VGA-compatible_text_mode
[stack pointer]: http://stackoverflow.com/a/1464052/866447
[.bss]: https://en.wikipedia.org/wiki/.bss

### Multiboot check
We rely on some Multiboot features in the next posts. So let's make sure the kernel was really loaded by a Multiboot compliant bootloader: according to the [specification] \(PDF), the `eax` register must contain the magic value `0x36d76289` after loading. Let's add a simple function that checks this:

```nasm
check_multiboot:
    cmp eax, 0x36d76289
    jne .no_multiboot
    ret
.no_multiboot:
    mov al, "0"
    jmp error
```
We compare the value in `eax` with the magic value and jump to the label `no_multiboot` if they're not equal (`jne` – “jump if not equal”). In `no_multiboot`, we jump to the error function with the error code `0`.

[specification]: http://nongnu.askapache.com/grub/phcoder/multiboot.pdf

### CPUID check
[CPUID] is a CPU instruction that can be used to get various information about the CPU. But not every processor supports it. Let's steal a detection function from the [OSDev wiki][CPUID detection]:

```nasm
check_cpuid:
    pushfd               ; Store the FLAGS-register.
    pop eax              ; Restore the A-register.
    mov ecx, eax         ; Set the C-register to the A-register.
    xor eax, 1 << 21     ; Flip the ID-bit, which is bit 21.
    push eax             ; Store the A-register.
    popfd                ; Restore the FLAGS-register.
    pushfd               ; Store the FLAGS-register.
    pop eax              ; Restore the A-register.
    push ecx             ; Store the C-register.
    popfd                ; Restore the FLAGS-register.
    xor eax, ecx         ; Do a XOR-operation on the A-register and the C-register.
    jz .no_cpuid         ; The zero flag is set, no CPUID.
    ret                  ; CPUID is available for use.
.no_cpuid:
    mov al, "1"
    jmp error
```

### Long Mode check
Now we can use CPUID to check whether Long Mode can be used. I will use code from [OSDev][Long Mode detection] again:

```nasm
check_long_mode:
    mov eax, 0x80000000    ; Set the A-register to 0x80000000.
    cpuid                  ; CPU identification.
    cmp eax, 0x80000001    ; Compare the A-register with 0x80000001.
    jb .no_long_mode       ; It is less, there is no long mode.
    mov eax, 0x80000000    ; Set the A-register to 0x80000000.
    cpuid                  ; CPU identification.
    cmp eax, 0x80000001    ; Compare the A-register with 0x80000001.
    jb .no_long_mode       ; It is less, there is no long mode.
    ret
.no_long_mode:
    mov al, "2"
    jmp error
```

### Putting it together
Now we just call these check functions right after start:

```nasm
global _start

section .text
bits 32
_start:
    mov esp, stack_top

    call check_multiboot
    call check_cpuid
    call check_long_mode

    ; print `OK` to screen
    ...
```
When the CPU doesn't support a needed feature, we get an error message with an unique error code. Now we can start the real work.

[CPUID]: http://wiki.osdev.org/CPUID
[CPUID detection]: http://wiki.osdev.org/Setting_Up_Long_Mode#Detection_of_CPUID
[Long Mode detection]: http://wiki.osdev.org/Setting_Up_Long_Mode#x86_or_x86-64

## Paging
_Paging_ is a memory management scheme that separates virtual and physical memory. The address space is split into equal sized _pages_ and a _page table_ specifies which virtual page points to which physical page. If you never heard of paging, you might want to look at the [paging introduction][paging chapter] (PDF) of the [Three Easy Pieces] OS book.

In Long Mode, x86 uses a page size of 4096 bytes and a 4 level page table that consists of:

- the Page-Map Level-4 Table (PML4),
- the Page-Directory Pointer Table (PDP),
- the Page-Directory Table (PD),
- and the Page Table (PT).

As I don't like these names, I will call them P4, P3, P2, and P1 from now on.

Each page table contains 512 entries and one entry is 8 bytes, so they fit exactly in one page (`512*8 = 4096`). To translate a virtual address to a physical address the CPU[^hardware_lookup] will do the following[^virtual_physical_translation_source]:

![translation of virtual to physical addresses in 64 bit mode]({{ site.url }}/images/X86_Paging_64bit.svg)

1. Get the address of the P4 table from the CR3 register
2. Use bits 39-47 (9 bits) as an index into P4 (`2^9 = 512 = number of entries`)
3. Use the following 9 bits as an index into P3
4. Use the following 9 bits as an index into P2
5. Use the following 9 bits as an index into P1
6. Use the last 12 bits as page offset (`2^12 = 4096 = page size`)

But what happens to bits 48-63 of the 64-bit virtual address? Well, they can't be used. The “64-bit” Long Mode is in fact just a 48-bit mode. The bits 48-63 must be copies of bit 47, so each valid virtual address is still unique. For more information see [Wikipedia][wikipedia_48bit_mode].

An entry in the P4, P3, P2, and P1 tables consists of the page aligned 52-bit _physical_ address of the page/next page table and the following bits that can be OR-ed in:

Bit(s)                | Name | Meaning
--------------------- | ------ | ----------------------------------
0 | present | the page is currently in memory
1 | writable | it's allowed to write to this page
2 | user accessible | if not set, only kernel mode code can access this page
3 | write through caching | writes go directly to memory
4 | disable cache | no cache is used for this page
5 | accessed | the CPU sets this bit when this page is used
6 | dirty | the CPU sets this bit when a write to this page occurs
7 | huge page/null | must be 0 in P1 and P4, creates a 1GiB page in P3, creates a 2MiB page in P2
8 | global | page isn't flushed from caches on address space switch (PGE bit of CR4 register must be set)
9-11 | available | can be used freely by the OS
52-62 | available | can be used freely by the OS
63 | no execute | forbid executing code on this page (the NXE bit in the EFER register must be set)

[Three Easy Pieces]: http://pages.cs.wisc.edu/~remzi/OSTEP/
[paging chapter]: http://pages.cs.wisc.edu/~remzi/OSTEP/vm-paging.pdf
[wikipedia_48bit_mode]: https://en.wikipedia.org/wiki/X86-64#Virtual_address_space_details

[^hardware_lookup]: In the x86 architecture, the page tables are _hardware walked_, so the CPU will look at the table on its own when it needs a translation. Other architectures, for example MIPS, just throw an exception and let the OS translate the virtual address.

[^virtual_physical_translation_source]: Image source: [Wikipedia](https://commons.wikimedia.org/wiki/File:X86_Paging_64bit.svg), with modified font size, page table naming, and removed sign extended bits. The modified file is licensed under the Creative Commons Attribution-Share Alike 3.0 Unported license.

### Setup Identity Paging
When we switch to Long Mode, paging will be activated automatically. The CPU will then try to read the instruction at the following address, but this address is now a virtual address. So we need to do _identity mapping_, i.e. map a physical address to the same virtual address.

The `huge page` bit is now very useful to us. It creates a 2MiB (when used in P2) or even a 1GiB page (when used in P3). So we just need to create one P4 and one P3 table to identity map the first _gigabytes_ of our kernel. Of course we will replace them with finer-grained tables later. But now that we're stuck with assembly, we choose the easiest way. Let's add these two tables at the beginning[^page_table_alignment] of the `.bss` section:

```nasm
...

section .bss
align 4096
p4_table:
    resb 4096
p3_table:
    resb 4096
stack_bottom:
    resb 64
stack_top:
```
The `resb` command reserves the specified amount of bytes without initializing them, so the 8KiB don't need to be saved in the executable. The `align 4096` ensures that the page tables are page aligned. When GRUB creates the `.bss` section in memory, it will initialize it to `0`. So our `p4_table` is already valid (it contains 512 non-present entries) but not very useful. Let's link its first entry to the `p3_table` and map the first P3 entry to a huge page:

```nasm
setup_page_tables:
    ; map first P4 entry to P3 table
    mov eax, p3_table
    or eax, 0b11 ; present + writable
    mov [p4_table], eax

    ; map first P3 entry to a huge page that starts at address 0
    mov dword [p3_table], 0b10000011 ; present + writable + huge

    ret
```
We just set the present and writable bits (`0b11` is a binary number) in the aligned P3 table address and move it to the first 4 bytes of the P4 table. To complete the identity mapping, we then map the first P3 entry to a huge 1GiB page that starts at address 0. Now the first gigabyte of our kernel is accessible through the same physical and virtual addresses.

[^page_table_alignment]: Page tables need to be page-aligned as the bits 0-11 are used for flags. By putting these tables at the beginning of `.bss`, the linker can just page align the whole section and we don't have unused padding bytes in between.

### Enable Paging
To enable paging and enter Long Mode, we need to do the following:

1. write the address of the P4 table to the CR3 register (the CPU will look there, see the [paging section](#paging))
2. Long Mode is an extension of [Physical Address Extension], so we need to enable it first
3. Set the Long Mode bit in the EFER register
4. Enable Paging

The assembly function looks like this (some boring bit moving to various registers):

```nasm
enable_paging:
    ; load P4 to cr3 register (cpu uses this to access the P4 table)
    mov eax, p4_table
    mov cr3, eax

    ; enable PAE-flag in cr4 (Physical Address Extension)
    mov eax, cr4
    or eax, 1 << 5
    mov cr4, eax

    ; set the long mode bit in the EFER MSR (model specific register)
    mov ecx, 0xC0000080
    rdmsr
    or eax, 1 << 8
    wrmsr

    ; enable paging in the cr0 register
    mov eax, cr0
    or eax, 1 << 31
    or eax, 1 << 16
    mov cr0, eax

    ret
```
The `or eax, 1 << X` is a pattern to set bit `X` in the eax register (`<<` is a left shift). Through `rdmsr` and `wrmsr` it's possible to read/write to the so-called model specific registers at address `ecx`.

Let's call our new functions in `start`:

```nasm
...
start:
    mov esp, stack_top

    call check_multiboot
    call check_cpuid
    call check_long_mode

    call setup_page_tables ; new
    call enable_paging     ; new

    ; print `OK` to screen
    mov dword [0xb8000], 0x2f4b2f4f
    hlt
...
```
To test it we run `make run`. If the green OK is printed, we have successfully enabled paging!

## The Global Descriptor Table
After enabling Paging, we are now in Long Mode. So we can use 64-bit instructions now, right? Wrong. We are still in some 32-bit compatibility submode. To actually execute 64-bit code, we need to setup a new Global Descriptor Table.
The Global Descriptor Table (GDT) was used for _Segmentation_ in ancient operating systems. I won't explain Segmentation but the [Three Easy Pieces] OS book has good [introduction][Segmentation] (PDF) again.

Today almost everyone uses Paging instead of Segmentation (and so do we). But on x86, a GDT is always required, even when you're not using Segmentation. GRUB has set up a valid 32-bit GDT for us but now we need to switch to a long mode GDT.

A GDT always starts with an 0-entry and contains a arbitrary number of segment entries afterwards. An entry has the following format:

Bit(s)                | Name | Meaning
--------------------- | ------ | ----------------------------------
0-15 | limit 0-15 | the first 2 byte of the segment's limit
16-39 | base 0-23 | the first 3 byte of the segment's base address
40 | accessed | set by the CPU when the segment is accessed
41 | read/write | reads allowed for code segments / writes allowed for data segments
42 | direction/conforming | the segment grows down (i.e. base>limit) for data segments / the current privilege level can be higher than the specified level for code segments (else it must match exactly)
43 | executable | if set, it's a code segment, else it's a data segment
44 | descriptor type | should be 1 for code and data segments
45-46 | privilege | the [ring level]: 0 for kernel, 3 for user
47 | present | must be 1 for valid selectors
48-51 | limit 16-19 | bits 16 to 19 of the segment's limit
52 | available | freely available to the OS
53 | 64-bit | should be set for 64-bit code segments
54 | 32-bit | should be set for 32-bit segments
55 | granularity | if it's set, the limit is the number of pages, else it's a byte number
56-63 | base 24-31 | the last byte of the base address

We need one code and one data segment. They have the following bits set: _descriptor type_, _present_, and _read/write_. The code segment has additionally the _executable_ and the _64-bit_ flag. In Long mode, it's not possible to use Segmentation and the base and limit fields must be 0. Translated to assembly it looks like this:

```nasm
section .rodata
gdt64:
    dq 0 ; zero entry
    dq (1<<44) | (1<<47) | (1<<41) | (1<<43) | (1<<53) ; code segment
    dq (1<<44) | (1<<47) | (1<<41) ; data segment
```
We chose the `.rodata` section here because it's initialized read-only data. The `(1<<44)` is a [bit shift] that sets bit 44.

[Three Easy Pieces]: http://pages.cs.wisc.edu/~remzi/OSTEP/
[Segmentation]: http://pages.cs.wisc.edu/~remzi/OSTEP/vm-segmentation.pdf
[ring level]: http://wiki.osdev.org/Security#Rings
[bit shift]: http://www.cs.umd.edu/class/sum2003/cmsc311/Notes/BitOp/bitshift.html

### Loading the GDT
To load our new GDT, we have to tell the CPU its address and length. We do this by passing the memory location of a special pointer structure to the `lgdt` (load GDT) instruction. Let's add this pointer structure at the end of our GDT:

```nasm
gdt64:
    ...
    dq (1<<44) | (1<<47) | (1<<41) ; data segment
.pointer:
    dw $ - gdt64 - 1
    dq gdt64
```
The first 2 bytes specify the (GDT length - 1). The `$` is a special symbol that is replaced with the current address (it's equal to `.pointer`). The following 8 bytes specify the GDT address. Labels that start with a point (like `.pointer`) are sub-labels of the last label without point. To access them, they must be prefixed with the parent label (like `gdt64.pointer`). Now we can load the GDT in start:

```nasm
start:
    ...
    call enable_paging

    ; load the 64-bit GDT
    lgdt [gdt64.pointer]

    ; print `OK` to screen
    ...
```
When you still see the green `OK`, everything went fine and the new GDT is loaded. But we still can't execute 64-bit code: The selector registers like the code selector `cs`, the data selector `ds`, the stack selector `ss`, and the extra selector `es` still have the values from the old GDT. To update them, we need to load them with the GDT offset (in bytes) of the desired segment. In our case the code segment starts at byte 8 of the GDT and the data segment at byte 16. Let's try it:

```nasm
    ...
    lgdt [gdt64.pointer]

    ; update selectors
    mov ax, 16
    mov ss, ax
    mov ds, ax
    mov es, ax

    ; print `OK` to screen
    ...
```
It should still work. The segment selectors are only 16-bits large, so we use the 16-bit ax subregister. Notice that we didn't update the code selector `cs`. We will do that later. First we should replace this hardcoded `16` by adding some labels to our GDT:

```nasm
section .rodata
gdt64:
    dq 0 ; zero entry
.code: equ $ - gdt64 ; new
    dq (1<<44) | (1<<47) | (1<<41) | (1<<43) | (1<<53) ; code segment
.data: equ $ - gdt64 ; new
    dq (1<<44) | (1<<47) | (1<<41) ; data segment
.pointer:
    ...
```
We can't just use normal labels here, as we need the table offset. We calculate this offset using the current address `$` and set the labels to this value using [equ]. Now we can use `gdt64.data` instead of 16 and `gdt64.code` instead of 8 and these labels will still work if we modify the GDT.

Now there is just one last step left to enter the true 64-bit mode: We need to load `cs` with `gdt64.code`. But we can't do it through `mov`. The only way to reload the code selector is a _far jump_ or a _far return_. These instructions work like a normal jump/return but change the code selector. We will use a far jump to a long mode label:

[equ]: http://www.nasm.us/doc/nasmdoc3.html#section-3.2.4

```nasm
    global start
    extern long_mode_start
    ...
    lgdt [gdt64.pointer]

    ; update selectors
    mov ax, gdt64.data
    mov ss, ax
    mov ds, ax
    mov es, ax

    jmp gdt64.code:long_mode_start
```
The actual `long_mode_start` label is defined as `extern`, so it's part of another file. The `jmp gdt64.code:long_mode_start` is the mentioned far jump.

I put the 64-bit code into a new file to separate it from the 32-bit code, thereby we can't call the (now invalid) 32-bit code accidentally. The new file (I named it `long_mode_init.asm`) looks like this:

```nasm
global long_mode_start

section .text
bits 64
long_mode_start:
    ; print `OKAY` to screen
    mov rax, 0x2f592f412f4b2f4f
    mov qword [0xb8000], rax
    hlt

bits 32
...
```
_Congratulations_! You have successfully wrestled through this CPU configuration and compatibility mode mess :). You should see a green `OKAY` on the screen. If you still have energy, here are some notes on this last step:

- As the CPU expects 64-bit instructions now, we use `bits 64`
- We can now use the extended registers. Instead of the 32-bit `eax`, `ebx`, etc. we now have the 64-bit `rax`, `rbx`, …
- and we can write these 64-bit registers directly to memory using `mov qword` (quad word)

## Whats next?
It's time to leave assembly behind[^leave_assembly_behind] and switch to some higher level language. We won't use any C or C++ (not even a single line). Instead we will use the relatively new [Rust] language. It's a systems language without garbage collections but with guaranteed memory safety. Through a real type system and many abstractions it feels like a high-level language but can still be low-level enough for OS development. The [next post] describes the Rust setup.

[Rust]: https://www.rust-lang.org/
[next post]: {{ site.url }}{{ page.next.url }}

[^leave_assembly_behind]: Actually we will still need some assembly in the future, but I'll try to minimize it.
