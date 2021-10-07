[toc]



## K8S资源

### 1. 指令

#### 本地连接远程apiserver

使用iptables的NAT，将集群端口转到api端口 18081

```shell
iptables -t nat -I PREROUTING -d masterip -p tcp --dport 28081 -j DNAT --to 127.0.0.1:18081
```

本地的/root/.kube/config对应内容

```yaml
apiVersion: v1
clusters:
- cluster:
    server: 127.0.0.1:18081
  name: local
contexts:
- context:
    cluster: local
    user: ""
  name: local
current-context: local
kind: Config
preferences: {}
users: []
```

获取证书内容base64编码

client-certificate-data

cat "证书文件" | base64

#### 常见指令

- kubectl get services 获取服务

- kubectl get deployment（deploy） 获取副本状态

- Kubect get pods -o wide 获取pod详情包含节点

- kubectl get po --show-labels  查看pod标签

- 删除pod或资源

  kubectl delete pod xxxx  --grace-period=0 --force 强制删除

- 获取标签

  1. 根据标签筛选 `kubectl get pod -l create_man=ginxzheng`
  2. 给节点或pod新增标签status `kubectl label node minikube status=true`(修改加上 --overwrite=true)
  3. 列出某一列标签 `kubectl get nodes -L status`

- namespace有关

  namespace 是在标签之上 再将容器进行划分，每个ns下都有着各自的容器标签

  1. 查看ns.

     Kubectl get ns 

  2. 可以使用yaml创建ns

     ```yaml
     apiVersion: v1
     kind: Namespace
     metadata:
       name: my-namespace
     ```

  3. 切换ns，上下文

     查看当前的上下文: 从配置中 `kubectl config view`获取上下文 

     切换当前namespace `kubectl config set-context minikube --namespace xxxx`

     删除namespace `kubectl delete ns xxxx`

### 2. 概念类 

#### 2.1 ServiceAccount（sa）

Kubernetes 分为用户账户和服务账户，用户账户是针对人而言的，服务账户是针对运行在 pod 中的进程而言的，服务账户是 namespace 隔离的。

#### 2.2 简单的创建一个应用

#####2.2.1 命令行创建

`kubectl run "server" --image=ginx:v3 --generator=run-pod/v1`

--image 指定一个镜像

--generator=run-pod/v1 创建一个 Replication*Controller* （没有删除rc，任何删除pod的操作会重拉pod）

#####2.2.2 yaml文件创建

kubectl create -f xxx.yaml

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: ginx
  labels:   # pod标签
    create_man: ginxzheng
    env: pod
spec:
  nodeSelector:
    gpu: true
  containers:
  - image: ginx:v3
    name: ginx-c1
    ports:
    - containerPort: 8089
      protocol: TCP
```

通过rc或者rs创建pod

replicationController 作为一种k8s资源，可以保证pod持续运行，需要将pod加入或移除rc则只需要更改pod

的标签，replicaSet 基本和rc一致，知识标签选择器更强大

包括三部分

- label selector pod标签选择其，用于确定rc的作用域
- replica count 副本个数，pod的数量
- pod template pod的模版

例子:

```yaml
apiVersion: v1  # 如果是rs 则这为 apps/v1
kind: ReplicationController  # rs 则为 ReplicaSet
metadata:
  name: kubia
spec:
  replicas: 3
  selector: 
    app: kubia  # 选择标签 app等于kubia的pod
  template:
    metadata:
      labels:
        app: kubia
    spec:
      containers:
      - name: kubia
        image: luksa/kubia
        ports:
        - containerPort: 8080
```

对于使用rs的负载selector,可以这么写:

```yaml
selector:
	matchExpressions:
	- key: app
    operator: In
    values:
    - kubia  # 标签值必须为kubia
```



修改rc `kubectl edit rc kubia` 这将会在默认编辑器里打开文件，修改模版只作用于后面生成的pod

对rc进行扩所容 

- 扩缩容 `kubectl scale rc kubia --replicas=10` 修改数字实现扩所容 实际是在对rc的声明文件进行修改
- 直接修改定义文件 

删除rc或是rs默认是把所关联的pod删除 `kubectl delete rc kubia --cascade=false` 以上为不删除pod

#####2.2.3 外部访问pod

1. 暴露服务，外部能够访问pod需要通过服务对象公开，创建LoadBalancer类型，原理也就是在nodeport上做了一层，把流量分配到每一个节点，然后节点上的pod通过Cluster IP进行通信。

    

   `kubectl expose rc myserver --type=LoadBalancer --name myserver-http`

   此时在用 `kubectl get services` 结果，minikube没有loadBalancer服务，所有外部ip为空

```shell
NAME            TYPE           CLUSTER-IP      EXTERNAL-IP   PORT(S)          AGE
kubernetes      ClusterIP      10.96.0.1       <none>        443/TCP          10d
myserver-http   LoadBalancer   10.96.238.208   <pending>     8080:32265/TCP   21m
```

2. 将本地端口映射到pod端口

   请求本地8888能访问服务kubia-manual 的pod的8080端口，常用于测试

   `kubectl port-forward kubia-manual 8888:8080 `

- 扩所容服务

  kubectl scale re kubia --replicas=3  针对kubia服务扩容

  

### 3. k8s资源

#### 单位进制

`1Mi=1024*1024；1M=1000*1000`

####Service

创建了Service后，会为每一个关联的pod创建endpoint，通过`readniessProbe`检查可以在服务断开后马上解绑pod和endpoint。

#### ClusterIP 实现原理

基于kube-proxy+iptables共同实现，当创建Service时，kube-proxy通过informer感知到Service的添加，之后便会创建一组iptables规则（iptables-save）就能看出来。kube-proxy支持IPVS模式，该模式会把每个pod的iptables规则记录到内核。

```
-A KUBE-SERVICES -d 10.0.1.175/32 -p tcp -m comment --comment "default/hostnames: cluster IP" -m tcp --dport 80 -j KUBE-SVC-NWV5X2332I4OT4T3
```

意思就是去往`10.0.1.175`的流量就会跳到`KUBE-SVC-NWV5X2332I4OT4T3`这条链，然后查看这条链能够找到如下规则：

```
-A KUBE-SVC-NWV5X2332I4OT4T3 -m comment --comment "default/hostnames:" -m statistic --mode random --probability 0.33332999982 -j KUBE-SEP-WNBA2IHDGP2BOBGZ
-A KUBE-SVC-NWV5X2332I4OT4T3 -m comment --comment "default/hostnames:" -m statistic --mode random --probability 0.50000000000 -j KUBE-SEP-X3P2623AGDH6CDF3
-A KUBE-SVC-NWV5X2332I4OT4T3 -m comment --comment "default/hostnames:" -j KUBE-SEP-57KPRZ3JQVENLNBR
```

其中--mode random代表随机链，分别指向的是Service代理的3个pod，probability代表该链被选择的规则的概率。

再次找到这些链的详情

```

-A KUBE-SEP-57KPRZ3JQVENLNBR -s 10.244.3.6/32 -m comment --comment "default/hostnames:" -j MARK --set-xmark 0x00004000/0x00004000
-A KUBE-SEP-57KPRZ3JQVENLNBR -p tcp -m comment --comment "default/hostnames:" -m tcp -j DNAT --to-destination 10.244.3.6:9376

-A KUBE-SEP-WNBA2IHDGP2BOBGZ -s 10.244.1.7/32 -m comment --comment "default/hostnames:" -j MARK --set-xmark 0x00004000/0x00004000
-A KUBE-SEP-WNBA2IHDGP2BOBGZ -p tcp -m comment --comment "default/hostnames:" -m tcp -j DNAT --to-destination 10.244.1.7:9376

-A KUBE-SEP-X3P2623AGDH6CDF3 -s 10.244.2.3/32 -m comment --comment "default/hostnames:" -j MARK --set-xmark 0x00004000/0x00004000
-A KUBE-SEP-X3P2623AGDH6CDF3 -p tcp -m comment --comment "default/hostnames:" -m tcp -j DNAT --to-destination 10.244.2.3:9376
```

`--to-destination`代表做了dnat，改变了目标IP地址。

#### Nodeport 实现原理

kube-proxy会在每一个宿主机上生成一条iptables规则：

```
-A KUBE-NODEPORTS -p tcp -m comment --comment "default/my-nginx: nodePort" -m tcp --dport 8080 -j KUBE-SVC-67RL4FN6JRUPOJYM
```

后续就和ClusterIP的流程基本一致，唯一不同，因为容器的流量会从集群出去，会做一次SNAT，将这个 IP 包的源地址替换成了这台宿主机上的 CNI 网桥地址，或者宿主机本身的 IP 地址（如果 CNI 网桥不存在的话）

```
-A KUBE-POSTROUTING -m comment --comment "kubernetes service traffic requiring SNAT" -m mark --mark 0x4000/0x4000 -j MASQUERADE
```

SNAT就是为了防止外部访问容器时原本是访问node1的IP，结果负载均衡到了node2，这个时候为了保证协议安全，必须将源地址NAT为node1IP或者node1的CNI网桥，然后返回给客户端。为了让容器知道是谁访问的，可以在svc里将`externalTrafficPolicy`设置为`local`,  请求就只会被代理到本地 endpoints 而不会被转发到其它节点。这样就保留了最初的源 IP 地址。如果没有本地 endpoints，发送到这个节点的数据包将会被丢弃。这样在应用到数据包的任何包处理规则下，你都能依赖这个正确的 source-ip 使数据包通过并到达 endpoint。

相关链接:`https://kubernetes.io/zh/docs/tutorials/services/source-ip/`

####LoadBalancer:

由云服务商提供。

####ExternalName:

在pod内访问 Service Name时返回的是设置的`externalName`, 其实是在 kube-dns 里为你添加了一条 CNAME 记录。

#### HeadlessService

clusterIP为None的Service，只负载均衡到被关联的pod

####在容器中访问service

<svc_name>.<namespace>.svc.cluster.local能访问clusterIP，如果是Headless就直接访问podIP。

#### 检查svc没有生效

查看svc的端口和endpoints，然后时kube-proxy, 再就是查看iptables。

如果 kube-proxy 一切正常，你就应该仔细查看宿主机上的 iptables 了。而一个 iptables 模式的 Service 对应的规则，

- KUBE-SERVICES 或者 KUBE-NODEPORTS 规则对应的 Service 的入口链，这个规则应该与 VIP 和 Service 端口一一对应；
- KUBE-SEP-(hash) 规则对应的 DNAT 链，这些规则应该与 Endpoints 一一对应；
- KUBE-SVC-(hash) 规则对应的负载均衡链，这些规则的数目应该与 Endpoints 数目一致；如果是 NodePort 模式的话，还有 POSTROUTING 处的 SNAT 链。

还有一种典型问题，就是 Pod 没办法通过 Service 访问到自己在默认情况下，网桥设备是不允许一个数据包从一个端口进来后，再从这个端口发出去的



####Deployment

管理rs，rs管理pod，可以实现一键回滚，版本记录

yaml示例

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
    name: alpine
    labels:
        app: goofy-deployment
spec:
    # 期望pod数量
    replicas: 2
    # pod选择器
    selector:
        matchLabels:
            app: goofy-alpine
    # pod 模版
    template:
        metadata:
            labels: 
                app: goofy-alpine
        spec:
         containers:
            - name: alpine-app
           image: goofy123/goofy-alpine:v1.0   
```

一键升级版本controller

`kubectl set image deployment/alpine alpine-app=goofy123/goofy-alpine:v2`

pod中存在多个容器

`kubectl -n kongtianbei set image deployment/kaifang-work-f5fbf57-m8v9g api=harbor.dsp.local/dsp-system/kongtianbei-api:dce-125`

- **deployment/alpine**代表资源类型和资源名称

- **alpine-app**是我们的容器名称，后面就是镜像名称

一键回退版本

`kubectl rollout undo deployment/alpine`



#### DaemonSet 

DaemonSet将pod部署在所有节点，并且每个节点都只部署一个pod，当然也可以指定selector来部署某个节点

yaml示例

```yaml
 apiVersion: apps/v1
kind: DaemonSet
 metadata:
  name: ssd-monitor
 spec:
  selector:
 matchLabels:
      app: ssd-monitor
  template:
    metadata:
      labels:
        app: ssd-monitor
    spec:
      nodeSelector:
        disk: ssd
      containers:
      - name: main
        image: luksa/ssh-monitor
```

#### Job资源

运行单个任务的pod，并且不会像rc或者rs那样在任务完成退出后还重启容器

yaml示例

  ```yaml
     apiVersion: batch/v1
     kind: Job
     metadata:
       name: batch-job
     spec:
       completions: 5   # 顺序运行5个pod
       parallelism: 2   # 最大并行2个
       activeDeadlineSeconds: 100  # 最大pod执行时间
       template:
         metadata:
           labels:
             app: batch-job
         spec:
           restartPolicy: OnFailure  # job的重启策略
           containers:
           - name: main
             image: luksa/batch-job
  ```

     同时也可以使用`kubectl scale job xxx --replicas 3` 并行扩到3个

#### CronJob

定时任务, 定时执行的对象为Job， 查看`k get cronjobs`

```yaml
apiVersion: batch/v1beta1
kind: CronJob
metadata:
  name: crontab-job
spec:
  schedule: 30 18 * * *  # 每天18.30执行job
  jobTemplate:
    spec:
      template:
        metadata:
          labels:
            app: periodic-batch-job
        spec:
          restartPolicy: OnFailure
          containers:
          - name: main
            image: luksa/batch-job
```

####Deployment

##### Deploy直接管理rs，然后通过rs来管理pod

####PVC

mountPath可以挂载容器中不存在的耳机线目录

template.spec.Containers

```
volumeMounts:
  - mountPath: /uploads
  	name: atr-api-uploads-data
  - mountPath: /project
  	name: atr-api-project-data
```

template.spec.下

```
volumes:
- name: atr-api-uploads-data
	persistentVolumeClaim:
	claimName: atr-api-uploads-pvc
```

####StatefulSet

sts直接管理pod，每一个pod都是带有固定顺序的编号，然后通过`Headless Service`为这些固定名字的pod生成DNS记录，只要sts的controller能够保证pod的名字一致，那么通过访问Service的流量都会打到正确的pod。

绑定pvc时，sts会为每一个pod分配到不同的pv，这样就保证了每一个pod有不同pv。

##### mountPath

###k8s开发

####Dynamic Admission Control

当一个 Pod 或者任何一个 API 对象被提交给 APIServer 之后，总有一些“初始化”性质的工作需要在它们被 Kubernetes 项目正式处理之前进行。比如，自动为所有 Pod 加上某些标签（Labels），而这个“初始化”操作的实现叫做Admission，但是如果你现在想要添加一些自己的规则到 Admission Controller，就会比较困难。因为，这要求重新编译并重启 APIServer，显然，这种使用方法对 Istio 来说，影响太大了，所以，Kubernetes 项目为我们额外提供了一种“热插拔”式的 Admission 机制，它就是Dynamic Admission Control，也叫作：Initializer。

自定义控制器原理





###etcd

下载etcdctl

```
https://github.com/etcd-io/etcd/tree/main/etcdctl
```

etcdctl 访问资源：

```
./etcdctl --endpoints=https://172.31.0.3:12379 --cert="/etc/daocloud/dce/certs/etcd-client.crt" --key="/etc/daocloud/dce/certs/etcd-client.key" --cacert=/etc/daocloud/dce/certs/ca.crt  role list
```



### 调用ApiServer的Restful接口

```
curl -k -H "Content-Type:application/json" \
-H "Authorization:Bearer xxxx"
-X PUT --data-binary @ns.json https://127.0.0.1:16443/api/v1/tekton-pi^Cline/tekton-pipeline/finalize
```

1. 获取本地apiserver的端口，`netstat -tlnup`找到进程`apiserver`

2. 获取token方式： 在clusterRolebinding找到admin用户，然后找到对应的sa，在找到sa关联的secret

   ```
   echo -n "xxxda"|base64 -d
   ```

#### 删除Teminating状态的ns或资源

- kubectl patch crd/此处写CRD的名字 -p  '{"metadata":{"finalizers":[]}}' --type=merge

- 获取ns的json详情， 然后删除掉`spec`里的所有字段

  发起`PUT`接口`https://127.0.0.1:16443/api/v1/tekton-pipeline/tekton-pipeline/finalize`

### K8S调度原理

#### 资源限制

CPU和Memory的设置通过为pod设置resource字段来控制

```yaml
resources: 
	requests: 
		memory: "64Mi" 
		cpu: "250m" 
	limits: 
		memory: "128Mi" 
		cpu: "500m"
```

在调度时，`kube-scheduler`只会按照`requests`的值进程计算，而真正设置`Cgroups`限制时，kubelet会按照`limits`来设置。

kubelet会监听宿主机的资源状况，当出现资源压力时会出发pod驱逐（Eviction），这个在kubelet启动时设置。

- memory.available 可用内存
- nodefs.available 可用磁盘空间
- imagefs.available 镜像存储空间

#### QoS

QoS 划分的主要应用场景，是当宿主机资源紧张的时候，kubelet 对 Pod 进行 Eviction（即资源回收）时需要用到的，。

- Guaranteed

  当 Pod 里的每一个 Container 都同时设置了 requests 和 limits，并且 requests 和 limits 值相等。当pod使用量大于limits限制，或宿主机处于Memory Pressure 状态时，Guaranteed 的 Pod 才可能被选中进行 Eviction 操作

- Burstable

  至少有一个 Container 设置了 requests， 资源使用量已经超出了 requests 的 Pod。

- BestEffort

  Pod 既没有设置 requests，也没有设置 limits，达到Eviction条件。

此外当节点进入`MemoryPressure` 或者 `DiskPressure`的状态时，该节点会被打上污点。

####调度过程

Kubernetes 的调度器的核心，实际上就是两个相互独立的控制循环。

1. Informer Path

   启动一系列informer来watch etcd中的Pod、Node、Service等，当监听资源创建时就会将其加入到调度队列，同时也会更新调度器缓存（scheduler cache），其缓存了集群信息。

2. Scheduling Path

   负责pod调度的主循环，不断的从调度队列拿出pod，调用Predicates 算法进行“过滤”，得到一组 可调度的Node，之后调用 Priorities算法对node打分,完成后会更新pod的spec.nodeName字段，这个过程被称为**Bind**，这里只会更新scheduler cache里的Pod和Node信息，之后会异步香APIServer发起更新Pod请求。

   当APIServer将创建命令下发到kubelet时，后者会通过`Admit`的操作来验证该pod是否能够运行在该节点上，实际上是执行一组叫作 GeneralPredicates 的、最基本的调度算法，比如：“资源是否可用”“端口是否冲突”等再执行一遍，作为 kubelet 端的二次确认。

#### 调度的两种策略

**Predicates**（谓词）：

- GeneralPredicates：
  1. 针对pod的`resources`进行调度，常见有GPU调度：`alpha.kubernetes.io/nvidia-gpu: 2`。
  2. spec.nodeName检查，匹配调度节点名称。
  3. PodFitsHostPorts， 检查节点端口是否被占用。
  4. PodMatchNodeSelector，Pod 的 nodeSelector 或者 nodeAffinity 指定的节点，是否与待考察节点匹配。
  5. 

**Priorities**（优先级）

###K8S创建pod的详细流程

1. 由kubectl解析创建pod的yaml，发送创建pod请求到APIServer。
2. APIServer首先做权限认证，然后检查信息并把数据存储到ETCD里，创建deployment资源初始化。
3. kube-controller通过list-watch机制，检查发现新的deployment，将资源加入到内部工作队列，检查到资源没有关联pod和replicaset,然后创建rs资源，rs controller监听到rs创建事件后再创建pod资源。
4. scheduler 监听到pod创建事件，执行调度算法，将pod绑定到合适节点，然后告知APIServer更新pod的`spec.nodeName`
5. kubelet 每隔一段时间通过其所在节点的NodeName向APIServer拉取绑定到它的pod清单，并更新本地缓存。
6. kubelet创建容器，并向APIService上报状态。
7. Kub-proxy为新创建的pod注册动态DNS到CoreOS。为Service添加iptables/ipvs规则，用于服务发现和负载均衡。
8. deploy controller对比pod的当前状态和期望来修正状态。

组件介绍：

/bin/bash -c pip install -r requirements.txt;python torch_mnist.py

Apiserver:

- 提供了集群管理的REST API接口。
- 负责与其他组件交互。

Controller-manager

- Deployment controller
- Replicaset controller

Kube-scheduler

- 接收调度pod到合适的节点。
- 两种策略：预选（predict，nodeslect、亲和性、污点检查等）、优选（主要对前一步得到的节点打分）。

kubelet

- 定时检查节点、和节点上pod的信息，并上报apiserver。
- 容器镜像清理。

Kube-proxy

- 为k8s每个节点做网络代理，使用iptables或Ipvs来做流量转发，负责service网络和pod网络的通信，并实现service的负载均衡。



