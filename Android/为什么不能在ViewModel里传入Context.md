
> 这是一个老生常谈的话题，很多时候一些人会图简便，在 ViewModel 里传入 Context，以进行方便的函数调用，但是这会导致几个问题。

## 内存泄漏
是的，一般一种方法不被推荐，那么八成和内存泄漏有关（剩下两成大抵是不易于测试维护吧）。

先一起看看ViewModel 的生命周期：

[`ViewModel`](https://developer.android.com/reference/androidx/lifecycle/ViewModel?hl=zh-cn) 的生命周期与其作用域直接关联。`ViewModel` 会一直保留在内存中，直到其作用域 [`ViewModelStoreOwner`](https://developer.android.com/reference/kotlin/androidx/lifecycle/ViewModelStoreOwner?hl=zh-cn) 消失：

- 对于 activity，是在 activity 完成时。
- 对于 fragment，是在 fragment 分离时。
- 对于 Navigation 条目，是在 Navigation 条目从返回堆栈中移除时。

下图说明了 activity 经历屏幕旋转的各种生命周期状态。

![说明 ViewModel 随着 activity 状态的改变而经历的生命周期。](https://developer.android.com/static/images/topic/libraries/architecture/viewmodel-lifecycle.png?hl=zh-cn)

我们在系统首次调用 activity 对象的 [`onCreate()`](https://developer.android.com/reference/android/app/Activity?hl=zh-cn#onCreate\(android.os.Bundle\)) 方法时请求 [`ViewModel`](https://developer.android.com/reference/androidx/lifecycle/ViewModel?hl=zh-cn)。系统可能会在 activity 的整个生命周期内多次调用 [`onCreate()`](https://developer.android.com/reference/android/app/Activity?hl=zh-cn#onCreate\(android.os.Bundle\))，如在旋转设备屏幕时。[`ViewModel`](https://developer.android.com/reference/androidx/lifecycle/ViewModel?hl=zh-cn) 存在的时间范围是从首次请求 [`ViewModel`](https://developer.android.com/reference/androidx/lifecycle/ViewModel?hl=zh-cn) 直到 activity 完成并销毁。

可以看到，ViewModel 的生命周期可能比 `ViewModelStoreOwner` 更长。因此 `ViewModel` 绝不应保留任何对与生命周期相关的 API（例如 `Context` 或 `Resources`）的引用，以免发生内存泄漏。

## 没有做好职责分离
**朋友，ViewModel 是干什么的？**

它的职责是持有和管理 UI 所需的数据。像访问数据库、网络请求、获取系统服务等需要 `Context` 的操作，应该由 **Repository** 或 **Use Case** 等更底层的模块来处理。将这些操作放在 `ViewModel` 中会混淆职责，使得 `ViewModel` 变得臃肿，也更难维护。

如果你需要使用 `Context` 来显示 Toast 或者启动 Activity，那么这些操作应该在 **Composable 函数内部**（通过 `LocalContext.current` 获取 `Context`）或在 **Composable 函数所观察到的状态变化触发的回调中**执行。例如，当 `ViewModel` 中的某个事件状态变为 `ShowToastEvent` 时，Composable 函数观察到这个事件，然后在 effect 块中获取 `LocalContext.current` 并显示 Toast。

## 不利于测试
`ViewModel` 的核心职责是提供 UI 显示的内容。如果 `ViewModel` 依赖 `Context`，那么在进行单元测试时，你需要模拟 `Context` 对象，这会大大增加测试的复杂性。一个不依赖 `Context` 的 `ViewModel` 可以更轻松地进行纯粹的逻辑测试。