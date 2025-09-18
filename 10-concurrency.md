# Chapter 10: Concurrency - async/await and Beyond

Remember callback hell? Race conditions? Deadlocks? Swift's modern concurrency model says "there's a better way." With async/await, actors, and structured concurrency, Swift makes concurrent programming almost as easy as writing synchronous code—but with all the power of parallelism.

## The Evolution: From Callbacks to async/await

### The Dark Ages: Callback Hell

```swift
// The old way - nested callbacks of doom
func fetchUserData(id: Int) {
    fetchUser(id: id) { user in
        fetchAvatar(for: user) { avatar in
            fetchPosts(for: user) { posts in
                fetchComments(for: posts) { comments in
                    // We're now 4 levels deep in callback hell
                    updateUI(user: user, avatar: avatar, posts: posts, comments: comments)
                }
            }
        }
    }
}
```

### The Renaissance: async/await

```swift
// The Swift way - clean, readable, maintainable
func fetchUserData(id: Int) async throws {
    let user = try await fetchUser(id: id)
    let avatar = try await fetchAvatar(for: user)
    let posts = try await fetchPosts(for: user)
    let comments = try await fetchComments(for: posts)

    await updateUI(user: user, avatar: avatar, posts: posts, comments: comments)
}

// Looks synchronous, runs asynchronous!
```

## Async Functions: The Building Blocks

```swift
// Declaring async functions
func downloadImage(from url: URL) async throws -> UIImage {
    let (data, _) = try await URLSession.shared.data(from: url)
    guard let image = UIImage(data: data) else {
        throw ImageError.invalidData
    }
    return image
}

// Calling async functions
func loadProfileImage() async {
    do {
        let image = try await downloadImage(from: profileURL)
        imageView.image = image
    } catch {
        print("Failed to load image: \(error)")
    }
}

// Async computed properties
struct UserProfile {
    let userId: Int

    var avatarImage: UIImage {
        get async throws {
            let url = try await fetchAvatarURL(for: userId)
            return try await downloadImage(from: url)
        }
    }
}
```

## Tasks: Units of Asynchronous Work

```swift
// Creating tasks
let task = Task {
    let data = try await fetchData()
    return processData(data)
}

// Getting results
let result = await task.value

// Cancellation
task.cancel()

// Checking cancellation
Task {
    for i in 0..<1000 {
        // Check if task was cancelled
        try Task.checkCancellation()

        // Or manually check
        if Task.isCancelled {
            break
        }

        await processItem(i)
    }
}

// Task priorities
Task(priority: .high) {
    await criticalWork()
}

Task(priority: .background) {
    await nonUrgentWork()
}
```

## Structured Concurrency: Order from Chaos

### async let: Parallel Execution

```swift
// Sequential - slow
func loadContentSequential() async throws -> Content {
    let user = try await fetchUser()
    let posts = try await fetchPosts()
    let friends = try await fetchFriends()

    return Content(user: user, posts: posts, friends: friends)
}

// Parallel - fast!
func loadContentParallel() async throws -> Content {
    async let user = fetchUser()
    async let posts = fetchPosts()
    async let friends = fetchFriends()

    // All three run in parallel, wait for all to complete
    return try await Content(user: user, posts: posts, friends: friends)
}

// Conditional parallel execution with racing
func smartFetch(useCache: Bool) async throws -> Data {
    if useCache {
        // Race between cache and server
        return try await withThrowingTaskGroup(of: Data.self) { group in
            group.addTask {
                try await loadFromCache()
            }
            group.addTask {
                try await fetchFromServer()
            }

            // Return first successful result
            if let first = try await group.next() {
                group.cancelAll()  // Cancel the other task
                return first
            }

            throw FetchError.noDataAvailable
        }
    } else {
        return try await fetchFromServer()
    }
}
```

### TaskGroup: Dynamic Concurrency

```swift
// Process multiple items concurrently
func downloadImages(urls: [URL]) async throws -> [UIImage] {
    try await withThrowingTaskGroup(of: UIImage.self) { group in
        // Add tasks to the group
        for url in urls {
            group.addTask {
                try await downloadImage(from: url)
            }
        }

        // Collect results
        var images: [UIImage] = []
        for try await image in group {
            images.append(image)
        }
        return images
    }
}

// With cancellation and error handling
func processItems<T>(_ items: [T]) async throws -> [Result<ProcessedItem, Error>] {
    await withTaskGroup(of: Result<ProcessedItem, Error>.self) { group in
        for item in items {
            group.addTask {
                do {
                    let processed = try await process(item)
                    return .success(processed)
                } catch {
                    return .failure(error)
                }
            }
        }

        var results: [Result<ProcessedItem, Error>] = []
        for await result in group {
            results.append(result)
        }
        return results
    }
}

// Limited concurrency
func downloadWithLimit(urls: [URL], maxConcurrent: Int = 3) async throws -> [Data] {
    try await withThrowingTaskGroup(of: (Int, Data).self) { group in
        var results = Array<Data?>(repeating: nil, count: urls.count)

        // Only run maxConcurrent tasks at once
        for (index, url) in urls.enumerated().prefix(maxConcurrent) {
            group.addTask {
                let data = try await URLSession.shared.data(from: url).0
                return (index, data)
            }
        }

        var nextIndex = maxConcurrent

        for try await (index, data) in group {
            results[index] = data

            if nextIndex < urls.count {
                let urlIndex = nextIndex
                group.addTask {
                    let data = try await URLSession.shared.data(from: urls[urlIndex]).0
                    return (urlIndex, data)
                }
                nextIndex += 1
            }
        }

        return results.compactMap { $0 }
    }
}
```

## Actors: Thread-Safe by Design

Actors protect their mutable state from data races:

```swift
actor BankAccount {
    private var balance: Double = 0

    func deposit(_ amount: Double) {
        balance += amount
    }

    func withdraw(_ amount: Double) throws -> Double {
        guard balance >= amount else {
            throw BankError.insufficientFunds
        }
        balance -= amount
        return amount
    }

    func getBalance() -> Double {
        return balance
    }
}

// Using actors
let account = BankAccount()

Task {
    await account.deposit(100)
    let balance = await account.getBalance()
    print("Balance: \(balance)")
}

// Multiple tasks access safely - no data races!
await withTaskGroup(of: Void.self) { group in
    for _ in 0..<100 {
        group.addTask {
            await account.deposit(1)
        }
    }
}
```

### MainActor: UI Updates Made Safe

```swift
@MainActor
class ViewController: UIViewController {
    // All methods run on main thread by default

    func updateUI() {
        // Guaranteed to run on main thread
        label.text = "Updated"
    }

    nonisolated func backgroundWork() async {
        // This can run on any thread
        let data = await fetchData()

        // Switch to main thread for UI update
        await updateUI()
    }
}

// Marking specific methods/properties
class DataManager {
    @MainActor var progressLabel: UILabel?

    func processInBackground() async {
        // Runs on background thread
        let result = await heavyComputation()

        // Automatically switches to main thread
        await MainActor.run {
            progressLabel?.text = "Complete: \(result)"
        }
    }
}
```

### Global Actors: Custom Thread Isolation

```swift
@globalActor
actor DataActor {
    static let shared = DataActor()
}

@DataActor
class Database {
    private var records: [String: Any] = [:]

    func save(key: String, value: Any) {
        records[key] = value
    }

    func load(key: String) -> Any? {
        return records[key]
    }
}

// All Database operations happen on the same actor
@DataActor
func migrateDatabase() async {
    // This runs on DataActor
}
```

## AsyncSequence and AsyncStream

```swift
// AsyncSequence: for await loops
func readLines(from url: URL) async throws {
    for try await line in url.lines {
        process(line)
    }
}

// Custom AsyncSequence
struct Counter: AsyncSequence {
    typealias Element = Int
    let limit: Int

    struct AsyncIterator: AsyncIteratorProtocol {
        var current = 0
        let limit: Int

        mutating func next() async -> Int? {
            guard current < limit else { return nil }
            defer { current += 1 }

            // Simulate async work
            try? await Task.sleep(nanoseconds: 100_000_000)
            return current
        }
    }

    func makeAsyncIterator() -> AsyncIterator {
        AsyncIterator(limit: limit)
    }
}

// Using custom AsyncSequence
for await number in Counter(limit: 5) {
    print(number)  // Prints 0, 1, 2, 3, 4 with delays
}

// AsyncStream for bridging callbacks
func observeChanges() -> AsyncStream<Change> {
    AsyncStream { continuation in
        let observer = NotificationCenter.default.addObserver(
            forName: .dataChanged,
            object: nil,
            queue: nil
        ) { notification in
            if let change = notification.object as? Change {
                continuation.yield(change)
            }
        }

        continuation.onTermination = { @Sendable _ in
            NotificationCenter.default.removeObserver(observer)
        }
    }
}

// Consuming the stream
for await change in observeChanges() {
    handleChange(change)
}
```

## Sendable: Thread Safety at Compile Time

```swift
// Sendable protocol marks types safe to share across actors
struct User: Sendable {
    let id: Int
    let name: String
    // All stored properties must be Sendable
}

// @Sendable closures
func performAsync(work: @Sendable @escaping () async -> Void) {
    Task {
        await work()
    }
}

// Compiler enforces Sendable
actor DataStore {
    func process(_ user: User) {  // User must be Sendable
        // Process user safely
    }
}

// Non-Sendable warnings
class MutableClass {  // Not Sendable by default
    var count = 0
}

actor SafeActor {
    func unsafeMethod(_ object: MutableClass) {  // ⚠️ Warning: MutableClass is not Sendable
        // Could cause data races
    }
}

// @unchecked Sendable for manual guarantees
class ThreadSafeClass: @unchecked Sendable {
    private let lock = NSLock()
    private var _value = 0

    var value: Int {
        get {
            lock.lock()
            defer { lock.unlock() }
            return _value
        }
        set {
            lock.lock()
            defer { lock.unlock() }
            _value = newValue
        }
    }
}
```

## Continuations: Bridging Old and New

```swift
// Converting callback-based APIs to async
func fetchLegacyData() async throws -> Data {
    try await withCheckedThrowingContinuation { continuation in
        legacyFetch { result in
            switch result {
            case .success(let data):
                continuation.resume(returning: data)
            case .failure(let error):
                continuation.resume(throwing: error)
            }
        }
    }
}

// For delegate patterns
class LocationManager: NSObject, CLLocationManagerDelegate {
    private var continuation: CheckedContinuation<CLLocation, Error>?

    func getCurrentLocation() async throws -> CLLocation {
        try await withCheckedThrowingContinuation { continuation in
            self.continuation = continuation
            locationManager.requestLocation()
        }
    }

    func locationManager(_ manager: CLLocationManager, didUpdateLocations locations: [CLLocation]) {
        if let location = locations.first {
            continuation?.resume(returning: location)
            continuation = nil
        }
    }

    func locationManager(_ manager: CLLocationManager, didFailWithError error: Error) {
        continuation?.resume(throwing: error)
        continuation = nil
    }
}
```

## Real-World Patterns

### The Repository Pattern with Actors

```swift
actor UserRepository {
    private var cache: [Int: User] = [:]
    private let database: Database

    init(database: Database) {
        self.database = database
    }

    func getUser(id: Int) async throws -> User {
        // Check cache first
        if let cached = cache[id] {
            return cached
        }

        // Fetch from database
        let user = try await database.fetchUser(id: id)
        cache[id] = user
        return user
    }

    func saveUser(_ user: User) async throws {
        try await database.saveUser(user)
        cache[user.id] = user
    }
}
```

### Rate Limiting with Actors

```swift
actor RateLimiter {
    private var tokens: Int
    private let maxTokens: Int
    private let refillRate: TimeInterval

    init(maxTokens: Int, refillRate: TimeInterval) {
        self.tokens = maxTokens
        self.maxTokens = maxTokens
        self.refillRate = refillRate

        Task {
            await startRefilling()
        }
    }

    private func startRefilling() async {
        while !Task.isCancelled {
            try? await Task.sleep(nanoseconds: UInt64(refillRate * 1_000_000_000))
            tokens = min(tokens + 1, maxTokens)
        }
    }

    func performRateLimited<T>(_ work: () async throws -> T) async throws -> T {
        while tokens <= 0 {
            try await Task.sleep(nanoseconds: 100_000_000)  // Wait 100ms
        }

        tokens -= 1
        return try await work()
    }
}
```

## Performance and Best Practices

### Avoiding Actor Reentrancy Issues

```swift
actor Counter {
    private var value = 0

    func increment() async {
        let oldValue = value
        // ⚠️ Suspension point - other code might run!
        await someAsyncWork()
        value = oldValue + 1  // Bug: value might have changed
    }

    func safeIncrement() async {
        // Better: minimize suspension points
        await someAsyncWork()
        value += 1  // Atomic operation after suspension
    }
}
```

### Task Local Values

```swift
enum TaskLocals {
    @TaskLocal
    static var userID: Int?

    @TaskLocal
    static var requestID: String?
}

func processRequest() async {
    await TaskLocals.$requestID.withValue(UUID().uuidString) {
        await TaskLocals.$userID.withValue(getCurrentUserID()) {
            // These values are available throughout the task tree
            await performWork()
        }
    }
}

func performWork() async {
    if let requestID = TaskLocals.requestID {
        print("Processing request: \(requestID)")
    }
}
```

## What's Next?

You've mastered Swift's modern concurrency—async/await for clean asynchronous code, actors for thread-safe state, and structured concurrency for predictable parallel execution. No more callback hell, no more data races, just clean, safe concurrent code.

Next: SwiftUI. You'll learn how to build modern, declarative UIs that automatically update when your data changes. It's the future of Apple platform development.

## Exercises

1. **Image Gallery**: Build a concurrent image downloader that limits parallel downloads and shows progress.

2. **Chat System**: Create a real-time chat system using actors for thread-safe message handling.

3. **Web Scraper**: Implement a concurrent web scraper with rate limiting and error recovery.

4. **Cache Manager**: Build an actor-based cache with automatic expiration and memory pressure handling.

Remember: Concurrency isn't about doing everything at once—it's about doing the right things at the right time, safely and efficiently.