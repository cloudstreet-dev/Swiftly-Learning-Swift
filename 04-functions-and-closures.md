# Chapter 4: Functions, Closures, and the Art of Being Lazy

Functions in Swift are first-class citizens. They can be passed around, stored in variables, returned from other functions, and created on the fly. If JavaScript made you appreciate functions as values, Swift will make you fall in love with them.

## Functions: More Than Just Procedures

Let's start with the basics and quickly escalate to the mind-bending stuff.

### The Fundamentals

```swift
// Basic function
func greet(name: String) -> String {
    return "Hello, \(name)!"
}

// Multiple parameters
func add(a: Int, b: Int) -> Int {
    return a + b
}

// No return value (actually returns Void)
func printMessage(_ message: String) {
    print(message)
}

// Multiple return values with tuples
func minMax(array: [Int]) -> (min: Int, max: Int)? {
    guard !array.isEmpty else { return nil }
    return (array.min()!, array.max()!)
}
```

### Parameter Labels: Swift's Linguistic Innovation

Swift functions read like English sentences. This isn't an accident:

```swift
// External and internal parameter names
func move(from start: Point, to end: Point) {
    // Internally use 'start' and 'end'
    let distance = calculate(from: start, to: end)
}

// Called like English
move(from: pointA, to: pointB)

// Omitting external labels with _
func addNumbers(_ a: Int, _ b: Int) -> Int {
    return a + b
}

addNumbers(5, 3)  // No labels needed

// Readable and self-documenting
func resize(image: UIImage, to size: CGSize, maintaining aspectRatio: Bool) -> UIImage {
    // Implementation
}

resize(image: photo, to: thumbnailSize, maintaining: true)
```

This isn't just syntactic sugar—it makes APIs self-documenting and delightful to use.

### Default Parameters and Variadic Arguments

```swift
// Default parameters
func greet(name: String, greeting: String = "Hello") -> String {
    return "\(greeting), \(name)!"
}

greet(name: "Alice")                     // "Hello, Alice!"
greet(name: "Bob", greeting: "Howdy")    // "Howdy, Bob!"

// Variadic parameters (accepts zero or more values)
func sum(_ numbers: Int...) -> Int {
    return numbers.reduce(0, +)
}

sum(1, 2, 3, 4, 5)  // 15
sum()               // 0

// Combining features
func configure(
    host: String = "localhost",
    port: Int = 8080,
    useSSL: Bool = false,
    endpoints: String...
) {
    print("Configuring \(host):\(port) (SSL: \(useSSL))")
    endpoints.forEach { print("  - \($0)") }
}

configure(endpoints: "/api", "/health", "/metrics")
```

### In-Out Parameters: When You Need to Mutate

```swift
// Modifying parameters directly
func swapValues(_ a: inout Int, _ b: inout Int) {
    let temp = a
    a = b
    b = temp
}

var x = 10, y = 20
swapValues(&x, &y)  // x is now 20, y is now 10

// More practical example
func append(value: Int, to array: inout [Int]) {
    array.append(value)
}

var numbers = [1, 2, 3]
append(value: 4, to: &numbers)  // numbers is now [1, 2, 3, 4]
```

The `&` makes it crystal clear you're passing by reference—no hidden surprises.

## Functions as Types: The Gateway Drug to Functional Programming

```swift
// Function types
let mathOperation: (Int, Int) -> Int

// Assigning functions to variables
mathOperation = add
let result = mathOperation(5, 3)  // 8

mathOperation = { $0 * $1 }  // Now it multiplies
let product = mathOperation(5, 3)  // 15

// Functions as parameters
func applyOperation(
    _ a: Int,
    _ b: Int,
    operation: (Int, Int) -> Int
) -> Int {
    return operation(a, b)
}

let sum = applyOperation(10, 5, operation: +)  // Yes, + is a function!
let difference = applyOperation(10, 5, operation: -)

// Functions returning functions
func makeMultiplier(factor: Int) -> (Int) -> Int {
    return { number in
        return number * factor
    }
}

let triple = makeMultiplier(factor: 3)
let result = triple(5)  // 15
```

## Closures: Functions' Cooler Younger Sibling

Closures are self-contained blocks of code that can capture values from their context. Think of them as anonymous functions with superpowers.

### The Evolution of Closure Syntax

```swift
let names = ["Charlie", "Alice", "Bob", "David"]

// Full closure syntax
let sorted1 = names.sorted(by: { (s1: String, s2: String) -> Bool in
    return s1 < s2
})

// Type inference
let sorted2 = names.sorted(by: { s1, s2 in
    return s1 < s2
})

// Single expression closures (implicit return)
let sorted3 = names.sorted(by: { s1, s2 in s1 < s2 })

// Shorthand argument names
let sorted4 = names.sorted(by: { $0 < $1 })

// Operator method
let sorted5 = names.sorted(by: <)

// All produce: ["Alice", "Bob", "Charlie", "David"]
```

This progression from verbose to concise isn't showing off—it's about using the right level of clarity for the context.

### Trailing Closure Syntax: Swift's Secret Sauce

When a closure is the last parameter, Swift lets you write it outside the parentheses:

```swift
// Without trailing closure
let doubled = numbers.map({ $0 * 2 })

// With trailing closure
let doubled = numbers.map { $0 * 2 }

// Multiple trailing closures (Swift 5.3+)
func loadData(
    onSuccess: (Data) -> Void,
    onFailure: (Error) -> Void
) {
    // Implementation
}

loadData { data in
    print("Got data: \(data)")
} onFailure: { error in
    print("Failed: \(error)")
}

// Makes APIs read beautifully
UIView.animate(withDuration: 0.3) {
    view.alpha = 0
} completion: { _ in
    view.removeFromSuperview()
}
```

### Capturing Values: Closures' Superpower

```swift
func makeIncrementer(incrementAmount: Int) -> () -> Int {
    var total = 0

    let incrementer: () -> Int = {
        total += incrementAmount  // Captures both variables
        return total
    }

    return incrementer
}

let incrementByTen = makeIncrementer(incrementAmount: 10)
print(incrementByTen())  // 10
print(incrementByTen())  // 20
print(incrementByTen())  // 30

let incrementByFive = makeIncrementer(incrementAmount: 5)
print(incrementByFive())  // 5 (separate instance!)
```

Closures capture and store references to constants and variables from their context. This is called "closing over" those values.

### Escaping Closures: When Closures Outlive Functions

```swift
var completionHandlers: [() -> Void] = []

// @escaping means the closure can outlive the function
func addCompletionHandler(handler: @escaping () -> Void) {
    completionHandlers.append(handler)
}

// Non-escaping by default
func performNow(action: () -> Void) {
    action()  // Must be called before function returns
}

// Real-world example
class AsyncManager {
    private var callbacks: [(Result<Data, Error>) -> Void] = []

    func fetchData(completion: @escaping (Result<Data, Error>) -> Void) {
        callbacks.append(completion)  // Stores for later

        // Start async operation
        URLSession.shared.dataTask(with: url) { data, _, error in
            let result = // ... create result
            self.callbacks.forEach { $0(result) }
        }.resume()
    }
}
```

### Autoclosures: Lazy Evaluation Made Easy

```swift
// Without autoclosure
func logIfTrue(_ condition: Bool, message: () -> String) {
    if condition {
        print(message())
    }
}

logIfTrue(2 > 1, message: { "This is true" })

// With autoclosure
func logIfTrue(_ condition: Bool, message: @autoclosure () -> String) {
    if condition {
        print(message())
    }
}

logIfTrue(2 > 1, message: "This is true")  // Cleaner!

// Real use case: assert
assert(expensive() > 0, "Value must be positive")
// expensive() only called if assertions are enabled
```

## Higher-Order Functions: Functional Programming Patterns

Swift's standard library is full of higher-order functions that make data transformation elegant:

### Map, Filter, Reduce: The Holy Trinity

```swift
let numbers = [1, 2, 3, 4, 5]

// Map: Transform each element
let doubled = numbers.map { $0 * 2 }  // [2, 4, 6, 8, 10]

// Filter: Keep elements matching a condition
let evens = numbers.filter { $0 % 2 == 0 }  // [2, 4]

// Reduce: Combine all elements into a single value
let sum = numbers.reduce(0, +)  // 15
let product = numbers.reduce(1, *)  // 120

// Chaining operations
let result = numbers
    .filter { $0 % 2 == 0 }
    .map { $0 * $0 }
    .reduce(0, +)  // 20 (4 + 16)

// More complex reduce
let words = ["Swift", "is", "awesome"]
let sentence = words.reduce("") { result, word in
    result.isEmpty ? word : "\(result) \(word)"
}  // "Swift is awesome"
```

### CompactMap and FlatMap: The Specialists

```swift
// CompactMap: Map + remove nils
let strings = ["1", "2", "abc", "4"]
let numbers = strings.compactMap { Int($0) }  // [1, 2, 4]

// FlatMap: Flatten nested collections
let nested = [[1, 2], [3, 4], [5, 6]]
let flat = nested.flatMap { $0 }  // [1, 2, 3, 4, 5, 6]

// Combining transformations
let matrix = [[1, 2, 3], [4, 5, 6], [7, 8, 9]]
let evenSquares = matrix
    .flatMap { $0 }
    .filter { $0 % 2 == 0 }
    .map { $0 * $0 }  // [4, 16, 36, 64]
```

### Custom Higher-Order Functions

```swift
// Creating your own higher-order functions
extension Array {
    func customMap<T>(_ transform: (Element) -> T) -> [T] {
        var result = [T]()
        for element in self {
            result.append(transform(element))
        }
        return result
    }

    func partition(by predicate: (Element) -> Bool) -> (matching: [Element], notMatching: [Element]) {
        var matching = [Element]()
        var notMatching = [Element]()

        for element in self {
            if predicate(element) {
                matching.append(element)
            } else {
                notMatching.append(element)
            }
        }

        return (matching, notMatching)
    }
}

let numbers = [1, 2, 3, 4, 5]
let (evens, odds) = numbers.partition { $0 % 2 == 0 }
// evens: [2, 4], odds: [1, 3, 5]
```

## Lazy Evaluation: Procrastination as a Feature

```swift
// Lazy sequences defer computation
let hugeArray = Array(1...1_000_000)

// This doesn't compute anything yet
let lazyResult = hugeArray.lazy
    .filter { $0 % 2 == 0 }
    .map { $0 * $0 }
    .prefix(10)

// Computation happens here, only for first 10 elements!
let actualResult = Array(lazyResult)

// Infinite sequences
func fibonacci() -> UnfoldSequence<Int, (Int, Int)> {
    return sequence(state: (0, 1)) { state in
        let current = state.0
        state = (state.1, state.0 + state.1)
        return current
    }
}

let first10Fibs = Array(fibonacci().prefix(10))
// [0, 1, 1, 2, 3, 5, 8, 13, 21, 34]
```

## Function Builders (Result Builders): DSL Magic

```swift
@resultBuilder
struct StringBuilder {
    static func buildBlock(_ parts: String...) -> String {
        parts.joined(separator: " ")
    }

    static func buildIf(_ part: String?) -> String {
        part ?? ""
    }
}

@StringBuilder
func buildMessage(name: String, includeGreeting: Bool) -> String {
    if includeGreeting {
        "Hello"
    }
    name
    "Welcome to Swift!"
}

let message = buildMessage(name: "Alice", includeGreeting: true)
// "Hello Alice Welcome to Swift!"
```

This is the technology behind SwiftUI's declarative syntax!

## Practical Patterns and Best Practices

### The Completion Handler Pattern
```swift
func fetchUser(id: Int, completion: @escaping (Result<User, Error>) -> Void) {
    DispatchQueue.global().async {
        // Simulate network call
        Thread.sleep(forTimeInterval: 1)

        if id > 0 {
            let user = User(id: id, name: "User \(id)")
            completion(.success(user))
        } else {
            completion(.failure(UserError.invalidId))
        }
    }
}

fetchUser(id: 42) { result in
    switch result {
    case .success(let user):
        print("Got user: \(user.name)")
    case .failure(let error):
        print("Failed: \(error)")
    }
}
```

### The Builder Pattern with Closures
```swift
class ViewBuilder {
    private var frame = CGRect.zero
    private var backgroundColor = UIColor.white

    func withFrame(_ frame: CGRect) -> Self {
        self.frame = frame
        return self
    }

    func withBackgroundColor(_ color: UIColor) -> Self {
        self.backgroundColor = color
        return self
    }

    func build() -> UIView {
        let view = UIView(frame: frame)
        view.backgroundColor = backgroundColor
        return view
    }
}

let customView = ViewBuilder()
    .withFrame(CGRect(x: 0, y: 0, width: 100, height: 100))
    .withBackgroundColor(.blue)
    .build()
```

## What's Next?

You've mastered functions and closures—the building blocks of Swift's expressive power. You've seen how functions are values, how closures capture context, and how higher-order functions transform data elegantly.

Next, we dive into the great debate: structs vs. classes. You'll learn why Swift developers reach for structs first, when classes are actually necessary, and how value semantics change everything about how you think about data.

## Exercises

1. **Curry Chef**: Write a curry function that transforms a function taking multiple parameters into a series of functions taking single parameters.

2. **Memoization Master**: Create a memoize function that caches results of expensive computations.

3. **Pipeline Builder**: Implement a pipe operator that chains functions together: `value |> function1 |> function2 |> function3`

4. **Async Sequencer**: Write a function that executes an array of async operations in sequence, not parallel.

Remember: In Swift, functions aren't just things you call—they're values you compose, transform, and pass around. Master this, and you'll write Swift that sings.