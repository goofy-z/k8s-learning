[toc]



### k8s问题排查

####容器状态CrashLoopBackOff

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


#### 容器状态Init:Error

内存不够。

集群不能apply文件

timeout 分析是webhook连接不通，检查svc和endpoint还有对应的pod都是正常的

```
for tekton.dev/v1alpha1, Kind=Task failed: Post https://tekton-pipeline-webhook.tekton-pipeline.svc:443/resource-conversion?timeout=30s: dial tcp 172.31.73.4:443: connect: connection timed out
```

原因：

Kube-proxy有一个副本挂了，下面是一个svc的分析文档

```
https://kubernetes.io/zh/docs/tasks/debug-application-cluster/debug-service/
```





2. k8s报错 No API token found for service account "default", retry after the token is automatically

  3.
  
  
  
   

3. pod出Terminating状态一直删除不了

4. 镜像被删除

   k8存在镜像回收机制，当磁盘压力过大，就会去删除镜像

   查看kube组件日志

   `journalctl -u kubelet`

5. 批量删除驱逐的pod

   ```shell
   for ns in `kubectl get ns|awk '{print $1}'`; do echo "$ns clean" && for i in `kubectl -n $ns get pods |grep Evicted|awk '{print $1}'`;do  kubectl -n $ns delete pod $i ;done ;done
   ```

6. 批量删除pr

   ```
   kubectl get pr -n kongtianbei|awk -F ' ' '{print $1}'
   ```

   ```
   kubectl get prs -n kongtianbei|awk -F ' ' '{print $1}'|xargs -I pr_name kubectl delete pr pr_name -n kongtianbei
   ```

   

7. service nodeport一台主机访问不通

   首先检查kube-proxy, 查看对应端口的规则

   ```
   iptables-save|grep 37031
   ```

   

8. 容器资源不足，发现一些pod报错`CreateContainerConfigError`

   这种状态pod还是占用着资源

   

