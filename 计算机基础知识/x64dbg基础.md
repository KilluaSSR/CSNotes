

> 调试为代码分析增加了一层动态、交互式的功能，提供了恶意软件行为的实时视图。它让分析人员能够确认自己的发现，观察运行时的影响，并加深对程序执行的理解。将代码分析与调试结合，可以全面了解恶意软件，从而有效暴露其有害行为。
> 
> 我们可以使用像x64dbg这样的调试器，它是一个专为分析和调试64位Windows可执行文件设计的用户友好工具。它配有图形界面，用于可视化反汇编代码、设置断点、检查内存和寄存器，并控制程序的执行。

1. **观察程序的执行：**
    
    - 在x64dbg中加载可执行文件后，程序将在入口点处停住，等待你进一步操作。
        
    - 你可以在这个时刻检查内存、寄存器和其他调试信息。通过x64dbg的界面，查看寄存器的内容，看看程序的初始状态。
        
2. **设置断点：**
    
    - 若想深入分析程序的特定部分，可以设置断点。右键点击你感兴趣的指令或地址，然后选择“Toggle Breakpoint”来设置断点。
        
    - 程序会在执行到该断点时暂停，允许你观察内存、寄存器变化并检查程序执行的路径。
        
3. **单步执行：**
    
    - 在程序暂停后，你可以使用“Step Into” (F7) 或 “Step Over” (F8) 来逐行执行程序。这样可以细致观察每一条指令的执行效果。
        
    - “Step Into” 会进入调用的函数，而 “Step Over” 会跳过函数调用，直接执行到下一条指令。
        
4. **查看内存和寄存器：**
    
    - 通过x64dbg的窗口查看内存内容，检查是否有恶意代码的迹象。
        
    - 你还可以查看和修改寄存器的值，检查程序运行时的状态，帮助你更好地理解程序行为。
        
5. **观察程序的行为：**
    
    - 继续调试程序，留意任何可疑的行为或特征，例如网络连接、文件操作、异常退出等。
        
    - 通过分析程序的执行过程，可以逐步揭示它可能的恶意行为或漏洞。
        

![[Pasted image 20250507062822.png]]

加载可执行文件到x64dbg后，界面会显示反汇编视图，展示程序的汇编指令，从而帮助理解代码流程。右侧是寄存器窗口，显示CPU寄存器的值，有助于了解程序的当前状态。在寄存器窗口下方，堆栈视图显示当前的堆栈帧，可以检查函数调用和局部变量。最后，在左下角是内存转储视图，提供程序内存的图像化表示，便于分析数据结构和变量。

## **模拟互联网服务**

**INetSim** 在我们受限的测试环境中模拟典型的互联网服务方面起着关键作用。它支持多种服务，包括 **DNS**、**HTTP**、**FTP**、**SMTP** 等。我们可以对其进行微调以重现特定响应，从而能够更针对性地检查恶意软件的行为。我们的方法是保持 INetSim 运行，以便它可以拦截来自恶意软件样本（shell.exe）的任何 DNS、HTTP 或其他请求，从而为其提供受控的、合成的响应。

```bash
sudo nano /etc/inetsim/inetsim.conf
```

需要取消注释并指定以下内容。

```conf
service_bind_address <本地机器IP地址127.0.0.1>
dns_default_ip <本地机器IP地址127.0.0.1>
dns_default_hostname www
dns_default_domainname iuqerfsowergwea.com
```

`iuqerfsowergwea.com`是恶意软件尝试连接的`url`

```bash
sudo inetsim
```

```
INetSim 1.3.2 (2020-05-19) by Matthias Eckert & Thomas Hungenberg
Using log directory:      /var/log/inetsim/
Using data directory:     /var/lib/inetsim/
Using report directory:   /var/log/inetsim/report/
Using configuration file: /etc/inetsim/inetsim.conf
Parsing configuration file.
Configuration file parsed successfully.
=== INetSim main process started (PID 34711) ===
Session ID:     34711
Listening on:   0.0.0.0
Real Date/Time: 2023-06-11 00:18:44
Fake Date/Time: 2023-06-11 00:18:44 (Delta: 0 seconds)
 Forking services...
  * dns_53_tcp_udp - started (PID 34715)
  * smtps_465_tcp - started (PID 34719)
  * pop3_110_tcp - started (PID 34720)
  * smtp_25_tcp - started (PID 34718)
  * http_80_tcp - started (PID 34716)
  * ftp_21_tcp - started (PID 34722)
  * https_443_tcp - started (PID 34717)
  * pop3s_995_tcp - started (PID 34721)
  * ftps_990_tcp - started (PID 34723)
 done.
Simulation running.
```

[有关配置 INetSim 的更详细教程如下](https://medium.com/@xNymia/malware-analysis-first-steps-creating-your-lab-21b769fb2a64))

最后，被分析的目标（shell.exe 运行的环境）的 DNS 应指向**运行 INetSim 的本地机器/虚拟机的 IP 地址**（例如 127.0.0.1 或其自身的局域网 IP）。需要在虚拟机的网络适配器设置中手动配置 DNS 服务器地址。

![[Pasted image 20250507063931.png]]

## 规避沙箱检测

鉴于沙箱检查会阻碍恶意软件在机器上直接执行，我们需要通过打补丁来规避沙箱检测。在使用 **x64dbg** 进行调试时，有几种方法可以引导我们找到执行沙箱检测的指令。我们将讨论其中一些方法。

**通过从 IDA 复制地址**

在用IDA进行代码分析过程中，我们观察到了与注册表键相关的沙箱检测检查。我们可以直接从 **IDA** 中提取第一个 `cmp` 指令的地址。

要找到该地址，让我们回到 IDA 窗口，打开我们之前重命名为 `assumed_Main` 的第一个函数，并寻找 `cmp` 指令。要查看地址，可以按空格键从图形视图切换到文本视图。

我们可以从 IDA 复制地址 `00000000004032C8`。
```
.text:00000000004032C8                 cmp     [rsp+148h+Type], 1
```

![[Pasted image 20250507064114.png]]
在 **x64dbg** 中，我们可以右键单击反汇编视图（CPU）中的任意位置，然后选择 **Go to** > **Expression**。或者，可以使用快捷键 **Ctrl+G**（跳转到表达式）。

**通过搜索字符串**

让我们在 **String references** 中查找 `Sandbox detected` 字符串，并设置一个**断点**，这样当我们点击“运行”时，执行会在此时暂停。

![[Pasted image 20250507064311.png]]

为此，首先点击一次“运行”按钮，然后右键单击反汇编视图中的任意位置，选择 **Search for** > **Current Module** > **String references**。

![[Pasted image 20250507064340.png]]

接下来，我们可以添加一个断点来标记位置，然后研究此沙箱 **MessageBox** 前的指令，以辨别是如何跳转到打印 `Sandbox detected` 的指令的。

让我们首先在最后一个 `Sandbox detected` 字符串处添加一个断点，如下所示。

![[Pasted image 20250507064402.png]]

然后，我们可以双击该字符串，跳转到打印 `Sandbox detected` 的指令所在的地址。

![[Pasted image 20250507064426.png]]

如观察到的，在此 **MessageBox** 上方存在一个 `cmp` 指令，它在执行注册表路径比较后将值与 `1` 进行比较。让我们修改此比较值，改为与 `0` 匹配。可以通过将光标放在该指令上并按键盘上的**空格键**来完成。这将允许我们编辑汇编代码指令。

我们可以将比较值从 `0x1` 更改为 `0x0`。使其不跳转到显示 **MessageBox** 的地址。
![[Pasted image 20250507064642.png]]
在 **x64dbg** 中点击“运行”或按 **F9** 后，它将不会命中第一个沙箱检测消息代码的断点。这意味着我们成功地修补了指令。

以类似的方式，我们可以在下一个沙箱检测函数打印 **MessageBox** 之前也添加一个断点。为此，应在**倒数第二个** `Sandbox detected` 字符串（`0000000000402F13`）处设置断点。如果我们双击此字符串，会注意到那里有一个 `jump` 指令，我们可以跳过它，将执行流导向调用另一个函数的下一条指令。这正是我们需要的——它跳转到另一个函数，而不是沙箱检测 **MessageBox**。

![[Pasted image 20250507064655.png]]

我们可以将指令从 `je shell.402F09` 修改为 `jne shell.402F09`。

![[Pasted image 20250507064703.png]]

shell.exe 还通过检查互联网连接进行沙箱检测。我们的模拟器没有互联网连接。因此，我们也应该修补这种沙箱检测方法。我们可以通过点击第一个 `Sandbox detected` 字符串（`0000000000402CBD`）并修补以下指令来完成。

![[Pasted image 20250507065355.png]]
![[Pasted image 20250507065407.png]]

现在，当我们按下“运行”后，已打补丁的 `shell.exe` 将继续执行，从 INetSim 下载默认的可执行文件，并运行它。

![[Pasted image 20250507065641.png]]

沙箱检查被绕过后，实际功能得以展现。我们可以按 **Ctrl+P** 并点击 **Patch File** 来保存已打补丁的可执行文件。此操作会保存跳过沙箱检查的文件。

![[Pasted image 20250507065703.png]]

我们执行此过程是为了确保下次运行保存的已打补丁文件时，它会直接执行而无需进行沙箱检查，并且我们可以在 **ProcessMonitor** 中观察到所有事件。

**分析恶意软件流量**

> 流量分析应该作为动态分析的一个组成部分。

现在让我们使用 **Wireshark** 来捕获和检查恶意软件生成的网络流量。请注意颜色编码的流量：红色对应客户端到服务器的流量，而蓝色表示服务器到客户端的交换。

检查 HTTP 请求显示，恶意软件样本在 User-agent 字段中附加了计算机主机名（在此示例中是 `RDSEMVM01`）。

![[Pasted image 20250507070413.png]]

检查 HTTP 响应时，很明显 INetSim 已将其默认二进制文件作为响应返回给恶意软件。

![[Pasted image 20250507070417.png]]

恶意软件对 `svchost.exe` 的请求，是向 INetSim 索要默认二进制文件。此二进制文件执行后会显示一个 **MessageBox**，消息内容是：`This is the INetSim default binary.`。

此外，恶意软件发送了对一个随机域名和地址 `ms-windows-update[.]com` 的 DNS 请求，INetSim 返回了伪造的响应（在此示例中，INetSim 运行在 `10.10.10.100` 上）。

![[Pasted image 20250507070424.png]]

**分析进程注入和内存区域**

在代码分析过程中，我们发现我们的可执行文件对 `notepad.exe` 执行了进程注入，并显示了一个 **`MessageBox`**，消息内容是 `Connection sent to C2`。

为了更深入地探查进程注入，在 WINAPI 函数 **`VirtualAllocEx`**、**`WriteProcessMemory`** 和 **`CreateRemoteThread`** 处设置断点。这些断点将允许我们在进程注入期间检查寄存器中保存的内容。

访问 **x64dbg** 界面，导航到顶部的 **Symbols** 选项卡。

![[Pasted image 20250507070639.png]]

在符号搜索框中，在左侧搜索所需的 DLL 名称，找到 **Kernel32.dll**，在右侧中搜索函数名称，例如 **VirtualAllocEx**、**WriteProcessMemory** 和 **CreateRemoteThread**等。

当函数名称出现在搜索结果中时，右键单击并从上下文菜单中选择 **Toggle breakpoint**（或使用快捷键 **F2**）为每个函数设置断点。

执行这些步骤将在每个函数的入口点设置一个断点。我们将为所有打算检查的函数重复这些步骤。

设置断点后，我们按 **F9** 或从工具栏中选择“运行”，直到达到 **WriteProcessMemory** 的断点。直到此刻，`notepad` 已经启动，但 shellcode 尚未写入 `notepad` 的内存中。

**在 x64dbg 中附加另一个运行中的进程**

为了进一步深入，让我们打开另一个 **x64dbg** 实例，并将其**附加**到 `notepad.exe` 进程。

1. 启动一个新的 **x64dbg** 实例。

2. 导航到 **File** 菜单并选择 **Attach**，或使用键盘快捷键 **Alt + A**。

3. 在 **Attach** 对话框中，将显示正在运行的进程列表。从列表中选择 `notepad.exe`。

4. 单击 **Attach** 按钮开始附加过程。

附加成功后，**x64dbg** 开始调试目标进程，主窗口会显示汇编代码以及其他调试信息。

![[Pasted image 20250507071014.png]]

现在，我们可以建立断点、单步执行代码、检查寄存器和内存，以及使用 **x64dbg** 研究附加的 `notepad.exe` 进程的行为。

**`WriteProcessMemory`** 的第二个参数是 **`lpBaseAddress`**，它包含一个指向指定进程中写入数据的基地址的指针。在我们的示例中，它应该在 **`RDX`** 寄存器中。

```CPP
BOOL WriteProcessMemory(
  [in]  HANDLE  hProcess,
  [in]  LPVOID  lpBaseAddress,
  [in]  LPCVOID lpBuffer,
  [in]  SIZE_T  nSize,
  [out] SIZE_T  *lpNumberOfBytesWritten
);
```

调用 **`WriteProcessMemory`** 函数时，`rdx` 寄存器保存着 `lpBaseAddress` 参数。此参数表示目标进程地址空间中将写入数据的地址。

我们在检查在运行 `shell.exe` 进程的 **x64dbg** 实例中调用 **`WriteProcessMemory`** 函数时寄存器的值。这表示 shellcode 将写入 `notepad.exe` 的地址。

![[Pasted image 20250507071337.png]]
我们复制此地址，以在运行 `notepad.exe` 的第二个 **x64dbg** 实例的内存转储中检查其内容。

现在，在运行 `notepad.exe` 的第二个 **x64dbg** 实例的内存转储中右键单击任意位置，选择 **Go to** > **Expression**。

![[Pasted image 20250507071423.png]]

输入复制的地址后，显示该地址的内容（右键单击地址并选择 **Follow in Dump** > **Selected Address**），该地址目前是空的。

接下来，我们在第一个 **x64dbg** 实例中单击“运行”按钮执行 `shell.exe`。我们观察 `notepad.exe` 的此内存区域中写入了什么。

![[Pasted image 20250507071511.png]]

执行后，我们识别出被注入的 shellcode，这与我们之前在代码分析期间发现的一致。我们可以在 **Process Hacker** 中验证这一点，并将其保存到文件中以供后续检查。