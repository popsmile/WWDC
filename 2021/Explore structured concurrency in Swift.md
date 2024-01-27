# Explore structured concurrency in Swift

- Structured Programming
  - Composition
  - Execution flows from top to bottom
  - Intuitive

- Asynchronous code with completion hanlders is unstructured.
  - Cannot use error handling
  - Cannot use loops
- Asynchronous code with async/await is structured.



## Tasks in Swift

- A task provides a new async context for executing code concurrently.
- Swift checks your usage of tasks to help prevent concurrency bugs.
- When calling an async function a task is not created.



## Async-let tasks

- Concurrent bindings

![image-20240127221822728](/Users/pony/Developments/WWDC/2021/images/image-20240127221822728.png)

- Structured concurrency 
  - forms a task tree.
  - guarantees.



## Cancellation

- Cancellation is cooperative
  - Tasks are **not** stopped immediately when cancelld.
  - Cancellation can be checked from anywhere.
  - Design your code with cancellation in mind.

- Cancellation checking

  - `try Task.checkCancellation()`
  - `if Task.isCancelled`

  ```swift
  func fetchThumbnails(for ids: [String]) async throws -> [String: UIImage] {
    var thumbnails: [String: UIImage] = [:]
    for id in ids {
      try Task.checkCancellation() // throws error only if cancelled
      thumbnails[id] = try await fetchOneThumbnail(withID: id)
    }
    return thumbnails
  }
  ```

  

## Group tasks

- A task group is for concurrency with dynamic width.

  - Accessing the results of tasks within a group.

  ```swift
  func fetchThumbnails(for ids: [String]) async throws -> [String: UIImage] {
    var thumbnails: [String: UIImage] = [:]
    try await withThrowingTaskGroup(of: (String, UIImgae).self) { group in 
  		for id in ids {                                                           		
        group.async {
        	return (id, try await fetchOneThumbnail(withID: id))
      	}
      }
  		for try await (id, thumbnail) in group {
        thumbnails[id] = thumbnail
      }
    }                                                         
  }
    return thumbnails
  }
  ```

- Data-race safety

  - Task creation tasks a `@Sendable` closure.
  - Cannot capture mutable variables.
  - Should only capture value types, actors, or classes that implement their own synchronization.



## Unstructured Tasks

- Not all tasks fit a structured pattern.
  - Some tasks need to launch from non-async contexts.
  - Some tasks live beyond the confines of a single scope.
- Inherit actor isolation and priority of the origin context.
- Lifetime is not confined to any scope.
- Can be launched anywhere, even non-async functions.
- Must be manually cancelled or awaited.

```swift
@MainActor
class Mydelegate: UICollectionViewDelegate {
  var thumbnailTasks: [IndexPath: Task<Void, Never>] = [:]
  
  func collectionView(_ view: UICollectionView,
                     willDisplay cell: UICollectionViewCell,
                     forItemAt item: IndexPath) {
    let ids = getThumbnailIDs(for: item)
    thumbnailTasks[item] = Task {
      defer { thumbnail[item] = nil }
      let thumbnails = await fetchThumbnail(for: ids)
      Task.detached(priority: .background) {
        withTaskGroup(of: Void.self) { g in 
          g.async { writeToLocalCache(thumbnails) }
          g.async { log(thumbnails) }
          g.async { ... }
        }
      }
      display(thumbnails, in: cell)
    }
  }
  
  func collectionView(_ view: UICollectionView,
                     didEndDisplay cell: UICollectionViewCell,
                     forItemAt item: IndexPath) {
    thumbnailTasks[item]?.cancel()
  }
}
```

- Detached tasks
  - Unscoped lifetime, manually cancelled and awaited
  - Do not inherit anything from their originating context
  - Optional parameters control priority and other traits



## Flavors of tasks

|                    | Launched by   | Launchable from | Lifetime             | Cancellation | Inherits from origin               |
| ------------------ | ------------- | --------------- | -------------------- | ------------ | ---------------------------------- |
| async-let tasks    | async let x   | async functinos | Scoped to statement  | automatic    | priority, task-local values        |
| Group tasks        | group.async   | withTaskGroup   | scoped to task group | automatic    | priority, task-local values        |
| Unstructured tasks | Task          | anywhere        | unscoped             | via Task     | priority, task-local values, actor |
| Detached tasks     | Task.detached | anywhere        | unscoped             | via Task     | nothing                            |



## Video

https://developer.apple.com/videos/play/wwdc2023/10170/