---
name: google-cpp-testing
description: Use when writing, fixing, or reviewing C++ tests in Google Style — GoogleTest/GMock setup, fixtures, mocking, sanitizers, coverage, and naming conventions.
origin: gglcpp
---

# C++ Testing (Google Style)

Comprehensive testing reference for C++ codebases following Google conventions. Google authored GoogleTest (gtest/gmock) and uses it as the standard C++ test framework.

## When to Use

- Writing new C++ tests or fixing existing tests following Google Style
- Designing unit/integration test coverage for C++ components
- Adding coverage reporting, CI gating, or regression protection
- Configuring CMake/CTest workflows for consistent execution
- Investigating test failures or flaky behavior
- Enabling sanitizers for memory/race diagnostics

### When NOT to Use

- Implementing new product features without test changes
- Large-scale refactors unrelated to test coverage or failures
- Non-C++ projects or non-test tasks

## Core Concepts

- **TDD loop**: red → green → refactor (tests first, minimal fix, then cleanups).
- **Isolation**: prefer dependency injection and fakes over global state.
- **Test layout**: test files adjacent to source, or in a parallel `test/` directory.
- **Mocks vs fakes**: mock for interactions, fake for stateful behavior.
- **CTest discovery**: use `gtest_discover_tests()` for stable test discovery.
- **File extension**: `.cc` for all C++ source files (Google convention).

## TDD Workflow

Follow the RED → GREEN → REFACTOR loop:

1. **RED**: write a failing test that captures the new behavior
2. **GREEN**: implement the smallest change to pass
3. **REFACTOR**: clean up while tests stay green

```cpp
// url_table_test.cc
#include <gtest/gtest.h>
#include "url_table.h"

TEST(UrlTableTest, AddEntryIncreasesSize) {  // RED
  UrlTable table;
  table.AddEntry("https://example.com", 42);
  EXPECT_EQ(table.Size(), 1);
}

// url_table.cc — implement just enough to pass (GREEN)
// Then refactor naming/structure while tests stay green
```

## Test Naming Conventions

Google style uses `PascalCase` for both suite and test names. Names describe the scenario and expected outcome:

```cpp
// Suite name matches the class or subsystem under test.
// Test name describes behavior: verb + subject + context.
TEST(PasswordValidatorTest, RejectsEmptyPassword) { ... }
TEST(PasswordValidatorTest, AcceptsValidPassword) { ... }
TEST(PasswordValidatorTest, RejectsPasswordUnder8Characters) { ... }

// Use TEST_F when a fixture class is involved.
TEST_F(DatabaseTest, WriteAndReadRoundTrips) { ... }
TEST_F(DatabaseTest, ReturnsNotFoundForMissingKey) { ... }
```

## Test File Naming

Test files use the `_test.cc` suffix and live adjacent to the code they test:

```
url_table.h / url_table.cc  →  url_table_test.cc
logging_storage.cc          →  logging_storage_test.cc
```

Or in a parallel directory:

```
src/url_table.cc
test/url_table_test.cc
```

## Basic Unit Test

```cpp
// url_table_test.cc
#include <gtest/gtest.h>
#include "url_table.h"

TEST(UrlTableTest, LookupReturnsNulloptForUnknownUrl) {
  UrlTable table;
  auto result = table.Lookup("https://unknown.example.com");
  EXPECT_FALSE(result.has_value());
}
```

## Test Structure: Arrange-Act-Assert

Follow AAA. Use comments only when the three phases aren't obvious:

```cpp
TEST(UrlTableTest, AddEntryIncreasesSize) {
  UrlTable table;                                   // Arrange

  table.AddEntry("https://example.com", 42);        // Act

  EXPECT_EQ(table.Size(), 1);                       // Assert
}
```

## Fixtures

Use `TEST_F` with a fixture class for shared setup/teardown:

```cpp
class DatabaseTest : public ::testing::Test {
 protected:
  void SetUp() override {
    db_ = std::make_unique<Database>(":memory:");
    ASSERT_TRUE(db_->Connect().ok());
  }
  void TearDown() override { db_.reset(); }

  std::unique_ptr<Database> db_;
};

TEST_F(DatabaseTest, WriteAndReadRoundTrips) {
  ASSERT_TRUE(db_->Write("key", "value").ok());
  auto result = db_->Read("key");
  ASSERT_TRUE(result.ok());
  EXPECT_EQ(*result, "value");
}

TEST_F(DatabaseTest, ReturnsNotFoundForMissingKey) {
  auto result = db_->Read("missing");
  EXPECT_FALSE(result.ok());
}
```

## ASSERT_* vs EXPECT_*

- Use `ASSERT_*` for preconditions that make the rest of the test meaningless if they fail (stops the test immediately).
- Use `EXPECT_*` to accumulate multiple failures in a single test run.

```cpp
TEST_F(DatabaseTest, WriteAndReadRoundTrips) {
  ASSERT_TRUE(db_->Write("key", "value").ok());  // stop if write failed

  auto result = db_->Read("key");
  ASSERT_TRUE(result.ok());                       // stop if read failed
  EXPECT_EQ(*result, "value");                    // report but continue
}
```

## Mocking with GMock

Declare mocks using `MOCK_METHOD`. Match the exact signature of the virtual function including cv-qualifiers:

```cpp
#include <gmock/gmock.h>
#include <gtest/gtest.h>
#include "storage_backend.h"

class MockStorageBackend : public StorageBackend {
 public:
  MOCK_METHOD(absl::Status, Write,
              (std::string_view key, std::string_view value), (override));
  MOCK_METHOD(absl::StatusOr<std::string>, Read,
              (std::string_view key), (override));
};

TEST(LoggingStorageTest, ForwardsWriteToInner) {
  auto mock = std::make_unique<MockStorageBackend>();
  EXPECT_CALL(*mock, Write("k", "v"))
      .WillOnce(::testing::Return(absl::OkStatus()));

  LoggingStorage storage(std::move(mock), &logger_);
  EXPECT_TRUE(storage.Write("k", "v").ok());
}
```

### Useful Matchers

```cpp
using ::testing::_;
using ::testing::Eq;
using ::testing::HasSubstr;
using ::testing::Return;
using ::testing::StartsWith;

EXPECT_CALL(mock, Send(HasSubstr("error"))).Times(1);
EXPECT_CALL(mock, Write(StartsWith("prefix"), _)).WillRepeatedly(Return(absl::OkStatus()));
```

### Fakes vs Mocks

Prefer **fakes** for stateful behavior (e.g., an in-memory store); use **mocks** only when verifying interactions:

```cpp
// Fake: real logic, lightweight backing store
class FakeStorageBackend : public StorageBackend {
 public:
  absl::Status Write(std::string_view key, std::string_view value) override {
    store_[std::string(key)] = std::string(value);
    return absl::OkStatus();
  }
  absl::StatusOr<std::string> Read(std::string_view key) override {
    auto it = store_.find(std::string(key));
    if (it == store_.end()) return absl::NotFoundError("key not found");
    return it->second;
  }
 private:
  absl::flat_hash_map<std::string, std::string> store_;
};
```

## CMake / CTest Setup

```cmake
cmake_minimum_required(VERSION 3.20)
project(example LANGUAGES CXX)

set(CMAKE_CXX_STANDARD 20)
set(CMAKE_CXX_STANDARD_REQUIRED ON)

include(FetchContent)
set(GTEST_VERSION v1.14.0)  # pin to a project-approved version
FetchContent_Declare(
  googletest
  URL https://github.com/google/googletest/archive/refs/tags/${GTEST_VERSION}.zip
)
FetchContent_MakeAvailable(googletest)

enable_testing()

add_executable(url_table_tests
  url_table_test.cc
  url_table.cc
)
target_link_libraries(url_table_tests
  GTest::gtest_main
  GTest::gmock
)
include(GoogleTest)
gtest_discover_tests(url_table_tests)
```

```bash
cmake -S . -B build -DCMAKE_BUILD_TYPE=Debug
cmake --build build -j
ctest --test-dir build --output-on-failure
```

## Running Tests

```bash
# Run all tests
ctest --test-dir build --output-on-failure

# Filter by suite
ctest --test-dir build -R UrlTableTest --output-on-failure

# Run a single test via gtest filter
./build/url_table_tests --gtest_filter=UrlTableTest.AddEntryIncreasesSize
```

## Coverage

Prefer target-level flags over global ones:

```cmake
option(ENABLE_COVERAGE "Enable coverage flags" OFF)

if(ENABLE_COVERAGE)
  if(CMAKE_CXX_COMPILER_ID MATCHES "GNU")
    target_compile_options(url_table_tests PRIVATE --coverage)
    target_link_options(url_table_tests PRIVATE --coverage)
  elseif(CMAKE_CXX_COMPILER_ID MATCHES "Clang")
    target_compile_options(url_table_tests PRIVATE
      -fprofile-instr-generate -fcoverage-mapping)
    target_link_options(url_table_tests PRIVATE -fprofile-instr-generate)
  endif()
endif()
```

GCC + lcov:

```bash
cmake -S . -B build-cov -DENABLE_COVERAGE=ON
cmake --build build-cov -j
ctest --test-dir build-cov --output-on-failure
lcov --capture --directory build-cov --output-file coverage.info
lcov --remove coverage.info '/usr/*' --output-file coverage.info
genhtml coverage.info --output-directory coverage-report
```

Clang + llvm-cov:

```bash
cmake -S . -B build-llvm -DENABLE_COVERAGE=ON -DCMAKE_CXX_COMPILER=clang++
cmake --build build-llvm -j
LLVM_PROFILE_FILE="build-llvm/default.profraw" ctest --test-dir build-llvm
llvm-profdata merge -sparse build-llvm/default.profraw -o build-llvm/default.profdata
llvm-cov report build-llvm/url_table_tests \
  -instr-profile=build-llvm/default.profdata
```

Minimum coverage target: **80%** for new code.

## Sanitizers

Always run tests with sanitizers in CI. Use CMake options:

```cmake
option(ENABLE_ASAN  "Enable AddressSanitizer"          OFF)
option(ENABLE_UBSAN "Enable UndefinedBehaviorSanitizer" OFF)
option(ENABLE_TSAN  "Enable ThreadSanitizer"            OFF)

if(ENABLE_ASAN)
  add_compile_options(-fsanitize=address -fno-omit-frame-pointer)
  add_link_options(-fsanitize=address)
endif()
if(ENABLE_UBSAN)
  add_compile_options(-fsanitize=undefined -fno-omit-frame-pointer)
  add_link_options(-fsanitize=undefined)
endif()
if(ENABLE_TSAN)
  add_compile_options(-fsanitize=thread)
  add_link_options(-fsanitize=thread)
endif()
```

```bash
# AddressSanitizer + UndefinedBehaviorSanitizer
cmake -S . -B build-asan -DENABLE_ASAN=ON -DENABLE_UBSAN=ON
cmake --build build-asan -j
ctest --test-dir build-asan --output-on-failure

# ThreadSanitizer (for concurrent code)
cmake -S . -B build-tsan -DENABLE_TSAN=ON
cmake --build build-tsan -j
ctest --test-dir build-tsan --output-on-failure
```

## Flaky Tests Guardrails

- Never use `sleep` for synchronization — use condition variables or `absl::Notification`.
- Generate unique temp paths per test; always clean them in `TearDown`.
- Avoid real time, network, or filesystem dependencies in unit tests.
- Use deterministic seeds for randomized inputs.
- Inject a clock abstraction rather than calling `absl::Now()` directly.

## What to Test

- Public APIs of each class
- Edge cases: empty inputs, boundary values, maximum sizes
- Error paths: invalid arguments, missing resources, network failures
- Concurrent behavior for thread-safe classes

## What NOT to Test

- Private implementation details — test behavior, not internals
- Trivial getters/setters with no logic
- Third-party library behavior

## Best Practices

### DO

- Keep tests deterministic and isolated
- Prefer dependency injection over global state
- Use `ASSERT_*` for preconditions, `EXPECT_*` for additional checks
- Separate unit vs integration tests with CTest labels or directories
- Run sanitizers in CI for memory and race detection
- Follow the same Google naming and formatting conventions as production code

### DON'T

- Don't test private methods directly — refactor if private logic needs its own tests
- Don't rely on real time or network in unit tests
- Don't use sleeps when condition variables or latches suffice
- Don't over-mock simple value objects — prefer fakes
- Don't use brittle string matching for non-critical log output

### Common Pitfalls

- **Fixed temp paths** → generate unique directories per test and clean in `TearDown`.
- **Wall clock time** → inject a clock or use `absl::FakeTime`.
- **Flaky concurrency tests** → use condition variables/latches and bounded waits.
- **Hidden global state** → reset in `SetUp`/`TearDown` or remove globals.
- **Over-mocking** → prefer fakes for stateful behavior; mock only interactions.
- **Missing sanitizer runs** → add ASan/UBSan/TSan builds in CI.

## Debugging Failures

1. Re-run the single failing test with `--gtest_filter`.
2. Add scoped logging around the failing assertion.
3. Re-run with a sanitizer build.
4. Expand to the full suite once the root cause is fixed.

## Related Skills

- `gglcpp:google-cpp-style-guide` — full coding conventions (naming, formatting, headers, etc.) that apply to test code as well
- `gglcpp:google-cpp-build` — fix build errors before running tests
- `gglcpp:google-cpp-review` — code review after tests pass
