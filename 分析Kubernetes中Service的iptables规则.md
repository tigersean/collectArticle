# 分析Kubernetes中Service的iptables规则

2019-03-19

 [kubernetes](https://tizeen.github.io/category/#kubernetes)

------

Service 通过 label selector 选择对应的 Pod，每个 Service 对应有 cluster IP，kube-proxy 会为 Service 创建相关的 iptables 规则，以此来将访问的流量转发到后端的 Pod 处理。本文来分析这些 iptables 规则是如何工作的。

假设： 服务的地址是：10.233.58.157:8080

当集群内部使用 clusterIP 访问时：

数据包一开始通过 nat 表的 prerouting 链

```
Chain PREROUTING (policy ACCEPT)
target         prot opt source               destination         
KUBE-SERVICES  all  --  0.0.0.0/0            0.0.0.0/0            /* kubernetes service portals */
DOCKER         all  --  0.0.0.0/0            0.0.0.0/0            ADDRTYPE match dst-type LOCAL
```

匹配到了第一条规则，直接跳转到 KUBE-SERVICES 链，再看 KUBE-SERVICES 链中的内容：

```
Chain KUBE-SERVICES (2 references)
...
KUBE-MARK-MASQ             tcp  -- !10.233.64.0/18       10.233.58.157        /* default/goapp: cluster IP */ tcp dpt:8080
KUBE-SVC-LFUHBTR6HVHGMZSY  tcp  --  0.0.0.0/0            10.233.58.157        /* default/goapp: cluster IP */ tcp dpt:8080
...
```

如果数据包的来源地址不是 10.233.64.0/18 这个网段的，则对数据包进行标记。否则跳转到 KUBE-SVC-LFUHBTR6HVHGMZSY 链处理，再看 KUBE-SVC-LFUHBTR6HVHGMZSY 链的内容：

```
Chain KUBE-SVC-LFUHBTR6HVHGMZSY (2 references)
target                     prot opt source               destination         
KUBE-SEP-WRCNLEQDP3XM7G5R  all  --  0.0.0.0/0            0.0.0.0/0  
```

KUBE-SVC-LFUHBTR6HVHGMZSY 链中又跳转到 KUBE-SEP-WRCNLEQDP3XM7G5R，再看 KUBE-SEP-WRCNLEQDP3XM7G5R 中的内容：

```
Chain KUBE-SEP-WRCNLEQDP3XM7G5R (1 references)
target          prot opt source               destination         
KUBE-MARK-MASQ  all  --  10.233.68.41         0.0.0.0/0           
DNAT            tcp  --  0.0.0.0/0            0.0.0.0/0            tcp to:10.233.68.41:8080
```

这里对数据包进行了 DNAT ，将目的地址和端口变成了 Pod 的地址和端口

如果通过NodePort访问，则会在 KUBE-SERVICES 的时候跳转到 KUBE-NODEPORTS 的链进行处理，看看 KUBE-NODEPORTS 中的内容：

```
KUBE-MARK-MASQ             tcp  --  0.0.0.0/0            0.0.0.0/0            /* default/goapp: */ tcp dpt:30375
KUBE-SVC-LFUHBTR6HVHGMZSY  tcp  --  0.0.0.0/0            0.0.0.0/0            /* default/goapp: */ tcp dpt:30375
```

之后的流程和clusterIP是一样的。

需要注意的是，在 NodePort 方式下，Kubernetes 会在IP包离开宿主机发往目的 Pod 时，对这个 IP 包在 POSTROUTING 链做一次 SNAT 操作，将源地址换成替换成宿主机上 CNI 网桥的地址，或者宿主机本身的 IP 地址（如果 CNI 网桥不存在的话），如下所示：

```
Chain POSTROUTING (policy ACCEPT)
target              prot opt  source      destination         
KUBE-POSTROUTING    all  --  0.0.0.0/0     0.0.0.0/0       /* kubernetes postrouting rules */
Chain KUBE-POSTROUTING (1 references)  
target     prot  opt source        destination         
MASQUERADE  all  --  0.0.0.0/0     .0.0.0/0          /* kubernetes service traffic requiring SNAT */ mark match 0x4000/0x4000
```

这个 SNAT 操作只对有 “0x4000” 标志的数据包进行。

为什么进行 SNAT 的操作？

如下示意图：

```
           client
             \ ^
              \ \
               v \
   node 1 <--- node 2
    | ^   SNAT
    | |   --->
    v |
 endpoint
```

当外部的 client 通过 node2 的地址访问一个 Service 的时候，node2 上的负载均衡规则，就可能把这个 IP 包转发给一个在 node1 上的 Pod。这里没有问题。

当 node1 上的这个 Pod 处理完请求之后，它就会按照这个 IP 包的源地址发出回复。

如果没有做 SNAT 操作的话，这时候，被转发来的 IP 包的源地址就是 client 的 IP 地址。所以此时，Pod 就会直接将回复发给 client。但是对于 client 来说，我的请求是发给了 node2 的，收到的回复却来自 node1 ，这个 client 很可能就会报错。