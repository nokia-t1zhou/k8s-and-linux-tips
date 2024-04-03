### 什么是K8s service
下图是一个简单的K8s cluster部署图，pod部署在不同的node中，每个pod都有被K8s CNI分配一个IP地址（在没有部署额外的多网络CNI的情况下，比如multus CNI），Kubernetes 假设 Pod 可与其它 Pod 通信，不管它们在哪个node上。 所以没必要在 Pod 与 Pod 之间创建连接或将容器的端口映射到主机端口。 这意味着同一个 Pod 内的所有容器能通过 localhost 上的端口互相连通，集群中的所有 Pod 只需要通过简单的路由，就能访问到彼此。

![](../picture/37.jpg)

但如果某个节点宕机会发生什么呢？ Pod 会终止，pod控制器内的 ReplicaSet 将创建新的 Pod，并且重新分配一个 IP，那么刚才创建的pod和pod之间的网络连接就会中断。
如何解决这个问题呢？ Kubernetes Service 就被提出来了，它是集群中提供相同功能的一组 Pod 的抽象表达。 当每个 Service 创建时，会被分配一个唯一的 虚拟IP 地址（也称为 clusterIP）。 这个 IP 地址与 Service 的生命周期绑定在一起，只要 Service 存在，它就不会改变。 可以配置 Pod 使它与 Service 进行通信，Pod 与 Service 的通信将被自动地负载均衡到该 Service 中的某些 Pod 上。

#### service如何绑定pod
```
apiVersion: v1
kind: Service
metadata:
  name: my-service
spec:
  selector:
    app.kubernetes.io/name: MyApp
  ports:
    - protocol: TCP
      port: 80
      targetPort: 9376
```
上面的例子，系统将创建一个名为 "my-service" 的、 服务类型默认为 ClusterIP 的 Service。 该 Service 指向带有标签 app.kubernetes.io/name: MyApp 的所有 Pod 的 TCP 端口 9376。
Kubernetes 为该服务分配一个 IP 地址（称为 “service IP”），供虚拟 IP 地址机制使用。 

此 Service 的控制器不断扫描与其选择算符匹配的 Pod 集合，然后对 Service 的 EndpointSlice 集合执行必要的更新。
注意，pod和service的创建顺序没有强制要求。

#### 如何获得service对应的service IP
在pod内部，Kubernetes 支持两种查找服务的主要模式：环境变量和 DNS。前者开箱即用，而后者则需要 CoreDNS 集群插件。

#### service IP是如何工作的
每当我们在k8s cluster中创建一个service，k8s cluster就会在K8s cluster IP的范围内为service分配一个cluster-ip，比如本文开始时提到的：
```
# kubectl get services
NAME           CLUSTER-IP      EXTERNAL-IP   PORT(S)     AGE
index-api      192.168.3.168   <none>        30080/TCP   18d
kubernetes     192.168.3.1     <none>        443/TCP     94d
my-nginx       192.168.3.179   <nodes>       80/TCP      90d
nginx-kit      192.168.3.196   <nodes>       80/TCP      12d
rbd-rest-api   192.168.3.22    <none>        8080/TCP    60d
```

这个cluster-ip只是一个虚拟的ip，并不真实绑定某个物理网络设备或虚拟网络设备，仅仅存在于iptables的规则中：
```
Chain PREROUTING (policy ACCEPT)
target         prot opt source               destination
KUBE-SERVICES  all  --  0.0.0.0/0            0.0.0.0/0            /* kubernetes service portals */
```
可以看到在PREROUTING环节，k8s设置了一个target: KUBE-SERVICES，这个链的source和destination都是0.0.0.0，表示所有的数据包都要经过这个链。
```
# iptables -t nat -nL|grep 192.168.3
Chain KUBE-SERVICES (2 references)
target                     prot opt source               destination
KUBE-SVC-XGLOHA7QRQ3V22RZ  tcp  --  0.0.0.0/0            192.168.3.182        /* kube-system/kubernetes-dashboard: cluster IP */ tcp dpt:80
KUBE-SVC-NPX46M4PTMTKRN6Y  tcp  --  0.0.0.0/0            192.168.3.1          /* default/kubernetes:https cluster IP */ tcp dpt:443
KUBE-SVC-AU252PRZZQGOERSG  tcp  --  0.0.0.0/0            192.168.3.22         /* default/rbd-rest-api: cluster IP */ tcp dpt:8080
KUBE-SVC-TCOU7JCQXEZGVUNU  udp  --  0.0.0.0/0            192.168.3.10         /* kube-system/kube-dns:dns cluster IP */ udp dpt:53
KUBE-SVC-BEPXDJBUHFCSYIC3  tcp  --  0.0.0.0/0            192.168.3.179        /* default/my-nginx: cluster IP */ tcp dpt:80
KUBE-SVC-UQG6736T32JE3S7H  tcp  --  0.0.0.0/0            192.168.3.196        /* default/nginx-kit: cluster IP */ tcp dpt:80
KUBE-SVC-ERIFXISQEP7F7OF4  tcp  --  0.0.0.0/0            192.168.3.10         /* kube-system/kube-dns:dns-tcp cluster IP */ tcp dpt:53
```
而KUBE-SERVICES下面又设置了许多target，一旦destination和dstport匹配，就会沿着chain进行处理。
比如：当我们在pod网络curl 192.168.3.22:8080时，匹配到下面的KUBE-SVC-AU252PRZZQGOERSG target：
```
KUBE-SVC-AU252PRZZQGOERSG  tcp  --  0.0.0.0/0            192.168.3.22         /* default/rbd-rest-api: cluster IP */ tcp dpt:8080
```

沿着target，我们看到”KUBE-SVC-AU252PRZZQGOERSG”对应的内容如下：
```
Chain KUBE-SVC-AU252PRZZQGOERSG (1 references)
target     prot opt source               destination
KUBE-SEP-I6L4LR53UYF7FORX  all  --  0.0.0.0/0            0.0.0.0/0            /* default/rbd-rest-api: */ statistic mode random probability 0.50000000000
KUBE-SEP-LBWOKUH4CUTN7XKH  all  --  0.0.0.0/0            0.0.0.0/0            /* default/rbd-rest-api: */
```
这里KUBE-SVC-AU252PRZZQGOERSG里有2个target，表示当前的service有2个实体pod来负责业务处理。
```
Chain KUBE-SEP-I6L4LR53UYF7FORX (1 references)
target     prot opt source               destination
KUBE-MARK-MASQ  all  --  172.16.99.6          0.0.0.0/0            /* default/rbd-rest-api: */
DNAT       tcp  --  0.0.0.0/0            0.0.0.0/0            /* default/rbd-rest-api: */ tcp to:172.16.99.6:8080

Chain KUBE-SEP-LBWOKUH4CUTN7XKH (1 references)
target     prot opt source               destination
KUBE-MARK-MASQ  all  --  172.16.99.7          0.0.0.0/0            /* default/rbd-rest-api: */
DNAT       tcp  --  0.0.0.0/0            0.0.0.0/0            /* default/rbd-rest-api: */ tcp to:172.16.99.7:8080
```
请求被按5：5开的比例分发（起到负载均衡的作用）到KUBE-SEP-I6L4LR53UYF7FORX 和KUBE-SEP-LBWOKUH4CUTN7XKH。
```
Chain KUBE-MARK-MASQ (17 references)
target     prot opt source               destination
MARK       all  --  0.0.0.0/0            0.0.0.0/0            MARK or 0x4000
```
而这两个chain的处理方式都是一样的，那就是先做mark，然后做dnat，将service ip改为service对应的Pod中的Pod IP，这样请求被实际传输到pod中处理了。