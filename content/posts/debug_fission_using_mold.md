+++
title = "Better Build Throughput via Debug Fission"
description = "Debug Fission splits debug information of ELF files into separate debug files that can be handled independently. This improves build throughput and enables leaner binary distribution."
date = 2025-06-12T23:00:00+02:00
type = 'post'
tags = ["cpp", "build-tooling", "mold", "debugging", "debug-fission", "split-dwarf"]
draft = false
showTableOfContents = true
+++

Programs are loaded from executable files and then executed.
These files contain either a program or library code that is used by one or more executables and is shared across the system.
Exectuable files in Linux are written in the [Executable and Linkable Format](https://en.wikipedia.org/wiki/Executable_and_Linkable_Format) -- ELF from now on -- containing metadata, dependency information, machine instructions and optionally additional useful information.
One form of particularly useful additional information are debug symbols, used by debuggers to cross reference specific instructions with human readable source code and is stored in the [DWARF](https://en.wikipedia.org/wiki/DWARF) format.
Debug information requires a high level of fidelity to properly relate to the original source code, leading to the situation that a significant portion of an ELF file might actually be debug information.
Normal end users on the other hand have no use for debug information and the natural question becomes, can executable code and debug information handled separatly?

## Issues with Debug Information within the Executable

The increased size of executable files leads to several drawbacks.
Distributions and software repositories storing and serving the executable files have higher costs due to increased network bandwidth and disk usage requirements.
Downloads take longer for end users.
Especially in constrained environments like embedded devices and/or connectivity via mobile networks are these down sides significant.
In case of proprietary software, distributing debug information leaks interna of the source code, reducing the burden to reverse engineer a software product.
All these aspects provide cost with no value to the end user.
On the other hand, if there is a crash or issue, the software developers _need_ debug information just to locate the source code to inspect.

Even during the development cycle of writing, compiling and then executing or debugging the program, the integrated debug information comes at a cost.
For the compilation process to finish, the executable file needs to be fully written, causing increased wait times before the program can be started.

Splitting debug information off of executables gives the best of both worlds: have debug information if needed but not needing to distribute and pay for it everywhere.
This is culturally more common in Windows environments but equally possible in Linux, too.
The process of generating distinct debug files is sometimes called _debug fission_ or _split dwarf_.

## Classical Split-Dwarf

ELF files are organized in _sections_ from the perspective of the linker and _segments_ from the perspective of the program loader.
DWARF defines multiple section types that contain specific pieces of the debug information.
These sections are present in the file but ignored by the program loader.
Let's see this happening for an example build of [Waybar](https://github.com/Alexays/Waybar).
The output is shortened to the interesting pieces.
```bash
$ cd <Waybar-Repo>
$ meson setup build --buildtype=debugoptimized
$ meson compile -C build

$ cd build
$ ls -lh waybar
> -rwxr-xr-x 1 jonas jonas 73M waybar

$ readelf --sections waybar
> There are 41 section headers, starting at offset 0x48f4138:
> 
> Section Headers:
>   [Nr] Name              Type             Address           Offset
>        Size              EntSize          Flags  Link  Info  Align
>   [ 0]                   NULL             0000000000000000  00000000
>        0000000000000000  0000000000000000           0     0     0
>   [ 1] .interp           PROGBITS         0000000000000318  00000318
>        000000000000001c  0000000000000000   A       0     0     1
>   ... 
>   [13] .text             PROGBITS         0000000000032480  00032480
>        00000000001eb3eb  0000000000000000  AX       0     0     64
>   ...
>   [29] .comment          PROGBITS         0000000000000000  002949f0
>        0000000000000051  0000000000000001  MS       0     0     1
>   [30] .debug_aranges    PROGBITS         0000000000000000  00294a41
>        0000000000013260  0000000000000000           0     0     1
>   [31] .debug_info       PROGBITS         0000000000000000  002a7ca1
>        0000000002950a77  0000000000000000           0     0     1
>   [32] .debug_abbrev     PROGBITS         0000000000000000  02bf8718
>        000000000009736b  0000000000000000           0     0     1
>   [33] .debug_line       PROGBITS         0000000000000000  02c8fa83
>        000000000058c029  0000000000000000           0     0     1
>   [34] .debug_str        PROGBITS         0000000000000000  0321baac
>        00000000005f03b0  0000000000000001  MS       0     0     1
>   [35] .debug_line_str   PROGBITS         0000000000000000  0380be5c
>        0000000000002ed6  0000000000000001  MS       0     0     1
>   [36] .debug_loclists   PROGBITS         0000000000000000  0380ed32
>        0000000000df9c9e  0000000000000000           0     0     1
>   [37] .debug_rnglists   PROGBITS         0000000000000000  046089d0
>        00000000002708d5  0000000000000000           0     0     1
>   [38] .symtab           SYMTAB           0000000000000000  048792a8
>        0000000000021c00  0000000000000018          39   2136     8
>   [39] .strtab           STRTAB           0000000000000000  0489aea8
>        00000000000590db  0000000000000000           0     0     1
>   ...
```
The various `.debug_*` sections contain the actual debug information with significant section sizes.
These sections are not actually loaded if the program is started.
```bash
$ readelf --segments waybar
> 
> Elf file type is DYN (Position-Independent Executable file)
> Entry point 0x76f00
> There are 13 program headers, starting at offset 64
> 
> Program Headers:
>   Type           Offset             VirtAddr           PhysAddr
>                  FileSiz            MemSiz              Flags  Align
>   PHDR           0x0000000000000040 0x0000000000000040 0x0000000000000040
>                  0x00000000000002d8 0x00000000000002d8  R      0x8
>   INTERP         0x0000000000000318 0x0000000000000318 0x0000000000000318
>                  0x000000000000001c 0x000000000000001c  R      0x1
>       [Requesting program interpreter: /lib64/ld-linux-x86-64.so.2]
>   LOAD           0x0000000000000000 0x0000000000000000 0x0000000000000000
>                  0x000000000002ac58 0x000000000002ac58  R      0x1000
>   LOAD           0x000000000002b000 0x000000000002b000 0x000000000002b000
>                  0x00000000001f2879 0x00000000001f2879  R E    0x1000
>   LOAD           0x000000000021e000 0x000000000021e000 0x000000000021e000
>                  0x000000000006c1a0 0x000000000006c1a0  R      0x1000
>   LOAD           0x000000000028aa38 0x000000000028ba38 0x000000000028ba38
>                  0x0000000000009fb8 0x000000000000a9a0  RW     0x1000
>   DYNAMIC        0x0000000000291e98 0x0000000000292e98 0x0000000000292e98
>                  0x00000000000003e0 0x00000000000003e0  RW     0x8
>   NOTE           0x000000000028a130 0x000000000028a130 0x000000000028a130
>                  0x0000000000000050 0x0000000000000050  R      0x8
>   NOTE           0x000000000028a180 0x000000000028a180 0x000000000028a180
>                  0x0000000000000020 0x0000000000000020  R      0x4
>   GNU_PROPERTY   0x000000000028a130 0x000000000028a130 0x000000000028a130
>                  0x0000000000000050 0x0000000000000050  R      0x8
>   GNU_EH_FRAME   0x0000000000239fa0 0x0000000000239fa0 0x0000000000239fa0
>                  0x0000000000007004 0x0000000000007004  R      0x4
>   GNU_STACK      0x0000000000000000 0x0000000000000000 0x0000000000000000
>                  0x0000000000000000 0x0000000000000000  RW     0x10
>   GNU_RELRO      0x000000000028aa38 0x000000000028ba38 0x000000000028ba38
>                  0x00000000000095c8 0x00000000000095c8  R      0x1
> 
>  Section to Segment mapping:
>   Segment Sections...
>    00
>    01     .interp
>    02     .interp .gnu.hash .dynsym .dynstr .gnu.version .gnu.version_r .rela.dyn .rela.plt
>    03     .init .plt .plt.got .plt.sec .text .fini
>    04     .rodata .gresource.waybar_icons .eh_frame_hdr .eh_frame .gcc_except_table .note.gnu.property .note.ABI-tag
>    05     .init_array .fini_array .data.rel.ro .dynamic .got .data .bss
>    06     .dynamic
>    07     .note.gnu.property
>    08     .note.ABI-tag
>    09     .note.gnu.property
>    10     .eh_frame_hdr
>    11
>    12     .init_array .fini_array .data.rel.ro .dynamic .got
```
The "Section to Segment mapping" does not contain any of the debug sections that are therefore not loaded during normal execution.

"Debug Fission" splits the sections containing debug information into a separate ELF file which does not need executable code itself.
A small `.gnu_debuglink` section in the executable file may point to the separate debug ELF file.

#### Separating Debug Sections Into Their Own ELF File

The following description is contained in the `man objcopy` under the `--only-keep-debug` option as well.
This approach is useful for distributors that do not develop the software themselves but compile it from source and may ship it as a final executable.

Splitting of debug information is performed with the tool `objcopy` of the binutils.
```bash
$ objcopy --only-keep-debug waybar waybar.dbg
$ ls -lh waybar{,.dbg}
> -rwxr-xr-x 1 jonas jonas 73M waybar
> -rwxr-xr-x 1 jonas jonas 71M waybar.dbg

$ objcopy --strip-debug waybar
$ # Optional, but useful, you can compress debug information!
$ objcopy --compress-debug-sections=zstd waybar.dbg
$ objcopy --add-gnu-debuglink=waybar.dbg waybar
$ ls -lh waybar{,.dbg}
> -rwxr-xr-x 1 jonas jonas 3,1M waybar
> -rwxr-xr-x 1 jonas jonas  71M waybar.dbg # Uncompressed
> -rwxr-xr-x 1 jonas jonas  25M waybar.dbg # Compressed with zstd

$ # Without compressed debug information:
$ readelf --debug-dump=links waybar
> waybar: Found separate debug info file: waybar.dbg
> Contents of the .gnu_debuglink section (loaded from waybar):
> 
>   Separate debug info file: waybar.dbg
>   CRC value: 0x97762243

$ # With compressed debug information:
$ readelf --debug-dump=links waybar
> waybar: Found separate debug info file: waybar.dbg
> Contents of the .gnu_debuglink section (loaded from waybar):
> 
>   Separate debug info file: waybar.dbg
>   CRC value: 0xe6281
```
_Almost all_ of the executable file was debug information, that's what I meant with 'significant size'.
You can see, that the `--only-keep-debug` option reduces the resulting `.dbg` file, but not by a lot.
It is possible to just `cp waybar waybar.dbg` as the first step, too!
Note that the "debug link" contains a [CRC](https://en.wikipedia.org/wiki/Cyclic_redundancy_check) value for simple checking of file integrity.
If the debug information is compressed _after_ the link is created, the `readelf --debug-dump=links` command will register the mismatch and complain.

#### Write Separate DWARF-Files During Compilation

The process of building an executable that contains all debug information has a price during the normal development cycle.
Development builds should be incremental and a small change should result in a small and fast build.
When linking the executable or library, all the debug information must be gathered and combined and finally be included in the ELF file, this has a runtime cost.
Therefore, the ["Split Dwarf"](https://gcc.gnu.org/wiki/DebugFission) feature was created that writes separate debug information files _for development builds_.

Both `gcc` and `clang` do support this feature under the `-gsplit-dwarf` flag.
During compilation, debug information is not emitted in the regular `.o` object file but into a separate `.dwo` file instead.
Performing an example build of [Waybar](https://github.com/Alexays/Waybar) with split-dwarf looks as follows:
```bash
$ cd <Waybar-Repo>
$ # When using 'gcc':
$ meson setup build --buildtype=debugoptimized -Dcpp_args="-gsplit-dwarf"
$ # When using 'clang':
$ export CC=clang
$ export CXX=clang++
$ meson setup build --buildtype=debugoptimized -Dcpp_args="-gsplit-dwarf=split"

$ meson compile -C build
$ ls -lh1 build/waybar.p/
> ...
> -rw-r--r-- 1 jonas jonas 422K src_AAppIconLabel.cpp.dwo
> -rw-r--r-- 1 jonas jonas 304K src_AAppIconLabel.cpp.o
> -rw-r--r-- 1 jonas jonas 165K src_AIconLabel.cpp.dwo
> -rw-r--r-- 1 jonas jonas 105K src_AIconLabel.cpp.o
> -rw-r--r-- 1 jonas jonas 443K src_ALabel.cpp.dwo
> -rw-r--r-- 1 jonas jonas 412K src_ALabel.cpp.o
> -rw-r--r-- 1 jonas jonas 397K src_AModule.cpp.dwo
> -rw-r--r-- 1 jonas jonas 284K src_AModule.cpp.o
> -rw-r--r-- 1 jonas jonas 151K src_ASlider.cpp.dwo
> -rw-r--r-- 1 jonas jonas  80K src_ASlider.cpp.o
> -rw-r--r-- 1 jonas jonas 634K src_bar.cpp.dwo
> -rw-r--r-- 1 jonas jonas 529K src_bar.cpp.o
> -rw-r--r-- 1 jonas jonas 818K src_client.cpp.dwo
> -rw-r--r-- 1 jonas jonas 900K src_client.cpp.o
> -rw-r--r-- 1 jonas jonas 982K src_config.cpp.dwo
> -rw-r--r-- 1 jonas jonas 1,3M src_config.cpp.o
> ...
```
The created `.dwo` files are similar in size to the `.o` files.
An incremental build now only changes the necessary files and the final linking can ignore the handling of debug information.
Debuggers search a matching `.dwo` file and read debug information from it if found.

All the "loose" `.dwo` files are not easy to distribute and handle though.
This is where the `dwp` (or `llvm-dwp`) tool comes in.
It bundles all `.dwo` files into a `.dwp` ([DWARF Package File Format](https://gcc.gnu.org/wiki/DebugFissionDWP)) file, yielding similar situation to the `waybar.dbg` file.
Note that the resulting `.dwp` file is _not_ an ELF file.

## Improved Debug Fission with Mold

[Mold](https://github.com/rui314/mold) entered the software development ecosystem with a splash.
It is a modern linker with the project goal to provide better developer tooling.
The linker is takes only twice the time of just copying the resulting executable with `cp`, proper benchmarks for comparison are provided in the project repository -- `mold` links _very_ fast.

Since version `2.33` the linker has the capability to write separate debug files during linking.
Subsequent releases improved upon that and include directly compressing debug information (since 2.37) and adding a _GDB Index_ for faster load times of the debug information during debugging.
Both features are availabe with `objcopy --compress-debug-sections` and `gdb-add-index <file>`.
But independent program invocations are harder to integrate into a build system and process.
The way `mold` handles debug information is like `objcopy --only-keep-debug` as described above, but nicer to use.

Please note, that the option names in `mold` did change through the releases and the documented flags in the releases notes are not necessarily accurate anymore.

Again, an example build with [Waybar](https://github.com/Alexays/Waybar):
```bash
$ export CXX_LD=mold 
$ MOLD_ARGS="-Wl,--separate-debug-file,--compress-debug-sections=zstd,--gdb-index"
$ meson setup build-mold --buildtype=debugoptimized -Dcpp_link_args="$MOLD_ARGS"
$ meson compile -C build-mold ; cd build-mold

$ ls -lh1 waybar{,.dbg}
> -rwxr-xr-x 1 jonas jonas 2,6M waybar
> -rw-r--r-- 1 jonas jonas  25M waybar.dbg

$ readelf --sections waybar
> There are 43 section headers, starting at offset 0x2929d0:
> 
> Section Headers:
>   [Nr] Name              Type             Address           Offset
>        Size              EntSize          Flags  Link  Info  Align
>   [ 0]                   NULL             0000000000000000  00000000
>        0000000000000000  0000000000000000           0     0     0
>   [ 1] .interp           PROGBITS         00000000000002e0  000002e0
>        000000000000001c  0000000000000000   A       0     0     1
>   [ 2] .note.gnu.pr[...] NOTE             0000000000000300  00000300
>        0000000000000050  0000000000000000   A       0     0     8
>   [ 3] .note.ABI-tag     NOTE             0000000000000350  00000350
>        0000000000000020  0000000000000000   A       0     0     4
>   [ 4] .hash             HASH             0000000000000370  00000370
>        00000000000024f0  0000000000000004   A       6     0     4
>   [ 5] .gnu.hash         GNU_HASH         0000000000002860  00002860
>        0000000000000338  0000000000000000   A       6     0     8
>   ...
>   [40] .comment          PROGBITS         0000000000000000  00292770
>        0000000000000077  0000000000000001  MS       0     0     1
>   [41] .gnu_debuglink    PROGBITS         0000000000000000  002927e8
>        0000000000000010  0000000000000000           0     0     4
>   [42] .shstrtab         STRTAB           0000000000000000  002927f8
>        00000000000001d8  0000000000000000           0     0     1
>   ...

$ readelf --debug-dump=links waybar
> waybar: Found separate debug info file: waybar.dbg
> readelf: Warning: Corrupt debuglink section: .gnu_debuglink
> Contents of the .gnu_debuglink section (loaded from waybar):
> 
>   Separate debug info file: waybar.dbg
>   CRC value: 0x4865a8d0
> 
> section '.gnu_debuglink' has the NOBITS type - its contents are unreliable.
```
Once the build is properly configured -- adding the flags could be done in the build system itself instead of passing them from the outside -- the final executable does not contain the debug information itself, but links to a separate `.dbg` file instead.

The warning message of `readelf` can be ignored.
In my experience, the debug information is properly connected and works with `gdb` and `lldb`.
This seems to be an oversight in `mold`.

`mold`s approach is an improved version of splitting out debug information after the final build.
It does not mix with the `-gsplit-dwarf` compilation.

Another goodie you get from using ` mold` is its ability to finalize the executable file before the corresponding `.dbg` file is fully written.
The `meson`, `cmake`, `make`, `ninja` or whatever command that builds your program would already be finished and you can start the program directly.
`mold` _detaches_ into the background and keeps running until the debug file is fully written, providing the fastest possible development cycle (controlled via `--detach`/`--no-detach`).

**HOW?!**

The `.gnu_debuglink` section must have been written already and this section includes the CRC checksum of the final `.dbg` file, but its not fully created yet.

When `mold` is instructed to write a separate debug file, it [creates a CRC value](https://github.com/rui314/mold/blob/f556d2b4aef8736fd7078acf5ff3513d4324b8a4/src/passes.cc#L3305) that is both reproducible and distinguishes different programs and writes the CRC into the `.gnu_debuglink` section.
The nature of CRC allows to _extend_ the input data with "garbage" to yield a desired CRC value.
This is done via [CRC polynomial solving](https://github.com/rui314/mold/blob/f556d2b4aef8736fd7078acf5ff3513d4324b8a4/lib/crc32.cc#L12).
The garbage values are append to the separate debug file resulting in the necessary CRC checksum.
I found this _such a clever_ approach and was very surprised to find this in a linker.

## Distributing Debug Information

The final aspect of debug fission is distribution and retrieval of the debug information once it becomes necessary or desirable.
Debuggers have multiple heuristics to identify debug information, like looking next to a `.o` file for a `.dwo` file.
These approachs don't scale well beyond a single developers machine.

A better approach involves another neat piece of meta data optionally embedded in ELF files: the _build-ID_.
The build-ID of an ELF file can be queried with `readelf` but may not be available.
```bash
$ readelf --display-section=.note.gnu.build-id waybar
> Displaying notes found in: .note.gnu.build-id
>   Owner                Data size        Description
>   GNU                  0x00000020       NT_GNU_BUILD_ID (unique build ID bitstring)
>     Build ID: c7780d7533763b59e06f0875378701a4237b5021e95d0735ce78fd14e7f431c6
```
To ensure a build-ID is generated with `mold` it is necessary to pass the option `--build-id=sha256` as additional linker flag.
Using `ld` or `lld` requires `--build-id=sha1` instead.
All linkers support `--build-id` but do not use the same algorithm and multiple algorithms are available in each linker.
What all have in common is that an unique 128bit or bigger number is assigned to the build that can be used as identification, enabling a form of [content based addressing](https://en.wikipedia.org/wiki/Content-addressable_storage).
In case of `sha1/sha256` the executable's code sections are hashed.
Using a random UUID instead is possible but not reproducible.
This ID is stable for the same executable even if other parts of the ELF file are changed retrospectively.
Because the chosen number is _big_ there are no clashes between different executables, every distinct program that will ever exist has a different ID.

Naming the separated `.dbg` files to a name that is derived from the build-ID makes the files manageable for distribution and retrieval.
Installing a matching debug information file of an executable in Linux works by using the naming convention `/usr/lib/debug/.build-id/XX/YYYY.debug` with `XX` being the first two digits of the hash and `YYYY` the rest.
In the example the `waybar.dbg` file should be moved to `/usr/lib/debug/.build-id/c7/780d7533763b59e06f0875378701a4237b5021e95d0735ce78fd14e7f431c6.debug`.
The debugger can query the build ID of the executable or library and search for matching debug information files.
Distributors can provide debug information independently and install or uninstall the files through classical packaging mechanisms.
Linux distributions like [Fedora provide independent debuginfo packages](https://docs.fedoraproject.org/en-US/packaging-guidelines/Debuginfo/) through exactly this mechanism.
A missing, not discussed in this post, piece is the handling of source code information.
The approach is similar, the matching source files must be found based on the debug information.
User/Developer specific file paths and/or incorrect versions of the files are the hurdles to take.
The `gcc` option `-ffile-prefix-map`, the `gdb` option `set substitute-path` and their `clang` and `lldb` equivalents solve this issue.
Analogous to debug info, the source files can be installed under `/usr/src/`.

Finally, the debugger may query the debug information on demand from `debuginfod` or `llvm-debuginfod` utilizing the same principles as describes before.

## Conclusion

Splitting of debug information from executables is common practice under Windows and Linux.
Doing so has advantages in the whole life cycle of a program.
This post explained how to perform this separation using modern Linux build tools and how it works under the hood.
Using `mold --separate-debug-file --compress-debug-sections=zstd --gdb-index --build-id=sha256` as linker invocation provides the most convenient way of doing so.
If distribution of the separated debug files is necessary, additional scripting to rename the debug files, provide the source code and package the artifacts is required.

## References

- [Executable and Linkable Format - Wikipedia](https://en.wikipedia.org/wiki/Executable_and_Linkable_Format)
- [DWARF Debug Information Format - Wikipedia](https://en.wikipedia.org/wiki/DWARF)
- [man objcopy](https://www.man7.org/linux/man-pages/man1/objcopy.1.html)
- [CRC Cyclic Redundancy Check: Wikipedia](https://en.wikipedia.org/wiki/Cyclic_redundancy_check)
- [GCC Wiki on Debug Fission](https://gcc.gnu.org/wiki/DebugFission)
- [GCC Wiki for the DWARF Package File Format](https://gcc.gnu.org/wiki/DebugFissionDWP)
- [GCC Wiki for the DebugGNUIndex](https://gcc.gnu.org/wiki/DebugGNUIndexSection)
- [Mold](https://github.com/rui314/mold)
- [GDB Source Code Finding](https://sourceware.org/gdb/current/onlinedocs/gdb.html/Source-Path.html)
