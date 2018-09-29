##网络安全HW01：基于VirtualBox的网络攻防基础环境搭建##

**一.实验目的**

熟悉linux系统的网络配置和终端命令行操作，加深对网络和网络拓扑结构的理解，掌握网卡IP地址与网关的配置。

**二.实验要求**

[节点]
靶机、网关、攻击者主机
[连通性]
- 靶机可以直接访问攻击者主机
- 攻击者主机无法直接访问靶机
- 网关可以直接访问攻击者主机和靶机
- 靶机的所有对外上下行流量必须经过网关
- 所有节点均可以访问互联网

**三.实验内容**

1.首先为三个节点配置网卡如下：
攻击机——kali-attack
![attack](https://i.imgur.com/aPEHClQ.png)

只有一块网卡，使用NAT网络接入

![IPaddress](https://i.imgur.com/G3ZZeOP.png)

已知attack的ip地址是10.0.2.15/24，默认网关的IP地址是10.0.2.1/24

靶机——kali-victim

![victim](https://i.imgur.com/63Ab6UI.png)

只有一块网卡，使用host-only接入

![IPaddress](https://i.imgur.com/LVDX7h3.png)

已知victim的ip地址是192.168.56.101/24,默认网关的IP地址是

网关

![gateway1](https://i.imgur.com/6ft31UQ.png)

第一块网卡，使用NAT网络接入，与攻击机在同一网络

![gateway2](https://i.imgur.com/dCz1g2g.png)

第二块网卡，使用host-only接入，与靶机在同一网络

![gateway](https://i.imgur.com/t7squ7t.png)

已知gateway网卡一的IP地址为10.0.2.5/24，网卡二的IP地址为192.168.56.103/24

此时进行发包测试，发现两台主机与网关可以相互ping通，靶机不能ping通攻击机，但攻击机可以ping通靶机。

![attackpingvictim](https://i.imgur.com/IB9I9TZ.png)

2.当攻击机ping靶机时，监听靶机，发现靶机收到的包来自地址192.168.56.1
![victimlisten](https://i.imgur.com/RusUwLI.png)

研究发现192.168.56.1是宿主机host-only网卡的IP地址

![hostonly](https://i.imgur.com/xeYatCs.png)

即攻击机发出去的包通过宿主机传递给了靶机

在靶机上配置iptables防火墙规则，增加一条——所有源地址为攻击地址10.0.2.15的input包全部采取动作drop
![victimdrop](https://i.imgur.com/1vRKJJV.png)

再次通过攻击机来ping靶机，发现已经是主机不可达了
![attckpingno](https://i.imgur.com/4iSiJ0l.png)

3.在网关上配置ipv4包转发规则，使靶机能ping通攻击机。
首先打开网关的ipv4转发功能，使网关知道帮靶机转发请求包
```
echo 1 > /proc/sys/net/ipv4/ip_forward
```
![openIpv4](https://i.imgur.com/qLG7eLp.png)

随后在网关上配置转发规则
iptable的可选参数如下：
```
-t<表>：指定要操纵的表；
-A：向规则链中添加条目；
-D：从规则链中删除条目；
-i：向规则链中插入条目；
-R：替换规则链中的条目；
-L：显示规则链中已有的条目；
-F：清楚规则链中已有的条目；
-Z：清空规则链中的数据包计算器和字节计数器；
-N：创建新的用户自定义规则链；
-P：定义规则链中的默认目标；
-h：显示帮助信息；
-p：指定要匹配的数据包协议类型；
-s：指定要匹配的数据包源ip地址；
-j<目标>：指定要跳转的目标；
-i<网络接口>：指定数据包进入本机的网络接口；
-o<网络接口>：指定数据包要离开本机所使用的网络接口。
```
iptables命令选项输入顺序：
```
iptables -t 表名 <-A/I/D/R> 规则链名 [规则号] <-i/o 网卡名> -p 协议名 <-s 源IP/源子网> --sport 源端口 <-d 目标IP/目标子网> --dport 目标端口 -j 动作
```
netfilter内部分为三个表，分别是 filter,nat,mangle，每个表又有不同的操作链。在filter过滤表中，也就是他的防火墙功能的这个表，定义了三个操作链 Chain。分别是INPUT,FORWARD,OUTPUT。也就是对包的入、转发、出进行定义的三个过滤链。对于这个filter表的操作和控制也是实现防火墙功能的一个重要手段（如上对靶机的防火墙配置）；在nat(Network Address Translation、网络地址翻译)表中，也就是我们用以实现地址转换和端口转发功能的这个表，定义了PREROUTING, POSTROUTING,OUTPUT三个链；而netfilter的mangle表则是一个自定义表，里面包括上面 的filter以及nat表中的各种chains,它可以让我们进行一些自定义的操作，同时这个mangle表中的chains在netfilter对包 的处理流程中处在一个比较优先的位置。

![](https://i.imgur.com/4pBkNE5.png)

然后根据nat地址翻译表对包的转发进行配置：
![](https://i.imgur.com/LK10THY.png)
```
iptables -t nat -A POSTROUTING　-s 192.168.56.0/24 -o eth0 -j  MASQUERADE
```
上述语句 修改表nat，添加转发规则：但凡源地址为192.168.56.0/24网段的包，全部经由网卡eth0转发出去，并且由网关自动伪装源地址。

**四.实验结果**

*靶机主机名：dragon 地址：192.168.56.101*
*攻击机主机名：localhost 地址：10.0.2.15*
*网关主机名：kali 网卡一地址：10.0.2.5
网卡二地址：192.168.56.103*

- 靶机可以直接访问攻击者主机<br>
![victimpingattack](https://i.imgur.com/r8iiVEk.png)

- 攻击者主机无法直接访问靶机<br>
![](https://i.imgur.com/erkvsVa.png)

- 网关可以直接访问攻击者主机和靶机<br>
![gateway](https://i.imgur.com/aZNPDS7.png)

- 靶机的所有对外上下行流量必须经过网关
- 所有节点均可以访问互联网
-攻击机

![attackpingbaidu](https://i.imgur.com/7N85fCG.png)

-靶机

![victimpingbaidu](https://i.imgur.com/wFdqIWU.png)

-网关

![gatewaypingbaidu](https://i.imgur.com/4hTAO3q.png)

**五.实验拓扑结构图**

![](https://i.imgur.com/s9K89qt.png)

**六.实验收获**

1.靶机的hostonly网卡由于借用了宿主机的网卡，如果宿主机可以上网，靶机应该可以借助宿主机直接上网，但在开始做实验时，用靶机ping外网时发现反馈信息不是host unreachable而是网络不可达，通过搜索资料之后，配置了靶机与宿主机一致的默认dns地址，并同时配置了route路由表具体过程如下
添加默认网关：
```
route add default gw 192.168.56.1
```
注：192.168.56.1是windows主机上host-only网卡上的ip地址。
配制dns服务器：
```
vim /etc/resolv.cnf
nameserver 192.168.1.1
```
注：192.168.1.1 和windows上的dns服务器要一致。

![dns](https://i.imgur.com/ug5HlMg.png)

2.虚拟机网卡未正常开启，因为内存不足无法正常用图形界面配置网卡的IP地址，可以用终端命令行手动配置开启网卡

打开配置文件
```
vim /etc/network/interfaces
```
i指令插入模式修改内容
```
# The loopback network interface
auto lo
iface lo inet loopback

# The primary network interface
allow-hotplug eth0
auto eth0                 //增加了该选项，因为使用networking restart时系统启动                            auto的网卡，如不加上则启动无反应
iface eth0 inet static    //配置了static 为静态ip
address 192.168.1.4       //设置ip地址
netmask 255.255.255.0     //设置掩码
gateway 192.168.1.1       //设置网关
```
esc退出修改模式，shift+；退出查看模式wq！保存并退出
随后用ip addr查看可发现网卡正常打开并显示IP地址