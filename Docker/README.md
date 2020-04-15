### 安装Docker

二进制安装

- 下载二进制安装包 https://download.docker.com/linux/static/stable/x86_64/

- 解压文件到 /usr/bin/下

- 配置NAT

  - brctl addbr docker0
  - ip addr add 192.168.10.1/24 dev docker0
  - ip link set dev docker0 up
  - ip addr show docker0

- 创建systemd文件，目录 /usr/lib/systemd/system/docker.service

  ```shell
  [Unit]
  Description=Docker Application Container Engine
  Documentation=https://docs.docker.com
  BindsTo=containerd.service
  After=network-online.target firewalld.service containerd.service
  Wants=network-online.target
  Requires=docker.socket
  
  [Service]
  Type=notify
  # the default is not to use systemd for cgroups because the delegate issues still
  # exists and systemd currently does not support the cgroup feature set required
  # for containers run by docker
  ExecStart=/usr/bin/dockerd -H fd:// --containerd=/run/containerd/containerd.sock
  ExecReload=/bin/kill -s HUP $MAINPID
  TimeoutSec=0
  RestartSec=2
  Restart=always
  
  # Note that StartLimit* options were moved from "Service" to "Unit" in systemd 229.
  # Both the old, and new location are accepted by systemd 229 and up, so using the old location
  # to make them work for either version of systemd.
  StartLimitBurst=3
  
  # Note that StartLimitInterval was renamed to StartLimitIntervalSec in systemd 230.
  # Both the old, and new name are accepted by systemd 230 and up, so using the old name to make
  # this option work for either version of systemd.
  StartLimitInterval=60s
  
  # Having non-zero Limit*s causes performance problems due to accounting overhead
  # in the kernel. We recommend using cgroups to do container-local accounting.
  LimitNOFILE=infinity
  LimitNPROC=infinity
  LimitCORE=infinity
  
  # Comment TasksMax if your systemd version does not supports it.
  # Only systemd 226 and above support this option.
  TasksMax=infinity
  
  # set delegate yes so that systemd does not reset the cgroups of docker containers
  Delegate=yes
  
  # kill only the docker process, not all processes in the cgroup
  KillMode=process
  
  [Install]
  WantedBy=multi-user.target
  ```

- 重启systemd，启动docker  

  - systemctl daemon-reload

  - systemctl start docker 

- Hello world 

  Docker run hello-world 构造docker的hello world

- d d d

### Docker安装遇到的问题

1. cp 二进制文件到/usr/bin 目录下时出现

   `cp: cannot create regular file './runc': Text file busy`

   原因：这个文件已经被进程占用，不允许修改，可以ps看看哪些进程占用了

2. docker run hello-world 启动容器时出现失败，再次用docker start 容器ID出现下面错误

   `x x x shim.sock: bind: address already in use: unknown`

   原因：根据提示可以判断是ip地址被占用，实际是在安装新版本前我安装了低版本的docker，导致我启动docker是server用的是低版本，但client用的高版本，`docker version` 替换掉低版本的二进制，将所有旧版本进程全部杀死，包括containerd、runc、shim，然后重启恢复了

   

3. 待续。。。

### Docker操作

1. 修改docker私有仓库

   - 修改docker配置文件

   ```json
   
   {
       "registry-mirrors": [
           "http://docker.oa.com:8080",
           "http://csighub.com"
       ],
       "insecure-registries" : [
           "hub.oa.com",
           "docker.oa.com:8080",
           "csighub.com",
           "bk.artifactory.oa.com:8080"
       ]
   }
   ```

   - 修改hosts文件

     /etc/hosts 下的文件 ip 域名

   - d d d

2. 基础操作

   登陆

   `docker login --username=xxxx 仓库域名`

   打包推送

   ​			首先需要打包成新tag `docker tag xxx:v3 csighub.com/qqq/xxx:v1`

   ​			然后push镜像 `docker push csighub.com/qqq/xxx:v1`

   启动容器：docker run --name newname -h hostname
   			-it 打开容器终端 并 进入容器
   			-h 指定主机名
   			-v 使用数据卷
   			-rm 容器退出时就能够自动清理容器内部的文件系统删除容器
   停止容器：`docker stop 容器ID` 这种是向容器发送停止命令等待返回 还有kill直接杀死容器进程
   查看容器：`docker ps -a`
   进入容器：`docker attach 容器ID`
   删除容器：`docker rm 容器ID`
   映射端口：`docker run -d -p 91:80 nginx`  容器端口为80 主机端口为91 //P 为随机映射端口
   查看容器存储目录：`docker inspect -f {{.Volumes}} 容器名`
   容器运行脚本：`docker run -it --name test5 --volumes-from test1 centos /bin/bash ./sh/test.sh` 
   			//必须制定shell编译器 /bin/bash 
   			//--volumes-from test1 指定附加卷容器 共享该附加卷的文件
   增加附加卷：docker run --name test1 -v /root/sh:/sh centos /bin/bash 
   			//-v /root/sh:/sh 指定附加卷，映射主机目录/root/sh 为容器目录sh 
   在已经运行的容器中运行新进程： docker exec -it 容器名 CMD
   			例：`docker exec -d competent_kilby /bin/sh -c "sleep 20"` 在容器中执行sleep 20 脚本
   查看容器中得log：
   			`docker logs -t -f --tail 0 test7`  -t显示时间 -f刷新  --tail 0 显示最新的1一条

   构建镜像:

   ​			 `docker build /home/admin/application/ -t riji:product`

   

3. 待续。。。