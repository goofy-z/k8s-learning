

至少两核cpu 4g内存

###安装docker

- 卸载docker

  ```
  apt-get remove docker docker-engine docker.io 
  ```

- 安装https依赖

  ```
  sudo apt-get install \
      apt-transport-https \
      ca-certificates \
      curl \
      software-properties-common
  ```

  

- 添加docker的GPG key

  ```
  curl -fsSL https://mirrors.aliyun.com/docker-ce/linux/ubuntu/gpg | sudo apt-key add -
  ```

- 设置docker镜像源

  ```
  add-apt-repository \
     "deb [arch=amd64] https://mirrors.aliyun.com/docker-ce/linux/ubuntu \
     $(lsb_release -cs) \
     stable"
  ```

- 查看docker-ce的版本

  ```
  apt-cache madison docker-ce (v1.18需要19.3以上docker)
  ```

- 安装指定版本docker

  ```
  apt install -y docker-ce=5:19.03.15~3-0~ubuntu-bionic
  ```

  

- 设置自启动

  ```
  systemctl enable docker && sudo systemctl start docker
  ```

- 修改docker配置

  ```
  tee /etc/docker/daemon.json <<-'EOF'
  {
   "registry-mirrors": ["https://xxxxxxxx.mirror.aliyuncs.com"],
   "iptables": false,
   "ip-masq": false,
   "storage-driver": "overlay2",
   "graph": "/home/lk/docker"
  }
  EOF
  systemctl restart docker
  ```

###安装k8

- 创建source文件

  ```
  apt-get update && sudo apt-get install -y apt-transport-https curl
  curl -s https://mirrors.aliyun.com/kubernetes/apt/doc/apt-key.gpg | sudo apt-key add -
  tee /etc/apt/sources.list.d/kubernetes.list <<-'EOF'
  deb https://mirrors.aliyun.com/kubernetes/apt kubernetes-xenial main
  EOF
  apt-get update
  ```

- 安装kubeadm、kubectl、kubelet

  ```
  apt-get install -y kubelet=1.18.20-00 kubeadm=1.18.20-00 kubectl=1.18.20-00
  ```

- 自启动kubelet

  ```
  systemctl enable kubelet && sudo systemctl start kubelet
  ```

- 初始化镜像

  ```
  kubeadm config images list --kubernetes-version=v1.18.20
  ```

- 安装镜像(阿里云镜像)

  ```
  cat > kubeadm_config_images_list.sh<<EOF
  #!/bin/bash
  # Script For Quick Pull K8S Docker Images
  # by hualinux
   
  KUBE_VERSION=v1.18.20
  PAUSE_VERSION=3.2
  CORE_DNS_VERSION=1.6.7
  ETCD_VERSION=3.4.3-0
   
  # pull kubernetes images from hub.docker.com
  docker pull kubeimage/kube-proxy-amd64:\$KUBE_VERSION
  docker pull kubeimage/kube-controller-manager-amd64:\$KUBE_VERSION
  docker pull kubeimage/kube-apiserver-amd64:\$KUBE_VERSION
  docker pull kubeimage/kube-scheduler-amd64:\$KUBE_VERSION
  # pull aliyuncs mirror docker images
  docker pull registry.cn-hangzhou.aliyuncs.com/google_containers/pause:\$PAUSE_VERSION
  docker pull registry.cn-hangzhou.aliyuncs.com/google_containers/coredns:\$CORE_DNS_VERSION
  docker pull registry.cn-hangzhou.aliyuncs.com/google_containers/etcd:\$ETCD_VERSION
   
  # retag to k8s.gcr.io prefix
  docker tag kubeimage/kube-proxy-amd64:\$KUBE_VERSION  k8s.gcr.io/kube-proxy:\$KUBE_VERSION
  docker tag kubeimage/kube-controller-manager-amd64:\$KUBE_VERSION k8s.gcr.io/kube-controller-manager:\$KUBE_VERSION
  docker tag kubeimage/kube-apiserver-amd64:\$KUBE_VERSION k8s.gcr.io/kube-apiserver:\$KUBE_VERSION
  docker tag kubeimage/kube-scheduler-amd64:\$KUBE_VERSION k8s.gcr.io/kube-scheduler:\$KUBE_VERSION
  docker tag registry.cn-hangzhou.aliyuncs.com/google_containers/pause:\$PAUSE_VERSION k8s.gcr.io/pause:\$PAUSE_VERSION
  docker tag registry.cn-hangzhou.aliyuncs.com/google_containers/coredns:\$CORE_DNS_VERSION k8s.gcr.io/coredns:\$CORE_DNS_VERSION
  docker tag registry.cn-hangzhou.aliyuncs.com/google_containers/etcd:\$ETCD_VERSION k8s.gcr.io/etcd:\$ETCD_VERSION
   
  # untag origin tag, the images won't be delete.
  docker rmi kubeimage/kube-proxy-amd64:\$KUBE_VERSION
  docker rmi kubeimage/kube-controller-manager-amd64:\$KUBE_VERSION
  docker rmi kubeimage/kube-apiserver-amd64:\$KUBE_VERSION
  docker rmi kubeimage/kube-scheduler-amd64:\$KUBE_VERSION
  docker rmi registry.cn-hangzhou.aliyuncs.com/google_containers/pause:\$PAUSE_VERSION
  docker rmi registry.cn-hangzhou.aliyuncs.com/google_containers/coredns:\$CORE_DNS_VERSION
  docker rmi registry.cn-hangzhou.aliyuncs.com/google_containers/etcd:\$ETCD_VERSION
   
  EOF
  #修改权限，执行
  chmod +x kubeadm_config_images_list.sh
  sh kubeadm_config_images_list.sh
  ```

  

- 初始化kubeadm

  calico需要这个网段

  ```
  kubeadm init --pod-network-cidr=192.168.0.0/16 --kubernetes-version=v1.18.20
  ```

- 复制kube config

  ```
  mkdir -p $HOME/.kube && cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
  ```

- 当前节点被污了

  ```
  kubectl describe node xxx |grep Taint
  ```

  ```
  kubectl taint nodes --all node-role.kubernetes.io/master-
  ```

###安装calico

相关链接：`https://docs.projectcalico.org/getting-started/kubernetes/quickstart`

operator的方式

```
kubectl create -f https://docs.projectcalico.org/manifests/tigera-operator.yaml
kubectl create -f https://docs.projectcalico.org/manifests/custom-resources.yaml
watch kubectl get pods -n calico-system
```





