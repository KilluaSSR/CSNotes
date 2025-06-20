
## 核心参与者

- **ActivityManagerService (AMS)**：
    - **位置**：运行在 `system_server` 进程中。
    - **角色**：Android 系统的“大管家”，**负责所有应用组件（Activity、Service、BroadcastReceiver、ContentProvider）的启动、停止、生命周期管理、进程调度、内存管理和权限检查等**。
    - **实现**：核心是一个 Binder 服务端。
- **ActivityManagerProxy (AMP)**：
    - **位置**：运行在**各个应用进程中**。
    - **角色**：AMS 在应用进程的代理（Binder 客户端），封装了 Binder 操作，允许应用进程远程调用 `system_server` 进程中的 AMS 方法。
- **ActivityThread**：
    - **位置**：运行在**每个应用进程中**。
    - **角色**：应用进程的 Java 层真正入口点，其 `main()` 方法是应用进程启动的起点。它**维护着应用进程的主线程（UI 线程）的 Looper，并负责接收和分发来自 AMS 的指令，驱动应用组件生命周期**。
- **ApplicationThread**：
    - **位置**：`ActivityThread` 的一个内部类，作为 `ActivityThread` 的成员变量，运行在应用进程中。
    - **角色**：AMS 在应用进程的远程调用接口（Binder 服务端），负责处理来自 `system_server` 进程（通过 `ApplicationThreadProxy`）的 IPC 请求，并将这些请求转发给 `ActivityThread` 的相应方法去处理。
- **ApplicationThreadProxy (ATP)**：
    - **位置**：运行在 `system_server` 进程中。
    - **角色**：`ApplicationThread` 在 `system_server` 进程的代理（Binder 客户端）。AMS 通过它远程调用应用进程中的 `ApplicationThread` 方法，从而实现对应用进程的控制和管理。
- **Instrumentation**：
    - **位置**：运行在应用进程中，是 `ActivityThread` 的一个成员变量。
    - **角色**：一个监控和控制应用与系统交互的类，负责实际反射实例化 Activity、调用 Activity 的生命周期方法等。

## 从点击Launcher图标开始，发生了什么

**阶段一：Launcher 进程与 `System_Server` 进程的通信**

1. **`Launcher` 发起 `startActivity` 请求（`Launcher` 进程 - UI 线程）**
    
    - 用户点击 Launcher 图标，**触发 `Launcher` 进程主线程（UI 线程）的 `startActivity()` 调用**。
    - `Activity.startActivity()` 最终会**通过 `Instrumentation.execStartActivity()` 方法，再通过 Binder 机制，调用 `ActivityManagerProxy (AMP)` 的 `startActivity()` 方法。**
    - **Binder IPC**: `AMP.startActivity()` 作为 Binder 客户端，**将启动 Activity 的 `Intent`、Flags 等参数打包，通过 Binder 驱动发送到 `system_server` 进程。**
2. **`AMS` 处理启动请求（`system_server` 进程 - Binder 线程）**
    
    - `system_server` 进程中的某个空闲 Binder 线程**接收到来自 Binder 驱动的 IPC 请求**。
    - Binder 机制根据**请求的 Method ID，将调用分发给 `ActivityManagerService (AMS)` 的 `startActivity()` 方法**。
    - **`showStartingWindow`**: 在 `AMS.startActivity()` 的处理过程中，**`AMS` 会通知 `WindowManagerService (WMS)` 显示Starting Window（也称预览窗口）**。这个窗口是系统为了提供平滑过渡的用户体验而设计的，它显示的是目标 Activity 主题中定义的背景或预览图。由于它由 `system_server` 进程渲染，不需要等待应用进程启动和初始化，所以显示速度非常快，用户几乎在点击图标的瞬间就能看到。
3. **`AMS` 通知 Launcher 暂停（`system_server` 进程 -> `Launcher` 进程）**
    
    - 为了确保目标 Activity 启动后，Launcher 的 Activity 能够正确进入暂停状态，`AMS` 会通过 `ApplicationThreadProxy (ATP)` 远程调用 `Launcher` 进程中 `ApplicationThread` 的 `schedulePauseActivity()` 方法。
    - **Binder IPC**: `ATP.schedulePauseActivity()` 作为 Binder 客户端，将暂停指令发送给 `Launcher` 进程。
    - **`Launcher` 进程处理暂停请求（`Launcher` 进程 - Binder 线程 -> UI 线程）**
        - `Launcher` 进程中的某个 Binder 线程接收到 IPC 请求，调用 `ApplicationThread.schedulePauseActivity()`。
        - 这个方法会将一个消息发送到 `Launcher` 进程主线程的 `Handler` (`mH`) 中。
        - `Launcher` 主线程从消息队列中取出消息，执行 `ActivityThread.handlePauseActivity()`。
        - `handlePauseActivity()` 会调用 `Instrumentation.callActivityOnPause()`，最终导致 `Launcher` 的当前 Activity 执行 `onPause()` 生命周期方法。
        - `Launcher` 的 Activity `onPause()` 执行完毕后，会再次通过 `AMP.activityPaused()` 通知 `AMS`，表示暂停已完成。


**阶段二：新应用进程的创建与初始化**

4. **`AMS` 判断并启动新进程（`system_server` 进程 - Binder 线程）**
    
    - `AMS` 收到 `Launcher` 的 `activityPaused()` 通知后，会继续执行 `startActivity` 的后续逻辑。
    - `AMS` 会检查要启动的 Activity 所需的进程是否存在。
    - **如果进程不存在**：`AMS` 将通过 `Process.start()` 方法，与 `Zygote` 进程进行 Socket 通信，请求 `Zygote` `fork` 出一个新的应用进程。
        - `Zygote` 进程是所有应用进程的父进程，它预加载了 Android Framework 的公共类和资源，通过 `fork` 新进程可以大大加快应用启动速度。
        - 在 `Process.start()` 中，会传递 `android.app.ActivityThread` 作为新进程的入口点（`entryPoint`）。
    - **如果进程已存在**：`AMS` 会直接进入 `realStartActivityLocked` 阶段（跳过 `Process.start` ）。
5. **新应用进程启动（新应用进程 - UI 线程）**
    
    - `Zygote fork` 成功后，新进程会从 `ActivityThread.main()` 方法开始执行 Java 代码。
    - **`ActivityThread.main()` 的核心操作：**
        - **`Looper.prepareMainLooper()`**: 为新进程的主线程创建并初始化一个 `Looper` 和 `MessageQueue`。这是主线程事件循环的基础。
        - **创建 `ActivityThread` 实例**: `ActivityThread` 会实例化自身，并创建 `ResourcesManager`、`ApplicationThread`（作为 Binder 服务端）等核心组件。
        - **`thread.attach(false)`**: 这是关键一步。`ActivityThread` 会通过 `AMP.attachApplication()` 将其内部的 `ApplicationThread` 对象（作为 Binder 客户端 `mAppThread`）注册到 `AMS`。
            - **Binder IPC**: `mAppThread` 作为新应用进程的 Binder 服务端，它的 `Binder` 对象被传递给了 `AMS`。从此，`AMS` 就获得了与该应用进程通信的能力。`AMS` 会将这个 `IApplicationThread` 对象关联到其内部的 `ProcessRecord` 中。
        - **`Looper.loop()`**: 启动主线程的消息循环，使主线程开始处理消息队列中的消息。

**阶段三：应用组件的初始化与 Activity 生命周期**

6. **`AMS` 绑定应用与初始化组件（`system_server` 进程 -> 新应用进程）**
    
    - `AMS` 收到 `attachApplication()` 请求后，会通过 `ATP.bindApplication()` 远程调用新应用进程中 `ApplicationThread` 的 `bindApplication()` 方法。
    - **Binder IPC**: `bindApplication()` 作为 Binder 客户端，将请求发送到新应用进程。
    - **新应用进程处理 `bindApplication`（新应用进程 - Binder 线程 -> UI 线程）**
        - 新应用进程的 Binder 线程收到请求，调用 `ApplicationThread.bindApplication()`。
        - 该方法将一个消息发送到主线程的 `Handler` (`mH`)。
        - 主线程处理 `handleBindApplication()` 消息，执行以下重要步骤：
            - **实例化 Application 对象**：通过 `r.packageInfo.makeApplication()` 实例化应用的 `Application` 类。如果该进程是第一次承载此应用，这里会调用 `Application.onCreate()`。注意，`ContentProvider` 的 `onCreate()` 通常会在此之前被调用（如果应用有 `ContentProvider`）。
            - **初始化 `Instrumentation`**：`ActivityThread` 内部的 `mInstrumentation` 对象也会在此处被初始化。
            - **安装 `ContentProvider`**：调用 `installContentProviders()` 方法，实例化应用中声明的所有 `ContentProvider`，并调用它们的 `onCreate()` 方法，最后将它们注册到 `AMS`。
            - **调用 `Application.onCreate()`**：通过 `mInstrumentation.callApplicationOnCreate()` 实际调用 `Application` 类的 `onCreate()` 方法。
        - 至此，应用进程的骨架（Application、ContentProvider）已经搭建完成并初始化。
7. **`AMS` 启动 Activity 实例（`system_server` 进程 -> 新应用进程）**
    
    - 在 `AMS` 内部，几乎与 `bindApplication()` 同时，`AMS` 还会通过 `ATP.scheduleLaunchActivity()` 远程调用新应用进程中 `ApplicationThread` 的 `scheduleLaunchActivity()` 方法。
    - **Binder IPC**: `scheduleLaunchActivity()` 作为 Binder 客户端，将启动 Activity 的请求发送到新应用进程。
    - **新应用进程处理 `scheduleLaunchActivity`（新应用进程 - Binder 线程 -> UI 线程）**
        - 新应用进程的 Binder 线程收到请求，调用 `ApplicationThread.scheduleLaunchActivity()`。
        - 该方法将一个消息发送到主线程的 `Handler` (`mH`)。
        - 主线程处理 `handleLaunchActivity()` 消息，这是 Activity 启动的核心：
            - **反射实例化 Activity**：通过 `Instrumentation.newActivity()` 反射创建目标 Activity 的实例。
            - **创建 `ContextImpl` 并 `attach` Activity**：为 Activity 创建一个 `ContextImpl` 实例作为其基本上下文，并通过 `activity.attach()` 方法将其关联到 Activity，同时还会创建 `PhoneWindow`。
            - **调用 `Activity` 生命周期的 `onCreate()`**：通过 `mInstrumentation.callActivityOnCreate()` 调用 `Activity.onCreate()`。
            - **调用 `Activity` 生命周期的 `onStart()`**：通过 `activity.performStart()` 调用 `Activity.onStart()`。
            - **调用 `Activity` 生命周期的 `onResume()`**：通过 `handleResumeActivity()` 最终调用 `Activity.onResume()`。
        - 至此，目标 Activity 已经完全启动并进入了可交互状态。

**阶段四：Launcher 进程的后续处理**

8. **`AMS` 通知 Launcher 停止（`system_server` 进程 -> `Launcher` 进程）**
    - 在 `AMS` 内部，当新的 Activity 成功启动后，`AMS` 会继续处理 `Launcher` 的暂停流程（之前 `activityPaused()` 的后续）。
    - `AMS` 会再次通过 `ATP.scheduleStopActivity()` 远程调用 `Launcher` 进程中 `ApplicationThread` 的 `scheduleStopActivity()` 方法。
    - **Binder IPC**: `scheduleStopActivity()` 将停止请求发送给 `Launcher` 进程。
    - **`Launcher` 进程处理停止请求（`Launcher` 进程 - Binder 线程 -> UI 线程）**
        - `Launcher` 进程的 Binder 线程收到请求，调用 `ApplicationThread.scheduleStopActivity()`。
        - 该方法将一个消息发送到主线程的 `Handler` (`mH`)。
        - 主线程处理消息，执行 `ActivityThread.handleStopActivity()`，最终导致 `Launcher` 的当前 Activity 执行 `onStop()` 生命周期方法。
