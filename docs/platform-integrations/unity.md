# R3 与 Unity 集成

R3 为 Unity 游戏引擎提供了强大而全面的集成，旨在帮助开发者在 Unity 项目中轻松应用响应式编程模式，以管理复杂的游戏逻辑、异步操作和事件流。本指南将详细介绍如何在 Unity 中安装、设置和有效使用 R3。

## 1. 安装和设置

在 Unity 项目中安装 R3 通常通过 Unity 包管理器 (UPM) 进行。

1.  打开您的 Unity 项目。
2.  导航到 `Window` > `Package Manager`。
3.  在 Package Manager 窗口的左上角，点击 `+` (加号) 图标。
4.  选择 `Add package from git URL...`。
5.  输入 R3 Unity 包的 Git URL。
    *   **注意**: 请始终从 R3 的官方 GitHub 仓库 ([https://github.com/Cysharp/R3](https://github.com/Cysharp/R3) - *请替换为实际的官方链接*) 查找最新的、推荐的安装 Git URL。它通常指向特定版本或包含 `package.json` 的子目录，例如 `https://github.com/Cysharp/R3.git?path=src/R3.Unity/Assets/R3.Unity`。
6.  点击 `Add`。Unity 将下载并导入 R3 包及其依赖项。

或者，如果 R3 发布在 OpenUPM (一个社区 Unity 包注册表) 上，您可以按照 OpenUPM 网站上的说明，通过修改项目的 `manifest.json` 文件或使用 OpenUPM-CLI 来安装。

安装完成后，R3 的命名空间 (主要是 `R3` 和 `R3.Triggers`) 就可以在您的 C# 脚本中使用了。

## 2. 与 UniRx 的详细比较

对于熟悉 UniRx (Unity Reactive Extensions) 的开发者来说，了解 R3 与 UniRx 的异同非常重要。两者都旨在为 Unity 带来响应式编程的强大功能，但在设计理念、特性集和实现细节上有所不同。

### 2.1. 相似之处

*   **核心 API 风格**: R3 的许多核心 Observable 操作符和事件转换 API 在命名和用法上与 UniRx 非常相似。例如，`Select`, `Where`, `Merge`, `Zip` 等操作符，以及将 Unity 事件转换为 Observable 的扩展方法如 `UpdateAsObservable()`, `OnCollisionEnterAsObservable()` 等，都具有相似的签名和行为。这使得从 UniRx 迁移或同时理解两者变得相对容易。
*   **基于 Trigger 的事件转换**: 两者都广泛使用附加到 `GameObject` 上的 MonoBehaviour 组件 (Triggers) 来捕获 Unity 的生命周期事件、物理事件、UI 事件等，并将其作为 Observable 序列暴露出来。
*   **`AddTo` 生命周期管理**: R3 同样提供了强大的 `AddTo(this IDisposable disposable, GameObject/Component target)` 扩展方法，用于将订阅的生命周期与 Unity 对象的生命周期绑定，自动处理取消订阅，防止内存泄漏。这是 UniRx 中非常受欢迎的一个特性。
*   **对 Unity 主线程的调度**: 两者都提供了将工作调度到 Unity 主线程特定执行阶段（如 Update, FixedUpdate, LateUpdate）的机制。

### 2.2. 不同之处

*   **性能和现代性**:
    *   R3 在设计上非常注重性能，可能会采用更现代的 C# 特性和优化技术，以减少内存分配和提高执行效率。
    *   R3 积极利用 Unity 新版本提供的特性，例如在 Unity 2022.2+ 中使用内置的 `MonoBehaviour.destroyCancellationToken` 来优化 `AddTo` 的实现，而在旧版本中则回退到使用 Trigger 组件。

*   **错误处理哲学**:
    *   R3 引入了 `OnErrorResume` 的概念（在 `Observer<T>` 基类层面），这可能允许在发生某些错误后，序列有条件地继续或执行恢复逻辑，而不是像传统 Rx 那样严格终止。
    *   R3 还提供了全局未处理异常处理器 (`ObservableSystem.GetUnhandledExceptionHandler()`)，有助于捕获和记录未被显式处理的错误。

*   **TimeProvider 和 FrameProvider**:
    *   R3 引入了更明确和细粒度的 `UnityFrameProvider` 和 `UnityTimeProvider` 抽象。
    *   `UnityFrameProvider` 为 PlayerLoop 的每个主要阶段（Initialization, EarlyUpdate, FixedUpdate, PreUpdate, Update, PreLateUpdate, PostLateUpdate, TimeUpdate）提供了独立的事件源。
    *   `UnityTimeProvider` 结合了 `UnityFrameProvider` 和 `TimeKind` (Time, UnscaledTime, Realtime)，允许开发者精确控制时间相关操作（如 `Timer`, `Delay`）所使用的 PlayerLoop 阶段和时间源（是否受 `Time.timeScale` 影响，或使用真实时间）。这比 UniRx 的 `MainThreadScheduler` 提供了更灵活和明确的控制。

*   **协程 (Coroutine) 转换**:
    *   根据目前的分析，R3 **似乎没有**内置与 UniRx 的 `Observable.FromCoroutine()` 或 `IEnumerator.ToObservable()` 直接对应的便捷方法来将 Unity 的传统协程转换为 Observable 序列。
    *   R3 可能更鼓励开发者使用基于 `System.Threading.Tasks.Task` 的异步模型（结合 `async/await` 和 `CancellationToken`，R3 提供了 `GetDestroyCancellationToken()` 扩展）或纯粹的 Observable 链来处理异步操作，而不是依赖协程转换。

*   **MessageBroker / EventBus**:
    *   根据目前的分析，R3 **似乎没有**内置一个与 UniRx 的 `MessageBroker` 或 `AsyncMessageBroker` 直接对应的全局事件总线实现。
    *   开发者如果需要此类功能，可能需要使用 R3 的 `Subject` 类型（如 `Subject<T>` 或 `ReplaySubject<T>`）自行构建，或者采用其他组件间通信模式。

*   **调试工具**:
    *   R3 提供了一个名为 `ObservableTrackerWindow` 的 Unity 编辑器窗口，用于可视化和跟踪活动中的 Observable 订阅、它们的创建位置（堆栈跟踪）以及事件流。这是一个强大的调试辅助工具，可能比 UniRx 提供的内置调试功能更为突出和集成。

*   **对 .NET 标准的遵循**:
    *   如果 R3 的 `TimeProvider` 和 `ITimer` 接口与 .NET 8 中引入的 `System.Threading.TimeProvider` 和 `System.Threading.ITimer` 兼容或基于其设计，那么 R3 在这方面可能更符合最新的 .NET 标准。

## 3. 在 MonoBehaviour 中使用 R3

R3 旨在与 Unity 的 `MonoBehaviour` 无缝协作。

### 3.1. 生命周期事件作为 Observables

您可以使用 `R3.Triggers` 命名空间下的扩展方法，轻松地将 `MonoBehaviour` 的标准生命周期事件转换为 Observable 序列：

```csharp
using UnityEngine;
using R3;
using R3.Triggers; // Important for trigger extensions

public class PlayerController : MonoBehaviour
{
    void Start()
    {
        // UpdateAsObservable: Emits Unit every frame during Update
        this.UpdateAsObservable()
            .Subscribe(_ => Debug.Log("Update! Frame: " + Time.frameCount))
            .AddTo(this); // Automatically disposes when this GameObject is destroyed

        // FixedUpdateAsObservable
        this.FixedUpdateAsObservable()
            .Subscribe(_ => Debug.Log("FixedUpdate!"))
            .AddTo(this);

        // LateUpdateAsObservable
        this.LateUpdateAsObservable()
            .Subscribe(_ => Debug.Log("LateUpdate!"))
            .AddTo(this);

        // OnDestroyAsObservable: Emits Unit when GameObject is being destroyed
        this.OnDestroyAsObservable()
            .Subscribe(_ => Debug.Log("GameObject is being destroyed!"))
            .AddTo(this); // Technically, AddTo(this) for OnDestroy is a bit redundant
                         // but good practice for other long-lived observables.
                         // For OnDestroy, the subscription naturally ends.

        // OnEnableAsObservable / OnDisableAsObservable
        this.OnEnableAsObservable()
            .Subscribe(_ => Debug.Log("Enabled!"))
            .AddTo(this);

        this.OnDisableAsObservable()
            .Subscribe(_ => Debug.Log("Disabled!"))
            .AddTo(this);
    }
}
```

### 3.2. 物理和碰撞事件

同样，物理事件也可以方便地转换为 Observables：

```csharp
using UnityEngine;
using R3;
using R3.Triggers;

public class CollisionDetector : MonoBehaviour
{
    void Start()
    {
        // OnCollisionEnterAsObservable
        this.OnCollisionEnterAsObservable()
            .Subscribe(collision => {
                Debug.Log($"Collided with: {collision.gameObject.name}");
                // Access collision.contacts, collision.impulse, etc.
            })
            .AddTo(this);

        // OnTriggerEnterAsObservable (requires a Collider set to IsTrigger=true)
        this.OnTriggerEnterAsObservable()
            .Subscribe(otherCollider => {
                Debug.Log($"Triggered by: {otherCollider.gameObject.name}");
            })
            .AddTo(this);

        // For 2D physics, use:
        // this.OnCollisionEnter2DAsObservable()
        // this.OnTriggerEnter2DAsObservable()
    }
}
```
R3 提供了 `OnCollisionStay/Exit`, `OnTriggerStay/Exit` 以及对应的 2D 版本。

### 3.3. UI 和输入事件

R3 也支持将 Unity UI 事件（如按钮点击）和输入事件转换为 Observables。

*   **`UnityUIComponentExtensions`**: (例如 `Button.OnClickAsObservable()`)
*   **`UnityEventExtensions`**: 通用的 `UnityEvent.AsObservable()`
*   **Pointer/Mouse Triggers**: `OnMouseDownAsObservable`, `OnPointerClickAsObservable` 等。

```csharp
using UnityEngine;
using UnityEngine.UI; // Required for UI components
using R3;
using R3.Triggers; // For some UI triggers if not on component directly

public class UIManager : MonoBehaviour
{
    public Button myButton;
    public EventTrigger eventTriggerComponent; // For more generic UI events

    void Start()
    {
        if (myButton != null)
        {
            // Button clicks
            myButton.OnClickAsObservable()
                .Subscribe(_ => Debug.Log("MyButton Clicked!"))
                .AddTo(this);
        }

        // Example with a generic EventTrigger component for PointerEnter
        if (eventTriggerComponent != null)
        {
            eventTriggerComponent.OnPointerEnterAsObservable()
                .Subscribe(pointerEventData => Debug.Log($"Pointer entered: {pointerEventData.pointerCurrentRaycast.gameObject.name}"))
                .AddTo(this);
        }
    }
}
```

## 4. 使用 `UnityFrameProvider` 和 `UnityTimeProvider`

R3 提供了对 Unity PlayerLoop 的精细控制，允许您将 Observable 操作调度到特定的执行阶段，并使用不同的时间源。

*   **`UnityFrameProvider`**: 代表 PlayerLoop 的不同阶段。
    *   `UnityFrameProvider.Update`
    *   `UnityFrameProvider.FixedUpdate`
    *   `UnityFrameProvider.LateUpdate`
    *   `UnityFrameProvider.EarlyUpdate`, `PreUpdate`, `PreLateUpdate`, `PostLateUpdate`, `TimeUpdate`, `Initialization`

*   **`UnityTimeProvider`**: 结合了 `UnityFrameProvider` 和 `TimeKind`。
    *   `TimeKind.Time`: 受 `Time.timeScale` 影响 (e.g., `UnityTimeProvider.Update`)。
    *   `TimeKind.UnscaledTime`: 不受 `Time.timeScale` 影响 (e.g., `UnityTimeProvider.UpdateIgnoreTimeScale`)。
    *   `TimeKind.Realtime`: 使用真实时间，不受 `timeScale` 和暂停影响 (e.g., `UnityTimeProvider.UpdateRealtime`)。

### 4.1. 调度操作 (`ObserveOn`, `SubscribeOn`)

```csharp
using UnityEngine;
using R3;
using System; // For TimeSpan
using System.Threading; // For Thread

public class SchedulerExample : MonoBehaviour
{
    void Start()
    {
        Debug.Log($"Start is on thread: {Thread.CurrentThread.ManagedThreadId}");

        Observable.Create<string>(observer =>
            {
                Debug.Log($"Observable.Create is on thread: {Thread.CurrentThread.ManagedThreadId}");
                // Simulate work on a background thread
                ThreadPool.QueueUserWorkItem(_ => {
                    Debug.Log($"Background work is on thread: {Thread.CurrentThread.ManagedThreadId}");
                    observer.OnNext("Data from background");
                    observer.OnCompleted();
                });
                return Disposable.Empty;
            })
            .SubscribeOn(TimeProvider.System) // Start subscription on a thread pool thread
            .ObserveOn(UnityTimeProvider.Update) // Switch to Unity's Update loop for notifications
            .Subscribe(
                value => Debug.Log($"Received '{value}' on thread: {Thread.CurrentThread.ManagedThreadId} during Update"),
                () => Debug.Log("Completed on Update loop")
            )
            .AddTo(this);
    }
}
```

### 4.2. 时间相关的操作符 (`Timer`, `Delay`, `Interval`)

```csharp
using UnityEngine;
using R3;
using System;

public class TimerExample : MonoBehaviour
{
    void Start()
    {
        // Timer that fires once after 2 seconds on the Update loop, using scaled time
        Observable.Timer(TimeSpan.FromSeconds(2), UnityTimeProvider.Update)
            .Subscribe(_ => Debug.Log("2 scaled seconds passed (Update loop)."))
            .AddTo(this);

        // Interval that fires every 1 second on FixedUpdate, ignoring timeScale
        // Note: R3's primary Timer can act as an Interval.
        // Or there might be a dedicated Observable.Interval.
        // Assuming Observable.Timer(dueTime, period, timeProvider) for interval:
        Observable.Timer(TimeSpan.Zero, TimeSpan.FromSeconds(1), UnityTimeProvider.FixedUpdateIgnoreTimeScale)
            .Subscribe(count => Debug.Log($"Unscaled Interval (FixedUpdate): {count}"))
            .AddTo(this);

        // Delaying a sequence
        Observable.Return("Delayed Message")
            .Delay(TimeSpan.FromSeconds(3), UnityTimeProvider.UpdateRealtime) // Delay by 3 real seconds
            .Subscribe(msg => Debug.Log(msg))
            .AddTo(this);
    }
}
```

## 5. `ObservableTrackerWindow` 使用指南

R3 提供了一个强大的编辑器工具 `ObservableTrackerWindow`，用于帮助开发者调试和理解项目中的响应式流。

1.  **打开窗口**: 在 Unity 编辑器中，通过 `Window` > `Observable Tracker` 打开。

2.  **功能**:
    *   **跟踪活动订阅**: 窗口会列出所有当前活动的 Observable 订阅。
    *   **来源信息**: 对于每个订阅，它可以显示创建该订阅的代码位置（堆栈跟踪）。这对于定位未正确释放的订阅或理解复杂流的起源非常有用。
    *   **事件计数**: 可能显示每个订阅发出的 `OnNext`, `OnError`, `OnCompleted` 事件的数量。
    *   **控制选项**:
        *   `Enable Tracking`: 启用/禁用对新订阅的跟踪。
        *   `Enable StackTrace`: 启用/禁用在订阅时捕获堆栈跟踪（可能会有性能开销，建议仅在调试时启用）。
        *   `Reload`: 手动刷新列表。
        *   `GC.Collect`: 手动触发垃圾回收，有助于观察订阅是否按预期被释放。

3.  **使用场景**:
    *   **检测内存泄漏**: 如果您怀疑某些订阅没有被正确 `Dispose`，`ObservableTrackerWindow` 可以帮助您找到这些“僵尸”订阅及其来源。
    *   **理解复杂流**: 当有多个 Observable 相互作用时，此窗口可以帮助您了解哪些流是活动的。
    *   **性能分析**: 通过观察订阅的创建频率和生命周期，可以间接了解某些响应式逻辑的性能影响。

使用 `ObservableTrackerWindow` 时，请注意启用堆栈跟踪可能会对性能产生影响，因此在性能分析或发布构建中应谨慎使用。

## 6. 示例和最佳实践

*   **始终管理订阅生命周期**: 使用 `AddTo(this)` 或 `AddTo(gameObject)` 是在 Unity 中最推荐的方式。对于非 MonoBehaviour 的类，使用 `CompositeDisposable`。
*   **选择合适的 `TimeProvider`**: 根据您的需求（是否受 `timeScale` 影响，执行时机）选择正确的 `UnityTimeProvider`。
*   **避免在 `OnNext` 中执行长时间操作**: 如果需要在 `OnNext` 中执行耗时操作，考虑使用 `ObserveOn` 将其切换到后台线程，完成后再切换回主线程更新 UI 或游戏状态。
*   **利用操作符组合复杂逻辑**: R3 的强大之处在于其操作符。学习并熟练使用它们来代替手动编写复杂的状态机和回调逻辑。
*   **保持流的清晰**: 避免创建过于复杂或嵌套过深的 Observable 链。如果逻辑变得难以理解，考虑将其分解为更小的、可重用的 Observable 方法或自定义操作符。
*   **谨慎使用 Hot Observables 和 Subjects**: 确保您理解它们的多播行为和潜在的副作用。
*   **利用 `ObservableTrackerWindow` 进行调试**: 在开发过程中定期检查活动的订阅，确保一切按预期工作。

(后续可以补充更具体的代码示例和场景应用)
