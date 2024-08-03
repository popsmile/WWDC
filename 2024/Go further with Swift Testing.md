# Go further with Swift Testing

## Expressive code

### Expectations

Validating an error is throws

```swift
import Testing

@Test func brewTeaError() throws {
  let teaLeaves = TeaLeaves(name: "EarlGrey", optimalBrewTime: 4)
  #expect(throws: BrewingError.self) {
    try teaLeaves.brew(forMinutes: 200) // We don't want this to faile the test!
  }
}

@Test func brewTea() {
  let teaLeaves = TeaLeaves(name: "EarlGrey", optimalBrewTime: 4)
  #expect {
    try teaLeaves.brew(forMinutes: 3)
  } throws: { error in 
    guard let error = error as? BrewingError,
             case let .needsMoreTime(optimalBrewTime) = error else {
               return false
             }
             return optiomalBrewTime == 4
  }
}
```

### Handling known issues

```swift
import Testing

@Test func softServeIceCreamInCone() throws {
  withKnownIssue {
    try softServeMachine.makeSoftServce(in: .cone)
  }
}
```

### Customizing parametrized output

CustomTestStringConvertible

```swift
import Testing

struct SoftServe: CustomTestStringConvertible {
  let flavor: Flavor
  let container: Container
  let toppings: [Topping]
  
  var testDescription: String {
    "\(flavor) in a \(container)"
  }
}
```



## Parameterized testing

- Test functions in Swift Testing can accept multiple inputs
- All combinations of elements from these two collections are automatically tested. 😆👍

```swift
import Testing

@Test(arguments: Ingredient.allCases, Dish.allCases)
func cook(_ ingredient: Ingredient, into dish: Dish) async throws {
  #expect(ingredient.isFresh)
  let result = try cook(ingredient)
  try #require(result.isDelicious)
  try #require(result == dish)
}
```

- zip

```swift
@Test(arguments: zip(Ingredient.allCases, Dish.allCases))
```



## Organizing tests

- Suites can also contains other suites

```swift
import Testing

@Suite("Various desserts")
struct DessertTests {
  @Suite struct WarmDesserts {
    @Test func 따뜻한디저트1() { /*...*/ }
    @Test func 따뜻한디저트2() { /*...*/ }
    @Test func 따뜻한디저트3() { /*...*/ }
  }
  
  @Suite struct ColdDesserts {
    @Test func 시원한디저트1() { /*...*/ }
    @Test func 시원한디저트2() { /*...*/ }
    @Test func 시원한디저트3() { /*...*/ }
  }
}
```

- Declaring and using a tag
  - Helps you run tests across groups

```swift
import Testing

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





## Testing in parallel

