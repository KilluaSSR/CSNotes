
## 逆向工程与代码分析

在没有源代码的情况下，我们转向分析反汇编的代码指令，也就是汇编代码分析。这种更深入的理解帮助我们揭示那些即使在静态和动态分析后仍然隐藏或难以发现的功能。

**反汇编器** 是我们进行静态分析时的首选工具，我们无需执行代码。它帮助我们在不激活潜在有害功能的情况下理解代码的结构和逻辑。一些常见的反汇编器包括 IDA、Cutter 和 Ghidra。

**调试器** 则像反汇编器一样，将机器代码解码为汇编指令。此外，它还允许我们以受控的方式执行代码，逐条指令执行，跳转到特定位置，或在指定的断点处暂停执行流程。调试器的例子包括 x32dbg、x64dbg、IDA 和 OllyDbg。

代码从人类可读的高级语言（如 C 或 C++）到机器代码的过程是单向的，由编译器引导。机器代码是一种计算机直接处理的二进制语言，它对人类来说是难以理解的。此时，汇编语言发挥了桥梁作用，它连接了我们与机器代码，使我们能够解码后者。

**反汇编器** 将机器代码转回为汇编语言，呈现给我们一系列可读的指令。

**代码分析** 是指仔细审查和解码已编译程序或二进制文件的行为和功能的过程。这涉及分析代码中的指令、控制流和数据结构，最终揭示出程序的目的、功能以及可能的攻击指示符（IOCs）。

![[Pasted image 20250506064505.png]]

反汇编器将二进制代码转换为相应的汇编指令，并通常通过附加的上下文信息来补充，例如内存地址、函数名称和控制流分析。IDA 是其中一款强大的工具，它被广泛使用并因其先进的分析功能而备受推崇。它支持多种可执行文件格式和架构，提供全面的反汇编视图和强大的分析功能。

## 示例

接下来，我们继续 shell.exe 恶意软件样本。到目前为止，我们已经发现它执行了沙箱检测（见 恶意软件的简单分析 一节），并且包含一个可能的延时机制——在执行其预定操作之前，会有一个 5 秒的 ping 延迟。

**将恶意样本导入 IDA**

在调查的下一阶段，我们需要在 IDA 中仔细审查代码，以确定其后续操作，并找出如何绕过恶意样本使用的沙箱检测。

选择“New”（新建），然后选择 shell.exe 样本进行分析。点击“OK”后，IDA 会将可执行文件加载到内存中，并对机器代码进行反汇编，将反汇编结果呈现给我们。下面的截图展示了 IDA 中的不同视图。

![[Pasted image 20250506064752.png]]

当可执行文件加载完成并且分析完成后，样本 shell.exe 的反汇编代码将显示在 IDA 的主视图窗口中。我们可以使用光标键或滚动条浏览代码，也可以通过鼠标滚轮或缩放控件来放大或缩小视图。

**文本视图和图形视图**

反汇编的代码以两种模式呈现，即图形视图和文本视图。默认视图是图形视图，它提供了函数基本块及其相互连接的图形化表示。基本块是指具有单一入口和出口的指令序列。这些基本块在图形视图中以节点的形式表示，而它们之间的连接则以边的形式呈现。

要在图形视图和文本视图之间切换，**只需按空格键**。

图形视图为程序的控制流提供了一个图形化的表示，帮助更好地理解执行流程，识别循环、条件语句和跳转，并可视化程序如何在不同的代码路径之间分支或循环。

![[Pasted image 20250506064847.png]]

在图形视图中，函数作为节点显示。每个函数都以一个独特的标识符和附加细节（如函数名称、地址和大小）来表示。

文本视图则显示汇编指令及其对应的内存地址。在文本视图中的每一行代表代码中的一条指令或数据元素，行首是节名称:虚拟地址格式（例如，`.text:00000000004014F0`，其中节名称是 `.text`，虚拟地址是 `00000000004014F0`）。

![[Pasted image 20250506064944.png]]

在这个例子中，`start` 函数是一个公开的函数，它位于地址 `00000000004014F0`。代码段开始于 `sub rsp, 28h` 指令，并继续显示多个指令。

IDA 的文本视图使用箭头来表示不同类型的控制流指令和跳转。

**实心箭头 (→)**: 实心箭头表示直接跳转或分支指令，指示程序流的无条件转移，执行从一个位置移动到另一个位置。这通常出现在像 `jmp` 或 `call` 这样的跳转或调用指令中。

**虚线箭头 (---→)**: 虚线箭头表示条件跳转或分支指令，暗示程序的执行流可能根据特定条件发生变化。跳转的目标取决于条件的结果。例如，`jz`（零标志跳转）指令只有在之前的比较结果为零时才会触发跳转。

这些箭头帮助分析人员快速识别和理解程序的控制流，识别无条件或有条件的跳转，进一步揭示程序的执行路径。

![[Pasted image 20250506065120.png]]

默认情况下，IDA 会首先显示程序的主函数或指定入口点处的函数。然而，我们可以自由探索并在图形视图中检查其他函数。

**在 IDA 中识别主函数**

以下截图展示了 `start` 函数，这是程序的入口点，通常负责在调用实际的主函数之前设置运行时环境。这是加载可执行文件后，IDA 初始显示的 `start` 函数。

在许多程序中，`start` 函数通常是程序执行的第一步，它可能会做一些初始化工作，如设置堆栈、分配内存或执行其他启动过程，随后才会跳转到程序的实际主函数（例如 `main` 函数）。通过分析这个入口点，我们可以进一步理解程序的启动流程。

![[Pasted image 20250506065155.png]]

我们的目标是定位实际的主函数，这需要我们进一步探索反汇编代码。我们将搜索调用或跳转指令，这些指令指向其他函数，其中一个很可能是主函数。IDA 的图形视图、交叉引用或函数列表可以帮助我们在反汇编中进行导航，并识别主函数。

然而，在找到主函数之前，我们首先需要理解 `start` 函数的功能。这个函数主要包含一些初始化代码、异常处理和函数调用。它最终跳转到 `loc_40150C` 标签，这是一个异常处理程序。因此，我们可以推断，`start` 函数不是程序逻辑所在的实际主函数。我们将检查其他函数调用，以找出实际的主函数。

**代码分析：**

代码首先通过从 `rsp`（栈指针）寄存器中减去 0x28（十进制的40），有效地在栈上为局部变量创建空间，并保存了之前的栈内容。

```asm
public start
start proc near

; FUNCTION CHUNK AT .text:00000000004022A0 SIZE 000001B0 BYTES

; __unwind { // __C_specific_handler
sub     rsp, 28h
```

上面截图中的中间代码块表示一个使用结构化异常处理（SEH）的异常处理机制。`__try` 和 `__except` 关键字表明设置了一个异常处理块。在这个块中，随后的 `call` 指令分别调用了两个子例程（函数） `sub_401650` 和 `sub_401180`。这些是 IDA 自动生成的占位符名称，用来表示子例程、程序位置和数据。自动生成的名称通常会有以下前缀，并附上相应的虚拟地址：`sub_<virtual_address>` 或 `loc_<virtual_address>` 等。

```
loc_4014F4:
;   __try { // __except at loc_40150C
mov     rax, cs:off_405850
mov     dword ptr [rax], 0
call    sub_401650         ; 我们将检查这个函数
call    sub_401180         ; 我们将检查这个函数
nop
;   } // starts at 4014F4
```

```
-----------------------------------------------

loc_40150C:
;   __except(TopLevelExceptionFilter) // owned by 4014F4
nop
add     rsp, 28h
retn
; } // starts at 4014F0
start endp
```

**在 IDA 中浏览函数**

现在，让我们通过导航到每个函数并查看反汇编代码，检查 `sub_401650` 和 `sub_401180` 这两个函数的内容。

![[Pasted image 20250506065445.png]]

我们将首先打开第一个函数/子例程 `sub_401650`。在 IDA 的反汇编视图中，要进入一个函数，首先将光标放在表示该函数调用（或跳转指令）的指令上，然后右键点击该指令，从上下文菜单中选择“Jump to Operand”选项。或者，我们也可以直接按下键盘上的 Enter 键。

这样，我们就可以跳转到 `sub_401650` 函数的起始位置，并开始查看该函数的反汇编代码。

![[Pasted image 20250506065453.png]]
接下来，IDA 会引导我们到跳转或函数调用的目标位置，带我们进入被调用函数的起始位置或跳转的目标位置。

现在，我们已经进入了第一个子例程 `sub_401650`，让我们努力理解它，以确定它是否是主函数。如果不是主函数，我们将继续导航通过其他函数，并辨别最终调用主函数的部分。

![[Pasted image 20250506065517.png]]

在这个子例程 `sub_401650` 中，我们可以看到对多个函数的调用，如 `GetSystemTimeAsFileTime`、`GetCurrentProcessId`、`GetCurrentThreadId`、`GetTickCount` 和 `QueryPerformanceCounter`。这样的模式通常出现在反汇编代码的开头，通常用于设置初始的栈帧并执行一些与系统相关的初始化任务。

这些调用通常用于获取系统时间、当前进程 ID、当前线程 ID、系统启动时间以及性能计数器等信息。它们的存在表明该子例程可能在执行一些与系统环境相关的初始化工作，而不是实现程序的主要逻辑。因此，`sub_401650` 可能并不是主函数，我们还需要进一步探索其他子例程。这里详细描述的指令类型通常出现在针对 x86/x64 架构的编译器生成的可执行代码中。当可执行文件被加载并运行时，操作系统负责为程序准备执行环境。这个过程包括栈的设置、寄存器初始化和系统相关数据结构的准备等任务。

广义上来说，这段代码是初始化执行环境的一部分，执行必要的与系统相关的初始化任务，为程序的主要逻辑执行做准备。其目的是确保程序在一致的状态下启动，能够访问到所需的系统资源和信息。需要明确的是，这不是程序的主逻辑所在，因此我们需要继续探索其他函数调用，以便找到主函数。

现在，让我们返回并打开第二个子例程 `sub_401180`，以检查它的内容。

要返回到我们之前正在检查的函数，我们可以按键盘上的 Esc 键，或者点击工具栏中的“Jump Back”按钮。这样我们就能回到之前的函数，继续进行分析。

![[Pasted image 20250506065845.png]]

![[Pasted image 20250506065900.png]]

**检查 `sub_401180`**

在检查时，我们可以观察到，该函数似乎涉及初始化 `StartupInfo` 结构体并根据其值执行某些检查。`rep stosq` 指令会清除一块内存，而后续的指令则修改寄存器的内容并执行基于寄存器值的条件跳转。这些操作看起来不像是程序的主逻辑所在的函数，但它确实包含一些调用指令，这些调用可能会引导我们找到主函数。因此，我们需要进一步调查所有在此函数返回之前的调用指令。

![[Pasted image 20250506070121.png]]

**继续探索函数的末尾**

我们需要滚动到此函数的终点，并从最底部开始搜索调用指令。

当我们从此块的结束部分向上滚动时，我们发现，在此函数返回之前，有一条调用指令，调用了另一个子例程 `sub_403250`。接下来，我们将继续分析这个新的子例程，看看它是否能将我们引导到程序的主函数。

![[Pasted image 20250506070204.png]]

在审查这些指令后，我们可以看到该函数正在查询注册表中与 `SOFTWARE\\VMware, Inc.\\VMware Tools` 路径相关的值，并执行比较操作以判断是否在计算机上安装了 VMware Tools。一般来说，看来这很可能就是我们在进程监视器和字符串中看到的主函数。

我们可以观察到，注册表查询是通过 `RegOpenKeyExA` 函数完成的，如下面反汇编代码中的 `call cs:RegOpenKeyExA` 指令所示：

```
xor     r8d, r8d        ; ulOptions
mov     [rsp+148h+cbData], 100h
mov     [rsp+148h+phkResult], rax ; phkResult
mov     r9d, 20019h     ; samDesired
lea     rdx, aSoftwareVmware ; "SOFTWARE\\VMware, Inc.\\VMware Tools"
mov     rcx, 0FFFFFFFF80000002h ; hKey
call    cs:RegOpenKeyExA
```

在上面的代码块中，最后一条指令 `call cs:RegOpenKeyExA` 很可能代表了对 `RegOpenKeyExA` 函数的调用，前缀 `cs` 表示它是由代码段寄存器引用的。`RegOpenKeyExA` 是 Windows 注册表 API 的一部分，用于打开指定注册表项的句柄。这使得程序能够访问 Windows 注册表。函数名中的 A 表示它是 ANSI 版本的函数，操作的是 ANSI 编码的字符串。

在 IDA 中，`cs` 是一个段寄存器，通常指向代码段。当我们点击 `cs:RegOpenKeyExA` 并按下 Enter 键时，IDA 会将我们带到 `.idata` 区段，那里包含了与导入相关的数据以及 `RegOpenKeyExA` 函数的导入地址。在这种情况下，`RegOpenKeyExA` 函数是从外部库（`advapi32.dll`）导入的，并且它的地址存储在 `.idata` 区段中，供后续使用。

这一系列操作表明，程序正通过访问注册表来确认是否安装了 VMware Tools，从而实现沙箱或虚拟机的检测。考虑到这个功能的性质，`sub_403250` 很可能是程序的主函数或与主程序逻辑密切相关的部分。

![[Pasted image 20250506070339.png]]

这并不是 `RegOpenKeyExA` 函数的实际地址，而是 `RegOpenKeyExA` 在 IAT（导入地址表）中的条目的地址。IAT 条目保存了一个地址，这个地址将在运行时动态解析，指向实际的函数实现（在这个案例中是 `advapi32.dll` 中的实现）。

在反汇编代码中，`extrn RegOpenKeyExA:qword` 表示 `RegOpenKeyExA` 是一个外部符号，需要在运行时解析。这告诉汇编器该函数定义在另一个模块或库中，链接器将在链接过程中处理其地址的解析。参考资料：[Microsoft - PE格式的IAT](https://learn.microsoft.com/en-us/windows/win32/debug/pe-format#import-address-table)

实际上，`cs:RegOpenKeyExA` 是一种通过相对引用访问 IAT 条目中 `RegOpenKeyExA` 的方式。`RegOpenKeyExA` 的实际地址将在运行时由操作系统的动态链接器/加载器解析并存储在 IAT 中。

根据这个函数的整体结构，我们可以推测这可能是主函数。为了方便记忆，建议将其重命名为 `assumed_Main`，以便将来遇到对该函数的引用时能更容易识别。

在 IDA 中重命名函数的步骤如下：

1. 将光标定位在函数名（`sub_403250`）或包含函数定义的行上。
    
2. 按下键盘上的 `N` 键，或者右键点击并从上下文菜单中选择 `Rename`。
    
3. 输入新函数名（例如 `assumed_Main`），然后按 Enter 键。
    
4. IDA 会在整个反汇编视图以及二进制文件中的任何引用处更新该函数的名称。
    

在 IDA 中重命名函数不会修改实际的二进制文件，它仅仅改变了 IDA 分析视图中的表示。

![[Pasted image 20250506070601.png]]

我们可以看到，在调用 Windows API 函数 `RegOpenKeyExA` 之前，这个函数有两个子函数调用（`sub_401610` 和 `sub_403110`）。在继续分析 WINAPI 函数之前，我们先检查这两个子函数。

通过将光标定位到各自的调用指令上，然后按下 Enter 键，我们可以进入这些子函数进行查看。

首先进入第一个子函数 `sub_401610` 的反汇编代码。

![[Pasted image 20250506070619.png]]

我们进入了第一个子程序 `sub_401610`，它将变量（`cs:dword_408030`）赋值为一。然后，它跳转到 `sub_4015A0`。

![[Pasted image 20250506070709.png]]

它首先将变量 `dword_408030` 的值转移到 `eax` 寄存器中。然后，它对 `eax` 执行按位与操作（AND），本质上是在检查该值是否为零。如果前面的测试指令结果表明 `eax` 为零，则跳转到 `sub_4015A0`。让我们进一步分析它的代码。

我们发现该函数首先将 `rsi` 和 `rbx` 寄存器的值推入堆栈，保留了这些寄存器的值。接着，它通过从栈指针（`rsp`）中减去 28h（40 十进制）字节来为栈分配空间。然后，它从地址 `off_405730` 中获取一个函数指针，并将其存入 `rax` 寄存器。

本质上，这些操作似乎是在执行与函数指针相关的初始化检查和操作，然后程序继续调用第二个子程序 `sub_403110` 和用于注册表操作的 WINAPI 函数。这并不是程序逻辑所在的主函数，因此我们将继续分析其他函数调用，找出实际的主函数。

为了方便记忆，我们可以将此函数重命名为 `initCheck`，按下 N 键并输入新函数名称。

此时，我们可以按 Esc 键或点击工具栏上的“跳转回”按钮，以返回到第二个子程序 `sub_403110` 并探索它的内部逻辑。

![[Pasted image 20250506071800.png]]

![[Pasted image 20250506071854.png]]

变量 `Parameters`、`File` 和 `Operation` 是字符串变量，存储在可执行文件的 `.rdata` 段中。`lea` 指令被用来获取这些字符串的内存地址，并将这些地址作为参数传递给 `ShellExecuteA` 函数。这段代码负责使程序休眠 5 秒钟。之后，程序会返回到前一个函数。理解了这段代码后，我们可以通过右键点击并选择重命名，将此函数重命名为 `pingSleep`。

现在，我们已经遇到了一些 Windows API 函数的引用，接下来我们将阐明如何在反汇编代码中理解 WINAPI 函数。

在分析了函数调用（`sub_401610` 和 `sub_403110`）中的操作后，在调用 Windows API 函数 `RegOpenKeyExA` 之前，我们来检查对 `RegOpenKeyExA` 函数的调用。在 IDA 的反汇编视图中，传递给 WINAPI 函数调用的参数会显示在调用指令的上方。这种标准约定在反汇编工具中提供了清晰的表示，显示了函数调用及其对应的参数。

Windows API 函数 `RegOpenKeyExA` 在此用于打开一个注册表项。根据 Microsoft 文档，该函数的语法如下：

```cpp
LSTATUS RegOpenKeyExA(
  [in]           HKEY   hKey,
  [in, optional] LPCSTR lpSubKey,
  [in]           DWORD  ulOptions,
  [in]           REGSAM samDesired,
  [out]          PHKEY  phkResult
);
```

```asm
lea     rax, [rsp+148h+hKey]      ; Calculate the address of hKey
xor     r8d, r8d                  ; Clear r8d register (ulOptions)
mov     [rsp+148h+phkResult], rax ; Store the calculated address of hKey in phkResult
mov     r9d, 20019h               ; Set samDesired to 0x20019h (which is KEY_READ in MS-DOCS)
lea     rdx, aSoftwareVmware      ; Load address of string "SOFTWARE\\VMware, Inc.\\VMware Tools"
mov     rcx, 0FFFFFFFF80000002h   ; Set hKey to 0xFFFFFFFF80000002h (HKEY_LOCAL_MACHINE)
call    cs:RegOpenKeyExA          ; Call the RegOpenKeyExA function
test    eax, eax                  ; Check the return value
jnz     short loc_40330F          ; Jump if the return value is not zero (error condition)
```

`lea` 指令计算了 `hKey` 变量的地址，这个地址很可能是一个注册表键的句柄。接着，`mov rcx, 0FFFFFFFF80000002h` 将 `HKEY_LOCAL_MACHINE` 作为第一个参数传递给函数（存储在 `rcx` 寄存器中）。`lea rdx, aSoftwareVmware` 指令使用加载有效地址（LEA）操作，计算存储字符串 "Software\VMware, Inc.\VMware Tools" 的内存位置的有效地址，并将这个地址存储在 `rdx` 寄存器中，作为函数的第二个参数。

该函数的第三个参数通过 `xor r8d, r8d` 指令传递，`xor` 操作将 `r8d` 寄存器清零，表示将 `ulOptions` 设置为 0。

第四个参数通过 `mov r9d, 20019h` 设置为 `KEY_READ`（对应 MS-DOCS 中的值）。

第五个参数 `phkResult` 位于堆栈上。通过将 `rsp+148h` 加到栈指针 `rsp`，代码访问了堆栈上 `phkResult` 参数所在的内存位置。`mov [rsp+148h+phkResult], rax` 指令将 `rax` 寄存器中的值（即 `hKey` 的地址）复制到 `phkResult` 指针指向的内存位置，从而将 `hKey` 的地址存储在 `phkResult` 中，作为下一个函数的第一个参数。

从现在开始，每当我们在代码中遇到 Windows API 函数的调用时，我们会参考 Microsoft 文档，了解该函数的语法、参数和返回值。这将帮助我们理解在调用这些函数时寄存器中的值。

如果我们继续向下滚动图视图，将会遇到下一个 Windows API 函数 `RegQueryValueExA`，该函数用于检索与打开的注册表键相关的值名称的数据。数据会进行比较，如果匹配，将弹出一个显示 "Sandbox Detected" 的消息框。如果不匹配，则会跳转到另一个子程序 `sub_402EA0`。我们稍后将在调试器中修复此沙盒检测问题。

![[Pasted image 20250506074652.png]]
 
 `sub_402EA0`
 
![[Pasted image 20250506074758.png]]

这个子程序似乎执行与网络相关的操作，使用了 Windows 套接字 API（Winsock）。它首先调用 `WSAStartup` 函数来设置 Winsock 库，然后调用 `getaddrinfo` 函数，该函数用于根据提供的提示 `pHints` 获取指定节点名称 (`pNodeName`) 的地址信息。子程序通过 `getaddrinfo` 函数验证地址解析的成功与否。

如果 `getaddrinfo` 函数返回值为零（表示成功），这意味着地址已经成功解析为 IP。成功后，如果确实成功，程序流程会跳转到一个显示 "Sandbox detected" 的消息框。如果失败，它会将流程引导到子程序 `sub_402D00`。

随后，它会调用 `WSACleanup` 函数。无论地址解析过程是否成功，这个操作都会启动 Winsock 相关资源的清理。为了清晰起见，我们将此函数命名为 `DomainSandboxCheck`。

**可能的 IOC:** 请注意域名 `iuqerfsodp9ifjaposdfjhgosurijfaewrwergwea[.]com` 作为潜在 IOC 的一部分。

为了探究绕过沙箱检查的后果，我们将深入了解子程序 `sub_402D00`。我们可以通过点击与 `sub_402D00` 函数相关的调用指令，按下 Enter 键来检查该子程序。下图显示了该函数的反汇编代码。

![[Pasted image 20250506075503.png]]

这个函数首先为局部变量在堆栈上保留空间，然后调用了 `sub_402C20`，这是一个独立的函数。该函数的输出结果会存储在 `eax` 寄存器中。根据从 `sub_402C20` 函数得到的结果，程序要么返回（`retn`），要么跳转到 `sub_402D20`。

因此，我们将选择第一个高亮显示的函数 `sub_402C20`，通过按 Enter 键来查看它的指令。仔细分析 `sub_402C20` 后，我们将返回到这个代码块，继续评估第二个高亮显示的函数 `sub_402D20`。

![[Pasted image 20250506075550.png]]
按下 Enter 键后，我们看到该函数的指令，如上图所示。此函数初始化 Winsock 库，生成一个套接字，并通过端口 31337 连接到 IP 地址 45.33.32.156。它评估返回值（eax）以确定连接是否成功。然而，这里有一个转折；在函数调用后，`inc eax` 指令将 eax 寄存器的值加 1。紧接着，在 `inc eax` 指令后，代码通过 `jnz`（若非零则跳转）指令评估 eax 的值。

如果连接到上述 IP 地址和端口失败，这个函数应当返回 -1，按照文档的说明。

![[Pasted image 20250506075741.png]]

```asm
call    cs:connect
inc     eax
jnz     short loc_402CD0
```

由于在函数调用后 eax 被加 1，这应当使 eax 的值增加到 0。因此，MessageBox 会显示 "Sandbox detected"。这意味着该函数正在检查互联网连接的状态。
![[Pasted image 20250506080223.png]]
如果连接成功，它将产生一个非零值，导致代码跳转到 loc_402CD0。该位置包含对另一个函数 sub_402F40 的调用。了解该函数的操作后，我们将其重命名为 `InternetSandboxCheck`。

可能的 IOC：请记住将此 IP 地址 45.33.32.156 和端口 31337 作为潜在 IOC 的组成部分。

接下来，我们将继续分析函数 sub_402F40 的操作。

![[Pasted image 20250506080249.png]]

该函数调用了 `getenv` 函数（rcx 作为传递 "TEMP" 参数的寄存器），并将结果保存在 eax 寄存器中。这个操作用于获取 TEMP 环境变量的值。

```assembly
lea     rcx, VarName    ; "TEMP"
call    getenv
```

为了验证输出，我们可以使用 PowerShell 打印 TEMP 环境变量的值。

```powershell
PS C:\> Get-ChildItem env:TEMP

Name                           Value
----                           -----
TEMP                           C:\Users\Killua\AppData\Local\Temp
```

然后，它使用 `sprintf` 函数将获取到的 TEMP 路径与字符串 `svchost.exe` 拼接，形成完整的文件路径。接着，调用 `GetComputerNameA` 函数来获取计算机名称，并将其存储在一个缓冲区中。

如果计算机名称不存在，它会跳过到标签 `loc_4030F8`（该位置包含返回指令）。相反，如果计算机名称非空（非零值），代码将继续执行左侧图像中显示的后续指令。

![[Pasted image 20250506080413.png]]在随后的指令中，我们发现了对 `sub_403220` 函数的调用。它格式化了一个字符串，该字符串包含自定义的用户代理值，格式为 `Windows-Update/7.6.7600.256 %s`。其中 `%s` 占位符被之前获取到的计算机名称替换，该计算机名称作为参数传递给此函数，并存储在 `rcx` 寄存器中。
![[Pasted image 20250506080552.png]]

现在，完整的值为 `Windows-Update/7.6.7600.256 HOSTNAME`，其中 `HOSTNAME` 是通过 `GetComputerNameA` 获取到的计算机名称。

需要特别注意的是这个独特的自定义用户代理，其中计算机名称也会在恶意软件发起网络连接时作为请求的一部分发送。

回到之前的函数，它接着调用了 `InternetOpenA` WINAPI 函数来启动一个互联网访问会话，并配置 `InternetOpenUrlA` 函数的参数。随后，它调用 `InternetOpenUrlA` 来打开 URL `http://ms-windows-update.com/svchost.exe`。

**可能的 IOC**：请注意该 URL 作为潜在的 IOC。恶意软件正从此位置下载另一个可执行文件。

如果 URL 成功打开，代码将跳转到 `loc_40301E` 标签。让我们通过双击它来检查 `loc_40301E` 位置的指令。

![[Pasted image 20250506080724.png]]

在打开该函数后，我们观察到它调用了 Windows API 函数 `CreateFileA`，该函数用于在本地系统上创建文件，指定了先前获得的文件路径。

接着，代码进入一个循环，反复调用 `InternetReadFile` 函数从已打开的 URL 中拉取数据。如果数据读取操作成功，代码便会继续使用 `WriteFile` 函数将接收到的数据写入到创建的文件（即位于 TEMP 目录下的 svchost.exe 文件）中。

请注意这种技术，恶意软件通过这种方式下载并将一个可执行文件 `svchost.exe` 存储在临时目录中。

![[Pasted image 20250506080826.png]]

在数据写入操作完成后，代码会循环回去继续读取更多数据，直到 `InternetReadFile` 函数返回一个指示数据流结束的值。所有数据读取并写入完成后，通过适当的函数（`CloseHandle` 和 `InternetCloseHandle`）关闭打开的文件和互联网句柄。接着，代码跳转到 `loc_4030D3`，并调用函数 `sub_403190`。

![[Pasted image 20250506080908.png]]
函数 `sub_403190` 揭示出一系列与注册表修改相关的 WINAPI 调用，例如 `RegOpenKeyExA` 和 `RegSetValueExA`。

![[Pasted image 20250506080915.png]]
这个函数似乎将文件（位于TEMP目录中的svchost.exe）放入注册表键路径 `SOFTWARE\Microsoft\Windows\CurrentVersion\Run`，并将值名称设置为 `WindowsUpdater`，然后关闭注册表键。这种技术常被恶意软件和合法应用程序用来保持在系统中的持久性，确保在每次系统启动或用户登录时自动运行。为了清晰起见，我们已在IDA中将此函数重命名为 `persistence_registry`。

以下是显示注册表键路径 `SOFTWARE\Microsoft\Windows\CurrentVersion\Run` 和值名称 `WindowsUpdater` 的汇编代码。

![[Pasted image 20250506080935.png]]

可能的IOC：请注意这种技术，恶意软件通过修改注册表实现持久性。它通过在 `SOFTWARE\Microsoft\Windows\CurrentVersion\Run` 注册表键下添加 `WindowsUpdater` 名称的条目来实现。

在建立注册表后，函数调用了另一个函数 `sub_403150`，该函数启动了已下载的文件 `svchost.exe` 并传递了一个参数。通过简单的Google搜索，我们推测这个参数可能是一个比特币钱包地址。因此，可以合理推测被下载的可执行文件可能是一个挖矿程序。


![[Pasted image 20250506080957.png]]

![[Pasted image 20250506081208.png]]

打开该子程序后，我们可以看到它正在为 `CreateProcessA` 函数设置必要的参数，以启动一个新的进程。接着，它会启动一个新的进程，`notepad.exe`，该程序位于 `C:\Windows\System32` 目录下。

以下是 `CreateProcessA` 函数的语法：

```ida
BOOL CreateProcessA(
  [in, optional]      LPCSTR                lpApplicationName,
  [in, out, optional] LPSTR                 lpCommandLine,
  [in, optional]      LPSECURITY_ATTRIBUTES lpProcessAttributes,
  [in, optional]      LPSECURITY_ATTRIBUTES lpThreadAttributes,
  [in]                BOOL                  bInheritHandles,
  [in]                DWORD                 dwCreationFlags,
  [in, optional]      LPVOID                lpEnvironment,
  [in, optional]      LPCSTR                lpCurrentDirectory,
  [in]                LPSTARTUPINFOA        lpStartupInfo,
  [out]               LPPROCESS_INFORMATION lpProcessInformation
);
```

在 `CreateProcessA` 函数文档中，我们注意到非零返回值表示函数执行成功。因此，如果执行成功，它将不会跳转到 `loc_402E89`，而是会继续执行下一段指令。

![[Pasted image 20250506081334.png]]


从代码中的 `rdx` 寄存器可以看出，传递给该函数的第二个参数被确定为 `C:\\Windows\\System32\\notepad.exe`。

接下来的指令块暗示了一个常见的进程注入方式，即通过 `VirtualAllocEx`、`WriteProcessMemory` 和 `CreateRemoteThread` 函数将 shellcode 插入到新创建的进程中。

让我们根据对代码的观察来解读这一过程注入的细节。

首先，使用 `CreateProcessA` 函数创建了一个新的 `notepad.exe` 进程。随后，通过 `VirtualAllocEx` 在该进程中分配内存。接着，使用 `WriteProcessMemory` 函数将 shellcode 写入到新分配的内存中。最后，使用 `CreateRemoteThread` 函数在 `notepad.exe` 中创建一个远程线程，并启动 shellcode 的执行。

如果注入成功，将弹出一个消息框，显示“Connection sent to C2”。相反，如果注入失败，则会弹出错误消息。


![[Pasted image 20250506081413.png]]

![[Pasted image 20250506081459.png]]

为了简化分析，我们将函数 `sub_402D20` 重命名为 `process_Injection`。

在这个函数的开头，我们可以看到一个未知地址 `unk_405057`，其有效地址通过指令 `lea rsi, unk_405057` 加载到 `rsi` 寄存器中。在调用进行进程注入的 WINAPI 函数之前，加载有效地址到 `rsi` 寄存器的原因可能有多种——它可能作为数据访问指针，或者作为函数调用的参数。然而，也有可能这个地址存储了潜在的 shellcode。我们将在调试这些 WINAPI 函数时，使用像 x64dbg 这样的调试器来验证这一点。

![[Pasted image 20250506081623.png]]

![[Pasted image 20250506081701.png]]