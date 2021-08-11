# 1. 网卡绑定原理  
在正常情况下，网卡的工作模式为直接模式（Direct Model），该模式下的网卡只接收目地址是自己 Mac地址的帧。将别的数据帧都滤掉，以减轻驱动程序的负担。但是网卡也支持另外一种混杂模式，可以接收网络上所有的帧。  

Bonding运行在这个模式下，使用两块网卡虚拟成为一块网卡，这个聚合起来的设备看起来是一个单独的以太网接口设备，通俗点讲就是两块网卡具有相同的IP地址而并行链接聚合成一个逻辑链路工作。物理网卡把相应的数据帧传送给bond驱动程序处理。<font color="red">通过网卡绑定可以实现网络设备的冗余和负载均衡</font>。
# 2. 绑定的类别
## 2.0. Bond0 ：Round-robin（平衡轮循策略）
特点：传输数据包顺序是依次传输（即：第1个包走ens33，下一个包就走ens34….一直循环下去，直到最后一个传输完毕），此模式提供负载平衡。  

缺点：我们知道如果一个连接或者会话的数据包从不同的接口发出的话，中途再经过不同的链路，在客户端很有可能会出现数据包无序到达的问题，而无序到达的数据包需要重新要求被发送，这样网络的吞吐量就会下降。
## 2.1. Bond1 ：Active-backup（主-备份策略）
特点：只有一个设备处于活动状态，当一个宕掉另一个马上由备份转换为主设备。Mac地址是外部可见得，从外面看来，bond的Mac地址是唯一的，以避免交换机发生混乱。此模式只提供了容错能力，提供高网络连接的可用性。  

缺点：资源利用率较低，只有一个接口处于工作状态，在有 N 个网络接口的情况下，资源利用率为1/N。


## 2.2. Bond2 ：Balance-xor（平衡策略）
特点：基于指定的传输策略传输数据包。通过<font color="red">xmit_hash_policy配置项</font>来确定具体的传输策略。默认为异或哈希负载分担（XOR Hash）。此模式提供负载平衡和容错能力。
异或哈希负载分担规则：选择网卡的序号 = (源MAC地址 XOR 目标MAC地址) % Slave网卡数量。
该策略算法概述：在接收和发送数据时，Bond通过异或哈希算法对需要调度的Slave进行选择，其中使用哈希的目的是为了数据的分布尽量分散到各个Slave上，从而实现负载均衡；使用XOR异或的目的是为了避免哈希冲突（例如我们使用平方取中法等方法目的是为了避免冲突，优化哈希查找效率），从而实现负载均衡的优化。

```bash
# 异或哈希负载均衡工作原理
举个例子：
此时BOND2的MAC为AA，BOND2中有四个Slave，编号为Slave_0,1,2,3
此时BOND2从网络上收到MAC地址为BB的数据帧，根据异或哈希计算出哈希：( AA XOR BB ) % 4 = 1
AA	| 1010 1010	
BB	| 1011 1011	
XOR	| 0001 0001 -> 16+1 -> 17
17 % 4 = 1
那么该数据就会交由Slave_1处理，而且之后的同样来自于该MAC地址的数据依然交由Slave_1处理。其他MAC地址的数据将大概率分配到Slave_0,2,3，小概率分配到Slave_1。
对于不同来源的MAC的数据，异或哈希能很平均地分担到各个网卡。
```

Q1：为什么是XOR异或，如果用与或呢？  

A1：因为在数学方面来说，与或的真值分布不均匀，与(同1出1  1的分布为25%)，或(同0出  0的分布为25%)，而异或能保证50%:50%的分布，可以保证哈希过程中更好的随机性，进而避免了哈希冲突发生的概率。

Q2：既然通过异或哈希实现了负载均衡，那么Bond2怎样实现了冗余？  

A2：在Bond2中，异或哈希算法是对Slave数量取余，其实是对当前活动的Slave取余，如果对于正常运行状态的4个Slave的Bond2，其中Slave_1宕掉了，此时Slave数量会变为3，同时Slave的顺位也发生变化：
[ 0 1 2 3 ] 正常状态-> [ 0 2 3 ] Slave1宕 -> [ 0 1 2 ] 新正常状态
此时原本分配给Slave_1的数据由新的Slave_1接收。  

Bond2中通过xmit_hash_policy指定Slave调度策略，默认为上边讲到的layer2，即仅用MAC地址作为区分数据分配的依据。可以增加对IP层，应用层的过滤，保证来自不同IP的数据，甚至来自不同端口的数据都能分配到不同的Slave当中去，进而实现更好的负载均衡，当然也要更具服务器的具体情况来设置最合适的负载均衡策略。至于该配置项在哪里进行配置，在后边的绑定实现中会讲。
```bash
xmit_hash_policy配置项：

xmit_hash_policy=layer2
(source MAC XOR destination MAC) % slave数量
解释：使用硬件MAC地址的XOR来生成哈希

xmit_hash_policy=layer2+3
(((source IP XOR destination IP) AND 0xffff) XOR ( source MAC XOR destination MAC )) % slave数量
解释：使用硬件 MAC 地址及 IP 地址生成哈希

xmit_hash_policy=layer3+4
((source port XOR dest port) XOR((source IP XOR dest IP) AND 0xffff)% slave数量
解释：使用硬件 端口号 及 IP 地址生成哈希
```
## 2.3. Bond3 ：Broadcast（广播策略）
特点：所有包从所有网络接口发出，此模式适用于金融行业，因为他们需要高可靠性的网络，不允许出现任何问题。需要和交换机的聚合强制不协商方式配合。
缺点：只有冗余机制，但过于浪费网络中的资源及本机的网络设备资源。  

## 2.4. Bond4 ：IEEE 802.3ad（动态链接聚合）
特点：启动时会根据IEEE 802.3ad规范创建一个聚合组。使用动态链接聚合策略，所有Slave网卡共享同样的速率和双工设定。
​		必要条件：
​			1．支持使用ethtool工具获取每个slave网卡的速率和双工设定；
​			2．需要交换机支持IEEE 802.3ad 动态链路聚合（Dynamic link aggregation）模式
​		<font color="red">IEEE 802.3ad规范：</font>
​		IEEE 802.3ad主要是对链路聚合控制协议进行了一定的标准和规范。

​		链路聚合，又称端口聚合、端口捆绑技术。功能是将交换机的多个低带宽端口捆绑成一条高带宽链路，同时通过几个端口进行链路负载均衡，避免链路出现拥塞现象。
## 2.5. Bond5 ：Balance-tlb（适配器传输负载均衡）
特点：是根据每个slave的负载情况选择slave进行发送，接收时使用当前轮到的slave。该模式要求slave接口的设备驱动有ethtool支持。​不需要交换机支持。在每个slave上根据当前的负载（根据速度计算）分配外出流量。
​必要条件：ethtool支持获取每个slave的速率。
缺点：不支持发送负载均衡
## 2.6. Bond6 ：Balance-alb（适配器适应性负载均衡）
特点：该模式包含了Bond5模式，同时加上接收负载均衡（receive load balance），同样不需要交换机的支持。接收负载均衡是通过ARP协商实现的。bonding驱动截获本机发送的ARP应答，并把源硬件地址改写为bond中某个slave的唯一硬件地址，从而使得不同的对端使用不同的硬件地址进行通信。
缺点：ARP协商中存在一些问题（未深入了解，仅列出）
# 3. 绑定的实现
## 3.1. 修改多个物理网卡配置文件
```bash
# 以bond1举例
# vim /etc/sysconfig/network-script/ifcfg-ens33
HWADDR=00:0C:29:E0:B9:BC
MACADDR=preserve
TYPE=Ethernet
NAME="bond1 slave 1"
UUID=ef2b8ef8-531f-4f09-902d-2442f096cb12
DEVICE=ens33
ONBOOT=yes
MASTER=bond1
SLAVE=yes
# vim /etc/sysconfig/network-script/ifcfg-ens34
HWADDR=00:0C:29:E0:B9:C6
MACADDR=preserve
TYPE=Ethernet
NAME="bond1 slave 2"
UUID=9a471884-2e42-4dd1-a876-1ad7aeedcc2f
DEVICE=ens34
ONBOOT=yes
MASTER=bond1
SLAVE=yes
```
## 3.2. 新建bond网卡配置文件，配置bond参数
```bash
# 网络部分
BOOTPROTO=none
IPADDR=192.168.157.203
PREFIX=24
GATEWAY=192.168.157.2
DNS1=8.8.8.8
# IPV6设置
IPV6INIT=yes
IPV6_AUTOCONF=yes
IPV6_DEFROUTE=yes
IPV6_FAILURE_FATAL=no
IPV6_PRIVACY=no
IPV6_ADDR_GEN_MODE=stable-privacy
DEFROUTE=yes
IPV4_FAILURE_FATAL=no
# bond参数部分
BONDING_OPTS="downdelay=0 miimon=1 mode=active-backup updelay=0"
TYPE=Bond
BONDING_MASTER=yes
PROXY_METHOD=none
BROWSER_ONLY=no
NAME="Bond connection 1"
UUID=bacf3665-d467-4059-bd33-98d8919a3e41
DEVICE=bond1
ONBOOT=yes
FAIL_OVER_MAC=2
```
BONDING_OPTS关键参数：
```bash
mode=		： 指定绑定模式0~6，或者对应模式的简称

miimon= 	： 指定ARP链路监控频率，单位是毫秒(ms)

downdelay	： 用于在发现链路故障后，等待一段时间然后禁止一个slave

updelay		： 当发现一个链路恢复时，在激活该链路之前的等待时间

xmit_hash_policy ： 传输策略

```
## 3.3. 启动bond模块
```bash
systemctl network restart
```

# 4. 绑定实验
## 实验准备
```bash
# VMware NAT模式使用静态地址分配
# 地址分配
192.168.157.103 bond0
192.168.157.203 bond1
192.168.157.113 bond2
192.168.157.123 bond3
192.168.157.133 bond5
192.168.157.53  bond6
192.168.157.8   pinger
192.168.157.6-7 对照组

# ifstat
# TX RATE是发送速度，RX RATE是接收速度

Every 1.0s: ifstat                                      Tue Aug  3 02:22:58 2021
#kernel
Interface        RX Pkts/Rate    TX Pkts/Rate    RX Data/Rate    TX Data/Rate
                 RX Errs/Drop    TX Errs/Drop    RX Over/Rate    TX Coll/Rate
lo                     0 0             0 0             0 0             0 0
                       0 0             0 0             0 0             0 0
ens33                  1 0             0 0            60 0             0 0
                       0 0             0 0             0 0             0 0
ens34                  0 0             1 0             0 0           130 0
                       0 0             0 0             0 0             0 0
bond2                  1 0             1 0            60 0           130 0
                       0 0             0 0             0 0             0 0
                       
以上的动态刷新实际上是通过 watch -n 1 "ifstat" 命令实现1秒刷新一次命令
```

## 4.0 Bond0实验
### 4.0.1 实验步骤
1. 在bond0虚拟机后台执行 ping pinger 命令
2. 在bond0虚拟机前台执行 watch -n 1 "ifstat" 命令 
### 4.0.2 实验结果
1. ifstat 显示两个网卡在交替接收来自ssh链接的网络数据
## 4.1 Bond1实验
### 4.1.1 实验步骤
1. 在bond1虚拟机前台执行 watch -n 1 "ifstat" 命令 
2. 在bond1虚拟机其他shell中执行 ifdown ens33 命令，观察ifstat后执行ifup ens33
3. 在bond1虚拟机其他shell中执行 ifdown ens34 命令，观察ifstat后执行ifup ens34
### 4.1.2 实验结果
1. bond1启动后，从ifstat可以看出仅有一个网卡在工作状态，该网卡即为主网卡。
2. 主网卡从挂掉到恢复运行后，网络出现短暂的未响应状态，若干秒后会恢复。
3. 从网卡从挂掉到恢复运行后，网络出现短暂的未响应状态，且其身份切换为主网卡。
4. 新加入bond的网卡将自动成为主网卡。
#### 2.4.2 Bond2 实验
## 4.2 Bond2实验
### 4.2.1 实验步骤
1. 在bond2虚拟机前台执行 watch -n 1 "ifstat" 命令 
2. 在bond2虚拟机后台执行 ping pinger 命令
3. 增加负载：在宿主机Win10上使用xftp与bond2进行数据传输
### 4.2.2 实验结果
1. 从ifstat看到在ping过程中，ens33只接收数据不发送数据，ens34只发送数据不接受数据。
2. 增加负载后，可以看到ens33兼顾接收和发送数据，ens34只发送数据不接受数据。
3. 推论：如果继续增加负载，那么可以看到ens33兼顾接收和发送数据，ens34也兼顾接收和发送数据。
		（该部分实验现象无法合理解释，需要进一步深入了解）
## 4.3 Bond3实验
### 4.3.1 实验步骤
1. 在bond3虚拟机后前台执行 ping pinger 命令
### 4.3.2 实验结果
1. 在收到的ping回复中，每收到一份正常的报文，就会紧接着收到一份DUP警告的报文。（DUP是DUPLICATE的一个缩写，也就是ping包的时候收到多个重复值回应）这是因为在bond3 ping其他主机时，由于双网卡广播icmp报文，所以会收到两份icmp应答报文，收到ping的DUP警告。
## 4.4 Bond4实验
### 4.4.1 实验步骤
1. 需要交换机支持，故该实验暂时无法进行。
### 4.4.2 实验结果
1. 无
## 4.5 Bond5实验
### 4.5.1 实验步骤
1. 在bond5虚拟机后台执行 ping pinger 命令
2. 在bond5虚拟机前台执行 watch -n 1 "ifstat" 命令 
### 4.5.2 实验结果
1. 从ifstat可以看出bond5网络在发送数据时，会将流量基本平均地分配到两个网卡上。
## 4.6 Bond6实验
### 4.6.1 实验步骤
1. 在bond6虚拟机后台执行 ping pinger 命令
2. 在bond6虚拟机前台执行 watch -n 1 "ifstat" 命令 
3. 在pinger虚拟机前台执行 ping bond6 命令
4. wireshark抓包查看该过程
### 4.6.2 实验结果
1. 从ifstat可以看出bond5网络在发送数据以及接收数据时，会将流量基本平均地分配到两个网卡上。
2. 通过抓包可以看到bond6在数据传输时进行的ARP协商过程

```bash
# ARP协商的大致过程
IP：
pinger 192.168.157.11
bond6 192.168.157.53
MAC：
pinger       x:08
bond6 ens33  x:03 
bond6 ens37  x:f9
--------------------------------------------------------------------------
time line ->  ->  ->  ->  ->  ->  ->  ->  ->  ->  ->  ->  ->  ->  ->  ->  
--------------------------------------------------------------------------
stage     |   ping request    |     reply     |       bond internal
IP        |    11 -> 53       |   53 -> 11    |       /           /
MAC       |    08 -> 03       |   f9 -> 08    |    f9 -> f9    03 -> 03
--------------------------------------------------------------------------
```
# 5. 配置文件
bond文件夹内包含bond0-6（除4外）所有的配置文件
