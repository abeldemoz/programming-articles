# Testable Swift Concurrency

I've written this article to address the challenges related to testing code that leverages Swift Concurrency.

## Problem

In Swift Concurrency, tasks are fundamental units of work that can be executed concurrently. They represent asynchronous operations that can be run in parallel with other tasks. Tasks allow you to create, manage, and execute code concurrently in a structured way.

However, when initialising an unstructured task within a synchronous method, the task will outlive the method. This is because the task is initialised with an escaping closure. Let's consider the following example:

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

The synchronous `doSomething()` method initialises a task that mutates the class's internal state. Print statements have been interspersed throughout the `doSomething()` and `test_doSomething()` methods in order to illustrate the order of execution. Below is an example of the order of execution when running the test:
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

Every time we run the test method, the order of execution could differ from the previous test run. In some cases, the internal state can be mutated before the test begins evaluating the value of the property. Because of the unpredictable order of execution, we get flaky tests.

The reason behind the unpredictable order of execution between test runs is that the task is initialised with an escapig closure and we have no control over when that closure begins or finishes executing.

### Solution

The next logical step is to ensure the asynchronous operation finishes before we begin evaluating/asserting our state. Once the asynchronous operation has finished, we can evaluate our state deterministically. The remainder of this article will be dedicated to exploring a solution that allows us to achieve that.

The solution leverages dependency inversion and abstracts the `Task` type behind a protocol that can be mocked for testing purposes. As such, parts of your code that use unstructured tasks will depend on the protocol. Here is the protocol:

```swift
public protocol TaskProvider: Sendable {

    @discardableResult
    func task<Success: Sendable>(priority: TaskPriority?, operation: sending @escaping () async -> Success) -> Task<Success, Never>

    @discardableResult
    func task<Success: Sendable>(priority: TaskPriority?, operation: sending @escaping () async throws -> Success) -> Task<Success, Error>

    @discardableResult
    func detachedTask<Success: Sendable>(priority: TaskPriority?, operation: sending @escaping () async -> Success) -> Task<Success, Never>

    @discardableResult
    func detachedTask<Success: Sendable>(priority: TaskPriority?, operation: sending @escaping () async throws -> Success) -> Task<Success, Error>
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
    public func task<Success: Sendable>(priority: TaskPriority?, operation: sending @escaping () async -> Success) -> Task<Success, Never> {
        Task(priority: priority, operation: operation)
    }

    @discardableResult
    public func task<Success: Sendable>(priority: TaskPriority?, operation: sending @escaping () async throws -> Success) -> Task<Success, Error> {
        Task(priority: priority, operation: operation)
    }

    @discardableResult
    public func detachedTask<Success: Sendable>(priority: TaskPriority?, operation: sending @escaping () async -> Success) -> Task<Success, Never> {
        Task.detached(priority: priority, operation: operation)
    }

    @discardableResult
    public func detachedTask<Success: Sendable>(priority: TaskPriority?, operation: sending @escaping () async throws -> Success) -> Task<Success, Error> {
        Task.detached(priority: priority, operation: operation)
    }
}
```

Each method initialises and returns the appropriate/corresponding task to the caller of the method. In other words, rather than initialising an unstructured task ourselves, we can call the methods instead.

#### Mock implementation

The mock implementation is slightly more complex than the production implementation. This is because the mock has two main objectives:
1. to keep track of both the number of unstructured tasks that have been initialised and finished executing
2. to wait until all of the created tasks have finished executing

To achieve the first objective, we create properties that store the number of initialised and completed tasks as integers. The values of these properties will be incremented accordingly.

For the second objective, we compare the number of initialised tasks to the number of completed tasks. If the number of intialised tasks is greater than the number completed tasks, that means there is at least one task in progress. Once those two numbers are equal, all tasks have completed and we no longer need to wait.

```swift
import Synchronization

public final class TaskProviderMock: TaskProvider, Sendable {

    public enum MethodCall: Equatable {
        case task(priority: TaskPriority?)
        case detachedTask(priority: TaskPriority?)
    }

    public let log = Mutex<[MethodCall]>([])
    private let completedTasksCount = Mutex(0)
    private let tasksCount = Mutex(0)

    public init() {}

    public func task<Success: Sendable>(priority: TaskPriority?, operation: sending @escaping () async -> Success) -> Task<Success, Never> {
        log.withLock { $0.append(.task(priority: priority)) }
        tasksCount.withLock { $0 += 1 }
        return Task(priority: priority) { [weak self] in
            defer { self?.completedTasksCount.withLock { $0 += 1 } }
            let result = await operation()
            return result
        }
    }

    public func task<Success: Sendable>(priority: TaskPriority?, operation: sending @escaping () async throws -> Success) -> Task<Success, Error> {
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

    public func detachedTask<Success: Sendable>(priority: TaskPriority?, operation: sending @escaping () async -> Success) -> Task<Success, Never> {
        log.withLock { $0.append(.detachedTask(priority: priority)) }
        tasksCount.withLock { $0 += 1 }
        return Task.detached(priority: priority) { [weak self] in
            defer { self?.completedTasksCount.withLock { $0 += 1 } }
            let result = await operation()
            return result
        }
    }

    public func detachedTask<Success: Sendable>(priority: TaskPriority?, operation: sending @escaping () async throws -> Success) -> Task<Success, Error> {
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
        while completedTasksCount.withLock({ $0 }) < tasksCount.withLock({ $0 }) { await Task.yield() }
    }
}
```

The [Mutex](https://developer.apple.com/documentation/synchronization/mutex) type allows us to update our mock's state safely from multiple threads. However, it's not available on all OS versions. I've provided a backwards-compatible alternative to `Mutex` at the end of this article.

Here's a summary of what each task method does:
1. A method call enum case corresponding to the task method is appended to the log to help verify the correct method is being called in testing
2. The number of tasks is incremented by 1 as a new task is about to be created to perform the asynchronous work submitted by our subject under test
3. The task is created and the asynchronous operation is executed within the task
4. Once the asynchronous operation is finished, we increment the number of completed tasks by one

#### Example usage

When writing asynchronous code, we declare a dependency on the `TaskProvider` protocol and await our asynchronous operations within a closure that we provide as an argument to the task method's `operation` parameter. Let's revisit our earlier example with the `SomethingDoer` class to see it in practice.

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
        XCTAssertTrue(sut.somethingHasBeenDone) // test fails
        print("finished asserting")
    }
}
```

We inject `TaskProvider` and `TaskProviderMock` into our production and test instances of `SomethingDoer` respectively. This allows us to use Swift Concurrency as normal in our production code whilst ensuring that our asynchronous operations are finished before we begin our assertions within our tests.

After calling the `doSomething()` method in our test, we call the `waitUntilTasks()` method which prevents the test from progressing until our asynchronous operations are finished, thus providing reliable and deterministic test results. Here is the order of execution with this approach:

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

### Benefits and Drawbacks

While the examples in this article only show a single unstructured task being initialised within our `doSomething` method, this mock is able to handle multiple unstructured tasks at a time. In addition, it is compliant with Swift 6's strict concurrency checks.

However, a limitation of this mock is the lack of timeout functionality. The `waitForTasks` method will wait for as long as it takes for the tasks to complete. 

## Mutex alternative

