# 核心概念详解

本章节将深入探讨构成 R3 库的基石的核心概念。理解这些概念对于有效利用响应式编程的强大功能至关重要。

## 1. Observables (可观察序列)

Observable (`Observable<T>`) 是 R3 中最基本的构建块。它代表一个可以随时间推移异步地产生一系列类型为 `T` 的值的序列。这些值可以是任何东西：用户输入事件、HTTP 响应、计时器事件、数据更新等等。

### 创建 Observables

R3 提供了多种静态工厂方法来创建 Observable 实例，以适应不同的场景：

*   **`Observable.Return<T>(T value)`**: 创建一个只发出单个 `value` 然后立即完成的 Observable。
    ```csharp
    Observable<string> name = Observable.Return("Alice");
    ```

*   **`Observable.Range(int start, int count)`**: 创建一个发出从 `start` 开始的 `count` 个连续整数的 Observable，然后完成。
    ```csharp
    Observable<int> numbers = Observable.Range(1, 5); // Emits 1, 2, 3, 4, 5
    ```

*   **`Observable.Timer(TimeSpan dueTime)`**: 创建一个在指定的 `dueTime` 之后发出一个 `Unit` 值 (代表一个信号) 然后完成的 Observable。
    ```csharp
    // Emits a Unit value after 3 seconds, then completes.
    Observable<Unit> timer = Observable.Timer(TimeSpan.FromSeconds(3));
    ```

*   **`Observable.Timer(TimeSpan dueTime, TimeSpan period, TimeProvider timeProvider)`**: 创建一个在指定的 `dueTime` 之后开始，然后以指定的 `period` 周期性地发出从 0 开始递增的 `long` 值的 Observable。需要指定一个 `TimeProvider`。
    ```csharp
    // In Unity, using Update TimeProvider
    // Starts after 1 second, then emits 0, 1, 2... every 2 seconds on the Update loop.
    Observable<long> interval = Observable.Timer(
        TimeSpan.FromSeconds(1),
        TimeSpan.FromSeconds(2),
        R3.UnityTimeProvider.Update);
    ```
    *注意：R3 中更通用的 `Interval` 功能可能通过 `Observable.Timer` 的周期性版本或专门的 `Observable.Interval(TimeSpan period, TimeProvider timeProvider)` 实现。请查阅具体 API。*

*   **`Observable.Create<T>(Func<Observer<T>, IDisposable> subscribe)`**: 这是创建自定义 Observable 最灵活的方式。您提供一个函数，该函数接收一个 `Observer<T>`，并负责调用其 `OnNext`, `OnError`, `OnCompleted` 方法。该函数必须返回一个 `IDisposable`，用于在取消订阅时执行清理逻辑。
    ```csharp
    Observable<int> customSequence = Observable.Create<int>(observer =>
    {
        observer.OnNext(10);
        observer.OnNext(20);
        observer.OnCompleted();
        return Disposable.Empty; // Or a custom disposable for cleanup
    });
    ```

*   **`Observable.Defer<T>(Func<Observable<T>> observableFactory)`**: 创建一个 Observable，直到有观察者订阅它时，才会调用 `observableFactory` 函数来创建实际的 Observable。这对于延迟昂贵操作或确保每个订阅者获得独立的序列非常有用。

*   **`Observable.FromEvent` 模式**: R3 提供了将标准 .NET 事件转换为 Observable 序列的方法。例如 `Observable.FromEventHandler` 或特定于平台的事件转换（如 Unity 中的 `UnityEventExtensions`）。
    ```csharp
    // Example: Converting a standard C# event
    // public event Action<string> MyCustomEvent;
    // Observable<string> eventAsObservable = Observable.FromEvent<string>(
    //    handler => MyCustomEvent += handler,
    //    handler => MyCustomEvent -= handler);
    ```

*   **`Observable.Empty<T>()`**: 创建一个不发出任何值就立即完成的 Observable。
*   **`Observable.Never<T>()`**: 创建一个不发出任何值也从不完成（或出错）的 Observable。
*   **`Observable.Throw<T>(Exception exception)`**: 创建一个不发出任何值，而是立即以指定的 `exception` 终止的 Observable。

### Hot vs Cold Observables

理解 Hot 和 Cold Observables 的区别对于正确使用响应式编程非常重要：

*   **Cold Observables**:
    *   为每个订阅者启动一个新的、独立的执行。
    *   只有当有 Observer 订阅时，它们才开始发出值。
    *   每个订阅者都会收到完整的序列，从头开始。
    *   例子：`Observable.Return`, `Observable.Range`, `Observable.Create`（如果其工厂函数为每个订阅者创建新资源）。HTTP 请求的 Observable 通常也是 Cold 的。

*   **Hot Observables**:
    *   无论是否有订阅者，它们都可能正在发出值。
    *   订阅者在订阅时开始接收值，但可能会错过在订阅之前已经发出的值。
    *   所有订阅者共享同一个 Observable 源。
    *   例子：鼠标移动事件的 Observable、股票价格的实时流。`Subject` 的实例通常表现为 Hot Observables。

您可以使用如 `Publish()` 和 `RefCount()` 或 `Share()` 等操作符将 Cold Observable 转换为 Hot Observable，或者更精确地控制其共享行为。

## 2. Subjects (主体)

Subject 是一种特殊的类型，它既是 `Observable<T>` 也是 `Observer<T>`。这意味着它可以被订阅，也可以调用 `OnNext`, `OnError`, `OnCompleted` 方法来向其订阅者推送值。Subjects 对于将非响应式代码桥接到响应式世界，或者在多个订阅者之间多播值非常有用。

R3 提供了几种主要的 Subject 类型：

*   **`Subject<T>`**:
    *   最基本的 Subject 类型。
    *   它简单地将从其源（或通过直接调用其 `OnNext` 方法）接收到的值多播给所有当前的订阅者。
    *   如果一个值在某个 Observer 订阅之前被发出，该 Observer 将不会收到那个值。

    ```csharp
    var subject = new Subject<string>();

    subject.Subscribe(x => Console.WriteLine($"Observer A: {x}"));
    subject.OnNext("Hello"); // Observer A: Hello

    subject.Subscribe(x => Console.WriteLine($"Observer B: {x}"));
    subject.OnNext("World"); // Observer A: World, Observer B: World

    subject.OnCompleted();
    ```

*   **`BehaviorSubject<T>`**:
    *   需要一个初始值。
    *   它会缓存最新的值。当有新的 Observer 订阅时，它会立即向该 Observer 发出这个最新的（或初始的）值，然后继续发出后续的值。
    *   如果序列已经完成或出错，新的订阅者将收到相应的完成或错误通知。
    *   可以通过 `Value` 属性同步获取其当前最新的值。

    ```csharp
    var behaviorSubject = new BehaviorSubject<int>(0); // Initial value is 0

    behaviorSubject.Subscribe(x => Console.WriteLine($"Observer 1: {x}")); // Observer 1: 0
    behaviorSubject.OnNext(1); // Observer 1: 1
    behaviorSubject.OnNext(2); // Observer 1: 2

    Console.WriteLine($"Current value: {behaviorSubject.Value}"); // Current value: 2

    behaviorSubject.Subscribe(x => Console.WriteLine($"Observer 2: {x}")); // Observer 2: 2 (gets the latest value)
    behaviorSubject.OnNext(3); // Observer 1: 3, Observer 2: 3
    ```

*   **`ReplaySubject<T>`**:
    *   缓存一定数量的最近发出的值（或者在某个时间窗口内的值）。
    *   当有新的 Observer 订阅时，它会将缓存的这些值“重放”给该 Observer，然后继续发出后续的值。
    *   如果序列已经完成或出错，它仍然会先重放缓存的值，然后再发送完成或错误通知。

    ```csharp
    var replaySubject = new ReplaySubject<string>(2); // Buffer size of 2

    replaySubject.OnNext("A");
    replaySubject.OnNext("B");
    replaySubject.OnNext("C"); // "A" is pushed out of buffer

    replaySubject.Subscribe(x => Console.WriteLine($"Observer X: {x}"));
    // Observer X: B
    // Observer X: C

    replaySubject.OnNext("D"); // Observer X: D
    ```

*   **`AsyncSubject<T>`**:
    *   只发出其源 Observable 序列的最后一个值，并且只有在源序列完成时才发出。
    *   如果源序列在发出任何值之前就完成了，`AsyncSubject` 也会完成而不发出任何值。
    *   如果源序列出错，`AsyncSubject` 将不会发出任何值，而是将错误通知传递给其订阅者。
    *   常用于表示一个只关心最终结果的异步操作。

    ```csharp
    var asyncSubject = new AsyncSubject<int>();

    asyncSubject.Subscribe(
        x => Console.WriteLine($"Value: {x}"),
        () => Console.WriteLine("Completed")
    );

    asyncSubject.OnNext(1);
    asyncSubject.OnNext(2);
    asyncSubject.OnNext(3); // These are cached, not emitted yet

    asyncSubject.OnCompleted(); // Now, the last value (3) is emitted, then completed.
    // Output:
    // Value: 3
    // Completed
    ```

## 3. Operators (操作符)

操作符是 Observable 的扩展方法，它们是响应式编程的核心和灵魂。操作符允许您以声明式的方式对 Observable 序列进行各种复杂的处理，如转换、过滤、合并、错误处理等，而无需编写嵌套的回调或手动管理状态。

每个操作符都接收一个源 Observable，并返回一个新的 Observable，而不会修改源 Observable（这称为“不可变性”）。这使得操作符可以被链接起来，形成强大的数据处理管道。

R3 提供了丰富的操作符集合，大致可以分为以下几类：

*   **创建型 (Creation)**: 如 `Return`, `Range`, `Timer`, `Interval`, `Create`, `Defer`, `Empty`, `Never`, `Throw` (已在上面“创建 Observables”部分介绍)。
*   **转换型 (Transformation)**: 修改 Observable 发出的值。
    *   `Select(Func<TSource, TResult> selector)`: 将每个元素投影到一个新形式。
    *   `SelectMany(Func<TSource, Observable<TResult>> selector)`: 将每个元素投影到一个 Observable 序列，然后将结果扁平化为一个序列。也称为 `FlatMap`。
    *   `Cast<TResult>()`, `OfType<TResult>()`
*   **过滤型 (Filtering)**: 根据条件选择或跳过元素。
    *   `Where(Func<TSource, bool> predicate)`: 只发出满足指定条件的元素。
    *   `Distinct()`, `DistinctUntilChanged()`
    *   `First()`, `Last()`, `Single()`
    *   `Skip(int count)`, `Take(int count)`
    *   `SkipWhile()`, `TakeWhile()`, `SkipUntil()`, `TakeUntil()`
    *   `Debounce()`, `ThrottleFirst()`, `ThrottleLast()`
*   **组合型 (Combining)**: 将多个 Observable 合并为一个。
    *   `Merge(Observable<T> other)`: 将多个 Observable 的输出合并到一个序列中，按它们发出的时间顺序。
    *   `Concat(Observable<T> other)`: 按顺序连接多个 Observable，只有当前一个完成后，下一个才开始发出值。
    *   `Zip(Observable<TSecond> second, Func<TFirst, TSecond, TResult> resultSelector)`: 将多个 Observable 的对应元素配对组合。
    *   `CombineLatest(Observable<TSecond> second, Func<TFirst, TSecond, TResult> resultSelector)`: 当任何一个源 Observable 发出新值时，结合所有源的最新值。
    *   `Switch()`: 将高阶 Observable（发出 Observable 的 Observable）转换为一阶 Observable，只订阅并转发最新的内部 Observable 发出的值。
*   **错误处理型 (Error Handling)**:
    *   `Catch<TSource, TException>(Func<TException, Observable<TSource>> handler)`: 捕获特定类型的异常，并用另一个 Observable 序列替换出错的序列。
    *   `Retry(int retryCount)`: 在发生错误时，重新订阅源 Observable 指定的次数。
    *   `OnErrorResumeNext(Observable<TSource> second)`: 当源序列出错时，无缝切换到另一个序列。
*   **工具型 (Utility)**:
    *   `Do(Action<T> onNext, Action<Exception> onError, Action onCompleted)`: 在序列的生命周期事件（OnNext, OnError, OnCompleted）发生时执行副作用操作，通常用于日志记录或调试。也称为 `Tap`。
    *   `ObserveOn(TimeProvider timeProvider)`: 指定观察者在其上接收通知的 `TimeProvider`（例如，切换到 UI 线程）。
    *   `SubscribeOn(TimeProvider timeProvider)`: 指定 Observable 在其上执行其订阅逻辑（包括操作符链的执行）的 `TimeProvider`。
    *   `Delay(TimeSpan dueTime, TimeProvider timeProvider)`: 延迟整个序列的发出（或每个元素的发出，取决于具体实现）。
    *   `Timeout(TimeSpan dueTime, TimeProvider timeProvider)`: 如果源 Observable 在指定时间内没有发出任何值，则以超时错误终止。
*   **聚合型 (Aggregate)**:
    *   `Count()`, `Sum()`, `Min()`, `Max()`, `Average()`
    *   `Aggregate(Func<TSource, TSource, TSource> func)`: 对序列应用累加器函数。
    *   `Scan(Func<TSource, TSource, TSource> func)`: 类似于 `Aggregate`，但发出每个中间累加结果。
*   **条件型和布尔型 (Conditional and Boolean)**:
    *   `All(Func<TSource, bool> predicate)`
    *   `Any(Func<TSource, bool> predicate)`
    *   `Contains(TSource value)`
    *   `SequenceEqual(Observable<TSource> second)`
*   **多播型 (Multicasting)**: 用于将 Cold Observable 转换为 Hot Observable 并共享订阅。
    *   `Publish()`, `RefCount()`, `Share()`
    *   `Replay()` (与 `ReplaySubject` 相关)

详细的操作符列表及其用法，请参阅 [API 参考 - 操作符](../api-reference/operators.md) (链接待实际文件创建后更新)。

## 4. Schedulers / TimeProviders (调度器 / 时间提供者)

在响应式编程中，特别是在涉及多线程或特定执行上下文（如 UI 线程、Unity 的游戏循环）的环境中，控制代码在何处以及何时执行至关重要。R3 使用 `TimeProvider` (可能继承自 .NET 8 的 `System.Threading.TimeProvider` 或具有类似概念的 R3 自定义抽象) 来实现这一目标。

`TimeProvider` 负责两件事：
1.  **提供时间概念**: 它定义了如何获取当前时间戳 (`GetTimestamp()`) 以及如何创建定时器 (`CreateTimer()`)。这对于实现 `Delay`, `Timer`, `Interval`, `Timeout` 等时间相关的操作符至关重要。
2.  **提供执行上下文**: 它决定了 Observable 的订阅逻辑和观察者的通知应该在哪个“线程”或“循环”上执行。

### 关键操作符：`ObserveOn` 和 `SubscribeOn`

*   **`ObserveOn(TimeProvider timeProvider)`**:
    *   指定其下游的观察者（即 `ObserveOn` 之后的操作符和最终的 `Subscribe` 回调）在哪个 `TimeProvider` 所代表的上下文中执行。
    *   例如，如果您从后台线程获取数据，但需要在 UI 线程上更新 UI，您可以使用 `ObserveOn(uiThreadTimeProvider)` 来切换上下文。

*   **`SubscribeOn(TimeProvider timeProvider)`**:
    *   指定源 Observable（即 `SubscribeOn` 上游的操作符链，直到实际的 Observable 创建逻辑）在哪个 `TimeProvider` 所代表的上下文中执行其 `SubscribeCore` 方法（即订阅的启动和资源分配）。
    *   这对于将耗时的或阻塞的订阅启动过程移出敏感线程（如 UI 线程）非常有用。

### R3 中的 TimeProviders

*   **通用 TimeProviders**: R3 核心库可能提供一些通用的 `TimeProvider` 实现，例如：
    *   `TimeProvider.System` (或类似名称): 代表标准的系统时间，可能使用线程池进行调度。
    *   `ImmediateTimeProvider`: 在当前线程上立即执行。
    *   `CurrentThreadTimeProvider`: 将工作安排在当前线程的队列中。

*   **特定平台的 TimeProviders**: R3 为其支持的平台提供了专门的 `TimeProvider` 实现，以与平台的主循环或 UI 线程集成。
    *   **Unity 中的 `UnityTimeProvider` 和 `UnityFrameProvider`**:
        *   [`UnityFrameProvider`](../src/R3.Unity/Assets/R3.Unity/Runtime/UnityFrameProvider.cs:1) 提供了与 Unity PlayerLoop 不同阶段（`Update`, `FixedUpdate`, `LateUpdate` 等）对应的帧更新事件源。
        *   [`UnityTimeProvider`](../src/R3.Unity/Assets/R3.Unity/Runtime/UnityTimeProvider.cs:1) 结合了 `UnityFrameProvider` 和 `TimeKind` (Time, UnscaledTime, Realtime)，允许将工作调度到特定的 PlayerLoop 阶段，并使用特定的时间源。
        *   这使得在 Unity 中可以非常精确地控制响应式逻辑的执行时机和时间行为。例如：
            ```csharp
            Observable.Timer(TimeSpan.FromSeconds(1), R3.UnityTimeProvider.Update)
                .ObserveOn(R3.UnityTimeProvider.Update) // Ensure OnNext is on Update loop
                .Subscribe(x => {
                    // This code will run on Unity's main thread during the Update phase
                    Debug.Log("Timer ticked on Update loop!");
                })
                .AddTo(this); // Assuming 'this' is a MonoBehaviour
            ```

## 5. Disposables (可释放对象)

当您订阅一个 Observable 时，会返回一个 `IDisposable` 对象。这个对象代表了该订阅。当您不再需要接收来自该 Observable 的通知时，或者当相关的上下文（如一个 UI 元素或一个游戏对象）被销毁时，**必须**调用这个 `IDisposable` 对象的 `Dispose()` 方法。

如果不这样做，可能会导致：
*   **内存泄漏**: Observer 和 Observable 之间可能存在引用，导致它们无法被垃圾回收。
*   **不必要的计算**: Observable 可能仍在后台执行工作并尝试向已无效的 Observer 推送值。
*   **意外行为**: 已销毁的对象可能仍然尝试响应事件。

### 管理 Disposables

*   **`IDisposable`**: .NET 中的标准接口，用于表示可以释放非托管资源或执行清理操作的对象。
    ```csharp
    IDisposable subscription = myObservable.Subscribe(...);
    // ...later...
    subscription.Dispose();
    ```

*   **`CompositeDisposable`**:
    *   一个实现了 `IDisposable` 的集合，可以容纳多个其他的 `IDisposable` 对象。
    *   当 `CompositeDisposable` 本身被 `Dispose` 时，它会 `Dispose` 其包含的所有 `IDisposable` 对象。
    *   非常适用于管理一个对象（如 ViewModel 或 MonoBehaviour）生命周期内的多个订阅。
    ```csharp
    private CompositeDisposable disposables = new CompositeDisposable();

    void StartWork()
    {
        var sub1 = observable1.Subscribe(...);
        var sub2 = observable2.Subscribe(...);

        disposables.Add(sub1);
        disposables.Add(sub2);
    }

    void Cleanup()
    {
        disposables.Dispose(); // Disposes sub1 and sub2
    }
    ```

*   **`Disposable.Create(Action disposeAction)`**:
    *   一个方便的静态方法，用于创建一个 `IDisposable` 对象，其 `Dispose()` 方法会执行您提供的 `disposeAction`。
    *   常用于 `Observable.Create` 中返回一个执行自定义清理逻辑的 `IDisposable`。

*   **R3 扩展方法 `AddTo(this IDisposable disposable, ICollection<IDisposable> container)`**:
    *   R3 提供了一个便捷的扩展方法 `AddTo`，可以将一个 `IDisposable` 直接添加到一个实现了 `ICollection<IDisposable>` 的容器中（如 `CompositeDisposable` 或 `List<IDisposable>`）。
    ```csharp
    observable1.Subscribe(...).AddTo(disposables);
    observable2.Subscribe(...).AddTo(disposables);
    ```

*   **Unity 中的 `AddTo(this IDisposable disposable, GameObject/Component target)`**:
    *   这是 R3 在 Unity 中进行资源管理的关键特性。它将订阅的生命周期与指定的 `GameObject` 或 `Component` 的生命周期绑定。当目标对象被销毁时，订阅会自动被 `Dispose`。
    *   极大地简化了 Unity 中的订阅管理，有效防止了常见的内存泄漏问题。
    ```csharp
    // In a MonoBehaviour
    void Start()
    {
        Observable.EveryUpdate().Subscribe(_ => {}).AddTo(this);
    }
    ```

## 6. 错误处理机制详解

R3 提供了比传统 `try-catch` 更强大和灵活的错误处理机制，这些机制是响应式编程的核心优势之一。

*   **`OnError` 通知**: 当 Observable 序列内部发生未被捕获的异常时，它会停止发出 `OnNext` 通知，并向其 Observer 发送一个包含该异常的 `OnError` 通知。之后，该序列被视为已终止。

*   **`Subscribe` 中的 `onError` 回调**:
    ```csharp
    source.Subscribe(
        onNext: value => { /* ... */ },
        onError: ex => {
            // Log the error, update UI to show error state, etc.
            Console.WriteLine($"An error occurred: {ex.Message}");
        },
        onCompleted: () => { /* ... */ }
    );
    ```

*   **`Catch` 操作符**:
    *   允许您在错误发生时拦截错误，并用另一个 Observable 序列替换出错的序列。
    *   `Catch<TSource, TException>(Func<TException, Observable<TSource>> handler)`: 只捕获特定类型的异常。
    *   `Catch(Func<Exception, Observable<TSource>> handler)`: 捕获所有异常。
    *   `Catch(Observable<TSource> second)`: 当第一个序列出错时，无缝切换到第二个序列。
    ```csharp
    riskyObservable
        .Catch((MySpecificException ex) => Observable.Return(defaultValueOnError)) // Handle specific exception
        .Catch((Exception ex) => { // Handle any other exception
            LogToServer(ex);
            return Observable.Empty<int>(); // Or rethrow, or return a fallback
        })
        .Subscribe(...);
    ```

*   **`Retry` 操作符**:
    *   当源 Observable 出错时，自动重新订阅它，尝试再次执行。
    *   可以指定重试次数：`Retry(3)` (重试3次)。
    *   可以不带参数，无限次重试（慎用！）。
    *   可以提供一个 `Func<Exception, int, bool>` 来决定是否基于异常类型和重试次数进行重试。
    ```csharp
    networkRequestObservable
        .Retry(3) // Try up to 3 times on failure
        .Subscribe(...);
    ```

*   **`OnErrorResumeNext(Observable<TSource> next)`**:
    *   如果源 Observable 成功完成，则其行为与源 Observable 一致。
    *   如果源 Observable 因错误而终止，`OnErrorResumeNext` 会忽略该错误，并订阅 `next` Observable 序列，继续从 `next` 序列发出值。
    *   与 `Catch` 不同，`OnErrorResumeNext` 不会给您机会检查错误或根据错误类型执行不同逻辑；它只是简单地切换到下一个序列。

*   **`Finally(Action finallyAction)`**:
    *   注册一个在源 Observable 序列终止时（无论是正常完成 `OnCompleted` 还是因错误 `OnError`）都会执行的动作。
    *   常用于执行资源清理等操作，无论序列如何结束。
    ```csharp
    myObservable
        .Do(x => Console.WriteLine(x))
        .Finally(() => Console.WriteLine("Sequence terminated."))
        .Subscribe(...);
    ```

*   **R3 特有的 `OnErrorResume(Exception error)` (在 Observer 层面)**:
    *   如之前在 [`Observable.cs`](../src/R3/Observable.cs:1) 中所见，R3 的 `Observer<T>` 基类有一个 `OnErrorResumeCore(Exception error)` 方法。这表明 R3 可能在 Observer 层面提供了一种错误恢复或继续的机制，允许 Observer 在接收到错误后，不是简单地终止，而是有机会处理错误并可能继续接收（或忽略）后续的 `OnNext` 调用（具体行为取决于 `Observer` 的实现）。这与传统 Rx 中 `OnError` 通常意味着序列严格终止有所不同，可能为某些场景提供了更大的灵活性。

*   **全局未处理异常处理器 (`ObservableSystem.GetUnhandledExceptionHandler()`)**:
    *   如果错误在 Observable 链中一直未被处理（即没有 `onError` 回调或 `Catch` 操作符捕获它），R3 会尝试通过一个全局的未处理异常处理器来报告这个错误。
    *   这有助于防止错误被“吞噬”而未被察觉，开发者可以配置这个处理器来进行日志记录或其他全局错误处理。

理解并善用这些错误处理机制，可以帮助您构建出更加健壮和用户友好的响应式应用程序。
