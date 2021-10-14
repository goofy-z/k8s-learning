[toc]

## 写在前面

本文翻译自KEP（k8s增强特性）其中关于[调度框架](https://github.com/kubernetes/enhancements/tree/master/keps/sig-scheduling/624-scheduling-framework)的一篇文档, 通读之后结合一些学习资料做了这篇翻译，接下来我还会有一篇关于kube-scheduler的详细解剖。


## 总结

本文档描述了k8s调度框架（scheduling framework）。调度框架能够提供API将一组自定义调度器以插件形式添加到现有k8s调度器中。插件将并被编译到调度程序中，这些API在允许自定义调度器的同时，保持调度的核心功能简单且可维护。

## 目的

k8s调度在添加了新特性的同时也使得代码更加复杂，这样就变得难以维护，这样使得自定义调度器很难跟上新版本。当前k8s提供了[webhooks to extend][https://github.com/kubernetes/community/blob/master/contributors/design-proposals/scheduling/scheduler_extender.md]，但还是有一些限制。

1.  扩展的节点有限制，`Filter`扩展器在默认的`predicate`函数之后调用。`Prioritize `在`priority `函数之后调用。`Preempt`扩展器在默认抢占机制执行后调用。`Bind`扩展器用于绑定`Pod`。能够扩展节点已经限制了这些，例如：扩展程序并不能在`predicate`之前调用。
2.  调用扩展器时需要json的序列化和反序列化，切通过webhook（HTTP request）的方式没有直接调用函数快。
3.  无法通知扩展器，调度器已经对该pod停止调度。
4.  扩展器无法使用调度器的缓存。

上述限制阻碍了构建高性能和通用调度器特性。理想情况下，我们希望有一种足够快的扩展机制，允许现有功能转换为插件，比如谓词和优先级函数。这样的插件将被编译到调度程序二进制文件中。此外，开发自定义调度器时可以使用(未修改的)调度器源代码和自定义代码来编译自定义调度器。

### Goals

- 使调度程序更可扩展。
- 将调度器的一些特性移到插件中，使其核心更简单。
- 在框架中提出扩展点。
- 提出一种机制来接收插件结果并基于此继续或中止
  接收到的结果。
- 提出一种机制来处理错误并与插件沟通。

### Non-Goals

-   解决所有调度程序的限制，尽管我们想要确保
    新的框架允许我们在未来解决已知的限制。
-   提供插件和回调函数的实现细节，例如
    所有的参数和返回值。

## 建议

调度框架在调度器中定义了新的扩展点和Go api，来提供给“插件”使用。插件将调度动作添加到调度器中，并一起编译。调度器的ComponentConfig将允许启用、禁用和重新排序插件。用户自定义制调度器可以编写他们的插件“[out- tree](# Custom -scheduler-plugins-out- tree)”，并编译一个包含他们自己插件的调度器二进制文件。

### 调度周期和绑定周期

pod的每次调度被分成两个过程：`调度周期`，`绑定周期`。`调度周期`为pod选择节点，`绑定周期`则应用前面该调度结果，两个周期合并被称为**调度上下文**。`调度周期`是串行进行的，`绑定周期`则是并非进行的([Concurrency](#concurrency))。

如果pod不可调度，或发生了调度内部错误，则`调度周期`和`绑定周期`都可以被中断，pod将会被返回到调度队列，如果是`绑定周期`终止，它会触发`reserve`插件的[Unreserve](#reserve) 方法。

### 扩展点

下图显示了调度框架公开的pod的调度上下文和扩展点。在这张图中，“Filter”相当于“Predicate”，“Scoring”相当于“Priority function”。插件可以注册到不同的扩展点来供调用，下面会介绍每一个扩展点的详细概述。

![image](../../image/scheduling-framework-extensions.png)

#### Queue sort

这个插件将被用于给调度队列的pod排序，一个队列排序插件本质上将提供一个“less(pod1, pod2)”函数。一次只能启用一个队列排序插件。

接口：

```go
type QueueSortPlugin interface {
  Plugin // 继承父Plugin，里面实现了一个 Name()
	Less(*QueuedPodInfo, *QueuedPodInfo) bool // 用于`activeQ`这个`heap`结构的比较函数。
}
```

#### PreFilter

这个插件用于对pod的信息进行预处理，或者检查集群或pod必须满足的某些条件。必须实现一个PreFilter函数，在每个pod的调度周期里只会调度一次，如果执行PreFilter返回一个错误，调度周期被中止。


Pre-filter插件可以实现可选的“PreFilterExtensions”接口，该接口定义了**AddPod**和**RemovePod**方法，均会在**PreFilter**函数之后调用，其中**PreFilter**只在每个pod调度时执行一次，而后面两个函数会在每个node执行，一般会在PreFilter中存储这两个函数需要的信息到CycleState中。

**AddPod**：在调度框架评估某一个新增的pod对当前待调度pod的影响时会被调用，主要是提名pod还未调度到当前node(等待被抢占pod退出)。

**RemovePod**：当某个pod需要从节点解除绑定，也就是被抢占调度时，执行其来评估对待调度pod的影响。

接口：

```go
type PreFilterPlugin interface {
	Plugin
	PreFilter(ctx context.Context, state *CycleState, p *v1.Pod) *Status
	PreFilterExtensions() PreFilterExtensions
}
```

这里的 CycleState ，表示调度的上下文，其实是一个 map 的封装，结构体内部通过读写锁实现了并发安全，开发者可以通过 CycleState 来实现多个调度插件直接的数据传递，也就是多个插件可以共享状态或通过此机制进行通信。

接口：

```go
type CycleState struct {
	mx      sync.RWMutex
	storage map[StateKey]StateData 
	recordPluginMetrics bool // 是否记录每个插件执行时间
}
```

#### Filter

用于过滤掉不能运行Pod的节点。对于每个节点，调度程序将按已配置的顺序调用筛选插件。如果任何过滤器插件将该节点标记为不可行的，则不会为该节点调用其余的插件。可以在同一个调度周期中多次调用，并且会在并发为每一个节点进行筛选操作。

接口：过滤插件其实很类似上一代 kubernetes 调度器中的预选环节，即 Predicates。

```go
type FilterPlugin interface {
	Plugin
	Filter(ctx context.Context, state *CycleState, pod *v1.Pod, nodeInfo *NodeInfo) *Status
}
```


#### PostFilter

插件在Filter阶段之后调用，但只在没有找到pod的可行节点时才调用。该插件按其配置的顺序被调用。如果任何PostFilter插件将节点标记为可调度的，则不会调用其余的插件。典型的PostFilter实现是抢占，通过抢占其他pod使pod可调度。

接口：

```go
type PostFilterPlugin interface {
	Plugin
	// 插件返回状态:
	// - Unschedulable: 插件执行成功，但pod不可调度。
	// - Success: 插件执行成功，pod可调度。
	// - Error: 插件执行报错。
	PostFilter(ctx context.Context, state *CycleState, pod *v1.Pod, filteredNodeStatusMap NodeToStatusMap) (*PostFilterResult, *Status)
}
```


#### PreScore

该扩展点将使用**通过 Filter 阶段的节点列表**来调用插件。插件可以使用此数据来更新内部状态或生成日志、指标。比如可以通过该扩展点收集各个节点中性能指标，所有节点中最大的内存的节点，性能最好的 CPU 节点等。

接口：

```go
type PreScorePlugin interface {
	Plugin
	PreScore(ctx context.Context, state *CycleState, pod *v1.Pod, nodes []*v1.Node) *Status
}
```

#### Scoring

打分插件共分为两个阶段：

1. 第一个阶段被称为“打分”，主要是对筛选阶段的node进行排名，为每一个node调用`Score`函数。
2. 第一个阶段称为“归一化打分”，可选的阶段，每一个“归一化”插件都会收到由同一个`Score`函数给所有节点的评分，然后再统一对评分结果进行修改，一般是对评分的结果修改成`[MinNodeScore, MaxNodeScore]`区间的值，比如统一的节点打分时1到100，但有的插件总分时50，这个时候需要`NormalizeScore`函数来合理扩大插件打分结果。

这两个函数无论谁返回error都会终止调度周期

#### Reserve

Reserve扩展的插件有两个方法：`Reserve`和`Unreserve`，分别支持两个信息调度阶段，分别称为Reserve和Unreserve。维护运行时状态的插件可以应实现此扩展点，这是在调度程序实际将 Pod 绑定到 Node 之前发生的，它的存在是为了防止在调度程序等待绑定成功时发生竞争(因为绑定是异步执行的，调度下一个Pod可能发生在绑定完成之前)。

```go
type ReservePlugin interface {
    Plugin
    // 在绑定Pod到Node之前为Pod预留资源
    Reserve(ctx context.Context, state *CycleState, p *v1.Pod, nodeName string) *Status
    // 在Reserve()与绑定成功之间有任何错误,则调用来恢复预留的资源
    Unreserve(ctx context.Context, state *CycleState, p *v1.Pod, nodeName string)
}
```



#### Permit

PermitPlugin插件用于阻止或延迟Pod绑定，可以做以下三个事情之一：

1.  **approve** 所有PermitPlugin都批准了Pod后，Pod才可以绑定.
    
1.  **deny** 如果任何PermitPlugin拒绝Pod，将返回到调度队列，这将触发ReservePlugin.Unreserve().
    
3.  **wait** (可设置超时) 如果PermitPlugin返回‘wait’，则Pod将保持在许可阶段，直到插件批准它为止；如果超时，等待将变成拒绝，并将Pod返回到调度队列，同时触发ReservePlugin.Unreserve().

PermitPlugin是在调度周期的最后一步执行，在PreBind插件执行之前。 

```go
type PermitPlugin interface {
	Plugin
	Permit(ctx context.Context, state *CycleState, p *v1.Pod, nodeName string) (*Status, time.Duration)
}
```

**Approving a pod binding**

虽然任何插件都可以从缓存中接收到保留pod的列表并批准它们(参见[' FrameworkHandle '](# FrameworkHandle))，但我们希望只有Approving 插件才能批准处于“等待”状态的保留pod的绑定。一但Pod被批准，将进入预绑定阶段。

#### PreBind

PreBindPlugin插件用于执行绑定Pod之前所需要的工作，例如：可以设置网络卷并将其安装在目标Node上，然后在允许Pod在节点上运行。如果插件报错则pod会被重新加入调度队列，并调用`Unreserve`来恢复状态。

```go
type PreBindPlugin interface {
    Plugin
    PreBind(ctx context.Context, state *CycleState, p *v1.Pod, nodeName string) *Status
}
```

#### Bind

这些插件用于将pod绑定到Node。直到所有的PreBind插件都完成后，Bind插件才会被调用。按照配置的顺序调用每个bind插件。绑定插件可以选择是否处理给定的Pod。如果一个绑定插件选择处理一个Pod， **其余绑定插件将被跳过**。如果插件报错则pod会被重新加入调度队列，并调用`Unreserve`来恢复状态。

```go
type PreBindPlugin interface {
	Plugin
	PreBind(ctx context.Context, state *CycleState, p *v1.Pod, nodeName string) *Status
}
```

#### PostBind

成功绑定pod后调用PostBind插件。这是绑定周期的结束，可以用来清理关联的资源。

```go

type PostBindPlugin interface {
	Plugin
	PostBind(ctx context.Context, state *CycleState, p *v1.Pod, nodeName string)
}
```

### 插件API

插件API有两个步骤。首先，插件必须注册和配置，然后它们使用扩展点接口。扩展点接口具有以下形式。

```go
type Plugin interface {
   Name() string
}

type QueueSortPlugin interface {
   Plugin
   Less(*PodInfo, *PodInfo) bool
}


type PreFilterPlugin interface {
   Plugin
   PreFilter(CycleState, *v1.Pod) *Status
}

// ...
```

#### CycleState

大部分的插件函数的调用需要`CycleState`参数，它代表着调度上下文，正是因为许多的插件可以并发执行，所以插件可以通过`CycleState`来做隔离，同时也作为不同插件之间的数据传递，该上下文随着当前pod的调度结束而销毁。

#### FrameworkHandle

`CycleState`提供了与单个调度上下文相关的api，而`FrameworkHandle`提供了与插件生命周期相关的api。例如插件如何获取k8客户端和内部缓存（不能保证数据同步），或者从调度程序的集群状态缓存中读取数据等等。

#### 插件注册

每一个参数必须有个构造函数，并将它添加到插件的注册表中进行编译。

例如：

```go
type PluginFactory = func(runtime.Unknown, FrameworkHandle) (Plugin, error)

type Registry map[string]PluginFactory

func NewRegistry() Registry {
   return Registry{
      fooplugin.Name: fooplugin.New,
      barplugin.Name: barplugin.New,
      // New plugins are registered here.
   }
}
```

注册表`/pkg/scheduler/framework/plugins/registry.go`

```go
// 新写的插件在这里用名字作为唯一标识加入到Registry这个map中
func NewInTreeRegistry() runtime.Registry {
	return runtime.Registry{
		selectorspread.Name:                        selectorspread.New,
		...
	}
}
```

### 插件的生命周期

#### 初始化

每个插件只初始化一次，即使有多个扩展点，插件的初始化传递两个参数`config args`和`FrameworkHandle`

#### 并发场景

插件在计算多个节点，或者不同上下文，都会被并发调用。但是在一个调度上下文里，每一个扩展点的调用都是串行的。

在调度程序的主线程中，一次只处理一个调度周期。任何达到或包含[permit](#permit)的扩展点将在下一个调度周期开始之前完成，一旦permit完成之后，绑定周期将异步执行，此时调度周期进行下一个pod的调度，这意味着一个插件可以从两个不同的调度上下文并发调用，插件如果有设置permit之后的扩展点，就需要注意这种情况。

最后，reserve插件中的[Unreserve](#reserve)方法可以从主线程或绑定线程调用，queue sort 扩展点不属于调度上下文，可以对许多pod并发调用。

### 插件配置

调度程序的组件配置将允许启用、禁用或以其他方式配置插件，插件配置分为两部分：

1.  为每一个扩展点配置一个有序的插件列表，并可以设置启停。
2.  每个插件的可选自定义插件参数集。

例如：

```json
{
  "plugins": {
    "preFilter": [
      {
        "name": "PluginA"
      },
      {
        "name": "PluginB"
      },
      {
        "name": "PluginC"
      }
    ],
    "score": [
      {
        "name": "PluginA",
        "weight": 30
      },
      {
        "name": "PluginX",
        "weight": 50
      },
      {
        "name": "PluginY",
        "weight": 10
      }
    ]
  },
  "pluginConfig": [
    {
      "name": "PluginX",
      "args": {
        "favorite_color": "#326CE5",
        "favorite_number": 7,
        "thanks_to": "thockin"
      }
    }
  ]
}
```

#### Optional Args

插件可以从配置中接收任意结构的参数。因为一个插件可能出现在多个扩展点，配置是在' PluginConfig '的单独列表中。

例如：

```json
{
   "name": "ServiceAffinity",
   "args": {
      "LabelName": "app",
      "LabelValue": "mysql"
   }
}
```

```go
// 插件的构造函数
func NewServiceAffinity(args *runtime.Unknown, h FrameworkHandle) (Plugin, error) {
    if args == nil {
        return nil, errors.Errorf("cannot find service affinity plugin config")
    }
    if args.ContentType != "application/json" {
        return nil, errors.Errorf("cannot parse content type: %v", args.ContentType)
    }
    var config struct {
        LabelName, LabelValue string
    }
    if err := json.Unmarshal(args.Raw, &config); err != nil {
        return nil, errors.Wrap(err, "could not parse args")
    }
    //...
}
```

#### 向后兼容

当前的`KubeSchedulerConfiguration`类型有`apiVersion: kubescheduler.config.k8s.io/v1alpha1`。这个新的配置格式将是`v1alpha2`或`v1beta1`。当更新版本的调度器解析一个`v1alpha1`时，"policy"部分将被用来构造一个等效的插件配置。

### 自动伸缩容

必须修改Cluster Autoscaler以运行Filter插件而不是谓词的形式，这可以通过创建一个框架实例并调用`RunFilterPlugins`来实现。

## 示例

这些只是如何使用调度框架的几个示例。

### Coscheduling

Functionality similar to
[kube-batch](https://github.com/kubernetes-sigs/kube-batch) (sometimes called
"gang scheduling") could be implemented as a plugin. For pods in a batch, the
plugin would "accumulate" pods in the [permit](#permit) phase by using the
"wait" option. Because the permit stage happens after [reserve](#reserve),
subsequent pods will be scheduled as if the waiting pod is using those
resources. Once enough pods from the batch are waiting, they can all be
approved.

### Dynamic Resource Binding

[Topology-Aware Volume Provisioning](https://kubernetes.io/blog/2018/10/11/topology-aware-volume-provisioning-in-kubernetes/)
can be (re)implemented as a plugin that registers for [filter](#filter) and
[pre-bind](#pre-bind) extension points. At the filtering phase, the plugin can
ensure that the pod will be scheduled in a zone which is capable of provisioning
the desired volume. Then at the PreBind phase, the plugin can provision the
volume before letting scheduler bind the pod.

### Custom Scheduler Plugins (out of tree)

调度框架允许人们编写定制的、高性能的调度器特性，而无需fork调度器的代码。要做到这一点，开发人员只需要编写他们自己的`main()`包装调度器。因为插件必须用调度程序编译，所以为了避免修改源调度器的代码，有必要为`main()`编写一个包装器。

```go
import (
    scheduler "k8s.io/kubernetes/cmd/kube-scheduler/app"
)

func main() {
    command := scheduler.NewSchedulerCommand(
      			// 在这里引入你的插件
            scheduler.WithPlugin("example-plugin1", ExamplePlugin1),
            scheduler.WithPlugin("example-plugin2", ExamplePlugin2))
    if err := command.Execute(); err != nil {
        fmt.Fprintf(os.Stderr, "%v\n", err)
        os.Exit(1)
    }
}
```

