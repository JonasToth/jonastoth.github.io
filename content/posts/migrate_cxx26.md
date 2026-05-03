+++
title = "Migrating a Toy Project to C++-26"
description = "C++26 is around, GCC-16 is released, lets see what the state of modules, contracts and the tooling ecosystem for C++ is right now."
date = 2026-05-02T10:00:00+02:00
type = 'post'
tags = ["cpp", "cpp-26", "cmake", "build-tooling", "gcc", "gcc-16", "clang", "clang-22", "clangd", "clangd-22"]
showTableOfContents = true
+++

This post provides a short experience report about my approach to migrating my [toy project `jt-computing`](https://github.com/JonasToth/jt-computing) to the latest features of C++-26.
Noone depends on the project,
I have full control over all aspects and thats why I can just fiddle around and see what works.
The `GNU` (`gcc`) and `LLVM` (`clang`) implementations of the compiler,
standard library and the rest of the ecosystem support the described features as of 2026-05-02 in their latest versions.

The project has only one external dependency,
`catch2` for tests and benchmarks.
Converting `jt-computing` to modules hopefully gives me a bit of experience and insights on how to actually do the build definition and basic mechanics that I can later apply to bigger real world projects.
My approach and experience may help you do the same for your toy project or even help with a production code base.

---

## Tooling Setup

I use two systems for coding,
my desktop PC with [gentoo Linux](https://gentoo.org) for customization and my laptop with [Fedora Linux](https://fedoraproject.org/).
Both allow me to install very recent version of the necessary programming tools:
- [`cmake-4.3`](https://cmake.org/)
- [`gcc-16`](https://gcc.gnu.org)
- [`mold-2.40`](https://github.com/rui314/mold) as linker for `gcc`
- [`nvim-0.11`](https://neovim.io/) and [`clangd-22`](https://clangd.llvm.org/)
- [`clang-22`](https://clang.llvm.org/), using the full LLVM stack, including [`libc++`](https://libcxx.llvm.org/) and [`lld`](https://lld.llvm.org/)

I prefer `gentoo` for the development tasks,
because it is easier to get the bleeding edge versions of all tools,
as one can usually install "from git".
For simpler day-to-day usage of the full LLVM stack,
I installed `clang` on gentoo with additional [compile time configuration](https://wiki.gentoo.org/wiki/LLVM/Clang) to use LLVM tools by default.
```
# File: /etc/portage/package.accept_keywords/development
sys-devel/gcc ~amd64

llvm-core/llvm-common ~amd64
llvm-core/llvm ~amd64
llvm-core/llvm-toolchain-symlinks ~amd64
llvm-core/clang-common ~amd64
llvm-core/clang ~amd64
llvm-core/clang-linker-config ~amd64
llvm-core/clang-toolchain-symlinks ~amd64
llvm-core/llvmgold ~amd64
llvm-core/lld ~amd64
llvm-core/lld-toolchain-symlinks ~amd64
llvm-core/lldb ~amd64
llvm-runtimes/clang-runtime ~amd64
llvm-runtimes/clang-stdlib-config ~amd64
llvm-runtimes/clang-rtlib-config ~amd64
llvm-runtimes/clang-unwindlib-config ~amd64
llvm-runtimes/compiler-rt ~amd64
llvm-runtimes/compiler-rt-sanitizers ~amd64
llvm-runtimes/libcxx ~amd64
llvm-runtimes/libcxxabi ~amd64
llvm-runtimes/libunwind ~amd64
llvm-runtimes/openmp ~amd64
dev-python/lit ~amd64
```
```
# File: /etc/portage/package.use/development
>=llvm-core/clang-common-22 default-libcxx default-lld default-compiler-rt
>=llvm-core/clang-linker-config-22 default-lld
>=llvm-runtimes/clang-runtime-22 default-lld default-libcxx default-compiler-rt default-lld
>=llvm-runtimes/clang-rtlib-config-22 default-compiler-rt
>=llvm-runtimes/clang-unwindlib-config-22 default-compiler-rt
>=llvm-runtimes/clang-stdlib-config-22 default-libcxx
>=llvm-runtimes/libunwind-22 static-libs

sys-libs/libunwind static-libs
virtual/zlib static-libs
sys-libs/zlib static-libs
```

## Project Layout

The repository and project layout was generated from a modern CMake template,
the original author I forgot (sorry!).
It is structured as follows:
- `include/` and `lib/` define and implement the library interface
- `include/` can be easily installed into a system by just copying the contents to the system headers directory
- `test/` is a discrete CMake project that implements unit and integration tests of the `lib/` components using `catch2`
- `bin/` separate directory for CLI tools that use `lib/`,
  in theory compilable independently with system installed `lib/` and `include/` components
- `cmake/` contains support code
- `build*/` build directories for the various compilers, ignored in `git`
```bash
$ tree -L1 bin lib include test
bin
├── calculate_fibonacci.cpp
├── CMakeLists.txt
├── collatz_chain.cpp
├── find_prime_numbers.cpp
├── ...
lib
├── container
├── core
├── crypto
└── math
include
└── jt-computing
test
├── CMakeLists.txt
└── lib
```
The project structure works well and helped me in the transition to modules.
I would like to redo the cmake definition of it,
as I find it a bit too verbose and messy.
Properly installing the project to a system library does not work either and I did not evaluate changes in that respect.

## Introducing `import std;`

Starting to use `import std;` is mostly a `cmake` change and requires an up-to-date toolchain.
It is mandatory to use `ninja` as the build system (e.g. with `cmake -B build_dir -S . -G Ninja`).
The standard must be changed to at least `C++23`,
I use `C++26` to enable even more features.
`cmake`'s support for importing the standard library module is experimental and requires setting a feature gate.
The proper value must be looked up in the [documentation of the corresponding `cmake` version](https://gitlab.kitware.com/cmake/cmake/-/blob/master/Help/dev/experimental.rst?ref_type=heads).
Note, that the feature gate must be enabled **before** your `project()` call.
```cmake
cmake_minimum_required(VERSION 4.2)
set (CMAKE_EXPERIMENTAL_CXX_IMPORT_STD d0edc3af-4c50-42ea-a356-e2862fe7a444)

project("JTComputing" VERSION 0.1.0 LANGUAGES CXX)

set (CMAKE_CXX_STANDARD 26)
set (CMAKE_CXX_MODULE_STD ON)
set (CMAKE_CXX_EXTENSIONS ON)
set (CMAKE_CXX_SCAN_FOR_MODULES ON)
```
Setting these properties can be done on a per-target basis, too.
```cmake
function(jt_compile_setup target)
    set_target_properties(${target}
        PROPERTIES
            CMAKE_CXX_STANDARD cxx_std_26
            CMAKE_CXX_EXTENSIONS ON
            CMAKE_CXX_MODULE_STD ON
            CMAKE_CXX_SCAN_FOR_MODULES ON
    )
endfunction()
```
_Edited, see below for details_.
Building with `set (CMAKE_CXX_EXTENSIONS OFF)` works fine with `gcc`.
Doing the same with `clang` though leads to the following error:
```
FAILED: [code=1] CMakeFiles/JTComputing.dir/lib/math/Operations.cpp.o CMakeFiles/JTComputing.dir/jt.Math-Operations.pcm
/usr/lib/llvm/22/bin/clang++-22  -I<DIR>/jt-computing/include -I<DIR>/jt-computing/lib \
    -D_LIBCPP_ENABLE_EXPERIMENTAL=1 -g -std=c++26 -Wall -Wextra -W... -MD \
    -MT CMakeFiles/JTComputing.dir/lib/math/Operations.cpp.o \
    -MF CMakeFiles/JTComputing.dir/lib/math/Operations.cpp.o.d @CMakeFiles/JTComputing.dir/lib/math/Operations.cpp.o.modmap \
    -o CMakeFiles/JTComputing.dir/lib/math/Operations.cpp.o -c <DIR>/jt-computing/lib/math/Operations.cpp
error: GNU extensions was enabled in precompiled file 'CMakeFiles/__cmake_cxx26.dir/std.pcm' but is currently disabled
error: precompiled file 'CMakeFiles/__cmake_cxx26.dir/std.pcm' cannot be loaded due to a configuration mismatch with the current compilation [-Wmodule-file-config-mismatch]
<DIR>/jt-computing/lib/math/Operations.cpp:8:17: warning: using directive refers to implicitly-defined namespace 'std'
    8 | using namespace std;
      |                 ^

```
I am not aware how this mismatch of the C++ standard version occurs and I failed to find the root cause.
The module dependency states that the `compiler-frontend-variant` is `GNU`.
```bash
➜  build_clang git:(master) ✗ rg "frontend"
CMakeFiles/JTComputing.dir/CXXDependInfo.json
3:      "compiler-frontend-variant" : "GNU",

CMakeFiles/__cmake_cxx26.dir/CXXDependInfo.json
3:      "compiler-frontend-variant" : "GNU",

CMakeFiles/__cmake_cxx23.dir/CXXDependInfo.json
3:      "compiler-frontend-variant" : "GNU",

bin/CMakeFiles/collatz_chain.x.dir/CXXDependInfo.json
3:      "compiler-frontend-variant" : "GNU",
```
Maybe the `.pcm` file is generated with `GNU` extensions because of that.
For now, I keep the extensions activated to keep working with `gcc` and `clang`.

The following code transformation introduced `import std;`:
- Perform a project-wide string search for `#include <`, using [`telescope.nvim`](https://github.com/nvim-telescope/telescope.nvim).
- Highlight each found standard header using `Tab` in the picker and finally open the files in the quick-fix list via `Alt-q`.
- Cycle through all locations of the quick-fix list via `]q`, remove each standard include and add `import std;` after all `#include` directives.
- Compile and test.
    - I had to outcomment or remove a few standard macros like `assert` (see contracts) and `CHAR_BIT`.
    - I had no issues with C standard library functions used in the global namespace -- you can use `import std.compat;` in these situations.

This process is a good candidate for a `clang-tidy > modernize` check to automate the cumbersome work.

### Code Navigation Sidequest

Using the `gcc` generated `compile_commands.json` with modules lead to warnings about unknown arguments, stemming from modules flags.
Instead, I maintain a second build directory using the full `LLVM` toolchain and link the `compile_commands.json` from there into my source directory.
`clangd`'s modules support is still experimental, so it must be started with the `--experimental-modules-support` flag -- adjust your editors LSP setup accordingly.
After doing so, I received error messages about mismatching compiler versions when consuming the internal `std.pcm` files from `clangd`, resulting in this [bug report](https://bugs.gentoo.org/973221) for gentoo.
The issue was present on Fedora, too.
Digging around in the logs, LLVM code base and cmake definition and finally clearing my mind by touching grass I figured the problem out.
The root cause was managing `clangd` via `mason` in `nvim` that includes different VCS information than the system's `clang` compiler used to build the project.  
Resolving this issue is of course simple, not doing that.
It may be a recurring situation though,
as its quiet common to install `clangd` via your editor's/IDE's packaging instead as part of your system compiler distribution.  
_NOTE_: the produced module artifacts of the build are _not_ standardized and need recompilation for each compiler,
even between different compiler versions,
hence the warning.

### Syntax Highlighting Sidequest

Another `nvim` related issue was syntax highlighting.
The latest released [tree-sitter-cpp-0.23.4](https://github.com/tree-sitter/tree-sitter-cpp) is missing highlight groups for the module keywords, checked using `:TSHighlightCapturesUnderCursor`.
The upstream project already contains the necessary code on `master`, but lacks a release.
Apparently, the maintainers with the power-to-release are currently inactive ([Bug Comment](https://github.com/tree-sitter/tree-sitter-cpp/issues/341#issuecomment-3492960158)).
I created the fork [JonasToth/tree-sitter-cpp](https://github.com/JonasToth/tree-sitter-cpp) and added a `v9999` tag to the latest commit on master.
Installation on my system uses my [personal gentoo overlay](https://github.com/JonasToth/jonas-overlay/blob/751050527dcc1dd5c2af31fe00091b2d8c6d39d7/dev-libs/tree-sitter-cpp/tree-sitter-cpp-9999.ebuild):
- remove `cpp` from the `nvim` installed tree-sitter parsers (and/or `:TSUninstall cpp`)
- add `dev-libs/tree-sitter-cpp **` to `/etc/portage/package.accept_keywords/development`
- install by `emerge --sync jonas-overlay ; emerge --ask dev-libs/tree-sitter-cpp::jonas-overlay`

Finally, the code-writing experience is on par with good old header includes.

## Migration to C++ Modules

My approach was inspired from the blog posts of [Adrian Bühlmann](https://abuehl.github.io) and additional resources for module insights:
- [Converting an App to Modules](https://abuehl.github.io/2026/04/26/code-examples-from-an-app-using-modules.html)
- [Unneeded Recompilations when using Modules](https://abuehl.github.io/2026/04/20/unneeded-recompilations-when-using-partitions.html)
- [C++20 Modules: Best Practices from a User's Perspective](https://chuanqixu9.github.io/c++/2025/12/30/C++20-Modules-Best-Practices.en.html#use-module-implementation-partition-units-not-module-implementation-units-to-implement-interfaces)
- [Rubén Pérez's Blog Posts about Modules](https://anarthal.github.io/cppblog/modules4#clangd-modules)

### High Level Approach

1. Each `lib/` subdirectory becomes a module, e.g. `jt.Math` or `jt.Crypto`.
1. Each test is _part of the corresponding module_ as an internal partition ([Recommendation from chuanqixu9](https://chuanqixu9.github.io/c++/2025/12/30/C++20-Modules-Best-Practices.en.html#use-module-implementation-partition-units-not-module-implementation-units-to-implement-interfaces))
1. Individual test cases have a 1-to-1 mapping of file to executable -- this is maintained from the header-based version of the tests.
1. The `include/` directory becomes obsolete as the project will only consist of `.cpp` and `.cppm` files.

### Technical Implementation

First, the build definition needs to manage source files for executables and libraries via `target_sources()`.
`cmake` introduced the concept of a [`FILE_SET`](https://cmake.org/cmake/help/latest/command/target_sources.html#file-sets) to support modules.
```cmake
add_library(JTComputing)
target_sources(JTComputing
    PUBLIC
        FILE_SET cxx_modules TYPE CXX_MODULES FILES
        lib/container/Container.cppm
        lib/container/BitVector.cpp
        # ...
)
```
Each test exectuable gets a similar `target_sources > FILE_SET` to build the code as module.
```cmake
add_executable(BitVector_Tests)
target_sources(BitVector_Tests
    PRIVATE
        FILE_SET cxx_test_modules TYPE CXX_MODULES FILES
        test/lib/container/BitVector.cpp
)
target_link_libraries(BitVector_Tests
  PUBLIC
    Catch2::Catch2WithMain
    Catch2::Catch2
    JTComputing::JTComputing
)
add_test(NAME BitVector COMMAND BitVector_Tests)
```

### Migration Pseudo-Algorithm

- for each subcomponent in `lib/`
    - add a `lib/<Component>/<Component>.cppm` file that defines the module interface
    - move contents of `<Aspect>.hpp` file to the top of the corresponding `<Aspect>.cpp` file
    - add the `<Component>.cppm` and `<Aspect>.cpp` file to the `FILE_SET` in the cmake project
    - the `<Aspect>.cpp` file exports a module partition matching its name using `export module jt.<Component>:<Aspect>`
    - the partition is added to the `<Component>.cppm` as export using `export import :<Aspect>;`
    - delete all header includes of `<Aspect.hpp>` throughout the project and introduce the matching `import jt.<Component>;` if not already present in the user file (this can be done the same way as described for `import std;` above)
    - export the whole namespace definition using `export namespace jt::<Component>` in the new module file `<Aspect>.cpp`
    - delete `<Aspect>.hpp` and remove it from the cmake definitions if present
```cpp
// Example of lib/container/Container.cppm
export import jt.Core;

export module jt.Container;
export import :BitVector;
```
```cpp
// Example for lib/container/BitVector.cpp
export module jt.Container:BitVector;

import std;
import jt.Core;

export namespace jt::container {

class BitVector {
public:
  BitVector() = default;

  /// Construct a @c BitVector that has enough bits to represent @c value and
  /// assign @c values bit pattern to the individual bits.
  explicit BitVector(unsigned_integral auto value);

  /// Construct a @c BitVector with initial capacity of at least @c length bits.
  BitVector(usize length, bool initialValue);

  /// ... Implementation is at the bottom of the file.
};
}
```
- for each testcase in `test/`
    - add the global module fragment and include `catch2` headers
    - define a private module partition `module jt.<Component>:Test<Aspect>;`
    - ensure the file is part of the `FILE_SET` for the test executable
```cpp
// Example for test/lib/container/BitVector.cpp
module;

#include <catch2/catch_test_macros.hpp>

module jt.Container:TestBitVector;

import std;
// This import is necessary!
import jt.Container;

using namespace std;
using namespace jt;
using namespace jt::container;

TEST_CASE("BitVector Construction", "") {
  SECTION("Default Construction") {
    BitVector b;
    REQUIRE(b.capacity() == 0);
    REQUIRE(b.size() == 0);
  }
}
// ...
```

### Hindsight and Learnings

- Because a `<Component>` module is split into multiple partitions,
  the individual partitions may need additional `import :<OtherAspect>` imports internally to have all dependent components available.
- Try to perform the conversion `<Component>` wise, mixing includes in the global module fragment with imports until having a compiling state after each `<Component>`.
- Containing all code in `namespace`s made the `export` trivial.
- If the project is small enough, the conversion can be done in one sessions.
- The module structure of the project finally mirrors the previous header/implementation structure -- I like this property because refactorings and migrations can be sequenced with well defined inbetween states.
- The fully modularized structure can be adjusted further and follow current best practices:
    - breaking up build time dependencies by splitting interface and implementation into separate `.cpp` file ([Cpp Files to Break Build-Dependencies](https://abuehl.github.io/2026/04/23/cpp-files-still-help-breaking-dependencies.html))
    - improving the partition structure and potentially reducing the number of partitions
    - reducing the exported interface by selectively exporting classes and functions instead of the whole namespace in each file
- You are free to reshuffle all file related aspects of your code within a single module without breaking the module user.

### Convert to `using namespace std;` everywhere

Once the code base is fully modularized it is possible to use `using namespace std;` (or other namespace you use often) by default.
Because there is no textual header inclusion, the `using` directive doesn't bleed into other headers and implementations.
The [CppCoreGuidelines Rule SF.7](https://isocpp.github.io/CppCoreGuidelines/CppCoreGuidelines#sf7-dont-write-using-namespace-at-global-scope-in-a-header-file) exists for exactly that reason.

1. Add `using namespace std;` after each `import std;` or to other `using namespace ...;` sections already present.
1. Textual replacement of `'std::' => ''` and fixing of compiler errors, e.g. from introduced ambiguities.

This change resonates with my personal preference to not write `std::vector<std::string>` _everywhere_ but qualify standard functionality only if necessary.
It changes the look and feel of interface definitions and implementations, reduces "ceremony" for stating simple things and therefore improves the signal to noise ratio of the code.

## Using C++ Contracts

Starting point for contract assertions in the code base was the good old `assert();` macro, already used to state pre/post conditions and invariants.
The modularized code base _could not use_ `assert()` anymore, because `import std;` does not export macros.
Of course it would be possible to add `#include <cassert>` in the global module fragment, but given the small code size, temporary outcommenting `assert()` worked better.

`gcc-16` is currently the only shipping compiler with contracts support, so I decided to use macros after all to still compile with `clang`.
The following steps enabled contracts:
- reintroduce `include/` and add `include/jt-computing/core/Contracts.hpp`, included in the global module fragment of contracts users
- pass [`-fcontracts`](https://gcc.gnu.org/onlinedocs/gcc-16.1.0/gcc/C_002b_002b-Dialect-Options.html#index-fcontracts) and [`-fcontract-evaluation-semantic=enforce`](https://gcc.gnu.org/onlinedocs/gcc-16.1.0/gcc/C_002b_002b-Dialect-Options.html#index-fcontract-evaluation-semantic) to `gcc` through quick-and-dirty extension of the `target_compile_options()` and `target_link_options()` in `cmake`

I expect future releases of `cmake` to expose the evaluation semantic through typical target properties and invoke the compiler correctly if `C++26` is the target standard.
The assertion macros just pass through to the proper contracts keywords. (_Edited, see below_)
```cpp
// File: include/jt-computing/core/Contracts.hpp
#pragma once

#ifndef __cpp_contracts
#  define PRE(...)
#  define POST(...)
#  define CONTRACT_ASSERT(...)
#else
#  define PRE(...) pre(__VA_ARGS__)
#  define POST(...) post(__VA_ARGS__)
#  define CONTRACT_ASSERT(...) contract_assert(__VA_ARGS__)
#endif
```
```cpp
// Example include for normal consumer that does not define a module.
#include "jt-computing/core/Contracts.hpp"

import std;
import jt.Core;
import jt.Math;

// ...
class Dist {
public:
  explicit constexpr Dist(i32 d) PRE(d >= 0) : value{d} {}
// ...

```
```cpp
// Example include for a module using the contract macros.
module;

#include "jt-computing/core/Contracts.hpp"

export module jt.Container:BitVector;

// ...
```
Once `clang` supports contracts, the macros will disappear again using simple search-and-replace.

## Quick and Dirty Build-Time Comparison

A C++ post requires time measurements, so lets measure compile times of clean builds.
Please note, this is still a toy project.
I don't want you to make definite conclusions from these results.
Only `gcc-16` together with `mold-2.40` and `libstdc++` is measured!

### Header-based Project

The old header based project revision is tracked on the branch `jt-computing-old` at `3fb47bb32d36aad15943bf82bb64fac1f0901eac`.
I had to backport minor changes to the build definition to consume `Catch2` via `FetchContent` and don't perform module scanning.
Building `Catch2` is done independently and in total 3 clean builds were measured.
```bash
$ cmake \
    --fresh \
    -B build_timed \
    -S . \
    -G Ninja \
    -DCMAKE_LINKER_TYPE=MOLD \
    -DCMAKE_BUILD_TYPE=RelWithDebInfo
$ function measure() {
     cmake --build build_timed --target clean
     cmake --build build_timed --target Catch2 Catch2WithMain
     time cmake --build build_timed -- -j${1}
}
$ measure 8  # 3 times
$ measure 32 # 3 times
> ...
> [82/82] Linking CXX executable bin/percolation_power.x
```

| Cores  | Average Time in Seconds | Raw Values     |
| ------ | ----------------------- | -------------- |
| 32     | ~4.4s                   | 4.33 4.48 4.38 |
|  8     | ~7.0s                   | 6.97 6.97 6.99 |

### Module-based Project

The modules and contracts based revision is on `master` at `bbc4900b28f07e8d516566bb49008968ce7ad86f`
Building performs module scanning and producing the standard library module.
```bash
$ measure 8  # 3 times
$ measure 32 # 3 times
> ...
> [117/117] Linking CXX executable bin/percolation_power.x
```

| Cores  | Average Time in Seconds | Raw Values     |
| ------ | ----------------------- | -------------- |
| 32     | ~7.6s                   | 7.57 7.63 7.57 |
|  8     | ~9.5s                   | 9.45 9.46 9.44 |

_Edited, see below_.
Sadly, the modules version performs slower clean builds.
Happily I don't have a lot of time to increase the size of the project, so its fast to build anyway /s.

In all seriousness, I am surprised to see such a significant increase in compile time.
The build takes `35` steps more due to scanning for module definitions before acutal compilation happens.
Modularized compilation is not embarrassingly parallel anymore, as internal dependencies between module units must be resolved and require ordering of the build steps.
The effect of having the tests be part of the module is particularly interesting to investigate.
I want to revisit this point and see, if I can improve the build speed by adjusting my module structure.

Measuring incremental builds may restore the module's honor, but I want to finish the blog post 😅.
The incremental builds feel quiet fast and I suspect _not_ building the standard library module makes a big difference.
Incremental builds of the project suffer from unnecessary build time dependencies from the module structure.
With a bit more experience and optimization, I want to remeasure, including incremental builds.

## Conclusion

The migration was easier than I thought but harder than I hoped.
As you might imagine, the time consuming part was adjusting all the tooling, versions and libraries to have a good experience.
Finally changing the code was quiet fast.

I read about modules over the years to keep up to date but was honestly confused by the "new words" I had no connection to, like global module fragment.
The conversion gave me a clearer picture on what the different aspects of modules mean and how they interact with each other.
A migration of a bigger project seems achievable as long as the code is already "modularized" in spirit.
If I had to migrate a production code base, I would start with the foundational components and perform multiple end-to-end transformations the way I described above.
Quick-and-dirty `python` scripts would likely suffice to perform the bulk of the changes with manual interventions and fixups to keep the code compiling.
`clang-tidy` based introduction of `import std;` seems possible, maybe I can renew my rusty `clang-tidy` knowledge and hack on that a bit.

In my opinion, `clangd` support for modules is as important as compiler support to not regress into "toolless development".
The syntax highlighting issues in `nvim` are an annoyance, eventually resolved.
I am glad to see that the whole development ecosystem adopts modules and hope for acceleration with `gcc-16` providing better support.

Removing `std::` everywhere improved readability and is a welcome change to C++.
I am looking forward to not remember header names and accidentally missing includes that lead to compiler errors after toolchain updates.

Thank you for your effort to everyone involved in the continued evolution of C++ and its tools!

## Updates

I posted this article to [r/cpp](https://www.reddit.com/r/cpp/comments/1t2kkoh/migrating_a_small_c_code_base_to_c26_modules) and received valuable feedback.
The changes to the original article are listed here:
- Use `#ifndef __cpp_contracts` as the proper feature test macro for the contract macros (thanks to Jonathan Wakely)
- `gcc` does not need standard extensions for module builds to work, _but_ `clang` does for some (probably cmake-related) reason.
  This seems to be a bug I want to follow-up on (clarified by Jonathan Wakely).
- Multiple comments gave more detailed explanations how the module dependencies impact the build speed (thanks to u/kamrann_, u/James20k and u/javascript).
    - I was aware of this change in general, but was surprised it effected this small project negative.
      The 2 second difference doesn't matter.
      Just the percentage _feels_ bad -- which is why some things shouldn't be measured I guess 😂.
      I find the encapsulation modules provide more important though.
    - Logically, the reduced parallelism should hit harder for smaller projects. So this might be expected, but might be overblown by my bad modularization approach.
- Provide a better explanation why `using namespace std;` is worthwhile _for me_ (thanks to u/Fit-Departure-8426).
