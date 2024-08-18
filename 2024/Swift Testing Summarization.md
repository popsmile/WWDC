# Swift Testing

- 새로운 테스트 프레임워크
  - Swift 6.0+, Xcode 16.0+
  - Xcode 16부터 프로젝트 생성 시 선택 가능
  - 오픈소스 ([swiftlang/swift-testing](https://github.com/swiftlang/swift-testing))
- 3가지 메인 피쳐
  - `@Test`, `@Suite` macro
  - Traits
  - `#expect`, `#require` expectation



## `@Test`

- 함수명을 test로 시작하지 않고, `@Test` 어노테이션 사용
- `async`, `throws`, `mutating` 함수 모두 가능

```swift
import Testing
@testable import DestinationVideo

@Test func videoMetadata() async throws {
  // ...
}
```



## `#expect`

- 테스트 코드 검증 API
- 명시한 조건이 true인지 검증

```swift
import Testing
@testable import DestinationVideo

@Test func videoMetadata() {
  let video = Video(fileName: "Lake.mov")
  let expectedMetadata = MetaData(duration: .seconds(90))
  #expect(video.metadata == expectedMetadata)
}
```



## `#require`

- 검증에 꼭 필요한 조건이라 준수되지 않을 경우 테스트 진행을 중단하고 싶을 경우 사용
- 옵셔널 언래핑에도 사용

```swift
try #require(session.isValid) // if failed
session.invalidate() // not executed
```

```swift
let method = try #require(paymentMethods.first) // if nil
#expect(method.isDefault) // not executed
```



## Traits

- 테스트 정보 명시
- 테스트 수행 조건

### Built-in traits

| `@Test("Custom name")`                           | 테스트 이름                 |
| ------------------------------------------------ | --------------------------- |
| `@Test(.bug("example.com/issues/999"), "Title")` | 관련 이슈 URL 태깅          |
| `@Test(.tags(.critical))`                        | 커스텀 키워드 태깅          |
| `@Test(.enabled(if: Server.isOnline))`           | 테스트 수행 전제 조건       |
| `@Test(.disabled("Currently broken"))`           | 테스트 비활성화             |
| `@Test(...) @available(macOS 15, *)`             | OS 버전 제한                |
| `@Test(.timeLimit(.minutes(3)))`                 | 최대 수행 시간 제한         |
| `@Suite(.serialized)`                            | Suite 내부 테스트 직렬 수행 |

```swift
@Test(.enabled(if: User.hasSubDevice))
@available(iOS 17, *)
func deactivateKakaopay() {
    let kakaopay = Kakaopay()
    #expect(kakaopay.isAvailable)
}
```

```swift
@Test(.bug("https://kakaopay.jira.com/KAKAOPAY-12345", "인증플랫폼 FIDO 노운 이슈"),
      .disabled("7차에 수정 예정입니다..!"))
func authenticateFido() async throws {
    let auth = PayAuth(transaction: "FIDORESET")
    let result = try await auth.start()
    #expect(result.isSuccess)
}
```

```swift
@Test("인증플랫폼 FIDO 생체인증 변경 플로우 테스트",
     .tags(.auth))
func authenticateFido() async throws {
    await withKnownIssue {
        let auth = PayAuth(transaction: "FIDORESET")
        let result = try await auth.start()
        #expect(result.isSuccess)
    }
}
```

### Tags

- 테스트 플랜에서 포함 여부 결정 및 필터링 가능
- 여러 프로젝트에서 공유 가능

```swift
extension Tag {
    @Tag static var auth: Self
}
```



## `@Suite`

- 테스트 함수의 그룹 또는 테스트 수트의 그룹
- 저장 변수 사용 가능
- 기존 `setup`, `teardown` 로직 >  `init`, `deinit`
- 개별 `@Test` 함수 수행마다 새로 생성

```swift
@Suite(.serialized)
struct VideoTests {
    let video = Video(fileName: "By the Lake.mov")
  
    @Test("Check video metadata") func videoMetadata() {
      let expectedMetadata = MetaData(duration: .seconds(90))
      #expect(video.metadata == expectedMetadata)
    }
    
    @Test func rating() async throws {
      #expect(video.contentRating == "G")
    }
}
```



### Suite & Tag

```swift
extension Tag {
  @Tag static var caffeinated: Self
  @Tag static var chocolatey: Self
}

@Suite(.tags(.caffeinated)) struct DrinkTests {
  @Test func espressoExtractionTime() { /*...*/ }
  @Test func greenTeaBrewTime() { /*...*/ }
  @Test(.tags(.chocolatey)) func mochaIngredientProportion() { /*...*/ }
}

@Suite struct DessertTests {
  @Test(.tags(.caffeinated, .chocolatey)) func espressoBrownieTexture() { /*...*/ }
  @Test func bungeoppangFilling() { /*...*/ }
  @Test func fruitMochiFlavors() { /*...*/ }
}
```



## Parameterized Testing

- Collection 파라미터
- 각 argument 테스트 병렬 수행

```swift
@Test("SMS 인증 생년월일 검증", arguments: [
    "251225",
    "941321",
    "020733"
])
func example(birthday: String) async throws {
    let birthday = Birthday(string: birthday)
    let validator = SMSValidator()
    validator.type(birthday: birthday)
    #expect(validator.hasError)
}
```

- 2가지 Collection의 모든 조합 테스트

```swift
@Test(arguments: Ingredient.allCases, Dish.allCases)
func cook(_ ingredient: Ingredient, into dish: Dish) async throws {
  let result = try cook(ingredient)
  try #require(result.isDelicious)
  #expect(result == dish)
}
```



## XCTest

|                  | XCTest                                                       | Swift Testing                                            |
| ---------------- | ------------------------------------------------------------ | -------------------------------------------------------- |
| Function         | "test"로 시작하는 함수명                                     | `@Test`                                                  |
|                  | Instance methods                                             | Instance methods, Static/class methods, Global functions |
| Suites           | class                                                        | struct, actor, class                                     |
|                  | `XCTestCase` 상속                                            | `@Suite`                                                 |
| Before each test | `setUp()`, `setUpWithError() throws`, `setUp() async throws` | `init() async throws`                                    |
| After each test  | `tearDown() async throws`, `tearDownWithError() throws`, `tearDown()` | `deinit`                                                 |
| Sub-groups       | Unsupported                                                  | Via type nesting                                         |

- 마이그레이션
  - Swift Testing, XCTest 같이 사용 가능하기 때문에 타겟 분리는 불필요
  - 비슷한 테스트들은 하나의 parameterized test로 통합
  - 함수명에 "test" 프리픽스 제거
  - UI 테스트나 성능 테스트, Objective-C 코드 테스트는 XCTest 유지