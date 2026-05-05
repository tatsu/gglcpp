---
name: google-cpp-style-guide
description: Google C++ Style Guide rules and conventions. Use when writing, reviewing, or refactoring C++ code in projects that follow Google's C++ style. Covers naming, formatting, header files, scoping, classes, functions, and modern C++ feature usage.
origin: gglcpp
---

# Google C++ Style Guide

Comprehensive coding standards derived from the [Google C++ Style Guide](https://google.github.io/styleguide/cppguide.html). Targets **C++20** (no C++23 features, no non-standard extensions).

## When to Use

- Writing or reviewing C++ code in a Google-style codebase
- Enforcing naming, formatting, and structural conventions
- Choosing which C++ features to use or avoid
- Setting up linting and formatting tooling (cpplint, clang-format)

### When NOT to Use

- Projects that explicitly follow a different style guide (LLVM, MISRA, etc.)
- Legacy C codebases that predate modern C++ conventions

---

## Core Philosophy

| Principle | Meaning |
|-----------|---------|
| Optimize for the reader | Code is read far more than written |
| Be consistent | Consistency within a file trumps personal preference |
| Avoid surprises | Dangerous or surprising constructs are restricted |
| Mind the scale | 100M+ lines — global namespace pollution is costly |

---

## C++ Version

```
Target: C++20
Do NOT use: C++23 features, compiler extensions, non-standard extensions
```

---

## Header Files

### Self-Contained Headers

- Every `.cc` file should have an associated `.h` file
- Headers must be self-contained (compile on their own)
- End headers with `.h`; files meant for inclusion in non-standard places use `.inc`

### #define Guards

Every header must use `#define` guards in the form `<PROJECT>_<PATH>_<FILE>_H_`:

```cpp
#ifndef FOO_BAR_BAZ_H_
#define FOO_BAR_BAZ_H_
// ...
#endif  // FOO_BAR_BAZ_H_
```

### Include What You Use

- Include the header that directly declares/defines each symbol you use
- Do NOT rely on transitive inclusions
- `foo.cc` must include `bar.h` if it uses symbols from it, even if `foo.h` already includes `bar.h`

### Forward Declarations

**Avoid** forward declarations where possible. Instead, include headers:

```cpp
// BAD: forward declaration
class Foo;
void Bar(Foo*);

// GOOD: include the header
#include "foo/foo.h"
void Bar(const Foo& foo);
```

Exceptions: when the full header is genuinely expensive and a forward declaration is the best engineering judgment.
Never forward-declare entities your project does not own.

### Names and Order of Includes

Order (separated by blank lines):

1. Related header (for `.cc` files: `foo.cc` includes `foo.h` first)
2. C system headers (`<unistd.h>`, `<stdlib.h>`)
3. C++ standard library headers (`<vector>`, `<string>`)
4. Other libraries' `.h` files
5. Your project's `.h` files

Within each group, alphabetical order.

### Inline Functions in Headers

Only define a function inline in a header if it is **10 lines or fewer**. Longer definitions belong in `.cc` unless they must be in the header for technical reasons (templates, `constexpr`).

---

## Scoping

### Namespaces

- Use namespaces to partition global scope
- Namespace names: `lowercase_with_underscores`
- Terminate with `// namespace foo`
- Do NOT use `using namespace` directives (pollutes scope)
- Do NOT use inline namespaces unless you understand the implications

```cpp
namespace project::module {

class Widget { /* ... */ };

}  // namespace project::module
```

### Internal Linkage

Prefer unnamed namespaces or `static` for file-scope symbols not needed externally:

```cpp
namespace {
void Helper() { /* ... */ }
}  // namespace
```

### Nonmember and Static Functions

- Prefer nonmember functions in a namespace over static class methods when possible
- Group related free functions in a namespace; do not create a class just to hold static members

### Local Variables

- Declare variables as close to their first use as possible
- Initialize variables at the point of declaration

```cpp
// GOOD: declare and initialize together
int i = f();
std::vector<int> v = {1, 2, 3};

// BAD: declare then assign
int i;
i = f();
```

### Static and Global Variables

- Avoid global variables with complex constructors/destructors
- Prefer `constexpr` for compile-time constants
- Prefer function-local statics over globals for lazily-initialized singletons
- Use `ABSL_CONST_INIT` or `constinit` to enforce constant initialization

```cpp
// GOOD: constexpr constant
constexpr int kMaxConnections = 100;

// BAD: dynamic initialization at global scope
std::string g_name = ComputeName();  // undefined init order
```

---

## Classes

### Constructors

- Do minimal work in constructors; prefer factory functions for operations that can fail
- If initialization can fail, use a separate `Init()` method or a factory

### Implicit Conversions

- Mark single-argument constructors `explicit` unless implicit conversion is intentional and documented
- Mark conversion operators `explicit` unless the conversion is lossless and obvious

```cpp
class BigInt {
public:
    explicit BigInt(int value);  // prevent accidental int->BigInt conversions
};
```

### Copyable and Movable Types

- If a class holds a resource, explicitly decide whether it is copyable/movable
- Use `= delete` to prohibit unwanted operations
- If you define any of {destructor, copy ctor, copy assign, move ctor, move assign}, define all five (Rule of Five) or none (Rule of Zero)

```cpp
// Rule of Zero: let compiler generate all
struct Point { double x, y; };

// Non-copyable resource wrapper
class Mutex {
public:
    Mutex() = default;
    ~Mutex();
    Mutex(const Mutex&) = delete;
    Mutex& operator=(const Mutex&) = delete;
};
```

### Structs vs. Classes

- Use `struct` for passive data carriers with no invariants
- Use `class` when there are invariants to maintain or non-trivial behavior

### Inheritance

- Prefer composition over inheritance
- Only use `public` inheritance to model "is-a" relationships
- All base class destructors must be `public virtual` or `protected non-virtual`
- Use `override` on every overriding function; never use `virtual` on an override
- Use `final` to prevent further subclassing when appropriate

```cpp
class Shape {
public:
    virtual ~Shape() = default;
    virtual double area() const = 0;
};

class Circle final : public Shape {
public:
    explicit Circle(double radius) : radius_(radius) {}
    double area() const override { return 3.14159 * radius_ * radius_; }
private:
    double radius_;
};
```

### Operator Overloading

- Overload operators only when their meaning is obvious and consistent with built-in types
- Do NOT overload `&&`, `||`, `,`, or unary `&`
- Prefer named functions when the semantic is not obvious

### Access Control

- Make data members `private` (rarely `protected`)
- Provide accessor/mutator methods as needed
- Declaration order: `public`, then `protected`, then `private`

---

## Functions

### Write Short Functions

- Prefer functions of 40 lines or fewer
- If a function is longer, consider breaking it into smaller well-named helpers

### Inputs and Outputs

- Use return values for outputs; avoid output parameters
- For multiple outputs, return a struct or `std::pair`/`std::tuple` with named members
- Input parameters: pass primitives by value, expensive types by `const&`
- Sink parameters (to be moved from): pass by value

```cpp
// GOOD: return values
struct ParseResult { std::string token; int position; };
ParseResult ParseNext(std::string_view input);

// BAD: output parameters
void ParseNext(std::string_view input, std::string* token, int* position);
```

### Function Overloading

Use overloading only when overloads have the same semantics and different signatures make the intent clear. Avoid overloads that differ only in subtle type conversions.

### Default Arguments

- Default arguments are allowed on non-virtual functions
- Do not use default arguments on virtual functions (they interact poorly with override)
- Place defaults only on trailing parameters

### Trailing Return Types

Use trailing return types (`auto f() -> T`) only when the return type cannot be expressed before the parameter list (e.g., complex templates). Prefer conventional syntax otherwise.

---

## Other C++ Features

### Ownership and Smart Pointers

| Ownership pattern | Tool |
|-------------------|------|
| Single owner | `std::unique_ptr` |
| Shared owner | `std::shared_ptr` |
| Non-owning observer | raw pointer `T*` or reference |

- Never use raw `new`/`delete` for ownership management
- Use `std::make_unique` and `std::make_shared`
- A raw pointer passed to a function means "non-owning"; document ownership clearly

### Rvalue References

- Use for move constructors/move assignment operators
- Use for perfect forwarding (`T&&` with `std::forward`)
- Do NOT use `&&` to make functions that accept both lvalues and rvalues via a single overload in general APIs

### Exceptions

**Do not use exceptions** in new code unless the project already uses them. Google's codebase largely avoids exceptions for historical reasons. Use error codes, `absl::Status`, or `std::optional`/`std::expected` instead.

### `noexcept`

- Mark functions `noexcept` when they genuinely cannot throw (destructors, swap, move operations)
- Do NOT use `noexcept` speculatively on functions that might call throwing code

### RTTI and `dynamic_cast`

- Avoid RTTI (`typeid`, `dynamic_cast`) in new code
- Use virtual dispatch instead of type-checking
- If RTTI is truly needed, prefer `absl::StatusOr` patterns

### Casting

| Use | When |
|-----|------|
| `static_cast<T>` | Well-defined conversions between compatible types |
| `const_cast<T>` | Remove `const` only when you know it's safe |
| `reinterpret_cast<T>` | Low-level, dangerous; avoid unless unavoidable |
| C-style `(T)` casts | **Never** |

### Use of `const`

- Use `const` everywhere possible: variables, parameters, member functions, return types
- `const` on a member function means it does not modify observable state
- Prefer `const` references over mutable ones for parameters

```cpp
// All these are good uses of const
const int kMax = 100;
void Print(const std::string& msg);
class Foo { int size() const; };
```

### Use of `constexpr`, `constinit`, `consteval`

- Use `constexpr` for values computable at compile time
- Use `constinit` for static/global variables that must be constant-initialized
- Use `consteval` for functions that must only execute at compile time

### Integer Types

- Prefer `int` for ordinary counters and loop variables
- Use `int64_t`, `uint32_t` etc. (from `<cstdint>`) when bit-width matters
- Do NOT use `long`, `short`, `unsigned` without explicit width justification
- Do NOT mix signed and unsigned in comparisons

### Preprocessor Macros

- Avoid macros for constants — use `constexpr`
- Avoid macros for functions — use inline functions or templates
- Macro names: `ALL_CAPS_WITH_UNDERSCORES`
- `#undef` macros after use when possible

### `nullptr` and `0`

- Use `nullptr` for null pointers, never `0` or `NULL`
- Use `0` for numeric zero only

### `sizeof`

- Prefer `sizeof(variable)` over `sizeof(type)` — it tracks changes automatically

### Type Deduction (`auto`)

- Use `auto` when the type is obvious from context (e.g., from `make_unique`, iterators)
- Do NOT use `auto` when the type is not obvious from the right-hand side

```cpp
auto it = map.begin();                      // GOOD: obvious
auto result = ProcessData(input);           // BAD: type unclear
std::vector<int> result = ProcessData(input); // GOOD: explicit
```

### Lambda Expressions

- Keep lambdas short; long lambdas should be named functions
- Avoid default capture (`[=]` or `[&]`) — capture only what is needed
- Do not pass lambdas by reference across thread boundaries

```cpp
// GOOD: explicit capture
auto predicate = [&threshold](const Item& item) {
    return item.value > threshold;
};

// BAD: captures everything
auto f = [&]() { /* ... */ };
```

### Template Metaprogramming

- Avoid TMP unless you have a clear, practical need
- Prefer `constexpr` and `if constexpr` over complex TMP

### Concepts and Constraints (C++20)

- Use `requires` clauses and concept constraints to document and enforce template requirements
- Prefer standard library concepts (`std::integral`, `std::copyable`, etc.) where available

```cpp
template<std::integral T>
T gcd(T a, T b);
```

### C++20 Modules

**Do not use** C++20 modules. Toolchain support is not yet mature in Google's build systems.

### Coroutines

Use only coroutine libraries **explicitly approved by your project leads**. Do not implement custom promise types.

### Disallowed Standard Library Features

Avoid:
- `<ratio>` (compile-time rational numbers)
- `<cfenv>` / `fenv_t` (floating-point environment)
- `std::filesystem` (use project-provided alternatives)
- `std::bind` (use lambdas instead)
- `std::regex` (use RE2 or similar)

---

## Naming

### General Principle

Names should be descriptive. Prefer clarity over brevity. Do not abbreviate unless the abbreviation is universally known in the domain.

### File Names

`lowercase_with_underscores.cc` / `.h`

```
my_useful_class.cc
my_useful_class.h
my_useful_class_test.cc
```

### Type Names (Classes, Structs, Enums, Type Aliases)

`PascalCase`

```cpp
class UrlTable;
struct UrlTableEntry;
enum class UrlTableError { ... };
using PropertiesMap = std::map<std::string, std::string>;
```

### Variable Names

`snake_case` (local variables and function parameters)

```cpp
std::string table_name;
int num_errors;
```

### Class Member Variables

`snake_case_` with trailing underscore:

```cpp
class Foo {
    std::string name_;
    int num_completed_connections_;
};
```

### Constant Names

`kPascalCase` (compile-time constants and `static constexpr`):

```cpp
constexpr int kDaysInAWeek = 7;
constexpr char kMyApplicationName[] = "My Application";
```

### Function Names

`PascalCase` for regular functions:

```cpp
void AddTableEntry();
bool DeleteUrl();
std::string OpenFileOrDie();
```

### Namespace Names

`lowercase_without_underscores` (short, non-conflicting):

```cpp
namespace mynamespace {}
namespace project::http {}
```

### Enumerator Names

`kPascalCase` (same as constants):

```cpp
enum class AlternateUrl {
    kOptionA,
    kOptionB,
    kOptionC,
};
```

### Macro Names

`ALL_CAPS_WITH_UNDERSCORES`:

```cpp
#define MY_MACRO_THAT_SCARES_SMALL_CHILDREN
```

---

## Comments

### File Comments

Every file should start with a license notice, then a description of the file's purpose.

### Class Comments

Describe what the class does and how to use it. Do not duplicate the implementation.

### Function Comments

Document what a function does, not how it does it. Note preconditions, postconditions, and error behavior.

```cpp
// Returns an iterator for this table.  It is the client's
// responsibility to delete the iterator when it is done with it,
// and it must not use the iterator once the GargantuanTable object
// on which the iterator was created has been deleted.
//
// The iterator is initially positioned at the beginning of the table.
GargantuanTableIterator* NewIterator() const;
```

### Implementation Comments

Explain the **why**, not the **what**. Use comments for non-obvious logic, workarounds, or important constraints.

### TODO Comments

```cpp
// TODO(username): Remove this when the bug is fixed.
// TODO(b/12345): Remove this when the bug is fixed.
```

---

## Formatting

### Line Length

**80 characters** maximum.

### Indentation

**2 spaces**. No tabs.

### Function Declarations

```cpp
ReturnType ClassName::FunctionName(Type par_name1, Type par_name2) {
  DoSomething();
  // ...
}
```

Long parameter lists wrap with 4-space indent:

```cpp
ReturnType LongClassName::ReallyLongFunctionName(
    Type par_name1,
    Type par_name2,
    Type par_name3) {
  DoSomething();
}
```

### Braces

Opening brace on the same line. Closing brace on its own line:

```cpp
if (condition) {
  DoOneThing();
  DoAnotherThing();
} else {
  DoSomethingOtherwise();
}
```

Always use braces, even for single-statement bodies.

### Pointer and Reference Alignment

Attach `*` and `&` to the type, not the variable:

```cpp
char* ptr;
const std::string& str;
```

### Class Format

```cpp
class MyClass : public OtherClass {
 public:
  MyClass();
  explicit MyClass(int var);
  ~MyClass() {}

  void SomeFunction();

 private:
  bool SomeInternalFunction();

  int member_var_;
};
```

### Constructor Initializer Lists

```cpp
// Single line if it fits
MyClass::MyClass(int var) : some_var_(var) {}

// Multiline: each member on its own line, 4-space indent
MyClass::MyClass(int var)
    : some_var_(var),
      some_other_var_(var + 1) {
  DoSomething();
}
```

---

## Tooling

### cpplint

Run `cpplint` on all C++ files before committing:

```bash
cpplint --filter=-build/include_order src/*.cpp src/*.h
```

### clang-format

Use the Google style preset:

```bash
clang-format -style=Google -i src/*.cpp src/*.h
```

Or configure `.clang-format` at the project root:

```yaml
BasedOnStyle: Google
```

### clang-tidy

Enable Google-relevant checks:

```bash
clang-tidy --checks='google-*,modernize-*,bugprone-*' src/*.cpp --
```

---

## Quick Reference Checklist

Before marking C++ work complete (Google style):

- [ ] File has `#define` guard in `<PROJECT>_<PATH>_<FILE>_H_` format
- [ ] Only includes what it uses (no transitive include reliance)
- [ ] No forward declarations of externally-owned entities
- [ ] `nullptr` used instead of `0` or `NULL` for pointers
- [ ] No `using namespace` at global scope in headers
- [ ] Types: `PascalCase`, variables: `snake_case`, members: `snake_case_`
- [ ] Constants: `kPascalCase`, macros: `ALL_CAPS`
- [ ] Functions: `PascalCase`
- [ ] Single-argument constructors marked `explicit`
- [ ] Base class destructors `public virtual` or `protected non-virtual`
- [ ] `override` used on every overriding method; no `virtual` on overrides
- [ ] Smart pointers (`unique_ptr`/`shared_ptr`) used instead of raw `new`/`delete`
- [ ] No C-style casts — use `static_cast`, `const_cast`, etc.
- [ ] `const` applied everywhere applicable
- [ ] No exceptions used unless project already uses them
- [ ] No RTTI (`dynamic_cast`, `typeid`) used
- [ ] No C++23 features, no non-standard extensions
- [ ] No C++20 modules, no custom coroutine promise types
- [ ] `cpplint` passes with no errors
- [ ] `clang-format -style=Google` applied
- [ ] Lines <= 80 characters, 2-space indent, no tabs

## Related Skills

- `gglcpp:google-cpp-testing` — GoogleTest/GMock setup, fixtures, mocking, sanitizers, and coverage
- `gglcpp:google-cpp-build` — incremental build error resolution (CMake, cpplint, clang-tidy)
- `gglcpp:google-cpp-review` — code review for memory safety, concurrency, and Google Style compliance
