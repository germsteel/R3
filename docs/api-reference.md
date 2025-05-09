# API 参考

本章节提供了 R3 库的详细 API 文档，包括 Observable 的工厂方法、各种操作符以及其他核心组件。

## Observable 工厂方法 (Factory Methods)

详细介绍了用于创建 Observable 序列的各种静态方法。

*   **[Observable 工厂方法参考](./reference_factory.md)**

## Observable 操作符 (Operators)

详细介绍了可用于转换、过滤、组合和处理 Observable 序列的各种操作符。

*   **[Observable 操作符参考](./reference_operator.md)**

## 其他核心类和方法

本部分将涵盖 R3 库中除了工厂方法和操作符之外的其他重要公共 API，例如：

*   **`Observable<T>`**: 核心可观察序列类。
*   **`Observer<T>`**: 核心观察者接口/基类。
*   **`Subject<T>` 类**:
    *   `Subject<T>`
    *   `BehaviorSubject<T>`
    *   `ReplaySubject<T>`
    *   `AsyncSubject<T>`
*   **`TimeProvider` 和特定平台的实现**:
    *   `System.Threading.TimeProvider` (如果 R3 基于此)
    *   `R3.UnityTimeProvider` (及其 `TimeKind`)
    *   `R3.UnityFrameProvider`
*   **`Disposable` 相关类型**:
    *   `IDisposable`
    *   `Disposable` (静态辅助类，如 `Disposable.Empty`, `Disposable.Create`)
    *   `CompositeDisposable`
    *   `SerialDisposable`
    *   `SingleAssignmentDisposable`
*   **`Unit` 类型**: 代表 `void` 的类型，常用于表示事件或信号。
*   **`Result<T>` (如果适用)**: 用于封装成功或失败结果的类型。
*   **`ObservableSystem`**: 用于配置全局设置，如未处理异常处理器。
*   **Unity 特有的扩展和组件**:
    *   `MonoBehaviourExtensions` (如 `AddTo`, `GetDestroyCancellationToken`)
    *   Trigger 组件 (如 `ObservableUpdateTrigger`, `ObservableCollisionTrigger` 等)
    *   `ObservableTrackerWindow` (编辑器工具，其公共 API 或配置，如果适用)

(详细内容待补充，将根据需要为每个重要组件或类型提供说明、属性、方法和使用示例。)
