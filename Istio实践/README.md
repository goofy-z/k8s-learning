### 安装Istio
istio的安装是基于k8s平台的，本地测试可以使用minikube搭建k8s本平台，具体看我的minikube安装 https://github.com/qq1141000259/k8s-learning/tree/master/Minikube

- 通过istioctl 安装

  curl -L https://istio.io/downloadIstio | ISTIO_VERSION=1.5.1 sh 下载

  istioctl profile list 查看安装规则一般选择 demo，istioctl manifest apply --set profile=demo

  你能够看到很多服务组件都是安装在 namespace istio-system下

  查看安装的服务 `kubectl get svc -n istio-system `

### 安装kiali

- 修改kiali

  `kubectl edit svc kiali -n istio-system`

  将type: clustIP 改为 type: NodePort,这样更好测试访问

- 给kiali添加密码，其实就是给kilia service挂载一个Secret

  ```shell
  USERNAME=$(echo -n 'admin' | base64)
  PASSPHRASE=$(echo -n 'admin' | base64)
  NAMESPACE=istio-system
  kubectl create namespace $NAMESPACE
  cat <<EOF | kubectl apply -f -
  apiVersion: v1
  kind: Secret
  metadata:
    name: kiali
    namespace: $NAMESPACE
    labels:
      app: kiali
  type: Opaque
  data:
    username: $USERNAME
    passphrase: $PASSPHRASE
  EOF
  ```

- 查看kiali映射在宿主机的端口

  `k describe svc kiali -n istio-system` 找到 NodePort（这个端口号是变换的）

  ```shell
  NodePort:                 http-kiali  30717/TCP
  ```

  接下来访问kiali: http://本机ip:30717/ ，效果如下：

  ![image-20200416120749366](../resource/istio-kiali.png)

### 安装应用 bookinfo

这是一个 istio的应用范例

- 我们在默认的namespace下创建我们的应用, 给default添加标签

  `kubectl label namespace default istio-injection=enabled`

- 到istio的安装包下安装bookinfo的应用

  `kubectl apply -f samples/bookinfo/platform/kube/bookinfo.yaml`

  这些应用安装在default下：`kubectl get pods`

  ![image-20200416122402383](../resource/istio-bookinfo-pod.png)

- 安装bookinfo-gateway

  `kubectl apply -f samples/bookinfo/networking/bookinfo-gateway.yaml`

  查看安装： `kubectl get gateway`

  查看网关：`kubectl get svc istio-ingressgateway -n istio-system`

  ![image-20200416122907161](../resource/istio-bookinfo-gw.png)

  其中 80:30833对应的就是的外网访问端口

  访问bookinfo http://本机IP:30833/productpage

  ![企业微信截图_97105daf-a538-457f-822d-61896328783b](../resource/istio-bookinfo-productpage.png)

  

- 在kiali中你能看到每个应用的调用关系

  istio-kiali-bookinfo

  ![企业微信截图_ac5fbd20-7742-467b-a2e0-879880751cf7](../resource/istio-kiali-bookinfo.png)

### 扩展 bookinfo

在简单的搭建好一个服务网格的系统之后，我们接下来就主要关注于在这个网格里，是如何上线一个应用的

- 结构图

- 首先实现外部请求访问服务网格的应用，本质就是实现外部访问k8s，在上述例子中使用的其实是Istio部署的LoadBalancer服务

  执行`kubectl get svc -n istio-system -l istio=ingressgateway -o yaml`查看这个服务详情

  ```yaml
  apiVersion: v1
  items:
  - apiVersion: v1
    kind: Service
    metadata:
    	...
    spec:
      clusterIP: 10.96.121.156
      externalTrafficPolicy: Cluster
      ports:
      - name: status-port
        nodePort: 32362
        port: 15020
        protocol: TCP
        targetPort: 15020
      - name: http2
        nodePort: 30833
        port: 80
        protocol: TCP
        targetPort: 80
      ...
      selector:
        app: istio-ingressgateway
  ```

  在selector中，它指定了一个标签为`app=istio-ingressgateway`的pod，也就是说所有发送给这个loadblance服务的请求，都会被发送到这个pod，且对外的端口`30833`对应的是pod的端口`80`

  找到这个pod`kubectl get pods -l app=istio-ingressgateway -n istio-system`

  ```shell
  你好,欢迎使用kubectl
  NAME                                    READY   STATUS    RESTARTS   AGE
  istio-ingressgateway-649f9646d4-m2cz2   1/1     Running   0          5d2h
  ```

  而这个pod则是通过一个叫 `istio-ingressgateway`的deployment（具有k8s rc的所有特性，还能进行版本控制）进行管理

- 我们已经知道了外界与服务网格间通信是通过一个叫 istio-ingressgateway 的 LoadBlance 服务实现的，那从LB到我们的应用`productpage`又是怎么走的呢

  回顾之前，我们在namepspace为`default`下创建bookinfo应用时，也创建了一个`bookinfo-gateway`的gateway资源, 显而易见，这个bookiinfo-gateway就是控制着任何要进入我们的应用的请求

  查看`bookiinfo-gateway`:`kubectl get gw bookinfo-gateway -o yaml`

  ```yaml
  apiVersion: networking.istio.io/v1alpha3
  kind: Gateway
  metadata:
  	...
  spec:
    selector:
      istio: ingressgateway
    servers:
    - hosts:
      - '*'
      port:
        name: http
        number: 80
        protocol: HTTP
  ```

  整个定义内容的意思：所有符合标签 `istio=ingressgateway`的pod将去监听 80 端口，且接受来自所有主机的请求

  我们来看看是这个Gateway作用在哪个pod上，其实这里涉及到一个问题，我们实际去查看的时候，在namespace为 `default`下是没有标签为这个的pod，反而在istio-system下存在这个pod

  `kubectl get pods -l istio=ingressgateway -n istio-system` 

  ```shell
  你好,欢迎使用kubectl
  NAME                                    READY   STATUS    RESTARTS   AGE
  istio-ingressgateway-649f9646d4-m2cz2   1/1     Running   0          5d4h
  ```

  

  且这个pod就是我们上面说的LB服务转发的pod，也就是通过deployment创建生成的那个pod，且这个pod是跨命名空间的，也就说虽然bookinfo的`bookinfo-gateway`和我们的deployment `istio-ingressgateway`不在一个namespace，但在`default`下的`bookinfo-gateway`还是能够作用到`istio-ingressgateway`上。

  当我们没有启动bookinfo应用时，进入到这个`istio-ingressgateway`的pod，查看端口监听情况，是没有找到80端口被监听，但当我们启动了bookinfo的`bookinfo-gateway`就能很快发现多了一个监听端口 80。

  到此我们已经能够从k8s集群之外(也就是 istio 框架外)访问到我们的服务网格的网关`istio-ingressgateway`,通过网关又能找到我们应用网关`bookinfo-gateway`

- 现在我们知道了从外部请求到应用的网关`bookinfo-gateway`，那么又是如何进入到应用的呢

  还是通过创建`bookinfo-gateway`的yaml来分析，发现在创建gateway资源的同时还创建了`VirtualService` `samples/bookinfo/networking/bookinfo-gateway.yaml `
  
  ```yaml
  apiVersion: networking.istio.io/v1alpha3
  kind: VirtualService
  metadata:
    name: bookinfo
  spec:
    hosts:
    - "*"
    gateways:
    - bookinfo-gateway
    http:
    - match:
      - uri:
          exact: /productpage
      - uri:
          prefix: /static
      - uri:
          exact: /login
      - uri:
          exact: /logout
      - uri:
          prefix: /api/v1/products
      route:
      - destination:
          host: productpage
          port:
            number: 9080
  ```
  
  在istio框架中，虚拟服务(Virtual service)和目标规则(destination rule)，是流量路由功能的关键组成。其中spec.gateways的`gateways`为`bookinfo-gateway`,也就是说这个gateway收到的请求会匹配到http中的uri下，并且转发的目标地址为`productpage`服务下的9080端口，我们可以看下这个`productpage`服务
  
  ```shell
  你好,欢迎使用kubectl
  NAME          TYPE        CLUSTER-IP     EXTERNAL-IP   PORT(S)    AGE
  productpage   ClusterIP   10.96.41.204   <none>        9080/TCP   4d22h
  ```
  
  再查看这个服务作用的pod为`app=productpage`
  
  ```
  你好,欢迎使用kubectl
  NAME                              READY   STATUS    RESTARTS   AGE
  productpage-v1-596598f447-wkvxj   2/2     Running   0          5d1h
  ```
  
  如果你用`kubectl exec -it productpage-v1-596598f447-wkvxj /bin/bash`进入，你就会发现这个productpage其实是一个Flask（python的web框架）的应用，代码里用`requests`发起请求去访问另外三个服务
  
  `ratings, reviews, details`,通过ClushIP进行容器间通信
  
  在pod内进入python命令行模式我们进行测试
  
  ```python
  >>> res = requests.get("http://reviews:9080/reviews/1")
  >>> res
  <Response [200]>
  >>> res.text
  '{"id": "1","reviews": [{  "reviewer": "Reviewer1",  "text": "An extremely entertaining play by Shakespeare. The slapstick humour is refreshing!", "rating": {"stars": 5, "color": "black"}},{  "reviewer": "Reviewer2",  "text": "Absolutely fun and entertaining. The play lacks thematic depth when compared to other plays by Shakespeare.", "rating": {"stars": 4, "color": "black"}}]}'
  ```
  
  再回到宿主机，我们来查看另外三个服务
  
  ```shell
  kubectl get svc
  你好,欢迎使用kubectl
  NAME          TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)    AGE
  details       ClusterIP   10.96.132.128   <none>        9080/TCP   5d1h
  kubernetes    ClusterIP   10.96.0.1       <none>        443/TCP    5d17h
  productpage   ClusterIP   10.96.41.204    <none>        9080/TCP   4d23h
  ratings       ClusterIP   10.96.95.141    <none>        9080/TCP   5d1h
  reviews       ClusterIP   10.96.235.173   <none>        9080/TCP   5d1h
  ```
  
  其中kubernetes 是minikube所生成的，在看它们作用的pod
  
  ```shell
  kubectl get pods
  你好,欢迎使用kubectl
  NAME                              READY   STATUS    RESTARTS   AGE
  details-v1-78d78fbddf-p98fn       2/2     Running   0          5d1h
  productpage-v1-596598f447-wkvxj   2/2     Running   0          5d1h
  ratings-v1-6c9dbf6b45-v8nqg       2/2     Running   0          5d1h
  reviews-v1-7bb8ffd9b6-hsdb7       2/2     Running   0          5d1h
  reviews-v2-d7d75fff8-2hw7m        2/2     Running   0          5d1h
  reviews-v3-68964bc4c8-6hllk       2/2     Running   0          5d1h
  ```
  
  我们会发现这个reviews居然有三个pod，还分成了1，2，3版本。这里其实是对review这个应用起了三个deployment来管理三个版本的pod
  
  ```shell
  kubek get deployment
  你好,欢迎使用kubectl
  NAME             READY   UP-TO-DATE   AVAILABLE   AGE
  details-v1       1/1     1            1           5d1h
  productpage-v1   1/1     1            1           5d1h
  ratings-v1       1/1     1            1           5d1h
  reviews-v1       1/1     1            1           5d1h
  reviews-v2       1/1     1            1           5d1h
  reviews-v3       1/1     1            1           5d1h
  ```
  
  而这三个版本就是deployement资源的特征，在管理pod的时候又能进行版本控制，三个版本对应的效果
  
  V1: 在productpage页面不会显示评分信息
  
  V2: 在productpage页面显示1到5个黑色评分信息
  
  V3: 在productpage页面显示1到5个红色评分信息
  
  在默认的情况下，你对 reviews服务发起请求，将会被均匀的分配在这三个版本的应用上，你不断在http://9.134.114.175:30833/productpage 这个页面刷新观察评分就能看到这样的效果。
  
  但是，如果我想取消掉某个应用的请求，或分配权重，我们可以通过给reviews服务增加virtualService + DestinationRule来控制转发, `samples/bookinfo/networking/virtual-service-reviews-v2-v3.yaml`
  
  ```yaml
  apiVersion: networking.istio.io/v1alpha3
  kind: VirtualService
  metadata:
    name: reviews
  spec:
    hosts:
      - reviews
    http:
    - route:
      - destination:
          host: reviews
          subset: v2
        weight: 50
      - destination:
          host: reviews
          subset: v3
        weight: 50
  ```
  
  如你所见，我们在为 reviews这个host配置了权重为50的转发规则，它们的subset 分别是v2和v3,这个subset将会指引我们去DestinationRule中找到对应的pod
  
  `samples/bookinfo/networking/destination-rule-reviews.yaml`
  
  ```yaml
  apiVersion: networking.istio.io/v1alpha3
  kind: DestinationRule
  metadata:
    name: reviews
  spec:
    host: reviews
    trafficPolicy:
      loadBalancer:
        simple: RANDOM
    subsets:
    - name: v1
      labels:
        version: v1
    - name: v2
      labels:
        version: v2
    - name: v3
      labels:
        version: v3
  ```
  
  我们能够看到对于subsets下分别分配了不同的label属性--version，其实这些就是我们的`reviews`服务下的所有pod
  
  ```
  kubectl get pods -l app=reviews --show-labels
  你好,欢迎使用kubectl
  NAME                          READY   STATUS    RESTARTS   AGE    LABELS
  reviews-v1-7bb8ffd9b6-hsdb7   2/2     Running   0          5d1h   app=reviews,pod-template-hash=7bb8ffd9b6,security.istio.io/tlsMode=istio,version=v1
  reviews-v2-d7d75fff8-2hw7m    2/2     Running   0          5d1h   app=reviews,pod-template-hash=d7d75fff8,security.istio.io/tlsMode=istio,version=v2
  reviews-v3-68964bc4c8-6hllk   2/2     Running   0          5d1h   app=reviews,pod-template-hash=68964bc4c8,security.istio.io/tlsMode=istio,version=v3
  ```
  
  接下来，我们分别应用这两个配置,必须两个都要
  
  `k apply -f destination-rule-reviews.yaml` 和 `kubectl apply -f virtual-service-reviews-v2-v3.yaml`
  
  ```shell
  k get virtual-service
  你好,欢迎使用kubectl
  NAME       GATEWAYS             HOSTS       AGE
  bookinfo   [bookinfo-gateway]   [*]         5d1h
  reviews                         [reviews]   16s
  
  k get destinationrule
  你好,欢迎使用kubectl
  NAME      HOST      AGE
  reviews   reviews   5s
  ```
  
  最终你就能看到你想要的效果了
  
  
  
  
  
  
  
  



