# Chapter 5: Structs vs Classes - The Great Debate

In most object-oriented languages, classes are the stars of the show. Swift flips the script: structs are the default choice, and classes are for special occasions. This isn't rebellion for rebellion's sake—it's a fundamental rethinking of how we model data and behavior.

## The Fundamental Difference: Value vs Reference

Here's the million-dollar distinction:

```swift
// Struct: Value Type
struct Point {
    var x: Double
    var y: Double
}

var point1 = Point(x: 10, y: 20)
var point2 = point1  // Creates a COPY
point2.x = 30

print(point1.x)  // Still 10
print(point2.x)  // Now 30

// Class: Reference Type
class Person {
    var name: String
    init(name: String) {
        self.name = name
    }
}

let person1 = Person(name: "Alice")
let person2 = person1  // Both variables point to SAME object
person2.name = "Bob"

print(person1.name)  // "Bob" - Changed!
print(person2.name)  // "Bob" - Same object
```

This difference isn't academic—it fundamentally changes how you reason about your code.

## Structs: Swift's Workhorses

### Why Structs Are Special in Swift

```swift
struct User {
    var name: String
    var age: Int
    var email: String

    // Memberwise initializer comes free!
    // init(name: String, age: Int, email: String) is automatic

    // Methods
    func displayName() -> String {
        return "\(name) (\(age))"
    }

    // Mutating methods change the struct
    mutating func haveBirthday() {
        age += 1
    }
}

// Using the automatic initializer
var user = User(name: "Alice", age: 30, email: "alice@example.com")
user.haveBirthday()  // age is now 31
```

### Computed Properties and Property Observers

```swift
struct Rectangle {
    var width: Double
    var height: Double

    // Computed property
    var area: Double {
        return width * height
    }

    // Computed property with getter and setter
    var perimeter: Double {
        get {
            return 2 * (width + height)
        }
        set {
            let ratio = width / height
            height = sqrt(newValue / (2 * (1 + ratio)))
            width = ratio * height
        }
    }

    // Property observers
    var color: String = "white" {
        willSet {
            print("About to change color from \(color) to \(newValue)")
        }
        didSet {
            print("Color changed from \(oldValue) to \(color)")
        }
    }
}
```

### Copy-on-Write: Having Your Cake and Eating It Too

Swift's standard library types use copy-on-write (COW) optimization:

```swift
var array1 = [1, 2, 3, 4, 5]  // Allocates memory
var array2 = array1            // No copy yet, both share same storage
print(array1 === array2)       // Conceptually sharing

array2.append(6)               // NOW it copies (copy-on-write)
// array1 is still [1, 2, 3, 4, 5]
// array2 is [1, 2, 3, 4, 5, 6]
```

This gives you value semantics with reference-type performance when possible.

## Classes: When You Actually Need Them

### The Unique Features of Classes

```swift
class Vehicle {
    var currentSpeed = 0.0
    var description: String {
        return "traveling at \(currentSpeed) mph"
    }

    func makeNoise() {
        // Base implementation
    }
}

// 1. Inheritance
class Bicycle: Vehicle {
    var hasBasket = false
    override func makeNoise() {
        print("Ring ring!")
    }
}

// 2. Type casting
let vehicles: [Vehicle] = [Bicycle(), Vehicle()]
for vehicle in vehicles {
    if let bike = vehicle as? Bicycle {
        print("It's a bike with basket: \(bike.hasBasket)")
    }
}

// 3. Deinitializers
class FileHandler {
    let filename: String

    init(filename: String) {
        self.filename = filename
        print("Opening \(filename)")
    }

    deinit {
        print("Closing \(filename)")
    }
}

// 4. Reference counting (multiple references to same instance)
class Counter {
    var count = 0
    func increment() { count += 1 }
}

let counter = Counter()
let sameCounter = counter
sameCounter.increment()
print(counter.count)  // 1 - Both variables reference same object
```

### Identity Operators

With classes, you can check if two variables reference the same instance:

```swift
class SomeClass {
    var value: Int
    init(value: Int) {
        self.value = value
    }
}

let instance1 = SomeClass(value: 5)
let instance2 = SomeClass(value: 5)
let instance3 = instance1

// Identity (same instance?)
if instance1 === instance3 {
    print("Same instance")
}

if instance1 !== instance2 {
    print("Different instances, even with same value")
}

// Equality would need Equatable protocol implementation
```

## The Decision Tree: Struct or Class?

### Use a Struct When:

1. **You want value semantics**
   ```swift
   struct Coordinate {
       var latitude: Double
       var longitude: Double
   }
   // Each variable should have its own copy
   ```

2. **The data is simple and self-contained**
   ```swift
   struct Color {
       let red: Double
       let green: Double
       let blue: Double
       let alpha: Double
   }
   ```

3. **You don't need inheritance**
   ```swift
   struct HTTPRequest {
       let method: String
       let url: URL
       let headers: [String: String]
       let body: Data?
   }
   ```

4. **Thread safety is important**
   ```swift
   struct Configuration {
       let apiKey: String
       let timeout: TimeInterval
       let retryCount: Int
   }
   // Safe to pass between threads
   ```

### Use a Class When:

1. **You need reference semantics**
   ```swift
   class ViewContoller: UIViewController {
       // Multiple objects need to reference the same view controller
   }
   ```

2. **You need inheritance**
   ```swift
   class Animal {
       func makeSound() { }
   }

   class Dog: Animal {
       override func makeSound() {
           print("Woof!")
       }
   }
   ```

3. **You need deinitializers**
   ```swift
   class ResourceManager {
       deinit {
           // Clean up resources
           closeFileHandles()
           releaseMemory()
       }
   }
   ```

4. **Interoperability with Objective-C**
   ```swift
   @objc class MySwiftClass: NSObject {
       @objc func doSomething() {
           // Can be called from Objective-C
       }
   }
   ```

## Advanced Patterns and Techniques

### Hybrid Approaches: Best of Both Worlds

```swift
// Struct with class property for shared state
struct Document {
    let id: String
    let title: String
    private let sharedState: SharedState

    class SharedState {
        var viewCount: Int = 0
        var lastModified: Date = Date()
    }

    init(id: String, title: String) {
        self.id = id
        self.title = title
        self.sharedState = SharedState()
    }

    func incrementViewCount() {
        sharedState.viewCount += 1
    }
}
```

### Making Structs Act Like Classes (When Needed)

```swift
// Using a box to get reference semantics with structs
final class Box<T> {
    var value: T
    init(_ value: T) {
        self.value = value
    }
}

struct SharedStruct {
    private var box: Box<InternalData>

    struct InternalData {
        var counter: Int = 0
    }

    init() {
        self.box = Box(InternalData())
    }

    var counter: Int {
        get { box.value.counter }
        set { box.value.counter = newValue }
    }
}

var shared1 = SharedStruct()
var shared2 = shared1
shared2.counter = 10
// Both now have counter = 10 due to shared Box
```

### Protocol-Oriented Programming with Structs

```swift
protocol Drawable {
    func draw()
}

protocol Resizable {
    mutating func resize(by scale: Double)
}

struct Square: Drawable, Resizable {
    var size: Double

    func draw() {
        print("Drawing square of size \(size)")
    }

    mutating func resize(by scale: Double) {
        size *= scale
    }
}

struct Circle: Drawable, Resizable {
    var radius: Double

    func draw() {
        print("Drawing circle with radius \(radius)")
    }

    mutating func resize(by scale: Double) {
        radius *= scale
    }
}

// No inheritance needed!
let shapes: [Drawable & Resizable] = [
    Square(size: 10),
    Circle(radius: 5)
]
```

## Memory and Performance Implications

### Stack vs Heap

```swift
// Structs typically go on the stack (fast!)
struct StackBased {
    let x: Int
    let y: Int
}  // Deallocated automatically when out of scope

// Classes always go on the heap (slower, but flexible)
class HeapBased {
    let x: Int
    let y: Int

    init(x: Int, y: Int) {
        self.x = x
        self.y = y
    }
}  // Needs reference counting
```

### The Cost of Reference Counting

```swift
class Node {
    var value: Int
    var next: Node?  // Strong reference

    init(value: Int) {
        self.value = value
    }
}

// Every assignment adjusts reference counts
var node1: Node? = Node(value: 1)  // refCount = 1
var node2 = node1                   // refCount = 2
node1 = nil                         // refCount = 1
node2 = nil                         // refCount = 0, dealloc
```

## Common Pitfalls and How to Avoid Them

### The Unexpected Mutation

```swift
// Pitfall: Forgetting classes are reference types
class Settings {
    var isDarkMode = false
}

struct App {
    let settings = Settings()
}

let app1 = App()
let app2 = app1
app2.settings.isDarkMode = true
// Surprise! app1.settings.isDarkMode is also true

// Solution: Use a struct or create copies
struct BetterSettings {
    var isDarkMode = false
}
```

### The Massive Struct Problem

```swift
// Bad: Huge struct gets copied frequently
struct HugeStruct {
    let array1: [Int] = Array(repeating: 0, count: 10000)
    let array2: [Int] = Array(repeating: 0, count: 10000)
    // ... many more properties
}

// Better: Use class for large data or implement COW
class LargeData {
    let array1: [Int]
    let array2: [Int]
    // Reference counting is cheaper than copying
}
```

### The Retain Cycle Trap

```swift
class Parent {
    var child: Child?
}

class Child {
    var parent: Parent?  // Strong reference cycle!
}

// Solution: Use weak references
class BetterChild {
    weak var parent: Parent?  // Breaks the cycle
}
```

## Real-World Guidelines

### Apple's Recommendations

Apple suggests starting with structs and enums, using classes only when:
- You need Objective-C interoperability
- You need identity (===) checks
- You need inheritance

### SwiftUI's Approach

SwiftUI heavily favors structs:

```swift
struct ContentView: View {  // View is a protocol!
    @State private var counter = 0  // Property wrapper handles mutability

    var body: some View {
        Button("Count: \(counter)") {
            counter += 1
        }
    }
}
// Entire UI built with structs!
```

## What's Next?

You now understand the fundamental difference between structs and classes, and more importantly, when to use each. You've seen how value semantics lead to safer, more predictable code, and when reference semantics are actually necessary.

Next up: Protocols and Extensions—Swift's secret weapons for building flexible, reusable code without the baggage of traditional inheritance. You'll discover why "protocol-oriented programming" isn't just a buzzword, but a paradigm shift.

## Exercises

1. **Copy-on-Write Implementation**: Implement your own struct with copy-on-write behavior for efficient value semantics.

2. **Struct vs Class Performance**: Write benchmarks comparing struct and class performance for various operations.

3. **Reference Cycle Detective**: Create a tool that detects potential retain cycles in a given class hierarchy.

4. **Immutable Builder**: Design a builder pattern using structs that maintains immutability throughout the build process.

Remember: In Swift, prefer structs and protocols. Use classes when you need reference semantics, not because that's what you're used to. This mental shift will make your code safer, clearer, and more Swift-like.