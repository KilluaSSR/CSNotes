
## **创建检测规则**

我们可以着手设计检测规则，例如 **Yara** 和 **Sigma** 规则。
### **Yara**

YARA (Yet Another Recursive Acronym)，一个广泛使用的开源模式匹配工具和基于规则的恶意软件检测与分类框架，允许我们创建自定义规则来识别文件、进程或内存中的特定模式或特征。为了为我们的样本拟一个 **YARA** 规则，我们需要检查该样本的独有行为、特征或特定的字符串/模式。

以下是一个简单的 YARA 规则示例，它匹配进程中存在字符串 `Sandbox detected`。例子中的`shell.exe` 展示了这种行为。

```yara
rule Shell_Sandbox_Detection {
    strings:
        $sandbox_string = "Sandbox detected"
    condition:
        $sandbox_string
}
```

现在让我们向规则中添加更多字符串和模式，以使其更完善。我们可以利用 **`yarGen`** 工具，该工具可以自动化生成 YARA 规则的过程，主要目标是为手动后期处理生成尽可能好的规则。然而，需要人类才能生成一个健壮的规则。

```bash
mkdir /home/killua/Samples/Test
```


```bash
cp /home/killua/Samples/shell.exe /home/killua/Samples/Test
```

要为 `shell.exe` 自动创建 Yara 规则，我们应执行以下命令

```bash
sudo python3 yarGen.py -m /home/killua/Samples/Test/
------------------------------------------------------------------------
                   _____            
    __ _____ _____/ ___/__ ___      
   / // / _ `/ __/ (_ / -_) _ \     
   \_, /\_,_/_/  \___/\__/_//_/     
  /___/  Yara Rule Generator        
         Florian Roth, July 2020, Version 0.23.3
   
  Note: Rules have to be post-processed
  See this post for details: https://medium.com/@cyb3rops/121d29322282
------------------------------------------------------------------------
[+] Using identifier 'Test'
[+] Using reference 'https://github.com/Neo23x0/yarGen'
[+] Using prefix 'Test'
[+] Processing PEStudio strings ...
[+] Reading goodware strings from database 'good-strings.db' ...
    (This could take some time and uses several Gigabytes of RAM depending on your db size)
[+] Loading ./dbs/good-imphashes-part3.db ...
[+] Total: 4029 / Added 4029 entries
[+] Loading ./dbs/good-strings-part9.db ...
<SNIP>
[+] Total: 12284943 / Added 3132327 entries
[+] Loading ./dbs/good-imphashes-part1.db ...
[+] Total: 19764 / Added 1556 entries
[+] Loading ./dbs/good-exports-part5.db ...
[+] Total: 404321 / Added 71962 entries
[+] Processing malware files ...
[+] Processing /home/htb-student/Samples/MalwareAnalysis/Test/shell.exe ...
[+] Generating statistical data ...
[+] Generating Super Rules ... (a lot of magic)
[+] Generating Simple Rules ...
[-] Applying intelligent filters to string findings ...
[-] Filtering string set for /home/killua/Samples/Test/shell.exe ...
[=] Generated 1 SIMPLE rules.
[=] All rules written to yargen_rules.yar
[+] yarGen run finished
```

我们会注意到 `yarGen` 生成了一个名为 `yargen_rules.yar` 的文件，其中包含了自动提取并插入到规则中的独特字符串。

```bash
cat yargen_rules.yar
/*
   YARA Rule Set
   Author: yarGen Rule Generator
   Date: 2023-08-02
   Identifier: Test
   Reference: https://github.com/Neo23x0/yarGen
*/

/* Rule Set ----------------------------------------------------------------- */

rule _home_htb_student_Samples_MalwareAnalysis_Test_shell {
   meta:
      description = "Test - file shell.exe"
      author = "yarGen Rule Generator"
      reference = "https://github.com/Neo23x0/yarGen"
      date = "2023-08-02"
      hash1 = "bd841e796feed0088ae670284ab991f212cf709f2391310a85443b2ed1312bda"
   strings:
      $x1 = "C:\\Windows\\System32\\cmd.exe" fullword ascii
      $s2 = "http://ms-windows-update.com/svchost.exe" fullword ascii
      $s3 = "C:\\Windows\\System32\\notepad.exe" fullword ascii
      $s4 = "/k ping 127.0.0.1 -n 5" fullword ascii
      $s5 = "iuqerfsodp9ifjaposdfjhgosurijfaewrwergwea.com" fullword ascii
      $s6 = "  VirtualQuery failed for %d bytes at address %p" fullword ascii
      $s7 = "[-] Error code is : %lu" fullword ascii
      $s8 = "C:\\Program Files\\VMware\\VMware Tools\\" fullword ascii
      $s9 = "Failed to open the registry key." fullword ascii
      $s10 = "  VirtualProtect failed with code 0x%x" fullword ascii
      $s11 = "Connection sent to C2" fullword ascii
      $s12 = "VPAPAPAPI" fullword ascii
      $s13 = "AWAVAUATVSH" fullword ascii
      $s14 = "45.33.32.156" fullword ascii
      $s15 = "  Unknown pseudo relocation protocol version %d." fullword ascii
      $s16 = "AQAPRQVH1" fullword ascii
      $s17 = "connect" fullword ascii /* Goodware String - occured 429 times */
      $s18 = "socket" fullword ascii /* Goodware String - occured 452 times */
      $s19 = "tSIcK<L" fullword ascii
      $s20 = "Windows-Update/7.6.7600.256 %s" fullword ascii
   condition:
      uint16(0) == 0x5a4d and filesize < 60KB and
      1 of ($x*) and 4 of them
}
```

我们可以审查该规则并根据需要进行修改，添加更多字符串和条件以增强其可靠性和有效性。

### **使用 Yara 规则检测恶意软件 **

然后我们可以使用此规则扫描目录
```bash
yara /home/killua/yarGen-0.23.4/yargen_rules.yar /home/killua/Samples

/home/killua/Samples/shell.exe
```

我们会注意到 `shell.exe` 被返回了！

以下是一些 YARA 规则的参考资料：

[Yara documentation](https://yara.readthedocs.io/en/stable/writingrules.html)

[Yara resources](https://github.com/InQuest/awesome-yara)

[The DFIR Report]( https://github.com/The-DFIR-Report/Yara-Rules)

### **Sigma**

**Sigma** 是一种全面且标准化的规则格式，被安全分析师和 **安全信息和事件管理 (SIEM)** 系统广泛使用。其目标是检测和识别可能预示安全威胁或事件的特定模式或行为。Sigma 规则的标准化格式使安全团队能够在不同的安全平台之间定义和传播检测逻辑。

要根据某些操作构建 **Sigma** 规则 - 例如，在临时目录放置文件，我们可以设计一个类似的示例规则。

```
title: Suspicious File Drop in Users Temp Location
status: experimental
description: Detects suspicious activity where a file is dropped in the temp location

logsource:
    category: process_creation
detection:
    selection:
        TargetFilename:
            - '*\\AppData\\Local\\Temp\\svchost.exe'
    condition: selection
    level: high

falsepositives:
    - Legitimate exe file drops in temp location
```

在此示例中，该规则旨在识别文件 `svchost.exe` 被放置在 `Temp` 目录中的情况。

在分析过程中，持续运行系统监视代理是有用的。我们选择**Sysmon** 来收集日志。Sysmon 是一个强大的工具，可以捕获详细的事件数据并帮助创建 Sigma 规则。其日志类别包括进程创建（EventID 1）、网络连接（EventID 3）、文件创建（EventID 11）、注册表修改（EventID 13）等。对这些事件的仔细检查有助于查明威胁指标（IOCs）和理解行为模式，从而促进创建有效的检测规则。

例如，Sysmon 针对 `shell.exe` 执行的活动收集了**进程创建**、**进程访问**、**文件创建**和**网络连接**等日志。这些编译的信息有助于增强我们对样本行为的理解，并开发更精确有效的检测规则。

**Process Create Logs**:
![[Pasted image 20250507080304.png]]
**Process Access Logs**:
![[Pasted image 20250507080309.png]]
**File Creation Logs**:
![[Pasted image 20250507080312.png]]
**Network Connection Logs**:
![[Pasted image 20250507080317.png]]

- [Sigma documentation]( [https://github.com/SigmaHQ/sigma-specification](https://github.com/SigmaHQ/sigma-specification?tab=readme-ov-file#specification))
- [Sigma resources]([https://github.com/SigmaHQ/sigma/tree/master/rules](https://github.com/SigmaHQ/sigma/tree/master/rules))
- [The DFIR Report]([https://github.com/The-DFIR-Report/Sigma-Rules/tree/main/rules](https://github.com/The-DFIR-Report/Sigma-Rules/tree/main/rules))