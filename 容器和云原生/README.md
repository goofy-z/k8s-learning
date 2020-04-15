### 云原生(CloudNative)

- ###云原生的背景

  随着云计算平台的成熟，应用上云已经形成一种趋势，未来的应用大致分为两种**构建云的应用**和**基于云构建**

- ####什么是云原生

  最初是由2013年Pivotal公司的Matt Stine 提出，可以看成是一套构建和运行应用程序的技术体系或方法论，

  并指出云原生的架构的特征：12因素、微服务、自敏捷架构、基于API协作、扛脆弱性；随后在2017年改为

  4个特点：**DevOps**+**持续交付**+**微服务**+**容器**， 在2018年，CNCF( Cloud Native Computing Foundation)云原生基金会将**服务网格**（Service Mesh）和 **声明式API**加入进来

  所以可以这样理解：云原生 = **DevOps+持续交付(Continuous Delivery)+微服务(Micro Services)+敏捷基础设施(Agile Infrastructure)+12要素(The Twelve-Factor App)+服务网格（Service Mesh）+声明式API**

- ####云原生和传统应用的区别

  - **云原生更倡导敏捷，自动化，容错**，而传统应用则大多还处于原生的瀑布开发模型和人工运维阶段。

    |              | **传统应用**           | 云原生应用                                   |
    | ------------ | ---------------------- | -------------------------------------------- |
    | **开发语言** | C/C++、企业级Java编写  | Go、Node.js等新兴语言编写                    |
    | **依赖关系** | 单体服务耦合严重       | 微服务化，高内聚，低耦合                     |
    | **部署方式** | 对机器、网络资源强绑定 | 容器化，对网络和存储都没有这种限制           |
    | **运维方式** | 人肉部署、手工运维     | 自动化部署，支撑频繁变更，持续交付，蓝绿部署 |
    | **开发模式** | 瀑布式开发             | DevOps、持续集成、敏捷开发                   |
    | **扩展性**   | 运维手工扩容           | 动态扩缩容                                   |
    | **可用性**   | 故障后可能影响业务     | 无状态，面向失败编程                         |
    | **可靠性**   | 本地资源，故障率高     | 云资源，可靠性高                             |

- ####云原生-12要素

  1. Codebase基准代码
     One codebase tracked in revision control, many deploys 一份基准代码，多份部署

  2. Dependencies依赖
     Explicitly declare and isolate dependencies显式声明依赖关系
  3. Config配置
     Store config in the environment在环境中存储配置
  4. Backing services后端服务
     Treat backing services as attached resources把后端服务当作附加资源
  5. Build, release, run构建，发布， 运行
     Strictly separate build and run stages 严格分离构建和运行
  6. Processes进程
     Execute the app as one or more stateless processes以一个或多个无状态进程运行应用
  7. Port binding端口绑定
     Export services via port binding通过端口绑定提供服务
  8. Concurrency并发
     Scale out via the process model通过进程模型进行扩展
  9. Disposability可处理
     Maximize robustness with fast startup and graceful shutdown 快速启动和优雅终止可最大化健壮性
  10. Dev/prod parity开发环境与 线上环境等价
      Keep development, staging, and production as similar as possible 尽可能的保持开发，预发布，线上环境相同
  11. Logs日志
      Treat logs as event streams把日志当作事件流
  12. Admin processes管理进程
      Run admin/management tasks as one-off processes后台管理任务当作一次性进程运行

- #### 云原生-服务网格

  - 服务网格定义

    A service mesh is a dedicated infrastructure layer for making service-to-service communication safe, fast, and reliable.

    服务网格是一个专注解决服务和服务之间连接的安全，快速和可靠的基础设施层，既是作为云原生技术下的一种网络模型，也是提供给Devops的一种解决方案

    由一系列的轻量级的网络代理组成，它们与应用程序部署在一起，但是应用程序不需要知道它们的存在。

    作为开发者，我们往往在实现一个业务服务之后，需要去考虑负载均衡，监控，日志，统计，故障恢复等等一些问题，这些工作往往都是重复，这个时候我们就可以把它们看作就是我们架构里的基础设施，通过服务网格的形式，让开发者不用去关心这些，让运维人员提高效率。

    常见基础设施层至少要包含 服务发现、负载平衡、故障恢复、日志、监视、追踪、a / b 测试、金丝雀发布、蓝绿部署、频控、访问控制、网关、端到端身份验证等等。

    很多的基础设施在k8s平台上已经提供了，甚于的功能需要一个新的软件来覆盖，代表就是Istio/Linkerd

  - 

- #### Serverless

  云计算的发展 **IDC -> IaaS -> SaaS -> Serverless/FaaS**

  Serverless 直译过来叫无服务器，实际上它不是真的不需要服务器，只不过服务器由云厂商来维护，Serverless 是一种软件系统架构思想和方法，不是软件框架、类库或者工具，它的核心思想：无须关注底层资源，比如：CPU、内存和数据库等，只需关注业务开发

  我们把系统资源进行 Serverless 化，这些系统资源大概分为 2 大类，一种是 CaaS：compute as a service 用来提供计算能力，一种是 BaaS：backend as a service，相当于把第三方组件也 Serverless 化，用户也不用去关注第三方组建的搭建和运维，只需要调用 API 去使用即可。所以 Serverless 大概可以理解为：CaaS + BaaS。

  在最开始的Serverless的最开始可能更像是FaaS，作为开发人员，只需要写一个函数，就可以在例如AWS Lambda等各种函数计算平台上运行起来，实现了对服务器的无感知，同时可以对外快速暴露API接口，可以基于函数级别的自动扩缩容，可以监听各种事件进行触发。

  ![img](http://km.oa.com/files/photos/pictures/201911/1574672045_90_w2298_h1126.png)

  上面的这个过程就是一个serveless服务， 从头到尾用户没有去申请系统资源，也没有对系统资源做任何运维相关的工作

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





