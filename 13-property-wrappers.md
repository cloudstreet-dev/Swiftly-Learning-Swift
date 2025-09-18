# Chapter 13: Property Wrappers - Custom Magic

You've seen `@State`, `@Published`, and `@ObservedObject` in SwiftUI. These aren't language keywords—they're property wrappers, and you can create your own. Property wrappers let you define custom behavior that's automatically applied to properties, eliminating boilerplate and enforcing patterns. It's like having a tiny, reusable class that manages each property.

## Understanding Property Wrappers

Property wrappers encapsulate property storage and access logic in a reusable way:

```swift
@propertyWrapper
struct Capitalized {
    private var value: String = ""

    var wrappedValue: String {
        get { value }
        set { value = newValue.capitalized }
    }

    init(wrappedValue: String) {
        self.wrappedValue = wrappedValue
    }
}

struct Person {
    @Capitalized var name: String
    @Capitalized var city: String
}

var person = Person(name: "alice smith", city: "new york")
print(person.name)  // "Alice Smith"
print(person.city)  // "New York"

person.name = "bob jones"
print(person.name)  // "Bob Jones"
```

The compiler transforms `@Capitalized var name` into a stored property of type `Capitalized` with computed accessors.

## The Anatomy of a Property Wrapper

### Required Components

```swift
@propertyWrapper
struct Wrapper<T> {
    // Required: The actual wrapped value
    var wrappedValue: T

    // Optional: Projected value (accessed with $)
    var projectedValue: SomeType

    // Optional: Initialization
    init(wrappedValue: T) {
        self.wrappedValue = wrappedValue
    }

    // Optional: Additional initializers
    init(wrappedValue: T, configuration: Config) {
        // Custom setup
    }
}
```

### Validation Wrapper

```swift
@propertyWrapper
struct ValidatedEmail {
    private var email: String = ""

    var wrappedValue: String {
        get { email }
        set {
            if isValidEmail(newValue) {
                email = newValue
            } else {
                print("Invalid email format: \(newValue)")
            }
        }
    }

    init(wrappedValue: String) {
        self.wrappedValue = wrappedValue
    }

    private func isValidEmail(_ email: String) -> Bool {
        let pattern = #"^[A-Z0-9._%+-]+@[A-Z0-9.-]+\.[A-Z]{2,}$"#
        return email.range(of: pattern, options: [.regularExpression, .caseInsensitive]) != nil
    }
}

struct User {
    @ValidatedEmail var email: String
}

var user = User(email: "test@example.com")  // Valid
user.email = "invalid"  // Prints warning, doesn't update
print(user.email)  // Still "test@example.com"
```

## Projected Values: The Dollar Sign Magic

Projected values provide additional functionality through the `$` prefix:

```swift
@propertyWrapper
struct Logged<T> {
    private var value: T
    private(set) var history: [T] = []

    var wrappedValue: T {
        get { value }
        set {
            history.append(value)
            value = newValue
            print("Value changed from \(history.last!) to \(newValue)")
        }
    }

    var projectedValue: [T] {
        return history
    }

    init(wrappedValue: T) {
        self.value = wrappedValue
        self.history = [wrappedValue]
    }
}

struct Configuration {
    @Logged var maxRetries: Int = 3
    @Logged var timeout: Double = 30.0
}

var config = Configuration()
config.maxRetries = 5
config.maxRetries = 10
print(config.$maxRetries)  // [3, 3, 5] - Full history
```

## Common Property Wrapper Patterns

### UserDefaults Wrapper

```swift
@propertyWrapper
struct UserDefault<T> {
    let key: String
    let defaultValue: T
    let store: UserDefaults

    var wrappedValue: T {
        get {
            store.object(forKey: key) as? T ?? defaultValue
        }
        set {
            if let optional = newValue as? AnyOptional, optional.isNil {
                store.removeObject(forKey: key)
            } else {
                store.set(newValue, forKey: key)
            }
        }
    }

    init(wrappedValue: T, _ key: String, store: UserDefaults = .standard) {
        self.key = key
        self.defaultValue = wrappedValue
        self.store = store
    }
}

// Helper protocol for optional handling
private protocol AnyOptional {
    var isNil: Bool { get }
}

extension Optional: AnyOptional {
    var isNil: Bool { self == nil }
}

// Usage
struct Settings {
    @UserDefault("app.theme") var theme = "light"
    @UserDefault("app.fontSize") var fontSize = 14.0
    @UserDefault("user.name") var username: String?
}

var settings = Settings()
settings.theme = "dark"  // Automatically saved to UserDefaults
print(settings.theme)     // Retrieved from UserDefaults
```

### Clamped Values

```swift
@propertyWrapper
struct Clamped<T: Comparable> {
    private var value: T
    private let range: ClosedRange<T>

    var wrappedValue: T {
        get { value }
        set { value = min(max(range.lowerBound, newValue), range.upperBound) }
    }

    init(wrappedValue: T, _ range: ClosedRange<T>) {
        self.range = range
        self.value = min(max(range.lowerBound, wrappedValue), range.upperBound)
    }
}

struct AudioSettings {
    @Clamped(0...100) var volume: Int = 50
    @Clamped(0.5...2.0) var playbackSpeed: Double = 1.0
    @Clamped(-12...12) var pitch: Int = 0
}

var audio = AudioSettings()
audio.volume = 150  // Clamped to 100
print(audio.volume)  // 100

audio.playbackSpeed = 0.1  // Clamped to 0.5
print(audio.playbackSpeed)  // 0.5
```

### Thread-Safe Properties

```swift
@propertyWrapper
class Atomic<T> {
    private var value: T
    private let lock = NSLock()

    var wrappedValue: T {
        get {
            lock.lock()
            defer { lock.unlock() }
            return value
        }
        set {
            lock.lock()
            defer { lock.unlock() }
            value = newValue
        }
    }

    init(wrappedValue: T) {
        self.value = wrappedValue
    }

    // Provide atomic operations
    func mutate(_ transform: (inout T) -> Void) {
        lock.lock()
        defer { lock.unlock() }
        transform(&value)
    }
}

class Counter {
    @Atomic var count = 0

    func increment() {
        _count.mutate { $0 += 1 }  // Thread-safe increment
    }
}

// Multiple threads can safely access
let counter = Counter()
DispatchQueue.concurrentPerform(iterations: 1000) { _ in
    counter.increment()
}
print(counter.count)  // 1000 (no race conditions)
```

### Lazy Initialization

```swift
@propertyWrapper
struct LazyInitialized<T> {
    private var storage: T?
    private let initializer: () -> T

    var wrappedValue: T {
        mutating get {
            if let existing = storage {
                return existing
            }
            let newValue = initializer()
            storage = newValue
            return newValue
        }
        set {
            storage = newValue
        }
    }

    init(wrappedValue: @autoclosure @escaping () -> T) {
        self.initializer = wrappedValue
    }

    init(_ initializer: @escaping () -> T) {
        self.initializer = initializer
    }
}

struct DataManager {
    @LazyInitialized
    var database = ExpensiveDatabase()

    @LazyInitialized
    var cache = {
        print("Creating cache...")
        return Cache(size: 1000)
    }()
}

var manager = DataManager()
// Nothing created yet

let db = manager.database  // ExpensiveDatabase created here
let cache = manager.cache   // Cache created here
```

## Advanced Property Wrapper Techniques

### Composition

```swift
@propertyWrapper
struct Trimmed {
    private var value: String = ""

    var wrappedValue: String {
        get { value }
        set { value = newValue.trimmingCharacters(in: .whitespacesAndNewlines) }
    }

    init(wrappedValue: String) {
        self.wrappedValue = wrappedValue
    }
}

@propertyWrapper
struct Uppercased {
    private var value: String = ""

    var wrappedValue: String {
        get { value }
        set { value = newValue.uppercased() }
    }

    init(wrappedValue: String) {
        self.wrappedValue = wrappedValue
    }
}

// Composition doesn't work directly, but we can nest:
struct NestedWrappers {
    @Trimmed @Uppercased var text = "  hello  "  // ❌ Can't compose directly

    // Instead, create a combined wrapper:
    @TrimmedAndUppercased var properText = "  hello  "
}

@propertyWrapper
struct TrimmedAndUppercased {
    @Trimmed @Uppercased private var value: String

    var wrappedValue: String {
        get { value }
        set { value = newValue }
    }

    init(wrappedValue: String) {
        self.value = wrappedValue
    }
}
```

### Generic Property Wrappers with Constraints

```swift
@propertyWrapper
struct Validated<T> {
    private var value: T
    private let validator: (T) -> Bool
    private let defaultValue: T

    var wrappedValue: T {
        get { value }
        set {
            if validator(newValue) {
                value = newValue
            } else {
                print("Validation failed for value: \(newValue)")
            }
        }
    }

    init(wrappedValue: T, validator: @escaping (T) -> Bool) {
        self.value = wrappedValue
        self.defaultValue = wrappedValue
        self.validator = validator

        // Validate initial value
        if !validator(wrappedValue) {
            fatalError("Initial value failed validation")
        }
    }
}

struct Product {
    @Validated(validator: { $0 > 0 })
    var price: Double = 9.99

    @Validated(validator: { $0.count >= 3 })
    var name: String = "Item"

    @Validated(validator: { (0...1000).contains($0) })
    var quantity: Int = 1
}
```

### Property Wrapper on Computed Properties

```swift
// This doesn't work directly:
// @SomeWrapper var computed: Int {  // ❌ Error
//     get { ... }
//     set { ... }
// }

// Instead, use a backing property:
struct Example {
    @Clamped(0...100) private var _percentage = 50

    var percentage: Int {
        get { _percentage }
        set { _percentage = newValue }
    }
}
```

## Codable with Property Wrappers

```swift
@propertyWrapper
struct DefaultEmpty<T: Collection & ExpressibleByArrayLiteral>: Codable
    where T.ArrayLiteralElement: Codable {

    var wrappedValue: T

    init(wrappedValue: T) {
        self.wrappedValue = wrappedValue
    }

    init(from decoder: Decoder) throws {
        let container = try decoder.singleValueContainer()
        self.wrappedValue = (try? container.decode(T.self)) ?? []
    }

    func encode(to encoder: Encoder) throws {
        var container = encoder.singleValueContainer()
        try container.encode(wrappedValue)
    }
}

struct Response: Codable {
    let id: String
    @DefaultEmpty var tags: [String]  // Defaults to [] if missing
    @DefaultEmpty var metadata: [String: String]  // Defaults to [:] if missing
}

// JSON missing 'tags' and 'metadata' still decodes successfully
let json = #"{"id": "123"}"#
let response = try JSONDecoder().decode(Response.self, from: json.data(using: .utf8)!)
print(response.tags)  // []
print(response.metadata)  // [:]
```

## Debugging Property Wrappers

```swift
@propertyWrapper
struct Debugged<T> {
    private var value: T
    private let name: String

    var wrappedValue: T {
        get {
            print("📖 Getting '\(name)': \(value)")
            return value
        }
        set {
            print("📝 Setting '\(name)': \(value) → \(newValue)")
            value = newValue
        }
    }

    init(wrappedValue: T, _ name: String) {
        self.value = wrappedValue
        self.name = name
        print("🏗 Initialized '\(name)' with: \(wrappedValue)")
    }
}

struct DebuggableModel {
    @Debugged("username") var username = "guest"
    @Debugged("score") var score = 0
}

var model = DebuggableModel()
// 🏗 Initialized 'username' with: guest
// 🏗 Initialized 'score' with: 0

model.username = "player1"
// 📝 Setting 'username': guest → player1

let currentScore = model.score
// 📖 Getting 'score': 0
```

## SwiftUI Property Wrappers Under the Hood

Understanding how SwiftUI's property wrappers work:

```swift
// Simplified @State implementation concept
@propertyWrapper
struct State<Value> {
    private var storage: Storage<Value>

    var wrappedValue: Value {
        get { storage.value }
        nonmutating set {
            storage.value = newValue
            storage.notifyObservers()  // Triggers view update
        }
    }

    var projectedValue: Binding<Value> {
        Binding(
            get: { self.wrappedValue },
            set: { self.wrappedValue = $0 }
        )
    }

    init(wrappedValue: Value) {
        self.storage = Storage(value: wrappedValue)
    }
}

// Simplified @Published concept
@propertyWrapper
class Published<Value> {
    private var value: Value
    private var publisher: PassthroughSubject<Value, Never>

    var wrappedValue: Value {
        get { value }
        set {
            value = newValue
            publisher.send(newValue)
        }
    }

    var projectedValue: AnyPublisher<Value, Never> {
        publisher.eraseToAnyPublisher()
    }

    init(wrappedValue: Value) {
        self.value = wrappedValue
        self.publisher = PassthroughSubject()
    }
}
```

## Best Practices and Pitfalls

### Do's

1. **Keep It Simple**: Property wrappers should have a single, clear responsibility
2. **Document Behavior**: Especially side effects and validation rules
3. **Provide Sensible Defaults**: Make initialization ergonomic
4. **Consider Thread Safety**: Especially for wrappers used across threads
5. **Test Thoroughly**: Property wrappers can hide complex behavior

### Don'ts

1. **Avoid Over-Engineering**: Not everything needs a property wrapper
2. **Don't Hide Too Much**: Magical behavior can confuse users
3. **Beware of Performance**: Wrappers add indirection
4. **Watch Memory**: Class-based wrappers can create retain cycles
5. **Don't Fight the System**: Work with Swift's design, not against it

### Common Pitfalls

```swift
// Pitfall 1: Forgetting mutating for structs
@propertyWrapper
struct BadWrapper {
    private var value: Int = 0

    var wrappedValue: Int {
        get { value }
        set { value = newValue }  // ❌ Error if used in struct
    }
}

// Fix: Make getter nonmutating or use class
@propertyWrapper
class GoodWrapper {
    // ... or use 'nonmutating set' with external storage
}

// Pitfall 2: Initialization order
struct Container {
    @Logged var value: Int = computeValue()  // ❌ Can't call methods

    func computeValue() -> Int { 42 }
}

// Fix: Use lazy or different initialization
struct BetterContainer {
    @Logged lazy var value: Int = computeValue()
}

// Pitfall 3: Encodable/Decodable complexity
// Property wrappers need special handling for Codable
```

## Real-World Example: Building a Preferences System

```swift
@propertyWrapper
struct Preference<T: Codable> {
    let key: String
    let defaultValue: T
    private let container: UserDefaults

    var wrappedValue: T {
        get {
            guard let data = container.data(forKey: key) else {
                return defaultValue
            }
            return (try? JSONDecoder().decode(T.self, from: data)) ?? defaultValue
        }
        set {
            if let data = try? JSONEncoder().encode(newValue) {
                container.set(data, forKey: key)
            }
        }
    }

    var projectedValue: Publisher<T> {
        NotificationCenter.default
            .publisher(for: UserDefaults.didChangeNotification)
            .map { _ in self.wrappedValue }
            .prepend(wrappedValue)
            .removeDuplicates(by: { prev, next in
                // Compare encoded representations
                let encoder = JSONEncoder()
                let prevData = try? encoder.encode(prev)
                let nextData = try? encoder.encode(next)
                return prevData == nextData
            })
            .eraseToAnyPublisher()
    }

    init(wrappedValue: T, _ key: String, container: UserDefaults = .standard) {
        self.key = key
        self.defaultValue = wrappedValue
        self.container = container
    }
}

class AppPreferences: ObservableObject {
    @Preference("theme") var theme = Theme.light
    @Preference("notifications") var notificationsEnabled = true
    @Preference("userData") var userData: UserData?

    init() {
        // Subscribe to changes
        $theme.sink { [weak self] _ in
            self?.objectWillChange.send()
        }
    }
}
```

## What's Next?

You've mastered property wrappers—Swift's way of encapsulating property behaviors in reusable, composable units. You can now eliminate boilerplate, enforce invariants, and create your own property-level abstractions.

In the final chapter, we'll explore Swift's cutting-edge features: macros for code generation, dynamic member lookup, and other advanced capabilities that push the boundaries of what's possible in Swift.

## Exercises

1. **Build a @Cached Wrapper**: Create a property wrapper that caches expensive computations with automatic expiration.

2. **Implement @Debounced**: Design a wrapper that delays updates until a certain time has passed without changes.

3. **Create @Persisted**: Build a wrapper that automatically saves and loads from a SQLite database.

4. **Design @Validated with Throwing**: Make a version that can throw errors instead of silently failing.

Remember: Property wrappers are powerful, but with great power comes great responsibility. Use them to make code cleaner and safer, not more magical and confusing.