# Meet Swift Testing

[github.com/swiftlang/swift-testing](https://github.com/swiftlang/swift-testing)

## Building blocks

### Test functions

- Annotated using  `@Test`
- Can be global functions or methods in a type
- May be `async` or `throws`
- May be global actor-isolated (such as `@MainActor`)

```swift
import Testing
@testable import DestinationVideo

@Test("Check video metadata") func videoMetadata() {
  let video = Video(fileName: "By the Lake.mov")
  let expectedMetadata = MetaData(duration: .seconds(90))
  #expect(video.metadata == expectedMetadata)
}
```

### Expectations

Validate expected conditions

- Use `#expect(...)` to validate that an expected condition is true
- Accepts ordinary expressions and operators
- Captures source code and subexpression values upon failure

```swift
#expect(1 == 2)
// Expectation failed: 1 == 2

#expect(user.name == "Alice")
// Expectation failed: (user.name -> "Sarah") == "Alice"

#expect(!array.isEmpty)
// Expectation failed: !((array -> []).isEmpty -> true)

#expect(numbers.contains(4))
// Expectation failed: (number -> [1, 2, 3].contains(4))
```

- Error shows the contents of the variables automatically 

### Required expectations

- Use `try #require(...)` to stop the test if the condition is false

```swift
try #require(session.isValid) // if failed
session.invalidate() // not executed
```

- Can unwrap optional values, and stop test when `nil`

```swift
let method = try #require(paymentMethods.first) // if nil
#expect(method.isDefault) // not executed
```

### Traits

- Add descriptive information about a test
- Customize whether a test runs
- Modify how a test behaves

| Built-in traits                                  |                                                              |
| ------------------------------------------------ | ------------------------------------------------------------ |
| `@Test("Custom name")`                           | Customize the display name of a test                         |
| `@Test(.bug("example.com/issues/999"), "Title")` | Reference an issue from a bug tracker                        |
| `@Test(.tags(.critical))`                        | Add a custom tag a test                                      |
| `@Test(.enabled(if: Server.isOnline))`           | Specify a runtime condition for a test                       |
| `@Test(.disable("Currently broken"))`            | Unconditionally disable a test                               |
| `@Test(...) @available(macOS 15, *)`             | Limit a test to certain OS versions                          |
| `@Test(.timeLimit(.minutes(3)))`                 | Set a maximum time limit for a test                          |
| `@Suite(.serialized)`                            | Run the tests in a suite one at a time, without parallelization |

### Suites

- Group related test functions and suites
- Annotated using `@Suite`
  - Implicit for types containing `@Test` functions or suites
- May have stored instance properties
- Use `init` and `deinit` for set-up and tear-down logic, respectively
- Initialized once per instance `@Test` method



## Common workflows

### Runtime conditions

- Specify a runtime-evaluated condition for a test using `.enabled(if: ...)`
  - Test will be skipped if condition is false
- Unconditionally disable a test using `.disabled(...)`
  - Comment describing reason is included in results
  - Pair any trait with `.bug(...)` to reference an issue in a bug-tracking system

```swift
@Test(.enabled(if: AppFeatures.isCommentingEnabled)) // Test 'videoCommenting()' skipped
func videoCommenting() { ... }

@Test(.disabled("Due to a known crash"))
		 (.bug("example.org/bugs/1234", "Programs crashes at <symbol>"))
func example() { ... }
```

- Use `@available(...)` instead of checking at runtime using `#available(...)`

```swift
@Test func hasRuntimeVersionCheck() {
  guard #available(macOS 15, *) else { return } // ❌
}

@Test @available(macOS 15, *) func useNewAPIs() { ... } // ✅
```

### Test tags

- Associate tests which have things in common to:
  - Run all tests with a specific tag
  - Filter by tag or see insights in Test Report

- Shared among thests anywher in a project
- Tags can be local to a project or share (multiple projects) 😎
- Prefer tags over test names when including/excluding in a Test Plan
- Use the most appropriate trati type
  - `.enabled(if: ...)` instead of `.tags(...)` to represent a condition

### Parameterized testing

- View details of each argument in results
- Re-run individual arguments to debug
- Run arguments in parallel

```swift
import Testing
@testable import DestinationVideo

struct VideoContinentsTests {
  
  @Test("Number of mentioned continents", arguments: [
    "A Beach",
    "By the Lake",
    "Camping in the Woods"
  ])
  func mentionedContinentCounts(videoName: String) async throws {
    let videoLibrary = try await VideoLibrary()
    let video = try #require(await videoLibrary.video(named: videoName))
    ...
  }
}
```



## Swift Testing and XCTest

### Comparison: Test functions

|                    | XCTest                                 | Swift Testing                                            |
| ------------------ | -------------------------------------- | -------------------------------------------------------- |
| Discovery          | Name begins with "test"                | `@Test`                                                  |
| Supported types    | Instance methods                       | Instance methods, Static/class methods, Global functions |
| Supported traits   | No                                     | Yes                                                      |
| Paralled execution | Multi process macOS and Simultaor only | In-process Supports devices                              |

### Comparison: Expectations

| XCTest                                             | Swift Testing       |
| -------------------------------------------------- | ------------------- |
| `XCTAssert`, `XCTAssertTrue`, `XCTAssertFalse`     | `#exepct(x)`        |
| `XCTAssertNil`, `XCTAssertNotNil`                  | `#expect(x == nil)` |
| `XCTAssertEqual`, `XCTAssertNotEqual`              | `#expect(x == y)`   |
| `XCTAssertIdentical`, `XCTAssertNotIdentical`      | `#expect(x === y)`  |
| `XCTAssertGreaterThan`, `XCTAssertLessThanOrEqual` | `#exepct(x > y)`    |
| `XCTAssertGreaterThanOrEqual`, `XCTAssertLessThan` | `#exepct(x >= y)`   |

### Comparison: Suites

|                  | XCTest                                                       | Swift Testing         |
| ---------------- | ------------------------------------------------------------ | --------------------- |
| Types            | class                                                        | struct, actor, class  |
| Discovery        | Subclass `XCTestCase`                                        | `@Suite`              |
| Before each test | `setUp()`, `setUpWithError() throws`, `setUp() async throws` | `init() async throws` |
| After each test  | `tearDown() async throws`, `tearDownWithError() throws`, `tearDown()` | `deinit`              |
| Sub-groups       | Unsupported                                                  | Via type nesting      |

### Migrating from XCTest

- Shared a single unit test target
  - Swift Testing tests can coexist with XCTests
- Consolidate similar XCTests into a parameterized test
- Migrate each XCTest class with only one test method to a global `@Test` function
- Remove redundant "test" prefix from method names
- Continue using XCTest for tests which:
  - Use UI automation APIs (such as `XCUIApplication`)
  - Use performance testing APIs (such as `XCTmetric`)
  - Can only be written in Objecive-C
- Avoid calling `XCTAssert(...)` from Swift Testing tests, or `#expect(...)` from XCTests