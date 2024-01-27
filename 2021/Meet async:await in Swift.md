# Meet async/await in Swift

- asynchronous functions: calling it unblocks threads.
- completion handlers are often forgetten to call.
- Swift has no ways to enforce calling completion handlers.

```swift
func tecthThumbnail(for id: String, completion: @escaping (UIImage?, Error?) -> Void) {
  let request = thumnailURLRequest(for: id)
  let task = URLSession.shared.dataTask(with: request) { data, response, error in            
    if let error = error {
      completion(nil, error)
    } else if (response as? HTTPURLResponse)?.statusCode != 200 {
      completion(nil, FetchError.badID)
    } else {
      guard let image = UIImage(data: data!) else {
        completion(nil, FetchError.badImage)
        return
      }
      image.prepareThumbnail(of: CGSize(width: 40, height: 40)) { thumbnail in 
        guard let thumbnail = thumbnail else {
          completion(nil, FetchError.badImage)
          return
        }
      	completion(thumbnail, nil)
      }
    }
  }
}
```



## Async/await

- safer and shorter

```swift
func fetchThumnail(for id: String) async throws -> UIImage {
  let request = thumnailURLRequest(for: id)
  
  // data
  let (data, response) = try await URLSession.shared.data(for: request)
  guard (response as? HTTPURLResponse)?.statusCode == 200 else { throw FetchError.badID }
  
  // UIImage(data:)
  let maybeImage = UIImage(data: data)
  
  // thumbnail
  guard let thumbnail = await maybeImage?.thumbnail else { throw FetchError.badImage }
  
  return thumbnail
}
```

![image-20240127215110408](/Users/pony/Developments/WWDC/2021/images/image-20240127215110408.png)

## Async properties

```swift
extension UIImage {
  var thumbnail: UIImage? {
    get async {
      let size = CGSize(width: 40, height: 40)
      return await self.byPreparingThumbnail(ofSize: size)
    }
  }
}
```

- Only read-only properties can be `async`



## Async sequences

```swift
for await id in staticImageIDsURL.lines {
  let thumbnail = await fetchThumbnail(for: id)
  collage.add(thumbnail)
}
let result = await collage.draw()
```

- `await` works in `for` loops!



## Async/await facts

- `async` enables a function to suspend
- `await` makrs where an async function **may** suspend execution
- Other work can happen during a sespnsion
- Once an awaited async call completes, execution resumes after the `await `



## Adoption async/await

### Testing async code

- `async` makes testing a snap

### Bridgin from sync to async

- `Task` function
- async APIs in the SDK

### Async alternatives and cotinuations

- `try await withCheckedThrowingContinuation`
- `continuation.resume()`
- Continuations must be resumed **excatly once** on every path.
- Discarding the continuation without resuming is not allowed.
- Swift will check you work!