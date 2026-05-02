+++
title = "Migrating a Toy Project to C++-26"
description = "This is a short experience report about migrating my C++ toy project `jt-computing` to the latest C++26 features and tooling."
date = 2026-04-02T10:00:00+02:00
type = 'post'
tags = ["cpp", "cpp-26", "cmake", "build-tooling", "gcc", "gcc-16", "clang", "clang-22", "clangd", "clangd-22"]
showTableOfContents = true
+++

This post provides a short experience report about my approach to migrating a [toy project](https://github.com/JonasToth/jt-computing) to the latest features of C++-26.
Noone depends on the project, I have full control over all aspects and thats why I can just fiddle around and see what works.
The `GNU` (`gcc`) and `LLVM` (`clang`) implementations of the compiler, standard library and rest support the descibed features as of 2026-05-02.

The project has only one external dependency, `catch2` for tests and benchmarks.
Converting the toy project to modules hopefully gives me a bit of experience and insights on how to actually do the build definition and basic mechanics that I can later apply to bigger real world projects.
My approach and experience may help you do the same for your toy project or even help with a production code base.

---

## Tooling Setup

- `gentoo` and `fedora` => easy to retrieve latest versions of toolchains and tools
- `cmake-4.3`
- no issues with `gcc-16`, allowing the introduction of `contracts`
- `mold-2.40` as linker for `gcc`
- `nvim` and `clangd` (first managed via `mason`, now managed via system versions)
- `clang-22` and `clangd-22` as secondary compiler and LSP for code navigation and diagnostics
- `clang-22` uses the full `LLVM` stack, including `libc++` and `lld` on gentoo
  - `/etc/portage/package.accept_keywords/development` => enable latest version of LLVM

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
  - `/etc/portage/package.use/development` => set use flags `default-libcxx default-lld default-compiler-rt` for LLVM components
```
# File: package.use/development
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

- the `cmake` project is separated into components that are in theory installable
    - `include/` and `lib/` define and implement the library interface
    - `include/` is structued such that it can be easily installed into a system by just copying the contents to the system headers directory
    - `test/` is a discrete CMake project that implements unit and integration tests of the `lib/` components using `catch2`
    - `bin/` separate directory for CLI tools that use `lib/`, in theory compilable independently with system installed `lib/` and `include/` components
    - `cmake/` contains support code
    - `build*/` build directories for the various compilers, ignored in `git`
```bash
$ tree -L1 bin lib include test
bin
├── calculate_fibonacci.cpp
├── CMakeLists.txt
├── collatz_chain.cpp
├── find_prime_numbers.cpp
├── generate_percolation_plot.bash
├── percolation_power.cpp
├── plot_graph.gnuplot
├── sha256sum.cpp
├── shortest_path.cpp
└── toy_rsa.cpp
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
- the project structure itself works well
- I would like to redo the cmake definition of it, as I find it imprecise and messy

## Introducing `import std;`

- change C++ standard in CMake project to `C++26`
- enable experimental support for `import std;` via the `CMAKE_EXPERIMENTAL_CXX_IMPORT_STD` feature gate
  - lookup the correct value for the gate in the [cmake repo for your version](https://gitlab.kitware.com/cmake/cmake/-/blob/master/Help/dev/experimental.rst?ref_type=heads)
  - add `set (CMAKE_EXPERIMENTAL_CXX_IMPORT_STD d0edc3af-4c50-42ea-a356-e2862fe7a444)` **BEFORE** your `project()` call in the toplevel `CMakeLists.txt`
- go through each file and replace all `#include <SOMETHING>` from the STL with a single `import std;`
- compile and test

### LSP Sidequest

- to have the best `clangd` support, I use a `clang` build as the source for my `compile_commands.json`
- adjusted `clangd` invocation of editor to add the `--experimental-modules-support` flag when started
- received error messages about mismatching compiler versions when consuming the internal `std.pcm` files from `clangd` resulting in this [bug report](https://bugs.gentoo.org/973221) for gentoo
- the issue was managing `clangd` via `mason` in `nvim` that includes different VCS information then the `clang` compiler used to build the project
- _NOTE_: the produced module artifacts of the build are _not_ standardized and need recompilation for each compiler, even between different compiler versions

### Tree-Sitter Sidequest

- the currently released [tree-sitter-cpp](https://github.com/tree-sitter/tree-sitter-cpp) is missing highlight groups for the module keywords (according to `:TSHighlightCapturesUnderCursor`)
- the project already contains the necessary code, but lacks a release, apparently because the maintainers with that right are inactive [Bug Comment](https://github.com/tree-sitter/tree-sitter-cpp/issues/341#issuecomment-3492960158)
- I created a fork [JonasToth/tree-sitter-cpp](https://github.com/JonasToth/tree-sitter-cpp), added a `v9999` tag to the latest commit on master
- use this version in [jonas-overlay](https://github.com/JonasToth/jonas-overlay/blob/751050527dcc1dd5c2af31fe00091b2d8c6d39d7/dev-libs/tree-sitter-cpp/tree-sitter-cpp-9999.ebuild)
- remove `cpp` from the `nvim` installed parsers
- add `dev-libs/tree-sitter-cpp **` to `/etc/portage/package.accept_keywords/development`
- install `emerge --sync jonas-overlay ; emerge --ask dev-libs/tree-sitter-cpp::jonas-overlay`

## Migration to C++ Modules

- Inspiration from the Blog Posts of [Adrian Bühlmann](https://abuehl.github.io) and additional resources for general knowledge:
  - [Converting an App to Modules](https://abuehl.github.io/2026/04/26/code-examples-from-an-app-using-modules.html)
  - [Unneeded Recompilations when using Modules](https://abuehl.github.io/2026/04/20/unneeded-recompilations-when-using-partitions.html)
  - [C++20 Modules: Best Practices from a User's Perspective](https://chuanqixu9.github.io/c++/2025/12/30/C++20-Modules-Best-Practices.en.html#use-module-implementation-partition-units-not-module-implementation-units-to-implement-interfaces)
  - [Rubén Pérez's Blog Posts about  Modules](https://anarthal.github.io/cppblog/modules4#clangd-modules)
- increase `cmake` required version to `4.2`
- introduce `FILE_SET` in cmake definition of the targets using `target_sources()`
```cmake
target_sources(JTComputing
    PUBLIC
        FILE_SET cxx_modules TYPE CXX_MODULES FILES
        lib/container/Container.cppm
        lib/container/BitVector.cpp
        # ...
)
```
- made `include/` directory obsolete, project consists only of `.cpp` and `.cppm` files
- each `lib/` subdirectory becomes a module, e.g. `jt.Math` or `jt.Crypto`
- each test is _part of the corresponding module_ as a partition and has access to all module interna ([Recommendation from chuanqixu9](https://chuanqixu9.github.io/c++/2025/12/30/C++20-Modules-Best-Practices.en.html#use-module-implementation-partition-units-not-module-implementation-units-to-implement-interfaces))
- individual test cases have a 1-to-1 mapping of file to executable
- each test exectuable get a similar `target_source` `FILE_SET` to build them as modules
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
    - delete all header includes of `<Aspect.hpp>` and introduce the matching `import jt.<Component>;` if not already present in the user file
    - in the new module file `<Aspect>.cpp` just export the whole namespace definition using `export namespace jt::<Component>`
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
    - add the global module fragment to include `catch2` headers
    - define a private module partition `module jt.<Component>:Test<Aspect>;`
    - ensure the file is part of the `FILE_SET` for the test executable
```cpp
// Example for test/lib/container/BitVector.cpp
module;

#include <catch2/catch_test_macros.hpp>

module jt.Container:TestBitVector;

import std;
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

- because a `<Component>` module is split into multiple partitions, the partitions may need additional `import :<OtherAspect>` imports internally to have all dependent components available
- the whole conversion requires repeated compilations and fixing of errors, missing definitions and similar fixes
- try to perform the conversion `<Component>` wise mixing includes in the global module fragment with imports until, having a compiling state after each `<Component>`
- if the project is small enough, this can be done in one sessions and all parts are modularized at once
- the module structure of the project finally mirrors the previous header/implementation structure
- the modularized structure can now be adjusted further to work well and follow current best practices
    - breaking up build time dependencies by splitting interface and implementation into separate `.cpp` file ([Cpp Files to Break Build-Dependencies](https://abuehl.github.io/2026/04/23/cpp-files-still-help-breaking-dependencies.html))
    - improving the partition structure and potentially reducing the number of partitions
    - reducing the exported interface by selectively export classes and functions instead of the whole namespace in each file

### Convert to `using namespace std;` everywhere

Once the code base is fully modularized it is possible to use `using namespace std;` everywhere and have it as a default.
Because there is no textual header inclusion the `using` directive doesn't bleed into other headers and implementations.
Just add this line after each `import std;` or to other `using namespace ...;` sections already existing.
Then perform a textual replacement of `'std::' => ''` and fix compiler errors.

## Using C++ Contracts

- the project already use `assert();` to state invariants and pre/post conditions
- conversion of the macro based `assert()` to contracts was done after the modularization
- during the modularization, the `assert()` macros where commented out, because `import std;` does not provide the `assert()` macro
- C++26 contracts are only supported by `gcc-16`, so conditional compilation required to still support `clang` for now
- this is easiest done via macros that are `#include`d
- added in `include/jt-computing/core/Contracts.hpp` and included in the global module fragment of users
```cpp
#pragma once

#ifdef __clang__
#  define PRE(...)
#  define POST(...)
#  define CONTRACT_ASSERT(...)
#else
#  define PRE(...) pre(__VA_ARGS__)
#  define POST(...) post(__VA_ARGS__)
#  define CONTRACT_ASSERT(...) contract_assert(__VA_ARGS__)
#endif
```
- users of the contracts need to do the following
```cpp
// EXAMPLE for normal consumer that does not define a module.
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
- if used in a module defintion or implementation, the global module fragment must be defined
```cpp
module;

#include "jt-computing/core/Contracts.hpp"

export module jt.Container:BitVector;

// ...
```

## Quick and Dirty Build-Time Comparison

- not the most scientific measurements, but you get a feeling
- again, this is a toy project, it already compiled quiet fast due to limited extent ;)

### Header-based Project

- based on the branch `jt-computing-old` at `3fb47bb32d36aad15943bf82bb64fac1f0901eac`
- compile `Catch2` outside of the measurement and take 3 measurements
- `gcc-16` and `mold`
- disabled module scanning and `import std;` is not in effect
```bash
$ cmake \
    --fresh \
    -B build_timed \
    -S . \
    -G Ninja \
    -DCMAKE_LINKER_TYPE=MOLD \
    -DCMAKE_BUILD_TYPE=RelWithDebInfo
$ cmake --build build_timed --target clean
$ cmake --build build_timed --target Catch2 Catch2WithMain
$ time cmake --build build_timed -- -j<Cores>
> ...
> [82/82] Linking CXX executable bin/percolation_power.x
```

| Cores  | Average Time in Seconds | Raw Values     |
| ------ | ----------------------- | -------------- |
| 32     | ~4.4s                   | 4.33 4.48 4.38 |
|  8     | ~7.0s                   | 6.97 6.97 6.99 |

### Module-based Project

- based on the `master` branch at `bbc4900b28f07e8d516566bb49008968ce7ad86f`
- again, compile `Catch2` 
- again `gcc-16` and `mold`
- includes modules scanning and building of the standard library module
```bash
$ cmake \
    --fresh \
    -B build_timed \
    -S . \
    -G Ninja \
    -DCMAKE_LINKER_TYPE=MOLD \
    -DCMAKE_BUILD_TYPE=RelWithDebInfo
$ cmake --build build_timed --target clean
$ cmake --build build_timed --target Catch2 Catch2WithMain
$ time cmake --build build_timed -- -j<Cores>
> ...
> [117/117] Linking CXX executable bin/percolation_power.x
```

| Cores  | Average Time in Seconds | Raw Values     |
| ------ | ----------------------- | -------------- |
| 32     | ~7.6s                   | 7.57 7.63 7.57 |
|  8     | ~9.5s                   | 9.45 9.46 9.44 |

Sadly, the modules version performs slower clean builds.
Happily I don't have a lot of time to increase the size of the project, so its fast to build anyway /s.

## Conclusion

- migration was easier than I thought
- I read a lot about modules to keep up to date but was honestly confused by the new words I had no real world connection to
- now I have a clearer picture on what the different aspects of modules mean and how they interact with each other
- a migration of a bigger project seems achievable as long as the code is already "modularized" in spirit
- it seems most profitable to start with the foundational components or libraries of a bigger project and convert in multiple sequential transformations
- `clangd` support for modules is as important as compiler support to not regress into "toolless development"
- syntax highlighting for modules and exports is not correct in `nvim` but I consider it a minor issue
- it seems that the whole development ecosystem I use slowly adopts modules, but its not plug-and-play with the desired developer experience
