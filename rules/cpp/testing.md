---
paths:
  - "**/*.cpp"
  - "**/*.hpp"
  - "**/*.cc"
  - "**/*.hh"
  - "**/*.cxx"
  - "**/*.h"
  - "**/CMakeLists.txt"
---
# C++ Testing (Google Style)

Testing standards for C++ codebases following Google conventions. Google authored GoogleTest (gtest/gmock) and uses it as the standard C++ test framework.

## Framework

Use **GoogleTest** (`gtest`/`gmock`) with **CMake/CTest**.

```cmake
include(FetchContent)
FetchContent_Declare(
  googletest
  URL https://github.com/google/googletest/archive/refs/tags/v1.14.0.zip
)
FetchContent_MakeAvailable(googletest)

add_executable(my_test my_test.cc)
target_link_libraries(my_test GTest::gtest_main GTest::gmock)
include(GoogleTest)
gtest_discover_tests(my_test)
```

## Running Tests

```bash
cmake --build build && ctest --test-dir build --output-on-failure
```

## Test Structure

Follow the Arrange-Act-Assert pattern. Name tests to describe behavior:

```cpp
#include <gtest/gtest.h>

TEST(UrlTableTest, AddEntryIncreasesSize) {
  // Arrange
  UrlTable table;

  // Act
  table.AddEntry("https://example.com", 42);

  // Assert
  EXPECT_EQ(table.Size(), 1);
}

TEST(UrlTableTest, LookupReturnsNulloptForUnknownUrl) {
  UrlTable table;
  auto result = table.Lookup("https://unknown.example.com");
  EXPECT_FALSE(result.has_value());
}
```

## Test Naming Conventions

Google style: `TEST(SuiteName, TestName)` — both use `PascalCase`. Test names describe the scenario and expected outcome:

```cpp
TEST(PasswordValidatorTest, RejectsEmptyPassword) { ... }
TEST(PasswordValidatorTest, AcceptsValidPassword) { ... }
TEST(PasswordValidatorTest, RejectsPasswordUnder8Characters) { ... }
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
```

## Mocking with GMock

```cpp
#include <gmock/gmock.h>

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

## Coverage

```bash
cmake -DCMAKE_CXX_FLAGS="--coverage" \
      -DCMAKE_EXE_LINKER_FLAGS="--coverage" \
      -Bbuild-cov .
cmake --build build-cov
ctest --test-dir build-cov --output-on-failure
lcov --capture --directory build-cov --output-file coverage.info
lcov --remove coverage.info '/usr/*' --output-file coverage.info
genhtml coverage.info --output-directory coverage-report
```

Minimum coverage target: **80%** for new code.

## Sanitizers

Always run tests with sanitizers in CI:

```bash
# AddressSanitizer + UndefinedBehaviorSanitizer
cmake -DCMAKE_CXX_FLAGS="-fsanitize=address,undefined" -Bbuild-asan .
cmake --build build-asan
ctest --test-dir build-asan --output-on-failure

# ThreadSanitizer (for concurrent code)
cmake -DCMAKE_CXX_FLAGS="-fsanitize=thread" -Bbuild-tsan .
cmake --build build-tsan
ctest --test-dir build-tsan --output-on-failure
```

## What to Test

- Public APIs of each class
- Edge cases: empty inputs, boundary values, maximum sizes
- Error paths: invalid arguments, missing resources, network failures
- Concurrent behavior for thread-safe classes

## What NOT to Test

- Private implementation details (test behavior, not internals)
- Trivial getters/setters with no logic
- Third-party library behavior

## Test File Naming

```
my_class.cc        -> my_class_test.cc
url_table.h/.cc    -> url_table_test.cc
```

Place tests adjacent to the code they test, or in a parallel `test/` directory.

## Reference

See skill `gglcpp:google-cpp-testing` for the full testing reference (fixtures, mocking, sanitizers, coverage, flaky-test guardrails, etc.).

See skill `gglcpp:google-cpp-style-guide` for full coding conventions that apply to test code too (naming, formatting, etc.).
