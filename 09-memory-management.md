# Chapter 9: Memory Management That Won't Keep You Up at Night

Coming from garbage-collected languages? Welcome to deterministic memory management. Coming from manual memory management? Put down that `free()` and relax. Swift's Automatic Reference Counting (ARC) gives you the control of manual memory management with (almost) the ease of garbage collection. No more stop-the-world pauses, no more forgetting to free memory—just predictable, efficient memory management.

## ARC: Your Silent Memory Butler

Automatic Reference Counting tracks how many references point to each class instance. When that count hits zero, the memory is immediately freed. No garbage collector scanning your heap, no unpredictable pauses—just deterministic deallocation.

```swift
class Person {
    let name: String

    init(name: String) {
        self.name = name
        print("\(name) is being initialized")
    }

    deinit {
        print("\(name) is being deinitialized")
    }
}

var reference1: Person?
var reference2: Person?
var reference3: Person?

reference1 = Person(name: "Alice")  // Reference count: 1
reference2 = reference1              // Reference count: 2
reference3 = reference1              // Reference count: 3

reference1 = nil  // Reference count: 2
reference2 = nil  // Reference count: 1
reference3 = nil  // Reference count: 0 - Alice is deinitialized
```

## Strong, Weak, and Unowned: The Reference Trilogy

### Strong References (The Default)

```swift
class Department {
    let name: String
    var employees: [Employee] = []

    init(name: String) {
        self.name = name
    }
}

class Employee {
    let name: String
    var department: Department  // Strong reference

    init(name: String, department: Department) {
        self.name = name
        self.department = department
    }
}

let accounting = Department(name: "Accounting")
let alice = Employee(name: "Alice", department: accounting)
accounting.employees.append(alice)
// Strong reference cycle! Neither can be deallocated
```

### Weak References: The Polite Option

```swift
class Employee {
    let name: String
    weak var department: Department?  // Weak reference - doesn't increase ref count

    init(name: String, department: Department?) {
        self.name = name
        self.department = department
    }
}

// Now when Department is deallocated, employee.department becomes nil
var dept: Department? = Department(name: "Engineering")
let bob = Employee(name: "Bob", department: dept)
dept = nil  // Department deallocated, bob.department is now nil
```

### Unowned References: The Confident Choice

```swift
class Customer {
    let name: String
    var creditCard: CreditCard?

    init(name: String) {
        self.name = name
    }
}

class CreditCard {
    let number: String
    unowned let customer: Customer  // Always expects customer to exist

    init(number: String, customer: Customer) {
        self.number = number
        self.customer = customer
    }
}

var john: Customer? = Customer(name: "John")
john?.creditCard = CreditCard(number: "1234-5678", customer: john!)
john = nil  // Both are deallocated properly
```

Use `unowned` when you know the reference will always be valid during the object's lifetime. It's like `weak` but without the optional—use with confidence, but be careful!

## Reference Cycles: The Memory Leak Makers

### The Classic Retain Cycle

```swift
// BAD: Creates a retain cycle
class Parent {
    var child: Child?

    deinit {
        print("Parent deallocated")
    }
}

class Child {
    var parent: Parent?  // Strong reference creates cycle

    deinit {
        print("Child deallocated")
    }
}

var parent: Parent? = Parent()
var child: Child? = Child()
parent?.child = child
child?.parent = parent

parent = nil
child = nil
// Neither deinit is called - memory leak!
```

### Breaking the Cycle

```swift
// GOOD: Using weak reference
class Child {
    weak var parent: Parent?  // Weak breaks the cycle

    deinit {
        print("Child deallocated")
    }
}

// Now both objects deallocate properly
```

## Closures and Capture Lists: The Subtle Trap

Closures capture values from their surrounding context, which can create retain cycles:

```swift
class ViewController {
    var buttonHandler: (() -> Void)?

    func setupButton() {
        // BAD: Creates retain cycle
        buttonHandler = {
            self.doSomething()  // Captures self strongly
        }
    }

    func doSomething() {
        print("Doing something")
    }

    deinit {
        print("ViewController deallocated")
    }
}
```

### Capture Lists to the Rescue

```swift
class ViewController {
    var buttonHandler: (() -> Void)?

    func setupButton() {
        // GOOD: Weak capture prevents cycle
        buttonHandler = { [weak self] in
            self?.doSomething()
        }

        // Or unowned if you're sure self will exist
        buttonHandler = { [unowned self] in
            self.doSomething()
        }
    }
}

// Multiple captures
class DataManager {
    func fetchData(completion: @escaping (Data) -> Void) {
        let url = self.apiURL
        let parser = self.parser

        URLSession.shared.dataTask(with: url) { [weak self, weak parser] data, _, _ in
            guard let self = self,
                  let parser = parser,
                  let data = data else { return }

            let processed = parser.process(data)
            self.cache(processed)
            completion(processed)
        }.resume()
    }
}
```

## Value Types: The Memory-Safe Default

Remember from Chapter 5? Structs and enums don't participate in reference counting:

```swift
struct Point {
    var x: Double
    var y: Double
}

struct Line {
    var start: Point
    var end: Point
}

// No reference counting needed - copied by value
var line1 = Line(start: Point(x: 0, y: 0), end: Point(x: 10, y: 10))
var line2 = line1  // Complete copy, no references
```

This is why Swift encourages structs—they can't create reference cycles!

## Advanced Memory Management Patterns

### The Weak-Strong Dance

```swift
class NetworkManager {
    func fetchUser(id: Int, completion: @escaping (User?) -> Void) {
        // Weak-strong dance prevents crashes
        URLSession.shared.dataTask(with: userURL(id)) { [weak self] data, _, error in
            guard let strongSelf = self else { return }  // Upgrade to strong

            // Now strongSelf is guaranteed to exist for this scope
            let user = strongSelf.parseUser(from: data)
            strongSelf.cache(user)
            completion(user)
        }.resume()
    }
}

// Modern alternative with guard
func fetchData() {
    performAsync { [weak self] in
        guard let self = self else { return }
        // self is now a strong reference for this scope
        self.processData()
    }
}
```

### Lazy Properties and Capture

```swift
class DataProcessor {
    lazy var processQueue: DispatchQueue = {
        // This captures self! Be careful
        return DispatchQueue(label: "\(self.identifier).process")
    }()

    // Better: avoid self capture in lazy properties
    lazy var betterQueue: DispatchQueue = {
        return DispatchQueue(label: "com.app.processor")
    }()

    let identifier = "processor"
}
```

### Unowned(unsafe): Living Dangerously

```swift
class RiskyBusiness {
    unowned(unsafe) var unsafeReference: SomeClass

    // This bypasses Swift's safety checks
    // Only use when you absolutely know what you're doing
    // and need the performance (rare!)
}
```

## Memory Debugging Tools

### Debug Helpers

```swift
class MemoryMonitor {
    static func printMemoryUsage() {
        var info = mach_task_basic_info()
        var count = mach_msg_type_number_t(MemoryLayout<mach_task_basic_info>.size / MemoryLayout<natural_t>.size)

        let result = withUnsafeMutablePointer(to: &info) {
            $0.withMemoryRebound(to: integer_t.self, capacity: Int(count)) {
                task_info(mach_task_self_,
                         task_flavor_t(MACH_TASK_BASIC_INFO),
                         $0,
                         &count)
            }
        }

        if result == KERN_SUCCESS {
            let usedMB = Double(info.resident_size) / 1024.0 / 1024.0
            print("Memory used: \(String(format: "%.2f", usedMB)) MB")
        }
    }
}

// Detecting retain cycles in debug
class LeakDetector {
    private weak var weakReference: AnyObject?

    init(_ object: AnyObject) {
        weakReference = object
    }

    func checkLeak(after delay: TimeInterval = 2.0) {
        DispatchQueue.main.asyncAfter(deadline: .now() + delay) { [weak weakReference] in
            if weakReference != nil {
                print("⚠️ Potential memory leak detected!")
            }
        }
    }
}
```

## Common Patterns and Best Practices

### The Delegate Pattern

```swift
protocol DataSourceDelegate: AnyObject {  // AnyObject = class-only protocol
    func dataDidUpdate()
}

class DataSource {
    weak var delegate: DataSourceDelegate?  // Always weak for delegates

    func fetchData() {
        // Fetch data
        delegate?.dataDidUpdate()
    }
}
```

### The Notification Pattern

```swift
class Observer {
    init() {
        // BAD: Creates retain cycle with notification center
        NotificationCenter.default.addObserver(
            self,
            selector: #selector(handleNotification),
            name: .someNotification,
            object: nil
        )
    }

    // GOOD: Remove observer in deinit
    deinit {
        NotificationCenter.default.removeObserver(self)
    }

    @objc private func handleNotification() {
        // Handle
    }
}

// Better: Use closure-based API with weak self
class ModernObserver {
    private var observer: NSObjectProtocol?

    init() {
        observer = NotificationCenter.default.addObserver(
            forName: .someNotification,
            object: nil,
            queue: .main
        ) { [weak self] _ in
            self?.handleNotification()
        }
    }

    deinit {
        if let observer = observer {
            NotificationCenter.default.removeObserver(observer)
        }
    }
}
```

### The Cache Pattern

```swift
class ImageCache {
    private let cache = NSCache<NSString, UIImage>()

    init() {
        // NSCache automatically removes objects under memory pressure
        cache.countLimit = 100
        cache.totalCostLimit = 100 * 1024 * 1024  // 100MB
    }

    func cache(_ image: UIImage, for key: String) {
        cache.setObject(image, forKey: key as NSString, cost: image.memorySize)
    }

    func image(for key: String) -> UIImage? {
        return cache.object(forKey: key as NSString)
    }
}
```

## Performance Tips

### Copy-on-Write for Custom Types

```swift
struct DataBuffer {
    private var storage: Storage

    private mutating func ensureUnique() {
        if !isKnownUniquelyReferenced(&storage) {
            storage = Storage(copying: storage)
        }
    }

    mutating func append(_ byte: UInt8) {
        ensureUnique()  // Copy only if shared
        storage.append(byte)
    }

    private class Storage {
        var bytes: [UInt8]

        init(bytes: [UInt8] = []) {
            self.bytes = bytes
        }

        init(copying other: Storage) {
            self.bytes = other.bytes
        }

        func append(_ byte: UInt8) {
            bytes.append(byte)
        }
    }
}
```

### Autoreleasepool for Loops

```swift
// When creating many temporary objects in a loop
for i in 0..<1000000 {
    autoreleasepool {
        // Temporary objects created here are freed
        // at the end of each iteration
        let data = processLargeData(index: i)
        save(data)
    }
}
```

## Memory Safety Features

Swift provides compile-time guarantees about memory safety:

```swift
// Exclusive access to memory
var numbers = [1, 2, 3]
// This would cause a runtime error:
// numbers.append(numbers[0])  // Simultaneous access!

// Safe alternative
let first = numbers[0]
numbers.append(first)

// Function parameters
func takesTwoInouts(_ a: inout Int, _ b: inout Int) {
    // Implementation
}

var x = 1
// takesTwoInouts(&x, &x)  // ❌ Compiler error: overlapping access
```

## What's Next?

You've mastered Swift's memory management—no manual `malloc`/`free`, no garbage collector pauses, just predictable, efficient ARC. You know how to avoid retain cycles, when to use weak vs unowned, and how to debug memory issues.

Next up: Concurrency. You'll learn Swift's modern approach to concurrent programming with async/await, actors, and structured concurrency. Say goodbye to callback hell and data races!

## Exercises

1. **Leak Detector**: Build a debug tool that automatically detects potential retain cycles in your code.

2. **Memory Pool**: Implement a memory pool for frequently allocated/deallocated objects.

3. **Weak Collection**: Create a collection that holds weak references to its elements.

4. **Reference Graph**: Build a visualizer that shows the reference relationships between objects.

Remember: ARC isn't magic—it's counting. Understand the count, break the cycles, and your apps will be lean, fast, and crash-free.