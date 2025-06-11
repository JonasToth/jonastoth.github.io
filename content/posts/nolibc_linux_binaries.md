+++
title = "Creating Minimal Statically Linked Linux Binaries"
description = "Statically linked binaries for Linux work distribution independent and can be copied around between Linux systems. Using the kernels 'nolibc'-headers provides a way to produce minimal executables performing simple tasks."
date = 2025-06-10T12:00:00+02:00
type = "post"
tags = ["c", "build-tooling", "nolibc", "linux"]
draft = false
showTableOfContents = true
+++

The linux kernel provides a stability guarantee for its syscall interface.
That means, that a userspace request to the kernel -- like opening a file, listing a directory, ... -- happens with the same instruction sequence for every version of the kernel for a given syscall.
Guaranteeing this interface makes statically linking executables in Linux viable.
Everyday programs still rely on the C standard library to actually interface with the kernel giving the code portability between operating systems and allowing transparent use of _newer and better_ syscalls between kernel versions.

Modern programming languages like [`Golang`](#go-linkage) or [`Rust`](#rust-linkage) avoid the hazzles of dynamic linking for library dependencies and statically link most libraries by default.
`C` and `C++` support static and dynamic linking and it's usually the developers or distributors choice how the software is shipped.
In any case, the `C` standard library is often dynamically linked in [`glibc`](https://www.gnu.org/software/libc/) based Linux distributions.
Containerization and the "just copy the binary" distribution model are a huge use case for fully statically linked executables.
[Alpine Linux](https://www.alpinelinux.org) with [`musl`](https://musl.libc.org) as the C standard library is the default choice in such situations.

In this post I want to show you an uncommon way of static linking for `C` by _not_ using a classical C standard library resulting in small, minimal programs.
This approach uses the Linux kernel's own `nolibc` header files that implement _just enough_ C standard library interfaces to do interesting things and interact with the Linux kernel.
Please note, that this approach is intended for single file programs that don't do a lot.
Knowing this approach and reading `nolibc` source code will deepen your understanding of program-kernel interaction in Linux.

## Get the kernel sources

[`nolibc`](https://github.com/torvalds/linux/blob/v6.15/tools/include/nolibc/nolibc.h) lives in the Linux git repository.
There are many ways to get access to the kernel source code, e.g. through your distribution packages, but this post gives you an example on how to clone it for yourself.
The `git` commands use a few more [advanced filtering techniques](https://github.blog/open-source/git/get-up-to-speed-with-partial-clone-and-shallow-clone/) to speedup the cloning process.
```bash
$ # Create the directories at a place of your liking. If you have the
$ # kernel source already, you can skip these steps.
$ mkdir -p ~/software/ && cd ~/software
$ git clone https://github.com/torvalds/linux.git \
    --branch v6.15 \
    --sparse --depth=1 \
    --filter=blob:none
$ cd linux
$ git sparse-checkout add tools/include/nolibc

$ ls -1 tools/include/nolibc
> arch-aarch64.h
> arch-arm.h
> arch.h
> ...
> nolibc.h
> ...
```

The file `nolibc.h` includes everything included in `nolibc` and can be passed on the compile commandline in a 'no-#include' program.

## A Small C-Program

Experiments are best done with simple programs.
Let's do "Hello World".
`nolibc` is designed for single file programs as explained in the [nolibc author's LWN article](#lwn-article) and "Hello World" is certainly one of them.
```c
// main.c
#ifndef NOLIBC
#include <stdio.h>
#endif

int main(int argc, char** argv) {
    puts("Hello World");
    return argc;
}
```
Optionally including the header `stdio.h` if `NOLIBC` is not defined ensures that the program is compatible with classical C standard libraries.

Compiling and linking this program is a little more involved compared to the classical "Hello World" in `C`.
```make
# Makefile
all: main.x

.PHONY: clean
clean:
        rm main.x main.o

main.x: main.o
        $(LD) \
            -nostdlib \
            -static \
            --strip-all \
            --gc-sections --print-gc-sections \
            -o main.x \
            main.o

main.o: main.c
        $(CC) \
            -nostdlib \
            -fno-asynchronous-unwind-tables -fno-ident \
            -include ~/software/linux/tools/include/nolibc/nolibc.h \
            -std=c11 -m64 -static -Oz -s -c \
            -o main.o \
            main.c

```
The most important compiler options are `-nostdlib` and `-include .../nolibc.h`.
These flags instruct the compiler to _not_ use the default C standard library -- most likely `glibc` on Linux -- and force the inclusion of `nolibc.h`.
This makes all `nolibc`-implemented functionality available in the C source code _without_ using an `#include`.

A first quick look onto the result shows that it works!
```bash
$ make
$ file main.x
> main.x: ELF 64-bit LSB executable, x86-64, version 1 (SYSV), statically linked, stripped
$ ls -lh
> -rw-r--r-- 1 jonas jonas  121 10. Jun 13:28 main.c
> -rw-r--r-- 1 jonas jonas 2,9K 10. Jun 16:11 main.o
> -rwxr-xr-x 1 jonas jonas  13K 10. Jun 16:11 main.x
> -rw-r--r-- 1 jonas jonas  384 10. Jun 16:11 Makefile
$ ./main.x 42 123 23 ; echo $?
> Hello World
> 4 
```
We got a program, it is `13K` big?!
First, let's evaluate how it compares to the other options of statically linked C executables.

## Size Comparison with glibc and musl

Executable files in Linux are `ELF` files.
`ELF` files are are composed of sections that all have a specific meaning and task.
`.text` contains the program instructions and is the only section this blog post looks at.
For more information consider reading the very informative book [Linkers and Loaders](#linkers-loaders) that explains the relevant ingredient to program creation in great detail or the shorter posts on [LWN.net](#programs-elf) on how [programs are run](#programs-run) in Linux.
```bash
$ readelf --sections main.x
> There are 8 section headers, starting at offset 0x3040:
> 
> Section Headers:
>   [Nr] Name              Type             Address           Offset
>        Size              EntSize          Flags  Link  Info  Align
>   [ 0]                   NULL             0000000000000000  00000000
>        0000000000000000  0000000000000000           0     0     0
>   [ 1] .note.gnu.pr[...] NOTE             0000000000400200  00000200
>        0000000000000040  0000000000000000   A       0     0     8
>   [ 2] .text             PROGBITS         0000000000401000  00001000
>        0000000000000117  0000000000000000  AX       0     0     1
>   [ 3] .rodata           PROGBITS         0000000000402000  00002000
>        000000000000000c  0000000000000001 AMS       0     0     1
>   [ 4] .got              PROGBITS         0000000000403fe0  00002fe0
>        0000000000000008  0000000000000000  WA       0     0     8
>   [ 5] .got.plt          PROGBITS         0000000000403fe8  00002fe8
>        0000000000000018  0000000000000008  WA       0     0     8
>   [ 6] .bss              NOBITS           0000000000404000  00003000
>        0000000000000018  0000000000000000  WA       0     0     8
>   [ 7] .shstrtab         STRTAB           0000000000000000  00003000
>        000000000000003f  0000000000000000           0     0     1
> Key to Flags:
>   W (write), A (alloc), X (execute), M (merge), S (strings), I (info),
>   L (link order), O (extra OS processing required), G (group), T (TLS),
>   C (compressed), x (unknown), o (OS specific), E (exclude),
>   D (mbind), l (large), p (processor specific)
```
The `.text` section is `0x117 = 279` bytes in size.
All other sections are smaller than `.text`.
The `13K` file size comes from the meta data in `ELF` and the internal file layout.

Building the same program with `glibc` or `musl` as statically linked executable provides a reference point for size comparisons.
I use [`incus`](https://linuxcontainers.org/incus/) to spin up containers of Linux distributions using these C standard libraries.
Classical `docker` containers or VMs work fine, too.
```bash
$ incus launch images:alpine/3.22/amd64 alpine-3-22
$ incus launch images:debian/13/amd64 debian-13
```
The same `main.c` file is created and compiled in each of those containers with a simpler compiler invocation.
`Alpine 3.22` Linux uses `musl` which is made for static linking.
The resulting binary size is comparable to `nolibc`.
Inspecting the ELF file shows that there is "more going on" and the `.text` section is more than 10x bigger with `0xe3d = 3645` bytes.
```bash
(alpine) $ gcc -static -fno-asynchronous-unwind-tables -fno-ident -std=c11 -m64 -static -Oz -s main.c -o main.x
(alpine) $ ls -lh main.x
> -rwxr-xr-x    1 root     root       13.4K Jun 10 14:38 main.x

(alpine) $ readelf --sections main.x
> There are 15 section headers, starting at offset 0x31b0:
> 
> Section Headers:
>   [Nr] Name              Type             Address           Offset
>        Size              EntSize          Flags  Link  Info  Align
>   [ 0]                   NULL             0000000000000000  00000000
>        0000000000000000  0000000000000000           0     0     0
>   [ 1] .note.gnu.pr[...] NOTE             0000000000400238  00000238
>        0000000000000030  0000000000000000   A       0     0     8
>   [ 2] .note.gnu.bu[...] NOTE             0000000000400268  00000268
>        0000000000000024  0000000000000000   A       0     0     4
>   [ 3] .init             PROGBITS         0000000000401000  00001000
>        0000000000000003  0000000000000000  AX       0     0     1
>   [ 4] .text             PROGBITS         0000000000401020  00001020
>        0000000000000e3d  0000000000000000  AX       0     0     32
>   [ 5] .fini             PROGBITS         0000000000401e5d  00001e5d
>        0000000000000003  0000000000000000  AX       0     0     1
>   [ 6] .rodata           PROGBITS         0000000000402000  00002000
>        0000000000000016  0000000000000001 AMS       0     0     1
>   [ 7] .eh_frame         PROGBITS         0000000000402018  00002018
>        0000000000000004  0000000000000000   A       0     0     8
>   [ 8] .init_array       INIT_ARRAY       0000000000403fc8  00002fc8
>        0000000000000008  0000000000000008  WA       0     0     8
>   [ 9] .fini_array       FINI_ARRAY       0000000000403fd0  00002fd0
>        0000000000000008  0000000000000008  WA       0     0     8
>   [10] .got              PROGBITS         0000000000403fd8  00002fd8
>        0000000000000028  0000000000000008  WA       0     0     8
>   [11] .data             PROGBITS         0000000000404000  00003000
>        000000000000010c  0000000000000000  WA       0     0     32
>   [12] .bss              NOBITS           0000000000404120  0000310c
>        00000000000006d0  0000000000000000  WA       0     0     32
>   [13] .comment          PROGBITS         0000000000000000  0000310c
>        000000000000001c  0000000000000001  MS       0     0     1
>   [14] .shstrtab         STRTAB           0000000000000000  00003128
>        0000000000000086  0000000000000000           0     0     1
> Key to Flags:
>   W (write), A (alloc), X (execute), M (merge), S (strings), I (info),
>   L (link order), O (extra OS processing required), G (group), T (TLS),
>   C (compressed), x (unknown), o (OS specific), E (exclude),
>   D (mbind), l (large), p (processor specific)
```
Doing the same exercise with `Debian 13` is a real contrast.
```bash
(debian) $ gcc -static -fno-asynchronous-unwind-tables -fno-ident -std=c11 -m64 -static -Oz -s main.c -Wl,--gc-sections -o main.x
(debian) $ ls -lah main.x
> -rwxr-xr-x 1 root root 657K Jun 10 14:47 main.x

(debian) $ readelf --sections main.x
> readelf --sections main.x
> There are 24 section headers, starting at offset 0xa3b98:
> 
> Section Headers:
>   [Nr] Name              Type             Address           Offset
>        Size              EntSize          Flags  Link  Info  Align
>   [ 0]                   NULL             0000000000000000  00000000
>        0000000000000000  0000000000000000           0     0     0
>   [ 1] .note.gnu.pr[...] NOTE             00000000004002a8  000002a8
>        0000000000000020  0000000000000000   A       0     0     8
>   [ 2] .note.gnu.bu[...] NOTE             00000000004002c8  000002c8
>        0000000000000024  0000000000000000   A       0     0     4
>   [ 3] .rela.plt         RELA             00000000004002f0  000002f0
>        0000000000000210  0000000000000018  AI       0    19     8
>   [ 4] .init             PROGBITS         0000000000401000  00001000
>        0000000000000017  0000000000000000  AX       0     0     4
>   [ 5] .plt              PROGBITS         0000000000401018  00001018
>        00000000000000b0  0000000000000000  AX       0     0     8
>   [ 6] .text             PROGBITS         0000000000401100  00001100
>        0000000000074b39  0000000000000000  AX       0     0     64
>   [ 7] .fini             PROGBITS         0000000000475c3c  00075c3c
>        0000000000000009  0000000000000000  AX       0     0     4
>   [ 8] .rodata           PROGBITS         0000000000476000  00076000
>        000000000001bb84  0000000000000000   A       0     0     32
>   [ 9] rodata.cst32      PROGBITS         0000000000491ba0  00091ba0
>        0000000000000060  0000000000000020  AM       0     0     32
>   [10] .eh_frame         PROGBITS         0000000000491c00  00091c00
>        000000000000b680  0000000000000000   A       0     0     8
>   [11] .gcc_except_table PROGBITS         000000000049d280  0009d280
>        00000000000000f2  0000000000000000   A       0     0     1
>   [12] .note.ABI-tag     NOTE             000000000049d374  0009d374
>        0000000000000020  0000000000000000   A       0     0     4
>   [13] .tdata            PROGBITS         000000000049ee28  0009de28
>        0000000000000018  0000000000000000 WAT       0     0     8
>   [14] .tbss             NOBITS           000000000049ee40  0009de40
>        0000000000000040  0000000000000000 WAT       0     0     8
>   [15] .init_array       INIT_ARRAY       000000000049ee40  0009de40
>        0000000000000010  0000000000000008  WA       0     0     8
>   [16] .fini_array       FINI_ARRAY       000000000049ee50  0009de50
>        0000000000000010  0000000000000008  WA       0     0     8
>   [17] .data.rel.ro      PROGBITS         000000000049ee60  0009de60
>        00000000000040e8  0000000000000000  WA       0     0     32
>   [18] .got              PROGBITS         00000000004a2f48  000a1f48
>        0000000000000088  0000000000000000  WA       0     0     8
>   [19] .got.plt          PROGBITS         00000000004a2fe8  000a1fe8
>        00000000000000c8  0000000000000008  WA       0     0     8
>   [20] .data             PROGBITS         00000000004a30c0  000a20c0
>        00000000000019d8  0000000000000000  WA       0     0     32
>   [21] .bss              NOBITS           00000000004a4aa0  000a3a98
>        0000000000005808  0000000000000000  WA       0     0     32
>   [22] .comment          PROGBITS         0000000000000000  000a3a98
>        000000000000001f  0000000000000001  MS       0     0     1
>   [23] .shstrtab         STRTAB           0000000000000000  000a3ab7
>        00000000000000e0  0000000000000000           0     0     1
> Key to Flags:
>   W (write), A (alloc), X (execute), M (merge), S (strings), I (info),
>   L (link order), O (extra OS processing required), G (group), T (TLS),
>   C (compressed), x (unknown), o (OS specific), E (exclude),
>   D (mbind), l (large), p (processor specific)
```
The `.text` section for this `glibc` based binaries is `0x74b39 = 478009` bytes.
Inspecting the sections of the ELF file unveils a more bloated executable in structure too.

## Conclusion

The presented binaries are not the most reduced binary files.
There are way more tricks to reduce binary size ([Blog Post on Teensy ELF Files](#teensy-elf), [The Art of Creating Minimal ELF64 Executables](#minimal-elf64-executables)) but is it worth it?
`nolibc` is interesting _educationally_ to understand and trace what essentials a program has to perform.
I don't have a big use case for `nolib` and `musl` is the better default static linking library.
For experimenting around at the lowest level, its the best option.
It will be helpful to understand more details of program loading and linking ([What Happens before main()](#before-main), [How programs get run](#programs-run), [How programs get run: ELF binaries](#programs-elf), [Linkers and Loaders](#linkers-loaders)).

## References

1. <a name="lwn-article"></a>[LWN Article of nolibc Author](https://lwn.net/Articles/920158/)
1. [`nolibc`](https://github.com/torvalds/linux/blob/v6.15/tools/include/nolibc/nolibc.h) code in the Kernel
1. <a name="linkers-loaders"></a>[Linkers and Loaders from John R. Levine](https://freecomputerbooks.com/Linkers-and-Loaders.html)
1. <a name="go-linkage"></a>[Go Linkage](https://www.freecodecamp.org/news/golang-statically-and-dynamically-linked-go-binaries/)
1. <a name="rust-linkage"></a>[Rust Linkage](https://doc.rust-lang.org/reference/linkage.html)
1. <a name="teensy-elf"></a>[Creating Really Tiny ELF Executables](https://www.muppetlabs.com/~breadbox/software/tiny/teensy.html)
1. <a name="minimal-elf64-executables"></a>[The Art of Creating Minimal ELF64 Executables by Unconventional Methods](https://dxrgrabowski.github.io/2024/09/27/3-tiniest-elf/)
1. <a name="before-main"></a>[What Happens Before main()](https://embeddedartistry.com/blog/2019/04/08/a-general-overview-of-what-happens-before-main/)
1. <a name="programs-run"></a>[LWN: How programs get run](https://lwn.net/Articles/630727/)
1. <a name="programs-elf"></a>[LWN: How programs get run: ELF binaries](https://lwn.net/Articles/631631/)
