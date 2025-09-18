# Chapter 8: Generics - Write Once, Use Everywhere

Ever written the same function three times for different types? Ever wished you could create a container that works with any type? Generics are your answer. They let you write flexible, reusable code that works with any type while maintaining Swift's strict type safety. Think of them as templates that the compiler fills in with concrete types.

## The Problem Generics Solve

Without generics, you'd write repetitive code:

```swift
// Without generics - repetitive and error-prone
func swapInts(_ a: inout Int, _ b: inout Int) {
    let temp = a
    a = b
    b = temp
}

func swapStrings(_ a: inout String, _ b: inout String) {
    let temp = a
    a = b
    b = temp
}

func swapDoubles(_ a: inout Double, _ b: inout Double) {
    let temp = a
    a = b
    b = temp
}

// With generics - write once, use with any type
func swap<T>(_ a: inout T, _ b: inout T) {
    let temp = a
    a = b
    b = temp
}

// Works with any type!
var x = 5, y = 10
swap(&x, &y)  // x is now 10, y is 5

var hello = "Hello", world = "World"
swap(&hello, &world)  // hello is "World", world is "Hello"
```

## Generic Functions: The Gateway Drug

### Basic Generic Functions

```swift
// T is a type parameter (placeholder for any type)
func identity<T>(_ value: T) -> T {
    return value
}

let number = identity(42)        // T is inferred as Int
let text = identity("Swift")     // T is inferred as String
let array = identity([1, 2, 3])  // T is inferred as [Int]

// Multiple type parameters
func pair<T, U>(_ first: T, _ second: U) -> (T, U) {
    return (first, second)
}

let mixed = pair("Age", 30)  // (String, Int)
```

### Generic Constraints: Adding Requirements

```swift
// T must conform to Comparable
func maximum<T: Comparable>(_ x: T, _ y: T) -> T {
    return x > y ? x : y
}

let greater = maximum(5, 10)  // Works with Int
let later = maximum("apple", "zebra")  // Works with String

// Multiple constraints
func findIndex<T: Equatable>(of value: T, in array: [T]) -> Int? {
    for (index, item) in array.enumerated() {
        if item == value {
            return index
        }
    }
    return nil
}

// Where clauses for complex constraints
func allItemsMatch<C1: Container, C2: Container>(
    _ container1: C1,
    _ container2: C2
) -> Bool where C1.Item == C2.Item, C1.Item: Equatable {
    if container1.count != container2.count {
        return false
    }

    for i in 0..<container1.count {
        if container1[i] != container2[i] {
            return false
        }
    }
    return true
}
```

## Generic Types: Containers for Anything

### Creating Generic Structs

```swift
struct Stack<Element> {
    private var items: [Element] = []

    mutating func push(_ item: Element) {
        items.append(item)
    }

    mutating func pop() -> Element? {
        return items.popLast()
    }

    func peek() -> Element? {
        return items.last
    }

    var isEmpty: Bool {
        return items.isEmpty
    }

    var count: Int {
        return items.count
    }
}

// Type is inferred from usage
var intStack = Stack<Int>()
intStack.push(1)
intStack.push(2)
let top = intStack.pop()  // Optional(2)

var stringStack = Stack<String>()
stringStack.push("Hello")
stringStack.push("World")
```

### Generic Classes

```swift
class LinkedListNode<T> {
    var value: T
    var next: LinkedListNode<T>?

    init(value: T) {
        self.value = value
    }
}

class LinkedList<T> {
    private var head: LinkedListNode<T>?
    private var tail: LinkedListNode<T>?

    func append(_ value: T) {
        let newNode = LinkedListNode(value: value)

        if let tailNode = tail {
            tailNode.next = newNode
        } else {
            head = newNode
        }

        tail = newNode
    }

    func removeFirst() -> T? {
        defer {
            head = head?.next
            if head == nil {
                tail = nil
            }
        }
        return head?.value
    }
}

let list = LinkedList<String>()
list.append("First")
list.append("Second")
```

### Generic Enums

```swift
// Swift's Optional is actually a generic enum!
enum Optional<Wrapped> {
    case none
    case some(Wrapped)
}

// Custom Result type (before Swift had one)
enum Result<Success, Failure: Error> {
    case success(Success)
    case failure(Failure)
}

// Binary tree with generic nodes
enum BinaryTree<T> {
    case empty
    indirect case node(BinaryTree<T>, T, BinaryTree<T>)

    func contains(_ value: T) -> Bool where T: Equatable {
        switch self {
        case .empty:
            return false
        case let .node(left, nodeValue, right):
            return nodeValue == value ||
                   left.contains(value) ||
                   right.contains(value)
        }
    }
}
```

## Extensions with Generics

```swift
// Extend generic types
extension Stack {
    var topItem: Element? {
        return items.last
    }

    func map<U>(_ transform: (Element) -> U) -> Stack<U> {
        var mappedStack = Stack<U>()
        for item in items {
            mappedStack.push(transform(item))
        }
        return mappedStack
    }
}

// Conditional extensions
extension Stack where Element: Equatable {
    func contains(_ item: Element) -> Bool {
        return items.contains(item)
    }
}

extension Stack where Element: Numeric {
    func sum() -> Element {
        return items.reduce(0, +)
    }
}

// Usage
var numbers = Stack<Int>()
numbers.push(1)
numbers.push(2)
numbers.push(3)
let total = numbers.sum()  // Available because Int is Numeric
let hasTwo = numbers.contains(2)  // Available because Int is Equatable
```

## Associated Types: Protocol Generics

```swift
protocol Container {
    associatedtype Item

    var count: Int { get }
    mutating func append(_ item: Item)
    subscript(i: Int) -> Item { get }
}

struct IntContainer: Container {
    // Inferred that Item = Int
    private var items: [Int] = []

    var count: Int { items.count }

    mutating func append(_ item: Int) {
        items.append(item)
    }

    subscript(i: Int) -> Int {
        return items[i]
    }
}

// Generic type conforming to protocol with associated type
struct GenericContainer<T>: Container {
    private var items: [T] = []

    var count: Int { items.count }

    mutating func append(_ item: T) {
        items.append(item)
    }

    subscript(i: Int) -> T {
        return items[i]
    }
}
```

## Advanced Generic Techniques

### Generic Type Aliases

```swift
typealias StringDictionary<T> = Dictionary<String, T>
typealias CompletionHandler<T> = (Result<T, Error>) -> Void

let userDict: StringDictionary<User> = ["alice": User(name: "Alice")]
let completion: CompletionHandler<Data> = { result in
    // Handle result
}
```

### Opaque Types (some)

```swift
// Opaque return types hide implementation details
protocol Shape {
    func area() -> Double
}

struct Square: Shape {
    let side: Double
    func area() -> Double { side * side }
}

struct Circle: Shape {
    let radius: Double
    func area() -> Double { .pi * radius * radius }
}

// 'some' means "some specific type that conforms to Shape"
func makeShape() -> some Shape {
    return Square(side: 10)
    // Type is opaque to callers but consistent
}

// Different from protocol return type
func makeAnyShape() -> Shape {
    // This could return different types on different calls
    return Bool.random() ? Square(side: 10) : Circle(radius: 5)
}
```

### Generic Subscripts

```swift
extension Collection {
    // Safe subscript that returns nil instead of crashing
    subscript(safe index: Index) -> Element? {
        return indices.contains(index) ? self[index] : nil
    }
}

let array = [1, 2, 3]
let value = array[safe: 10]  // nil instead of crash

// Generic subscript with multiple parameters
struct Matrix<T> {
    private var grid: [[T]]

    subscript<S: Sequence>(rows: S) -> [T] where S.Element == Int {
        var result: [T] = []
        for row in rows {
            if row < grid.count {
                result.append(contentsOf: grid[row])
            }
        }
        return result
    }
}
```

### Conditional Conformance

```swift
// Array conforms to Equatable only if its elements do
extension Array: Equatable where Element: Equatable {
    // Implementation provided by Swift
}

// Your own conditional conformance
struct Pair<T, U> {
    let first: T
    let second: U
}

extension Pair: Equatable where T: Equatable, U: Equatable {
    static func == (lhs: Pair, rhs: Pair) -> Bool {
        return lhs.first == rhs.first && lhs.second == rhs.second
    }
}

let pair1 = Pair(first: 1, second: "A")
let pair2 = Pair(first: 1, second: "A")
print(pair1 == pair2)  // true
```

## Real-World Generic Patterns

### The Repository Pattern

```swift
protocol Repository {
    associatedtype Entity

    func findAll() -> [Entity]
    func findById(_ id: String) -> Entity?
    func save(_ entity: Entity)
    func delete(_ id: String)
}

class InMemoryRepository<T>: Repository {
    private var storage: [String: T] = [:]

    func findAll() -> [T] {
        return Array(storage.values)
    }

    func findById(_ id: String) -> T? {
        return storage[id]
    }

    func save(_ entity: T) {
        // Assume entity has an id property somehow
        // This is simplified
        storage[UUID().uuidString] = entity
    }

    func delete(_ id: String) {
        storage.removeValue(forKey: id)
    }
}
```

### Generic Networking

```swift
protocol APIRequest {
    associatedtype Response: Decodable

    var endpoint: String { get }
    var method: String { get }
}

class APIClient {
    func send<Request: APIRequest>(_ request: Request) async throws -> Request.Response {
        let url = URL(string: baseURL + request.endpoint)!
        var urlRequest = URLRequest(url: url)
        urlRequest.httpMethod = request.method

        let (data, _) = try await URLSession.shared.data(for: urlRequest)
        return try JSONDecoder().decode(Request.Response.self, from: data)
    }
}

struct GetUserRequest: APIRequest {
    typealias Response = User

    let userId: Int
    var endpoint: String { "/users/\(userId)" }
    let method = "GET"
}

// Usage
let client = APIClient()
let user = try await client.send(GetUserRequest(userId: 42))
```

### Generic Builders

```swift
class Builder<T> {
    private var instance: T

    init(_ instance: T) {
        self.instance = instance
    }

    func with<V>(_ keyPath: WritableKeyPath<T, V>, value: V) -> Builder<T> {
        instance[keyPath: keyPath] = value
        return self
    }

    func build() -> T {
        return instance
    }
}

struct Person {
    var name: String = ""
    var age: Int = 0
    var email: String = ""
}

let person = Builder(Person())
    .with(\.name, value: "Alice")
    .with(\.age, value: 30)
    .with(\.email, value: "alice@example.com")
    .build()
```

## Performance Considerations

### Specialization

```swift
// Swift automatically specializes generic code for performance
func genericAdd<T: Numeric>(_ a: T, _ b: T) -> T {
    return a + b
}

// At compile time, Swift generates specialized versions:
// func genericAdd(_ a: Int, _ b: Int) -> Int
// func genericAdd(_ a: Double, _ b: Double) -> Double
// etc.
```

### Existential Containers

```swift
// Protocol type (existential) - has overhead
let shapes: [Shape] = [Square(side: 10), Circle(radius: 5)]

// Generic with constraint - more efficient
func processShapes<T: Shape>(_ shapes: [T]) {
    // All shapes are same concrete type
}

// Opaque type - also efficient
func getShape() -> some Shape {
    return Square(side: 10)
}
```

## Common Pitfalls and Solutions

### The "Protocol can only be used as a generic constraint" Error

```swift
protocol Container {
    associatedtype Item
}

// ❌ Cannot use as a type directly
// let container: Container

// ✅ Use as a constraint
func process<C: Container>(_ container: C) {
    // Work with container
}

// ✅ Or use type erasure
struct AnyContainer<Item> {
    // Implementation
}
```

### Generic Type Inference Failures

```swift
// Sometimes Swift can't infer the type
let empty = []  // ❌ Cannot infer type

// Be explicit when needed
let empty: [Int] = []  // ✅
let empty = [Int]()    // ✅
```

## What's Next?

You've mastered generics—the key to writing flexible, reusable Swift code. You can now create types and functions that work with any type while maintaining type safety. This is the foundation of Swift's standard library and most sophisticated Swift code.

Next, we'll explore memory management in Swift. You'll learn about ARC, reference cycles, and how Swift manages memory without a garbage collector—all while keeping you productive and your apps performant.

## Exercises

1. **Generic Cache**: Build a thread-safe, generic cache with expiration times and memory limits.

2. **Type-Safe Database**: Create a generic, type-safe wrapper around a key-value store.

3. **Generic Tree**: Implement a generic tree structure with traversal methods.

4. **Function Composition**: Write generic functions for composing functions (pipe, compose).

Remember: Generics aren't about being clever—they're about writing code once and using it safely with any type. Master them, and you'll write less code that does more.