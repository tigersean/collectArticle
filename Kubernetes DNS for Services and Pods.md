# Kubernetes DNS for Services and Pods

As we know, a Kubernetes master stores all service definitions and updates. Client pods that need to communicate with backend pods load-balanced by a service, however, also need to know where to send their requests. They can store network information in the container environmental variables, but this is not viable in the long run. If the network details and a set of backend pods change in the future, client pods would be incapable of communicating with them.

Kubernetes DNS system is designed to solve this problem. [Kube-DNS](https://github.com/kubernetes/dns) and [CoreDNS](https://github.com/coredns/coredns) are two established DNS solutions for defining DNS naming rules and resolving pod and service DNS to their corresponding cluster IPs. With DNS, Kubernetes services can be referenced by name that will correspond to any number of backend pods managed by the service. The naming scheme for DNS also follows a predictable pattern, making the addresses of various services more memorable. Services can also be referenced not only via a Fully Qualified Domain Name (FQDN) but also via only the name of the service itself.

In this blog post, we discuss the design of Kubernetes DNS system and show practical examples of using DNS with services and debugging DNS issues. Let’s get started!

# How Does Kubernetes DNS Work?

In Kubernetes, you can set up a DNS system with two well-supported add-ons: CoreDNS and Kube-DNS. CoreDNS is a newer add-on that became a default DNS server as of Kubernetes v1.12. However, Kube-DNS may still be installed as a default DNS system by certain Kubernetes installer tools.

Both add-ons schedule a DNS pod or pods and a service with a static IP on the cluster and both are named kube-dns in the `metadata.name` field for interoperability. When the cluster is configured by the administrator or installation tools, the kubelet passes DNS functionality to each container with the`--cluster-dns=<dns-service-ip>` flag. When configuring the kubelet , the administrator can also specify the name of a local domain using the flag `--cluster-domain=<default-local-domain>` .

Kubernetes DNS add-ons currently support forward lookups (A records), port lookups (SRV records), reverse IP address lookups (PTR records), and some other options. In the following sections, we discuss the Kubernetes naming schema for pods and services within these types of records.

# Service DNS Records

In general, Kubernetes services support A records, CNAME, and SRV records.

## A Record

[A Record](https://en.wikipedia.org/wiki/List_of_DNS_record_types) is the most basic type of a DNS record used to point a domain or subdomain to a certain IP address. The record consists of the domain name, the IP address to resolve it, and TTL in seconds. TTL stands for Time To Live, and is a sort of expiration date put on a DNS record. A TTL tells the DNS server how long it should keep a given record in its cache.

Kubernetes assigns different A record names for “normal” and “headless” services. As you remember from our [earlier tutorial](https://supergiant.io/blog/kubernetes-networking-services), “headless” services are different from “normal” services in that they are not assigned a ClusterIP and don’t perform load balancing.

“Normal” services are assigned a DNS A record for a name of the form `your-svc.your-namespace.svc.cluster.local` (the root domain name may be changed in the kubelet settings). This name resolves to the cluster IP of the Service. “Headless” services are also assigned a DNS A record for a name of the form `your-svc.your-namespace.svc.cluster.local` . However, in contrast to a “normal” service, this name resolves to a set of IPs of the pods selected by the service. The DNS will not resolve this set to a specific IP automatically so the clients should take care of load balancing or round-robin selection from the set.

## CNAME

[CNAME](https://en.wikipedia.org/wiki/CNAME_record) records are used to point a domain or subdomain to another hostname. To achieve this, CNAMEs use the existing A record as their value. In its turn, an A record subsequently resolves to a specified IP address. Also, in Kubernetes, CNAME records can be used for [cross-cluster service discovery with federated services](https://kubernetes.io/docs/tasks/federation/federation-service-discovery/). In this scenario, there is a common Service across multiple Kubernetes clusters. This service can be discovered by all pods no matter what cluster they are living on. Such an approach allows for cross-cluster service discovery, which is a big topic in its own right to be discussed in another tutorial.

## SRV Records

SRV records facilitate service discovery by describing the protocol/s and address of certain services.

An SRV record usually defines a symbolic name and the transport protocol (e.g., TCP) used as part of the domain name and defines the priority, weight, port, and target for a given service (see the example below)

```
_sip._tcp.example.com.   3600 IN    SRV 10       70     5060 srvrecord.example.com_sip._tcp.example.com.   3600 IN    SRV 10       20     5060 srvrecord2.example.com
```

In the example above, `_sip` is the service’s symbolic name and `_tcp` is the transport protocol used by the service. The record’s content defines a priority of 10 for both records. Additionally, the first record has a weight of 70 and the second one has a weight of 20. The priority and weight are often used to encourage the use of certain servers over others. The final two values in the record define the port and hostname to connect to in oder to communicate with the service.

In Kubernetes, SRV Records are created for named ports that are part of a “normal” or “headless” service. The SRV record takes the form of `_my-port-name._my-port-protocol.my-svc.my-namespace.svc.cluster.local` . For a regular service, this resolves to the port number and the domain name: `my-svc.my-namespace.svc.cluster.local `. In case of a “headless” service, this name resolves to multiple answers, one for each pod backing the service. Each answer contains the port number and the domain name of the pod of the form `auto-generated-name.my-svc.my-namespace.svc.cluster.local` .

# Pod DNS Records

## A Records

If DNS is enabled, pods are assigned a DNS A record in the form of `pod-ip-address.my-namespace.pod.cluster.local` . For example, a pod with IP `172.12.3.4` in the namespace `default` with a DNS name of `cluster.local` would have an entry of the form `172–12–3–4.default.pod.cluster.local `.

## Pod’s Hostname and Subdomain Fields

The default hostname for a pod is defined by a pod’s `metadata.name` value. However, users can change the default hostname by specifying a new value in the optional hostname field. Users can also define a custom subdomain name in a subdomain field. For example, a pod with its hostname set to `custom-host`, and subdomain set to `custom-subdomain`, in namespace `my-namespace`, will have the fully qualified domain name (FQDN) `custom-host.custom-subdomain.my-namespace.svc.cluster.local`.

# Tutorial

Now we demonstrate for you how to address services by their DNS names, check the DNS resolution, and debug DNS issues when they occur. To complete examples used below, you’ll need the following prerequisites:

- A running Kubernetes cluster. See [Supergiant documentation](https://supergiant.readme.io/docs) for more information about deploying a Kubernetes cluster with Supergiant. As an alternative, you can install a single-node Kubernetes cluster on a local system using [Minikube](https://kubernetes.io/docs/getting-started-guides/minikube/).
- A **kubectl** command line tool installed and configured to communicate with the cluster. See how to install **kubectl** [here](https://kubernetes.io/docs/tasks/tools/install-kubectl/).

First, let’s create a Deployment with three Python HTTP servers that listen on the port 80 for connections and return a custom greeting containing a pod’s hostname.

```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: test-deployment
spec:
  replicas: 3
  selector:
    matchLabels:
      app: test-pod
  template:
    metadata:
      labels:
        app: test-pod
    spec:
      containers:
      - name: python-http-server
        image: python:2.7
        command: ["/bin/bash"]
        args: ["-c", "echo \" Hello from $(hostname)\" > index.html; python -m SimpleHTTPServer 80"]
        ports:
        - name: http
          containerPort: 80
```

Let’s create the deployment:

```
kubectl create -f deployment.yml
deployment.extensions “test-deployment” created
```

Next, we need to create a service that will discover the deployment’s pods and distribute client requests among them. Below is a manifest for a “normal” service that will be assigned a ClusterIP.

```
kind: Service
apiVersion: v1
metadata:
  name: test-service
spec:
  selector:
    app: test-pod
  ports:
  - protocol: TCP
    port: 4000
    targetPort: http
```

Please, note that the `spec.selector` field of the service should match the `spec.template.metadata.labels` of the pod created by the Deployment.

```
kubectl create -f service.yml
service “test-service” created
```

Finally, we need to create a client pod that will `curl` the service by its name. This way we don’t need to know the IPs of the service’s endpoints and be dependent on the ephemeral nature of Kubernetes pods.

```
apiVersion: v1
kind: Pod 
metadata: 
  name: client-pod
spec:
  containers:
  - name: curl
    image: appropriate/curl
    command: ["/bin/sh"]
    args: ["-c","curl test-service:4000 "]
```

Please note that we are using the name of the Service instead of its ClusterIP or IPs of pods created by the Deployment. We can use a DNS name of the service (“test-service”) because our Kubernetes cluster uses a **Kube-DNS** add-on that watches the Kubernetes API for new services and creates DNS records for each of them. If **Kube-DNS** is enabled across your cluster, then all pods can perform name resolution of services automatically. However, you can certainly continue to use the **ClusterIP** of your service.

```
kubectl create -f client.yml
pod “client-pod” created
```

Once the client pod is created, let’s see its logs to verify that the Service’s name resolved to correct backend pods:

```
kubectl logs client-pod% Total    % Received % Xferd  Average Speed   Time    Time     Time  CurrentDload  Upload   Total   Spent    Left  SpeedHello from test-deployment-84dc998fc5-pkmsc100    45  100    45    0     0   3000      0 --:--:-- --:--:-- --:--:--  3214
```

The response above indicates that Kube-DNS has correctly resolved the service name to the service’s ClusterIP and the service has successfully forwarded the client request to random backend pod picked in a round-robin fashion. In its turn, the selected pod returned its custom greeting, which you can see in the response above.

# Using `nslookup` to Check DNS Resolution

Now, let’s verify that DNS works correctly if we look up the FQDN defined by the A record. To do this, we’ll need to get a shell to a running pod and use `nslookup` command inside it.

First, let’s find the pods created by the deployment:

```
kubectl get pods -l app=test-podNAME                              READY     STATUS    RESTARTS   AGEtest-deployment-84dc998fc5-772gj   1/1       Running   0          1m
test-deployment-84dc998fc5-fh5pf   1/1       Running   0          1m
test-deployment-84dc998fc5-pkmsc   1/1       Running   0          1m
```

Select one of these pods and get a shell to it using the command below (use your unique pod’s name):

```
kubectl exec -ti test-deployment-84dc998fc5-772gj -- /bin/bash
```

Next step, we’ll need to install nslookup command available in the BusyBox package:

```
apt-get update
apt-get install busybox
```

After BusyBox is installed, let’s check the DNS of the Service:

```
busybox nslookup test-service.default.svc.cluster.local
Server:    10.96.0.10
Address 1: 10.96.0.10 kube-dns.kube-system.svc.cluster.local
Name:      test-service.default.svc.cluster.local
Address 1: 10.109.90.121 test-service.default.svc.cluster.local
```

In the command above, we used the naming schema for the service’s A record. Let’s verify that the DNS lookup resolved the service DNS to the correct IP (A record).

```
kubectl describe svc test-serviceName:              test-service
Namespace:         default
Labels:            <none>
Annotations:       <none>
Selector:          app=test-pod
Type:              ClusterIP
IP:                10.109.90.121
Port:              <unset>  4000/TCP
TargetPort:        http/TCP
Endpoints:         172.17.0.11:80,172.17.0.15:80,172.17.0.19:80
Session Affinity:  None
Events:            <none>
```

That looks correct! You can see that the ClusterIP of the service is `10.109.90.121` — the same IP to which the DNS lookup resolved.

# Debugging DNS

If the `nslookup` command failed for some reason, you have several debugging and troubleshooting options. However, how do you know that the DNS lookup failed in the first place? If DNS fails, you’ll usually get responses like this:

```
kubectl exec -ti busybox -- nslookup kubernetes.defaultServer:    10.0.0.10
Address 1: 10.0.0.10
nslookup: can't resolve 'kubernetes.default'
```

The first thing you need to do in case of this error, is to check if DNS configuration is correct. Let’s take a look at the `resolv.conf` file inside the container.

```
kubectl exec test-deployment-84dc998fc5-772gj cat /etc/resolv.conf
```

Verify that a search path and a name server are set up correctly as in the example below (note that search path may vary for different cloud providers):

```
nameserver 10.96.0.10
search default.svc.cluster.local svc.cluster.local cluster.local
options ndots:5
```

If the `/etc/resolve.conf` has all correct entries, you’ll need to check whether the kube-dns/coredns plugin is enabled. On Minikube, run:

```
minikube addons list- addon-manager: enabled
- coredns: disabled
- dashboard: enabled
- default-storageclass: enabled
- efk: disabled
- freshpod: disabled
- heapster: disabled
- ingress: disabled
- kube-dns: enabled
- metrics-server: enabled
- registry: disabled
- registry-creds: disabled
- storage-provisioner: enabled
```

As you see, we have the `kube-dns` enabled. If your DNS add-on is not running, you can try to enable it with the following command:

```
minikube addons enable kube-dnskube-dns was successfully enabled
```

Alternatively, you can check if the kubedns/coredns pods are running:

```
kubectl get pods --namespace=kube-systemNAME                                    READY     STATUS    RESTARTS   AGE....
kube-dns-86f4d74b45-2qkfd               3/3       Running   232        133d
kube-proxy-b2frq                        1/1       Running   0          15m
...
```

If the pod is running, there might be something wrong with the global DNS service. Let’s check it:

```
$ kubectl get svc --namespace=kube-system
NAME      TYPE       CLUSTER-IP     EXTERNAL-IP   PORT(S)     AGE kube-dns ClusterIP   10.96.0.10       <none>   53/UDP,53/TCP
```

You might also need to check whether DNS endpoints are exposed:

```
kubectl get ep kube-dns --namespace=kube-systemNAME       ENDPOINTS                     AGE
kube-dns   172.17.0.5:53,172.17.0.5:53   133d
```

These debugging actions will usually indicate the problem with your DNS configuration, or it will simply show you that a DNS add-on should be enabled in your cluster configuration.

# Conclusion

To summarize, Kubernetes enables efficient service discovery with its built-in DNS add-ons: Kube-DNS or CoreDNS.

Kubernetes DNS system assigns domain and sub-domain names to pods, ports, and services, which allows them to be discoverable by other components inside your Kubernetes cluster.

DNS-based service discovery is very powerful because you don’t need to hard-code network parameters like IPs and ports into your application. Once a set of pods is managed by a service, you can easily access them using the service’s DNS.

