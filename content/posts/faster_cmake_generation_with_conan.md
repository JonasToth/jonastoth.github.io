+++
title = "Reducing CMake Generation Time by 10x"
description = "Building many libraries utilizing conan2 as package manager can lead to long CMake generation times by default. This issue can be circumvented."
date = 2025-04-06T10:00:00+02:00
type = 'post'
tags = ["conan2", "cmake", "cpp", "build-tooling"]
showTableOfContents = false
+++

Do you know how programs become actually executable, in the sense of loading and running machine instructions, in Linux and how `CMake` ensures programs can be run from the build directory?
Why does a `CMake` project with hundreds of shared object files lead to bad generation time of `CMake` and why does `conan2` make it even worse?

---

[`conan2`](https://conan.io/) established itself as package manager for C++.
It provides an easy way to add dependencies and build tooling for a C++ project that integrates into existing build systems like `CMake + Ninja`.
`conan2` supports _profiles_ that control build settings of projects, including consuming them as binary artifact or building from source.
This leads to an extremely flexible approach to dependencies, build tooling and the whole end-to-end process from writing C++ to shipping final binaries.

`CMake` on the other hand is the tried and tested build system generator everyone hates but uses it anyway because `CMake` gets everything working somehow.

As is tradition in C++, this flexibility comes with the price tag of "you got to know what you are doing" once things get more complex.
So, why is `CMake` generation time bad in a `conan2` project with many shared object files?

### TL;DR: What do I have to change in my CMake Build?

Globally set the following properties of your project in `CMake`:
```cmake
# Collect all ELF-files in single directories instead spread across the build tree.
set(CMAKE_RUNTIME_OUTPUT_DIRECTORY "${CMAKE_BINARY_DIR}/bin")
set(CMAKE_LIBRARY_OUTPUT_DIRECTORY "${CMAKE_BINARY_DIR}/lib")

# If you don't need to run the executable from the build directory, you can even
# disable RPATH addition.
set(CMAKE_BUILD_WITH_INSTALL_RPATH ON)
set(CMAKE_SKIP_BUILD_RPATH ON)
```

The impact on a big (closed source) project that has hundreds of dynamic libraries is dramatic and almost 10x :innocent:.

| Modification       | CMake Times           | Size of `build.ninja` |
| :----------------: | :-------------------- | :-------------------- |
| Naive Setup        | `6.6s` configuration  | `138M`                |
|                    | `108.1s` generation   |                       |
| Output Directories | `5.8s` configuration  | `105M`                |
|                    | `14.0s` generation    |                       |
| Output Directories | `5.6s` configuration  | `104M`                |
| + Skipping RPATH   | `12.6s` generation    |                       |

Additionally, not blindly copy-pasting the `conan2` provided list of targets to link, obviously reduces generation time again.
E.g. `boost::boost` is provided by `conan2` and includes all boost libraries you specified as dependency.
Adding only `Boost::program_options` via `target_link_libraries` instead of `boost::boost` for all used libraries gave an additional factor 2-3 speedup from the original `300s` generation time.
This was more complicated in the referenced project due to the project structure and previous build approach and therefore not universally interesting.

---

### Why are these changes so effective?

[CMAKE_BUILD_WITH_INSTALL_RPATH](https://cmake.org/cmake/help/latest/variable/CMAKE_BUILD_WITH_INSTALL_RPATH.html) and [CMAKE_SKIP_RPATH](https://cmake.org/cmake/help/latest/variable/CMAKE_SKIP_RPATH.html#variable:CMAKE_SKIP_RPATH) control if `CMake` adds runtime loading information to binaries.
The `RPATH` is a property of `ELF` files that contains paths that point to dependent shared object files.
Setting the `RPATH` makes it _unnecessary_ to include library paths in the `$LD_LIBRARY_PATH` and allows an override to prefer custom over system provided libraries.
For more details I recommend [this blog post](https://duerrenberger.dev/blog/2021/08/04/understanding-rpath-with-cmake/).
`CMake` defaults these properties to include the `RPATH`, making binaries executable from the build directory.
This default is sane and leads to a simple development cycle.

The second ingredient to bad generation time is a high number of `ELF` files _distributed_ over the build tree.
By default, `CMake` mirrors the source directory structure and puts every `ELF` file in the nested path it was defined in.
This is again a sane default providing familiar structure in a project.

`conan2` has to build all dependencies in separate directories to allow switching between variations, e.g. a debug build vs a release build, of the same dependency.
Every variation gets one directory in the [conan cache](https://docs.conan.io/2/reference/commands/cache.html).
A normal development build requires sourcing `conanbuild.sh` and `conanrun.sh` to add the necessary directories to the `$PATH` and `$LD_LIBRARY_PATH`.
Again, sane and necessary. `conan2` shall not pollute the global environment!

The combination of these factors slows `CMake` down.
Build system generation determines the paths of each `ELF` file and dynamic dependency, all leading to different directories!
These directories are concatenated and added to each targets `RPATH`.
The corresponding link commands contain one screen full of paths and are barely digestable.
Not doing this by setting `CMAKE_RUNTIME_OUTPUT_DIRECTORY` and `CMAKE_LIBRARY_OUTPUT_DIRECTORY` saves `CMake` a ton of work.

I tried to use `cmake --profiling-format=google-trace --profiling-output=cmake-generation.json` for profiling.
The resulting file was too big to load into `chrome://tracing` and [perfetto.dev](https://perfetto.dev).
By manually inspecting the trace I got the impression, that the dynamic evaluation of each targets `RPATH` properties lead to the massive slowdown.
It certainly creates noise and deep call stacks in the bits and pieces I could analyze.

### Conclusion

Noone is to blame for the bad performance and with some experience in defining builds under Linux the issue seems obvious.
It still came as a surprise to me _how much_ impact the `RPATH` property has on generation time.
Maybe `CMake`'s code path could be optimized, but at the end of the day, just adjust the defaults in your build.

`conan2` provides [deployers](https://docs.conan.io/2/reference/extensions/deployers.html) that copy dependencies around.
Collecting libraries into one directory is a small optimization to try out.
Using `source conanbuild.sh` and `source conanrun.sh` removes the need for setting the `RPATH` the same way, too.
Play around with it and keep optimizing your `CMake` project.
