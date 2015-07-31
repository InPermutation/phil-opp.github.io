---
layout: post
title: '[DRAFT] Rust OS Part 1: Booting'
related_posts: null
---
## Multiboot
Fortunately there is a bootloader standard: the [Multiboot Specification][multiboot].  So our kernel just needs to indicate that it supports Multiboot and every Multiboot-compliant bootloader can boot it. We will use the [GRUB 2] bootloader together with the Multiboot 2 specification ([PDF][Multiboot 2]).

To indicate our Multiboot 2 support to the bootloader, our kernel must contain a _Multiboot Header_, which has the following format:

Field         | Type            | Value
------------- | --------------- | ----------------------------------------
magic number  | u32             | 0xE85250D6
architecture  | u32             | 0 for i386, 4 for MIPS
header length | u32             | total header size, including tags
checksum      | u32             | -(magic + architecture + header length)
tags          | variable        |
end tag       | (u16, u16, u32) | (0, 0, 8)

Converted to a x86 assembly file it looks like this (Intel syntax):

```nasm
section .multiboot_header
header_start:
    dd 0xe85250d6                ; magic number (multiboot 2)
    dd 0                         ; architecture 0 (protected mode i386)
    dd header_end - header_start ; header length
    ; checksum
    dd 0x100000000 - (0xe85250d6 + 0 + (header_end - header_start))

    ; insert optional multiboot tags here

    ; required end tag
    dw 0    ; type
    dw 0    ; flags
    dd 8    ; size
header_end:
```
If you don't know x86 assembly, here is some quick guide:

- the header will be written to a section named `.multiboot_header` (we need this later)
- `header_start` and `header_end` are _labels_ that mark a memory location. We use them to calculate the header length easily
- `dd` stands for `define double` (32bit) and `dw` stands for `define word` (16bit)
- the additional `0x100000000` in the checksum calculation is a small hack[^fn-checksum_hack] to avoid a compiler warning

We can already _assemble_ this file (which I called `multiboot_header.asm`) using `nasm`. As it produces a flat binary by default, the resulting file just contains our 24 bytes (in little endian if you work on a x86 machine):

```
> nasm multiboot_header.asm
> hexdump -x multiboot_header
0000000    50d6    e852    0000    0000    0018    0000    af12    17ad
0000010    0000    0000    0008    0000
0000018
```

## The Boot Code //TODO rename
To boot our kernel, we must add some code that the bootloader can call. Let's create a file named `boot.asm`:

```nasm
global _start

BITS 32
section .text
_start:
  mov dword [0xb8000], 0x2f4b2f4f
  hlt
```
There are some new assembly lines:

- `global` exports a label (makes it public). As `_start` will be the entry point of our kernel, it needs to be public.
- `BITS 32` specifies that the following lines are 32-bit instructions. We don't need it in our multiboot header file, as it doesn't contain any runnable code.
- the `.text` section is the default section for executable code
- the `mov dword` instruction moves the 32bit constant `0x2f4f2f4b` to the memory at address `b8000` (it should write `ok` to the screen)
- `hlt` is the halt instruction and causes the CPU to stop

Through assembling, viewing and disassembling it we can see the CPU [Opcodes] in action:

```
> nasm boot.asm
> hexdump -x boot
0000000    05c7    8000    000b    2f4b    2f4f    00f4
000000b
> ndisasm -b 32 boot
00000000  C70500800B004B2F  mov dword [dword 0xb8000],0x2f4b2f4f
         -4F2F
0000000A  F4                hlt
```

## Building the Executable
Now we create an [ELF] executable from these two files. We therefore need the object files of the two assembly files and a custom linker script (`linker.ld`):

```
ENTRY(_start)

SECTIONS {
  . = 2M + SIZEOF_HEADERS;

  .boot :
  {
    /* ensure that the multiboot header is at the beginning */
    *(.multiboot_header)
  }

  .text :
  {
    *(.text)
  }
}
```
The important things are:

- `_start` is the entry point, the bootloader will jump to it after loading the kernel
- the executable will have two sections: `.boot` at the beginning and `.text` afterwards
- the `.text` output section contains all input sections named `.text`
- sections named `.multiboot_header` are added to the first output section (`.boot`) to ensure they are at the beginning of the executable

So let's create the ELF object files and link them using our new linker script. We can use `objdump` to print the sections of the generated executable.

```
> nasm -f elf64 multiboot_header.asm
> nasm -f elf64 boot.asm
> ld -o kernel.bin -T linker.ld multiboot_header.o boot.o
> objdump -h kernel.bin
kernel.bin:     file format elf64-x86-64

Sections:
Idx Name          Size      VMA               LMA               File off  Algn
  0 .boot         00000018  0000000000200078  0000000000200078  00000078  2**0
                  CONTENTS, ALLOC, LOAD, READONLY, DATA
  1 .text         0000000b  0000000000200090  0000000000200090  00000090  2**4
                  CONTENTS, ALLOC, LOAD, READONLY, CODE
```

## Creating the ISO
The last step is to create a bootable ISO image with GRUB. We need to create the following directory structure and copy the `kernel.bin` to the right place:

```
isofiles
└── boot
    ├── grub
    │   └── grub.cfg
    └── kernel.bin

```
The `grub.cfg` specifies the file name of our kernel and that it's Multiboot 2 compliant. It looks like this:

```
set timeout=0
set default=0

menuentry "my os" {
   multiboot2 /boot/kernel.bin
   boot
}
```
Now we can create a bootable image using the command:

```
grub-mkrescue -o os.iso isofiles
```

## Booting

```
qemu-system-x86_64 -hda os.iso
```

[^fn-checksum_hack]: The formula from the table, `-(magic + architecture + header length)`, creates a negative value that doesn't fit into 32bit. By subtracting from `0x100000000` instead, we keep the value positive without changing its truncated value. Without the additional sign bit(s) the result fits into 32bit and the compiler is happy.

[multiboot]: https://en.wikipedia.org/wiki/Multiboot_Specification
[grub 2]: http://wiki.osdev.org/GRUB_2
[multiboot 2]: http://nongnu.askapache.com/grub/phcoder/multiboot.pdf
[Opcodes]: https://en.wikipedia.org/wiki/Opcode
[ELF]: https://en.wikipedia.org/wiki/Executable_and_Linkable_Format
