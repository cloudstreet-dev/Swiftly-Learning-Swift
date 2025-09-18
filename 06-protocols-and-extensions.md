# Chapter 6: Protocols and Extensions - Swift's Secret Weapons

If you're coming from Java or C#, you know interfaces. If you're from Ruby or Python, you know mixins. Swift's protocols are like interfaces that went to the gym, got swole, and learned magic tricks. Combined with extensions, they form the foundation of Swift's "protocol-oriented programming" paradigm—and once you get it, you'll never want to go back to traditional OOP.

## Protocols: Contracts with Benefits

A protocol defines a blueprint of methods, properties, and other requirements. It's like saying "anything that claims to be X must be able to do Y and Z."

### The Basics

```swift
protocol Vehicle {
    var numberOfWheels: Int { get }
    var currentSpeed: Double { get set }

    func startEngine()
    func stopEngine()
}

struct Car: Vehicle {
    let numberOfWheels = 4
    var currentSpeed = 0.0

    func startEngine() {
        print("Vroom vroom!")
    }

    func stopEngine() {
        print("Engine stopped")
    }
}

struct Bicycle: Vehicle {
    let numberOfWheels = 2
    var currentSpeed = 0.0

    func startEngine() {
        print("Bicycles don't have engines, using leg power!")
    }

    func stopEngine() {
        print("Just stop pedaling")
    }
}
```

### Property Requirements

```swift
protocol Named {
    var name: String { get }  // Read-only requirement
}

protocol Aged {
    var age: Int { get set }  // Read-write requirement
}

// Can satisfy read-only with computed property
struct Person: Named, Aged {
    let firstName: String
    let lastName: String

    var name: String {  // Computed property satisfying Named
        return "\(firstName) \(lastName)"
    }

    var age: Int  // Stored property satisfying Aged
}

// Can provide setter even if protocol only requires getter
struct Company: Named {
    var name: String  // Read-write property satisfies read-only requirement
}
```

### Method Requirements and Mutating

```swift
protocol Togglable {
    mutating func toggle()  // 'mutating' required for structs
}

enum Switch: Togglable {
    case on, off

    mutating func toggle() {
        self = (self == .on) ? .off : .on
    }
}

class Light: Togglable {
    var isOn = false

    func toggle() {  // No 'mutating' needed for classes
        isOn.toggle()
    }
}
```

### Initializer Requirements

```swift
protocol Initializable {
    init(value: Int)
}

class BaseClass: Initializable {
    var value: Int

    required init(value: Int) {  // 'required' ensures subclasses implement it
        self.value = value
    }
}

struct SimpleStruct: Initializable {
    var value: Int

    init(value: Int) {  // No 'required' needed for structs
        self.value = value
    }
}
```

## Protocol Composition: Mix and Match

Swift lets you combine protocols on the fly:

```swift
protocol Payable {
    func calculateWages() -> Double
}

protocol TimeTrackable {
    var hoursWorked: Double { get set }
}

protocol Employee: Payable, TimeTrackable {  // Protocol inheritance
    var id: String { get }
}

// Use multiple protocols as a type
func processWorker(_ worker: Payable & TimeTrackable) {
    let wages = worker.calculateWages()
    print("Wages for \(worker.hoursWorked) hours: $\(wages)")
}

// Even cooler: combine with concrete types
protocol Drawable {
    func draw()
}

class Shape {
    var color: String = "red"
}

class Circle: Shape, Drawable {
    func draw() {
        print("Drawing a \(color) circle")
    }
}

// Type that must be a Shape subclass AND conform to Drawable
func render(_ shape: Shape & Drawable) {
    print("Rendering shape with color: \(shape.color)")
    shape.draw()
}
```

## Extensions: Retrofit Awesomeness

Extensions add new functionality to existing types—even types you don't own!

### Extending Your Own Types

```swift
struct Point {
    var x: Double
    var y: Double
}

extension Point {
    // Add computed properties
    var magnitude: Double {
        return sqrt(x * x + y * y)
    }

    // Add methods
    func distance(to other: Point) -> Double {
        let dx = x - other.x
        let dy = y - other.y
        return sqrt(dx * dx + dy * dy)
    }

    // Add initializers
    init(magnitude: Double, angle: Double) {
        self.x = magnitude * cos(angle)
        self.y = magnitude * sin(angle)
    }

    // Add mutating methods
    mutating func moveBy(x deltaX: Double, y deltaY: Double) {
        x += deltaX
        y += deltaY
    }
}

var point = Point(x: 3, y: 4)
print(point.magnitude)  // 5.0
```

### Extending Types You Don't Own

```swift
extension Int {
    var isEven: Bool {
        return self % 2 == 0
    }

    var squared: Int {
        return self * self
    }

    func times(_ closure: () -> Void) {
        for _ in 0..<self {
            closure()
        }
    }
}

print(42.isEven)     // true
print(5.squared)     // 25

3.times {
    print("Swift is cool!")
}

extension String {
    var words: [String] {
        return self.components(separatedBy: .whitespacesAndNewlines)
            .filter { !$0.isEmpty }
    }

    func truncated(to length: Int, trailing: String = "...") -> String {
        if count <= length {
            return self
        }
        return String(prefix(length)) + trailing
    }
}

let sentence = "Swift protocols are amazing"
print(sentence.words)  // ["Swift", "protocols", "are", "amazing"]
print(sentence.truncated(to: 10))  // "Swift prot..."
```

### Extension with Constraints

```swift
extension Collection where Element: Numeric {
    func sum() -> Element {
        return reduce(0, +)
    }
}

extension Collection where Element: Comparable {
    func isSorted(ascending: Bool = true) -> Bool {
        zip(self, dropFirst()).allSatisfy { ascending ? $0 <= $1 : $0 >= $1 }
    }
}

let numbers = [1, 2, 3, 4, 5]
print(numbers.sum())  // 15
print(numbers.isSorted())  // true

// Won't work on non-numeric collections
// let strings = ["a", "b", "c"]
// strings.sum()  // ❌ Compilation error
```

## Protocol Extensions: Default Implementations

This is where the magic happens. Protocol extensions provide default implementations:

```swift
protocol Greetable {
    var name: String { get }
    func greet()
}

extension Greetable {
    // Default implementation
    func greet() {
        print("Hello, \(name)!")
    }

    // Add new methods to all conforming types
    func farewell() {
        print("Goodbye, \(name)!")
    }
}

struct Person: Greetable {
    let name: String
    // Gets greet() and farewell() for free!
}

struct Robot: Greetable {
    let name: String

    // Override default implementation
    func greet() {
        print("BEEP BOOP, HELLO \(name.uppercased())")
    }
    // Still gets farewell() for free
}

let human = Person(name: "Alice")
human.greet()      // "Hello, Alice!"
human.farewell()   // "Goodbye, Alice!"

let robot = Robot(name: "R2D2")
robot.greet()      // "BEEP BOOP, HELLO R2D2"
robot.farewell()   // "Goodbye, R2D2!"
```

## Associated Types: Generic Protocols

Protocols can have placeholder types that conforming types fill in:

```swift
protocol Container {
    associatedtype Item

    var count: Int { get }
    mutating func append(_ item: Item)
    subscript(i: Int) -> Item { get }
}

struct IntStack: Container {
    // Inferred that Item = Int
    private var items: [Int] = []

    var count: Int {
        return items.count
    }

    mutating func append(_ item: Int) {
        items.append(item)
    }

    subscript(i: Int) -> Int {
        return items[i]
    }
}

// Generic struct conforming to Container
struct Stack<Element>: Container {
    private var items: [Element] = []

    var count: Int {
        return items.count
    }

    mutating func append(_ item: Element) {
        items.append(item)
    }

    subscript(i: Int) -> Element {
        return items[i]
    }
}

// Using where clauses with associated types
extension Container where Item: Equatable {
    func contains(_ item: Item) -> Bool {
        for i in 0..<count {
            if self[i] == item {
                return true
            }
        }
        return false
    }
}
```

## Protocol-Oriented Programming in Action

### Building a Flexible System

```swift
// Define capabilities as protocols
protocol Flyable {
    var airspeedVelocity: Double { get }
    func fly()
}

protocol Swimmable {
    var maxDepth: Double { get }
    func swim()
}

protocol Predator {
    func hunt()
}

// Provide default implementations
extension Flyable {
    func fly() {
        print("Flying at \(airspeedVelocity) mph")
    }
}

extension Swimmable {
    func swim() {
        print("Swimming to depth of \(maxDepth) meters")
    }
}

// Compose behaviors
struct Duck: Flyable, Swimmable {
    let airspeedVelocity = 55.0
    let maxDepth = 2.0
}

struct Penguin: Swimmable, Predator {
    let maxDepth = 100.0

    func hunt() {
        swim()
        print("Catching fish!")
    }
}

struct Eagle: Flyable, Predator {
    let airspeedVelocity = 120.0

    func hunt() {
        fly()
        print("Swooping down on prey!")
    }
}

// Process any flying creature
func migrate<T: Flyable>(_ creature: T) {
    print("Migrating south...")
    creature.fly()
}
```

### Real-World Example: Building a Network Layer

```swift
protocol Request {
    associatedtype Response: Decodable

    var path: String { get }
    var method: HTTPMethod { get }
    var parameters: [String: Any]? { get }
}

extension Request {
    var method: HTTPMethod { .get }  // Default to GET
    var parameters: [String: Any]? { nil }  // Default to no parameters
}

protocol NetworkClient {
    func send<R: Request>(_ request: R) async throws -> R.Response
}

// Concrete request
struct UserRequest: Request {
    typealias Response = User

    let userId: Int
    var path: String { "/users/\(userId)" }
}

struct CreatePostRequest: Request {
    typealias Response = Post

    let title: String
    let content: String

    var path: String { "/posts" }
    var method: HTTPMethod { .post }
    var parameters: [String: Any]? {
        ["title": title, "content": content]
    }
}

// Mock implementation
class MockNetworkClient: NetworkClient {
    func send<R: Request>(_ request: R) async throws -> R.Response {
        // Simulate network delay
        try await Task.sleep(nanoseconds: 100_000_000)

        // Return mock data based on request type
        if R.Response.self == User.self {
            return User(id: 1, name: "Alice") as! R.Response
        }
        // ... handle other types

        throw NetworkError.notImplemented
    }
}
```

## Advanced Protocol Techniques

### Self Requirements

```swift
protocol Copyable {
    func copy() -> Self  // Returns the conforming type
}

class MyClass: Copyable {
    var value: Int

    init(value: Int) {
        self.value = value
    }

    func copy() -> Self {
        return type(of: self).init(value: value)
    }

    required init(value: Int) {
        self.value = value
    }
}
```

### Protocol Witnesses (Type Erasure)

```swift
// Problem: Can't use protocol with associated types directly
// let container: Container  // ❌ Error!

// Solution: Type erasure
struct AnyContainer<Item>: Container {
    private let _count: () -> Int
    private let _append: (Item) -> Void
    private let _subscript: (Int) -> Item

    init<C: Container>(_ container: C) where C.Item == Item {
        var mutableContainer = container
        _count = { container.count }
        _append = { mutableContainer.append($0) }
        _subscript = { container[$0] }
    }

    var count: Int { _count() }
    mutating func append(_ item: Item) { _append(item) }
    subscript(i: Int) -> Item { _subscript(i) }
}

// Now we can use it
let intStack = Stack<Int>()
let anyContainer: AnyContainer<Int> = AnyContainer(intStack)
```

### Conditional Conformance

```swift
extension Array: TextRepresentable where Element: TextRepresentable {
    var textualDescription: String {
        let itemsAsText = self.map { $0.textualDescription }
        return "[" + itemsAsText.joined(separator: ", ") + "]"
    }
}
// Array conforms to TextRepresentable only when elements do
```

## Common Patterns and Best Practices

### The Delegate Pattern

```swift
protocol TableViewDelegate: AnyObject {
    func didSelectRow(at index: Int)
    func numberOfRows() -> Int
}

class TableView {
    weak var delegate: TableViewDelegate?

    func userTappedRow(at index: Int) {
        delegate?.didSelectRow(at: index)
    }
}
```

### The Strategy Pattern

```swift
protocol SortStrategy {
    func sort<T: Comparable>(_ array: [T]) -> [T]
}

struct QuickSort: SortStrategy {
    func sort<T: Comparable>(_ array: [T]) -> [T] {
        // Quick sort implementation
        return array.sorted()
    }
}

struct MergeSort: SortStrategy {
    func sort<T: Comparable>(_ array: [T]) -> [T] {
        // Merge sort implementation
        return array.sorted()
    }
}

class DataProcessor {
    var sortStrategy: SortStrategy = QuickSort()

    func process<T: Comparable>(_ data: [T]) -> [T] {
        return sortStrategy.sort(data)
    }
}
```

## What's Next?

You've mastered protocols and extensions—the building blocks of protocol-oriented programming. You've seen how they enable composition over inheritance, provide flexible abstractions, and let you extend any type with new capabilities.

Next, we'll tackle error handling in Swift. No more silent failures or cryptic exceptions—Swift's error handling is explicit, recoverable, and actually pleasant to work with.

## Exercises

1. **Protocol Playground**: Create a drawing application using protocols for shapes, colors, and rendering strategies.

2. **Extension Challenge**: Extend the Collection protocol to add statistical methods (mean, median, mode) that work with numeric elements.

3. **Type Eraser**: Build a type-erased wrapper for a protocol with associated types.

4. **Protocol Hierarchy**: Design a protocol hierarchy for a game with characters, abilities, and items.

Remember: In Swift, protocols aren't just interfaces—they're a way of thinking about code. Start with protocols, compose behaviors, and extend as needed. This is the Swift way.