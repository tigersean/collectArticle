# Kube Proxy 原理

## 部署环境

3 master + 2 work 节点，只有两个work节点部署kubelet 和 kube-proxy。2个work节点 ip：

| haofan-test-1 | haofan-test-2 |
| ----------------- | ----------------- |
| 192.168.3.233     | 192.168.3.232     |

## 搭建一个GuestBook 例子

kubectl apply -f guestbook-all-in-one.yaml
 guestbook-all-in-one.yaml 如下：

```
apiVersion: v1
kind: Service
metadata:
  name: redis-master
  labels:
    app: redis
    role: master
    tier: backend
spec:
  ports:
  - port: 6379
    targetPort: 6379
  selector:
    app: redis
    role: master
    tier: backend
---
apiVersion: extensions/v1beta1
kind: Deployment
metadata:
  name: redis-master
spec:
  replicas: 1
  template:
    metadata:
      labels:
        app: redis
        role: master
        tier: backend
    spec:
      containers:
      - name: master
        image: hub.baidubce.com/public/guestbook-redis-master:e2e  # or just image: redis
        resources:
          requests:
            cpu: 100m
            memory: 100Mi
        ports:
        - containerPort: 6379
---
apiVersion: v1
kind: Service
metadata:
  name: redis-slave
  labels:
    app: redis
    role: slave
    tier: backend
spec:
  ports:
  - port: 6379
  selector:
    app: redis
    role: slave
    tier: backend
---
apiVersion: extensions/v1beta1
kind: Deployment
metadata:
  name: redis-slave
spec:
  selector:
    matchLabels:
      app: redis
      role: slave
      tier: backend
  replicas: 2
  template:
    metadata:
      labels:
        app: redis
        role: slave
        tier: backend
    spec:
      containers:
      - name: slave
        image: hub.baidubce.com/public/guestbook-redis-slave:v1
        resources:
          requests:
            cpu: 100m
            memory: 100Mi
        env:
        - name: GET_HOSTS_FROM
          value: dns
          # Using `GET_HOSTS_FROM=dns` requires your cluster to
          # provide a dns service. As of Kubernetes 1.3, DNS is a built-in
          # service launched automatically. However, if the cluster you are using
          # does not have a built-in DNS service, you can instead
          # instead access an environment variable to find the master
          # service's host. To do so, comment out the 'value: dns' line above, and
          # uncomment the line below:
          # value: env
        ports:
        - containerPort: 6379
---
apiVersion: v1
kind: Service
metadata:
  name: frontend
  labels:
    app: guestbook
    tier: frontend
spec:
  # comment or delete the following line if you want to use a LoadBalancer
  type: NodePort
  # if your cluster supports it, uncomment the following to automatically create
  # an external load-balanced IP for the frontend service.
  ports:
  - port: 80
  selector:
    app: guestbook
    tier: frontend
---
apiVersion: apps/v1beta2
kind: Deployment
metadata:
  name: frontend
spec:
  selector:
    matchLabels:
      app: guestbook
      tier: frontend
  replicas: 3
  template:
    metadata:
      labels:
        app: guestbook
        tier: frontend
    spec:
      containers:
      - name: php-redis
        image: hub.baidubce.com/public/guestbook-frontend:v4
        resources:
          requests:
            cpu: 100m
            memory: 100Mi
        env:
        - name: GET_HOSTS_FROM
          value: dns
          # Using `GET_HOSTS_FROM=dns` requires your cluster to
          # provide a dns service. As of Kubernetes 1.3, DNS is a built-in
          # service launched automatically. However, if the cluster you are using
          # does not have a built-in DNS service, you can instead
          # instead access an environment variable to find the master
          # service's host. To do so, comment out the 'value: dns' line above, and
          # uncomment the line below:
          # value: env
        ports:
        - containerPort: 80
123456789101112131415161718192021222324252627282930313233343536373839404142434445464748495051525354555657585960616263646566676869707172737475767778798081828384858687888990919293949596979899100101102103104105106107108109110111112113114115116117118119120121122123124125126127128129130131132133134135136137138139140141142143144145146147148149
```

## 分析iptables

查看新创建的service, frontend的容器端口是80, nodePort端口是30784

```
[root@haofan-test-2 ~]# kubectl get svc --all-namespaces
default    frontend                NodePort    172.16.92.224    <none>        80:30784/TCP        23h
default    redis-master            ClusterIP   172.16.155.140   <none>        6379/TCP            23h
default    redis-slave             ClusterIP   172.16.67.204    <none>        6379/TCP            23h
```

查看frontend pod的分布情况

```
[root@haofan-test-2 ~]# kubectl get pods -o wide -n production
NAME                            READY     STATUS    RESTARTS   AGE       IP               NODE         NOMINATED NODE
frontend-5798c4bfc7-7bt7k       1/1       Running   0          23h       172.18.1.22   192.168.3.233   <none>
frontend-5798c4bfc7-nbmvp       1/1       Running   0          23h       172.18.0.20   192.168.3.232   <none>
frontend-5798c4bfc7-v7889       1/1       Running   0          23h       172.18.1.23   192.168.3.233   <none>
12345
```

### 1 创建iptables实现外网通过nodePort访问

思考：

如果在集群中通过cluster_ip:port 是如何访问到frontend 172.18.1.22:80

方法：

1. 要进行DNAT转换，因为数据包是经过本地协议发出 会经过nat的OUTPUT chain. 目的是访问cluster_ip:port时可以访问到pod ip。

```
iptables -t nat -I OUTPUT -p tcp -m
comment --comment "this is clusterip demo" -d 172.16.92.224/32 --dport 80 -j DNAT --to-destination 172.18.1.22:80
```

2. 如果在集群外部中通过node_ip: port 是如何访问到frontend 172.18.1.22:80, 同样要进行DNAT转换，为了让消息能返回客户端，还需要进行SNAT转换。思考为什么要进行SNAT ？

```
iptables -t nat -I POSTROUTING -p tcp -d 172.18.1.22 --dport 80 -j MASQUERADE
iptables -t nat -I PREROUTING -p tcp -m comment --comment "this is nodeport demo" --dport 30784 -j DNAT --to-destination 172.18.1.22:80
```

MASQUERADE和SNAT的功能类似，只是SNAT需要明确指明源IP的的值，MASQUERADE会根据网卡IP自动更改，所以更实用一些。

所以说，如果在开发过程中，想外部访问某个应用(比如redis)，但是呢碰巧这个应用的svc又没有开启nodeport，那你就可以模仿我刚才设置的iptable规则，从而达到不修改SVC也能通过外部应用访问。

**这里为什么要进行SNAT呢 ？**
 原因是，为了支持从任一节点IP+NodePort都可以访问应用。

```
                                                                client
                                                          \ ^
                                                            \ \
                                                              v \
   (eth0:192.168.3.1)node 1 <--- node 2（eth0: 192.168.2.1）
    | ^   SNAT
    | |   --->
    v |
 endpoint
123456789
```

假设跳过上图的SNAT，只做DNAT。报文的确可以经过 node 2 转发到 node 1上的endpoint，source地址是client ip，但是endpoint如何应答呢？endpoint内的确有默认路由指向 node 1，但是如果没有做SNAT，应答时会直接从 node 1 发送给client。

**这是一个三角流量！！**

跳过SNAT后，node 1 在转发应答流量的时候，会将应答报文的源地址替换为 node 1的地址，这样的报文，client是不会接受的，连接将无法建立，因为TCP协议：client request报文的源地址 和 server的response报文的目的地址要相同，否则无法建立连接。

如果有了SNAT, node 2 到 node 1的packet 到了node1之后，source地址是node 2的 eth0 地址，这样在response后，就会从node1 到node2再转发出去，而不会直接通过node1转发出去。

所以，必须要做SNAT，必须要FULLNAT。

### 2. 分析k8s的 iptables

#### 2.1 集群内部通过cluster ip 访问到Pod

##### 2.1.1 iptables分析

1. ​	数据包是通过本地协议发出的，然后需要更改NAT表，k8s只能在OUTPUT这个链上来动手

   到达OUPUT chain后，要经过kube-services这个k8s自定义的链。

```
root@haofan-test-2 ~]# iptables -L OUTPUT
Chain OUTPUT (policy ACCEPT)
target     prot opt source               destination
KUBE-SERVICES  all  --  anywhere             anywhere             ctstate NEW /* kubernetes service portals */
```

然后匹配到下面两条链：

```
[root@haofan-test-2 ~]# iptables -L KUBE-SERVICES -t nat -n --line-number | grep "80"
KUBE-MARK-MASQ  tcp  -- !172.18.0.0/16         172.18.1.23        /* default/frontend: cluster IP */ tcp dpt:80
KUBE-SVC-GYQQTB6TY565JPRW  tcp  --  0.0.0.0/0             172.18.1.23        /* default/frontend: cluster IP */ tcp dpt:80
```

第一条chain，是对源地址不是172.18.0.0/16的，目的地址是 172.18.1.23，目的端口是80打标签，标签后的packet进行SNAT，目的就是为了伪装所有访问 Service Cluster IP 的外部流量。

再看 KUBE-SVC-GYQQTB6TY565JPRW， 发现了probability，实现了svc能够随机访问到后端

```
[root@haofan-test-1 ~]# iptables -S -t nat | grep KUBE-SVC-GYQQTB6TY565JPRW
-A KUBE-SVC-GYQQTB6TY565JPRW -m comment --comment "production/frontend:" -m statistic --mode random --probability 0.33332999982 -j KUBE-SEP-ABPIR2RYBUVZX2WC
-A KUBE-SVC-GYQQTB6TY565JPRW -m comment --comment "production/frontend:" -m statistic --mode random --probability 0.50000000000 -j KUBE-SEP-G27GUN3FGJW5PMGU
-A KUBE-SVC-GYQQTB6TY565JPRW -m comment --comment "production/frontend:" -j KUBE-SEP-P2WQD6E6PI2AV6SJ
```

因为有3个pod, 只看其中一个。发现有两条规则，第一条打标签0x4000是为了做SNAT，第二条实现DNAT

```
 [root@haofan-test-2 ~]# iptables -L KUBE-SEP-P2WQD6E6PI2AV6SJ -t nat -n
Chain KUBE-SEP-P2WQD6E6PI2AV6SJ (1 references)
target     prot opt source               destination
KUBE-MARK-MASQ  all  --  172.18.1.23          0.0.0.0/0            /* default/frontend: */
DNAT       tcp  --  0.0.0.0/0            0.0.0.0/0            /* default/frontend: */ tcp to:172.18.1.23:80
[root@haofan-test-1 ~]# iptables -S -t nat | grep KUBE-MARK-MASQ
-N KUBE-MARK-MASQ
-A KUBE-MARK-MASQ -j MARK --set-xmark 0x4000/0x4000
```

然后在KUBE-POSTROUTING链时，只对打上了标签的packet进行SNAT

```
root@haofan-test-1 ~]# iptables -S -t nat | grep 0x4000/0x4000
-A KUBE-MARK-MASQ -j MARK --set-xmark 0x4000/0x4000
-A KUBE-POSTROUTING -m comment --comment "kubernetes service traffic requiring SNAT" -m mark --mark 0x4000/0x4000 -j MASQUERADE
```

至此，一个访问cluster ip 最终访问到Pod的流程走完了。

##### 2.1.2 抓包分析

抓包分析的时候，为方便，只部署了一个frontend实例, frontend实例只跑在haofan-test-1 node上。

###### 2.1.1.1 在haofan-test-1上 直接访问 cluster ip: port

```
[root@haofan-test-1 ~]# curl 172.16.92.224:80
```

在frontend pod 中抓包如下，看到source地址是172.18.1.1，可以认为是容器网关地址。在haofan-test-1上直接curl 172.18.1.23:80, source地址其实是192.168.3.233，因为默认路由指向了eth0。匹配到了chain <是对源地址不是172.18.0.0/16的，目的地址是 172.18.1.23，目的端口是80打标签，标签后的packet进行SNAT> ，注意这是request packet的SNAT，不是response的SNAT。

```
 [root@haofan-test-1 ~]# kubectl exec -it frontend-5798c4bfc7-vrwcx bash
 root@frontend-5798c4bfc7-vrwcx:/var/www/html# tcpdump port 80 -n
        1:35:24.027687 IP 172.18.1.23.80 > 172.18.1.1.34266: Flags [P.], seq 1:1185, ack 78, win 211, options [nop,nop,TS val 2316549007 ecr 2316549007], length 1184: HTTP: HTTP/1.1 200 OK
    01:35:24.027728 IP 172.18.1.1.34266 > 172.18.1.23.80: Flags [.], ack 1185, win 234, options [nop,nop,TS val 2316549008 ecr 2316549007], length 0
    01:35:24.027893 IP 172.18.1.1.34266 > 172.18.1.23.80: Flags [F.], seq 78, ack 1185, win 234, options [nop,nop,TS val 2316549008 ecr 2316549007], length 0
    01:35:24.027957 IP 172.18.1.23.80 > 172.18.1.1.34266: Flags [F.], seq 1185, ack 79, win 211, options [nop,nop,TS val 2316549008 ecr 2316549008], length 0
    01:35:24.027984 IP 172.18.1.1.34266 > 172.18.1.23.80: Flags [.], ack 1186, win 234, options [nop,nop,TS val 2316549008 ecr 2316549008], length 0
```

##### 2.1.1.2 在redis pod 内上 直接访问 cluster ip: port

可以看到并没有做SNAT,source 地址是redis的pod地址。

```
 [root@haofan-test-2 ~]# kubectl exec -it redis-slave-6566d8d846-n2phs bash
root@redis-slave-6566d8d846-n2phs:/data# curl 172.16.92.224:80

root@frontend-5798c4bfc7-vrwcx:/var/www/html# tcpdump port 80 -n
tcpdump: verbose output suppressed, use -v or -vv for full protocol decode
listening on eth0, link-type EN10MB (Ethernet), capture size 262144 bytes
01:56:01.425643 IP 172.18.1.52.41838 > 172.18.1.23.80: Flags [S], seq 4271004944, win 27200, options [mss 1360,sackOK,TS val 2317786405 ecr 0,nop,wscale 7], length 0
01:56:01.425723 IP 172.18.1.23.80 > 172.18.1.52.41838: Flags [S.], seq 3853670533, ack 4271004945, win 26960, options [mss 1360,sackOK,TS val 2317786406 ecr 2317786405,nop,wscale 7], length 0
01:56:01.425753 IP 172.18.1.52.41838 > 172.18.1.23.80: Flags [.], ack 1, win 213, options [nop,nop,TS val 2317786406 ecr 2317786406], length 0
```

#### 2.2 集群外部通过node ip 访问到Pod

##### 2.2.1 iptables分析

根据iptables, 肯定是对PREROUTING链动手脚

```
 [root@haofan-test-1 ~]# iptables -S -t nat | grep PREROUTING
    -A PREROUTING -m comment --comment "kubernetes service portals" -j KUBE-SERVICES
 [root@haofan-test-2 ~]# iptables -L PREROUTING -t nat
Chain PREROUTING (policy ACCEPT)
target     prot opt source               destination
KUBE-SERVICES  all  --  anywhere             anywhere             /* kubernetes service portals */
```

然后又调到KUBE-SERVICES这个chain上去了，但是因为根据具体的destination 地址，只能匹配到下面˙这条chain

```
[root@haofan-test-2 ~]# iptables -L KUBE-SERVICES -t nat -n
KUBE-NODEPORTS  all  --  0.0.0.0/0            0.0.0.0/0            /* kubernetes service nodeports; NOTE: this must be the last rule in this chain */ ADDRTYPE match dst-type LOCAL
```

KUBE-NODEPORTS最后还是调到之前的KUBE-SVC-PKCF2FTRAH6WFOQR

```
    [root@haofan-test-2 ~]# iptables -L KUBE-NODEPORTS -t nat --line-number
Chain KUBE-NODEPORTS (1 references)
num  target                     prot opt  source             destination
1    KUBE-MARK-MASQ             tcp  --  anywhere             anywhere             /* default/frontend: */ tcp dpt:30784
2    KUBE-SVC-PKCF2FTRAH6WFOQR  tcp  --  anywhere             anywhere             /* default/frontend: */ tcp dpt:30784
```

然后这里注意，2条规则并不是匹配完第一条，就不匹配第二条了，iptables匹配完第一条不匹配第二条是对于prerouting/input/output/postrouting 的规则，自定义的规则最终都会append到这4个chain上的, 也就是在内核中实际上是不同的chain。 第一条就是目的端口是30784的packet进行SNAT，第二条就不用说了。SNAT的目的前面已经说了，SNAT之后进入pod的packet的source地址是容器网关地址，抓包可以验证。

经过上面的描述，应该对网络packet数据转发有一个比较清楚的认识。

##### 2.2.2 抓包分析

在haofan-test-1 节点上，访问node1 ip:node port, 可以看到source地址是172.18.1.1, 说明做了SNAT

```
[root@haofan-test-1 ~]# curl 192.168.3.233:32214
root@frontend-5798c4bfc7-vrwcx:/var/www/html# tcpdump port 80 -n
tcpdump: verbose output suppressed, use -v or -vv for full protocol decode
listening on eth0, link-type EN10MB (Ethernet), capture size 262144 bytes
02:02:56.127994 IP 172.18.1.1.46634 > 172.18.1.23.80: Flags [S], seq 1822455295, win 43690, options [mss 65495,sackOK,TS val 2318201108 ecr 0,nop,wscale 7], length 0
02:02:56.128057 IP 172.18.1.23.80 > 172.18.1.1.46634: Flags [S.], seq 1731607107, ack 1822455296, win 26960, options [mss 1360,sackOK,TS val 2318201108 ecr 2318201108,nop,wscale 7], length 0
02:02:56.128088 IP 172.18.1.1.46634 > 172.18.1.23.80: Flags [.], ack 1, win 342, options [nop,nop,TS val 2318201108 ecr 2318201108], length 0
1234567
```

如果在haofan-test-1节点上，访问node2 ip:node port, 可以看到source地址是node2的IP，原因就是上面解释的为什么要做SNAT。

```
[root@haofan-test-1 ~]# curl 192.168.3.232:32214
root@frontend-5798c4bfc7-vrwcx:/var/www/html# tcpdump port 80 -n
tcpdump: verbose output suppressed, use -v or -vv for full protocol decode
listening on eth0, link-type EN10MB (Ethernet), capture size 262144 bytes
02:21:58.521488 IP 192.168.3.232.49114 > 172.18.1.23.80: Flags [S], seq 2810163436, win 27200, options [mss 1360,sackOK,TS val 2319343500 ecr 0,nop,wscale 7], length 0
02:21:58.521561 IP 172.18.1.23.80 > 192.168.3.232.49114: Flags [S.], seq 3145056659, ack 2810163437, win 26960, options [mss 1360,sackOK,TS val 2319343501 ecr 2319343500,nop,wscale 7], length 0
02:21:58.522490 IP 192.168.3.232.49114 > 172.18.1.23.80: Flags [.], ack 1, win 213, options [nop,nop,TS val 2319343502 ecr 2319343501], length 0
1234567
```

#### 2.3 总结

1. (inbound)在PREROUTING阶段，将所有报文转发到KUBE-SERVICES

2. (outbound)在OUTPUT阶段，将所有报文转发到KUBE-SERVICES5. 

3. (outbound)在POSTROUTING阶段，将所有报文转发到KUBE-POSTROUTING

   ![image-20200821120858180](./Untitled.assets/image-20200821120858180.png)

4. 图中共有三个地方看到了KUBE-MARK-MASQ，前2个的原因是为了防止上面所说的"三角流量"， 最后一个的原因，是进行SNAT，将pod的地址，转成网关地址，类似容器网关地址，然后再通过SNAT转成node ip地址，最后转发出去。图中共有三个地方看到了KUBE-MARK-MASQ，前2个的原因是为了防止上面所说的"三角流量"， 最后一个的原因，是进行SNAT，将pod的地址，转成网关地址，类似容器网关地址，然后再通过SNAT转成node ip地址，最后转发出去。

5. 这样如上图描述，每添加一个有N个endpoints的nodeport类型service:port，新增(2 + 2 + (1 + 2) * N)条规则，新增1+N条链。

## Refer

1. https://zhuanlan.zhihu.com/p/28289080
2. http://cizixs.com/2017/03/30/kubernetes-introduction-service-and-kube-proxy/
3. https://k8smeetup.github.io/docs/tutorials/services/source-ip/
4. https://www.lijiaocn.com/%E9%A1%B9%E7%9B%AE/2017/03/27/Kubernetes-kube-proxy.html