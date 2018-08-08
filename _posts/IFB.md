# IFB
IFB（中介功能块设备）是IMQ（中介队列设备）的继任者，IMQ从来没有被集成过，IFB拥有IMQ的优点，在SMP上更加清晰明了，并且代码量缩减了非常多，旧的中介设备功能仍然保留，但在你使用actions时你需要新的

## 目录
1. IFB使用场景
2. 典型应用
3. 跑一个小测试
4. IFB样例
5. IFB的依赖
6. IFB样例

## IFB使用场景
据我所知，如下场景是人们使用IMQ的理由：

* 队列/策略是针对单个网卡而不是应用到整个系统的，而IMQ允许多个设备共享队列/策略
* 允许入境流量被整流而不仅仅是被丢弃。我不知道有什么研究能说明丢包比整流更差，如果有我会十分感兴趣。
* 思科的[Comparing Traffic Policing and Traffic Shaping for Bandwidth Limiting](http://www.cisco.com/c/en/us/support/docs/quality-of-service-qos/qos-policing/19645-policevsshape.html#selectioncriteria)表明整流（排队）比策略（限流）更好，因为策略会丢弃多余的包，限制TCP的窗口大小并且减少基于TCP的流的输出速率
* 现实中几乎[所有的互联网](http://www.caida.org/data/realtime/passive/?monitor=equinix-chicago-dirA)流量都是基于TCP的
* 非常有意思的应用：如果你在提供P2P服务，你可能会想要优先处理本地流量而不是其他使用你的系统上/下载的流量。所以基于状态的QoS是一种解决方案，人们用在本地流量入口之前挂载IMQ的方式实现了它。我认为这是Linux上非常有特点的应用（不仅仅是IMQ）

不过我不会再使用netfilter hooks的方式实现这个，我也不会认为这值的我修改IFB获取三层网络的信息来实现它

替代的方案是使用一个跟踪连接的action。这个action会选择性的在入境数据包上查询/创建连接跟踪状态。数据包因此可以根据发生了什么来被重定向到IFB。如果我们发现它们是已知的状态我们就可以把它发送到不同于还没有状态的数据包的队列里。这取决于管理员制定的规则。

现在这个功能还不存在，我决定不在补丁上release它，如果有强烈要求我会增加这个特性

你现在可以用IFB做的是actions

假设你在来自192.168.200.200/32上的数据包上限流到100kbps

```
tc filter add dev eth0 parent 1: protocol ip prio 10 u32 \
 match ip src 192.168.200.200/32 flowid 1:2 \
 action police rate 100kbit burst 90k drop
```

如果你在eth0上运行tcpdump，你会看到所有来自于192.168.200.200/32的数据包是否被丢弃。扩展这条规则来只观察这些包：

```
tc filter add dev eth0 parent 1: protocol ip prio 10 u32 \
 match ip src 192.168.200.200/32 flowid 1:2 \
 action police rate 10kbit burst 90k drop \
 action mirred egress mirror dev ifb0
```

现在在ifb0上开启tcpdump就可以只观察这些包：

```
tcpdump -n -i ifb0 -x -e -t
```

这是一个非常好的debug和log的接口

如果你用重定向代替镜像，这些包会进入黑洞再也无法出来，这个重定向的行为会在新的补丁上被修改（但是镜像没有问题）

## 典型应用
你可以使用新的补丁来实现以前用IMQ实现的功能

```
export TC="/sbin/tc"
$TC qdisc add dev ifb0 root handle 1: prio 
$TC qdisc add dev ifb0 parent 1:1 handle 10: sfq
$TC qdisc add dev ifb0 parent 1:2 handle 20: tbf rate 20kbit buffer 1600 limit 3000
$TC qdisc add dev ifb0 parent 1:3 handle 30: sfq                                
$TC filter add dev ifb0 protocol ip pref 1 parent 1: handle 1 fw classid 1:1
$TC filter add dev ifb0 protocol ip pref 2 parent 1: handle 2 fw classid 1:2
ifconfig ifb0 up
$TC qdisc add dev eth0 ingress
# redirect all IP packets arriving in eth0 to ifb0 
# use mark 1 --> puts them onto class 1:1
$TC filter add dev eth0 parent ffff: protocol ip prio 10 u32 \
  match u32 0 0 flowid 1:1 \
  action ipt -j MARK --set-mark 1 \
  action mirred egress redirect dev ifb0
```

## 跑一个小测试
从另一台机器ping过来，这样你可以获得入境流量：

```
[root@jzny action-tests]# ping 10.22
PING 10.22 (10.0.0.22): 56 data bytes
64 bytes from 10.0.0.22: icmp_seq=0 ttl=64 time=2.8 ms
64 bytes from 10.0.0.22: icmp_seq=1 ttl=64 time=0.6 ms
64 bytes from 10.0.0.22: icmp_seq=2 ttl=64 time=0.6 ms
--- 10.22 ping statistics ---
3 packets transmitted, 3 packets received, 0% packet loss
round-trip min/avg/max = 0.6/1.3/2.8 ms
[root@jzny action-tests]#
```

现在看一下stats：

```
[root@jmandrake]:~# $TC -s filter show parent ffff: dev eth0
filter protocol ip pref 10 u32 
filter protocol ip pref 10 u32 fh 800: ht divisor 1 
filter protocol ip pref 10 u32 fh 800::800 order 2048 key ht 800 bkt 0 flowid 1:1 
  match 00000000/00000000 at 0
        action order 1: tablename: mangle  hook: NF_IP_PRE_ROUTING 
        target MARK set 0x1  
        index 1 ref 1 bind 1 installed 4195sec  used 27sec 
         Sent 252 bytes 3 pkts (dropped 0, overlimits 0) 
        action order 2: mirred (Egress Redirect to device ifb0) stolen
        index 1 ref 1 bind 1 installed 165 sec used 27 sec
         Sent 252 bytes 3 pkts (dropped 0, overlimits 0) 
[root@jmandrake]:~# $TC -s qdisc
qdisc sfq 30: dev ifb0 limit 128p quantum 1514b 
 Sent 0 bytes 0 pkts (dropped 0, overlimits 0) 
qdisc tbf 20: dev ifb0 rate 20Kbit burst 1575b lat 2147.5s 
 Sent 210 bytes 3 pkts (dropped 0, overlimits 0) 
qdisc sfq 10: dev ifb0 limit 128p quantum 1514b 
 Sent 294 bytes 3 pkts (dropped 0, overlimits 0) 
qdisc prio 1: dev ifb0 bands 3 priomap  1 2 2 2 1 2 0 0 1 1 1 1 1 1 1 1
 Sent 504 bytes 6 pkts (dropped 0, overlimits 0) 
qdisc ingress ffff: dev eth0 ---------------- 
 Sent 308 bytes 5 pkts (dropped 0, overlimits 0) 
[root@jmandrake]:~# ifconfig ifb0
ifb0    Link encap:Ethernet  HWaddr 00:00:00:00:00:00  
          inet6 addr: fe80::200:ff:fe00:0/64 Scope:Link
           UP BROADCAST RUNNING NOARP  MTU:1500  Metric:1
           RX packets:6 errors:0 dropped:3 overruns:0 frame:0
           TX packets:3 errors:0 dropped:0 overruns:0 carrier:0
           collisions:0 txqueuelen:32 
           RX bytes:504 (504.0 b)  TX bytes:252 (252.0 b)
```

伪设备（指IFB）仍然表现得像以前一样，你



