# Chapter 14: Advanced Swift - Macros and Beyond

Swift continues to evolve, adding powerful features that push the boundaries of what's possible. From compile-time code generation with macros to runtime flexibility with dynamic member lookup, these advanced features open new possibilities while maintaining Swift's commitment to safety and performance. Let's explore the cutting edge of Swift.

## Swift Macros: Code That Writes Code

Introduced in Swift 5.9, macros enable compile-time code generation, eliminating boilerplate while maintaining type safety and debuggability.

### Understanding Macros

```swift
// Using a macro (the @ prefix)
@AddCompletionHandler
func fetchData() async throws -> Data {
    // Implementation
}

// The macro generates this additional method:
// func fetchData(completion: @escaping (Result<Data, Error>) -> Void) {
//     Task {
//         do {
//             let result = try await fetchData()
//             completion(.success(result))
//         } catch {
//             completion(.failure(error))
//         }
//     }
// }
```

### Types of Macros

#### Freestanding Macros

```swift
// Expression macros - produce a value
let url = #URL("https://example.com")  // Compile-time URL validation

// Declaration macros - create new declarations
#warning("This API is deprecated")

// Code item macros - generate multiple declarations
#generateMockData(for: User.self, count: 10)
```

#### Attached Macros

```swift
// Member macros - add members to types
@AddInit
struct Point {
    let x: Double
    let y: Double
}
// Generates: init(x: Double, y: Double)

// Peer macros - add alongside declarations
@AddAsync
func fetchUser(id: String, completion: @escaping (User?) -> Void) {
    // Implementation
}
// Generates async version alongside

// Accessor macros - add computed property behavior
@Clamped(min: 0, max: 100)
var volume: Int = 50
// Generates getters/setters with clamping logic

// Extension macros - add protocol conformances
@Codable
struct User {
    let id: UUID
    let name: String
}
// Adds Codable conformance with synthesis
```

### Creating Custom Macros

```swift
// Define the macro interface (in your main module)
@freestanding(expression)
public macro stringify<T>(_ value: T) -> (T, String) = #externalMacro(
    module: "MyMacros",
    type: "StringifyMacro"
)

// Implement the macro (in a separate macro module)
import SwiftSyntax
import SwiftSyntaxBuilder
import SwiftSyntaxMacros

public struct StringifyMacro: ExpressionMacro {
    public static func expansion(
        of node: some FreestandingMacroExpansionSyntax,
        in context: some MacroExpansionContext
    ) -> ExprSyntax {
        guard let argument = node.argumentList.first?.expression else {
            fatalError("Missing argument")
        }

        return "(\(argument), \(literal: argument.description))"
    }
}

// Usage
let result = #stringify(2 + 2)
print(result)  // (4, "2 + 2")
```

### Real-World Macro Examples

#### Observable Macro

```swift
import Observation

@Observable
class ViewModel {
    var count = 0
    var text = ""
    private var internalState = 0  // Not observed

    func increment() {
        count += 1
    }
}

// The @Observable macro generates:
// - Property observers for each stored property
// - Notification mechanisms for SwiftUI
// - Dependency tracking for computed properties
```

#### CaseDetection Macro

```swift
@CaseDetection
enum State {
    case idle
    case loading
    case loaded(Data)
    case error(Error)
}

// Generates:
// var isIdle: Bool { if case .idle = self { true } else { false } }
// var isLoading: Bool { if case .loading = self { true } else { false } }
// var isLoaded: Bool { if case .loaded = self { true } else { false } }
// var isError: Bool { if case .error = self { true } else { false } }

// Usage
if state.isLoading {
    showProgressIndicator()
}
```

## Dynamic Member Lookup

Dynamic member lookup allows types to provide dot syntax for arbitrary properties:

```swift
@dynamicMemberLookup
struct DynamicDictionary {
    private var storage: [String: Any] = [:]

    subscript(dynamicMember key: String) -> Any? {
        get { storage[key] }
        set { storage[key] = newValue }
    }
}

var dict = DynamicDictionary()
dict.name = "Swift"  // Works even though 'name' isn't defined
dict.version = 5.9
print(dict.name)  // Optional("Swift")
```

### Typed Dynamic Members

```swift
@dynamicMemberLookup
struct TypedJSON {
    private let data: [String: Any]

    init(_ data: [String: Any]) {
        self.data = data
    }

    subscript<T>(dynamicMember key: String) -> T? {
        return data[key] as? T
    }

    subscript(dynamicMember key: String) -> TypedJSON? {
        guard let dict = data[key] as? [String: Any] else { return nil }
        return TypedJSON(dict)
    }
}

let json = TypedJSON([
    "user": [
        "name": "Alice",
        "age": 30
    ]
])

let name: String? = json.user?.name
let age: Int? = json.user?.age
```

### KeyPath Dynamic Member Lookup

```swift
@dynamicMemberLookup
struct Wrapper<T> {
    let wrapped: T

    subscript<U>(dynamicMember keyPath: KeyPath<T, U>) -> U {
        return wrapped[keyPath: keyPath]
    }

    subscript<U>(dynamicMember keyPath: WritableKeyPath<T, U>) -> U {
        get { wrapped[keyPath: keyPath] }
        set { /* Would need mutable wrapped */ }
    }
}

struct Person {
    let name: String
    let age: Int
}

let wrappedPerson = Wrapper(wrapped: Person(name: "Bob", age: 25))
print(wrappedPerson.name)  // "Bob" - accessing through wrapper
print(wrappedPerson.age)   // 25
```

## Callable Types

Make your types callable like functions:

```swift
@dynamicCallable
struct Multiplier {
    let factor: Int

    func dynamicallyCall(withArguments args: [Int]) -> Int {
        return args.reduce(0, +) * factor
    }

    func dynamicallyCall(withKeywordArguments args: KeyValuePairs<String, Int>) -> Int {
        return args.reduce(0) { $0 + $1.value } * factor
    }
}

let triple = Multiplier(factor: 3)
let result = triple(5, 10)  // 45 (15 * 3)

// With keywords
let result2 = triple(first: 5, second: 10)  // 45
```

### Combining Dynamic Features

```swift
@dynamicMemberLookup
@dynamicCallable
struct FlexibleAPI {
    private var properties: [String: Any] = [:]
    private var methods: [String: ([Any]) -> Any] = [:]

    subscript(dynamicMember key: String) -> Any? {
        get { properties[key] }
        set { properties[key] = newValue }
    }

    func dynamicallyCall(withArguments args: [Any]) -> Any? {
        guard let methodName = args.first as? String,
              let method = methods[methodName] else { return nil }

        return method(Array(args.dropFirst()))
    }

    mutating func registerMethod(_ name: String, _ implementation: @escaping ([Any]) -> Any) {
        methods[name] = implementation
    }
}

var api = FlexibleAPI()
api.version = "1.0"
api.registerMethod("greet") { args in
    "Hello, \(args.first ?? "World")!"
}

print(api.version)  // Optional("1.0")
let greeting = api("greet", "Swift")  // Optional("Hello, Swift!")
```

## Custom String Interpolation

Create domain-specific string interpolation:

```swift
struct HTMLString: ExpressibleByStringInterpolation {
    let html: String

    init(stringLiteral value: String) {
        self.html = value
    }

    init(stringInterpolation: StringInterpolation) {
        self.html = stringInterpolation.output
    }

    struct StringInterpolation: StringInterpolationProtocol {
        var output = ""

        init(literalCapacity: Int, interpolationCount: Int) {
            output.reserveCapacity(literalCapacity * 2)
        }

        mutating func appendLiteral(_ literal: String) {
            output.append(literal)
        }

        mutating func appendInterpolation(_ value: String) {
            output.append(value.htmlEscaped())
        }

        mutating func appendInterpolation(bold value: String) {
            output.append("<b>\(value.htmlEscaped())</b>")
        }

        mutating func appendInterpolation(link url: String, text: String) {
            output.append(#"<a href="\#(url.htmlEscaped())">\#(text.htmlEscaped())</a>"#)
        }

        mutating func appendInterpolation(code value: String) {
            output.append("<code>\(value.htmlEscaped())</code>")
        }
    }
}

extension String {
    func htmlEscaped() -> String {
        return self
            .replacingOccurrences(of: "&", with: "&amp;")
            .replacingOccurrences(of: "<", with: "&lt;")
            .replacingOccurrences(of: ">", with: "&gt;")
            .replacingOccurrences(of: "\"", with: "&quot;")
    }
}

// Usage
let userInput = "<script>alert('XSS')</script>"
let html: HTMLString = """
    <h1>\(bold: "Welcome")</h1>
    <p>Check out \(link: "https://swift.org", text: "Swift's website")</p>
    <p>User said: \(userInput)</p>
    <p>Run \(code: "swift build") to compile</p>
    """
print(html.html)
// Output: Safe HTML with escaped user input
```

## Result Builders (Function Builders)

The technology behind SwiftUI's declarative syntax:

```swift
@resultBuilder
struct ArrayBuilder<Element> {
    static func buildBlock(_ components: Element...) -> [Element] {
        components
    }

    static func buildOptional(_ component: [Element]?) -> [Element] {
        component ?? []
    }

    static func buildEither(first component: [Element]) -> [Element] {
        component
    }

    static func buildEither(second component: [Element]) -> [Element] {
        component
    }

    static func buildArray(_ components: [[Element]]) -> [Element] {
        components.flatMap { $0 }
    }

    static func buildExpression(_ expression: Element) -> [Element] {
        [expression]
    }
}

func buildArray<T>(@ArrayBuilder<T> _ content: () -> [T]) -> [T] {
    content()
}

// Usage
let numbers = buildArray {
    1
    2
    3
    if Bool.random() {
        4
        5
    } else {
        6
    }
    for i in 7...10 {
        i
    }
}
// Result: [1, 2, 3, 4, 5, 7, 8, 9, 10] or [1, 2, 3, 6, 7, 8, 9, 10]
```

### Creating a DSL with Result Builders

```swift
@resultBuilder
struct HTMLBuilder {
    static func buildBlock(_ components: HTMLElement...) -> HTMLElement {
        HTMLGroup(children: components)
    }

    static func buildOptional(_ component: HTMLElement?) -> HTMLElement {
        component ?? HTMLEmpty()
    }

    static func buildEither(first component: HTMLElement) -> HTMLElement {
        component
    }

    static func buildEither(second component: HTMLElement) -> HTMLElement {
        component
    }
}

protocol HTMLElement {
    func render() -> String
}

struct HTMLGroup: HTMLElement {
    let children: [HTMLElement]

    func render() -> String {
        children.map { $0.render() }.joined()
    }
}

struct HTMLEmpty: HTMLElement {
    func render() -> String { "" }
}

struct Div: HTMLElement {
    let content: HTMLElement

    init(@HTMLBuilder content: () -> HTMLElement) {
        self.content = content()
    }

    func render() -> String {
        "<div>\(content.render())</div>"
    }
}

struct Paragraph: HTMLElement {
    let text: String

    func render() -> String {
        "<p>\(text)</p>"
    }
}

// Usage
let showWarning = true
let html = Div {
    Paragraph(text: "Welcome")
    if showWarning {
        Paragraph(text: "Warning: Beta version")
    }
    Paragraph(text: "Enjoy!")
}
print(html.render())
// <div><p>Welcome</p><p>Warning: Beta version</p><p>Enjoy!</p></div>
```

## Existential Types and any/some Keywords

Understanding Swift's type system evolution:

```swift
// 'any' for existential types (type-erased)
let shapes: [any Shape] = [Circle(), Square(), Triangle()]
// Can hold different concrete types but with performance cost

// 'some' for opaque types (concrete but hidden)
func makeShape() -> some Shape {
    Circle()  // Always returns same concrete type
}

// Primary associated types
protocol Collection<Element> {
    associatedtype Element
}

let numbers: any Collection<Int> = [1, 2, 3]  // Constrained existential

// Existential 'any' with protocols
protocol Drawable {
    func draw()
}

func render(drawable: any Drawable) {  // Explicit existential
    drawable.draw()
}

// Opaque types preserve type information
func createStack<T>() -> some Collection<T> {
    return [T]()  // Concrete type hidden but preserved
}
```

## Variadic Generics (Future Swift)

A glimpse into upcoming Swift features:

```swift
// Proposed syntax for variadic generics
func zip<each T>(_ values: repeat each T) -> (repeat each T) {
    return (repeat each values)
}

let result = zip(1, "hello", true)  // (1, "hello", true)

// Function taking variable number of different types
func printAll<each T: CustomStringConvertible>(_ items: repeat each T) {
    repeat print(each items)
}

printAll(42, "Swift", 3.14, true)
```

## Custom Operators

Define your own operators:

```swift
// Define precedence group
precedencegroup CompositionPrecedence {
    associativity: left
    higherThan: AssignmentPrecedence
}

// Define operator
infix operator •: CompositionPrecedence

// Implement for function composition
func • <A, B, C>(f: @escaping (B) -> C, g: @escaping (A) -> B) -> (A) -> C {
    return { f(g($0)) }
}

// Usage
let addOne = { $0 + 1 }
let double = { $0 * 2 }
let addOneThenDouble = double • addOne

print(addOneThenDouble(5))  // 12 ((5 + 1) * 2)

// Custom operator for optional chaining with default
infix operator ??=: AssignmentPrecedence

func ??= <T>(left: inout T?, right: @autoclosure () -> T) {
    left = left ?? right()
}

var name: String?
name ??= "Default"  // Sets to "Default" if nil
```

## Unsafe Swift

When you need to bypass safety for performance:

```swift
// Unsafe pointers
func unsafeOperation() {
    let array = [1, 2, 3, 4, 5]

    array.withUnsafeBufferPointer { buffer in
        // Direct memory access
        for i in 0..<buffer.count {
            print(buffer[i])
        }
    }

    // Unsafe mutable access
    var mutableArray = [1, 2, 3]
    mutableArray.withUnsafeMutableBufferPointer { buffer in
        buffer[0] = 100
    }
}

// Unsafe bitcast
let intValue = 42
let floatBits = unsafeBitCast(intValue, to: Float.self)

// Manual memory management
let count = 10
let pointer = UnsafeMutablePointer<Int>.allocate(capacity: count)
defer { pointer.deallocate() }

pointer.initialize(repeating: 0, count: count)
for i in 0..<count {
    pointer[i] = i * i
}
```

## Reflection with Mirror

Runtime introspection:

```swift
struct Person {
    let name: String
    let age: Int
    private let secret = "Hidden"
}

let person = Person(name: "Alice", age: 30)
let mirror = Mirror(reflecting: person)

print("Type: \(mirror.subjectType)")
print("Properties:")
for child in mirror.children {
    print("  \(child.label ?? "unknown"): \(child.value)")
}
// Output:
// Type: Person
// Properties:
//   name: Alice
//   age: 30
//   secret: Hidden

// Generic serialization using Mirror
func serialize<T>(object: T) -> [String: Any] {
    let mirror = Mirror(reflecting: object)
    var dict: [String: Any] = [:]

    for child in mirror.children {
        if let label = child.label {
            dict[label] = child.value
        }
    }

    return dict
}
```

## Performance Optimization Attributes

```swift
// Force inlining
@inline(__always)
func criticalPath() -> Int {
    return 42
}

// Prevent inlining
@inline(never)
func debugFunction() {
    print("Debug")
}

// Optimize for size
@_optimize(size)
func rarelyUsedFunction() {
    // Complex implementation
}

// Specialize generic functions
@_specialize(where T == Int)
@_specialize(where T == String)
func genericFunction<T>(_ value: T) {
    // Implementation
}

// Semantic attributes
@_semantics("array.append_element")
func customAppend<T>(_ element: T, to array: inout [T]) {
    array.append(element)
}

// Fixed layout for library evolution
@frozen
public struct Point {
    public var x: Double
    public var y: Double
}
```

## The Future of Swift

Swift continues to evolve with proposals for:

- **Ownership and Borrowing**: Fine-grained memory control
- **Distributed Actors**: Actor model across networks
- **Regex Literals**: First-class regular expression support
- **Package Plugins**: Build tool extensions
- **Embedded Swift**: Swift for resource-constrained devices

## Your Advanced Swift Journey

You've explored the cutting edge of Swift—from macros that generate code at compile time to dynamic features that add flexibility at runtime. These advanced features aren't everyday tools, but when you need them, they're incredibly powerful.

## Final Project

Build a macro-powered, type-safe SQL query builder:

```swift
@SQLTable
struct User {
    @PrimaryKey let id: UUID
    @Column let name: String
    @Column let email: String
    @Column(indexed: true) let createdAt: Date
}

// Macro generates:
let query = User.select()
    .where(\.name == "Alice")
    .orderBy(\.createdAt, .descending)
    .limit(10)

// Type-safe, compile-time checked SQL generation
```

## Resources

- **Swift Evolution**: Follow proposals at swift.org/evolution
- **Swift Forums**: Discuss advanced topics at forums.swift.org
- **Swift Package Index**: Find packages at swiftpackageindex.com
- **Macro Examples**: github.com/apple/swift-syntax

## The End of the Beginning

```swift
extension Developer {
    func masterSwift() -> Never {
        while true {
            learn()
            build()
            share()
            // Swift keeps evolving, and so do you
        }
    }
}

let you = Developer()
// you.masterSwift()  // The journey never ends
print("Ready to build amazing things with Swift! 🚀")
```

Congratulations! You've completed your journey through Swift, from basics to the bleeding edge. You're now equipped to build anything from simple scripts to complex applications, using the full power of one of the most modern programming languages available.

Remember: The best Swift code isn't the cleverest—it's the clearest. Use these advanced features when they make code better, not just because you can.

Now go forth and build something incredible. The Swift community is waiting to see what you create!