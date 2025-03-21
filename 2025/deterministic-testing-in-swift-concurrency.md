# Deterministic Unit Tests in Swift Concurrency

## Introduction

Testing code that leverages Swift Concurrency can be challenging, especially when dealing with unstructured tasks. These tasks execute asynchronously, making the order of execution, and thus test results, unpredictable. In this article, we explore how dependency inversion and a custom TaskProvider abstraction can help control asynchronous execution, ensuring reliable and deterministic tests.

## Problem

In Swift Concurrency, tasks are fundamental units of work that can be executed concurrently. They represent asynchronous operations that can be run in parallel with other tasks. Tasks allow you to create, manage, and execute code concurrently in a structured way.

However, when initializing an unstructured task inside a synchronous method, the task may continue executing even after the method returns. This is because the task is initialized with an escaping closure. Let's consider the following example:

```swift
class SomethingDoer {

    var somethingHasBeenDone = false

    init() {}

    func doSomething() {

        guard !somethingHasBeenDone else {
            return
        }

        print("task about to start executing")
        Task(priority: .background) { [weak self] in
            print("task started executing")
            await self?.mutateState()
            print("task finished executing")
        }
        print("synchronous method `doSomething` about to return")
    }

    func mutateState() async {
        print("state about to be mutated")
        somethingHasBeenDone = true
        print("state mutated")
    }
}

class SomethingDoerTests: XCTestCase {
    func test_doSomething() {
        let sut = SomethingDoer()
        print("about to call synchronous `doSomething` method")
        sut.doSomething()
        print("about to assert")
        XCTAssertTrue(sut.somethingHasBeenDone) // test fails
        print("finished asserting")
    }
}
```

`doSomething()` is synchronous and initializes a task that mutates the class's internal state. Print statements have been interspersed throughout `doSomething()` and `test_doSomething()` in order to illustrate the order of execution. Below is an example of the order of execution when running the test:
```
about to call synchronous `doSomething` method
task about to start executing
synchronous method `doSomething` about to return
task started executing
about to assert
finished asserting
state about to be mutated
state mutated
task finished executing
```

Every time we run the test, the order of execution could differ from the previous test run. In the case above, the task starts executing after we assert our state. In other cases, the internal state can be mutated before the test begins evaluating the value of the property. Because of the unpredictable order of execution, we get flaky tests.

The unpredictable execution order is due to the task being initialized with an escaping closure, meaning we have no control over when it starts or completes.

### Solution

To make our tests reliable, we need to ensure that the asynchronous operation completes before we assert the state. Once the asynchronous operation has finished, we can evaluate our state deterministically. The remainder of this article will be dedicated to exploring a solution that allows us to achieve that.

The solution leverages dependency inversion and abstracts `Task` behind a protocol that can be mocked for testing purposes. As such, parts of your code that use unstructured tasks will depend on the protocol.

Apple's [Task](https://developer.apple.com/documentation/swift/task) type has two main initializers that we're interested in:
```swift
@discardableResult
init(
    priority: TaskPriority? = nil,
    operation: sending @escaping @isolated(any) () async -> Success
)

@discardableResult
init(
    priority: TaskPriority? = nil,
    operation: sending @escaping @isolated(any) () async throws -> Success
)
```

We need the members of our protocol to resemble those two initializers. Here is the protocol:

```swift
public protocol TaskProvider: Sendable {

    @discardableResult
    func task<Success: Sendable>(priority: TaskPriority?, operation: sending @escaping @isolated(any) () async -> Success) -> Task<Success, Never>

    @discardableResult
    func task<Success: Sendable>(priority: TaskPriority?, operation: sending @escaping @isolated(any) () async throws -> Success) -> Task<Success, Error>

    @discardableResult
    func detachedTask<Success: Sendable>(priority: TaskPriority?, operation: sending @escaping @isolated(any) () async -> Success) -> Task<Success, Never>

    @discardableResult
    func detachedTask<Success: Sendable>(priority: TaskPriority?, operation: sending @escaping @isolated(any) () async throws -> Success) -> Task<Success, Error>
}
```

This protocol will have two implementations: 
- production - the implementation that will be used in production
- mock - the implementation that will be used in tests 

The production implementation is trivial, so we'll take a look at that first.

#### Production implementation


```swift
struct TaskProviderImpl: TaskProvider {

    public init() {}

    @discardableResult
    public func task<Success: Sendable>(priority: TaskPriority?, operation: sending @escaping @isolated(any) () async -> Success) -> Task<Success, Never> {
        Task(priority: priority, operation: operation)
    }

    @discardableResult
    public func task<Success: Sendable>(priority: TaskPriority?, operation: sending @escaping @isolated(any) () async throws -> Success) -> Task<Success, Error> {
        Task(priority: priority, operation: operation)
    }

    @discardableResult
    public func detachedTask<Success: Sendable>(priority: TaskPriority?, operation: sending @escaping @isolated(any) () async -> Success) -> Task<Success, Never> {
        Task.detached(priority: priority, operation: operation)
    }

    @discardableResult
    public func detachedTask<Success: Sendable>(priority: TaskPriority?, operation: sending @escaping @isolated(any) () async throws -> Success) -> Task<Success, Error> {
        Task.detached(priority: priority, operation: operation)
    }
}
```

Each method initializes and returns a corresponding Task instance, encapsulating the asynchronous operation. In other words, rather than initializing an unstructured task ourselves, we can call the methods instead.

#### Mock implementation

The mock implementation is slightly more complex than the production implementation. This is because the mock must:
1. keep track of both the number of unstructured tasks that have been initialized and finished executing
2. wait until all of the created tasks have finished executing

To achieve the first objective, we create properties that store the number of initialized and completed tasks as integers. The values of these properties will be incremented accordingly.

For the second objective, we compare the number of initialized tasks to the number of completed tasks. If the number of initialized tasks is greater than the number completed tasks, that means there is at least one task in progress. Once those two numbers are equal, all tasks have completed and we no longer need to wait.

```swift
import Synchronization

public final class TaskProviderMock: TaskProvider, Sendable {

    public enum MethodCall: Equatable, Sendable {
        case task(priority: TaskPriority?)
        case detachedTask(priority: TaskPriority?)
    }

    public let log = Mutex<[MethodCall]>([])
    private let completedTasksCount = Mutex(0)
    private let tasksCount = Mutex(0)

    public init() {}

    public func task<Success: Sendable>(priority: TaskPriority?, operation: sending @escaping @isolated(any) () async -> Success) -> Task<Success, Never> {
        log.withLock { $0.append(.task(priority: priority)) }
        tasksCount.withLock { $0 += 1 }
        return Task(priority: priority) { [weak self] in
            defer { self?.completedTasksCount.withLock { $0 += 1 } }
            let result = await operation()
            return result
        }
    }

    public func task<Success: Sendable>(priority: TaskPriority?, operation: sending @escaping @isolated(any) () async throws -> Success) -> Task<Success, Error> {
        log.withLock { $0.append(.task(priority: priority)) }
        tasksCount.withLock { $0 += 1 }
        return Task(priority: priority) { [weak self] in
            defer { self?.completedTasksCount.withLock { $0 += 1 } }
            do {
                let result = try await operation()
                return result
            } catch {
                throw error
            }
        }
    }

    public func detachedTask<Success: Sendable>(priority: TaskPriority?, operation: sending @escaping @isolated(any) () async -> Success) -> Task<Success, Never> {
        log.withLock { $0.append(.detachedTask(priority: priority)) }
        tasksCount.withLock { $0 += 1 }
        return Task.detached(priority: priority) { [weak self] in
            defer { self?.completedTasksCount.withLock { $0 += 1 } }
            let result = await operation()
            return result
        }
    }

    public func detachedTask<Success: Sendable>(priority: TaskPriority?, operation: sending @escaping @isolated(any) () async throws -> Success) -> Task<Success, Error> {
        log.withLock { $0.append(.detachedTask(priority: priority)) }
        tasksCount.withLock { $0 += 1 }
        return Task.detached(priority: priority) { [weak self] in
            defer { self?.completedTasksCount.withLock { $0 += 1 } }
            do {
                let result = try await operation()
                return result
            } catch {
                throw error
            }
        }
    }

    public func waitForTasks() async {
        // Wait until all tasks have completed before proceeding.
        while completedTasksCount.withLock({ $0 }) < tasksCount.withLock({ $0 }) { await Task.yield() }
    }
}
```

Apple's [Mutex](https://developer.apple.com/documentation/synchronization/mutex) allows us to update our mock's state safely from multiple threads. However, it's not available on all OS versions. I've provided a backwards-compatible alternative to `Mutex` at the end of this article.

Here's a summary of what each task method does:
1. A `MethodCall` enum case corresponding to the task method is appended to the log to help verify the correct method is being called in testing
2. The number of tasks is incremented by 1 as a new task is about to be created to perform the asynchronous work submitted by our subject under test
3. The task is created and the asynchronous operation is executed within the task
4. Once the asynchronous operation is finished, we increment the number of completed tasks by one

#### Example usage

When writing asynchronous code, we declare a dependency on the `TaskProvider` protocol and await our asynchronous operations within a closure that we provide as an argument to the task method's `operation` parameter. Let's revisit our earlier example with `SomethingDoer` to see it in practice.

```swift
class SomethingDoer {

    var somethingHasBeenDone = false

    private let taskProvider: TaskProvider

    init(taskProvider: TaskProvider) {
        self.taskProvider = taskProvider
    }

    func doSomething() {
        print("task about to start executing")
        taskProvider.task(priority: .background) { [weak self] in
            print("task started executing")
            await self?.mutateState()
            print("task finished executing")
        }
        print("synchronous method `doSomething` about to return")
    }

    ...

}

class SomethingDoerTests: XCTestCase {
    func test_doSomething() async {
        let mock = TaskProviderMock()
        let sut = SomethingDoer(taskProvider: mock)
        print("about to call synchronous `doSomething` method")
        sut.doSomething()
        await mock.waitForTasks()
        print("about to assert")
        XCTAssertTrue(sut.somethingHasBeenDone) // test passes
        XCTAssertEqual(mock.log, [.task(priority: .background)])
        print("finished asserting")
    }
}
```

We inject instances of `TaskProviderImpl` and `TaskProviderMock` into our production and test instances of `SomethingDoer` respectively. This allows us to use Swift Concurrency as normal in our production code whilst ensuring that our asynchronous operations are finished before we begin our assertions within our tests.

After calling `doSomething()` in our test, we call `waitUntilTasks()` which prevents the test from progressing until our asynchronous operations are finished, thus providing reliable and deterministic test results. Here is the order of execution with this approach:

```
about to call synchronous `doSomething` method
task about to start executing
synchronous method `doSomething` about to return
task started executing
state about to be mutated
state mutated
task finished executing
about to assert
finished asserting
```

### Strengths and Weaknesses

While the examples in this article only show a single unstructured task being initialized within `doSomething()`, this mock is able to handle multiple unstructured tasks at a time. In addition, it is compliant with Swift 6's strict concurrency checks.

However, one limitation of this approach is the lack of a timeout mechanism, meaning `waitForTasks()` could hang indefinitely if a task never completes. In other words, `waitForTasks()` will wait for as long as it takes for the tasks to complete.

### Suggestions

Passing `nil` instead of the actual priority to the unstructured tasks within the mock can help improve the performance of the tests. Below is an example of how to do that:

```swift
public final class TaskProviderMock: TaskProvider, Sendable {

    ...

    public func task<Success: Sendable>(priority: TaskPriority?, operation: sending @escaping @isolated(any) () async -> Success) -> Task<Success, Never> {
        log.withLock { $0.append(.task(priority: priority)) }
        tasksCount.withLock { $0 += 1 }

        // passing `nil` instead of `priority` can improve the performance of tests
        return Task(priority: nil) { [weak self] in
            defer { self?.completedTasksCount.withLock { $0 += 1 } }
            let result = await operation()
            return result
        }
    }

    ...
}
```

While this may seem like it would lead to incorrect behavior, it shouldn't impact the results of our tests as we're only concerned about ensuring the task is finished before reaching our test assertions, not the priority of the task. In addition, if you want to ensure that the tasks in your production code are running with a specific priority, you can assert the value of `log` in your tests as it will contain correct priority.

## Mutex Alternatives

Apple's Mutex isn't supported on all OS versions, so you may want to create your own type which is backwards compatible. I'll provide two alternatives:
1. recreate the Mutex type
2. use [Grand Central Dispatch](https://developer.apple.com/documentation/DISPATCH) to synchronize access to integers and arrays

### LegacyMutex

If you cannot use Apple's Mutex, you can create your own:

```swift
import Foundation

public class LegacyMutex<Value: Sendable>: @unchecked Sendable {
    private var value: Value
    private let lock = NSLock()

    public init(_ value: Value) {
        self.value = value
    }

    public func withLock<Result>(_ body: (inout sending Value) throws -> Result) rethrows -> Result {
        lock.lock()
        defer { lock.unlock() }
        return try body(&value)
    }
}
```

Then you can simply update the properties in `TaskProviderMock` from `Mutex` to `LegacyMutex`.

Note: `NSLock` provides basic thread safety but does not support fairness mechanisms like `Mutex`.

### SynchronizedArray and SynchronizedValue

Alternatively, we can use dispatch queues to synchronize access to our `log`, `tasksCount` and `completedTasksCount`.

```swift
import Foundation

public final class SynchronizedArray<Element>: @unchecked Sendable {

    private var underlyingArray: Array<Element>
    private let queue = DispatchQueue(label: "com.testable-swift-concurrency.synchronizedArray", attributes: .concurrent)

    public init(_ underlyingArray: Array<Element> = []) {
        self.underlyingArray = underlyingArray
    }

    public var content: Array<Element> {
        get { queue.sync { underlyingArray } }
        set { queue.sync(flags: .barrier) { underlyingArray = newValue } }
    }

    public func append(_ element: Element) {
        queue.sync(flags: .barrier) {
            underlyingArray.append(element)
        }
    }
}

public final class SynchronizedValue<Value: Sendable>: @unchecked Sendable {
    private var underlyingValue: Value
    private let queue = DispatchQueue(label: "com.testable-swift-concurrency.synchronizedValue", attributes: .concurrent)

    public init(_ initialValue: Value) {
        underlyingValue = initialValue
    }

    public var value: Value {
        get { queue.sync { underlyingValue } }
        set { queue.sync(flags: .barrier) { underlyingValue = newValue } }
    }
}

public extension SynchronizedValue where Value == Int {
    func increment(by amount: Int = 1) {
        queue.sync(flags: .barrier) {
            underlyingValue += amount
        }
    }
}
```

The `TaskProviderMock` will need to be updated to make use of `SynchronizedArray` and `SynchronizedValue` since they are implemented differently to `Mutex` and `LegacyMutex`.

```swift
public final class TaskProviderMock: TaskProvider, Sendable {

    ...

    public let log = SynchronizedArray<MethodCall>([])
    private let completedTasksCount = SynchronizedValue(0)
    private let tasksCount = SynchronizedValue(0)

    ...

    public func task<Success: Sendable>(priority: TaskPriority?, operation: sending @escaping @isolated(any) () async -> Success) -> Task<Success, Never> {
        log.append(.task(priority: priority))
        tasksCount.increment()
        return Task(priority: priority) { [weak self] in
            defer { self?.completedTasksCount.increment() }
            let result = await operation()
            return result
        }
    }

    ...

    public func waitForTasks() async {
        while completedTasksCount.value < tasksCount.value { await Task.yield() }
    }
}
```

Here's what's changed between the previous mock and this one:
- The type for `log` has been changed from `Mutex<[MethodCall]>` to `SynchronizedArray<MethodCall>`
- The types for `tasksCount` and `completedTasksCount`  have been changed from `Mutex<Int>` to `SynchronizedValue<Int>`
- The task methods use `append` and `increment` instead of `withLock`
- `waitForTasks()` uses `value` to get the count instead of `withLock`