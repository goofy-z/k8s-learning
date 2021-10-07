## 从一个pod的创建开始

1. 由kubectl解析创建pod的yaml，发送创建pod请求到APIServer。
2. APIServer首先做权限认证，然后检查信息并把数据存储到ETCD里，创建deployment资源初始化。
3. kube-controller通过list-watch机制，检查发现新的deployment，将资源加入到内部工作队列，检查到资源没有关联pod和replicaset,然后创建rs资源，rs controller监听到rs创建事件后再创建pod资源。
4. scheduler 监听到pod创建事件，执行调度算法，将pod绑定到合适节点，然后告知APIServer更新pod的`spec.nodeName`
5. kubelet 每隔一段时间通过其所在节点的NodeName向APIServer拉取绑定到它的pod清单，并更新本地缓存。
6. kubelet发现新的pod属于自己，调用容器API来创建容器，并向APIService上报pod状态。
7. Kub-proxy为新创建的pod注册动态DNS到CoreOS。为Service添加iptables/ipvs规则，用于服务发现和负载均衡。
8. deploy controller对比pod的当前状态和期望来修正状态。

从上述流程中，我们能大概清楚kube-scheduler的主要工作，负责整个k8s中pod选择和绑定node的工作，这个选择的过程就是应用调度策略，包括NodeAffinity、PodAffinity、节点资源筛选等等，而绑定便就是将pod的定义文件里的`nodeName`进行更新。

## 总览

kube-scheduler的设计有两个阶段：

1. 基于谓词（predicate）和优先级（priority）的筛选
2. 基于调度框架的调度器，新版本已经把所有的旧的设计都改造成扩展点插件形式(1.19+)



## 详细设计

###谓词和优先级调度

所谓的谓词和优先级都是对调度算法的分类，在kube-scheduler里，谓词调度算法是来选择出一组能够绑定pod的node，而优先级算法则是在这群node中进行打分，得处一个最高分的node。



### 谓词调度





Scheduler入口: 

/cmd/kube-scheduler/scheduler.go

```
func main() {
	command := app.NewSchedulerCommand()
	...
	command.Execute()
}
```

k8中组件通用的启动模版，我们需要找到这个command定义的。

Scheduler的command定义：

/cmd/kube-scheduler/app/server.go

```go
func NewSchedulerCommand(registryOptions ...Option) *cobra.Command {
	...
  cmd := &cobra.Command{ // 定义了一个cobra的Comand结构体, cmd.Execute(),会执行定义的Run函数。
		Run: func(cmd *cobra.Command, args []string) {
			if err := runCommand(cmd, opts, registryOptions...); err != nil { 
				fmt.Fprintf(os.Stderr, "%v\n", err)
				os.Exit(1)
			}
		}
		...
	}
}
```

查看runCommand定义

```go
func runCommand(cmd *cobra.Command, opts *options.Options, registryOptions ...Option) error {
	...
	cc, sched, err := Setup(ctx, opts, registryOptions...) // 初始化配置、Scheduler
	...
	return Run(ctx, cc, sched)
}
```

查看Run定义

```
func Run(ctx context.Context, cc *schedulerserverconfig.CompletedConfig, sched *scheduler.Scheduler) error {
	// To help debugging, immediately log version
	klog.V(1).Infof("Starting Kubernetes Scheduler version %+v", version.Get())

	// Configz registration.
	if cz, err := configz.New("componentconfig"); err == nil {
		cz.Set(cc.ComponentConfig)
	} else {
		return fmt.Errorf("unable to register configz: %s", err)
	}

	// 事件广播
	cc.EventBroadcaster.StartRecordingToSink(ctx.Done())

	// 选举检查
	var checks []healthz.HealthChecker
	if cc.ComponentConfig.LeaderElection.LeaderElect {
		checks = append(checks, cc.LeaderElection.WatchDog)
	}

	// http和metric服务
	if cc.InsecureServing != nil {
		...
	}
	if cc.InsecureMetricsServing != nil {
		... 
	}
	// https服务
	if cc.SecureServing != nil {
		...
	}

	// 启动Informer，和
	go cc.PodInformer.Informer().Run(ctx.Done())
	cc.InformerFactory.Start(ctx.Done())

	// 等待informer缓存完毕
	cc.InformerFactory.WaitForCacheSync(ctx.Done())

	// 选举机制启动
	if cc.LeaderElection != nil {
		...
	}

	// 非选举机制启动过， 无论是选举和非选举启动都会调用最后处理逻辑都会到sched.Run()
	sched.Run(ctx)
	return fmt.Errorf("finished without leader elect")
}
```

sched.Run在`/pkg/scheduler/scheduler.go`里

```
func (sched *Scheduler) Run(ctx context.Context) {
	...
	sched.SchedulingQueue.Run()
	wait.UntilWithContext(ctx, sched.scheduleOne, 0)
	sched.SchedulingQueue.Close()
}
```

其中wait.UntilWithContext将会不间断的调用`sched.scheduleOne`函数，这个函数正是处理pod调度算法的地方，而获取需要调度的pod是执行`sched.NextPod()`。

看到这我们得去找到`sched`的`SchedulingQueue`的接口方法Run方法, 寻找New出`sched`的代码

`/pkg/scheduler/scheduler.go`

```go
func New(...) (*Scheduler, error) {
  configurator = &Configurator{
  	...
  }
  ...
  options := defaultSchedulerOptions // 设置默认配置项
  ...
	var sched *Scheduler
	source := options.schedulerAlgorithmSource
  switch {
  case source.Provider != nil: // source.Provider是设置过默认值
		sc, err := configurator.createFromProvider(*source.Provider)
		sched = sc
  ...
  }
  // 为informer设置监听事件，包括pod（已调度(字段NodeName)-添加到SchedulerCache, 为调度则添加到SchedulingQueue队列中。
  // Node、PV、PVC、SC、CSINode、Service
  addAllEventHandlers(sched, informerFactory, podInformer)
  return sched, nil
}
```

`sched`是由`Scheduler`的声明出来，但是声明出来的`sched`的`SchedulingQueue`是一个nil，在`switch`中`sched`被重新赋值，进入到`createFromProvider`：

```go
func (c *Configurator) createFromProvider(providerName string) (*Scheduler, error) {
	r := algorithmprovider.NewRegistry()
	defaultPlugins, exist := r[providerName]
	if !exist {
		return nil, fmt.Errorf("algorithm provider %q is not registered", providerName)
	}

	for i := range c.profiles {
		prof := &c.profiles[i]
		plugins := &schedulerapi.Plugins{}
		plugins.Append(defaultPlugins)
		plugins.Apply(prof.Plugins)
		prof.Plugins = plugins
	}
	return c.create()
}
```



在`NewRegistry`里其实是声明了默认的调度插件配置

```go
// NewRegistry returns an algorithm provider registry instance.
func NewRegistry() Registry {
	defaultConfig := getDefaultConfig() // 默认调度插件配置
	applyFeatureGates(defaultConfig)
	caConfig := getClusterAutoscalerConfig()  // 集群自动伸缩容有关，当发生集群扩缩容使用的调度策略
	applyFeatureGates(caConfig)
	return Registry{
		schedulerapi.SchedulerDefaultProviderName: defaultConfig,
		ClusterAutoscalerProvider:                 caConfig,
	}
}
```





```go
type Scheduler struct {
  ...
	SchedulingQueue internalqueue.SchedulingQueue // pod调度队列
  ...
}
```

在

查看`createFromProvider`函数中定义创建`sched`的地方，然后找`c.create()`中。

`/pkg/scheduler/factory.go`

```go
// 通过注册的插件来创建sched
func (c *Configurator) create() (*Scheduler, error) {
  ...
  // 获取第一个
	lessFn := profiles[c.profiles[0].SchedulerName].QueueSortFunc()
	podQueue := internalqueue.NewSchedulingQueue(...)
  ...
  algo := core.NewGenericScheduler(...)
	return &Scheduler{
		SchedulerCache:  c.schedulerCache,
		Algorithm:       algo,  // 默认调度算法
		Profiles:        profiles,
    NextPod:         internalqueue.MakeNextPodFunc(podQueue), // nextpod正是调用podQueue.pop()
		Error:           MakeDefaultErrorFunc(c.client, c.informerFactory.Core().V1().Pods().Lister(), podQueue, c.schedulerCache),
		StopEverything:  c.StopEverything,
		SchedulingQueue: podQueue,
	}, nil
}
```

这里我们发现了`SchedulingQueue`是 由`NewSchedulingQueue`声明的一个对象。

`/pkg/scheduler/internal/queue/scheduling_queue.go`

```go
func NewPriorityQueue(
	lessFn framework.LessFunc,
	opts ...Option,
) *PriorityQueue {
	...
	pq := &PriorityQueue{  // 定义了3种队列，activeQ、unschedulableQ、podBackoffQ
		PodNominator:              options.podNominator,
		clock:                     options.clock,
		stop:                      make(chan struct{}),
		podInitialBackoffDuration: options.podInitialBackoffDuration,
		podMaxBackoffDuration:     options.podMaxBackoffDuration,
		activeQ:                   heap.NewWithRecorder(), 
		unschedulableQ:            newUnschedulablePodsMap(),
		moveRequestCycle:          -1,
	}
  pq.podBackoffQ = heap.NewWithRecorder()
	return pq
}
```

`SchedulingQueue`的结构

```
type SchedulingQueue interface {
	...
	Pop() (*framework.QueuedPodInfo, error)
	Update(oldPod, newPod *v1.Pod) error
	Delete(pod *v1.Pod) error
	MoveAllToActiveOrBackoffQueue(event string)
}
```

找到了`sched`的属性`SchedulingQueue`实际上是一个`PriorityQueue`对象，我们找到它的`Run`方法。

```
func (p *PriorityQueue) Run() {
	// 每一秒从podBackoffQ拿出最近的pod检查是否可以加入到activeQ
	go wait.Until(p.flushBackoffQCompleted, 1.0*time.Second, p.stop) 
	// 没30秒从无法调度pod的队列拿出pod检查是否可以加入到activeQ
	go wait.Until(p.flushUnschedulableQLeftover, 30*time.Second, p.stop)
}
```

现在我们找到了整个sched的启动和调度队列管理的功能，接下来查看具体调度一个pod的详细经过。

从`sched.Run`中我们找打了`scheduleOne`方法：`/pkg/scheduler/scheduler.go`

```
func (sched *Scheduler) scheduleOne(ctx context.Context) {
	podInfo := sched.NextPod() // 获取activeQ的下一个pod
	

}
```











参与极客大赛判分功能开发、算法题翻译。

参与极客大赛决赛平台和需求开发及联调，决赛大屏功能开发。

参与可视化比赛功能设计需求开发及联调。

参与白马杯比赛参赛选手数据导入及平台问题对接。

参与极客决赛初赛及决赛邮件通知，比赛数据导入导出、初赛决赛比赛运维支持。

对接客户决赛判分流程及判分镜像制作

参与决赛期间平台运维支持