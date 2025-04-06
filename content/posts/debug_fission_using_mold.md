+++
title = "A Faster Way to Perform Debug Fission"
description = "Debug Fission splits debug information of ELF files into separate debug files that can be handled independently. Mold improved the workflow for this approach"
date = 2025-04-06T12:00:00+02:00
type = 'post'
tags = ["cpp", "build-tooling", "mold", "debugging"]
draft = true
showTableOfContents = false
+++

- add `-gsplit-dwarf` during compilation
- add `--separate-debug-info` as linker argument to `mold` (requires >=2.33)
- add `--compressed-debug-info` to reduce file size (requires >=2.37 to work with `--separate-debug-info`)
- add `--gdb-indx` for faster loading times in `gdb` (requires `-ggnu-pubnames` during compilation)
- this leads to faster building because the file IO is separated from compiling
- mold can detach the "write debug info process" and allow earlier execution of the executable
