### 安装Minikube

1. 安装kubectl

   -  下载最新稳定版本 

     ```shell
     curl -LO https://storage.googleapis.com/kubernetes-release/release/`curl -s https://storage.googleapis.com/kubernetes-release/release/stable.txt`/bin/linux/amd64/kubectl
     ```

   - 修改可执行文件权限 `chmod +x kubectl`

   - 将可执行文件放到用户bin `mv ./kubectl /usr/local/bin/kubectl`

   - 检查版本 `kubectl version`

2. 安装minkube

   - 下载最新版Minikube 

     ```shell
     curl -Lo minikube https://storage.googleapis.com/minikube/releases/latest/minikube-linux-amd64 \
     && chmod +x minikube
     ```

   - 移动minikube到user下的bin目录 install minikube /usr/local/bin

   - 启动minikube minikube start --vm-driver=none 不用虚拟机启动，需要本机上自带docker

   - Minikube delete 卸载minikube

     

3. 遇到问题

   - journalctl -f查看发现报错 

     ```shell
     dns.go  Nameserver limits were exceeded, some nameservers have been omitted, the applied nameserver line is: 192.96.0.10 10.2.5.1 10.2.5.2
     ```

     原因 [kubelet](https://github.com/kubernetes/kubernetes/tree/master/pkg/kubelet)/[network](https://github.com/kubernetes/kubernetes/tree/master/pkg/kubelet/network)/[dns](https://github.com/kubernetes/kubernetes/tree/master/pkg/kubelet/network/dns)/dns.go代码里去读resolv.conf文件，但是里面的nameserver不能超过3个，这个是kubernetes/pkg/apis/core/validation定义了`MaxDNSNameservers = 3`的常量

     解决方法，在kubelet的配置文件(/var/lib/kubelet/config.yaml)里指定 resolv文件地址`resolvConf: /etc/kubectl-resolve.conf`

     

   - 使用kubectl get pods 报错

     ```shell
     Unable to connect to the server: unexpected EOF
     ```

     查看kubectl config view 发现 

     ```yaml
     clusters:
     - cluster:
         certificate-authority: /root/.minikube/ca.crt
         server: https://9.134.114.175:8443
     ```

     这个ip是我的本机ip，修改/root/.kube/config里server换成localhost 发现结果

     ```
     No resources found in default namespace.
     ```

     这个是正常的，因为我一个pod都没起

   - 

4. 改写kubectl的代码

   - 先拉去k8s的代码 go get gitHub.com/kubernets/kubernets

   - 改变最外层的目录名字 k8s.io 

   - 修改/k8s.io/kubernetes/cmd/kubectl/kubectl.go文件后重新编译

   - ```shell
     KUBE_BUILD_PLATFORMS=linux/amd64 make WHAT=cmd/kubectl
     ```

     - 查看kubectl的配置 `kubectl config view` 使用的配置文件 /root/.kube/config 

   - 

5. 