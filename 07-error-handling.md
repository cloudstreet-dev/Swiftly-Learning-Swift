# Chapter 7: Error Handling Without the Drama

Remember catching exceptions in Java, where any method could throw anything at any time? Or JavaScript, where errors silently propagate until your app crashes in production? Swift says "no more surprises." Error handling in Swift is explicit, typed, and designed to make you handle failures gracefully.

## The Swift Way: Explicit and Recoverable

Swift's error handling makes potential failure points visible in your code:

```swift
// You KNOW this function might fail - it's marked with 'throws'
func readFile(at path: String) throws -> String {
    // Implementation
}

// Calling it requires acknowledgment
do {
    let content = try readFile(at: "/path/to/file")
    print(content)
} catch {
    print("Something went wrong: \(error)")
}
```

No hidden exceptions. No surprise crashes. Just clear, explicit error handling.

## Defining Errors: Enums Are Your Friends

In Swift, errors are just types that conform to the `Error` protocol:

```swift
enum FileError: Error {
    case notFound
    case permissionDenied
    case corrupted(bytes: Int)
}

enum NetworkError: Error {
    case timeout
    case noConnection
    case badResponse(statusCode: Int)
    case invalidURL(String)
}

// More descriptive errors with associated values
enum ValidationError: Error {
    case tooShort(minimumLength: Int)
    case tooLong(maximumLength: Int)
    case invalidCharacters(Set<Character>)
    case missingRequiredField(fieldName: String)
    case invalidValue(field: String, value: Any)
}
```

## Throwing Functions: Advertising Failure

Functions that can fail are marked with `throws`:

```swift
struct User {
    let name: String
    let age: Int

    init(name: String, age: Int) throws {
        guard !name.isEmpty else {
            throw ValidationError.missingRequiredField(fieldName: "name")
        }

        guard age >= 0 else {
            throw ValidationError.invalidValue(field: "age", value: age)
        }

        self.name = name
        self.age = age
    }
}

func fetchUser(id: Int) throws -> User {
    guard id > 0 else {
        throw ValidationError.invalidValue(field: "id", value: id)
    }

    // Simulate fetching
    if id == 42 {
        return User(name: "Douglas", age: 42)
    } else {
        throw NetworkError.notFound
    }
}
```

## Handling Errors: Do-Try-Catch

The `do-try-catch` pattern is Swift's structured approach to error handling:

```swift
do {
    let user = try fetchUser(id: 42)
    print("Found user: \(user.name)")
} catch NetworkError.timeout {
    print("Request timed out")
} catch NetworkError.noConnection {
    print("No internet connection")
} catch NetworkError.badResponse(let code) {
    print("Server returned error code: \(code)")
} catch {
    // Generic catch-all (required if not all errors are handled)
    print("An unexpected error occurred: \(error)")
}
```

### Pattern Matching in Catch

```swift
func processFile(at path: String) {
    do {
        let data = try readFile(at: path)
        let processed = try processData(data)
        try saveResults(processed)
    } catch FileError.notFound {
        print("File not found at \(path)")
    } catch FileError.corrupted(let bytes) where bytes > 1000 {
        print("File severely corrupted: \(bytes) bad bytes")
    } catch FileError.corrupted(let bytes) {
        print("File has \(bytes) corrupted bytes")
    } catch let error as NetworkError {
        print("Network error: \(error)")
    } catch {
        print("Unexpected error: \(error)")
    }
}
```

## Try Variations: Choose Your Level of Safety

Swift provides three ways to call throwing functions:

### try - The Standard Way

```swift
// Must be in do-catch or throwing function
do {
    let result = try somethingThatMightFail()
    use(result)
} catch {
    handleError(error)
}
```

### try? - The Optional Way

```swift
// Converts error to nil
let result = try? somethingThatMightFail()
// result is Optional

if let actualResult = try? fetchUser(id: 42) {
    print("Got user: \(actualResult.name)")
} else {
    print("Failed to fetch user")
}

// Great for when you don't care about the specific error
let lines = try? String(contentsOfFile: path).components(separatedBy: "\n")
```

### try! - The Force Way

```swift
// Force-try: Crashes if error is thrown
let result = try! somethingThatDefinitelyWontFail()

// Use only when you're CERTAIN it won't fail
let url = try! URL(string: "https://apple.com")  // Valid URL format
let regex = try! NSRegularExpression(pattern: "[0-9]+")  // Valid regex
```

## Propagating Errors: Let Someone Else Deal With It

You don't always need to handle errors immediately. You can propagate them:

```swift
func loadUserProfile(userId: Int) throws -> Profile {
    let user = try fetchUser(id: userId)  // Propagates error
    let avatar = try loadAvatar(for: user)  // Also propagates
    let posts = try fetchPosts(for: user)  // And this too

    return Profile(user: user, avatar: avatar, posts: posts)
}

// Caller handles all possible errors
do {
    let profile = try loadUserProfile(userId: 123)
    displayProfile(profile)
} catch {
    showErrorMessage(error)
}
```

## Rethrowing Functions: Generic Error Propagation

```swift
// rethrows means: only throws if the closure throws
func withLogging<T>(_ operation: () throws -> T) rethrows -> T {
    print("Starting operation...")
    let result = try operation()
    print("Operation completed successfully")
    return result
}

// Non-throwing usage
let value = withLogging {
    return 42
}

// Throwing usage
do {
    let user = try withLogging {
        try fetchUser(id: 1)
    }
} catch {
    print("Failed: \(error)")
}
```

## Defer: Cleanup Guaranteed

The `defer` statement ensures code runs before scope exit, regardless of how you exit:

```swift
func processFile(_ path: String) throws {
    let file = openFile(path)

    defer {
        // This ALWAYS runs, even if an error is thrown
        closeFile(file)
        print("File closed")
    }

    let data = try readData(from: file)
    let processed = try process(data)

    if processed.isEmpty {
        return  // defer still runs
    }

    try write(processed, to: file)  // defer runs even if this throws
}

// Multiple defers execute in reverse order
func complexOperation() {
    defer { print("3") }  // Runs third
    defer { print("2") }  // Runs second
    defer { print("1") }  // Runs first
    print("Operation")
}
// Prints: Operation, 1, 2, 3
```

## Result Type: Errors as Values

Swift's `Result` type makes error handling functional:

```swift
enum Result<Success, Failure: Error> {
    case success(Success)
    case failure(Failure)
}

func fetchUserAsync(id: Int, completion: @escaping (Result<User, NetworkError>) -> Void) {
    DispatchQueue.global().async {
        // Simulate network delay
        Thread.sleep(forTimeInterval: 1)

        if id > 0 {
            let user = User(name: "User \(id)", age: 25)
            completion(.success(user))
        } else {
            completion(.failure(.invalidRequest))
        }
    }
}

// Using Result
fetchUserAsync(id: 42) { result in
    switch result {
    case .success(let user):
        print("Got user: \(user.name)")
    case .failure(let error):
        print("Failed: \(error)")
    }
}

// Result has useful methods
let result: Result<Int, Error> = .success(42)

let doubled = result.map { $0 * 2 }  // Result<Int, Error>.success(84)
let description = result.mapError { MyError.wrapped($0) }  // Transform error

// Convert between Result and throwing
func getUserResult(id: Int) -> Result<User, Error> {
    return Result { try fetchUser(id: id) }
}

let user = try getUserResult(id: 1).get()  // Throws if failure
```

## Advanced Error Handling Patterns

### Error Transformation

```swift
protocol UserFacing: Error {
    var userMessage: String { get }
}

enum UserError: UserFacing {
    case networkUnavailable
    case invalidCredentials
    case serverError

    var userMessage: String {
        switch self {
        case .networkUnavailable:
            return "Please check your internet connection"
        case .invalidCredentials:
            return "Invalid username or password"
        case .serverError:
            return "Something went wrong. Please try again later"
        }
    }
}

func login(username: String, password: String) throws {
    do {
        try performNetworkLogin(username, password)
    } catch NetworkError.noConnection {
        throw UserError.networkUnavailable
    } catch NetworkError.unauthorized {
        throw UserError.invalidCredentials
    } catch {
        throw UserError.serverError
    }
}
```

### Async Error Handling with async/await

```swift
func fetchUserAsync(id: Int) async throws -> User {
    guard id > 0 else {
        throw ValidationError.invalidId
    }

    let response = try await URLSession.shared.data(from: userURL(id))
    return try JSONDecoder().decode(User.self, from: response.0)
}

// Using async throwing functions
Task {
    do {
        let user = try await fetchUserAsync(id: 42)
        print("Got user: \(user.name)")
    } catch {
        print("Failed to fetch user: \(error)")
    }
}
```

### LocalizedError for User-Friendly Messages

```swift
enum DataError: LocalizedError {
    case fileNotFound(String)
    case invalidFormat
    case insufficientPermissions

    var errorDescription: String? {
        switch self {
        case .fileNotFound(let filename):
            return "Could not find file: \(filename)"
        case .invalidFormat:
            return "The file format is invalid"
        case .insufficientPermissions:
            return "You don't have permission to access this file"
        }
    }

    var failureReason: String? {
        switch self {
        case .fileNotFound:
            return "The file may have been moved or deleted"
        case .invalidFormat:
            return "The file may be corrupted"
        case .insufficientPermissions:
            return "Your account lacks the necessary privileges"
        }
    }

    var recoverySuggestion: String? {
        switch self {
        case .fileNotFound:
            return "Check the file path and try again"
        case .invalidFormat:
            return "Try opening the file in a different application"
        case .insufficientPermissions:
            return "Contact your administrator for access"
        }
    }
}
```

## Testing Error Conditions

```swift
import XCTest

class ErrorTests: XCTestCase {
    func testInvalidUserCreation() {
        XCTAssertThrowsError(try User(name: "", age: 25)) { error in
            guard let validationError = error as? ValidationError else {
                XCTFail("Wrong error type")
                return
            }

            if case .missingRequiredField(let field) = validationError {
                XCTAssertEqual(field, "name")
            } else {
                XCTFail("Wrong validation error")
            }
        }
    }

    func testValidUserCreation() {
        XCTAssertNoThrow(try User(name: "Alice", age: 30))
    }
}
```

## Best Practices and Common Patterns

### The Guard Pattern

```swift
func processData(_ data: Data?) throws -> ProcessedData {
    guard let data = data else {
        throw DataError.noData
    }

    guard data.count > 0 else {
        throw DataError.emptyData
    }

    guard data.count < 1_000_000 else {
        throw DataError.dataTooLarge
    }

    return try actuallyProcess(data)
}
```

### Error Aggregation

```swift
struct ValidationErrors: Error {
    let errors: [ValidationError]

    static func validate(_ user: User) -> ValidationErrors? {
        var errors: [ValidationError] = []

        if user.name.isEmpty {
            errors.append(.missingRequiredField(fieldName: "name"))
        }

        if user.age < 0 {
            errors.append(.invalidValue(field: "age", value: user.age))
        }

        if user.email.isEmpty {
            errors.append(.missingRequiredField(fieldName: "email"))
        }

        return errors.isEmpty ? nil : ValidationErrors(errors: errors)
    }
}
```

### The Railway Pattern

```swift
func processUser(id: Int) -> Result<ProcessedUser, Error> {
    return Result { try fetchUser(id: id) }
        .flatMap { user in validateUser(user) }
        .flatMap { validUser in enrichUser(validUser) }
        .map { enrichedUser in ProcessedUser(from: enrichedUser) }
}
```

## What's Next?

You've mastered Swift's error handling—a system that makes failures explicit, recoverable, and manageable. No more silent failures, no more unexpected crashes, just clear communication about what can go wrong and how to handle it.

Next up: Generics. You'll learn how to write code once and use it everywhere, creating flexible, reusable components that work with any type while maintaining type safety.

## Exercises

1. **Error Chain**: Create a file processing pipeline that propagates and transforms errors through multiple stages.

2. **Retry Logic**: Implement a generic retry mechanism that attempts an operation multiple times before giving up.

3. **Error Logger**: Build an error logging system that categorizes and tracks different types of errors.

4. **Validation Framework**: Create a validation framework using protocols and error handling for form validation.

Remember: In Swift, errors aren't exceptions—they're expected outcomes. Plan for them, handle them gracefully, and your code will be more robust and user-friendly.