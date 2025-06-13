+++
title = "Perform Debug Fission Better"
description = "Debug Fission splits debug information of ELF files into separate debug files that can be handled independently. Mold improved the workflow for this approach"
date = 2025-06-12T12:00:00+02:00
type = 'post'
tags = ["cpp", "build-tooling", "mold", "debugging"]
draft = false
showTableOfContents = true
+++

Programs are read from executable files.
These files contain either a directly executable program or contain library code that is used by one or more executables and shared across the system.
Exectuable files in Linux are written in the [Executable and Linkable Format](https://en.wikipedia.org/wiki/Executable_and_Linkable_Format) -- ELF from now on -- containing metadata, dependency information, executable code and optionally additional useful information.
The additional useful information may contain debug information, used by debuggers to cross reference executable code with human readable source code and is stored in the [DWARF](https://en.wikipedia.org/wiki/DWARF) format.
Debug information requires a high level of fidelity to properly relate to the original source code, leading to the situation that a significant portion of an ELF file might actually be debug information.

## Issues with Debug Information within the Executable

The increased size of executable files leads to several drawbacks.
Distributions and software repositories storing and serving the executable files have higher costs due to increased network bandwidth and disk usage requirements.
Downloads take longer for end users.
Especially in constrained environments like embedded devices and/or connectivity via mobile networks these down sides are significant.
In case of proprietary software, distributing debug information leaks information about the internal source code, reducing the burden to reverse engineer a software product.
All these drawbacks have little value if software is not debugged by the end user.
On the other hand, if there is a crash or issue, the software developers _need_ debug information just to locate the source code to inspect.

Even during the development cycle of writing, compiling and then executing or debugging the program, the integrated debug information comes at a cost.
For the compilation process to finish, the executable file needs to be fully written, causing increased wait times before the program can be started.

Splitting debug information off of executables gives the best of both worlds: have debug information if needed but not needing to distribute and pay for it everywhere.
This is more common in Windows environments but possible in Linux, too.
The process of generating distinct debug files is sometimes called _debug fission_.

## Classical Split-Dwarf

- split dwarf relies on the section nature of ELF
- a second ELF file containing only the debug-information sections is written
- a `debuglink` section informs other programs to look for the debug file next to the executable file

### Separating Debug Sections Into Their Own ELF File

- `objcopy` on the final file => see `man objcopy` section on it
- `debug-link` as property to say which file contains the debug information, used by the `debugger`

### Write Separate DWARF-Files During Compilation

- `-gsplit-dwarf` during compilation
- each `.o` object file gets a parallel `.dwo` file that contains the debug information
- multiple `.dwo` files can be bundled to a `.dwp` file, similar to a `.a` archive that bundles multiple `.o` files

## Improved Debug Fission with Mold

- add `--separate-debug-info` as linker argument to `mold` (requires >=2.33)
- add `--compressed-debug-info` to reduce file size (requires >=2.37 to work with `--separate-debug-info`)
- add `--gdb-index` for faster loading times in `gdb` (requires `-ggnu-pubnames` during compilation)
- this leads to faster building because the file IO is separated from compiling
- mold can detach the "write debug info process" and allow earlier execution of the executable
- nugget: `mold` writes debug information in the background and the executable can be started even before the debug info is fully written

## Distributing Debug Information

- build-id is hash of executable code in the ELF file
- build-id is the fingerprint of the executable and can be used to uniquely identify a build of an ELF file
- opportunity to retrieve the matching debug information via this build-id
- the separate debug information file is named with the build-id in `/usr/lib/debug/<XX>/<XXXXXXXXXX>.debug` for efficient retrieval by the debugger
- allows `debuginfod` to hold the debug files and retrieve them over network

## References

- [Executable and Linkable Format - Wikipedia](https://en.wikipedia.org/wiki/Executable_and_Linkable_Format)
- [DWARF Debug Information Format - Wikipedia](https://en.wikipedia.org/wiki/DWARF)
