# Chapter 12: Real-World Swift - Patterns, Packages, and Production

You've learned the language. You've mastered the concepts. Now let's talk about building real Swift applications—the patterns professionals use, the packages that save time, and the practices that keep production code maintainable. This is where theory meets practice.

## Architectural Patterns: Beyond MVC

### The Composable Architecture

```swift
// State: What your app knows
struct AppState: Equatable {
    var todos: [Todo] = []
    var filter: Filter = .all
    var isLoading = false
}

// Actions: What can happen
enum AppAction {
    case todoAdded(String)
    case todoToggled(id: UUID)
    case todoDeleted(id: UUID)
    case filterChanged(Filter)
    case todosLoaded([Todo])
}

// Reducer: How state changes
let appReducer = Reducer<AppState, AppAction, AppEnvironment> { state, action, environment in
    switch action {
    case let .todoAdded(title):
        let todo = Todo(id: UUID(), title: title, isCompleted: false)
        state.todos.append(todo)
        return .none

    case let .todoToggled(id):
        if let index = state.todos.firstIndex(where: { $0.id == id }) {
            state.todos[index].isCompleted.toggle()
        }
        return .none

    case let .todoDeleted(id):
        state.todos.removeAll { $0.id == id }
        return .none

    case let .filterChanged(filter):
        state.filter = filter
        return .none

    case let .todosLoaded(todos):
        state.todos = todos
        state.isLoading = false
        return .none
    }
}

// Environment: Dependencies
struct AppEnvironment {
    var mainQueue: AnySchedulerOf<DispatchQueue>
    var todoClient: TodoClient
}
```

### Coordinator Pattern for Navigation

```swift
protocol Coordinator: AnyObject {
    var childCoordinators: [Coordinator] { get set }
    var navigationController: UINavigationController { get set }

    func start()
}

class MainCoordinator: Coordinator {
    var childCoordinators = [Coordinator]()
    var navigationController: UINavigationController

    init(navigationController: UINavigationController) {
        self.navigationController = navigationController
    }

    func start() {
        let vc = HomeViewController()
        vc.coordinator = self
        navigationController.pushViewController(vc, animated: false)
    }

    func showDetail(for item: Item) {
        let vc = DetailViewController(item: item)
        vc.coordinator = self
        navigationController.pushViewController(vc, animated: true)
    }

    func startPurchaseFlow() {
        let child = PurchaseCoordinator(navigationController: navigationController)
        child.parentCoordinator = self
        childCoordinators.append(child)
        child.start()
    }

    func childDidFinish(_ child: Coordinator?) {
        childCoordinators.removeAll { $0 === child }
    }
}
```

### Clean Architecture with Use Cases

```swift
// Domain Layer - Business Logic
protocol UserRepository {
    func fetchUser(id: String) async throws -> User
    func saveUser(_ user: User) async throws
}

struct GetUserUseCase {
    let repository: UserRepository

    func execute(userId: String) async throws -> User {
        // Business logic here
        let user = try await repository.fetchUser(id: userId)

        // Validate, transform, etc.
        guard user.isActive else {
            throw DomainError.inactiveUser
        }

        return user
    }
}

// Data Layer - Implementation
class RemoteUserRepository: UserRepository {
    private let apiClient: APIClient
    private let cache: Cache<String, User>

    func fetchUser(id: String) async throws -> User {
        // Check cache first
        if let cached = cache[id] {
            return cached
        }

        // Fetch from API
        let user = try await apiClient.request(UserEndpoint.get(id))
        cache[id] = user
        return user
    }

    func saveUser(_ user: User) async throws {
        try await apiClient.request(UserEndpoint.update(user))
        cache[user.id] = user
    }
}

// Presentation Layer - ViewModels
@MainActor
class UserViewModel: ObservableObject {
    @Published var user: User?
    @Published var isLoading = false
    @Published var error: Error?

    private let getUserUseCase: GetUserUseCase

    init(getUserUseCase: GetUserUseCase) {
        self.getUserUseCase = getUserUseCase
    }

    func loadUser(id: String) async {
        isLoading = true
        error = nil

        do {
            user = try await getUserUseCase.execute(userId: id)
        } catch {
            self.error = error
        }

        isLoading = false
    }
}
```

## Dependency Injection: Managing Complexity

### Container-Based DI

```swift
protocol DIContainer {
    func register<T>(_ type: T.Type, factory: @escaping () -> T)
    func resolve<T>(_ type: T.Type) -> T?
}

class AppContainer: DIContainer {
    private var services: [String: Any] = [:]
    private var factories: [String: () -> Any] = [:]

    func register<T>(_ type: T.Type, factory: @escaping () -> T) {
        let key = String(describing: type)
        factories[key] = factory
    }

    func resolve<T>(_ type: T.Type) -> T? {
        let key = String(describing: type)

        // Return existing instance if available
        if let service = services[key] as? T {
            return service
        }

        // Create new instance
        if let factory = factories[key], let service = factory() as? T {
            services[key] = service
            return service
        }

        return nil
    }
}

// Usage
let container = AppContainer()

container.register(APIClient.self) {
    APIClient(baseURL: "https://api.example.com")
}

container.register(UserRepository.self) {
    RemoteUserRepository(apiClient: container.resolve(APIClient.self)!)
}

container.register(GetUserUseCase.self) {
    GetUserUseCase(repository: container.resolve(UserRepository.self)!)
}
```

### Property Wrapper-Based DI

```swift
@propertyWrapper
struct Injected<T> {
    private var value: T?

    var wrappedValue: T {
        get {
            return value ?? Container.shared.resolve(T.self)!
        }
        mutating set {
            value = newValue
        }
    }
}

class ProfileViewModel {
    @Injected var userService: UserService
    @Injected var analyticsService: AnalyticsService

    func loadProfile() async {
        analyticsService.track(.profileViewed)
        let user = try? await userService.getCurrentUser()
        // ...
    }
}
```

## Testing: Confidence in Your Code

### Unit Testing Best Practices

```swift
import XCTest
@testable import YourApp

class UserViewModelTests: XCTestCase {
    var sut: UserViewModel!  // System Under Test
    var mockRepository: MockUserRepository!

    override func setUp() {
        super.setUp()
        mockRepository = MockUserRepository()
        let useCase = GetUserUseCase(repository: mockRepository)
        sut = UserViewModel(getUserUseCase: useCase)
    }

    override func tearDown() {
        sut = nil
        mockRepository = nil
        super.tearDown()
    }

    func testLoadUser_Success() async {
        // Given
        let expectedUser = User(id: "1", name: "Alice", isActive: true)
        mockRepository.userToReturn = expectedUser

        // When
        await sut.loadUser(id: "1")

        // Then
        XCTAssertEqual(sut.user?.id, expectedUser.id)
        XCTAssertFalse(sut.isLoading)
        XCTAssertNil(sut.error)
    }

    func testLoadUser_Failure() async {
        // Given
        mockRepository.errorToThrow = NetworkError.notFound

        // When
        await sut.loadUser(id: "1")

        // Then
        XCTAssertNil(sut.user)
        XCTAssertNotNil(sut.error)
        XCTAssertFalse(sut.isLoading)
    }
}

// Mock implementation
class MockUserRepository: UserRepository {
    var userToReturn: User?
    var errorToThrow: Error?
    var saveUserCalled = false

    func fetchUser(id: String) async throws -> User {
        if let error = errorToThrow {
            throw error
        }
        return userToReturn ?? User(id: id, name: "Mock", isActive: true)
    }

    func saveUser(_ user: User) async throws {
        saveUserCalled = true
        if let error = errorToThrow {
            throw error
        }
    }
}
```

### UI Testing with Page Object Pattern

```swift
class LoginScreen {
    let app: XCUIApplication

    init(app: XCUIApplication) {
        self.app = app
    }

    var usernameField: XCUIElement {
        app.textFields["username"]
    }

    var passwordField: XCUIElement {
        app.secureTextFields["password"]
    }

    var loginButton: XCUIElement {
        app.buttons["login"]
    }

    var errorLabel: XCUIElement {
        app.staticTexts["error"]
    }

    func login(username: String, password: String) {
        usernameField.tap()
        usernameField.typeText(username)

        passwordField.tap()
        passwordField.typeText(password)

        loginButton.tap()
    }
}

class LoginUITests: XCTestCase {
    var app: XCUIApplication!
    var loginScreen: LoginScreen!

    override func setUp() {
        continueAfterFailure = false
        app = XCUIApplication()
        app.launch()
        loginScreen = LoginScreen(app: app)
    }

    func testSuccessfulLogin() {
        loginScreen.login(username: "user@example.com", password: "password123")

        // Verify we navigated to home screen
        XCTAssertTrue(app.navigationBars["Home"].exists)
    }

    func testInvalidCredentials() {
        loginScreen.login(username: "wrong", password: "wrong")

        XCTAssertTrue(loginScreen.errorLabel.exists)
        XCTAssertEqual(loginScreen.errorLabel.label, "Invalid credentials")
    }
}
```

## Package Management: Standing on Giants' Shoulders

### Creating Swift Packages

```swift
// Package.swift
// swift-tools-version: 5.9

import PackageDescription

let package = Package(
    name: "NetworkKit",
    platforms: [
        .iOS(.v15),
        .macOS(.v12),
        .watchOS(.v8),
        .tvOS(.v15)
    ],
    products: [
        .library(
            name: "NetworkKit",
            targets: ["NetworkKit"]
        ),
        .executable(
            name: "NetworkKitExample",
            targets: ["NetworkKitExample"]
        )
    ],
    dependencies: [
        .package(url: "https://github.com/apple/swift-log.git", from: "1.5.0"),
        .package(url: "https://github.com/apple/swift-metrics.git", from: "2.3.0")
    ],
    targets: [
        .target(
            name: "NetworkKit",
            dependencies: [
                .product(name: "Logging", package: "swift-log"),
                .product(name: "Metrics", package: "swift-metrics")
            ]
        ),
        .testTarget(
            name: "NetworkKitTests",
            dependencies: ["NetworkKit"]
        ),
        .executableTarget(
            name: "NetworkKitExample",
            dependencies: ["NetworkKit"]
        )
    ]
)
```

### Essential Packages for Real Apps

```swift
// Common dependencies in Package.swift
dependencies: [
    // Networking
    .package(url: "https://github.com/Alamofire/Alamofire.git", from: "5.8.0"),
    .package(url: "https://github.com/Moya/Moya.git", from: "15.0.0"),

    // Database
    .package(url: "https://github.com/realm/realm-swift.git", from: "10.42.0"),
    .package(url: "https://github.com/groue/GRDB.swift.git", from: "6.20.0"),

    // Reactive
    .package(url: "https://github.com/CombineCommunity/CombineExt.git", from: "1.8.0"),

    // Utilities
    .package(url: "https://github.com/kishikawakatsumi/KeychainAccess.git", from: "4.2.0"),
    .package(url: "https://github.com/SwiftyJSON/SwiftyJSON.git", from: "5.0.0"),

    // Code Quality
    .package(url: "https://github.com/realm/SwiftLint.git", from: "0.52.0"),
    .package(url: "https://github.com/nicklockwood/SwiftFormat.git", from: "0.52.0")
]
```

## Performance Optimization: Making It Fast

### Profiling with Instruments

```swift
// Signpost for performance measurement
import os.signpost

class DataProcessor {
    private let log = OSLog(subsystem: "com.app.processor", category: .pointsOfInterest)

    func processLargeDataset(_ data: [DataPoint]) {
        let signpostID = OSSignpostID(log: log)

        os_signpost(.begin, log: log, name: "Process Dataset", signpostID: signpostID)

        // Processing phases
        os_signpost(.begin, log: log, name: "Sort", signpostID: signpostID)
        let sorted = data.sorted { $0.timestamp < $1.timestamp }
        os_signpost(.end, log: log, name: "Sort", signpostID: signpostID)

        os_signpost(.begin, log: log, name: "Filter", signpostID: signpostID)
        let filtered = sorted.filter { $0.isValid }
        os_signpost(.end, log: log, name: "Filter", signpostID: signpostID)

        os_signpost(.begin, log: log, name: "Transform", signpostID: signpostID)
        let results = filtered.map { transform($0) }
        os_signpost(.end, log: log, name: "Transform", signpostID: signpostID)

        os_signpost(.end, log: log, name: "Process Dataset", signpostID: signpostID)
    }
}
```

### Memory and Performance Optimizations

```swift
// String optimization
extension String {
    // Avoid creating intermediate strings
    func efficientSplit(separator: Character) -> [Substring] {
        return self.split(separator: separator)  // Returns Substring, not String
    }
}

// Collection optimization
extension Collection where Element: Comparable {
    // Lazy evaluation for large collections
    func topN(_ n: Int) -> [Element] {
        // Don't sort entire collection
        return Array(self.sorted().prefix(n))  // Bad

        // Use partial sorting
        return Array(self.prefix(n).sorted())  // Better for large collections
    }
}

// Struct optimization
struct OptimizedData {
    // Pack data efficiently
    private let storage: UInt64  // Pack multiple values

    var flag1: Bool { storage & 0x1 != 0 }
    var flag2: Bool { storage & 0x2 != 0 }
    var smallValue: UInt8 { UInt8((storage >> 8) & 0xFF) }

    init(flag1: Bool, flag2: Bool, smallValue: UInt8) {
        storage = (flag1 ? 1 : 0) |
                  (flag2 ? 2 : 0) |
                  (UInt64(smallValue) << 8)
    }
}
```

## Security: Protecting User Data

```swift
import CryptoKit
import LocalAuthentication

class SecurityManager {
    // Biometric authentication
    func authenticateWithBiometrics() async throws -> Bool {
        let context = LAContext()
        var error: NSError?

        guard context.canEvaluatePolicy(.deviceOwnerAuthenticationWithBiometrics, error: &error) else {
            throw SecurityError.biometricsNotAvailable
        }

        return try await withCheckedThrowingContinuation { continuation in
            context.evaluatePolicy(
                .deviceOwnerAuthenticationWithBiometrics,
                localizedReason: "Authenticate to access your account"
            ) { success, error in
                if success {
                    continuation.resume(returning: true)
                } else if let error = error {
                    continuation.resume(throwing: error)
                } else {
                    continuation.resume(returning: false)
                }
            }
        }
    }

    // Keychain storage
    func saveToKeychain(key: String, data: Data) throws {
        let query: [String: Any] = [
            kSecClass as String: kSecClassGenericPassword,
            kSecAttrAccount as String: key,
            kSecValueData as String: data,
            kSecAttrAccessible as String: kSecAttrAccessibleWhenUnlockedThisDeviceOnly
        ]

        SecItemDelete(query as CFDictionary)  // Remove existing

        let status = SecItemAdd(query as CFDictionary, nil)
        guard status == errSecSuccess else {
            throw SecurityError.keychainError(status)
        }
    }

    // Encryption
    func encrypt(data: Data, using key: SymmetricKey) throws -> Data {
        let sealedBox = try AES.GCM.seal(data, using: key)
        return sealedBox.combined ?? Data()
    }

    func decrypt(data: Data, using key: SymmetricKey) throws -> Data {
        let sealedBox = try AES.GCM.SealedBox(combined: data)
        return try AES.GCM.open(sealedBox, using: key)
    }
}
```

## Logging and Analytics: Understanding Your App

```swift
import OSLog

// Structured logging
class AppLogger {
    static let shared = AppLogger()

    private let defaultLog = Logger(subsystem: "com.app", category: "general")
    private let networkLog = Logger(subsystem: "com.app", category: "network")
    private let uiLog = Logger(subsystem: "com.app", category: "ui")

    func logNetworkRequest(url: String, method: String) {
        networkLog.info("Request: \(method, privacy: .public) \(url, privacy: .private)")
    }

    func logError(_ error: Error, context: String) {
        defaultLog.error("Error in \(context, privacy: .public): \(error.localizedDescription, privacy: .public)")
    }

    func logUserAction(_ action: String, metadata: [String: Any] = [:]) {
        uiLog.info("User action: \(action, privacy: .public) - \(metadata, privacy: .private)")
    }
}

// Analytics abstraction
protocol AnalyticsService {
    func track(event: String, properties: [String: Any]?)
    func setUser(id: String, traits: [String: Any]?)
}

class AnalyticsManager: AnalyticsService {
    private let services: [AnalyticsService]

    init(services: [AnalyticsService]) {
        self.services = services
    }

    func track(event: String, properties: [String: Any]?) {
        services.forEach { $0.track(event: event, properties: properties) }
    }

    func setUser(id: String, traits: [String: Any]?) {
        services.forEach { $0.setUser(id: id, traits: traits) }
    }
}
```

## CI/CD: Ship It!

### Fastlane Configuration

```ruby
# Fastfile
default_platform(:ios)

platform :ios do
  desc "Run tests"
  lane :test do
    run_tests(
      scheme: "YourApp",
      devices: ["iPhone 14 Pro", "iPad Pro (12.9-inch)"],
      code_coverage: true
    )
  end

  desc "Build and upload to TestFlight"
  lane :beta do
    increment_build_number(xcodeproj: "YourApp.xcodeproj")

    build_app(
      scheme: "YourApp",
      export_method: "app-store",
      export_options: {
        provisioningProfiles: {
          "com.yourcompany.app" => "YourApp Distribution"
        }
      }
    )

    upload_to_testflight(
      skip_waiting_for_build_processing: true,
      apple_id: ENV["APPLE_ID"]
    )

    slack(
      message: "New build uploaded to TestFlight! 🚀",
      channel: "#releases"
    )
  end

  desc "Deploy to App Store"
  lane :release do
    ensure_git_status_clean
    test

    build_app(scheme: "YourApp")
    upload_to_app_store(
      skip_screenshots: true,
      skip_metadata: false,
      force: true,
      submit_for_review: true,
      automatic_release: true
    )

    add_git_tag
    push_git_tags
  end
end
```

## Production Checklist

```swift
// App configuration for different environments
enum Environment {
    case development
    case staging
    case production

    var baseURL: String {
        switch self {
        case .development: return "https://dev-api.example.com"
        case .staging: return "https://staging-api.example.com"
        case .production: return "https://api.example.com"
        }
    }

    var analyticsEnabled: Bool {
        self == .production
    }

    var logLevel: Logger.Level {
        switch self {
        case .development: return .debug
        case .staging: return .info
        case .production: return .warning
        }
    }
}

// Build configuration
#if DEBUG
let environment = Environment.development
#elseif STAGING
let environment = Environment.staging
#else
let environment = Environment.production
#endif
```

## Your Swift Journey Continues

Congratulations! You've journeyed from Swift basics to production-ready patterns. You now have the tools to build robust, maintainable, and performant Swift applications. But remember: Swift evolves rapidly. Stay curious, keep learning, and most importantly, keep building.

## Final Project

Build a complete iOS app that demonstrates everything you've learned:

1. **Architecture**: Use MVVM or Clean Architecture
2. **Concurrency**: Implement async/await for network calls
3. **SwiftUI**: Build the entire UI declaratively
4. **Testing**: Achieve 80% code coverage
5. **Packages**: Use SPM for dependencies
6. **CI/CD**: Set up automated testing and deployment

## Resources for Continued Learning

- **Swift Evolution**: Follow Swift's development at swift.org/evolution
- **WWDC Videos**: Apple's annual goldmine of Swift knowledge
- **Swift Forums**: Engage with the community at forums.swift.org
- **Open Source**: Contribute to Swift packages on GitHub

Remember: Great Swift developers aren't born—they're compiled, optimized, and continuously deployed. Welcome to the Swift community. Now go build something amazing!

## The End... Or Just the Beginning?

```swift
struct Developer {
    var knowledge: Set<Skill> = []
    let curiosity = Double.infinity

    mutating func learn(_ skill: Skill) {
        knowledge.insert(skill)
        print("Learned \(skill). \(knowledge.count) skills and counting...")
    }
}

var you = Developer()
you.learn(.swift)
// Your journey continues...
```