+++
title = "Reducing CMake Generation Time by 5x"
description = "Building many libraries utilizing conan2 as package manager can lead to long CMake generation times by default. This issue can be circumvented."
date = 2025-03-10T10:00:00+02:00
type = 'post'
tags = ["conan", "cmake", "cpp", "build-tooling"]
showTableOfContents = false
+++
- the following post explains how and why cmake generation time under linux can be drastically improved
  by changing one option
- `conan2` for dependency management
- packages are installed in separate directories and then added to each target as link directory
- packages live in the `conan cache`
- `CMake` by default creates executables that can be run from the build directory
- this is done by setting the `RPATH` to paths in the build directory / `conan cache`
