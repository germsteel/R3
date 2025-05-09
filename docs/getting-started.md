# 入门指南

本指南将引导您完成 R3 库的基本设置和使用，帮助您快速上手响应式编程。

## 1. 安装

根据您的项目类型，R3 的安装方式有所不同。

### 对于标准 .NET 项目 (非 Unity)

您可以通过 NuGet 包管理器来安装 R3 核心库。在您的项目（例如，控制台应用、ASP.NET Core 应用、WPF/WinForms 应用等）中，执行以下操作：

*   **使用 .NET CLI:**
    ```bash
    dotnet add package R3
    ```
*   **使用 NuGet 包管理器控制台 (Visual Studio):**
    ```powershell
    Install-Package R3
    ```
*   **通过 Visual Studio NuGet UI:**
    搜索 "R3" 并安装由 "Cysharp" 发布的包。

如果您需要针对特定 UI 框架（如 WPF, WinForms, Blazor, Uno, WinUI3）的集成（例如，特定的 `TimeProvider`），请查找并安装相应的 `R3.[FrameworkName]` 包，例如 `R3.WPF`。

### 对于 Unity 项目

R3 为 Unity 提供了专门的集成包，通常通过 Unity 包管理器 (UPM) 进行安装。

1.  打开 Unity 编辑器。
2.  进入 `Window` > `Package Manager`。
3.  点击左上角的 `+` 按钮，选择 `Add package from git URL...`。
4.  输入 R3 Unity 包的 Git URL。
    *   *您需要在此处填写实际的 R3 Unity 包的 Git URL。通常，这会是类似 `https://github.com/Cysharp/R3.git?path=src/R3.Unity/Assets/R3.Unity` 的形式，或者如果它发布在 OpenUPM 等仓库，则有其特定的安装方式。请查阅 R3 的官方 GitHub 仓库获取最新的安装说明。*
    *   (假设 URL 为 `https://github.com/Cysharp/R3.git?path=src/R3.Unity/Assets/R3.Unity` 作为示例)
    ```
    https://github.com/Cysharp/R3.git?path=src/R3.Unity/Assets/R3.Unity
    ```
5.  点击 `Add`。Unity 将下载并导入 R3 包。

或者，如果 R3 发布在 OpenUPM 上，您可以按照 OpenUPM 的说明进行安装。

安装完成后，您就可以在 Unity 脚本中使用 R3 的功能了。

## 2. 创建一个简单的 Observable

Observable 是响应式编程的核心。它代表一个可以随时间发出值的序列。R3 提供了多种创建 Observable 的方式。

最简单的方式之一是使用 `Observable.Return`，它创建一个只发出单个值然后立即完成的 Observable：

```csharp
using R3;
using System;

public class Example
{
    public static void CreateSimpleObservable()
    {
        // 创建一个发出字符串 "Hello, R3!" 的 Observable
        Observable<string> greeting = Observable.Return("Hello, R3!");

        // Observable 还未开始发出值，直到有 Observer 订阅它
    }
}
```

另一个常见的工厂方法是 `Observable.Range`，它创建一个发出指定范围内连续整数的 Observable：

```csharp
using R3;
using System;

public class Example
{
    public static void CreateRangeObservable()
    {
        // 创建一个发出 1, 2, 3, 4, 5 的 Observable
        Observable<int> numbers = Observable.Range(1, 5);
    }
}
```

## 3. 订阅 Observable

创建了 Observable 之后，您需要订阅 (Subscribe) 它才能接收其发出的值。订阅时，您需要提供一个 Observer，或者更常见的是提供处理 `OnNext` (接收值)、`OnError` (处理错误) 和 `OnCompleted` (序列完成时) 事件的回调方法。

```csharp
using R3;
using System;

public class Example
{
    public static void SubscribeToObservable()
    {
        Observable<string> greeting = Observable.Return("Hello, R3!");

        // 订阅 Observable 并处理其事件
        IDisposable subscription = greeting.Subscribe(
            onNext: value => Console.WriteLine($"Received: {value}"),
            onError: ex => Console.WriteLine($"Error: {ex.Message}"),
            onCompleted: () => Console.WriteLine("Observable completed.")
        );

        // 输出:
        // Received: Hello, R3!
        // Observable completed.

        // 当不再需要订阅时，应该释放它
        subscription.Dispose();
    }
}
```
`Subscribe` 方法返回一个 `IDisposable` 对象，代表了这个订阅。当您不再对 Observable 的值感兴趣，或者 Observable 的生命周期结束时，调用其 `Dispose()` 方法来取消订阅并释放相关资源非常重要。

## 4. 使用操作符 (Operators)

操作符是 R3 (以及 Rx) 的强大之处。它们是 Observable 的扩展方法，允许您以声明式的方式转换、过滤、组合和处理 Observable 序列。

例如，使用 `Where` 操作符过滤序列，使用 `Select` 操作符转换序列中的每个值：

```csharp
using R3;
using System;
using System.Linq; // Select 和 Where 通常定义在 System.Linq 命名空间下，R3 的操作符也可能如此或在 R3 命名空间下

public class Example
{
    public static void UseOperators()
    {
        Observable<int> numbers = Observable.Range(1, 10); // 1 到 10

        IDisposable subscription = numbers
            .Where(x => x % 2 == 0)  // 过滤出偶数: 2, 4, 6, 8, 10
            .Select(x => x * x)      // 将每个偶数平方: 4, 16, 36, 64, 100
            .Subscribe(
                onNext: value => Console.WriteLine($"Processed value: {value}"),
                onCompleted: () => Console.WriteLine("Processing completed.")
            );

        // 输出:
        // Processed value: 4
        // Processed value: 16
        // Processed value: 36
        // Processed value: 64
        // Processed value: 100
        // Processing completed.

        subscription.Dispose();
    }
}
```
R3 提供了大量操作符，涵盖了各种场景。详细信息请参阅 API 参考部分的[操作符文档](api-reference/operators.md) (稍后创建)。

## 5. 错误处理

Observable 序列可能会因为各种原因而出错。当错误发生时，Observable 会向其 Observer 发送一个 `OnError` 通知，之后序列通常会终止。

您可以在 `Subscribe` 方法中提供 `onError` 回调来处理这些错误：

```csharp
using R3;
using System;

public class Example
{
    public static void HandleErrors()
    {
        Observable<int> errorSource = Observable.Create<int>(observer =>
        {
            observer.OnNext(1);
            observer.OnError(new Exception("Something went wrong!"));
            return Disposable.Empty; // 返回一个可释放对象
        });

        IDisposable subscription = errorSource.Subscribe(
            onNext: value => Console.WriteLine($"Received: {value}"),
            onError: ex => Console.WriteLine($"Error caught: {ex.Message}"), // 捕获并处理错误
            onCompleted: () => Console.WriteLine("Observable completed (this won't be called if an error occurs).")
        );

        // 输出:
        // Received: 1
        // Error caught: Something went wrong!

        subscription.Dispose();
    }
}
```

R3 还提供了如 `Catch`、`Retry` 等操作符来以更高级的方式处理错误，例如从错误中恢复或重试失败的操作。

## 6. 资源管理 (Disposables)

如前所述，`Subscribe` 方法返回一个 `IDisposable` 对象。管理这些 `IDisposable` 对象对于防止内存泄漏和不必要的后台工作至关重要。

*   **手动 Dispose**: 在不再需要订阅时，显式调用 `subscription.Dispose()`。

*   **`CompositeDisposable`**: 当您有多个订阅需要管理时，可以使用 `CompositeDisposable`。它可以将多个 `IDisposable` 对象聚合在一起，然后通过一次 `Dispose()` 调用来释放所有这些对象。

    ```csharp
    using R3;
    using System;

    public class ResourceManager
    {
        private CompositeDisposable disposables = new CompositeDisposable();

        public void StartObserving()
        {
            Observable.Interval(TimeSpan.FromSeconds(1))
                .Subscribe(x => Console.WriteLine($"Timer 1: {x}"))
                .AddTo(disposables); // 将订阅添加到 CompositeDisposable

            Observable.Timer(TimeSpan.FromSeconds(0.5), TimeSpan.FromSeconds(2))
                .Subscribe(x => Console.WriteLine($"Timer 2: {x}"))
                .AddTo(disposables);
        }

        public void StopObserving()
        {
            disposables.Dispose(); // 一次性释放所有订阅
            Console.WriteLine("All subscriptions disposed.");
        }
    }
    ```
    注意: `AddTo` 是 R3 提供的一个便捷扩展方法，可以将 `IDisposable` 直接添加到 `CompositeDisposable` 或其他集合中。

*   **Unity 中的 `AddTo(GameObject/Component)`**: 在 Unity 中，R3 提供了非常方便的 `AddTo(this IDisposable disposable, GameObject gameObject)` 或 `AddTo(Component component)` 扩展方法。这会将订阅的生命周期与指定的 `GameObject` 或 `Component` 绑定。当该 `GameObject` 被销毁时，所有添加到它的订阅都会自动被 `Dispose`。

    ```csharp
    // In a Unity MonoBehaviour script
    using R3;
    using UnityEngine;
    using System; // For TimeSpan

    public class MyUnityComponent : MonoBehaviour
    {
        void Start()
        {
            Observable.EveryUpdate() // R3 提供的 Unity 特定 Observable
                .Subscribe(_ => Debug.Log("Update called!"))
                .AddTo(this); // 订阅会自动在本 MonoBehaviour 被销毁时释放

            Observable.Timer(TimeSpan.FromSeconds(5))
                .Subscribe(_ => Debug.Log("5 seconds passed, and this will be auto-disposed if object is destroyed earlier."))
                .AddTo(this.gameObject); // 也可以添加到 GameObject
        }
    }
    ```
    这是在 Unity 中使用 R3 时管理订阅的首选方式。

通过本入门指南，您应该对 R3 的基本用法有了初步了解。要深入学习，请继续阅读后续的文档章节，特别是核心概念和特定平台集成的部分。
