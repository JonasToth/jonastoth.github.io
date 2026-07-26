+++
title = "rtest - A C++-26 reflection based testing library"
description = "Writing software tests in C++ is commonly done through macro-based test definitions. This can change with C++-26 reflection and the `rtest` library is a design study for a macro-free testing library similar to pythons `unittest`."
date = 2026-07-27T22:30:00+02:00
type = 'post'
tags = ["cpp", "cpp-26", "testing", "reflection"]
showTableOfContents = true
+++

A new `C++` programmer approaches you and asks what the proper way of writing tests is.
You respond with "thats a good question, let me show you an example",
open a new `.cpp` file and type
```c++
#include <gtest/gtest.h>

TEST(Example, myTestExample) {
    ASSERT_EQ(100, 200);
}
```
Adding this file to an existing executable, building and running it shows the well known `gtest` output of a failed test.
You see confused eyes, expressing a "where is the `TEST` section on `cppreference.com`" look.
You respond a with reassuring "this is the way" look, but try to avoid a preprocessor introduction.
After some time the newling gets used to "the way".

## "The Way" until C++-26

Why is this "the way"?

Naming and defining a test case should happen exactly once to not introduce inconsistencies between the code and e.g. test filtering or error reporting.
In order to do that, testing libraries rely on [macro expansion](https://cppreference.com/cpp/preprocessor/replace) to stringify `Example` and `myTestExample` using the preprocessor operator `#`.
At the same time, the macros introduce classes and member functions with corresponding names to attach the test code to.
Multiple testing libraries define different "dialects" for this approach, see [Google Test](https://google.github.io/googletest/), [Catch2](https://github.com/catchorg/Catch2) or [doctest](https://github.com/doctest/doctest) as examples.
Secondary reasons for macro usage are printing the exact assertion expression and reporting the source information for faster identification of the failing condition.

An honorable mention for a macro free testing library is [boost-ext/ut](https://github.com/boost-ext/ut).
`ut` creates a "testing sub-language" using user defined literals.
I am afraid, an `ut` introduction to a new `C++` programmer is even more confusing than the macro-approaches.

## Test Definition and C++-26 Reflection

Other languages, other than `C`, tend to provide an easier option for test definitions.
`C++` gets flag for many things and having such a convoluted way to defining tests is a minus in terms of developer appeal.

What if instead something like _this_ becomes "the way"?
```c++
#include <rtest/rtest.h>
struct MyClassTest : rtest::TestSuite {
  void testSetup() {
    MyClass object;
    assertTrue(object.empty());
  }
};
int main(int argc, char** argv) { return rtest::execute(MyClassTest{}); }
```
```bash
$ cmake --build build && ./build/TestCase
> [=] Running with following config
> [=]  Random Seed: 0
> [=] Repeat Count: 1
> [=]   Inclusions: []
> [=]   Exclusions: []
> [=]     Parallel: true
> [=] Executing     1 test suites
> [*] Running test suite MyClassTest
>  RUN Setup
> PASS Setup (took 655ns)
> [*] TestSuite MyClassTest PASSED (took 20us)
> [*] tests:     1 passed
> [*] tests:     0 failed
> [*] tests:     0 skipped
> [=] ------------------------------------------------------------
> [=] finished execution (took 49us)
> [=] test suites:     1 passed
> [=] test suites:     0 failed
```
The interface is inspired by [Pythons `unittest` module](https://docs.python.org/3/library/unittest.html).
I claim that basic experience in _any_ procedural or object oriented programming language provides the necessary intuition to follow whats going on here.
Codewise, its functions and `struct` -- `C++` constructs you needed for the feature implementation already.
Handwaving `: rtest::TestSuite` and `rtest::execute()` as "marking" this as tests and then running it seems fair to me.

This example is compiling `C++-26` code and implemented in my [rtest Library](https://codeberg.org/JonasToth/rtest).
The rest of this blog post explains the approach and central aspects I needed to understand and implement.
Please note that I consider this a _design study_ and not the new target library for all your projects.

## Basic Architecture

Above all, writing a test shall be straight forward and look like "normal code".
This is achieved through the central `rtest::TestSuite` class, that has a `run()` method to execute all of its test cases.
There is no global state and no automatic registration of tests.
Instead, a bunch of `TestSuite`-objects are passed to the `rtest::Executor` that calls each objects `run()` method.
The executor can be configured and `rtest::execute()` is a convenience wrapper as simple default.

`rtest::TestSuite` itself provides access to test assertion methods.
Using the [Mixin Design Pattern](https://en.wikipedia.org/wiki/Mixin) seemed like a good fit for an extensible approach.
Hence, the assertion mechanism is loosely coupled with the `rtest::TestSuite` and explained down below.

The composition of these individual pieces heavily relies on templates and compile time computation.
Even methods like `setUp()` and `tearDown()` are _not_ `virtual` as the concrete test suite classes are used directly.
The only thing `virtual` is the inheritance relationship for assertion providers to ensure exactly one pair of assertion counters.

## Test-Suite Generation and Execution (The Reflection Part)

During compile time, `TestSuite::run()` reflects on all `public` member functions and collects those starting with `test`.
For each collected test `t`, a lambda performing the `t.setUp(); t.testCase(); t.tearDown();` calls is stored in a `std::function`.
All collected tests and their meta data are stored in a runtime `std::vector`.
The assertion failure count is taken before and after each test execution and an increase manifests as test failure.
This post uses simplified examples condensed from various points of `rtest`s development.

### Support Code

The reflection part benefits massively from a helper class to manage individual `constexpr` contexts,
limiting the lifetime of `constexpr vector/string` objects.
I took this approach from the [swordfaith/reflect ORM library](https://github.com/swordfatih/reflect) and adjusted it for my use case.
```c++
#include <type_traits>
#include <meta>
#include <print>
using namespace std;
template <typename Type> class ClassType {
public:
  using PlainType = remove_cvref_t<Type>;
  // NOTE: reflection turning a concrete type -> meta::info
  consteval static meta::info info() { return ^^Type; }
  consteval static auto name() {
    return define_static_string(meta::identifier_of(info()));
  }
  consteval static auto members() {
    constexpr auto context = meta::access_context::current();
    auto members = meta::members_of(info(), context);
    return define_static_array(members);
  }
};
struct ReflectOnMe {};
int main(int argc, char** argv) {
  print("{}", string_view{ClassType<ReflectOnMe>::name()});
}
```
[Godbolt Link](https://godbolt.org/z/8bzo96Y9P)

The important pieces I was missing at the beginning were the [`define_static_array/string`](https://cppreference.com/cpp/meta/define_static_string) functions.
They form a "worm hole" that takes `consteval` results, embeds them in the final executable and makes it accessible for classical runtime code.
Without structuring this process well, programming with reflection was honestly annoying and I made slow progress.

### Test Case Discovery and Gathering

The `members()` function is the next important building block to automatically determine tests.
Again, `consteval` computation needs to occur, gathering all desired member functions in a vector that can be iterated at runtime.
This is done in a callback-style using templated lambdas.
```c++
// ...
template <typename Type> class ClassType {
  // ...
  template <typename Function>
  constexpr static void forEachPublicMethod(Function &&function) {
    // NOTE: 'members()' triggers reflection under the hood.
    template for (constexpr auto member : ClassType<PlainType>::members()) {
      if constexpr (!meta::is_public(member) || !meta::is_function(member) ||
                    !meta::has_identifier(member) ||
                    meta::is_special_member_function(member))
        continue;
      function.template operator()<member>();
    }
  }
};

/// Helper for stringification of reflected entities.
/// ReflectedMember is a std::meta object.
template <auto ReflectedMember> consteval auto displayName() {
  return define_static_string(meta::display_string_of(ReflectedMember));
}
/// Store a test name and member function pointer to call the test.
template <typename ConcreteTestSuite> struct SingleTestInfo {
  string_view name = "";
  void (ConcreteTestSuite::*testMethodPointer)() = nullptr;
};
/// Collect all 'test*' member functions. Implementation simplified to avoid the
/// parsing noise.
template <typename ConcreteTestSuite> auto getAllTestMethods() {
  using ReflectedType = remove_cvref_t<ConcreteTestSuite>;

  vector<SingleTestInfo<ConcreteTestSuite>> testMethods;
  // Extend the vector 'testMethods' in each callback.
  ClassType<ReflectedType>::forEachPublicMethod([&]<auto reflectedMethod>() {
    constexpr auto methodName = string_view{displayName<reflectedMethod>()};
    if constexpr (methodName.contains("::test")) {
      testMethods.emplace_back(SingleTestInfo<ReflectedType>{
          .name = methodName,
          // NOTE: meta::info -> actual method entity whose pointer is taken.
          .testMethodPointer = &[:reflectedMethod:],
      });
    }
  });
  return testMethods;
}

// Sketch out test execution.
struct ReflectOnMe {
  void checkIgnore() { println("Not this one"); }
  void testOne() { println("Testing conditions"); }
};
int main(int argc, char** argv) {
  ReflectOnMe testSuite{};
  auto tests = getAllTestMethods<ReflectOnMe>();
  for (auto& t: tests) {
    println("{}", t.name);
    invoke(t.testMethodPointer, testSuite);
  }
}
```
[Godbolt Link](https://godbolt.org/z/fbh7Yn5o8)

`displayName()` is a `consteval` function working on [`std::meta::info`](https://cppreference.com/cpp/meta/info) objects.
The given example does not parse the method name to extract a human readable test name.
It is a bit cumbersome and adds no conceptual value.
The real `rtest` implementation tries to be smarter and assigns a better `name` to each method pointer.

Note, that the reflection is not enough to actually execute the tests.
It is mandatory to have an object of the test-suite class constructed during runtime.
This makes it a natural fit for parametrized tests to just take a `std::vector` of objects and `std::invoke()` each test method on each object, too.

### Transformation to Generic Test-Suite Type

Effectively, the previous `main` function needs to be packaged better.
You can see that `getAllTestMethods<ReflectOnMe>()` is type specific and the rest generic.
Templating a class, adding a `run` method as `main` substitute and instantiating it with the concrete test suite (known as the [Curiously Recurring Template (CRTP) Pattern](https://en.cppreference.com/cpp/language/crtp)) is a direct refactoring.
Using a variadic template function with arbitrary many test suite arguments serves as basic runner.
`main` is then just a single function call that sets the appropriate exit code of the test process.
```c++
// ... getAllTestmethods ...

template <typename ConcreteTestSuite> class TestSuite {
public:
  // returns true if all tests succeeded and false otherwise.
  bool run() {
    using ReflectedType = remove_cvref_t<ConcreteTestSuite>;
    auto tests = getAllTestMethods<ReflectedType>();
    bool anyFailure = false;

    // Assertion Handling is left out in the example code.
    for (auto& t: tests) {
        int const failedAssertionsBefore = 0/* = getFailedAssertions()*/;
        println("[*] {}", t.name);
        invoke(t.testMethodPointer, static_cast<ConcreteTestSuite*>(this));
        int const failedAssertionsAfter = 1/* = getFailedAssertions()*/;
        anyFailure |= (failedAssertionsAfter > failedAssertionsBefore);
    }
    return !anyFailure;
  }
};
template <typename... Tests>
int execute(Tests &&...tests) {
    bool testSuiteFailure = false;
    template for (auto &testSuite : {tests...}) {
        testSuiteFailure &= testSuite.run();
    }
    return testSuiteFailure ? EXIT_FAILURE : EXIT_SUCCESS;
}
// Sketch out test execution.
struct ReflectOnMe : TestSuite<ReflectOnMe> {
  void checkIgnore() { println("Not this one"); }
  void testOne() { println("Testing conditions"); }
};
struct SecondTest : TestSuite<SecondTest> {
    void testTwo() { println("Another condition"); }
};
int main(int argc, char** argv) {
  return execute(ReflectOnMe{}, SecondTest{});
}
```
[Godbolt Link](https://godbolt.org/z/vMG5Tv175)

Using [deducing this](https://en.cppreference.com/cpp/language/member_functions#Explicit_object_member_functions) gets rid of the weird self-referential template inheritance.
```c++
// ...
class TestSuite {
public:
  // This is a templated method and 'self' has the
  // type of the class this method is called through.
  bool run(this auto&& self) {
    using ReflectedType = remove_cvref_t<decltype(self)>;
    auto tests = getAllTestMethods<ReflectedType>();
    // ...
  }
};
// ...
struct ReflectOnMe : TestSuite {};
struct SecondTest : TestSuite {};
```
[Godbolt Link](https://godbolt.org/z/bYYPr1vE9)

Et voilà -- macro free stringification and execution for a testing library.

## Assertions

`rtest`s assertions follow the [mixin design pattern](https://en.wikipedia.org/wiki/Mixin).
The minimal class `rtest::TestAssertions` holds two integers for passed and failed assertions and corresponding getters/setters.
Every assertion provider _virtually_ inherits from `rtest::TestAssertions` to access the same counters accross different providers.
By convention,
an assertion should be a `protected:` `assert*` method,
increase the assertion counters and report failures together with the corresponding source location.

### Domain Specific Testing Vocabulary

By default, `rtest::TestSuite` inherits the basic set of assertions for comparison, container and exception tests.
Using more specialized assertions in a test suite is simple.
```c++
// Virtual inheritance is important, to not duplicate the assertion counters.
class IntegerAssertions : virtual public rtest::TestAssertions {
protected:
  // Just to make a point.
  bool assertIsPrime(
    unsigned long long number,
    std::source_location const& code=std::source_location::current()) {
    bool isPrime = primeCheck(number);
    if (isPrime) {
      passAssertion();
    } else {
      failAssertion();
      rtest::detail::printErrorExplanation(std:format("{} is not prime", number),
                                           code, "");
    }
    return isPrime;
  }
};

struct NumberTest : rtest::TestSuite, IntegerAssertions {
  void testPrimesToTen() {
    assertIsPrime(2);
    assertIsPrime(3);
    assertIsPrime(5);
    assertIsPrime(7);
  }
};
```
Introducing convenience base classes combining the inheritance relationships provide conciser definitions.
```c++
struct NumberTheoryTestSuite : rtest::TestSuite, IntegerAssertions {};
struct NumberTests : NumberTheoryTestSuite {
    // Define test cases...
};
```

### Individual Assertion Methods

The builtin comparison assertions are implemented as templated methods, constrained via concepts.
Each of those methods follows the same pattern, examplified with `assertLess`.
```c++
template <class T>
concept boolean_testable = std::convertible_to<T, bool>;
template <class T, class U>
concept less_comparable = requires(std::remove_reference_t<T> const &t,
                                   std::remove_reference_t<U> const &u) {
  { t < u } -> boolean_testable;
};
template <class T, class U> requires less_comparable<T, U>
struct compare_less {
  constexpr static std::string_view representation = "<";
  [[nodiscard]] constexpr bool operator()(T const &t, U const &u) const {
    return t < u;
  }
};

template <class T, class U> requires less_comparable<T, U>
bool assertLess(T const &value,
                U const &expected,
                std::string_view customMessage="",
                std::source_location const &code=std::source_location::current()) {
  return genericAssert(value, expected, compare_less<T, U>{}, customMessage,
                       code);
}
```
The generic programming aspects mirror the STLs functors and concepts.
Using [`std::less<T>`](https://en.cppreference.com/cpp/utility/functional/less)/[`std::ranges::less`](https://en.cppreference.com/cpp/utility/functional/ranges/less) and [`std::totally_ordered_with`](https://en.cppreference.com/cpp/concepts/totally_ordered) seemed unfitting as they require overloads for all operators or identical types.
In terms of type design, providing all necessary comparison operations is important.
Still, I did not want to point a user to this fact with a compilation error for missing operators if the equivalent `if (x < y)` statement would compile.
Even worse, using `std::equal<T>` fails to compile `assertEqual(std::string{"MyString"}, "OtherString");` -- `std::string` and `const char*` are different types!
In that spirit, my concept only checks that the desired comparison is valid and is not concerned with symmetry of operations.
Each functor allows two different types.
Adding the `std::string_view representation;` helped to generalize error reporting that prints `value <operator> expected` if they are formattable.
Finally, `assertThrow<exception_type>` is a simple template that stringifies its type parameter for error reporting.

## Test Executor and More Generic Tests

Turning the presented ideas into a more mature testing library took mostly throwing examples at it and fixing the errors.
I want to point out aspects related to reflection interactions.

### Templates & Parameters

Doing `rtest::execute(PlainTestSuite{});` works quiet fast.
`rtest::execute(TemplatedTestSuite<int>{});` doesn't.

The issue lies in the `meta::has_identifier()` aspect, because the instantiation has no "natural" identifier.
Instead, the type needs to be checked for an instantiation of a template.
By iterating over all template arguments and stringifying them, an identifier can be synthesized.
This is of course cumbersome in the general case.
Each argument can be a template itself or be a non-type template a.k.a an integer type or `enum` type.
This leads to the requirement of stringifying enumerators -- one of the most prominent motivations for reflection.
Stringification for all these situations is not implemented fully.
My attempt uses the same `ClassType<Type>::forEachTemplateArgument()` style and tries to compute a readable identifier.

Similar automatic stringification of test parameters is desireable.
I failed to find a way to do so.
It does not seem to be possible to inject a single method to provide runtime formatting of members.
I only found [defining new aggregate types](https://en.cppreference.com/cpp/meta/data_member_spec).
Just pasting together all members is not useful for more complex test suites anyway.
Requiring a custom `std::string formatParameters()` if parameters shall be part of the test suite name does not seem excessive.
By default, `formatParameters()` returns an empty string in the `rtest::TestSuite` class.

### Test Inheritance

Composing multiple tests,
e.g. defining a templated base class for common tests and deriving from it with concrete types does not work exactly with the previously shown reflection statements.
Instead,
one must recursively reflect on all `public` base classes and collect their `public` `test*` methods, too.
```c++
template <class Type> class ClassType {
  // ...
  consteval static auto bases() {
    constexpr auto context = meta::access_context::current();
    auto bases = meta::bases_of(info(), context);
    return define_static_array(bases);
  }
  template <class Function>
  constexpr static void forEachDirectPublicMethod(Function &&function) {
    template for (constexpr auto member : ClassType<PlainType>::members()) {
      if constexpr (!meta::is_public(member) || !meta::is_function(member) ||
                    !meta::has_identifier(member) ||
                    meta::is_special_member_function(member))
        continue;
      function.template operator()<member>();
    }
  }
  template <class Function>
  constexpr static void forEachPublicBase(Function &&function) {
    template for (constexpr auto base : ClassType<PlainType>::bases()) {
      if constexpr (!meta::is_public(base))
        continue;
      function.template operator()<base>();
    }
  }
  template <class Function>
  constexpr static void forEachPublicMethod(Function &&function) {
    // Directly defined methods ...
    forEachDirectPublicMethod(function);
    // ... and recursively for any inherited method from any base.
    ClassType<PlainType>::forEachPublicBase([&]<auto reflectedBase>() {
      using BaseClassType = typename[:type_of(reflectedBase):];
      ClassType<BaseClassType>::forEachPublicMethod(function);
    });
  }
// ...
```

### Dynamic Execution Control

Getting more control on how the collected test cases are executed on runtime,
e.g. filtering,
randomization or repetition,
requires to collect all tests in a `std::vector` first and then execute them after a preprocessing step.
Because the executor handles diverse types of test suites,
it can not simply store a `std::vector` of them.
Instead, I chose type erasure via `std::function` and collect a bunch of lambdas.
```c++
struct TestSuiteRunInfo {
  std::string name;
  std::function<TestSuiteResult(RunConfig const &)> runCall;
};
// ...
template <class... Tests> int execute(Tests &&...tests) {
  auto testSuiteCalls = vector<detail::TestSuiteRunInfo>();
  testSuiteCalls.reserve(sizeof...(Tests));
  template for (auto &testSuite : {tests...}) {
    testSuiteCalls.emplace_back(
      testSuite.name(),
      [&](RunConfig const &config) { return testSuite.run(config); });
  }
  return executeTestSuiteCalls(std::move(testSuiteCalls));
}
```
Furthermore type erasing the execution of each _test case_ (method pointers) call by passing the test suite object pointer as `void*` allows to split most execution aspects into separate `.cpp` files.
Only the reflection adjacent code is mandatory header territory.
Each `std::function` is assigned a lambda that reifies the test suite types for the necessary casts from `void*` to a `TestSuite` object.
All executor related operations then use classical vectors and operate on those.
My goal was to reduce compile times and avoid recompilation of the "executor ceremony" for each individual test binary.

### Annotations

A testing library needs a way to skip individual tests or mark them as expected to fail for practical purposes.
[Annotations](https://en.cppreference.com/cpp/language/annotations) are the perfect fit to provide the necessary meta data.
When collecting individual `test*` member functions, each function is analyzed for known annotations.
```c++
// ...
constexpr auto hasSkipAnnotation =
    !meta::annotations_of_with_type(reflectedMethod, ^^rtest::Skip_t).empty();
constexpr auto hasFailAnnotation =
    !meta::annotations_of_with_type(reflectedMethod, ^^rtest::Fail_t).empty();
// ... pass booleans to 'SingleTestInfo'.
```
The types of the annotation are defined together with globals for convenient use.
```c++
namespace rtest {
/// Skip the test method during execution.
struct Skip_t {};
/// Expect the test to fail.
struct Fail_t {};

constexpr inline Skip_t skip{};
constexpr inline Fail_t fail{};
} // namespace rtest
```
Usage is simple.
```c++
struct NumberTest : NumberTheoryTestSuite {
  [[=rtest::fail]]
  void testFailure() {
    assertIsPrime(4);
  }
  void testPrimesToTen() {
    assertIsPrime(2);
    assertIsPrime(3);
    assertIsPrime(5);
    assertIsPrime(7);
  }
  [[=rtest::skip]]
  void testExpensiveCheck() {
    assertIsPrime(101);
  }
};
```

More annotations are of course thinkable, but `skip` and `fail` seemed mandatory.
Maybe it is useful to allow categorization or "coloring" of test cases,
e.g. to disable long running tests for specific executions.

## Learnings & Opportunities

After initial confusion I learned how practical and easy reflection is.
Especially the type erasure application happened naturally without planning ahead for it.
`rtest` library usage is technically full of templates.
None are visible.
In my opinion, `C++` succeeded on its journey to make compile time meta programming part of "normal" programming and overcome the previous template madness.

Of course, now I want more!

- collect tests in "this" translation unit → [does not seem possible to me](https://cppreference.com/cpp/meta/reflection#Scope_identification)
- collect tests of a [module](https://godbolt.org/z/ddbKvxx1v) / [namespace](https://cppreference.com/cpp/meta/members_of) → should be possible, working on it
- annotating simple functions with `[[=rtest::test]]` and collecting them as even simpler way to defining tests would be interesting
- if code generation becomes more versatile, maybe the inheritance becomes unnecessary and is replaced with an annotation, too

Automatic test discovery is still the weak point of my approach.
I want to avoid introducing global state or a macro that hides the global state.
Some form of discovery will be possible and I suspect,
that default constructed test suites will be collected and parametrized/templated test suites registered (not unlike to `GoogleTest`).
All of this gets `C++` closer to a natural "inline definition" of tests within modules or classical `.cpp` files and tiny test executables as entrypoints.

The `rtest` library is not well suited to get custom output formats, like a `JUnit` export.
I believe, this requires a rearrangement of the information flow.
This is not important to me and might not be implemented.

On the other hand,
optionally parallelizing the execution of test suites is straight forward using parallel `STL` algorithms.
This requires proper locking of output operations and hence a result sink that synchronizes output.
Maybe this is effectively an exporter and `JUnit` comes for free.
Parallelization is very much in my interest and I already started it, accepting wrong text output to `stdout`.

Other goodies like `CMake` support,
`CTest` discovery and library packaging will happen after the `C++` puzzles are solved.

## Conclusion

Getting the `rtest` library into a usable state happened faster than I thought.
It uses the accumulated features of the newer standard versions and reflection is only one part of its ergonomics.
In terms of written `C++` standard text, everything seems to be in place to provide a new _native_ experience for writing tests.
Compiler support is of course still evolving.

I am satisfied with the functionality of `rtest`.
As a design study it has everything I need for my recreational coding.
For a production library, it misses a few things, but no groundbreaking developments.
In my opinion, a reflection based approach like with `rtest` can already replace the macro approach of `GoogleTest` in all situations.
Existing libraries should look into a macro-free feature set and play around with that idea.

Compile times are good but it seems to be too early for conclusions.
[This GCC bug](https://gcc.gnu.org/bugzilla/show_bug.cgi?id=124582) hindered the full modules + reflection version for `rtest`.
After GCC-16.2 releases, there is [another migration](https://jonastoth.github.io/posts/migrate_cxx26/) on the todo list ;)

_Macro free testing code would be a big (the biggest?) step towards a preprocessor-free `C++`_ and I am looking forward to that future.
