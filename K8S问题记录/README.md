### k8s问题排查

1. 容器状态CrashLoopBackOff

   - 状态原因：Kubernetes试图启动该Pod，但是过程中出现错误，导致容器启动失败或者正在被删除。

   - 查看日志

     - 先查看CrashLoop状态的pod.  `kubectl get pods`

       - 查看详细信息 `kubectl describe pod podName`,  查看字段 State  Laste State 观察各自的Reason是否是ERROR

       - 观察到如下 意思是启动容器后马上就完成了，然后终止，然后k8s一直在启动

         ```shell
         State:          Waiting
               Reason:       CrashLoopBackOff
             Last State:     Terminated
               Reason:       Completed
               Exit Code:    0
               Started:      Sun, 12 Jan 2020 13:38:10 +0800
               Finished:     Sun, 12 Jan 2020 13:38:10 +0800
         ```

         

   

2. k8s报错 No API token found for service account "default", retry after the token is automatically

  3.
  
  
  
   

3. pod出Terminating状态一直删除不了

###K8S操作

1. 基础命令

   获取pdo，node简单命令：kubectl get pods | kubectl get nodes

   - kubectl get services 获取服务

   - kubectl get rc 获取副本状态

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

2. 简单的创建一个应用

   - 创建应用  

     1. 命令行创建

        `kubectl run "server" --image=ginx:v3 --generator=run-pod/v1`

        --image 指定一个镜像

        --generator=run-pod/v1 创建一个 Replication*Controller* （没有删除rc，任何删除pod的操作会重拉pod）

     2. yaml文件创建

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

     3. 通过rc或者rs创建pod

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

   - 外部访问pod

     1. 暴露服务，外部能够访问pod需要通过服务对象公开，创建LoadBalancer类型，常规服务Clusterip

        `kubectl expose rc myserver --type=LoadBalancer --name myserver-http`

        此时在用 `kubectl get services` 结果，minikube没有loadBalancer服务，所有外部ip为空

     ```shell
     NAME            TYPE           CLUSTER-IP      EXTERNAL-IP   PORT(S)          AGE
     kubernetes      ClusterIP      10.96.0.1       <none>        443/TCP          10d
     myserver-http   LoadBalancer   10.96.238.208   <pending>     8080:32265/TCP   21m
     ```

     2. 将本地网络端口转发到pod端口

        请求本地8888能访问集群pod的8080端口，常用于测试

        `kubectl port-forward kubia-manual 8888:8080 `

   - 扩所容服务

     kubectl scale re kubia --replicas=3  针对kubia服务扩容

     

3. k8s的一些其他资源

   - DaemonSet 

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

   - Job资源

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

   - CronJob

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

     
