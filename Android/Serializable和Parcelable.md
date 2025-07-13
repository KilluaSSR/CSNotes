
> `Serializable` 和 `Parcelable` 都是 Android 中用于实现对象**序列化**（将对象转换为字节序列以便存储或传输）的接口。

#### `Serializable` (Java 标准序列化)

- **实现方式：** 实现了 `java.io.Serializable` 接口。无需实现任何方法，由 Java 的反射机制自动完成序列化和反序列化过程。
- **原理：**
    - 通过 **Java 反射机制**将对象的状态写入到一个序列化流中。
    - 在反序列化时，通过反射机制将字节流转换回对象。
- **优点：**
    - **实现简单：** 只需实现一个接口，几乎不需要写额外代码。
    - **通用性：** 适用于 Java 平台，可用于存储到磁盘、网络传输（例如，通过 `ObjectOutputStream` 和 `ObjectInputStream`）。
- **缺点：**
    - **效率低：** 由于使用了大量的反射，序列化和反序列化过程会产生大量的临时对象，导致性能开销较大，且频繁 GC。
    - **开销大：** 序列化后的数据量通常比 `Parcelable` 大。
    - **不适合 Android IPC：** 在 Android 中进行频繁的 IPC 时，性能瓶颈明显。
    - **`serialVersionUID`：** 建议手动指定 `serialVersionUID`，否则在类结构发生变化时可能导致反序列化失败。

#### `Parcelable` (Android 特有序列化)

- **实现方式：** 实现了 `android.os.Parcelable` 接口，并需要实现 `writeToParcel()` 和 `describeContents()` 方法，以及一个静态的 `CREATOR` 字段。
- **原理：**
    - 通过**底层 C++ 实现**，直接将数据写入到一块共享内存中，并可以从这块内存中读取数据。
    - 它的设计目标是为了在 Android 的 Binder 机制中高效传输数据。
- **优点：**
    - **效率高：** 性能远优于 `Serializable`，因为它避免了反射和大量的临时对象创建，直接操作内存。
    - **速度快：** 序列化和反序列化速度快。
    - **数据量小：** 序列化后的数据量更小。
    - **专为 IPC 设计：** 是 Android 平台进行组件间数据传输（尤其是通过 `Intent` 或 `Binder`）的首选方式。
- **缺点：**
    - **实现复杂：** 需要手动编写序列化和反序列化代码，通常需要实现 `writeToParcel()` 和 `CREATOR` 静态内部类。
    - **平台限制：** 只能在 Android 平台使用，不能跨平台。

| 特性       | `Serializable`         | `Parcelable`                             |
| :------- | :--------------------- | :--------------------------------------- |
| **接口**   | `java.io.Serializable` | `android.os.Parcelable`                  |
| **实现**   | 简单，无方法实现（基于反射）         | 复杂，需实现 `writeToParcel()` 和 `CREATOR`     |
| **效率**   | **低**，大量反射和临时对象，频繁 GC  | **高**，直接内存操作，性能优异                        |
| **适用场景** | 存储到磁盘、网络传输（Java 标准）    | **Android IPC (Intent, Binder)**，组件间数据传递 |
| **跨平台**  | 是 (Java 标准)            | **否** (Android 平台特有)                     |
| **性能瓶颈** | 在 Android IPC 中性能瓶颈明显  | Android IPC 首选，高性能                       |
