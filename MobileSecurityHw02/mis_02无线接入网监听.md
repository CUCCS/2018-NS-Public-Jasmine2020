#移动互联网安全
###Chapter2 无线接入网监听
---

###一.实验过程
- 首先，确认网卡接入
<br>![](https://i.imgur.com/ShqnI6b.png)


- 查看所有活动（不包括被禁用）网卡
```
ifconfig
```
![](https://i.imgur.com/O6viYLH.png)
- 查看无线网卡参数配置命令
```
iwconfig
```
![](https://i.imgur.com/94wgGqq.png)
- 在确保无线网卡的工作模式是managed，且Access Poit值为Not-Associated的情况下，命令行下查看附近无线网络的SSID
```
iw dev wlan0 scan | grep SSID
```
![](https://i.imgur.com/2zUA22H.png)
- 使用 iw phy 查看到 phy_name
```
iw phy
```
![](https://i.imgur.com/bGbQvY2.png)
- 查看当前网卡支持的监听 channel
``` 
iw phy phy0 channels
```
![](https://i.imgur.com/W9Y6KDg.png)
![](https://i.imgur.com/kKqzrEE.png)
<br>该图输出为支持单频（2.4GHz）无线网卡的典型输出
- 开启无线网卡监听模式
```
# 解决部分无线网卡在Kali 2.0系统中设置监听模式失败，杀死可能会导致aircrack-ng套件工作异常的相关进程
airmon-ng check kill
```
![](https://i.imgur.com/mtPqKQv.png)

```
# 设置wlan0工作在监听模式
airmon-ng start wlan0
```
![](https://i.imgur.com/h61oEnz.png)
```
# 检查无线网卡是否已处于启用状态，以下命令的输出中应出现网卡名：wlan0mon
ifconfig

# 检查无线网卡是否已切换到监听模式，wlan0mon的Mode应显示为：Monitor
iwconfig 
```
![](https://i.imgur.com/ZZxgIXd.png)
<br>![](https://i.imgur.com/DauPA84.png)
<br>![](https://i.imgur.com/25rdyvx.png)
- 进行抓包
```
# 开始以channel hopping模式抓包
airodump-ng wlan0mon
```
![](https://i.imgur.com/SfvvJsA.png)
<br>将抓取的包保存到本地
```
# 选择一个"感兴趣"的目标AP进行定向（指定工作channel）监听并将结果保存到本地文件
airodump-ng wlan0mon --channel 13 -w saved --beacons --wps
```
- 查看保存的文件<br>
![](https://i.imgur.com/ZanpcHv.png)

- wireshark 查看pcap文件
![](https://i.imgur.com/asopqRO.png)

###二.实验背景
802.11标准将所有的数据包分为3种:


1. 数据: 数据数据包的作用是用来携带更高层次的数据(如IP数据包，ISO7层协议)，它负责在工作站之间传输数据
	- Data：wlan.fc.type_subtype == 0x20
	- Null：wlan.fc.type_subtype ==0x24
	- Qos Data：wlan.fc.type_subtype == 0x28
	- Qos Null：wlan.fc.type_subtype == 0x2c

![](https://i.imgur.com/yhvOvo5.png)

2. 管理: 管理数据包控制网络的管理功能，管理帧负责监督，主要用来加入或退出无线网络，以及处理接入点之间连接的转移事宜
	- 身份认证帧 (Authentication frame)：wlan.fc.type_subtype == 0x0b
	802.11认证开始时无线网卡发送认证帧给AP（开放系统(open system)认证，无线网卡只发送一个单独的认证帧，AP返回接受/拒绝结果；共享密钥(shared key)认证，在无线网卡发送初始认证请求后，AP返回一个包含挑战(chanllenge)的认证帧；无线网卡返回包含加密了的挑战的认证帧，AP解密确保挑战无误。该过程决定了无线网卡的认证状态。）
	- 关联请求帧 (Association request frame)：wlan.fc.type_subtype == 0x00
	- 关联响应帧(Association response frame)：wlan.fc.type_subtype==0x01
	- 信标帧 (Beacon frame)：wlan.fc.type_subtype == 0x08；在无线设备中，定时依次按指定间隔发送的有规律的无线信号(类似心跳包)，主要用于定位和同步使用
	- 解除认证帧 (Deauthentication frame)：wlan.fc.type_subtype == 0x0c；STA发送，希望终结认证
	- 解除关联帧 (Disassociation frame)：wlan.fc.type_subtype == 0x0a；STA发送，清除内存并从AP关联表中删除该无线网卡
	- 探测请求帧 (Probe request frame)：wlan.fc.type_subtype == 0x04；STA发送，希望得到可关联的AP的回应和相关信息（指定SSID或广播）
	- 探测响应帧 (Probe response frame)：wlan.fc.type_subtype == 0x05
	- 重新关联请求帧 (Reassociation request frame)：wlan.fc.type_subtype == 0x02；当无线网卡离开AP的信号范围内，STA会发送该请求给同ESS的另一个信号更强的AP
	- 重新关联响应帧 (Reassociation response frame)：wlan.fc.type_subtype == 0x03
	- action：wlan.fc.type_subtype == 0x0d

![](https://i.imgur.com/arh6Ihf.png)
![](https://i.imgur.com/EPFPzMS.png)

3. 控制: 控制数据包得名于术语"媒体接入控制(Media Access Control, MAC)"，是用来控制对共享媒体(即物理媒介，如光缆)的访问;控制帧通常与数据帧搭配使用，负责区域的清空、信道的取得以及载波监听的维护，并于收到数据时予以正面的应答，借此促进工作站间数据传输的可靠性
	- 请求发送(Request To Send，RTS)数据包 wlan.fc.type_subtype == 0x1b
	- 清除发送(Clear To Send，CTS)数据包 wlan.fc.type_subtype == 0x1c
	- ACK确认(RTS/CTS) wlan.fc.type_subtype == 0x1d 
	- PS-Poll: 当一部移动工作站从省电模式中苏醒，便会发送一个 PS-Poll 帧给基站，以取得任何暂存帧 

![](https://i.imgur.com/fxbg3eE.png)

![](https://i.imgur.com/HRjcpuc.png)

![](https://i.imgur.com/7f2Sotb.png)

**Wi-Fi认证过程**
1. AP发送Beacon广播管理帧：因为AP发送的这个Beacon管理帧数据包是广播地址，所以我们的PCMIA内置网卡、或者USB外界网卡会接收到这个数据包，然后在我们的"无线连接列表"中显示出来
2. 客户端向承载指定SSID的AP发送Probe Request(探测请求)帧：当我们点击"连接"的时候，无线网卡就会发送一个Prob数据帧，用来向AP请求连接
3. AP接入点对客户端的SSID连接请求进行应答：AP对客户端的连接作出了回应，并表示不接受任何形式的"帧有效负载加密(frame-payload-encryption)"
4. 客户端对目标AP请求进行身份认证(Authentication)
5. AP对客户端的身份认证(Authentication)请求作出回应：AP回应，表示接收身份认证
6. 客户端向AP发送连接(Association)请求：身份认证通过之后，所有的准备工作都做完了，客户端这个时候可以向WLAN AP发起正式的连接请求，请求接入WLAN
7. AP对连接(Association)请求进行回应：AP对客户端的连接请求(Association)予以了回应(包括SSID、性能、加密设置等)。至此，Wi-Fi的连接身份认证交互就全部结束了，之后就可以正常进行数据发送了
8. 客户端向AP请求断开连接(Disassociation)：当我们点击"断开连接"的时候，网卡会向AP发送一个断开连接的管理数据帧，请求进行断开连接


由此，我们可以发现，基于对数据帧格式的了解，黑客可以发起一些针对协议的攻击
```
1. Deanthentication攻击
2. Disassociation攻击
```
黑客可以利用这种方式加快对WEP/WPS-PSK保护的无线局域网的攻击，迫使客户端重新连接并且产生ARP流量(基于WEP的攻击)、或捕获重新进行WPA连接的四次握手，然后可以对密码进行离线字典或彩虹表破解攻击



###三.实验分析
- 查看统计当前信号覆盖范围内一共有多少独立的SSID？其中是否包括隐藏SSID？
---

IEEE 802.11 WLAN标准由3种不同的数据包类型组成：管理，控制和数据。当您的无线网络接口处于monitor时，它允许网络接口捕获所有数据，即使数据不是针对您的特定接口也是如此。通常在managed模式下，网卡接口将自动丢弃定向到其他接口的数据包以及用于通过无线电波进行通信的管理和控制数据包。
通常，所有接入点都在Beacon 帧中发送其SSID和其他信息。此Beacon Frames允许网络范围内的客户轻松发现它们。隐藏SSID是一种特殊配置，其中接入点不在Beacon帧中广播其SSID。因为借助这些设置。只有知道接入点SSID的以前的客户端才能连接到它。这种特殊配置以简单的方式从不了解真实SSID的新客户端隐藏了接入点网络。但这种配置并不能提供良好的安全性。 

从捕捉到的包中过滤出所有的Beacon frames

![](https://i.imgur.com/gvY5hdo.png)

独立的SSID可以通过查看Beacon帧得出；隐藏的SSID有两种可能，一种是广播的Beacon的SSID为空或者没广播Beacon但回复了Probe Response
通过wireshark的wireless管理可以直接查看所有SSID
![](https://i.imgur.com/sX5BKCJ.png)
SSID显示为broadcast表示AP没有用任何SSID发送了自己的beacon frame即为隐藏SSID

我们也可以直接用命令行来查看SSID名
```
tshark -r 20181002-01.cap -T fields -e wlan.sa -e wlan.ssid | sort -u | cat -v
  # -r 设置tshark分析的输入文件
  # -T 设置解码结果输出的格式
  # -e 添加一个字段列表显示，当-T被选中时，该命令可多次选用，这里仅仅显示地址和SSID
  # -e 输出源地址和SSID
  # cat -v 为了表示不可打印字符
  # sort -u 去除重复行
```
![](https://i.imgur.com/lgxmIRl.png)
比对两个结果可知 抓包范围内有五个独立的SSID和一个隐藏SSID（广播的beacon，无回复Probe Response的隐藏SSID）

- 哪些无线热点是加密/非加密的？加密方式是否可知？
---
热点是否是加密的可以查看帧的wlan.fixed.capabilities.privacy段
- 0是未加密状态
- 1是加密状态

![](https://i.imgur.com/yxUaMay.png)

![](https://i.imgur.com/q0aRbZT.png)

筛选后可知，加密的热点有3个；未加密的热点有1个，通过宿主机确认后发现，未加密的热点即为不需要输入密码可以直接连接的热点。

![](https://i.imgur.com/FykbiKy.png)

![](https://i.imgur.com/lqJAtxp.png)

加密的方式可以通过wlan.fixed.auth.alg来进行判断
- 0是OpenSystem，完全不认证也不加密，任何人都可以连到无线基地台使用网络
- 1是SharedKey
但筛选该字段后发现没有捕捉到可以判断热点加密方式的帧，因此无法判断各个热点是通过什么方式加密的
<br>![](https://i.imgur.com/PpRw13i.png)


- 如何分析出一个指定手机在抓包时间窗口内在手机端的无线网络列表可以看到哪些SSID？这台手机尝试连接了哪些SSID？最终加入了哪些SSID？
---
只要指定手机被覆盖在处于监听模式的无线网卡区域内，那么网卡监听到的SSID非空的，即不是隐藏SSID的Beacon，则就能在手机端的无线网络看到（因为隐藏SSID只允许以前接入过的设备连接，因此不能断定网卡捕捉到的隐藏SSID手机的无线不能接入）
<br>![](https://i.imgur.com/gvY5hdo.png)

在手机等STA尝试连接热点的时候会向信号覆盖范围内的AP广播发送Probe Request帧，因此可以通过判断手机向哪些AP发送了探测请求连接的Probe Request帧，可以知道手机尝试连接了哪些SSID
<br>![](https://i.imgur.com/UGizDsQ.png)

手机端等STA会记录已经连接过的AP的SSID，同时在和AP建立连接请求服务的时候也会发送关联请求Association Request帧，AP(SSID)会回复关联响应Association Response来确认建立连接，因此可以通过Association Response帧判断手机端最终加入了哪些SSID
![](https://i.imgur.com/COIIKDV.png)
如图可知，没有符合条件的捕捉帧，说明手机没有接入任何一个捕捉到的AP

- SSID包含在哪些类型的802.11帧？
---
理论上来说，SSID应该包含在Beacon Frame、Probe Request、Probe Response、Association RequestReassociation Request中，然而捕捉到的帧因为类型有限，只有Beacon Frame和Probe Reqest中可以观察到SSID
SSID只能出现在管理帧中。
![](https://i.imgur.com/5RllZA6.png)

![](https://i.imgur.com/tQi10OQ.png)


参考资料：
- View SSID AP names in Wireshark [https://www.algissalys.com/network-security/pcap-capture-view-ssid-ap-names-in-wireshark](https://www.algissalys.com/network-security/pcap-capture-view-ssid-ap-names-in-wireshark)
- Wi-Fi认证过程[http://www.cnblogs.com/littlehann/p/3700357.html](http://www.cnblogs.com/littlehann/p/3700357.html)
- 李沅城同学实验报告[https://github.com/CUCCS/2018-NS-Public-MrCuihi/blob/e8a2ee6064d77329531057556d233cbe0d01a25c/802.11%E7%BD%91%E7%BB%9C%E7%9B%91%E5%90%AC%E5%AE%9E%E9%AA%8C%E6%8A%A5%E5%91%8A.md#%E4%B8%80%E5%AE%9E%E9%AA%8C%E5%90%8D%E7%A7%B0](https://github.com/CUCCS/2018-NS-Public-MrCuihi/blob/e8a2ee6064d77329531057556d233cbe0d01a25c/802.11%E7%BD%91%E7%BB%9C%E7%9B%91%E5%90%AC%E5%AE%9E%E9%AA%8C%E6%8A%A5%E5%91%8A.md#%E4%B8%80%E5%AE%9E%E9%AA%8C%E5%90%8D%E7%A7%B0)
- 林淑琪同学实验报告[https://github.com/CUCCS/2018-NS-Public-jckling/blob/6f7677e8995a2633073ef26063541779245eacea/mis-0x02/%E7%AC%AC%E4%BA%8C%E7%AB%A0%E5%AE%9E%E9%AA%8C%E7%BB%83%E4%B9%A0%E9%A2%98.md](https://github.com/CUCCS/2018-NS-Public-jckling/blob/6f7677e8995a2633073ef26063541779245eacea/mis-0x02/%E7%AC%AC%E4%BA%8C%E7%AB%A0%E5%AE%9E%E9%AA%8C%E7%BB%83%E4%B9%A0%E9%A2%98.md)
