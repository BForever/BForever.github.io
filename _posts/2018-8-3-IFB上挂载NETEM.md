# IFB上挂载NETEM
## 转发虚拟网卡的ingress
### 建立虚拟网卡的ingress转发到ifb0（每一个Pod）：
```
tc qdisc add dev calixxxxxxxxxxx ingress
tc filter add dev calixxxxxxxxxxx parent ffff: protocol ip prio 10 u32 match u32 0 0 flowid 1:1 action mirred egress redirect dev ifb0
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
### 为每一个类建立一个netem队列x+1:1（每一个Pod）
```
tc qdisc add dev ifb0 parent 1:1 handle x+1: netem delay 100ms
```
## 转发虚拟网卡的egress
### 为虚拟网卡的egress建立HTB队列1:（每一个Pod）
```
tc qdisc add dev calixxxxxxxxxxx root handle 1: htb default 1
```

### 为HTB建立子类1:1（每一个Pod）
```
tc class add dev calix parent 1: classid 1:1 htb rate 100mbps
```

### 为子类建立队列（每一个Pod）
```
tc qdisc add dev calix parent 1:1 pfifo limit 1600
```

### 转发到IFB1（每一个Pod）
```
tc filter add dev calix parent 1: proto ip prio 1 u32 match u32 0 0 flowid 1:1 action mirred egress redirect dev ifb1
```

### 建立ifb1的根队列htb（每一个Node）：
```
tc qdisc add dev ifb1 root handle 1: htb default 0
```

### 为x号Pod建立一个类1:x（每一个Pod）：
```
tc class add dev ifb1 parent 1: classid 1:x htb rate 100mbps
```

### 为Podx的IPx.x.x.x建立一个过滤器，使来自该Podx的流量进入子类1:x（每一个Pod）：
```
tc filter add dev ifb1 protocol ip parent 1:0 prio 1 u32 match ip src x.x.x.x flowid 1:x
```
### 为每一个类建立一个netem队列x+1:1（每一个Pod）
```
tc qdisc add dev ifb1 parent 1:1 handle x+1: netem delay 100ms
```