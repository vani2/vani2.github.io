---
tags: [ios, network]
date: "2021-07-16T12:23:00Z"
title: Update existing library for async await
---

Async-await is a great new way to work with concurrency.

Let's add async-await support for [Apexy](https://github.com/RedMadRobot/apexy-ios) – library for organizing a network layer. The changes I describe you can find in [this PR](https://github.com/RedMadRobot/apexy-ios/pull/28).

<!--more-->

## Watch sessions and install Xcode 13 beta

Install latest Xcode 13 beta. It's expected that API changes from one beta to another. Also, the new API is not available for older OS versions. If you're not familiar with async-await, watching WWDC sessions this year is highly recommended! Described changes are updated for Xcode 13 beta 4.

## Inspect existing methods

This is a high-level diagram of the library:

![Apexy](/assets/2021-07-16/apexy-uml.jpg)

I'll start from the Client after we'll move to Service and View Controller changes.
In Apexy, we have the following method for the request:

```swift
public protocol Client: AnyObject {
    
    /// Send request to specified endpoint.
    ///
    /// - Parameters:
    ///   - endpoint: endpoint of remote content.
    ///   - completionHandler: The completion closure to be executed when request is completed.
    /// - Returns: The progress of fetching the response data from the server for the request.
    func request<T>(
        _ endpoint: T,
        completionHandler: @escaping (APIResult<T.Content>) -> Void
    ) -> Progress where T: Endpoint
}
```

We return the result in the `completionHandler` and cancel the request by calling `cancel()` for `Progress`.
So let's add a new method for the request, but with async-await:

```swift
@available(macOS 12, iOS 15, watchOS 8, tvOS 15, *)
func request<T>(_ endpoint: T) async throws -> T.Content where T: Endpoint

```

Notice we don't need a completion handler and progress with async-await API.

### Alamofire

To conform new method requirement with Alamofire (version 5.0), we can wrap our existing method with a completion block:

```swift
open class AlamofireClient: Client {

    // <...>

    @available(macOS 12, iOS 15, watchOS 8, tvOS 15, *)
    public func request<T>(_ endpoint: T) async throws -> T.Content where T : Endpoint {
        typealias ContentContinuation = CheckedContinuation<T.Content, Error>
        return try await withCheckedThrowingContinuation { (continuation: ContentContinuation) in
            _ = request(endpoint) { continuation.resume(with: $0) }
        }
    }

    // <...>

}
```

Looks easy, right? But with this approach, we are not able to cancel our request.
So we should use function `withTaskCancellationHandler` instead of `withCheckedThrowingContinuation`. We need to get progress reference from blocked-based request method call and cancel it later if needed.

```swift
@available(macOS 12, iOS 15, watchOS 8, tvOS 15, *)
public func request<T>(_ endpoint: T) async throws -> T.Content where T : Endpoint {
    typealias ContentContinuation = CheckedContinuation<T.Content, Error>
    var progress: Progress?
    return try await withTaskCancellationHandler(handler: {
        progress?.cancel() // Error: Reference to captured var 'progress' in concurrently-executing code
    }, operation: {
        try await withCheckedThrowingContinuation { (continuation: ContentContinuation) in
            progress = request(endpoint) { continuation.resume(with: $0) }
        }
    })
}
```

But we got the error `Reference to captured var 'progress' in concurrently-executing code`. This is because `handler` marked `@Sendable`.

```swift
public func withTaskCancellationHandler<T>(handler: @Sendable () -> (), operation: () async throws -> T) async rethrows -> T
```

Read [more about @Sendable](https://github.com/apple/swift-evolution/blob/main/proposals/0302-concurrent-value-and-concurrent-closures.md#new-sendable-attribute-for-functions).

Here I did the same trick: wrapped progress by a class.

```swift
private final class ProgressWrapper {
    var progress: Progress?
}
```

Now it compiles without error.

```swift
@available(macOS 12, iOS 15, watchOS 8, tvOS 15, *)
public func request<T>(_ endpoint: T) async throws -> T.Content where T : Endpoint {
    typealias ContentContinuation = CheckedContinuation<T.Content, Error>
    let progressWrapper = ProgressWrapper()
    return try await withTaskCancellationHandler(handler: {
        progressWrapper.progress?.cancel()
    }, operation: {
        try await withCheckedThrowingContinuation { (continuation: ContentContinuation) in
            progressWrapper.progress = request(endpoint) { continuation.resume(with: $0) }
        }
    })
}
```

But that's concurrent code. So let's make it thread-safely and add a sync mechanism to `ProgressWrapper`.

```swift
private final class ProgressWrapper {

    var progress: Progress? {
        get {
            lock.lock()
            defer { lock.unlock() }
            return _progress
        }
        set {
            lock.lock()
            defer { lock.unlock() }
            _progress = newValue
        }
    }

    private var _progress: Progress?
    private let lock = NSLock()

    func cancel() {
        progress?.cancel()
    }
}
```

The final method implementation is

```swift
@available(macOS 12, iOS 15, watchOS 8, tvOS 15, *)
public func request<T>(_ endpoint: T) async throws -> T.Content where T : Endpoint {
    typealias ContentContinuation = CheckedContinuation<T.Content, Error>
    let progressWrapper = ProgressWrapper()
    return try await withTaskCancellationHandler(handler: {
        progressWrapper.cancel()
    }, operation: {
        try await withCheckedThrowingContinuation { (continuation: ContentContinuation) in
            progressWrapper.progress = request(endpoint) { continuation.resume(with: $0) }
        }
    })
}
```

### URLSession

With `URLSession`, this is a different story. `URLSession` has a new method with async-await support:

```swift
public func data(for request: URLRequest, delegate: URLSessionTaskDelegate? = nil) async throws -> (Data, URLResponse)
```

I rewrote it without using the existing block-based method. The first version was

```swift
open func request<T>(_ endpoint: T) async throws -> T.Content where T : Endpoint {
    var request = try endpoint.makeRequest()
    request = try requestAdapter.adapt(request)

    do {
        response = try await session.data(for: request)
        
        if let httpResponse = response?.response as? HTTPURLResponse {
            try endpoint.validate(request, response: httpResponse, data: response?.data)
        }
        
        let data = response?.data ?? Data()
        return try endpoint.content(from: response?.response, with: data)
    } catch {
        throw error
    }
}
```

That looks almost the same as a block-based one but shorter and more straightforward to read.
Only one thing is forgotten here is calling `responseObserver`. So we observe each request and response result with the same handler, for instance, to add logging to console into one place.
I want to add this above `do-catch` block

```swift
open func request<T>(_ endpoint: T) async throws -> T.Content where T : Endpoint {
    var request = try endpoint.makeRequest()
    request = try requestAdapter.adapt(request)

    defer {
        completionQueue.async {
            //  // Error: Reference to captured var <...> in concurrently-executing code
            self.responseObserver?(request, response as? HTTPURLResponse, response?.data, error)
        }
    }
    //<...>
}
```

But I got the same errors about concurrently-execution code. The same trick can help us. I created `ResponseObserverWrapper`.

```swift
private struct ResponseObserverWrapper {
    var request: URLRequest
    var data: Data?
    var response: URLResponse?
    var error: Error?
}
```

The final implementation looks like this:

```swift
@available(macOS 12, iOS 15, watchOS 8, tvOS 15, *)
open func request<T>(_ endpoint: T) async throws -> T.Content where T : Endpoint {
    var request = try endpoint.makeRequest()
    request = try requestAdapter.adapt(request)
    var response: (data: Data, response: URLResponse)?
    var error: Error?
    
    defer {
        let wrapper = ResponseObserverWrapper(request: request, data: response?.data, response: response?.response, error: error)
        completionQueue.async {
            self.responseObserver?(wrapper.request, wrapper.response as? HTTPURLResponse, wrapper.data, wrapper.error)
        }
    }
    
    do {
        response = try await session.data(for: request)
        
        if let httpResponse = response?.response as? HTTPURLResponse {
            try endpoint.validate(request, response: httpResponse, data: response?.data)
        }
        
        let data = response?.data ?? Data()
        return try endpoint.content(from: response?.response, with: data)
    } catch let someError {
        error = someError
        throw someError
    }
}
```

## Services

On the higher level of abstraction, there is a service that uses a client.
In our example project, this is `BookService`. So I added a new method with async-await API right below the block-based one.

```swift
protocol BookService {
    
    @discardableResult
    func fetchBooks(completion: @escaping (Result<[Book], Error>) -> Void) -> Progress
    
    @available(macOS 12, iOS 15, watchOS 8, tvOS 15, *)
    func fetchBooks() async throws -> [Book]
}
```

Implementation should use our recently added method:

```swift
final class BookServiceImpl: BookService {
    
    let apiClient: Client
    
    // <...>

    @available(macOS 12, iOS 15, watchOS 8, tvOS 15, *)
    func fetchBooks() async throws -> [Book] {
        let endpoint = BookListEndpoint()
        return try await apiClient.request(endpoint)
    }
}
```

## Using service in View Controller

Example View Controller holds a book service and performs a block-based method call or new async-await

```swift
class ViewController: UIViewController {
    
    private let bookService: BookService = // <...>
    private var task: Any? // to cancel it later, type is Any because properties cannot be marked as @available
    private var progress: Progress?

    // <...>

    @IBAction private func performRequest() {
        // show acitivity indicator
        
        guard #available(macOS 12, iOS 15, watchOS 8, tvOS 15, *) else { performLegacyRequest(); return }
        
        task = Task {
            do {
                let books = try await bookService.fetchBooks()
                show(books: books)
            } catch {
                show(error: error)
            }
            // hide activity indicator
        }
    }

    @IBAction private func cancel() {
        if #available(macOS 12, iOS 15, watchOS 8, tvOS 15, *) {
            (task as? Task<Void, Never>)?.cancel()
        } else {
            progress?.cancel()
        }
    }

}
```

While canceling, we have two options: call `cancel()` to `Progress` or `Task`.
So, that's it! You can find all changes in [this PR](https://github.com/RedMadRobot/apexy-ios/pull/28). Also, I implemented a method for upload, but they are very similar.
I hope you found this explanation helpful. Thank you!
