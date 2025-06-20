
MVVM是一种软件架构设计模式，它强制性地将用户界面（UI）与业务逻辑和数据处理分离开来。这个名字代表了它的三个核心组件：

- **Model (模型层)**：负责数据处理和业务逻辑。它包含从网络、数据库或缓存中获取和存储数据的代码。Model层不直接与View层交互，它只关心数据本身。
- **View (视图层)**：负责UI展示，通常是Activity或Fragment。它只做与UI相关的事情，比如显示数据、响应用户的点击事件。View层非常“薄”，它不包含任何业务逻辑。
- **ViewModel (视图模型层)**：是View和Model之间的桥梁。它从Model层获取数据，并对数据进行处理，转换成View层可以直接展示的格式。ViewModel通过**数据绑定(Data Binding)**或**可观察的数据(Observable Data)**（如`LiveData`）将数据暴露给View。

**MVVM的核心优势**：

1. **低耦合**：View和Model不直接通信，而是通过ViewModel。这使得各层可以独立开发和测试。
2. **生命周期感知**：Android Jetpack中的`ViewModel`组件具有生命周期感知能力。它存活于配置变更（如屏幕旋转）中，不会因为Activity或Fragment的重建而销毁，从而避免了数据的重复加载和丢失。
3. **可测试性**：由于ViewModel不持有任何View的引用，也不处理UI逻辑，因此可以轻松地进行单元测试。

### MVVM持有关系

- View持有ViewModel引用（通过观察者模式）
- ViewModel持有Model引用
- Model不持有其他组件引用
- View和ViewModel通过数据绑定连接，没有直接引用

### ViewModel数据保持机制

ViewModel通过 `ViewModelStore` 保持数据。Activity 销毁重建时，ViewModelStore 会被保存在 `NonConfigurationInstances` 中，新Activity创建时会恢复ViewModelStore，从而保持ViewModel实例不变。只有Activity真正finish时，ViewModel才会被清理。