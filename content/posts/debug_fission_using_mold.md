+++
title = "A Faster Way to Perform Debug Fission"
description = "Debug Fission splits debug information of ELF files into separate debug files that can be handled independently. Mold improved the workflow for this approach"
date = 2025-03-10T10:00:00+02:00
type = 'post'
tags = ["cpp", "build-tooling", "mold"]
showTableOfContents = false
+++
- add `--separate-debug-info` as linker argument to `mold` (requires >=2.33)
- add `--compressed-debug-info` to reduce file size
- this leads to faster building because the file IO is separated from compiling
- mold can detach the "write debug info process" and allow earlier execution of the executable
