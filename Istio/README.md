###什么是Istio

istio是一个开源的实现了服务网格的平台，基于k8s平台，具有负载均衡、服务间认证、监控等功能，为业务应用服务。

### 安装Istio

- 通过istioctl 安装

  curl -L https://istio.io/downloadIstio | ISTIO_VERSION=1.4.3 sh 下载

  istioctl profile list 查看安装规则一般选择 demo，istioctl manifest apply --set profile=demo

  所有的组件都是安装在 namespace istio-system下

- 修改kiali

  `kubectl edit svc kiali -n istio-system`

  将type: clust 改为 type: NodePort

- 给kiali添加密码

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

  

  

  

  







