
> **NDK (Native Development Kit)** 是 Android 提供的一套工具集，允许开发者在 Android 应用中使用 **C、C++ 等原生语言**编写代码。这些原生代码被编译为**本地库 (`.so` 文件)**，然后可以通过 **JNI (Java Native Interface)** 与 Java 代码进行交互。


## **为什么要使用 NDK / C/C++ 开发？**

尽管 Android 应用主要使用 Java/Kotlin 开发，但 NDK 在以下场景中提供了重要的优势：

1. **性能优化：** 对于计算密集型任务（如图像处理、音频/视频编解码、游戏引擎、物理模拟），C/C++ 通常比 Java 提供更高的执行效率和更低的延迟。
2. **代码复用：** 许多现有的库和算法都是用 C/C++ 编写的（例如 OpenCV, FFmpeg）。通过 NDK，可以直接在 Android 应用中集成这些库，避免重复开发。
3. **系统级编程：** 某些底层操作或与硬件交互的场景可能需要直接使用 C/C++。
4. **数据安全：** 相较于 Java/Kotlin 层，C/C++ 层的代码更难被反编译，可以在一定程度上增加核心算法的安全性（但并非绝对安全）。
5. **内存控制：** C/C++ 允许更精细的内存管理（手动分配和释放），对于内存敏感的应用可能有用（但也更容易引入内存泄漏）。

## **NDK 开发的主要流程**

1. **编写 C/C++ 代码：** 编写实现特定功能的 `.c` 或 `.cpp` 源文件。
2. **JNI 接口定义：** 在 Java 代码中声明 `native` 方法，这些方法将与 C/C++ 函数对应。
    
    
    ```Java
    public class MyNativeLib {
        static {
            System.loadLibrary("my_native_lib"); // 加载本地库
        }
        public native String getNativeString(); // 声明native方法
        public native int add(int a, int b);
    }
    ```
    
3. **JNI 实现 (C/C++)：** 在 C/C++ 代码中实现对应的 JNI 函数。这些函数需要遵循 JNI 规范的命名约定，以便 Java 虚拟机能够找到并调用它们。
    - 函数签名通常为 `JNIEXPORT ReturnType JNICALL Java_PackageName_ClassName_MethodName(JNIEnv* env, jobject thiz, ...)`。
    - **示例 (`my_native_lib.cpp`):**
        

        
        ```  C++
        #include <jni.h>
        #include <string>
        
        extern "C" JNIEXPORT jstring JNICALL
        Java_com_example_myapp_MyNativeLib_getNativeString(
                JNIEnv* env,
                jobject /* this */) {
            std::string hello = "Hello from C++";
            return env->NewStringUTF(hello.c_str());
        }
        
        extern "C" JNIEXPORT jint JNICALL
        Java_com_example_myapp_MyNativeLib_add(
                JNIEnv* env,
                jobject /* this */,
                jint a,
                jint b) {
            return a + b;
        }
        ```
        
4. **配置 CMake 或 NDK-Build：** 在 `build.gradle` 中配置 CMake 或 NDK-Build，指定 C/C++ 源文件、目标平台等信息，让 Gradle 构建系统能够编译本地代码。
5. **编译与打包：** Gradle 会调用 NDK 工具链将 C/C++ 代码编译成针对不同 CPU 架构的共享库文件（`.so` 文件），并将它们打包到 APK 的 `lib` 目录下。
6. **加载本地库：** 在 Java 代码中，使用 `System.loadLibrary("your_lib_name")` 加载 `.so` 库。
7. **调用 Native 方法：** 像调用普通 Java 方法一样调用 `native` 方法。


## NDK 库

> **NDK 库**指的是使用 Android NDK 工具集编译生成的**本地共享库文件**，通常以 `.so` 为后缀（Shared Object）。这些库文件包含了用 C、C++ 或其他原生语言编写的机器码。它们可以在 Android 设备上运行，并通过 JNI (Java Native Interface) 与 Java/Kotlin 代码进行交互。

可以把 NDK 库理解为：

- **平台特定的二进制文件：** 每个 `.so` 文件都是针对特定的 CPU 架构（如 ARMv7, ARM64, x86）编译的。
- **包含原生代码：** 它们是 Java 虚拟机无法直接执行的原生机器指令。
- **通过 JNI 暴露接口：** 它们通过 JNI 接口向 Java 层暴露函数，使得 Java 代码可以调用这些原生函数。

## 如何在 JNI 中注册 Native 函数

> 在 JNI 中注册 Native 函数主要有两种方式：**静态注册**和**动态注册**。

##### 1. 静态注册 (Static Registration)

- **原理：** JNI 根据 Java `native` 方法的完整签名（包名、类名、方法名、参数类型）来查找对应的 C/C++ 函数。C/C++ 函数的命名必须严格遵循 JNI 规范。
- **命名规范：**`Java_PackageName_ClassName_MethodName_ParameterSignatures`
    - `Java_`：固定前缀。
    - `PackageName`：Java 类所在的包名，点号 `.` 替换为下划线 `_`。
    - `ClassName`：Java 类名。
    - `MethodName`：Java `native` 方法名。
    - `ParameterSignatures` (可选)：如果存在同名重载方法，需要加上参数签名以区分。
- **优点：** 实现简单，不需要额外的注册代码。
- **缺点：**
    - 函数名很长且容易出错。
    - 如果 Java 类名、方法名或包名改变，C/C++ 函数名也必须相应改变。
    - 加载时需要遍历 `.so` 文件查找对应函数，效率略低。
- **使用示例：**
    1. **Java 代码：**
        
        
        ```
        package com.example.myapp;
        
        public class MyNativeLib {
            static {
                System.loadLibrary("my_lib"); // 加载名为 'my_lib.so' 的库
            }
            public native String getStringFromNative(); // Java native方法
        }
        ```
        
    2. **C/C++ 实现 (`my_lib.cpp`):**
        
        
        ```        C++
        #include <jni.h>
        #include <string>
        
        // 遵循JNI命名规范：Java_com_example_myapp_MyNativeLib_getStringFromNative
        extern "C" JNIEXPORT jstring JNICALL
        Java_com_example_myapp_MyNativeLib_getStringFromNative(JNIEnv *env, jobject thiz) {
            std::string hello = "Hello from static JNI!";
            return env->NewStringUTF(hello.c_str());
        }
        ```
        

##### 2. 动态注册 (Dynamic Registration)

- **原理：** 在 `.so` 库加载时，通过 JNI 函数 `RegisterNatives()` 将 Java `native` 方法和 C/C++ 函数进行**映射关联**。这种方式不依赖于函数名，而是通过方法签名。
- **优点：**
    - C/C++ 函数名可以任意命名，更简洁。
    - Java 代码修改后，C/C++ 代码无需修改函数名（只要方法签名不变）。
    - 加载效率高，因为直接注册了映射关系。
- **缺点：** 稍微复杂一点，需要编写注册代码。
- **使用示例：**
    1. **Java 代码：**
        
        ```Java
        package com.example.myapp;
        
        public class MyNativeLib {
            static {
                System.loadLibrary("my_lib"); // 加载名为 'my_lib.so' 的库
            }
            public native String getDynamicString(); // Java native方法
            public native int calculateSum(int a, int b);
        }
        ```
        
    2. **C/C++ 实现 (`my_lib.cpp`):**
        
        ```C++
        #include <jni.h>
        #include <string>
        
        // C++ 实现函数，名称可以任意
        jstring native_getDynamicString(JNIEnv *env, jobject thiz) {
            std::string hello = "Hello from dynamic JNI!";
            return env->NewStringUTF(hello.c_str());
        }
        
        jint native_calculateSum(JNIEnv *env, jobject thiz, jint a, jint b) {
            return a + b;
        }
        
        // 定义JNI_METHOD_ARGS结构体，用于映射Java方法和C++函数
        // { "Java方法名", "Java方法签名", (void*)C++函数指针 }
        static const JNINativeMethod gMethods[] = {
            {"getDynamicString", "()Ljava/lang/String;", (void*)native_getDynamicString},
            {"calculateSum", "(II)I", (void*)native_calculateSum},
        };
        
        // JNI_OnLoad 是当.so库被System.loadLibrary()加载时，JVM会自动调用的函数
        // 可以在这里进行动态注册
        extern "C" JNIEXPORT jint JNICALL JNI_OnLoad(JavaVM* vm, void* reserved) {
            JNIEnv* env = nullptr;
        
            // 获取JNIEnv
            if (vm->GetEnv((void**)&env, JNI_VERSION_1_6) != JNI_OK) {
                return JNI_ERR;
            }
        
            // 查找对应的Java类
            jclass clazz = env->FindClass("com/example/myapp/MyNativeLib");
            if (clazz == nullptr) {
                return JNI_ERR;
            }
        
            // 注册Native方法
            // RegisterNatives(类引用, 方法数组, 方法数量)
            if (env->RegisterNatives(clazz, gMethods, sizeof(gMethods) / sizeof(gMethods[0])) < 0) {
                return JNI_ERR;
            }
        
            env->DeleteLocalRef(clazz); // 释放局部引用
        
            return JNI_VERSION_1_6; // 返回支持的JNI版本
        }
        
        // JNI_OnUnload (可选): 当.so库被卸载时调用，可以进行一些清理工作
        // extern "C" JNIEXPORT void JNICALL JNI_OnUnload(JavaVM* vm, void* reserved) {
        //     // 清理资源
        // }
        ```
        
        - **Java 方法签名说明：**
            - `()`：表示无参数。
            - `Ljava/lang/String;`：表示 `java.lang.String` 类型。
            - `I`：表示 `int`。
            - `Z`：`boolean`
            - `B`：`byte`
            - `S`：`short`
            - `J`：`long`
            - `F`：`float`
            - `D`：`double`
            - `[Ljava/lang/String;`：表示 `String[]` 数组。
            - 更多签名请查阅 JNI 官方文档。

**在 Android 开发中，推荐使用**动态注册**方式，因为它更灵活、更高效，并且代码维护性更好。**