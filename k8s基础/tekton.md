## 1. 什么是tekton

Tekton 是一个功能强大且灵活的Kubernetes 原生开源框架，用于创建持续集成和交付（CI/CD）系统。 特点是kubernetes原生，什么是kubernetes原生呢？简单的理解，就是all in kubernetes，所以用容器化的方式构建容器镜像是必然，另外，基于kubernetes CRD定义的pipeline流水线也是Tekton最重要的特征。

### 1.1 Tekton提供的CRD

- Task： 任务模版，你可以在里面定义相应的steps来表示需要执行的步骤，每个step代表一个pod


###Tekton 升级

- v0.25  开始废弃 `PipelineRunCancelled`状态，改为`Cancelled`
- v0.17  增加QPS、ControllerThread、brush参数
  - DefaultThreadsPerController： 2
  - QPS： 5
  - DefaultBurst： 10
- v0.17  修改CRD资源api version 为apiextensions.k8s.io/v1（原来是v1beta1）

## 版本升级

获取各版本安装yaml：

tekton-operator: `https://github.com/tektoncd/pipeline/releases`

Dashbord: `https://github.com/tektoncd/dashboard/releases`

1. 删除后Apply（这个会干掉所有tekton的有关资源）

   `kubectl delete -f xxxx`

   `kubectl apply -f xxxx`

2. Apply升级（按一个历史issue的说法，0.11+以上的版本只要apply新的yaml就好）

   `kubectl apply -f xxxx`

建于官方提供的镜像都在`gcr.io`上，如果访问不到我们可以在dockerhub上找到对应的版本。

```
https://hub.docker.com/u/tektondev
```

目前我已经打包好上传到了daocloud.io

- controller

   `daocloud.io/atsctoo/tekton-controller:v0.26.0`

- entrypoint 

   `daocloud.io/atsctoo/tekton-entrypoint:v0.26.0`

- webhook

   `daocloud.io/atsctoo/tekton-webhook:v0.26.0`

- dashbord

  ```
  daocloud.io/atsctoo/tekton-dashboard:v0.18.1
  ```

之后在升级的yaml中替换对应的镜像



一些问题：

1. Apply升级

   apply yaml时可能，创建configmap失败，错误如下：

   ```
   ame: "config-leader-election", Namespace: "tekton-pipelines"
   for: "tekton_v_0_26_0.yaml": Internal error occurred: failed calling webhook "config.webhook.pipeline.tekton.dev": Post https://tekton-pipelines-webhook.tekton-pipelines.svc:443/config-validation?timeout=30s: dial tcp 172.31.205.153:443: connect: connection refused
   ```

   其实是webhook还没起来，这个configmap创立要去请求webhook服务。









