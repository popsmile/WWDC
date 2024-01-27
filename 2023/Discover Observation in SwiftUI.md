# Observation

- What is Observation?
- SwiftUI property wrappers
- Advanced uses
- ObservableObject



## What is Observation?

- `@Observable`
  - Macro
  - Tracks access
  - Property changes cause UI updates
- in new Swift 5.9
- don't need other property wrappers

```swift
@Observable class FoodTruckModel {
  var orders: [Order] = []
  var donuts = Donut.all
  var orderCount: Int { orders.count }
}

struct DonutMenu: View {
  let model: FoodTruckModel
  
  var body: some View {
    List {
      Section("Donuts") {
        ForEach(model.donuts) { donut in 
          Text(donet.name)
        }
        Button("Add new donut") {
          model.addDonut()
        }
      }
    }
  }
}
```



## SwiftUI property wrappers

- `@State`
  - part of the view
- `@Environment`
  - global to application
- `@Bindable`
  - just need bindings
  - Lightweight
  - connect reference to UI
  - uses $ syntax



## Advanced Uses

- computed properties

  - SwfitUI tracks access
  - composed properties
  - manual control when needed

  ```swift
  @Observable class Donut {
    var name: String {
      get {
        access(keyPath: \.name)
        return someNonObservableLocation.name
      }
      set {
        withMutation(keyPath: \.name) {
          someNonObservableLocation.name = newValue
        }
      }
    }
  }
  ```



## ObservableObject

### Migration ObservableObject to `@Observable`

- just delete annotations
- simplify three primary property wrappers
  - `@State`
  - `@Environment`
  - `@Bindable`

- Before

  ```swift
  public class FoodTruckModel: ObservableObject {
    @Published public var truck = Truck()
    @Published public var orders: [Order] = []
    @Published public var donuts = Donut.all
    
    var dailyOrderSumaries: [City.ID: [OrderSummary]] = [:]
      var monthlyOrderSumaries: [City.ID: [OrderSummary]] = [:]
  }
  ```

- After

  - remove `ObservableObject`
  - remove `@Published`
  - mark `@Observable`

  ```swift
  @Observable public class FoodTruckModel {
    public var truck = Truck()
    public var orders: [Order] = []
    public var donuts = Donut.all
    
    var dailyOrderSumaries: [City.ID: [OrderSummary]] = [:]
      var monthlyOrderSumaries: [City.ID: [OrderSummary]] = [:]
  }
  ```

  

## Harness the magic

- Macro for observing changes
- New projects use @Observable
- Simplify code for new features



## Video

- https://developer.apple.com/videos/play/wwdc2023/10149/