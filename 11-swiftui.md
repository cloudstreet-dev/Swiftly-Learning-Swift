# Chapter 11: SwiftUI - Building UIs Like It's 2025

Remember XML layouts? Storyboards? Connecting IBOutlets and praying you didn't typo? SwiftUI throws all that out the window. It's declarative, reactive, and absolutely delightful. Your UI is now code—real, type-safe, version-controllable code. And when your data changes, your UI updates automatically. Magic? No, just really good design.

## The Paradigm Shift: Declarative UI

### From Imperative to Declarative

```swift
// The old UIKit way - imperative
class OldViewController: UIViewController {
    override func viewDidLoad() {
        super.viewDidLoad()

        let label = UILabel()
        label.text = "Hello"
        label.textColor = .blue
        label.font = .systemFont(ofSize: 24)

        view.addSubview(label)
        label.translatesAutoresizingMaskIntoConstraints = false
        NSLayoutConstraint.activate([
            label.centerXAnchor.constraint(equalTo: view.centerXAnchor),
            label.centerYAnchor.constraint(equalTo: view.centerYAnchor)
        ])
    }
}

// The SwiftUI way - declarative
struct ContentView: View {
    var body: some View {
        Text("Hello")
            .foregroundColor(.blue)
            .font(.system(size: 24))
    }
}
// That's it. Centering is automatic.
```

## Views: Everything Is a Struct

```swift
struct WelcomeView: View {
    var body: some View {
        VStack(spacing: 20) {
            Text("Welcome to SwiftUI")
                .font(.largeTitle)
                .fontWeight(.bold)

            Text("Build amazing apps")
                .font(.subheadline)
                .foregroundColor(.secondary)

            Button("Get Started") {
                print("Button tapped")
            }
            .buttonStyle(.borderedProminent)
        }
        .padding()
    }
}

// Views are just descriptions - lightweight and fast
let view1 = Text("Hello")
let view2 = Text("Hello")
// These create new view descriptions, not actual UI elements
```

## State Management: The Heart of SwiftUI

### @State: Local View State

```swift
struct CounterView: View {
    @State private var count = 0

    var body: some View {
        VStack {
            Text("Count: \(count)")
                .font(.largeTitle)

            HStack {
                Button("Decrement") {
                    count -= 1
                }

                Button("Increment") {
                    count += 1
                }
            }
            .buttonStyle(.bordered)
        }
    }
}
// When count changes, the UI automatically updates!
```

### @Binding: Two-Way Data Flow

```swift
struct ParentView: View {
    @State private var isOn = false

    var body: some View {
        VStack {
            Toggle("Master Switch", isOn: $isOn)  // $ creates a binding

            ChildView(isEnabled: $isOn)
        }
        .padding()
    }
}

struct ChildView: View {
    @Binding var isEnabled: Bool

    var body: some View {
        VStack {
            Text(isEnabled ? "Enabled" : "Disabled")
                .foregroundColor(isEnabled ? .green : .red)

            Button("Toggle from Child") {
                isEnabled.toggle()  // Updates parent's state!
            }
        }
    }
}
```

### @StateObject and @ObservedObject: Reference Types

```swift
class UserViewModel: ObservableObject {
    @Published var name = "Anonymous"
    @Published var score = 0

    func incrementScore() {
        score += 1
    }
}

struct GameView: View {
    @StateObject private var viewModel = UserViewModel()  // Owns the object

    var body: some View {
        VStack {
            Text("Player: \(viewModel.name)")
            Text("Score: \(viewModel.score)")

            Button("Score!") {
                viewModel.incrementScore()
            }

            DetailView(viewModel: viewModel)
        }
    }
}

struct DetailView: View {
    @ObservedObject var viewModel: UserViewModel  // Observes but doesn't own

    var body: some View {
        TextField("Name", text: $viewModel.name)
            .textFieldStyle(.roundedBorder)
    }
}
```

### @EnvironmentObject: Dependency Injection

```swift
class AppSettings: ObservableObject {
    @Published var theme = "Light"
    @Published var fontSize = 16.0
}

struct RootView: View {
    @StateObject private var settings = AppSettings()

    var body: some View {
        NavigationView {
            ContentView()
        }
        .environmentObject(settings)  // Inject into environment
    }
}

struct DeepChildView: View {
    @EnvironmentObject var settings: AppSettings  // Automatically injected

    var body: some View {
        VStack {
            Text("Current theme: \(settings.theme)")
            Slider(value: $settings.fontSize, in: 12...24)
        }
    }
}
```

## Layout: Stacks, Grids, and More

### Stacks: The Foundation

```swift
struct LayoutExample: View {
    var body: some View {
        VStack(alignment: .leading, spacing: 10) {
            Text("Title")
                .font(.title)

            HStack {
                Image(systemName: "star.fill")
                    .foregroundColor(.yellow)
                Text("5.0")
                Spacer()  // Pushes content apart
                Text("100 reviews")
            }

            ZStack {
                Rectangle()
                    .fill(.blue)
                    .frame(width: 100, height: 100)

                Text("Overlay")
                    .foregroundColor(.white)
            }
        }
        .padding()
    }
}
```

### Grids: When You Need More Control

```swift
struct GridExample: View {
    let items = Array(1...20)
    let columns = [
        GridItem(.fixed(100)),
        GridItem(.flexible()),
        GridItem(.adaptive(minimum: 50))
    ]

    var body: some View {
        ScrollView {
            LazyVGrid(columns: columns, spacing: 20) {
                ForEach(items, id: \.self) { item in
                    RoundedRectangle(cornerRadius: 10)
                        .fill(.blue)
                        .frame(height: 50)
                        .overlay(Text("\(item)").foregroundColor(.white))
                }
            }
            .padding()
        }
    }
}
```

## Modifiers: Transform Your Views

```swift
struct ModifierChaining: View {
    var body: some View {
        Text("SwiftUI")
            .font(.largeTitle)
            .fontWeight(.bold)
            .foregroundColor(.blue)
            .padding()
            .background(.yellow)
            .cornerRadius(10)
            .shadow(radius: 5)
            .scaleEffect(1.2)
            .rotationEffect(.degrees(5))
    }
}

// Custom modifiers
struct CardModifier: ViewModifier {
    func body(content: Content) -> some View {
        content
            .padding()
            .background(Color(.systemBackground))
            .cornerRadius(12)
            .shadow(radius: 4)
    }
}

extension View {
    func cardStyle() -> some View {
        modifier(CardModifier())
    }
}

// Usage
Text("Custom styled").cardStyle()
```

## Lists and Navigation

```swift
struct Todo: Identifiable {
    let id = UUID()
    var title: String
    var isCompleted: Bool
}

struct TodoListView: View {
    @State private var todos = [
        Todo(title: "Learn SwiftUI", isCompleted: false),
        Todo(title: "Build an app", isCompleted: false)
    ]

    var body: some View {
        NavigationStack {
            List {
                ForEach($todos) { $todo in
                    NavigationLink(destination: TodoDetailView(todo: $todo)) {
                        HStack {
                            Image(systemName: todo.isCompleted ? "checkmark.circle.fill" : "circle")
                                .foregroundColor(todo.isCompleted ? .green : .gray)
                            Text(todo.title)
                                .strikethrough(todo.isCompleted)
                        }
                    }
                }
                .onDelete(perform: deleteTodos)
                .onMove(perform: moveTodos)
            }
            .navigationTitle("Todos")
            .toolbar {
                ToolbarItem(placement: .navigationBarTrailing) {
                    EditButton()
                }
                ToolbarItem(placement: .navigationBarLeading) {
                    Button("Add") {
                        todos.append(Todo(title: "New Todo", isCompleted: false))
                    }
                }
            }
        }
    }

    func deleteTodos(at offsets: IndexSet) {
        todos.remove(atOffsets: offsets)
    }

    func moveTodos(from source: IndexSet, to destination: Int) {
        todos.move(fromOffsets: source, toOffset: destination)
    }
}
```

## Animations: Bringing UI to Life

```swift
struct AnimationExamples: View {
    @State private var scale = 1.0
    @State private var isRotated = false
    @State private var isVisible = true

    var body: some View {
        VStack(spacing: 30) {
            // Implicit animation
            Circle()
                .fill(.blue)
                .frame(width: 100, height: 100)
                .scaleEffect(scale)
                .onTapGesture {
                    scale = scale == 1.0 ? 1.5 : 1.0
                }
                .animation(.spring(), value: scale)

            // Explicit animation
            Rectangle()
                .fill(.green)
                .frame(width: 100, height: 100)
                .rotationEffect(.degrees(isRotated ? 180 : 0))
                .onTapGesture {
                    withAnimation(.easeInOut(duration: 0.5)) {
                        isRotated.toggle()
                    }
                }

            // Transition
            if isVisible {
                Text("Fade me!")
                    .padding()
                    .background(.orange)
                    .transition(.asymmetric(
                        insertion: .scale.combined(with: .opacity),
                        removal: .slide
                    ))
            }

            Button("Toggle Visibility") {
                withAnimation {
                    isVisible.toggle()
                }
            }
        }
    }
}

// Custom animation
extension Animation {
    static var myBounce: Animation {
        Animation.spring(response: 0.5, dampingFraction: 0.5, blendDuration: 0.5)
    }
}
```

## Gestures: Touch Interactions

```swift
struct GestureExamples: View {
    @State private var offset = CGSize.zero
    @State private var scale = 1.0
    @State private var rotation = 0.0

    var body: some View {
        VStack {
            // Drag gesture
            Circle()
                .fill(.blue)
                .frame(width: 100, height: 100)
                .offset(offset)
                .gesture(
                    DragGesture()
                        .onChanged { value in
                            offset = value.translation
                        }
                        .onEnded { _ in
                            withAnimation(.spring()) {
                                offset = .zero
                            }
                        }
                )

            // Combined gestures
            Image(systemName: "star.fill")
                .font(.system(size: 100))
                .foregroundColor(.yellow)
                .scaleEffect(scale)
                .rotationEffect(.degrees(rotation))
                .gesture(
                    MagnificationGesture()
                        .onChanged { value in
                            scale = value
                        }
                        .simultaneously(with:
                            RotationGesture()
                                .onChanged { value in
                                    rotation = value.degrees
                                }
                        )
                )
        }
    }
}
```

## Drawing and Shapes

```swift
struct CustomDrawing: View {
    var body: some View {
        VStack {
            // Basic shapes
            Circle()
                .fill(LinearGradient(
                    colors: [.blue, .purple],
                    startPoint: .topLeading,
                    endPoint: .bottomTrailing
                ))
                .frame(width: 100, height: 100)

            // Custom path
            CustomShape()
                .stroke(Color.red, lineWidth: 3)
                .frame(width: 200, height: 100)

            // Canvas for complex drawing
            Canvas { context, size in
                let rect = CGRect(origin: .zero, size: size)

                // Draw gradient background
                context.fill(
                    Path(rect),
                    with: .linearGradient(
                        Gradient(colors: [.yellow, .orange]),
                        startPoint: .zero,
                        endPoint: CGPoint(x: size.width, y: size.height)
                    )
                )

                // Draw circle
                context.draw(
                    Circle().path(in: CGRect(x: 50, y: 50, width: 100, height: 100)),
                    with: .color(.white)
                )
            }
            .frame(height: 200)
        }
    }
}

struct CustomShape: Shape {
    func path(in rect: CGRect) -> Path {
        var path = Path()
        path.move(to: CGPoint(x: rect.minX, y: rect.midY))
        path.addQuadCurve(
            to: CGPoint(x: rect.maxX, y: rect.midY),
            control: CGPoint(x: rect.midX, y: rect.minY)
        )
        return path
    }
}
```

## Data Flow and Architecture

### MVVM Pattern in SwiftUI

```swift
// Model
struct Task: Identifiable, Codable {
    let id: UUID
    var title: String
    var isCompleted: Bool
    var priority: Priority

    enum Priority: String, CaseIterable, Codable {
        case low, medium, high
    }
}

// ViewModel
@MainActor
class TaskViewModel: ObservableObject {
    @Published var tasks: [Task] = []
    @Published var isLoading = false
    @Published var errorMessage: String?

    private let repository: TaskRepository

    init(repository: TaskRepository = TaskRepository()) {
        self.repository = repository
    }

    func loadTasks() async {
        isLoading = true
        errorMessage = nil

        do {
            tasks = try await repository.fetchTasks()
        } catch {
            errorMessage = error.localizedDescription
        }

        isLoading = false
    }

    func addTask(title: String, priority: Task.Priority) async {
        let task = Task(
            id: UUID(),
            title: title,
            isCompleted: false,
            priority: priority
        )

        tasks.append(task)

        do {
            try await repository.save(task)
        } catch {
            // Remove on failure
            tasks.removeAll { $0.id == task.id }
            errorMessage = "Failed to save task"
        }
    }

    func toggleTask(_ task: Task) {
        if let index = tasks.firstIndex(where: { $0.id == task.id }) {
            tasks[index].isCompleted.toggle()
        }
    }
}

// View
struct TaskListView: View {
    @StateObject private var viewModel = TaskViewModel()

    var body: some View {
        NavigationStack {
            ZStack {
                if viewModel.isLoading {
                    ProgressView("Loading...")
                } else {
                    List(viewModel.tasks) { task in
                        TaskRow(task: task) {
                            viewModel.toggleTask(task)
                        }
                    }
                }
            }
            .navigationTitle("Tasks")
            .task {
                await viewModel.loadTasks()
            }
            .alert("Error", isPresented: .constant(viewModel.errorMessage != nil)) {
                Button("OK") {
                    viewModel.errorMessage = nil
                }
            } message: {
                Text(viewModel.errorMessage ?? "")
            }
        }
    }
}
```

## Platform-Specific Features

```swift
struct PlatformSpecificView: View {
    var body: some View {
        VStack {
            #if os(iOS)
            Text("Running on iOS")
                .navigationBarTitleDisplayMode(.inline)
            #elseif os(macOS)
            Text("Running on macOS")
                .frame(minWidth: 400, minHeight: 300)
            #elseif os(watchOS)
            Text("Running on watchOS")
                .font(.caption)
            #endif

            // Conditional modifiers
            Text("Adaptive Text")
                .modifier(AdaptiveModifier())
        }
    }
}

struct AdaptiveModifier: ViewModifier {
    func body(content: Content) -> some View {
        #if os(iOS)
        content
            .padding()
            .background(Color(.systemBackground))
        #else
        content
            .padding(4)
        #endif
    }
}
```

## Performance Optimization

```swift
struct OptimizedList: View {
    let items = Array(1...10000)

    var body: some View {
        ScrollView {
            // LazyVStack only creates visible views
            LazyVStack {
                ForEach(items, id: \.self) { item in
                    ExpensiveRow(number: item)
                }
            }
        }
    }
}

struct ExpensiveRow: View {
    let number: Int

    var body: some View {
        HStack {
            Text("Item \(number)")
            Spacer()
            // Use task modifier for async work
            Text("")
                .task {
                    // Expensive computation here
                }
        }
        .padding()
        // Use id modifier to control view identity
        .id(number)
    }
}

// Prevent unnecessary redraws
struct OptimizedView: View {
    @State private var counter = 0

    var body: some View {
        VStack {
            ExpensiveChildView()
                // Child won't rebuild when counter changes
                .equatable()

            Button("Increment: \(counter)") {
                counter += 1
            }
        }
    }
}

struct ExpensiveChildView: View, Equatable {
    var body: some View {
        // Complex view hierarchy
        Text("Expensive to render")
    }

    static func == (lhs: Self, rhs: Self) -> Bool {
        true  // Never changes, so never rebuilds
    }
}
```

## What's Next?

You've mastered SwiftUI—the future of Apple platform development. You can build reactive, declarative UIs that automatically update with your data. No more massive view controllers, no more manual UI updates, just clean, composable views.

In our final chapter, we'll bring everything together with real-world patterns, best practices, and production-ready Swift code.

## Exercises

1. **Weather App**: Build a weather app with animations, gestures, and custom drawings for weather conditions.

2. **Task Manager**: Create a full-featured task management app with Core Data integration and sync.

3. **Custom Component**: Design a reusable, customizable chart component using Canvas and Shape.

4. **Multi-Platform**: Build an app that runs on iOS, macOS, and watchOS with platform-specific optimizations.

Remember: SwiftUI isn't just a UI framework—it's a new way of thinking about app development. Embrace the declarative paradigm, and your UIs will practically write themselves.