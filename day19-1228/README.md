### 容器发展史

- 进程容器内核补丁Cgroup(2006)

  Cgroup可以对cpu内存进行分组，从而隔离资源 

- LXC (2008) 

  LXC(LinuxContainer)容器发布，这是一种内核虚拟化技术，可以提供轻量级的 虚拟化，以便隔离进程和资源。LXC是Docker最初使用的具体内核功能实现 

- Docker (2013)

  组合LXC，Union File System和cgroups等Linux技术创建容器化 

  开发了一套build，ship，run开发流程。 

- Kubernetes (2014) 

  google发布了kubernetes

- OCI (2015) 

  制定并维护容器镜像格式和容器运行时的正式 规范，以便在不同的操作系统和平台之间移植 

- CNCF (2015) 

  容器编排开源组织，发布k8s1.0

- 容器厂商支持k8s

  包括 docker Mesos

- Kubernetes从CNCF毕业 (2018) 

  意味着该项目已经成熟了

### 容器概念

容器是一种打包应用的方式，可以帮你打包应用中的所有软件和软件所依赖的环境，并且可以实现跨平台部署 

优势：

	1. 更加有效的利用系统资源
 	2. 更快速的启动服务
 	3. 从开发到上线完全一致的运行环境
 	4. 持续交付和部署
 	5. 更加容易迁移，维护

### Docker是什么

容器是一种技术的统称，Docker只是实现了容器的生命周期管理，而真正启动容器的是runc（容器运行时），它是根据OCI定制的标准来控制容器。在runc上一层是Containerd，作为底层管理容器的标准，调用runc来处理容器。然后再之上就是Docker，Mesos等容器平台，总的来说就是我们看到容器技术分为两部分 最上层：容器平台 docker，Mesos； 中层：各种容器运行时: runC（docker 使用Containerd来管理runC） 、kata container（实现容器内核隔离等，都是需要遵守k8s的OCR 容器运行时标准接口，这样才能被接入

### Docker 命令

- 停止容器：docker stop 容器ID 这种是向容器发送停止命令等待返回 还有kill直接杀死容器进程
- 查看容器：docker ps -a
- 进入容器：docker attach 容器ID | docker exec -it 容器ID bash
- 删除容器：docker rm 容器ID
- 映射端口：docker run -d -p 91:80 nginx  容器端口为80 主机端口为91 //P 为随机映射端口
- 查看容器存储目录：docker inspect -f "{{.State.Pid}}" 容器ID
- 查看容器日志：docker logs 容器ID





