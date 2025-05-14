## 识别哈希种类

> 大多数哈希算法会生成**固定长度**的哈希值。特定哈希的长度可以用来将其映射到生成它的算法。例如，一个长度为 32 个字符的哈希值可能是 MD5 或 NTLM 哈希。
> 
> 有时，哈希值以特定格式存储。例如，`hash:salt` 或 `$id$salt$hash`。

哈希值 `2fc5a684737ce1bf7b3b239df432416e0dd07357:2014` 是一个 SHA1 哈希，其盐值为 `2014`。

哈希值 `$6$vb1tLY1qiY$M.1ZCqKtJBxBtZm1gRi8Bbkn39KU0YJW1cuMFzTRANcNKFKR4RmAQVk4rqQQCkaJT6wXqjUkFcA/qNxLyqW.U/` 包含由 `$` 分隔的三个字段，其中第一个字段是 **id**，即 `6`。这用于识别哈希时使用的算法类型。以下列表包含了一些 id 及其对应的算法。

```
$1$  : MD5
$2a$ : Blowfish
$2y$ : Blowfish, with correct handling of 8 bit characters
$5$  : SHA256
$6$  : SHA512
```

下一个字段 `vb1tLY1qiY` 是哈希过程中使用的盐值，最后一个字段是实际的哈希值。

开源和闭源软件使用许多不同类型的哈希格式。例如，**Apache** Web 服务器将其哈希存储为 `$apr1$71850310$gh9m4xcAn3MGxogwX/ztb.` 格式，而 **WordPress** 将哈希存储为 `$P$984478476IagS59wHZvyQMArzfx58u..` 格式。

### **Hashid**

**Hashid** 是一个 **Python** 工具，可用于检测各种类型的哈希，hashid 可以识别 200 多种独特的哈希类型，对于其他类型，它会进行尽力猜测，这仍然需要一些额外的工作来缩小范围。支持的哈希的完整列表可以在这里找到。可以使用 `pip` 安装它。

```bash
pip install hashid
```


```bash
hashid '$apr1$71850310$gh9m4xcAn3MGxogwX/ztb.'
```

```
Analyzing '$apr1$71850310$gh9m4xcAn3MGxogwX/ztb.'
[+] MD5(APR)
[+] Apache MD5
```

```bash
bashid hashes.txt --File 'hashes.txt'--
```

```
Analyzing '2fc5a684737ce1bf7b3b239df432416e0dd07357:2014'
[+] SHA-1
[+] Double SHA-1
[+] RIPEMD-160
[+] Haval-160
[+] Tiger-160
[+] HAS-160
[+] LinkedIn
[+] Skein-256(160)
[+] Skein-512(160)
[+] Redmine Project Management Web App
[+] SMF ≥ v1.1
Analyzing '$P$984478476IagS59wHZvyQMArzfx58u.'
[+] Wordpress ≥ v2.6.2
[+] Joomla ≥ v2.5.18
[+] PHPass' Portable Hash
--End of file 'hashes.txt'--
```

如果已知，`hashid` 还可以使用 `-m` 标志提供对应的 **Hashcat** 哈希模式，如果它能够确定哈希类型的话。


```bash
hashid '$DCC2$10240#tom#e4e938d12fe5974dc42a90120bd9c90f' -m
```

```
Analyzing '$DCC2$10240#tom#e4e938d12fe5974dc42a90120bd9c90f'
[+] Domain Cached Credentials 2 [Hashcat Mode: 2100]
```

并非总是能够仅根据获取的哈希值来确定算法。根据软件的不同，明文可能会经过多轮加密和加盐转换，使得恢复更加困难。`hashid` 使用正则表达式来尽力确定提供的哈希类型。通常 `hashid` 会为给定的哈希值提出多种可能猜测。我们通常有一些关于我们正在尝试识别的哈希类型的上下文。它是在 Active Directory 攻击中获得的，还是来自 Windows 主机？它是通过成功利用 SQL 注入漏洞获得的吗？了解哈希的来源将极大地帮助我们缩小哈希类型范围，从而确定尝试破解它所需的 Hashcat 哈希模式。Hashcat 提供了一份优秀的**参考资料**，可以帮助确定我们正在处理的哈希类型以及将其传递给 Hashcat 所需的模式。

例如，将哈希 `a2d1f7b7a1862d0d4a52644e72d59df5:500:lp@trash-mail.com` 传递给 `hashid` ，将给出多种可能性：

```bash
hashid 'a2d1f7b7a1862d0d4a52644e72d59df5:500:lp@trash-mail.com'
```

```
Analyzing 'a2d1f7b7a1862d0d4a52644e72d59df5:500:lp@trash-mail.com'
[+] MD5
[+] MD4
[+] Double MD5
[+] LM
[+] RIPEMD-128
[+] Haval-128
[+] Tiger-128
[+] Skein-256(128)
[+] Skein-512(128)
[+] Lotus Notes/Domino 5
[+] Skype
[+] Lastpass
```

然而，快速查阅 Hashcat 示例哈希参考资料将帮助我们确定它确实是 Lastpass 哈希，对应的哈希模式是 `6800`。

## **Hashcat**

**Hashcat** 是一个流行的开源密码破解工具。

```bash
hashcat -h
```

```
hashcat (v6.1.1) starting...

Usage: hashcat [options]... hash|hashfile|hccapxfile [dictionary|mask|directory]...

- [ Options ] -

 Options Short / Long           | Type | Description                                          | Example
================================+======+======================================================+=======================
 -m, --hash-type                | Num  | 哈希类型，见下方参考                                  | -m 1000
 -a, --attack-mode              | Num  | 攻击模式，见下方参考                                | -a 3
 -V, --version                  |      | 打印版本                                          |
 -h, --help                     |      | 打印帮助                                          |
     --quiet                    |      | 抑制输出                                          |
     --hex-charset              |      | 假设字符集以十六进制给出                                  |
     --hex-salt                 |      | 假设盐值以十六进制给出                                    |
     --hex-wordlist             |      | 假设字典中的单词以十六进制给出                             |
     --force                    |      | 忽略警告                                          |
     --status                   |      | 启用状态屏幕自动更新                                  |
     --status-json              |      | 启用状态输出的 JSON 格式                              |
     --status-timer             | Num  | 设置状态屏幕更新间隔为 X 秒                              | --status-timer=1
     --stdin-timeout-abort      | Num  | 如果标准输入 X 秒没有输入则中止                             | --stdin-timeout-abort=300
     --machine-readable         |      | 以机器可读格式显示状态视图                             |

<SNIP>
```

`-a` 和 `-m` 参数用于指定**攻击模式**和**哈希类型**。Hashcat 支持以下攻击模式：

```
# Mode
0 | Straight (字典攻击)
1 | Combination (组合攻击)
3 | Brute-force (暴力破解)
6 | Hybrid Wordlist + Mask (混合攻击：字典 + 掩码)
7 | Hybrid Mask + Wordlist (混合攻击：掩码 + 字典)
```

哈希类型的值基于要破解的哈希算法。也可以使用以下命令通过命令行查看示例哈希列表：


```bash
hashcat --example-hashes | less
```

```
hashcat (v6.1.1) starting...

MODE: 0
TYPE: MD5
HASH: 8743b52063cd84097a65d1633f5c74f5
PASS: hashcat

MODE: 10
TYPE: md5($pass.$salt)
HASH: 3d83c8e717ff0e7ecfe187f088d69954:343141
PASS: hashcat

MODE: 11
TYPE: Joomla < 2.5.18
HASH: b78f863f2c67410c41e617f724e22f34:89384528665349271307465505333378
PASS: hashcat

MODE: 12
TYPE: PostgreSQL
HASH: 93a8cf6a7d43e3b5bcd2dc6abb3e02c6:27032153220030464358344758762807
PASS: hashcat

MODE: 20
TYPE: md5($salt.$pass)
HASH: 57ab8499d08c59a7211c77f557bf9425:4247
PASS: hashcat

<SNIP>
```

可以使用 `-b` 标志对特定哈希类型执行性能测试。

```bash
killuassr@htb[/htb]$ hashcat -b -m 0
```

```
hashcat (v6.1.1) starting in benchmark mode...

Benchmarking uses hand-optimized kernel code by default.
You can use it in your cracking session by setting the -O option.
Note: Using optimized kernel code limits the maximum supported password length.
To disable the optimized kernel code in benchmark mode, use the -w option.
OpenCL API (OpenCL 1.2 pocl 1.5, None+Asserts, LLVM 9.0.1, RELOC, SLEEF, DISTRO, POCL_DEBUG) - Platform #1 [The pocl project]
=============================================================================================================================
* Device #1: pthread-Intel(R) Core(TM) i7-5820K CPU @ 3.30GHz, 4377/4441 MB (2048 MB allocatable), 6MC
Benchmark relevant options:
===========================
* --optimized-kernel-enable

Hashmode: 0 - MD5
Speed.#1.........:   449.4 MH/s (12.84ms) @ Accel:1024 Loops:1024 Thr:1 Vec:8
Started: Fri Aug 28 21:52:35 2020
Stopped: Fri Aug 28 21:53:25 2020
```


### 优化

Hashcat 有两种主要优化速度的方法：

|             **选项**              |                                                           **描述**                                                            |
| :-----------------------------: | :-------------------------------------------------------------------------------------------------------------------------: |
| **Optimized Kernels** (`-O` 标志) | 根据文档，这意味着**启用优化内核（限制密码长度）**。密码长度数字通常是 32，大多数字典甚至达不到这个数字。这可以将估计时间从几天缩短到几小时，因此**始终建议**先使用 `-O` 运行，如果您的 GPU 空闲，之后不带 `-O` 重新运行。 |
|     **Workload** (`-w` 标志)      |          根据文档，这意味着**启用特定的工作负载配置文件**。默认数字是 `2`，但如果想在使用 Hashcat 时同时使用计算机，请将其设置为 `1`。如果计划计算机只运行 Hashcat，则可以将其设置为 `3`。          |
好的，这是您提供的文本的中文翻译，并包含对 `cut` 和 `awk` 命令的解释。

## 字典攻击

顾名思义，这种攻击读取一个字典文件，并尝试破解提供的哈希值。

```bash
hashcat -a 0 -m <hash type> <hash file> <wordlist>
```

```bash
echo -n '!academy' | sha256sum | cut -f1 -d' ' > sha256_hash_example
```

* `echo -n '!academy'`：输出字符串 `!academy`，`-n` 选项表示不输出末尾的换行符。
* `sha256sum`：计算输入内容的 SHA256 哈希值。默认输出格式是 `哈希值 文件名` 或 `哈希值 -` (如果从标准输入读取)。
* `cut -f1 -d' '`：`cut` 命令用于从文本流中提取指定部分。`-f1` 表示提取第一个字段，`-d' '` 表示使用空格作为字段之间的分隔符。在这里，它从 `sha256sum` 的输出中提取第一个字段（即哈希值本身），忽略后面的文件名或 `-`。
* `>`：将 `cut` 命令的标准输出重定向到 `sha256_hash_example` 文件，创建并写入哈希值。

```bash
hashcat -a 0 -m 1400 sha256_hash_example /opt/useful/seclists/Passwords/Leaked-Databases/rockyou.txt
```


让我们看看一个更复杂的哈希，例如 Bcrypt。Bcrypt 是一种基于 Blowfish 加密算法的密码哈希类型。它使用盐值来防御彩虹表攻击，并且可以应用多轮算法。

例如，使用相同密码 "!academy" 的 bcrypt 哈希值，应用 5 轮 Blowfish 算法，将是 `$2a$05$ZdEkj8cup/JycBRn2CX.B.nIceCYR8GbPbCCg6RlD7uvuREexEbVy`。在相同的硬件上使用相同的字典文件运行此哈希，需要更长的时间来破解。

## 组合攻击

组合攻击模式接收两个字典文件作为输入，并从它们中创建组合。用户将两个或更多单词组合在一起并认为这能创建更强的密码的情况并不少见，例如 `welcomehome` 或 `hotelcalifornia`。

考虑以下字典文件：

```bash
cat wordlist1
```

```
super
world
secret
```

```bash
cat wordlist2
```

```
hello
password
```

如果给定这两个字典文件，Hashcat 将生成恰好 3 x 2 = 6 个单词，如下所示：

```bash
killuassr@htb[/htb]$ awk '(NR==FNR) { a[NR]=$0 } (NR != FNR) { for (i in a) { print $0 a[i] } }' file2 file1
```

* `awk '...' file2 file1`：使用 `awk` 工具处理 `file2` 和 `file1` 这两个文件。`awk` 逐行读取文件。
* `(NR==FNR) { a[NR]=$0 }`：只在处理**第一个文件** (`file2`) 时执行。`NR` 是 `awk` 到目前为止读取的总行数，`FNR` 是 `awk` 在**当前**文件中读取的行号。当处理第一个文件时，`NR` 和 `FNR` 是相等的。`$0` 表示当前行的全部内容。这个代码块的作用是将 `file2` 中的每一行存储到一个名为 `a` 的关联数组中，使用当前行号 `NR` 作为数组的索引。
* `(NR != FNR) { for (i in a) { print $0 a[i] } }`：只在处理**第二个文件** (`file1`) 时执行。`for (i in a)` 循环遍历数组 `a` 的所有索引（这些索引对应 `file2` 的行号）。在循环内部，对于 `file1` 的当前行 (`$0`)，它会与数组 `a` 中的每个元素 (`a[i]`) 进行拼接（即将 `file1` 的行内容放在前面，`file2` 的行内容放在后面），然后打印结果。这有效地生成了 `file1` 中每一行与 `file2` 中每一行的所有组合。

```
superhello
superpassword
worldhello
wordpassword
secrethello
secretpassword
```

这也可以使用 Hashcat 的 `--stdout` 标志来完成

```bash
hashcat -a 1 --stdout file1 file2
```

```
superhello
superpassword
worldhello
worldpassword
secrethello
secretpassword
```


```bash
hashcat -a 1 -m <hash type> <hash file> <wordlist1> <wordlist2>
```

首先，创建密码 `secretpassword` 的 md5 哈希值。

```bash
echo -n 'secretpassword' | md5sum | cut -f1 -d' '  > combination_md5
```

```
2034f6e32958647fdff75d265b455ebf
```

接下来，让我们使用组合攻击模式，通过上面两个字典文件对哈希运行 Hashcat。

```bash
hashcat -a 1 -m 0 combination_md5 wordlist1 wordlist2
```


## 掩码攻击 

掩码攻击用于生成符合特定模式的单词。当密码长度或格式已知时，这种攻击类型特别有用。可以使用静态字符、字符范围（例如 `[a-z]` 或 `[A-Z0-9]`）或占位符创建掩码。以下列表显示了一些重要的占位符：

| **占位符** | **含义**                                   |
| ------- | ---------------------------------------- |
| `?l`    | 小写 ASCII 字母 (a-z)                        |
| `?u`    | 大写 ASCII 字母 (A-Z)                        |
| `?d`    | 数字 (0-9)                                 |
| `?h`    | 小写十六进制字符 (0-9abcdef)                     |
| `?H`    | 大写十六进制字符 (0-9ABCDEF)                     |
| `?s`    | 特殊字符（空格!"#$%&'()*+,-./:;&lt;=>?@[]^_` {） |

上述占位符可以与选项 "-1" 到 "-4" 结合使用，这些选项可用于自定义占位符。有关如何配置这四个自定义字符集的详细说明，请参阅**此处**的 Custom charsets 部分。

考虑 Inlane Freight 公司，这次他们的密码方案是 `ILFREIGHT<userid><year>`，其中 `userid` 长 5 个字符。掩码 "`ILFREIGHT?l?l?l?l?l20[0-1]?d`" 可用于破解符合指定模式的密码，其中 `?l` 是一个字母，`20[0-1]?d` 将包含从 2000 到 2019 的所有年份。

```bash
echo -n 'ILFREIGHTabcxy2015' | md5sum | tr -d " -" > md5_mask_example_hash
```

**命令解释：**

- `tr -d " -"`：`tr` 命令用于删除或替换字符。`-d` 选项表示删除。`" -"` 是要删除的字符集合，包含一个空格和一个连字符 `-`。在这里用于移除 `md5sum` 输出末尾的空格和连字符，只留下纯净的哈希值。

在下面的示例中，攻击模式是 `3`，MD5 的哈希类型是 `0`。

```bash
hashcat -a 3 -m 0 md5_mask_example_hash -1 01 'ILFREIGHT?l?l?l?l?l20?1?d'
```

"-1" 选项用于指定一个只包含 0 和 1 的占位符。可以使用 "--increment" 标志自动增加掩码长度，并使用 "--increment-max" 标志提供长度限制。


## 混合模式

混合模式是组合攻击的一种变体，其中可以结合使用多种模式来创建**精细调整的字典**。这种模式可用于通过创建定制的字典来执行非常有针对性的攻击。大致了解组织的密码策略或常见密码语法时，它特别有用。在单词后面添加掩码的混合攻击模式是 "**6**"。

让我们考虑一个密码`football1$`。
```bash
echo -n 'football1$' | md5sum | tr -d " -" > hybrid_hash
```

Hashcat 会从字典文件中读取单词，并根据提供的**掩码**附加一个独特的字符串。掩码 `?d?s` 告诉 Hashcat 在 `rockyou.txt` 字典文件中的每个单词后面附加一个数字和一个特殊字符。

```bash
hashcat -a 6 -m 0 hybrid_hash /opt/useful/seclists/Passwords/Leaked-Databases/rockyou.txt '?d?s'
```

攻击模式 "**7**" 可以用于使用给定的掩码在单词**前面**添加字符。

```bash
echo -n '2015football' | md5sum | tr -d " -" > hybrid_hash_prefix
```

```bash
hashcat -a 7 -m 0 hybrid_hash_prefix -1 01 '20?1?d' /opt/useful/seclists/Passwords/Leaked-Databases/rockyou.txt
```

（这里 `-1 01` 定义了自定义字符集 `?1` 为 `0` 和 `1`。掩码 `'20?1?d'` 则表示以 "20" 开头，后面接一个 `?1` (即 `0` 或 `1`)，最后接一个 `?d` (即一个数字 0-9)。结合 `-a 7` 模式，这个掩码将作为前缀添加到字典的每个单词前面。）


## 创建自定义字典文件 

在渗透测试过程中，我们可能会获取一个或多个对项目成功至关重要的密码哈希。尽管我们尽了最大努力，这些哈希仍然无法通过前面介绍的知识破解。在这种情况下，可能有必要创建**定制的、有针对性的**字典文件来实现我们的目标。

花时间细化字典文件是非常必要的，因为成功率在很大程度上取决于它。字典文件可以从各种来源获取，并根据目标进行定制，然后通过规则进一步调整。字典文件可以用于密码、用户名、文件名、Payload 以及许多其他数据类型。SecLists 仓库也包含许多对用户名枚举和密码识别有用的字典文件。
### Crunch

`Crunch` 可以根据特定长度的单词、有限的字符集或特定模式等参数创建字典文件。它可以生成`排列`和`组合`。


```bash
crunch <minimum length> <maximum length> <charset> -t <pattern> -o <output file>
```

`-t` 选项用于指定生成密码的模式。模式中，`@` 代表小写字母，`,`（逗号）代表大写字母，`%` 代表数字，`^` 代表符号。


```bash
crunch 4 8 -o wordlist
```

上面的命令使用默认字符集，创建长度为 4 到 8 个字符的单词组成的字典文件。

假设 Inlane Freight 用户的密码形式为 `ILFREIGHTYYYYXXXX`，其中 `XXXX` 是由字母组成的员工 ID，`YYYY` 是年份。我们可以使用 `crunch` 创建这样一个密码列表。


```bash
crunch 17 17 -t ILFREIGHT201%@@@@ -o wordlist
```

"`ILFREIGHT201%@@@@` 将创建年份为 2010-2019 开头，后面跟着四个小写字母的单词。这里的长度是 17，对所有单词都是固定的。

如果我们通过社交媒体等途径知道用户的出生日期是`10/03/1998`，我们可以将其包含在他们的密码中，后面跟着一串字母。`Crunch`可以用来创建这样一个单词列表。`-d` 选项用于指定重复的次数限制。

```bash
crunch 12 12 -t 10031998@@@@ -d 1 -o wordlist
```

上面的命令将创建长度为 12 个字符的单词，模式是 `10031998`后面跟着四个小写字母，`-d 1` 限制了每个字符集中的字符最多重复 1 次（在此模式下对 `@` 应用，意味着连续的相同字母不会出现）。

### CUPP

`CUPP (Common User Password Profiler)`用于基于从社会工程和 `OSINT` 获得的信息创建高度有针对性的自定义字典文件。人们倾向于在创建密码时使用个人信息，例如电话号码、宠物名称、出生日期等。`CUPP` 接收这些信息并从中创建密码。这些字典文件主要用于获取社交媒体账户的访问权限。

```bash
cupp -i
```

```
[+] Insert the information about the victim to make a dictionary
[+] If you don't know all the info, just hit enter when asked! ;)

> First Name: roger (名字)
> Surname: penrose (姓氏)
> Nickname:       (昵称)
> Birthdate (DDMMYYYY): 11051972 (出生日期 日月年)

> Partners) name: beth (伴侣名字)
> Partners) nickname: (伴侣昵称)
> Partners) birthdate (DDMMYYYY): (伴侣出生日期)

> Child's name: john (孩子名字)
> Child's nickname: johnny (孩子昵称)
> Child's birthdate (DDMMYYYY): (孩子出生日期)

> Pet's name: tommy (宠物名字)
> Company name: INLANE FREIGHT (公司名称)

> Do you want to add some key words about the victim? Y/[N]: Y (是否添加关键词？)
> Please enter the words, separated by comma. [i.e. hacker,juice,black], spaces will be removed: sysadmin,linux,86391512 (请输入关键词，用逗号分隔)
> Do you want to add special chars at the end of words? Y/[N]: (是否在词末添加特殊字符？)
> Do you want to add some random numbers at the end of words? Y/[N]: (是否在词末添加随机数字？)
> Leet mode? (i.e. leet = 1337) Y/[N]: (是否启用 Leet 模式？)

[+] Now making a dictionary... (正在生成字典...)
[+] Sorting list and removing duplicates... (正在排序和去重...)
[+] Saving dictionary to roger.txt, counting 2419 words. (正在保存字典到 roger.txt，共计 2419 个词)
[+] Now load your pistolero with roger.txt and shoot! Good luck! (现在用 roger.txt 装填你的破解工具，祝好运！)
```

上面的命令展示了如何向 `CUPP` 提供用户 Roger Penrose 的信息。未知字段可以留空。输入所有数据后，`CUPP` 会根据这些数据创建一个字典文件。它还支持附加随机字符和`Leet模式`，该模式使用常见单词中字母和数字的组合。`CUPP` 还可以使用 `-l` 选项从各种在线数据库中获取常见名称。

### Kwprocessor

`Kwprocessor` 是一个创建包含`键盘行走模式`的字典文件的工具。另一种常见的密码生成技术是遵循键盘上的模式。这些密码被称为`键盘行走模式`，因为它们看起来像沿着按键行走。例如，字符串 `"qwertyasdfg"` 是通过使用键盘前两行的前五个字符创建的。这在常人看来很复杂，但很容易预测。`Kwprocessor` 使用各种算法来猜测此类模式。

```bash
git clone https://github.com/hashcat/kwprocessor
cd kwprocessor
make
```

帮助菜单显示了 `kwp` 支持的各种选项。模式基于用户可以在键盘上选择的`地理方向`。例如，`--keywalk-west` 选项用于指定从`基础字符`向西移动。程序接收`基础字符`作为参数，这是模式将开始使用的字符集。接下来，它需要一个 `keymap`，该 `keymap` 映射了特定语言键盘布局上按键的位置。最后一个选项用于指定要使用的 `route`。`Route` 是密码将遵循的模式。它定义了密码如何形成，从`基础字符`开始。例如，`route 222` 可以表示从`基础字符`开始，向东移动 2 次 + 向南移动 2 次 + 向西移动 2 次的路径。如果`基础字符`被视为 "T"，那么在美国键盘布局上，该 `route` 生成的密码将是 "TYUJNBV"。更多信息请参阅 `kwprocessor` 的 `README` 文件。


```bash
kwp -s 1 basechars/full.base keymaps/en-us.keymap routes/2-to-10-max-3-direction-changes.route
```

上面的命令生成单词，这些单词包含按住 `Shift` 键 (`-s 1`) 可达的字符，使用完整的 `base` 字符集，标准的 `en-us.keymap`，以及 3 次方向改变的 `route`。

### Princeprocessor

`PRINCE` 或 `PRobability INfinite Chained Elements` 是一种高效的密码猜测算法，用于提高密码破解率。`Princeprocessor` 是一个使用 `PRINCE` 算法生成密码的工具。程序接收一个字典文件，并从该字典文件中取出单词生成链。例如，如果一个字典文件包含以下单词：

```bash
dog
cat
ball
```

生成的字典文件将是以下形式：

```
dog
cat
ball
dogdog
catdog
dogcat
catcat
dogball
catball
balldog
ballcat
ballball
dogdogdog
catdogdog
dogcatdog
catcatdog
dogdogcat
<SNIP>
```

`PRINCE` 算法在创建每个单词时考虑了各种排列和组合。`--keyspace` 选项可用于查找从输入字典文件生成的组合数量。

```bash
./pp64.bin --keyspace < words
```

```
232
```

根据 `princeprocessor` 的计算，从我们上面的字典文件中可以形成 232 个独特的单词。


```bash
./pp64.bin -o wordlist.txt < words
```

上面的命令将输出的单词写入名为 `wordlist.txt` 的文件中。默认情况下，`princeprocessor` 只输出长度不超过 16 的单词。这可以通过 `--pw-min` 和 `--pw-max` 参数控制。

```
./pp64.bin --pw-min=10 --pw-max=25 -o wordlist.txt < words
```

上面的命令将输出长度在 10 到 25 之间的单词。每个单词中的`元素数量`可以通过 `--elem-cnt-min` 和 `--elem-cnt-max` 控制。这些值确保输出单词中的`元素数量`高于或低于给定值。

```
./pp64.bin --elem-cnt-min=3 -o wordlist.txt < words
```

上面的命令将输出包含三个或更多元素的单词，例如 `"dogdogdog"`。

### CeWL

`CeWL` 是另一个可用于创建自定义字典文件的工具。它会爬取和抓取网站，并创建其中存在的单词列表。这种字典文件很有效，因为人们倾向于使用与他们撰写或操作内容相关的密码。例如，一位撰写自然、野生动物等主题博客的博主，其密码可能与这些主题相关。这是出于人性，因为这样的密码也容易记住。组织通常会使用与其品牌和行业特定词汇相关的密码。例如，一家网络公司的用户密码可能包含诸如 `router`、`switch`、`server` 等词语。这些词语可以在他们的网站上的博客、客户评价和产品描述中找到。

```bash
cewl -d <depth to spider> -m <minimum word length> -w <output wordlist> <url of website>
```

`CeWL` 可以爬取给定网站上的多个页面。可以使用 `-m` 参数更改输出单词的长度，具体取决于密码要求（即某些网站有最小密码长度限制）。

`CeWL` 还支持使用 `-e` 选项从网站提取电子邮件。在后续进行钓鱼、密码喷洒或暴力破解密码时，获取这些信息很有帮助。

```bash
cewl -d 5 -m 8 -e http://inlanefreight.com/blog -w wordlist.txt
```

上面的命令从 "`http://inlanefreight.com/blog`" 抓取深度达五页的内容，并且只包含长度大于 8 的单词。

### 先前已破解的密码 

默认情况下，`Hashcat` 将所有已破解的密码存储在 `hashcat.potfile` 文件中；格式是 `hash:password`。此文件的主要目的是从工作日志中移除先前已破解的哈希，并使用 `--show` 命令显示已破解的密码。

```bash
cut -d: -f 2- ~/hashcat.potfile
```

（这里的 `cut -d: -f 2-` 命令用于从 `hashcat.potfile` 文件中提取数据。`-d:` 表示使用冒号 `:` 作为字段分隔符。`-f 2-` 表示提取从第二个字段到最后一个字段的所有内容。由于 `hashcat.potfile` 的格式是 `哈希:密码`，这个命令实际上就是提取冒号后面的密码部分。）

### Hashcat-utils

`Hashcat-utils` 仓库包含许多对更高级密码破解有用的工具。例如，工具 `maskprocessor` 可以用于使用给定的掩码创建字典文件。此工具的详细用法可以在此处找到。`maskprocessor` 可以用于在单词末尾附加所有特殊字符：

```bash
/mp64.bin Welcome?s
```

```
Welcome
Welcome!
Welcome"
Welcome#
Welcome$
Welcome%
Welcome&
Welcome'
Welcome(
Welcome)
Welcome*
Welcome+

<SNIP>
```


## 使用规则

基于规则的攻击是最先进、最复杂的密码破解模式。规则有助于对输入的字典文件执行各种操作，例如添加前缀、添加后缀、切换大小写、剪切、反转等等。规则将基于掩码的攻击提升到新的水平，并提高了破解率。此外，使用规则可以节省因字典文件过大而产生的磁盘空间和处理时间。

规则可以使用**函数**创建，函数将一个单词作为输入并输出其修改后的版本。下表描述了一些与 `JtR` 和 `Hashcat` 兼容的函数。

| **Function**            | **Description**              | **Input**                                 | **Output**                                                                                                                |
| ----------------------- | ---------------------------- | ----------------------------------------- | ------------------------------------------------------------------------------------------------------------------------- |
| `l`                     | 将所有字母转换为小写                   | `InlaneFreight2020`                       | `inlanefreight2020`                                                                                                       |
| `u`                     | 将所有字母转换为大写                   | `InlaneFreight2020`                       | `INLANEFREIGHT2020`                                                                                                       |
| `c` / `C`               | 首字母大写/小写，其余字母反转大小写           | `inlaneFreight2020` / `Inlanefreight2020` | `Inlanefreight2020` / `iNLANEFREIGHT2020`                                                                                 |
| `t` / `tN`              | 切换大小写：整个单词 / 指定位置 N 处的字符     | `InlaneFreight2020`                       | `iNLANEfREIGHT2020`                                                                                                       |
| `d` / `q` / `zN` / `ZN` | 复制单词 / 所有字符 / 第一个字符 / 最后一个字符 | `InlaneFreight2020`                       | `InlaneFreight2020InlaneFreight2020` / `IInnllaanneeFFrreeiigghhtt22002200` / `IInlaneFreight2020` / `InlaneFreight20200` |
| `{` / `}`               | 将单词向左 / 向右循环移动               | `InlaneFreight2020`                       | `nlaneFreight2020I` / `0InlaneFreight202`                                                                                 |
| `^X` / `$X`             | 在单词前面 / 后面添加字符 X             | `InlaneFreight2020` (`^!` / `$!` )        | `!InlaneFreight2020` / `InlaneFreight2020!`                                                                               |
| `r`                     | 反转单词                         | `InlaneFreight2020`                       | `0202thgierFenalnI`                                                                                                       |

有时，输入字典文件包含与我们目标规范不匹配的单词。例如，公司的密码策略可能不允许用户设置长度小于 7 个字符的密码。在这种情况下，可以使用**拒绝规则**来阻止处理此类单词。

长度小于 N 的单词可以使用 `>N` 拒绝，而长度大于 N 的单词可以使用 `<N` 拒绝。

**注意：** 拒绝规则仅在使用 `hashcat-legacy` 或在使用 `Hashcat` 时配合 `-j` 或 `-k` 选项时起作用。它们不能作为常规规则（在规则文件中）与 `Hashcat` 一起使用。

让我们看看如何基于常见密码创建规则。通常用户倾向于用相似的数字替换字母，例如 "o" 可以替换为 "0"，或者 "i" 可以替换为 "1"。这通常被称为 **`L33tspeak`**，效率很高。公司密码经常在前面或后面附加年份。让我们创建一个生成此类单词的规则。

**规则 (Rules)**

```
c so0 si1 se3 ss5 sa@ $2 $0 $1 $9
```

单词的首字母使用 `c` 函数大写。然后规则使用替换函数 `s` 将 `o` 替换为 `0`，将 `i` 替换为 `1`，将 `e` 替换为 `3`，并将 `a` 替换为 `@`。最后，在单词后面附加年份 `2019`。将规则复制到文件，以便我们可以调试它。

```bash
echo 'c so0 si1 se3 ss5 sa@ $2 $0 $1 $9' > rule.txt
```

```bash
echo 'password_ilfreight' > test.txt
```

可以使用 `-r` 标志指定规则，后面跟着字典文件

```bash
hashcat -r rule.txt test.txt --stdout
```

```bash
P@55w0rd_1lfr31ght2019
```

正如预期的那样，首字母被大写，字母被数字替换了。让我们考虑密码 `St@r5h1p2019`。

```bash
echo -n 'St@r5h1p2019' | sha1sum | awk '{print $1}' | tee hash
```

**命令解释：**

- `sha1sum`：计算输入内容的 `SHA1` 哈希值。默认输出格式是 `哈希值 文件名` 或 `哈希值 -` (如果从标准输入读取)。
- `awk '{print $1}'`：`awk` 命令是一个强大的文本处理工具。`'{print $1}'` 是 `awk` 脚本，表示对于每一行输入，打印第一个字段 (`$1`)。在这里，它从 `sha1sum` 的输出中提取第一个字段（即哈希值本身），忽略后面的 `-`。
- `tee hash`：`tee` 命令将接收到的标准输入**既**输出到标准输出，**也**写入到指定的文件（这里是 `hash` 文件）。最终，`SHA1` 哈希值会显示在屏幕上，同时也被保存在 `hash` 文件中。

然后，我们可以使用上面创建的自定义规则和 `rockyou.txt` 字典文件，使用 `Hashcat` 破解哈希。

```Bash
hashcat -a 0 -m 100 hash /opt/useful/seclists/Passwords/Leaked-Databases/rockyou.txt -r rule.txt
```

我们成功使用自定义规则和 `rockyou.txt` 破解了哈希。`Hashcat` 支持使用多个规则，通过重复使用 `-r` 标志即可。`Hashcat` 默认安装时会附带各种规则。它们可以在 rules 文件夹中找到。

```Bash
ls -l /usr/share/hashcat/rules/
```

```
total 2576
-rw-r--r-- 1 root root    933 Jun 19 06:20 best64.rule
-rw-r--r-- 1 root root    633 Jun 19 06:20 combinator.rule
-rw-r--r-- 1 root root 200188 Jun 19 06:20 d3ad0ne.rule
-rw-r--r-- 1 root root 788063 Jun 19 06:20 dive.rule
-rw-r--r-- 1 root root 483425 Jun 19 06:20 generated2.rule
-rw-r--r-- 1 root root  78068 Jun 19 06:20 generated.rule
drwxr-xr-x 1 root root   2804 Jul  9 21:01 hybrid
-rw-r--r-- 1 root root 309439 Jun 19 06:20 Incisive-leetspeak.rule
-rw-r--r-- 1 root root  35280 Jun 19 06:20 InsidePro-HashManager.rule
-rw-r--r-- 1 root root  19478 Jun 19 06:20 InsidePro-PasswordsPro.rule
-rw-r--r-- 1 root root    298 Jun 19 06:20 leetspeak.rule
-rw-r--r-- 1 root root   1280 Jun 19 06:20 oscommerce.rule
-rw-r--r-- 1 root root 301161 Jun 19 06:20 rockyou-30000.rule
-rw-r--r-- 1 root root   1563 Jun 19 06:20 specific.rule
-rw-r--r-- 1 root root  64068 Jun 19 06:20 T0XlC-insert_00-99_1950-2050_toprules_0_F.rule
-rw-r--r-- 1 root root   2027 Jun 19 06:20 T0XlC-insert_space_and_special_0_F.rule
-rw-r--r-- 1 root root  34437 Jun 19 06:20 T0XlC-insert_top_100_passwords_1_G.rule
-rw-r--r-- 1 root root  34813 Jun 19 06:20 T0XlC.rule
-rw-r--r-- 1 root root 104203 Jun 19 06:20 T0XlCv1.rule
-rw-r--r-- 1 root root     45 Jun 19 06:20 toggles1.rule
-rw-r--r-- 1 root root    570 Jun 19 06:20 toggles2.rule
-rw-r--r-- 1 root root   3755 Jun 19 06:20 toggles3.rule
-rw-r--r-- 1 root root  16040 Jun 19 06:20 toggles4.rule
-rw-r--r-- 1 root root  49073 Jun 19 06:20 toggles5.rule
-rw-r--r-- 1 root root  55346 Jun 19 06:20 unix-ninja-leetspeak.rule
```

最好在创建自定义规则之前先尝试使用这些默认规则。

`Hashcat` 提供了一个选项，可以即时生成随机规则并将其应用于输入字典文件。通过指定 `-g` 标志，以下命令将生成 1000 条随机规则并将其应用于 `rockyou.txt` 中的每个单词。由于生成的规则不是固定的，此攻击的成功率没有确定性。


```Bash
hashcat -a 0 -m 100 -g 1000 hash /opt/useful/seclists/Passwords/Leaked-Databases/rockyou.txt
```

还有各种公开可用的规则，例如 `nsa-rules`、`Hob0Rules` 和 `corporate.rule`（该规则收录在书籍《How to Hack Like a Legend》中）。这些是经过整理的规则集，通常针对常见的企业 Windows 密码策略，或基于统计数据和可能的行业密码模式。

## 破解常见哈希

### 常见哈希类型

在渗透测试任务中，我们会遇到各种各样的哈希类型；有些非常常见，几乎在所有任务中都能看到，而另一些则非常罕见或根本看不到。如前所述，Hashcat 的开发者维护了一个 Hashcat 支持的大多数哈希模式的 [示例哈希](https://hashcat.net/wiki/doku.php?id=example_hashes) 列表。该列表包含哈希模式、哈希名称以及指定类型的示例哈希。一些最常见的哈希如下：

| **Hashmode** | **Hash Name**                | **Example Hash**                                                                                                                                                                    |
| ------------ | ---------------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| 0            | MD5                          | 8743b52063cd84097a65d1633f5c74f5                                                                                                                                                    |
| 100          | SHA1                         | b89eaac7e61417341b710b727768294d0e6a277b                                                                                                                                            |
| 1000         | NTLM                         | b4b9b02e6f09a9bd760f388b67351e2b                                                                                                                                                    |
| 1800         | sha512crypt 6, SHA512 (Unix) | $6$52450745$k5ka2p8bFuSmoVT1tzOyyuaREkkKBcCNqoDKzYiJL9RaE8yMnPgh2XzzF0NDrUhgrcLwg78xs1w5pJiypEdFX/3                                                                                 |
| 3200         | bcrypt 2∗, Blowfish (Unix)   | $2a$05$LhayLxezLhK1LhWvKxCyLOj0j1u.Kj0jZ0pEmm134uzrQlFvQJLF                                                                                                                         |
| 5500         | NetNTLMv1 / NetNTLMv1+ESS    | u4-netntlm::kNS:338d08f8e26de93300000000000000000000000000000000:9526fb8c23a90751cdd619b6cea564742e1e4bf33006ba41:cb8086049ec4736c                                                  |
| 5600         | NetNTLMv2                    | admin::N46iSNekpT:08ca45b7d7ea58ee:88dcbe4446168966a153a0064958dac6:5c7830315c7830310000000000000b45c67103d07d7b95acd12ffa11230e0000000052920b85f78d013c31cdb3b92f5d765c78303013100 |
| 13100        | Kerberos 5 TGS-REP etype 23  | $krb5tgs$23$user$realm$test/spn$63386d22d359fe42230300d56852c9eb$ < SNIP >                                                                                                          |

### 数据库转储

MD5、SHA1 和 bcrypt 哈希通常出现在数据库转储中。这些哈希可能是在成功的 SQL 注入攻击后检索到的，或者在公开可用的密码数据泄露数据库转储中找到。MD5 和 SHA1 通常比 bcrypt 更容易破解，bcrypt 可能应用了多轮 Blowfish 算法。

### Linux Shadow 文件

Sha512crypt 哈希通常在 Linux 系统的 `/etc/shadow` 文件中被应用。此文件包含分配了登录 shell 的所有帐户的密码哈希。在渗透测试期间，我们可能通过 Web 应用程序攻击或成功利用易受攻击的服务获得 Linux 系统的访问权限。我们可能利用一个已在最高权限 root 帐户上下文中运行的服务，并执行成功的权限提升攻击并访问 `/etc/shadow` 文件。密码重用是普遍存在的。破解的密码可能使我们能够访问其他服务器、网络设备，甚至可以用作进入目标 Active Directory 环境的立足点。

让我们看一下标准 Ubuntu 安装中的一个哈希。以下哈希对应的明文是 `password123`。

```
root:$6$tOA0cyybhb/Hr7DN$htr2vffCWiPGnyFOicJiXJVMbk1muPORR.eRGYfBYUnNPUjWABGPFiphjIjJC5xPfFUASIbVKDAHS3vTW1qU.1:18285:0:99999:7:::
```

该哈希包含九个由冒号分隔的字段。前两个字段包含用户名及其加密哈希。其余字段包含各种属性，例如密码创建时间、上次更改时间和过期时间。

回到哈希本身，我们已经知道它包含三个由 "$" 分隔的字段。"6" 表示 SHA-512 哈希算法；接下来的 16 个字符表示盐值，其余部分是实际的哈希。

让我们使用 Hashcat 破解这个哈希。

```
hashcat -m 1800 nix_hash /opt/useful/seclists/Passwords/Leaked-Databases/rockyou.txt
```

```
hashcat (v6.1.1) starting...
<SNIP>
$6$tOA0cyybhb/Hr7DN$htr2vffCWiPGnyFOicJiXJVMbk1muPORR.eRGYfBYUnNPUjWABGPFiphjIjJC5xPfFUASIbVKDAHS3vTW1qU.1:password123

Session..........: hashcat
Status...........: Cracked
Hash.Name........: sha512crypt $6$, SHA512 (Unix)
Hash.Target......: $6$tOA0cyybhb/Hr7DN$htr2vffCWiPGnyFOicJiXJVMbk1muPO...W1qU.1
Time.Started.....: Fri Aug 28 22:25:26 2020, (1 sec)
Time.Estimated...: Fri Aug 28 22:25:27 2020, (0 secs)
Guess.Base.......: File (/opt/useful/seclists/Passwords/Leaked-Databases/rockyou.txt)
Guess.Queue......: 1/1 (100.00%)
Speed.#1.........:      955 H/s (4.62ms) @ Accel:32 Loops:256 Thr:1 Vec:4
Recovered........: 1/1 (100.00%) Digests
Progress.........: 1536/14344385 (0.01%)
Rejected.........: 0/1536 (0.00%)
Restore.Point....: 1344/14344385 (0.01%)
Restore.Sub.#1...: Salt:0 Amplifier:0-1 Iteration:4864-5000
Candidates.#1....: teacher -> mexico1
```

### 常见的 Active Directory 密码哈希类型

在针对使用 Active Directory 管理其环境的组织进行评估时，凭据盗窃和密码重用是普遍存在的策略。通常可以通过 Pass-the-Hash 或 SMB Relay 攻击获取明文凭据或重用密码哈希以进一步访问。但是，某些技术会产生必须离线破解的密码哈希，以进一步扩大我们的访问权限。一些示例包括通过中间人 (MITM) 攻击获得的 NetNTLMv1 或 NetNTLMv2，通过 Kerberoasting 攻击获得的 Kerberos 5 TGS-REP 哈希，或通过使用 `Mimikatz` 工具从内存中转储凭据或从 Windows 机器本地 SAM 数据库获得的 NTLM 哈希。

#### NTLM

让我们来看一个示例。我们可以使用三行 Python 代码快速生成密码 "Password01" 的 NTLM 哈希以供我们使用：

```bash
Python 3.8.3 (default, May 14 2020, 11:03:12)
[GCC 9.3.0] on linux
Type "help", "copyright", "credits" or "license" for more information.

>>> import hashlib,binascii
>>> hash = hashlib.new('md4', "Password01".encode('utf-16le')).digest()
>>> print (binascii.hexlify(hash))

b'7100a909c7ff05b266af3c42ec058c33'
```

然后我们可以使用标准的 `rockyou.txt` 字典文件通过 Hashcat 来运行生成的 NTLM 密码哈希值 `7100a909c7ff05b266af3c42ec058c33。

#### 破解 NTLM 哈希

```
hashcat -a 0 -m 1000 ntlm_example /opt/useful/seclists/Passwords/Leaked-Databases/rockyou.txt
```

```
hashcat (v6.1.1) starting...
<SNIP>

7100a909c7ff05b266af3c42ec058c33:Password01

Session..........: hashcat
Status...........: Cracked
Hash.Name........: NTLM
Hash.Target......: 7100a909c7ff05b266af3c42ec058c33
Time.Started.....: Fri Aug 28 22:27:40 2020, (0 secs)
Time.Estimated...: Fri Aug 28 22:27:40 2020, (0 secs)
Guess.Base.......: File (/opt/useful/seclists/Passwords/Leaked-Databases/rockyou.txt)
Guess.Queue......: 1/1 (100.00%)
Speed.#1.........:  2110.5 kH/s (0.62ms) @ Accel:1024 Loops:1 Thr:1 Vec:8
Recovered........: 1/1 (100.00%) Digests
Progress.........: 61440/14344385 (0.43%)
Rejected.........: 0/61440 (0.00%)
Restore.Point....: 55296/14344385 (0.39%)
Restore.Sub.#1...: Salt:0 Amplifier:0-1 Iteration:0-1
Candidates.#1....: gonoles -> sinead1
```

#### NetNTLMv2

在渗透测试期间，通常会运行 Responder 等工具执行 MITM 攻击，以尝试“窃取”凭据。这些类型的攻击在其他模块中详细介绍。在繁忙的企业网络中，使用这种方法检索许多 NetNTLMv2 密码哈希是很常见的。这些哈希通常可以被破解并用于在 Active Directory 环境中建立立足点，有时甚至可以获得许多或所有系统的完全管理权限，这取决于与密码哈希关联的用户帐户被授予的权限。考虑以下在评估开始时使用 Responder 检索到的密码哈希：

#### Responder - NTLMv2

```
sqladmin::INLANEFREIGHT:f54d6f198a7a47d4:7FECABAE13101DAAA20F1B09F7F7A4EA:0101000000000000C0653150DE09D20126F3F71DF13C1FD8000000000200080053004D004200330001001E00570049004E002D00500052004800340039003200520051004100460056000400140053004D00420033002E006C006F00630061006C0003003400570049004E002D00500052004800340039003200520051004100460056002E0053004D0033002E006C006F00630061006C000500140053004D00420033002E006C006F00630061006C0007000800C0653150DE09D201060004000200000008003000300000000000000000000000003000001A67637962F2B7BF297745E6074934196D5F4371B6BA3E796F2997306FD4C1C00A001000000000000000000000000000000000000900280063006900660073002F003100390032002E003100360038002E003100390035002E00310037003000000000000000000000000000
```

Responder 等一些工具会告知您收到的哈希类型。如果不确定，我们还可以查看 [示例哈希](https://hashcat.net/wiki/doku.php?id=example_hashes) 页面，确认这确实是 NetNTLMv2 哈希，在 Hashcat 中模式为 5600。

#### 破解 NTLMv2 哈希

```
hashcat -a 0 -m 5600 inlanefreight_ntlmv2 /opt/useful/seclists/Passwords/Leaked-Databases/rockyou.txt
```

```
hashcat (v6.1.1) starting...
<SNIP>

SQLADMIN::INLANEFREIGHT:f54d6f198a7a47d4:7fecabae13101daaa20f1b09f7f7a4ea:0101000000000000c0653150de09d20126f3f71df13c1fd8000000000200080053004d004200330001001e00570049004e002d00500052004800340039003200520051004100460056000400140053004d00420033002e006c006f00630061006c0003003400570049004e002d00500052004800340039003200520051004100460056002e0053004d00420033002e006c006f00630061006c000500140053004d00420033002e006c006f00630061006c0007000800c0653150de09d201060004000200000008003000300000000000000000000000003000001a67637962f2b7bf297745e6074934196d5f4371b6ba3e796f2997306fd4c1c00a001000000000000000000000000000000000000900280063006900660073002f003100390032002e00310036802e003100390035002e00310037003000000000000000000000000000:Database99

Session..........: hashcat
Status...........: Cracked
Hash.Name........: NetNTLMv2
Hash.Target......: SQLADMIN::INLANEFREIGHT:f54d6f198a7a47d4:7fecabae13...000000
Time.Started.....: Fri Aug 28 22:29:26 2020, (6 secs)
Time.Estimated...: Fri Aug 28 22:29:32 2020, (0 secs)
Guess.Base.......: File (/opt/useful/seclists/Passwords/Leaked-Databases/rockyou.txt)
Guess.Queue......: 1/1 (100.00%)
Speed.#1.........:  1754.7 kH/s (2.32ms) @ Accel:1024 Loops:1 Thr:1 Vec:8
Recovered........: 1/1 (100.00%) Digests
Progress.........: 11237376/14344385 (78.34%)
Rejected.........: 0/11237376 (0.00%)
Restore.Point....: 11231232/14344385 (78.30%)
Restore.Sub.#1...: Salt:0 Amplifier:0-1 Iteration:0-1
Candidates.#1....: Devanique -> Darrylw
```

## 破解杂项文件哈希

我们经常会遇到受密码保护的文档，例如 Microsoft Word 和 Excel 文档、OneNote 笔记本、KeePass 数据库文件、SSH 私钥密码短语、PDF 文件、zip文件等等。大多数这些哈希都可以通过 Hashcat 来尝试破解。

存在各种工具来帮助我们从这些文件中提取密码哈希，使其成为 Hashcat 可以解析的格式。密码破解工具 JohnTheRipper 附带了许多用 C 编写的工具。

还有许多这些工具的 Python 移植版本，它们非常易于使用。其中大部分包含在 JohnTheRipper jumbo GitHub 仓库中，位于 [这里](https://github.com/magnumripper/JohnTheRipper/tree/bleeding-jumbo/run)。

### 破解Microsoft Office 文档

Hashcat 可以用来尝试破解使用 `office2john`工具从Microsoft Office 文档中提取的密码哈希。

Hashcat 支持以下 Microsoft Office 文档的哈希模式：

| **Mode** | **Target**     |
| -------- | -------------- |
| 9400     | MS Office 2007 |
| 9500     | MS Office 2010 |
| 9600     | MS Office 2013 |

对于早于 2003 的 MS Office 文档，还有几种 "oldoffice" 哈希模式。让我们以一个密码为 "pa55word" 的 Word 文档为例。我们可以首先使用 `office2john.py` 从文档中提取哈希。

```
office2john hashcat_Word_example.docx
```

```
hashcat_Word_example.docx:$office$*2013*100000*256*16*6e059661c3ed733f5730eaabb41da13a*aa38e007ee01c07e4fe95495934cf68f*2f1e2e9bf1f0b320172cd667e02ad6be1718585b6594691907b58191a6
```

然后我们可以使用模式 9600 在 Hashcat 中运行该哈希，并使用 rockyou.txt 字典文件快速破解它。这是一种相当慢的哈希类型，在单个 CPU 上运行完整的 rockyou.txt 字典文件将需要超过 12 小时。在 GPU 或多个 GPU 上会快得多，但仍然比 MD5 和 NTLM 等其他哈希慢得多。

#### 破解 MS Office 密码

```
hashcat -m 9600 office_hash /opt/useful/seclists/Passwords/Leaked-Databases/rockyou.txt
```
### 破解 Zip 文件

Hashcat 支持多种压缩文件格式，例如：

| **Mode** | **Target**                                  |
| -------- | ------------------------------------------- |
| 11600    | 7-Zip                                       |
| 13600    | WinZip                                      |
| 17200    | PKZIP (Compressed)                          |
| 17210    | PKZIP (Uncompressed)                        |
| 17220    | PKZIP (Compressed Multi-File)               |
| 17225    | PKZIP (Mixed Multi-File)                    |
| 17230    | PKZIP (Compressed Multi-File Checksum-Only) |
| 23001    | SecureZIP AES-128                           |
| 23002    | SecureZIP AES-192                           |
| 23003    | SecureZIP AES-256                           |

为 ZIP 文件设置密码：

```
zip --password zippyzippy blueprints.zip dummy.pdf
```

```
adding: dummy.pdf (deflated 7%)
```

然后我们可以使用 `zip2john` 版本提取哈希。

```
zip2john blueprints.zip
```

```
ver 2.0 efh 5455 efh 7875 blueprints.zip/dummy.pdf PKZIP Encr: 2b chk, TS_chk, cmplen=12324, decmplen=13264, crc=7EB29321
blueprints.zip/dummy.pdf:$pkzip2$1*2*2*0*3024*33d0*7eb29321*0*43*8*3024*7eb2*69f2*d796c9cde7b7ed8d7b76c1efd12d222d2bfcc7a2e5a94b21a55c965c36c5875ea17ba1ca63d8164dc214c8845fa42d4f5953792634e860360cf9936199f989f1e4ae65e75cd8443cd4973eff78c418b146e4155c33d977f8ca4e30b2e52cc8c8de3884545752ad2a4c36ea69ec07d638a664acc69fc4c9ce1fd30975587ead31ccd7c4ba791fa57e3610995f437332b0c5c48580a582986c1c9891cdeccf876655ea965a51e35ff045d685c23e365876df4c58e88882d581effac3b9effadf9ce2deeeebd18fb6a16e691a22c869c5da3cc0e2979f301a7eb6305cee3228300e673fc47cf2726e739b617e49a11f50e85bdb68df2fc876410a7cf661370ba63ab11ae13299cc27f5530753820c*$/pkzip2$:dummy.pdf:blueprints.zip::blueprints.zip
```

从这个哈希中我们可以看到，这是模式 17200 - PKZIP (Compressed)。要在 Hashcat 中运行它，我们需要从 `$pkzip2$1` 开始到 `/pkzip2$` 结束的整个哈希。有了哈希，我们来使用 Hashcat 进行纯字典攻击。

#### Hashcat - 破解 ZIP 文件

```
hashcat -a 0 -m 17200 pdf_hash_to_crack /opt/useful/seclists/Passwords/Leaked-Databases/rockyou.txt
```

```
hashcat (v6.1.1) starting...

<SNIP>
$pkzip2$1*2 <FULL HASH SNIPPED> k*$/pkzip2$:zippyzippy

Session..........: hashcat
Status...........: Cracked
Hash.Name........: PKZIP (Compressed)
Hash.Target......: $pkzip2$1*2*2*0*3024*33d0*7eb29321*0*43*8*3024*7eb2...kzip2$
Time.Started.....: Fri Aug 28 22:34:46 2020, (1 sec)
Time.Estimated...: Fri Aug 28 22:34:47 2020, (0 secs)
Guess.Base.......: File (/opt/useful/seclists/Passwords/Leaked-Databases/rockyou.txt)
Guess.Queue......: 1/1 (100.00%)
Speed.#1.........:  3665.1 kH/s (0.32ms) @ Accel:1024 Loops:1 Thr:1 Vec:8
Recovered........: 1/1 (100.00%) Digests
Progress.........: 2500608/14344385 (17.43%)
Rejected.........: 0/2500608 (0.00%)
Restore.Point....: 2494464/14344385 (17.39%)
Restore.Sub.#1...: Salt:0 Amplifier:0-1 Iteration:0-1
Candidates.#1....: zj4usm0z -> zietz5632
Started: Fri Aug 28 22:34:24 2020
Stopped: Fri Aug 28 22:34:48 2020
```

现在我们可以使用这个密码来提取 zip 文件中的内容。

### 示例 3 - 破解受密码保护的 KeePass 文件

系统管理员的工作站上或可访问的文件共享上。这些文件通常是凭据的宝库，因为系统管理员、网络管理员、帮助台等可能将各种密码存储在共享的 KeePass 数据库中。获得访问权限可能会提供 Windows 机器的本地管理员密码、ESXi 和 vCenter 等基础设施的密码、对网络设备的访问等等。

我们可以使用 `keepass2john` 工具提取这些哈希。Hashcat 支持多种压缩文件格式，例如：

Hashcat 支持以下 KeePass 数据库的哈希名称，所有这些都由相同的哈希模式指定：

| **Mode** | **Target**                       |
| -------- | -------------------------------- |
| 13400    | KeePass 1 AES / without keyfile  |
| 13400    | KeePass 2 AES / without keyfile  |
| 13400    | KeePass 1 Twofish / with keyfile |
| 13400    | Keepass 2 AES / with keyfile     |

```
keepass2john Master.kdbx
```

```
Master:$keepass$*2*60000*222*d14132325949a3b4efacdb2e729ec54403308c85654fe4ababccfb8ddc185d09*5c09bed9c98f8ee08aa7a71fe735b30849ec87e6cb7f1caa96d606ce9f077f7e*bd372d79d8aceea9689ad49428b8efde*28d21caedf25617db0833bd721a42c963e874e0b9fbe7fe1187a4a8ecb3b1d19*a539abd3cfd7ee5982fa28c44dd226ce05a1102d04a5f590eabf5138cd2a6403
```

从输出中我们可以看到，这确实是一个 KeePass 哈希，类型是 `KeePass 2 AES / without keyfile`。有了哈希，我们就可以使用 Hashcat 字典攻击来攻击它。这种哈希使用 AES 算法，比 MD5 或 SHA1 等其他哈希更难破解，在 Hashcat 中运行速度更慢。因此，非常复杂的密码可能难以破解，但 rockyou.txt 等字典文件中的密码在单个 CPU 上破解可能需要大约 8 小时，但在 GPU 破解设备上破解速度会呈指数级增长。

#### 破解 KeePass 文件

```
hashcat -a 0 -m 13400 keepass_hash /opt/useful/seclists/Passwords/Leaked-Databases/rockyou.txt
```


### 破解受保护的 PDF 文件

与其他文件类型一样，我们在工作站、文件共享甚至用户的电子邮件收件箱中经常遇到受密码保护的 PDF。

我们可以使用 `pdf2john` 提取密码短语的哈希。

```
pdf2john inventory.pdf | awk -F":" '{ print $2}'
```

```
$pdf$4*4*128*-1028*1*16*f7d77b3d22b9f92829d49ff5d78b8f28*32*d33f35f776215527d65155f79d9ed79800000000000000000000000000000000*32*6cfb859c107acaae8c0ca9ceec56fd91ff75fe7b1cddb03f629ca3583f59e52f
```

Hashcat 支持多种压缩文件格式，例如：

| **Mode** | **Target**                                 |
| -------- | ------------------------------------------ |
| 10400    | PDF 1.1 - 1.3 (Acrobat 2 - 4)              |
| 10410    | PDF 1.1 - 1.3 (Acrobat 2 - 4), collider #1 |
| 10420    | PDF 1.1 - 1.3 (Acrobat 2 - 4), collider #2 |
| 10500    | PDF 1.4 - 1.6 (Acrobat 5 - 8)              |
| 10600    | PDF 1.7 Level 3 (Acrobat 9)                |
| 10700    | PDF 1.7 Level 8 (Acrobat 10 - 11)          |

我们可以使用模式 10500 破解哈希。

####  破解 PDF 文件

```
hashcat -a 0 -m 10500 pdf_hash /opt/useful/seclists/Passwords/Leaked-Databases/rockyou.txt
```

## 使用 Hashcat 破解无线 (WPA/WPA2) 握手包

如果无线网络通常与公司的企业网络没有正确隔离，成功认证到无线网络可能会授予对内部企业网络的完全访问权限。

Hashcat 可以成功破解 MIC (四次握手) 和 PMKID (第一个数据包/握手包)。

### 破解 MIC

当客户端连接到无线网络并与无线接入点 (AP) 通信时，它们必须确保都拥有/知道无线网络密钥，但不能在网络上传输该密钥。密钥由 AP 加密和验证。

要执行这种离线破解攻击，我们需要通过发送去认证帧来强制客户端（用户）断开与 AP 的连接，从而捕获一个有效的四次握手包。当客户端重新认证（通常是自动）时，攻击者可以尝试在他们不知情的情况下嗅探 WPA 四次握手包。这个握手包是客户端和关联 AP 在认证过程中交换的密钥集合。注意：无线攻击不在本模块的范围内，但在其他模块中会介绍。

这些密钥用于生成一个称为消息完整性检查 (MIC) 的通用密钥，AP 使用它来验证每个数据包是否未被破坏并以原始状态接收。

四次握手包在下图中所示：

![[Pasted image 20250508142401.png]]

一旦我们使用 airodump-ng 等工具成功捕获了四次握手包，我们需要将其转换为 Hashcat 可以理解的格式进行破解。所需的格式是 hccapx，Hashcat 提供了一个在线服务来转换为此格式：cap2hashcat online。要离线执行转换，我们需要从 GitHub 获取 hashcat-utils 仓库。

#### Hashcat-Utils - 安装

```
git clone https://github.com/hashcat/hashcat-utils.git
cd hashcat-utils/src
make
```

工具编译完成后，我们可以运行它并查看用法选项：

#### Cap2hccapx - 语法

```
./cap2hccapx.bin usage: ./cap2hccapx.bin input.cap output.hccapx [filter by essid] [additional network essid:bssid]
```

接下来，我们需要向工具提供一个数据包捕获 (.cap) 文件，以转换为 .hccapx 格式提供给 Hashcat。

#### Cap2hccapx - 转换为可破解文件

```
./cap2hccapx.bin corp_capture1-01.cap mic_to_crack.hccapx
```

```
Networks detected: 1

[*] BSSID=cc:40:d0:a4:d0:96 ESSID=CORP-WIFI (Length: 9)
 --> STA=48:e2:44:a7:c4:fb, Message Pair=0, Replay Counter=1
 --> STA=48:e2:44:a7:c4:fb, Message Pair=2, Replay Counter=1
 --> STA=48:e2:44:a7:c4:fb, Message Pair=0, Replay Counter=1
 --> STA=48:e2:44:a7:c4:fb, Message Pair=2, Replay Counter=1
 --> STA=48:e2:44:a7:c4:fb, Message Pair=0, Replay Counter=1
 --> STA=48:e2:44:a7:c4:fb, Message Pair=2, Replay Counter=1
 --> STA=48:e2:44:a7:c4:fb, Message Pair=0, Replay Counter=1
 --> STA=48:e2:44:a7:c4:fb, Message Pair=2, Replay Counter=1

Written 8 WPA Handshakes to: /home/mrb3n/Desktop/mic_to_crack.hccapx
```

有了这个文件，我们就可以使用本模块前面讨论的一种或多种技术进行破解。在本例中，我们将执行纯字典攻击来破解 WPA 握手包。为了尝试破解此哈希，我们将使用模式 22000，因为之前的模式 2500 已被弃用。我们破解此哈希的命令将如下所示：`hashcat -a 0 -m 22000 mic_to_crack.hccapx /opt/useful/seclists/Passwords/Leaked-Databases/rockyou.txt`。

我们来尝试恢复密钥！

#### Hashcat - 破解 WPA 握手包

```
hashcat -a 0 -m 22000 mic_to_crack.hccapx /opt/useful/seclists/Passwords/Leaked-Databases/rockyou.txt
```

```
hashcat (v6.1.1) starting...

<SNIP>

18cbc1c03cd674c75bb81aee4a75a086:cc40d0a4d096:48e244a7c4fb:CORP-WIFI:rockyou1
62b1bb7345e110abaaf8304c096239b0:cc40d0a4d096:48e244a7c4fb:CORP-WIFI:rockyou1
be2430ce7a4ed2ddb36fc94373197add:cc40d0a4d096:48e244a7c4fb:CORP-WIFI:rockyou1
15c472b7641042af642fc9ec0b65b500:cc40d0a4d096:48e244a7c4fb:CORP-WIFI:rockyou1

Session..........: hashcat
Status...........: Cracked
Hash.Name........: WPA-PBKDF2-PMKID+EAPOL
Hash.Target......: mic_to_crack.hccapx
Time.Started.....: Wed Mar  9 11:20:36 2022 (0 secs)
Time.Estimated...: Wed Mar  9 11:20:36 2022 (0 secs)
Guess.Base.......: File (/usr/share/wordlists/rockyou.txt)
Guess.Queue......: 1/1 (100.00%)
Speed.#1.........:    10052 H/s (8.25ms) @ Accel:128 Loops:512 Thr:1 Vec:8
Recovered........: 4/4 (100.00%) Digests
Progress.........: 2888/14344385 (0.02%)
Rejected.........: 2120/2888 (73.41%)
Restore.Point....: 0/14344385 (0.00%)
Restore.Sub.#1...: Salt:0 Amplifier:0-1 Iteration:3-7
Candidates.#1....: 123456789 -> celtic07
Started: Wed Mar  9 11:20:26 2022
Stopped: Wed Mar  9 11:20:38 2022
```

有了这个密钥，我们现在就可以尝试认证到无线网络并尝试访问内部企业网络。

### 破解 PMKID

这种攻击可以针对使用 WPA/WPA2-PSK（预共享密钥）的无线网络执行，并通过直接攻击 AP 来获取目标无线网络使用的 PSK。这种攻击不需要对目标 AP 的任何用户进行去认证。PMK 与 MIC（四次握手）攻击中的 PMK 相同，但通常可以更快地获取，而不会中断任何用户。

配对主密钥标识符 (PMKID) 是 AP 用于跟踪客户端使用的配对主密钥 (PMK) 的唯一标识符。PMKID 位于四次握手包的第一个数据包中，并且更容易获取，因为它不需要捕获整个四次握手包。PMKID 使用 HMAC-SHA1 计算，PMK（无线网络密码）用作密钥，字符串“PMK Name”，接入点的 MAC 地址和站点的 MAC 地址。下面是 PMKID 计算的表示：

![[Pasted image 20250508142614.png]]

要执行 PMKID 破解，我们需要获取 pmkid 哈希。第一步是使用 hcxtools 中的 hcxpcapngtool 等工具从捕获 (.cap) 。

注意：过去，这种技术可以使用 hcxpcaptool 执行，但它已被 hcxpcapngtool 替代。
#### Hcxpcapngtool - 帮助

```
hcxpcapngtool -h
```

```
hcxpcapngtool 6.3.5-44-g6be8d76 (C) 2025 ZeroBeat
convert pcapng, pcap and cap files to hash formats that hashcat and JtR use
usage:
hcxpcapngtool <options>
hcxpcapngtool <options> input.pcapng
hcxpcapngtool <options> *.pcapng
hcxpcapngtool <options> *.pcap
hcxpcapngtool <options> *.cap
hcxpcapngtool <options> *.*

short options:
-o <file> : output WPA-PBKDF2-PMKID+EAPOL hash file (hashcat -m 22000)
            get full advantage of reuse of PBKDF2 on PMKID and EAPOL
-E <file> : output wordlist (autohex enabled on non ASCII characters) to use as input wordlist for cracker
            retrieved from every frame that contain an ESSID
-R <file> : output wordlist (autohex enabled on non ASCII characters) to use as input wordlist for cracker
            retrieved from PROBEREQUEST frames only
-I <file> : output unsorted identity list to use as input wordlist for cracker
-U <file> : output unsorted username list to use as input wordlist for cracker
-D <file> : output device information list
            format MAC MANUFACTURER MODELNAME SERIALNUMBER DEVICENAME UUID
-h        : show this help
-v        : show version

<SNIP>
```

尽管该工具可用于各种任务，但我们可以使用 hcxpcapngtool 按如下方式提取哈希：

```
hcxpcapngtool cracking_pmkid.cap -o pmkidhash_corp
```

```
reading from cracking_pmkid.cap...

summary capture file
--------------------
file name................................: cracking_pmkid.cap
version (pcapng).........................: 1.0
operating system.........................: Linux 5.7.0-kali1-amd64
application..............................: hcxdumptool 6.0.7-22-g2f82e84
interface name...........................: wlan0
interface vendor.........................: 00c0ca
weak candidate...........................: 12345678
MAC ACCESS POINT.........................: 0c8112953006 (incremented on every new client)
MAC CLIENT...............................: fcc23374f354
REPLAYCOUNT..............................: 63795
ANONCE...................................: 4e0fee9e1a8961ca63b74023d90ac081d8677ae748b7050a559cf481cf50d31f
SNONCE...................................: 90d86a9fc2a314df52b3b36b9080c88e90488594f0aa83e84196bfce8b90d1ac
timestamp minimum (GMT)..................: 17.07.2020 10:07:19
timestamp maximum (GMT)..................: 17.07.2020 10:14:21
used capture interfaces..................: 1
link layer header type...................: DLT_IEEE802_11_RADIO (127)
endianess (capture system)...............: little endian
packets inside...........................: 75
frames with correct FCS..................: 75
BEACON (total)...........................: 1
PROBEREQUEST.............................: 3
PROBERESPONSE............................: 1
EAPOL messages (total)...................: 69
EAPOL RSN messages.......................: 69
ESSID (total unique).....................: 3
EAPOLTIME gap (measured maximum usec)....: 172313401
EAPOL ANONCE error corrections (NC)......: working
REPLAYCOUNT gap (suggested NC)...........: 5
EAPOL M1 messages (total)................: 47
EAPOL M2 messages (total)................: 18
EAPOL M3 messages (total)................: 4
EAPOL pairs (total)......................: 41
EAPOL pairs (best).......................: 1
EAPOL pairs written to combi hash file...: 1 (RC checked)
EAPOL M12E2 (challenge)..................: 1
PMKID (total)............................: 45
PMKID (best).............................: 1
PMKID written to combi hash file.........: 1
```

我们可以检查文件内容以确保我们捕获到了有效的哈希：

#### PMKID-哈希

```
cat pmkidhash_corp
```

```
7943ba84a475e3bf1fbb1b34fdf6d102*10da43bef746*80822381a9c8*434f52502d57494649
```

再次，我们将执行简单的字典攻击，尝试破解 WPA 握手包。为了尝试破解此哈希，我们将使用模式 22000，因为之前的模式 16800 已被弃用。这里的命令格式如下：

#### Hashcat - 破解 PMKID

```
hashcat -a 0 -m 22000 pmkidhash_corp /opt/useful/seclists/Passwords/Leaked-Databases/rockyou.txt.tar.gz
```

```
hashcat (v6.2.6) starting...

<SNIP>

7943ba84a475e3bf1fbb1b34fdf6d102:10da43bef746:80822381a9c8:CORP-WIFI:cleopatra

Session..........: hashcat
Status...........: Cracked
Hash.Name........: WPA-PBKDF2-PMKID+EAPOL
Hash.Target......: pmkidhash_corp
Time.Started.....: Wed Mar  9 11:27:21 2022 (1 sec)
Time.Estimated...: Wed Mar  9 11:27:22 2022 (0 secs)
Guess.Base.......: File (/usr/share/wordlists/rockyou.txt)
Guess.Queue......: 1/1 (100.00%)
Speed.#1.........:    12563 H/s (3.49ms) @ Accel:1024 Loops:32 Thr:1 Vec:8
Recovered........: 1/1 (100.00%) Digests
Progress.........: 18130/14344385 (0.13%)
Rejected.........: 11986/18130 (66.11%)
Restore.Point....: 0/14344385 (0.00%)
Restore.Sub.#1...: Salt:0 Amplifier:0-1 Iteration:0-1
Candidates.#1....: 123456789 -> celtic07
Started: Wed Mar  9 11:27:20 2022
Stopped: Wed Mar  9 11:27:23 2022
```

使用现已弃用的 hcxpcaptool 的过程类似。

#### Hcxpcaptool - 遗留

```
hcxpcaptool -h
```

```
hcxpcaptool 6.0.3-23-g1c078e4 (C) 2020 ZeroBeat
usage:
hcxpcaptool <options>
hcxpcaptool <options> [input.pcap] [input.pcap] ...
hcxpcaptool <options> *.cap
hcxpcaptool <options> *.*

options:
-o <file> : output hccapx file (hashcat -m 2500/2501)
-O <file> : output raw hccapx file (hashcat -m 2500/2501)
            this will disable all(!) 802.11 validity checks
            very slow!
-k <file> : output PMKID file (hashcat hashmode -m 16800 new format)
-K <file> : output raw PMKID file (hashcat hashmode -m 16801 new format)
            this will disable usage of ESSIDs completely
-z <file> : output PMKID file (hashcat hashmode -m 16800 old format and john)
-Z <file> : output raw PMKID file (hashcat hashmode -m 16801 old format and john)
            this will disable usage of ESSIDs completely
-j <file> : output john WPAPSK-PMK file (john wpapsk-opencl)
-J <file> : output raw john WPAPSK-PMK file (john wpapsk-opencl)
            this will disable all(!) 802.11 validity checks
            very slow!
-E <file> : output wordlist (autohex enabled) to use as input wordlist for cracker
-I <file> : output unsorted identity list
-U <file> : output unsorted username list
-M <file> : output unsorted IMSI number list
-P <file> : output possible WPA/WPA2 plainmasterkey list
-T <file> : output management traffic information list
            format = mac_sta:mac_ap:essid
-X <file> : output client probelist
            format: mac_sta:probed ESSID (autohex enabled)
-D <file> : output unsorted device information list
            format = mac_device:device information string
-g <file> : output GPS file
            format = GPX (accepted for example by Viking and GPSBabel)
-V        : verbose (but slow) status output
-h        : show this help
-v        : show version

<SNIP>
```

语法略有不同，使用 `-z` 标志而不是 `-o` 作为输出文件。使用 hcxpcaptool，我们可以提取 PMKID 哈希，然后像处理 hcxpcapngtool 生成的哈希一样，在 Hashcat 中运行。

#### 提取 PMKID - 使用 Hcxpcaptool

```
hcxpcaptool -z pmkidhash_corp2 cracking_pmkid.cap
```

```
reading from cracking_pmkid.cap
summary capture file:
---------------------
file name........................: cracking_pmkid.cap
file type........................: pcapng 1.0
file hardware information........: x86_64
capture device vendor information: 00c0ca
file os information..............: Linux 5.7.0-kali1-amd64
file application information.....: hcxdumptool 6.0.7-22-g2f82e84 (custom options)
network type.....................: DLT_IEEE802_11_RADIO (127)
endianness.......................: little endian
read errors......................: flawless
minimum time stamp...............: 17.07.2020 14:07:19 (GMT)
maximum time stamp...............: 17.07.2020 14:14:21 (GMT)
packets inside...................: 75
skipped damaged packets..........: 0
packets with GPS NMEA data.......: 0
packets with GPS data (JSON old).: 0
packets with FCS.................: 75
beacons (total)..................: 1
probe requests...................: 3
probe responses..................: 1
association responses............: 1
EAPOL packets (total)............: 69
EAPOL packets (WPA2).............: 69
PMKIDs (not zeroed - total)......: 1
PMKIDs (WPA2)....................: 45
PMKIDs from access points........: 1
best handshakes (total)..........: 1 (ap-less: 0)
best PMKIDs (total)..............: 1
summary output file(s):
-----------------------
1 PMKID(s) written to pmkidhash_corp
```

我们可以检查文件内容以确保我们捕获到了有效的哈希：

#### PMKID-哈希

```
cat pmkidhash_corp2
```

```
7943ba84a475e3bf1fbb1b34fdf6d102*10da43bef746*80822381a9c8*434f52502d57494649
```

从这里，我们可以使用模式 22000 在 Hashcat 中运行 `pmkidhash_corp2` 文件，如上所示。

### Cheetsheet

|                                           **Command**                                            |                                                 **Description**                                                  |
| :----------------------------------------------------------------------------------------------: | :--------------------------------------------------------------------------------------------------------------: |
|                                       `pip install hashid`                                       |                                            Install the `hashid` tool                                             |
|                             `hashid <hash>` OR `hashid <hashes.txt>`                             |                                      Identify a hash with the `hashid` tool                                      |
|                                    `hashcat --example-hashes`                                    |                              View a list of `Hashcat` hash modes and example hashes                              |
|                                   `hashcat -b -m <hash mode>`                                    |                            Perform a `Hashcat` benchmark test of a specific hash mode                            |
|                                           `hashcat -b`                                           |                                      Perform a benchmark of all hash modes                                       |
|                                           `hashcat -O`                                           |                         Optimization: Increase speed but limit potential password length                         |
|                                          `hashcat -w 3`                                          | Optimization: Use when Hashcat is the only thing running, use 1 if running hashcat on your desktop. Default is 2 |
|                       `hashcat -a 0 -m <hash type> <hash file> <wordlist>`                       |                                                Dictionary attack                                                 |
|                `hashcat -a 1 -m <hash type> <hash file> <wordlist1> <wordlist2>`                 |                                                Combination attack                                                |
|                `hashcat -a 3 -m 0 <hash file> -1 01 'ILFREIGHT?l?l?l?l?l20?1?d'`                 |                                                Sample Mask attack                                                |
|                    `hashcat -a 7 -m 0 <hash file> -1=01 '20?1?d' rockyou.txt`                    |                                               Sample Hybrid attack                                               |
|        `crunch <minimum length> <maximum length> <charset> -t <pattern> -o <output file>`        |                                          Make a wordlist with `Crunch`                                           |
|                                       `python3 cupp.py -i`                                       |                                           Use `CUPP` interactive mode                                            |
| `kwp -s 1 basechars/full.base keymaps/en-us.keymap routes/2-to-10-max-3-direction-changes.route` |                                              `Kwprocessor` example                                               |
|    `cewl -d <depth to spider> -m <minimum word length> -w <output wordlist> <url of website>`    |                                              Sample `CeWL` command                                               |
|                        `hashcat -a 0 -m 100 hash rockyou.txt -r rule.txt`                        |                                           Sample `Hashcat` rule syntax                                           |
|                            `./cap2hccapx.bin input.cap output.hccapx`                            |                                               `cap2hccapx` syntax                                                |
|                        `hcxpcaptool -z pmkidhash_corp cracking_pmkid.cap`                        |                                               `hcxpcaptool`syntax                                                |

`cat DC01.inlanefreight.local.ntds | cut -d ‘:’ -f4 | sort -rn | uniq -c`