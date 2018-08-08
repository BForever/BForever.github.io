# IFB上挂载NETEM
### 建立虚拟网卡的ingress转发到ifb0（每一个Pod）：
```
tc qdisc add dev calicoxxxxxxxxxxx ingress
tc filter add dev calicoxxxxxxxxxxx parent ffff: protocol ip prio 10 u32 \
> match u32 0 0 flowid 1:1 \
> action mirred egress redirect dev ifb0
```

### 建立ifb0的根队列htb（每一个Node）：
```
tc qdisc add dev ifb0 root handle 1: htb default 0
```

### 为x号Pod建立一个类1:x（每一个Pod）：
```
tc class add dev ifb0 parent 1: classid 1:x htb rate 100mbps
```

### 为Podx的IPx.x.x.x建立一个过滤器，使来自该Podx的流量进入子类1:x（每一个Pod）：
```
tc filter add dev ifb0 protocol ip parent 1:0 prio 1 u32 match ip src x.x.x.x flowid 1:x
```


```
tc qdisc add dev tunl0 ingress

tc filter add dev tunl0 parent ffff: protocol ip prio 10 u32 match u32 0 0 flowid 1:1 action mirred egress redirect dev ifb1

tc qdisc add dev ifb1 root handle 1: htb default 0

tc class add dev ifb1 parent 1: classid 1:1 htb rate 100mbps

tc filter add dev ifb1 protocol ip parent 1:0 prio 1 u32 match ip src 192.168.102.232 flowid 1:1
tc qdisc add dev ifb1 parent 1:1 netem delay 100ms
```