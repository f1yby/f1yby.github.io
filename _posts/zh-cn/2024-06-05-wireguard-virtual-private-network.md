---
title: "基于 Wireguard 的虚拟组网"
date: 2024-06-05
categories:
  - zh-cn
  - gossip

tags:
  - Wireguard
  - Network
---

由于国内 ipv4 地址的稀缺性，和朋友联机遇到了许多困难，比如年初火爆的幻兽帕鲁（PalWorld）或是我的世界。他们都依赖一个拥有 ipv4 地址的机器来作为服务器，直接去阿里云去租服务器作为游戏服务器很难满足需求。一方面游戏的瘾是一阵一阵的，但是为了做游戏服务器，设备一般需要较大的内存，而服务器租赁昂贵的价格和到期销毁磁盘，这还是让人有些接受不了。所以我尝试基于 Wireguard 来进行组网，以满足联机需要。相对于其他解决方案而言，Wireguard 跨平台能力良好，配置简单，游戏联机稳定。

## 配置一台具有 ipv4 地址的设备

这里我是购买的腾讯云的丐版服务器，99/年，阿里云也有类似的套餐，这个服务器的配置可以尽可能的低来节省预算，买尽量少的cpu、内存、硬盘、网络带宽等。

购买会提示你要烧录什么系统，一般选择 Ubuntu 即可。

> 曾经我也尝试过包括阿里云、腾讯云、华为云这三家服务商提供的定制发行版，一般都会说兼容 CentOS 7，但是使用体验相对来说比较差。
> 我自己笔记本电脑使用的发行版是 fedora。网上出现问题讨论比较多、解决方案比较多的系统是 Ubuntu。

### 进入终端

对于腾讯云 `控制台 > 服务器 > 登陆`

需要注意的是，之后的操作需要 `root` 权限，在登陆后执行 `echo $USER`，检查返回值是否为`root`，如果不是，执行`sudo su`并输入安装系统时配置的密码来获取，输入的密码不会显示在命令行里，这是正常现象，输入完密码后回车即可。

### 配置 ipv4 转发

使用文本编辑器编辑 `/etc/sysctl.conf`，可以执行`nano /etc/sysctl.conf`，将其中的`net.ipv4.conf.all.forwarding = 0`，修改为`net.ipv4.conf.all.forwarding = 1`,如果这一条不存在，那么添加进这个文件。如果你使用的是 nano，按 `Ctrl+X Y` 保存并退出。

修改好后执行 `sysctl -w` 同步配置到系统。


### 安装 Wireguard

安装步骤可以在[官网](https://www.wireguard.com/install/)找到。

使用 apt 管理软件的发行版（Ubuntu / Debian）运行：

```bash
apt install wireguard
```

### 配置 Wireguard

#### 配置权限

由于 Wireguard 的配置和执行会影响到服务器的网络状态，因此Wireguard需要配置文件禁止可能的非`root`访问。因此在之后的操作之前执行：

```bash
umask 077
```

> 关于 `umask 077` 做了什么，可以执行 `man umask` 查阅，同理也可以查阅 `man sysctl`

#### 生成密钥对

```bash
wg genkey | tee /etc/wireguard/priv | wg pubkey > /etc/wireguard/pub
```

可以执行 `cat` 指令查看内容，之后的配置中会用到

查看私钥：

```bash
cat /etc/wireguard/priv
```

查看公钥：

```bash
cat /etc/wireguard/pub
```


#### 确认网卡

执行 `ip link`。

以我自己的服务器为例，它的返回值是这样的

```
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN mode DEFAULT group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
2: eth0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc mq state UP mode DEFAULT group default qlen 1000
    link/ether 52:54:00:9b:e7:d9 brd ff:ff:ff:ff:ff:ff
    altname enp0s5
    altname ens5
    link/ether 52:b4:12:53:d2:54 brd ff:ff:ff:ff:ff:ff link-netns netns-120f5242-e872-52de-3cc2-9f9082f65985
```

其中， `lo` 是回环网卡，`eth0`是我使用到的实际用于连接互联网的网卡，记住这个网卡的名字


#### 生成配置文件

执行 `nano /etc/wireguard/wg0.conf`，将以下内容复制黏贴。

```
[Interface]
PrivateKey = # private key
Address = 192.168.40.1/24
PostUp = iptables -A FORWARD -i wg0 -j ACCEPT; iptables -t nat -A POSTROUTING -o eth0 -j MASQUERADE
PostDown = iptables -D FORWARD -i wg0 -j ACCEPT; iptables -t nat -D POSTROUTING -o eth0 -j MASQUERADE
ListenPort = 51820

# [Peer]
# PublicKey = 
# AllowedIPs = 192.168.40./32
```

之后需要做一些替换：
1. 将其中的 `PrivateKey = # private key` 替换成你生成的[私钥](#生成密钥对)，即`cat /etc/wireguard/priv`的结果。
2. 如果你了解 ip 地址和子网掩码，可以自由替换 `Address = 192.168.40.1/24` 的内容，而`192.168.40.1`将被作为云服务器在虚拟网络中的地址。
3. 将你自己的网卡名字替换 `PostUp = iptables -A FORWARD -i wg0 -j ACCEPT; iptables -t nat -A POSTROUTING -o eth0 -j MASQUERADE`和 `PostDown = iptables -D FORWARD -i wg0 -j ACCEPT; iptables -t nat -D POSTROUTING -o eth0 -j MASQUERADE` 中的 `eth0`


一个替换好的例子：

```
[Interface]
PrivateKey = IO0CPwRUEBFairCXZWGSdgxuop00MIfSPD7ngDxEbl4=
Address = 192.168.40.1/24
PostUp = iptables -A FORWARD -i wg0 -j ACCEPT; iptables -t nat -A POSTROUTING -o eth0 -j MASQUERADE
PostDown = iptables -D FORWARD -i wg0 -j ACCEPT; iptables -t nat -D POSTROUTING -o eth0 -j MASQUERADE
ListenPort = 51820

# [Peer]
# PublicKey = 
# AllowedIPs = 192.168.40./32
```



#### 启动 Wireguard

以下指令可以启动 Wireguard 并在每次服务器启动时自动启动。

```bash
systemctl enable --now wg-quick@wg0
```

## 配置其他设备

### 安装 Wireguard

对于 Windows 用户，可以在[官网](https://download.wireguard.com/windows-client/)中找到下载链接，这里推荐下载后缀为`.msi`的版本，后缀为`.exe`的版本在安装时需要科学上网。

### 配置 Wireguard Client 端

对于 Windows 用户，打开 Wireguard，按下`Ctrl + N`，填入名称，比如`wg0`。将以下内容**追加**到文本框后。

```
Address = 192.168.40.2/32

[Peer]
PublicKey = # vps public key
Endpoint = # ip address
AllowedIPs = 192.168.40.0/24
PersistentKeepalive = 25
```

此时文本框中内容应该如下

```
[Interface]
PrivateKey = # auto generated private key by wireguard
Address = 192.168.40.2/32

[Peer]
PublicKey = # vps public key
Endpoint = # ip address
AllowedIPs = 192.168.40.0/24
PersistentKeepalive = 25
```

之后需要做一些替换：
1. 将其中的 `Address = # 192.168.40.2/32` 替换成你希望的这台服务器在虚拟网络中的 ip 地址，对于在服务器使用 `192.168.40.1/24` 作为ip地址的虚拟网络而言，可行的 `Address`，为 `192.168.40.2/32` 到 `192.168.40.255/32`。
2. 将其中的 `PublicKey = # vps public key` 替换成[前一章生成的公钥](#生成密钥对)，对于在服务器使用 `192.168.40.1/24` 作为ip地址的虚拟网络而言，可行的 `Address`，为 `192.168.40.2/32` 到 `192.168.40.255/32`。
3. 将其中的 `Endpoint = # ip address` 替换`云服务器的ip地址:ListenPort`。`ListenPort`需要和[上一章配置文件](#生成配置文件)里的相匹配。
4. 将其中的 `AllowedIPs = 192.168.40.0/24` 修改为和[上一章配置文件](#生成配置文件)里的相匹配，*如果你在云服务器上使用的和我一样是`192.168.40.1/24`，就****无需修改***。




一个替换好的例子

```
[Interface]
PrivateKey = aGXScHJheIMUItT6ybzBqjiqc6gZgGN9xAPGRfp+XXU=
Address = 192.168.40.2/32

[Peer]
PublicKey = 6IiO9H9FonFolRZny4TTn+o7P6qYBH2I6yKtQCaSkj4=
Endpoint = 11.4.51.4:51820
AllowedIPs = 192.168.40.0/24
PersistentKeepalive = 25
```

### 记录 Client 端公钥

此时该界面上还会显示一个公钥，记录下来。

### 配置 Wireguard Server 端


执行 `nano /etc/wireguard/wg0.conf`，**追加**以下内容。

```
[Peer]
PublicKey = # client public key
AllowedIPs = 192.168.40.2/32
```

之后需要做一些替换：
1. 将其中的 `PublicKey = # client public key` 替换成[Client 上的公钥](#记录-client-端公钥)
2. 将其中的 `AllowedIPs = 192.168.40.2/32` 替换成[在 Client 上配置的 Address](#配置-wireguard-client-端)

一个替换好的例子：

```
[Interface]
PrivateKey = IO0CPwRUEBFairCXZWGSdgxuop00MIfSPD7ngDxEbl4=
Address = 192.168.40.1/24
PostUp = iptables -A FORWARD -i wg0 -j ACCEPT; iptables -t nat -A POSTROUTING -o eth0 -j MASQUERADE
PostDown = iptables -D FORWARD -i wg0 -j ACCEPT; iptables -t nat -D POSTROUTING -o eth0 -j MASQUERADE
ListenPort = 51820

[Peer]
PublicKey = 228n/EgOH2/moLu+pf2UzfNHVpM5UIxLD3FiX4nBtmQ=
AllowedIPs = 192.168.40.2/32
```

同步配置：

```bash
systemctl restart wg-quick@wg0
```

### 启动 Client 端

在客户端上点保存，并点击连接。

此时可以尝试测试连接状态： `Win+R`输入`cmd`，执行`ping 192.168.40.1`，测试与服务器的连接。也可以测试同在虚拟网络中的其他设备的连接情况。

当设备A与购买的云服务器能连接，而和虚拟网络中的其他设备B连接出现问题时，请尝试***关闭设备B的防火墙***。

## 结