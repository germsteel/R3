# R3 项目架构与文档计划

## 1. 项目理解

**项目名称:** R3

**核心功能:**
R3 是一个为 .NET 设计的高性能响应式编程库，提供了类似于 Reactive Extensions (Rx) 的功能。它允许开发者使用可观察序列 (Observable sequences) 来处理异步事件流和数据流。核心组件包括 Observables, Observers, Subjects ([`BehaviorSubject<T>`](src/R3/BehaviorSubject.cs:5) 等), Operators, Schedulers/TimeProviders, 和 Disposables。R3 特别注重性能、健壮的错误处理 (如 `OnErrorResume` 机制和全局未处理异常处理器 ([`Observable.cs:78`](src/R3/Observable.cs:78))) 以及通过 `ObservableTracker` ([`ObservableTrackerWindow.cs`](src/R3.Unity/Assets/R3.Unity/Editor/ObservableTrackerWindow.cs:13)) 提供的调试能力。

**平台集成:**
R3 提供了与多种主流 UI 框架和游戏引擎的深度集成，包括 Unity, Blazor, Godot, MonoGame, Stride, Uno Platform, WinForms, WinUI3, 和 WPF。
*   **Unity 集成**:
    *   通过 `MonoBehaviourExtensions` ([`MonoBehaviourExtensions.cs`](src/R3.Unity/Assets/R3.Unity/Runtime/MonoBehaviourExtensions.cs:1)) 提供与 Unity生命周期紧密集成的资源管理 (如 `AddTo` 方法)，并利用新版 Unity 的 `destroyCancellationToken`。
    *   通过 `Triggers/` ([`src/R3.Unity/Assets/R3.Unity/Runtime/Triggers/`](src/R3.Unity/Assets/R3.Unity/Runtime/Triggers/)) 目录下的众多 Trigger 组件和 `ObservableTriggerExtensions` ([`ObservableTriggerExtensions.cs`](src/R3.Unity/Assets/R3.Unity/Runtime/Triggers/ObservableTriggerExtensions.cs:1))，将大量 Unity 事件 (生命周期、物理、UI、鼠标等) 转换为 Observables，API 风格与 UniRx 高度相似。
    *   通过 `UnityFrameProvider` ([`UnityFrameProvider.cs`](src/R3.Unity/Assets/R3.Unity/Runtime/UnityFrameProvider.cs:1)) 和 `UnityTimeProvider` ([`UnityTimeProvider.cs`](src/R3.Unity/Assets/R3.Unity/Runtime/UnityTimeProvider.cs:1))，提供对 Unity PlayerLoop 不同阶段的精确调度控制，并支持多种时间类型 (`Time`, `UnscaledTime`, `Realtime`)。
    *   目前分析未发现直接的 Unity 协程到 Observable 的转换功能，也未发现内置的 `MessageBroker` 实现，这表明 R3 可能鼓励使用基于 Task 的异步或自行构建事件总线。

**项目目标:**
旨在为 .NET 开发者提供一个现代化、高性能、易于调试且功能丰富的响应式编程解决方案，简化异步和事件驱动编程，并与各种流行的开发平台（尤其是 Unity）无缝集成。

## 2. 项目架构图 (Mermaid)

```mermaid
graph TD
    A[R3 Core Library] --> B(Observables);
    A --> C(Observers);
    A --> D(Subjects);
    A --> E(Operators);
    A --> F(Schedulers/TimeProviders);
    A --> G(Disposables);
    A --> H(Error Handling);
    A --> Z(Testing Infrastructure);

    B --> X{Data/Event Stream};
    C --> X;
    D --> X;

    I[Platform Integrations] --> A;
    I --> J[Unity];
    I --> K[Blazor];
    I --> L[Godot];
    I --> M[MonoGame];
    I --> N[Stride];
    I --> O[Uno Platform];
    I --> P[WinForms];
    I --> Q[WinUI3];
    I --> R[WPF];

    J --> S[Unity Editor Tools (ObservableTrackerWindow)];
    J --> T[UnityFrameProvider / UnityTimeProvider];
    J --> U[MonoBehaviour Lifecycle Integration (AddTo, Triggers)];
    J --> V[Unity Event Observables (Collisions, UI, etc.)];

    subgraph R3_Core_Details
        direction LR
        B -.-> E;
        E -.-> B;
        D -.-> B;
        F -.-> E;
    end

    subgraph Unity_Integration_Details
        direction LR
        T -- Manages scheduling for --> V;
        U -- Manages lifetime of --> V;
        S -- Debugs/Visualizes --> V;
    end
```

## 3. 建议的文档结构

1.  **简介 (Introduction)**
    *   什么是 R3？
    *   核心概念 (Observables, Observers, Subjects, Operators, Schedulers/TimeProviders, Disposables)
    *   为什么使用 R3？（解决的问题，优势：性能, 错误处理, 调试工具）
    *   **与 Reactive Extensions (Rx) 和 UniRx 的主要区别与联系** (高层概述)
        *   提及与标准 Rx 的相似性。
        *   简要说明与 UniRx 在 Unity 集成方面的相似性 (事件转换, 生命周期管理) 和差异点 (协程转换, MessageBroker, 时间模型细节, 错误处理哲学)。详细比较放在 Unity 特定文档中。

2.  **入门指南 (Getting Started)**
    *   安装 (核心库, Unity 包)
    *   创建一个简单的 Observable
    *   订阅 Observable
    *   使用操作符 (Operators)
    *   错误处理 (`try-catch`, `OnErrorResume`, 全局处理器)
    *   资源管理 (`AddTo` in Unity, `IDisposable`)

3.  **核心概念详解 (Core Concepts in Depth)**
    *   Observables, Subjects, Operators, Schedulers/TimeProviders, Disposables
    *   错误处理机制详解

4.  **平台集成 (Platform Integrations)**
    *   **Unity**:
        *   安装和设置
        *   **与 UniRx 的详细比较**:
            *   **相似之处**:
                *   事件的 Observable 化 (Triggers, 扩展方法 API 风格)
                *   `AddTo` 生命周期管理
            *   **不同之处**:
                *   **协程转换**: R3 目前未发现直接的 `FromCoroutine` 等效功能，鼓励 Task-based 异步。
                *   **MessageBroker**: R3 未内置，UniRx 提供。讨论替代方案 (手动实现 Subjects)。
                *   **TimeProvider/FrameProvider**: R3 提供更细致的 `TimeKind` 和 PlayerLoop 阶段控制。
                *   **错误处理**: R3 的 `OnErrorResume` 和全局处理器。
                *   **调试工具**: R3 的 `ObservableTrackerWindow`。
                *   **对新版 Unity 特性的利用**: 如 `destroyCancellationToken`。
        *   在 MonoBehaviour 中使用 R3 (生命周期事件, UpdateAsObservable 等)
        *   使用 `UnityFrameProvider` 和 `UnityTimeProvider` (调度, Timer, Delay, Interval, TimeKind)
        *   `ObservableTrackerWindow` 使用指南
        *   示例和最佳实践
    *   Blazor
    *   Godot
    *   MonoGame
    *   Stride
    *   Uno Platform
    *   Windows Forms (WinForms)
    *   WinUI3
    *   WPF
        *   (针对每个平台，说明安装、核心集成点、示例和最佳实践)

5.  **高级主题 (Advanced Topics)**
    *   测试响应式代码
    *   背压 (Backpressure) (如果适用)
    *   自定义操作符
    *   性能考量

6.  **API 参考 (API Reference)**
    *   链接到 [`docs/reference_factory.md`](docs/reference_factory.md:1)
    *   链接到 [`docs/reference_operator.md`](docs/reference_operator.md:1)
    *   其他核心类和方法的参考

7.  **贡献指南 (Contributing)**
    *   如何报告问题
    *   如何提交代码更改
    *   编码规范

8.  **FAQ (Frequently Asked Questions)**
    *   可以包含一些与 UniRx 比较的常见问题。
    *   如何实现事件总线/MessageBroker？
    *   如何处理类似协程的异步序列？
