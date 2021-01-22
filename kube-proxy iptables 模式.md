# 手撕kube-proxy —— iptable模式实现原理实例分析

全栈工程师，致力于区块链与云计算

20 人赞同了该文章



作为一个转行人员，计算机网络一直是我最薄弱的环节。里面错综复杂的细节和让我抓狂。学习k8s的过程中，我曾在AWS上使用hard way模式尝试搭过几次k8s的cluster，最后都在pod network那个环节失败。最近辞职准备跳槽，有将近1个月的空闲时间，总算有空好好钻研这部分知识的空缺。我准备利用这段时间从最基本最基础的计算机网络知识出发，把k8s的网络拔个底朝天，彻底解决这个弱点。目前并没有特别的文章顺序，学到哪写到哪，每一篇文章都算是一段学习的总结。大家如果感兴趣可以和我一起学习，相互交流，相互指正。

最后，我对linux的network理解还较浅，抱着学习的态度来写的这篇文章，如果大家发现文中有错误，还请在评论区指出，我马上改正，坚决不误人子弟。

------

## 本文主要内容

- 介绍kube-proxy以及iptable；
- 搭建minikube cluster，部署实验环境；
- 在实验环境中彻底分析kube-proxy在iptable模式下的Service实现细节

------

## kube-proxy简介

我们知道，Kubernetes中Pod的生命是短暂了，它随时可能被终止。即使使用了Deployment或者ReplicaSet保证Pod挂掉之后还会重启，但也没法保证重启后Pod的IP不变。从服务的高可用性与连续性的角度出发，我们不可能把Pod的IP直接暴露成service端口。因此我们需要一个更加可靠的“前端”去代理Pod，这就是k8s中的Service。引用官方文档的定义就是：

> Service 定义了这样一种抽象：逻辑上的一组 Pod，一种可以访问它们的策略 —— 通常称为微服务。 这一组 Pod 能够被 Service 访问到。

也可以说，Service作为“前端”提供稳定的服务端口，Pod作为“后端”提供服务实现。Service会监控自己组内的Pod的运行状态，剔除终止的Pod，添加新增的Pod。配合k8s的服务发现机制，我们再也不用担心IP改变，Pod终止等问题了。kube-proxy则是实现Service的关键组件，到目前为止共有3种实现模式：

- Userspace模式：应用发往Service的请求会通过iptable规则转发给kube-proxy，kube-proxy再转发到Service所代理的后端Pod。可以看到这个模式下，发往Service的请求会先进入内核空间的iptable，再回到用户空间由kube-proxy代理转发。内核空间和用户空间来回地切换成为了该模式的主要性能问题。但由于发往后端Pod的请求是由kube-proxy代理转发的，请求失败时，是可以让kube-proxy重试的。
- Iptable模式：iptable模式是目前的默认模式，可以看成是userspace模式的升级版，它将请求的代理转发规则全部写入iptable中，砍掉了kube-proxy转发的部分。整个过程全部发生在内核空间，提高了转发性能。但是，iptable的规则是基于链表实现的，规则数量随着Service数量的增加线性增加，查找时间复杂度为O(n)。当Service数量到达一定量级时，CPU消耗和延迟增加显著。

> 针对iptable的O(n)级别长链问题，Calico插件优化了iptable的规则，使用ipset保存IP，大大提高了IP的查找效率（ipset内查找的复杂度是O(1)），减少了iptable的规则数量。

- ipvs模式：ipvs模式是基于章文嵩博士开发的LVS实现的，ipvs和iptables都是基于内核的netfilter框架实现的，不同的是iptable主攻防火墙，ipvs主攻内核态4层负载均衡。可以说先天上，ipvs就比iptable更适合做Service的实现。关于iptable与ipvs的性能对比，可以参考

[Comparing kube-proxy modes: iptables or IPVS? | Tigerawww.tigera.io![图标](https://pic1.zhimg.com/v2-39effd545676f083735221802cd7d6a4_180x120.jpg)](https://link.zhihu.com/?target=https%3A//www.tigera.io/blog/comparing-kube-proxy-modes-iptables-or-ipvs/)



## iptable简介

再介绍iptable之前，我强烈建议没有相关基础的朋友去看朱双印大佬写的iptable系列博客，这是我学习iptable过程中看到的最好的入门文章：

[iptables详解：图文并茂理解iptableswww.zsythink.net![图标](https://pic4.zhimg.com/v2-73e11e98881f07db72ae98728a30a137_ipico.jpg)](https://link.zhihu.com/?target=https%3A//www.zsythink.net/archives/1199)

在此，本文仅对iptable的基本概念做一些简单的介绍。

首先来一段网上的定义：

> netfilter/iptables（简称为iptables）组成Linux平台下的包过滤防火墙，与大多数的Linux软件一样，这个包过滤防火墙是免费的，它可以代替昂贵的商业防火墙解决方案，完成封包过滤、封包重定向和网络地址转换（NAT）等功能。

从定义上来看它是个防火墙，基于内核框架netfilter实现，运行在内核态，具备过滤，重定向，NAT等功能。每个数据包都要通过iptable的过滤，经过iptable中定义的重重关卡，这些关卡被称为链，不同流向的数据包会经过不同的链：

- 到本机某进程的报文：PREROUTING --> INPUT
- 由本机转发的报文：PREROUTING --> FORWARD --> POSTROUTING
- 由本机的某进程发出报文：OUTPUT --> POSTROUTING

每个链上会有一些规则去过滤数据包进行操作，这些规则在大体上又可以分为4类，分别存在4张table中：

- filter表：负责过滤功能，防火墙；内核模块：iptables_filter
- nat表：network address translation，网络地址转换功能；内核模块：iptable_nat
- mangle表：拆解报文，做出修改，并重新封装 的功能；内核模块：iptable_mangle
- raw表：关闭nat表上启用的连接追踪机制；内核模块：iptable_raw

每条规则会通过一些条件去匹配数据包，比如：源IP，目的IP，端口，协议种类，出入网卡名称等等，匹配到数据包后还会指定target，也就是对这个数据包进行操作。操作包括：

- ACCEPT：允许数据包通过。
- DROP：直接丢弃数据包，不给任何回应信息，这时候客户端会感觉自己的请求泥牛入海了，过了超时时间才会有反应。
- REJECT：拒绝数据包通过，必要时会给数据发送端一个响应的信息，客户端刚请求就会收到拒绝的信息。
- SNAT：源地址转换，解决内网用户用同一个公网地址上网的问题。
- MASQUERADE：是SNAT的一种特殊形式，适用于动态的、临时会变的ip上。
- DNAT：目标地址转换。
- REDIRECT：在本机做端口映射。
- LOG：在内核日志中记录日志信息，然后将数据包传递给下一条规则，也就是说除了记录以外不对数据包做任何其他操作，仍然让下一条规则去匹配。

其中ACCEPT，DROP，REJECT，SNAT，MASQUERADE，REDIRECT，DNAT为终结操作，也就是匹配到它们之后不再继续匹配当前表中剩下的规则，如果数据包还存在的话就继续匹配后面剩的其他表中的规则。对于DROP和REJECT来讲，数据包经过它们后将不复存在，所以也就不再有后续的匹配了。其他的为非终结操作，操作完成后还会让数据包继续匹配当前表后面的规则。

> 举个例子：数据包A由是本机的某进程发出报文，经过OUTPUT --> POSTROUTING，在OUTPUT中首先依次进入raw，mangle，nat，filter这4个表，每个表都有自己的OUTPUT链，每条链都有若干规则，下面分析几种情况：
> \1. 如果A在nat表的OUTPUT的第一条规则中被REJECT或者DROP，那么A的处理到此为止；
> \2. 如果A在nat表的OUTPUT的第一条规则中被ACCEPT，那么它将进入filter表的OUTPUT链继续匹配规则；
> \3. 如果A在nat表的OUTPUT的第一条规则中被LOG，那么它将继续匹配nat表的第二条规则直到遇到终结操作。

可以总结数据包的流向如图：



![img](/home/ejungon/Documents/收集的文章/kube-proxy iptables 模式.assets/v2-68f3aa78a480b26c29f45f8ac76598c6_720w.jpg)iptables 数据包流向图 （出自：https://www.zsythink.net/archives/1199）

iptable的操作（需要root权限）命令可以总结如下：

![img](https://pic2.zhimg.com/80/v2-c931bbdff203733fdc5f0ba00a351c45_720w.jpg)iptables 基本操作

通常使用来查看本机table名为TABLE_NAME的iptable规则

```text
iptables -t TABLE_NAME -vnL
```

例如：

```text
$ iptables -t raw -nvL       
Chain PREROUTING (policy ACCEPT 9156K packets, 2610M bytes)
 pkts bytes target     prot opt in     out     source               destination         

Chain OUTPUT (policy ACCEPT 8912K packets, 2321M bytes)
 pkts bytes target     prot opt in     out     source               destination
```

每条链中都可以添加多条规则，规则是按照顺序从前到后执行的。我们来看下规则的表头定义。

- **pkts**：处理过的匹配的报文数量
- **bytes**：累计处理的报文大小（字节数）
- **target**：如果报文与规则匹配，指定目标就会被执行。
- **prot**：协议，例如 `tdp`、`udp`、`icmp` 和 `all`。
- **opt**：很少使用，这一列用于显示 IP 选项。
- **in**：入站网卡。
- **out**：出站网卡。
- **source**：流量的源 IP 地址或子网，后者是 `anywhere`。
- **destination**：流量的目的地 IP 地址或子网，或者是 `anywhere`。

还有一列没有表头，显示在最后，表示规则的选项，作为规则的扩展匹配条件，用来补充前面的几列中的配置。`prot`、`opt`、`in`、`out`、`source` 和 `destination` 和显示在 `destination` 后面的没有表头的一列扩展条件共同组成匹配规则。当流量匹配这些规则后就会执行 `target`。

关于iptable本文只介绍到这了，没理解的话请一定看看本章头推荐的大佬博客！

------

## 环境准备

为了观察k8s创建service后对iptable进行了哪些修改，我们先启动一个minikube cluster，使用下面这个yaml创建一个简单的部署，其实就是1个Service后面代理着3个nginx pod，还有一个web-server用来访问这个service。本例的service type是ClusterIP，后面再继续分析NodePort类型。

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
  labels:
    app: nginx
spec:
  replicas: 3
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx
        ports:
        - containerPort: 80
---
apiVersion: v1
kind: Service
metadata:
  name: nginx-service
spec:
  selector:
    app: nginx
  ports:
    - protocol: TCP
      port: 80
      targetPort: 80
---
apiVersion: v1
kind: Pod
metadata:
  name: web-server
  labels:
    app: web-server
spec:
  containers:
  - name: nginx
    image: nginx
    ports:
    - containerPort: 80
```

等所有resource准备就绪之后，查看资源详情。主要是看看Pod和Service的IP，后面我们会在iptables的rule里面找到对应的记录。我们发现有3个nginx-pod，它们的IP分别是172.17.0.4，172.17.0.5，172.17.0.6；一个web-server pod，它的IP为172.17.0.14；此外还有一个nginx-service，它的IP是10.111.175.78。

```text
$ kubectl get all -o wide
NAME                                   READY   STATUS    RESTARTS   AGE     IP            NODE       NOMINATED NODE   READINESS GATES
pod/nginx-deployment-d46f5678b-847qb   1/1     Running   0          6h37m   172.17.0.5    minikube   <none>           <none>
pod/nginx-deployment-d46f5678b-99wkt   1/1     Running   0          6h37m   172.17.0.6    minikube   <none>           <none>
pod/nginx-deployment-d46f5678b-c9flw   1/1     Running   0          6h37m   172.17.0.4    minikube   <none>           <none>
pod/web-server                         1/1     Running   0          31s     172.17.0.14   minikube   <none>           <none>

NAME                    TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)   AGE     SELECTOR
service/kubernetes      ClusterIP   10.96.0.1       <none>        443/TCP   7h14m   <none>
service/nginx-service   ClusterIP   10.111.175.78   <none>        80/TCP    6h37m   app=nginx

NAME                               READY   UP-TO-DATE   AVAILABLE   AGE     CONTAINERS   IMAGES   SELECTOR
deployment.apps/nginx-deployment   3/3     3            3           6h37m   nginx        nginx    app=nginx

NAME                                         DESIRED   CURRENT   READY   AGE     CONTAINERS   IMAGES   SELECTOR
replicaset.apps/nginx-deployment-d46f5678b   3         3         3       6h37m   nginx        nginx    app=nginx,pod-template-hash=d46f5678b
```

OK，到此为止我们的环境准备好了，现在我们使用“minikube ssh“命令登陆到minikube的节点上，由于iptables命令需要root权限，所以“sudo su”使用root用户。

------

## kube-proxy iptable模式分析之ClusterIP

现在使用web-server的Pod访问nginx-service（10.111.175.78:80），web-server的ip为172.17.0.14。即数据包从172.17.0.14:xxxx（xxxx为随机端口）发往10.111.175.78:80，即

```text
Original Packet: 172.17.0.14:xxxx --> 10.111.175.78:80
```

我们来跟踪这个数据包流向来逐条查看iptables的规则，分析ClusterIP类型的Service的实现。

> 为什么只分析从本机出去的流量？
> 因为kube-proxy是以daemonSet的形式部署在所有节点上的，所以每个节点都会有相同的iptable规则，当任何一个节点上的pod访问service时，其实都是可以在该pod所在的node的的iptable中找到对应的service规则从而找到service所代理的pod的，而对于node而言，寄宿在自己上的pod的发出的流量就是从本机的某进程出去的流量。

在iptables的简介中我们得知，从本机的某进程出去的流量路线为：OUTPUT链 --> POSTROUTING链。注意，接下来的分析中显示的iptable的rule均为与本例相关的rule，无关rule将被省略。

首先我们分析OUTPUT链，OUTPUT链涉及到4张表raw，mangle，nat和filter，在minikube的iptable中我们发现raw和mangle是空表，所以后续分析均忽略这2张表。我们重点观察nat表和filter表。首先我们看nat表的OUTPUT规则：

```text
Chain OUTPUT (policy ACCEPT 9785 packets, 587K bytes)
 pkts bytes target     prot opt in     out     source               destination         
47092 2828K KUBE-SERVICES  all  --  *      *       0.0.0.0/0            0.0.0.0/0            /* kubernetes service portals */
 2799  168K DOCKER     all  --  *      *       0.0.0.0/0           !127.0.0.0/8          ADDRTYPE match dst-type LOCAL
```

可以看到，所有OUTPUT流量都被导向了名叫KUBE-SERVICES的自定义链，我们来看看它是做什么的：

```text
Chain KUBE-SERVICES (2 references)
 pkts bytes target     prot opt in     out     source               destination         
    ...
 17  1020 KUBE-SVC-GKN7Y2BSGW4NJTYL  tcp  --  *      *       0.0.0.0/0            10.111.175.78        /* default/nginx-service: cluster IP */ tcp dpt:80
    ...
1264 75952 KUBE-NODEPORTS  all  --  *      *       0.0.0.0/0            0.0.0.0/0            /* kubernetes service nodeports; NOTE: this must be the last rule in this chain */ ADDRTYPE match dst-type LOCAL
```

可以看到这里定义了所有namespace的service相关的规则，其中就有我们创建的nginx-service规则（其他几条service与coreDNS和apiServer相关，大家感兴趣的话可以自己分析一下），可以看到它匹配的是目标地址为10.111.175.78，端口为80的数据包，而我们发往nginx-service的数据包正好匹配这条规则，我们看到这条规则的target是名叫KUBE-SVC-GKN7Y2BSGW4NJTYL的自定义链，我们来继续挖这个链：

```text
Chain KUBE-SVC-GKN7Y2BSGW4NJTYL (1 references)
 pkts bytes target     prot opt in     out     source               destination         
    4   240 KUBE-SEP-ISPQE3VESBAFO225  all  --  *      *       0.0.0.0/0            0.0.0.0/0            /* default/nginx-service: */ statistic mode random probability 0.33333333349
    7   420 KUBE-SEP-RSPFZT7AP5F3PVUL  all  --  *      *       0.0.0.0/0            0.0.0.0/0            /* default/nginx-service: */ statistic mode random probability 0.50000000000
    6   360 KUBE-SEP-Y53CQAJAGI3VFGQO  all  --  *      *       0.0.0.0/0            0.0.0.0/0            /* default/nginx-service: */
```

我们看到这个KUBE-SVC-GKN7Y2BSGW4NJTYL链里面定义了3条规则，第一条规则有0.33333333349的概率匹配，也就是1/3的概率命中，第一条没命中的话第二条规则有1/2的概率命中，也就是2/3 * 1/2 = 1/3，第二条没命中的话就去第3条了。很明显，这里是在做负载均衡，那我们可以猜到这3条规则后面的target就是这个service代理的3个pod相关的规则了。我们来验证下：

```text
Chain KUBE-SEP-ISPQE3VESBAFO225 (1 references)
 pkts bytes target     prot opt in     out     source               destination         
    3   180 KUBE-MARK-MASQ  all  --  *      *       172.17.0.4           0.0.0.0/0            /* default/nginx-service: */
    4   240 DNAT       tcp  --  *      *       0.0.0.0/0            0.0.0.0/0            /* default/nginx-service: */ tcp to:172.17.0.4:80

Chain KUBE-SEP-RSPFZT7AP5F3PVUL (1 references)
 pkts bytes target     prot opt in     out     source               destination         
    0     0 KUBE-MARK-MASQ  all  --  *      *       172.17.0.5           0.0.0.0/0            /* default/nginx-service: */
    7   420 DNAT       tcp  --  *      *       0.0.0.0/0            0.0.0.0/0            /* default/nginx-service: */ tcp to:172.17.0.5:80

Chain KUBE-SEP-Y53CQAJAGI3VFGQO (1 references)
 pkts bytes target     prot opt in     out     source               destination         
    0     0 KUBE-MARK-MASQ  all  --  *      *       172.17.0.6           0.0.0.0/0            /* default/nginx-service: */
    6   360 DNAT       tcp  --  *      *       0.0.0.0/0            0.0.0.0/0            /* default/nginx-service: */ tcp to:172.17.0.6:80
```

可以看到这3个自定义链的规则很类似，注意到第一条匹配的是是Pod自己访问自己的情况，会去KUBE-MARK-MASQ这个target，其他的情况会去第二条规则，也就是DNAT。在我们的假设中，是另外一个pod访问nginx-service，所以不会命中第一条，命中第二条DNAT。

假设我们的数据包在KUBE-SEP-ISPQE3VESBAFO225链中被负载均衡分配到了第一个target，也就是KUBE-SEP-ISPQE3VESBAFO225，那么DNAT之后，该数据包的destination从10.111.175.78:80被改成了172.17.0.4:80，即：

```text
Original Packet: 172.17.0.14:xxxx --> 10.111.175.78:80
After DNAT:      172.17.0.14:xxxx --> 172.17.0.4:80
```

之后nat表的OUTPUT链中的规则结束，进入filter的OUTPUT链，其规则为：

```text
Chain OUTPUT (policy ACCEPT 583K packets, 167M bytes)
 pkts bytes target     prot opt in     out     source               destination         
48441 2909K KUBE-SERVICES  all  --  *      *       0.0.0.0/0            0.0.0.0/0            ctstate NEW /* kubernetes service portals */
4824K 1194M KUBE-FIREWALL  all  --  *      *       0.0.0.0/0            0.0.0.0/0
```

可以看到所有新建的连接（ctstate NEW）都会匹配第一条规则KUBE-SERVICES，而当查看filter表中的KUBE-SERVICES链时，我们发现这是一条空链。所以我们重点看第二条规则KUBE-FIREWALL：

```text
Chain KUBE-FIREWALL (2 references)
 pkts bytes target     prot opt in     out     source               destination         
    0     0 DROP       all  --  *      *       0.0.0.0/0            0.0.0.0/0            /* kubernetes firewall for dropping marked packets */ mark match 0x8000/0x8000
```

可以看到，所有被标记了0x8000/0x8000的数据包都会被直接DROP掉，而我们的数据包一路走过来没有被标记，所以不会被DROP。这样一来filter的OUTPUT规则也走完了，终于进入了下一个阶段 -- POSTROUTRING链。

POSTROUTING链涉及到2个表：mangle和nat，如前所述，mangle是张空表，所以我们只需要关注nat表的POSTROUTING规则：

```text
Chain POSTROUTING (policy ACCEPT 10630 packets, 667K bytes)
 pkts bytes target     prot opt in     out     source               destination         
50235 3118K KUBE-POSTROUTING  all  --  *      *       0.0.0.0/0            0.0.0.0/0            /* kubernetes postrouting rules */
   99  6102 MASQUERADE  all  --  *      !docker0  172.17.0.0/16        0.0.0.0/0           
```

首先进入第一个target，KUBE-POSTROUTING：

```text
Chain KUBE-POSTROUTING (1 references)
 pkts bytes target     prot opt in     out     source               destination         
    0     0 MASQUERADE  all  --  *      *       0.0.0.0/0            0.0.0.0/0            /* kubernetes service traffic requiring SNAT */ mark match 0x4000/0x4000 random-fully
```

我们发现，这条规则会把所有标记了0x4000/0x4000的数据包全部MASQUERADE（SNAT），由于我们的数据包没有被标记过，所以不匹配这条规则。我们的数据包又回到nat表的POSTROUTING链中，继续匹配第二条规则。第二规则表示所有source ip在172.17.0.0/16，且不是由docker0的网卡发出的数据包会匹配，而我们的数据包的source ip为172.17.0.14，符合172.17.0.0/16范围内，但是我们的数据包是从docker0网卡发出来的，所以不会命中这一条规则。

> 为什么说我们的数据包是从docker0网卡发出来的？
> 因为minikube默认采用的是bridge网桥模式，所以所有的pod的network namespace通过虚拟网卡和虚拟网线连接到docker0网桥上。因此，web-server pod其实是通过docker0网桥与nginx pod连接在一起的。当数据包经过iptable的OUTPUT链之后，经过路由表，路由表会把数据包导向docker0网卡的gateway，web-server pod发往nginx pod的数据包也是从docker0网卡走出来的。（这里我将单独写一遍文章专门分析）

为了验证数据包是从docker0网卡发出来的，我在nat的POSTROUTING链中增加了3条规则，target为LOG，执行以下命令：

```text
iptables -t nat -A POSTROUTING -d 172.17.0.4,172.17.0.5,172.17.0.6 -j LOG --log-prefix="After SNAT: "
```

执行上述命令后的nat的POSTROUTING链的规则：

```text
Chain POSTROUTING (policy ACCEPT 10630 packets, 667K bytes)
 pkts bytes target     prot opt in     out     source               destination         
50235 3118K KUBE-POSTROUTING  all  --  *      *       0.0.0.0/0            0.0.0.0/0            /* kubernetes postrouting rules */
   99  6102 MASQUERADE  all  --  *      !docker0  172.17.0.0/16        0.0.0.0/0           
    0   0 LOG        all  --  *      *       0.0.0.0/0            172.17.0.4           LOG flags 0 level 4 prefix "After SNAT: "
    0   0 LOG        all  --  *      *       0.0.0.0/0            172.17.0.5           LOG flags 0 level 4 prefix "After SNAT: "
    0   0 LOG        all  --  *      *       0.0.0.0/0            172.17.0.6           LOG flags 0 level 4 prefix "After SNAT: "
```

可以看出，这里对所有destination是nginx pod的数据包都打了日志，该日志可以在内核日志中查看，使用命令“dmesg”。在查看日志之前我们先模拟一次对nginx-service的请求“:

```text
$ kubectl exec web-server -it -- /bin/bash
root@web-server:/# curl 10.111.175.78:80
<!DOCTYPE html>
<html>
<head>
<title>Welcome to nginx!</title>
...
```

前面提到MASQUERADE是终结操作，如果数据包匹配到MASQUERADE的话，当前表的剩余规则将不再匹配。也就是说如果我们的数据包被MASQUERADE的话，将没有任何日志。查看日志如下：

```text
$ dmesg | grep "After SNAT: "
...
[26875.144760] After SNAT: IN= OUT=docker0 PHYSIN=veth926dffc PHYSOUT=vethb7031a1 SRC=172.17.0.14 DST=172.17.0.4 LEN=60 TOS=0x00 PREC=0x00 TTL=64 ID=52799 DF PROTO=TCP SPT=46782 DPT=80 WINDOW=64240 RES=0x00 SYN URGP=0 
```

我们发现有日志，说明没有被匹配到MASQUERADE。我们发现OUT=docker0，SRC=172.17.0.14，也就是数据包是从docker0网卡发出来的，因此并没有做SNAT操作，source ip由依然是web-server pod的IP 172.17.0.14。值得注意的是经历了之前的DNAT的修改，现在的数据包的DST=172.17.0.4，而不是service的ip 10.111.175.78。

现在总结下ClusterIP类型下的数据包经历的链：

```text
数据包 --> nat的OUTPUT --> nat的KUBE-SERVICES --> nat的KUBE-SVC-GKN7Y2BSGW4NJTYL
--> nat的KUBE-SEP-ISPQE3VESBAFO225（3选1，被DNAT）--> filter的OUTPUT --> filter的KUBE-SERVICES -->
filter的KUBE-FIREWALL --> nat的POSTROUTING --> nat的KUBE-POSTROUTING --> 没有被NAT
```

## kube-proxy iptable模式分析之NodePort

从大体上讲，NodePort类型的Service的iptable规则基本和ClusterIP的类似。下面仅列出几处不同的地方，首先是在nat的KUBE-SERVICES表处：

```text
Chain KUBE-SERVICES (2 references)
 pkts bytes target     prot opt in     out     source               destination         
    ...
1264 75952 KUBE-NODEPORTS  all  --  *      *       0.0.0.0/0            0.0.0.0/0            /* kubernetes service nodeports; NOTE: this must be the last rule in this chain */ ADDRTYPE match dst-type LOCAL
```

发往NodePort类型的数据包会命中最后一条，进入nat表的KUBE-NODEPORTS链：

```text
Chain KUBE-NODEPORTS (1 references)
 pkts bytes target     prot opt in     out     source               destination         
    ...
    0     0 KUBE-MARK-MASQ  tcp  --  *      *       0.0.0.0/0            0.0.0.0/0            /* default/nginx-service: */ tcp dpt:31628
    0     0 KUBE-SVC-GKN7Y2BSGW4NJTYL  tcp  --  *      *       0.0.0.0/0            0.0.0.0/0            /* default/nginx-service: */ tcp dpt:31628
    ...
```

可以看到数据包先会命中第一条规则KUBE-MARK-MASQ：

```text
Chain KUBE-MARK-MASQ (21 references)
 pkts bytes target     prot opt in     out     source               destination         
    0     0 MARK       all  --  *      *       0.0.0.0/0            0.0.0.0/0            MARK or 0x4000
```

在这里数据包会被标记上0x4000标记，然后回到KUBE-NODEPORTS链中继续匹配下一条规则，我们发现下一条规则就是KUBE-SVC-GKN7Y2BSGW4NJTYL，接下来一路到nat的POSTROUTING为止都与ClusterIP模式相同，但在接下来的nat的KUBE-POSTROUTING阶段：

```text
Chain KUBE-POSTROUTING (1 references)
 pkts bytes target     prot opt in     out     source               destination         
    0     0 MASQUERADE  all  --  *      *       0.0.0.0/0            0.0.0.0/0            /* kubernetes service traffic requiring SNAT */ mark match 0x4000/0x4000 random-fully
```

这里由于我们的数据包在KUBE-MARK-MASQ被打上了0x4000标记，在这里会命中这条规则，从而被MASQUERADE（SNAT）。我们来验证下，执行：

```text
$ iptables -t nat -I POSTROUTING -j LOG -d 172.17.0.4,172.17.0.5,172.17.0.6 --log-prefix="Before KUBE-POSTROUTING: "
$ iptables -t nat -I POSTROUTING 5 -j LOG -d 172.17.0.4,172.17.0.5,172.17.0.6 --log-prefix="After KUBE-POSTROUTING: "
```

在KUBE-POSTROUTING前后都插入LOG规则：

```text
Chain POSTROUTING (policy ACCEPT 45 packets, 3120 bytes)
 pkts bytes target     prot opt in     out     source               destination         
    0     0 LOG        all  --  *      *       0.0.0.0/0            172.17.0.6           LOG flags 0 level 4 prefix "Before KUBE-POSTROUTING: "
    0     0 LOG        all  --  *      *       0.0.0.0/0            172.17.0.5           LOG flags 0 level 4 prefix "Before KUBE-POSTROUTING: "
    0     0 LOG        all  --  *      *       0.0.0.0/0            172.17.0.4           LOG flags 0 level 4 prefix "Before KUBE-POSTROUTING: "
 136K 8668K KUBE-POSTROUTING  all  --  *      *       0.0.0.0/0            0.0.0.0/0            /* kubernetes postrouting rules */
    0     0 LOG        all  --  *      *       0.0.0.0/0            172.17.0.6           LOG flags 0 level 4 prefix "After KUBE-POSTROUTING: "
    0     0 LOG        all  --  *      *       0.0.0.0/0            172.17.0.5           LOG flags 0 level 4 prefix "After KUBE-POSTROUTING: "
    0     0 LOG        all  --  *      *       0.0.0.0/0            172.17.0.4           LOG flags 0 level 4 prefix "After KUBE-POSTROUTING: "
```

再用我们的web-server模拟一次访问

```text
$ kubectl exec web-server -it -- /bin/bash
root@web-server:/# curl 192.168.64.10:31628
<!DOCTYPE html>
<html>
<head>
<title>Welcome to nginx!</title>
...
```

其中192.168.64.10是minikube运行的虚拟机的IP，可以通过“minikube ip”命令查看。打印内核日志发现：

```text
[57038.560008] Before POSTROUTING: IN= OUT=docker0 PHYSIN=veth926dffc PHYSOUT=vethf4dbb0c SRC=172.17.0.14 DST=172.17.0.4 LEN=60 TOS=0x00 PREC=0x00 TTL=64 ID=35536 DF PROTO=TCP SPT=51372 DPT=80 WINDOW=64240 RES=0x00 SYN URGP=0 MARK=0x4000
```

只有Before POSTROUTING的日志，而且在这条日志的最后我们发现了MARK=0x4000，并没有发现After POSTROUTING的日志，因此在我们的数据包命中了KUBE-POSTROUTING中的规则被SNAT。source IP改成了docker0网卡的IP：172.17.0.1

```text
$ ip addr
...
4: docker0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP group default 
    link/ether 02:42:e2:bd:84:66 brd ff:ff:ff:ff:ff:ff
    inet 172.17.0.1/16 brd 172.17.255.255 scope global docker0
       valid_lft forever preferred_lft forever
```



总结下NodePort类型的数据包经历的链：

```text
数据包 --> nat的OUTPUT --> nat的KUBE-SERVICES --> nat的KUBE-NODEPORTS -->
--> nat的KUBE-MARK-MASQ (被标记) --> nat的KUBE-SVC-GKN7Y2BSGW4NJTYL
--> nat的KUBE-SEP-ISPQE3VESBAFO225（3选1，被DNAT）--> filter的OUTPUT --> filter的KUBE-SERVICES -->
filter的KUBE-FIREWALL --> nat的POSTROUTING --> nat的KUBE-POSTROUTING --> 被SNAT
```

------

## 总结

到此为止我们的数据包走完了所有的iptables的规则，总结一下主要的流程：

1. web-server向nginx-service发送请求，产生数据包 172.17.0.14:xxxx --> 10.111.175.78:80
2. 数据包经过nat的OUTPUT链后，经过iptable简陋的负载均衡找到service代理的其中一个pod的ip以及端口，数据包经过DNAT后变成 172.17.0.14:xxxx --> 172.17.0.4:80。
3. 数据包经过nat的POSTROUTING链后，service类型为ClusterIP时并没有发生什么变化，因为本次实验环境的minikue cluster使用的bridge模式；service类型为NodePort时，触发了SNAT。
4. 需要注意的是，使用不同的network plugin会对iptables产生不同的影响。大家可以试试使用flannel，calico等网络插件后，iptable的规则会产生哪些变化。其中，关于flannel的iptable可以参考另一位大佬的文章：

[int32bit：IPVS从入门到精通kube-proxy实现原理zhuanlan.zhihu.com](https://zhuanlan.zhihu.com/p/94418251)

到此为止，我们的数据包终于走完了所有的iptable的规则，但是接下来它该怎么到达nginx pod呢？下一篇文章我打算从linux network namespace开始一步步分析docker bridge network的构建，这将会给数据包的后续一个完整的解释，也将解释为什么web-server pod发往nginx pod的数据包会经过docker0网卡发出。敬请期待。