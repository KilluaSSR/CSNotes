### **LaunchedEffect：在某个可组合项的作用域内运行挂起函数**

> 它让你能在 Compose 组件里跑**挂起函数**（也就是那些可以暂停和恢复的异步任务。`LaunchedEffect` 会在它所在的 Composable **首次进入组合时**运行你提供的挂起函数。此后，它会**持续监听**你作为参数传入的**键（keys）**。

> - 如果这些键中的**任何一个值发生了变化**，`LaunchedEffect` 会**取消**当前正在运行的协程，然后**重新启动**一个新的协程来执行你的挂起函数。
>     
> - 如果键的**值保持不变**（即使 Composable 自身因为其他原因重组），`LaunchedEffect` 也**不会重新执行**其内部代码。

```kotlin
LaunchedEffect(uiState.value.isChoosingDir) {
    if (uiState.value.isChoosingDir) {
        launcher.launch(null)
    }
}
```

将仅在以下两种情况之一中，尝试执行 `launcher.launch(null)`：

1. 当包含此 `LaunchedEffect` 的 Composable **首次被添加到组合中**时，如果 `uiState.value.isChoosingDir` 的初始值为 `true`。
    
2. 当 `uiState.value.isChoosingDir` 的值**从 `false` 变为 `true`** 时。

---

### **rememberCoroutineScope：获取组合感知作用域，以便在可组合项外启动协程**

`rememberCoroutineScope` 用于获取一个 `CoroutineScope`，这个**作用域与调用它的可组合项的生命周期绑定**。当你需要在可组合项的**外部**启动一个协程，但又希望这个协程能感知到组合的生命周期（例如，当可组合项离开组合时，该协程也能被取消）时，就可以使用它。典型的应用场景是在点击事件回调中启动一个协程，或者在 `ViewModel` 中执行与 UI 相关的异步操作。

```kotlin
@Composable
fun MyTimerScreen(onTimeout: () -> Unit) {
    // onTimeout 会随着外部重新组合而改变，但我们不想 LaunchedEffect 每次都重启
    val currentOnTimeout by rememberUpdatedState(onTimeout)

    LaunchedEffect(Unit) { // 使用 Unit 作为 key，表示这个 LaunchedEffect 只会运行一次
        delay(5000) // 延迟5秒
        currentOnTimeout() // 调用最新的 onTimeout 函数
    }

    Text("Timer running...")
}
```
---

### **rememberUpdatedState：在效应中引用某个值，该效应在值改变时不应重启**

`rememberUpdatedState` 用于在某个效应（例如 `LaunchedEffect` 或 `DisposableEffect`）中引用一个可能随时间变化的值，但又不希望这个值的变化导致效应重新启动。它返回一个每次组合时都会更新的 State 对象，但在效应内部引用它时，该效应不会因为这个 State 值的变化而重新执行。这在处理回调函数或需要引用最新状态但又不想破坏效应稳定性的场景中非常有用。

---

### **DisposableEffect：需要清理的效应**

- **开始先做一件事：** 当 Compose 组件出现时，或者你给它设定的 key 变了**的时候，`DisposableEffect` 就会启动，开始执行你让它做的事情（比如注册一个监听器）。
    
- **保证事后清理：** 最关键的是，它会确保在你做完这件事后，或者这个 Compose 组件消失时，**一定会执行清理工作**（比如注销监听器）。

```kotlin
val lifeCycleOwner = LocalLifecycleOwner.current  
DisposableEffect(lifeCycleOwner) {  
    val eventObserver = LifecycleEventObserver{ _, event ->   
		when(event){  
		            Lifecycle.Event.ON_CREATE -> TODO()  
		            Lifecycle.Event.ON_START -> TODO()  
		            Lifecycle.Event.ON_RESUME -> TODO()  
		            Lifecycle.Event.ON_PAUSE -> TODO()  
		            Lifecycle.Event.ON_STOP -> TODO()  
		            Lifecycle.Event.ON_DESTROY -> TODO()  
		            Lifecycle.Event.ON_ANY -> TODO()  
		        }    }  
    lifeCycleOwner.lifecycle.addObserver(eventObserver)  
    onDispose {   
lifeCycleOwner.lifecycle.removeObserver(eventObserver)  
    }  
}
```
---

### **SideEffect：将 Compose 状态发布为非 Compose 代码**

`SideEffect` 是一个在**每次成功重组时都会运行**的副作用。它的主要用途是将 Compose 内部的状态“发布”给外部的非 Compose 管理的代码。例如，当你需要将 Compose 中的状态同步给一个传统的 Android View 或一个外部库时，可以使用 `SideEffect` 来执行这个同步操作。它总是会在组合更新成功后同步执行。

---

### **produceState：将非 Compose 状态转换为 Compose 状态**

`produceState` 是一个协程构建器，用于将外部的、非 Compose 管理的状态源转换为 Compose 能够观察和响应的 State 对象。它启动一个协程来收集来自非 Compose 源（如 `Flow`、LiveData 或其他异步数据源）的数据，并将其作为 Compose `State` 发布出来。当可组合项退出组合时，`produceState` 会自动取消其内部的协程。它非常适合将 ViewModel 中的 `Flow` 或其他异步流数据转换为 UI 可以直接消费的状态。


### **snapshotFlow：将 Compose 的 State 转换为 Flow**

> 把 Compose 状态（`State<T>`）变成一个 **“冷”数据流**（Flow）。

> 简单来说，它的作用是：
> 
> 1. **监视 Compose 状态变化：** 你给 `snapshotFlow` 一个代码块，这个代码块会读取一个或多个 Compose 的 `State` 对象。
>     
> 2. **只在状态不同时才发出新值：** 每当这些被读取的 `State` 对象中**有任何一个发生变化时**，`snapshotFlow` 就会重新运行你的代码块，如果计算出的新结果**和上一次发出的结果不一样**，它就会把这个新结果发送给它的订阅者。


想象一下你有一个显示用户名的 Compose 文本框。用户名是一个 `State` 对象，比如 `val username by remember { mutableStateOf("Killua") }`。

现在你想： “当这个 **用户名改变时**，我想做点什么（比如保存到数据库）。”

传统的 `Flow` 适合处理“事件流”（比如点击、网络响应），但直接监视 Compose 的 `State` 变化并把它们当成一个流来处理，就比较麻烦了。

而`snapshotFlow` ：

- 你告诉它：“去**盯着**那个用户名 (`username.value`)。”
    
- 它会运行一个“任务”（就是你给它的那个代码块），读取用户名。
    
- 如果用户名从 “Killua” 变成了 “Zoldyck”，它会**立刻发现**
    
- 然后它会检查：“新的‘Zoldyck’和旧的‘Killua’一样吗？” 显然不一样。
    
- 于是，它就会把这个新的“Bob”**传递出去**，让订阅它的人知道：“用户名变了，现在是 Zoldyck！”
    
- 如果用户名从 “Zoldyck” 变成 “Zoldyck”（比如因为某种操作又重新设了一次，但值没变），它就不会传递出去，因为值没变。

```kotlin
val listState = rememberLazyListState()

LazyColumn(state = listState) {
    // ...
}

LaunchedEffect(listState) {
    snapshotFlow { listState.firstVisibleItemIndex }
        .map { index -> index > 0 }
        .distinctUntilChanged()
        .filter { it == true }
        .collect {
            MyAnalyticsService.sendScrolledPastFirstItemEvent()
        }
}
```