# Android Handler 机制详解

## 核心组件及其作用

### 1. Handler
**作用**：消息的发送者和处理者
- 负责发送消息到MessageQueue
- 处理从MessageQueue中取出的消息
- 提供发送消息的API：`sendMessage()`、`post()`等

**核心方法**：
- `sendMessage(Message msg)`：发送消息
- `post(Runnable r)`：发送一个Runnable任务
- `handleMessage(Message msg)`：处理消息的回调方法

### 2. MessageQueue
**作用**：消息队列，存储待处理的消息
- 内部使用单链表结构存储Message
- 按照消息的执行时间排序（时间早的在前）
- 提供消息的入队和出队操作

**核心方法**：
- `enqueueMessage()`：消息入队
- `next()`：取出下一个要处理的消息

### 3. Looper
**作用**：消息循环器，从MessageQueue中取消息并分发
- 每个线程最多只能有一个Looper
- 通过`loop()`方法开始无限循环处理消息
- 管理当前线程的MessageQueue

**核心方法**：
- `prepare()`：为当前线程创建Looper
- `loop()`：开始消息循环
- `myLooper()`：获取当前线程的Looper

### 4. Message
**作用**：消息载体，封装要传递的数据
- 包含消息标识、数据内容、目标Handler等信息
- 使用对象池机制，通过`obtain()`获取，避免频繁创建

## 创建时机和流程

### 1. 主线程Handler创建
主线程在启动时，ActivityThread的main方法中已经：
```java
// 1. 创建主线程Looper
Looper.prepareMainLooper();

// 2. 开始消息循环
Looper.loop();
```

因此主线程可以直接创建Handler：
```java
Handler handler = new Handler() {
    @Override
    public void handleMessage(Message msg) {
        // 处理消息
    }
};
```

### 2. 子线程Handler创建
子线程必须先准备Looper：
```java
// 在子线程中
Looper.prepare();  // 创建Looper
Handler handler = new Handler() {
    @Override
    public void handleMessage(Message msg) {
        // 处理消息
    }
};
Looper.loop();  // 开始消息循环
```

## 工作原理

### 完整流程
1. **初始化阶段**：线程创建Looper和MessageQueue
2. **消息发送**：Handler调用sendMessage将消息加入MessageQueue
3. **消息循环**：Looper.loop()不断从MessageQueue取消息
4. **消息处理**：将消息分发给对应的Handler处理

### 线程通信原理
- Handler持有创建它的线程的Looper引用
- 不同线程的Handler发送的消息都会进入同一个MessageQueue
- Looper在自己的线程中处理消息，实现线程切换

## 面试常考问题

### 1. Handler内存泄漏问题
**问题**：为什么Handler会造成内存泄漏？
**答案**：
- 非静态内部类Handler持有外部Activity引用
- 如果有延时消息未处理完，Activity无法被GC回收
- **解决方案**：使用静态内部类+弱引用，或在onDestroy中移除消息

### 2. Handler是阻塞还是非阻塞？
**答案**：
- MessageQueue.next()方法在没有消息时会阻塞
- 使用Linux的epoll机制，当有新消息时会被唤醒
- 这种阻塞不会消耗CPU资源

### 3. Handler.post(Runnable)的执行机制
**原理**：
```java
// post方法将Runnable包装成Message
private static Message getPostMessage(Runnable r) {
    Message m = Message.obtain();
    m.callback = r;  // Runnable存储在callback字段
    return m;
}

// 在dispatchMessage中执行
public void dispatchMessage(Message msg) {
    if (msg.callback != null) {
        handleCallback(msg);  // 直接调用runnable.run()
    } else {
        handleMessage(msg);   // 调用handleMessage
    }
}
```

### 4. 一个线程可以有几个Handler？
**答案**：
- 一个线程可以有多个Handler
- 但一个线程只能有一个Looper和一个MessageQueue
- 所有Handler共享同一个MessageQueue

### 5. MessageQueue是队列吗？
**答案**：
- 虽然叫Queue，但内部实现是单链表
- 按照消息的执行时间排序，不是严格的FIFO
- 支持优先级和延时消息

### 6. 为什么主线程使用Handler不会ANR？
**答案**：
- Handler机制本身不会导致ANR
- ANR是因为主线程被长时间阻塞（如网络请求、复杂计算）
- Handler只是消息分发机制，关键是handleMessage中的操作要快速完成

### 7. Handler发送消息有哪些方式？
**答案**：
- `sendMessage(Message msg)`：发送消息对象
- `sendEmptyMessage(int what)`：发送空消息
- `sendMessageDelayed(Message msg, long delayMillis)`：延时发送
- `post(Runnable r)`：发送Runnable任务
- `postDelayed(Runnable r, long delayMillis)`：延时发送Runnable

### 8. 如何保证Handler消息的顺序性？
**答案**：
- MessageQueue内部是单线程消费，天然保证顺序
- 同一时间只有一个消息在处理
- 延时消息会按照执行时间重新排序
