### 什么是Istio

istio是一个开源的实现了服务网格的平台，基于k8s平台，具有负载均衡、服务间认证、监控等功能，为业务应用服务。

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





