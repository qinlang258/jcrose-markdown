# k8s-shceduler 自定义调度插件

自身k8s版本 1.26.7

**基于kubernetes-1.26.7/pkg/scheduler/framework/plugins 项目进行二次开发扩展**

![1][([https://github.com/qinlang258/jcrose-markdown/main/kubernetes/images/1.png](https://github.com/qinlang258/jcrose-markdown/blob/main/images/1.png))](https://github.com/qinlang258/jcrose-markdown/blob/main/images/1.png)

调度 Pod 可以分为以下几个阶段：

- 预入队列：将 Pod 被添加到内部活动队列之前被调用，在此队列中 Pod 被标记为准备好进行调度
- 排队：用于对调度队列中的 Pod 进行排序。
- 过滤：过滤掉不满足条件的节点
- 打分：对通过的节点按照优先级排序
- 保留资源：因为是 Pod 的创建是异步的，所以需要在找到最合适的机器后先进行资源的保留
- 批准、拒绝、等待：调度完成后对于发送 Pod 的信息进行批准、拒绝、等待
- 绑定：最后从中选择优先级最高的节点，如果中间任何一步骤有错误，就直接返回错误



**预入队列**
**PreEnqueue**
这些插件在将 Pod 被添加到内部活动队列之前被调用，在此队列中 Pod 被标记为准备好进行调度。

只有当所有 PreEnqueue 插件返回`Success`时，Pod 才允许进入活动队列。 否则，它将被放置在内部无法调度的 Pod 列表中，并且不会获得`Unschedulable`状态。

------

**排队**
**Sort**
Sort 用于对调度队列中的 Pod 进行排序。 队列排序插件本质上提供 less(Pod1, Pod2) 函数。 一次只能启动一个队列插件。

------

**过滤**
**PreFilter**
这些插件用于预处理 Pod 的相关信息，或者检查集群或 Pod 必须满足的某些条件。如果 PreFilter 插件返回错误，则调度周期将终止。

**Filter**
这些插件用于过滤出不能运行该 Pod 的节点。对于每个节点，调度器将按照其配置顺序调用这些过滤插件。如果任何过滤插件将节点标记为不可行，则不会为该节点调用剩下的过滤插件。节点可以被同时进行评估。

**PostFilter**
这些插件在 Filter 阶段后调用，但`仅在该 Pod 没有可行的节点时调用`。 插件按其配置的顺序调用。如果任何 PostFilter 插件标记节点为“Schedulable”， 则其余的插件不会调用。典型的 PostFilter 实现是抢占，试图通过抢占其他 Pod 的资源使该 Pod 可以调度。

------

**打分**
**PreScore**
这些插件用于执行 “前置评分（pre-scoring）” 工作，即生成一个可共享状态供 Score 插件使用。 如果 PreScore 插件返回错误，则调度周期将终止。

**Score**
这些插件用于对通过过滤阶段的节点进行排序。调度器将为每个节点调用每个评分插件。 将有一个定义明确的整数范围，代表最小和最大分数。 在标准化评分阶段之后，调度器将根据配置的插件权重合并所有插件的节点分数。

**NormalizeScore**
这些插件用于在调度器计算 Node 排名之前修改分数。 在此扩展点注册的插件被调用时会使用同一插件的 Score 结果。 每个插件在每个调度周期调用一次。

------

**保留资源**
**Reserve**
实现了 Reserve 扩展的插件，拥有两个方法，即 Reserve 和 Unreserve， 他们分别支持两个名为 Reserve 和 Unreserve 的信息处理性质的调度阶段。 维护运行时状态的插件（又称 "有状态插件"）应该使用这两个阶段，以便在节点上的资源被保留和未保留给特定的 Pod 时得到调度器的通知。

Reserve 阶段发生在调度器实际将一个 Pod 绑定到其指定节点之前。 它的存在是为了防止在调度器等待绑定成功时发生竞争情况。 每个 Reserve 插件的 Reserve 方法可能成功，也可能失败； 如果一个 Reserve 方法调用失败，后面的插件就不会被执行，Reserve 阶段被认为失败。 如果所有插件的 Reserve 方法都成功了，Reserve 阶段就被认为是成功的，剩下的调度周期和绑定周期就会被执行。

如果 Reserve 阶段或后续阶段失败了，则触发 Unreserve 阶段。发生这种情况时，所有 Reserve 插件的 Unreserve 方法将按照 Reserve 方法调用的相反顺序执行。这个阶段的存在是为了清理与保留的 Pod 相关的状态。

这个是调度周期的最后一步。一旦 Pod 处于保留状态，它将在绑定周期结束时触发 Unreserve 插件（失败时）或 PostBind 插件（成功时）。

------

**批准、拒绝、等待**
**Permit**
Permit 插件在每个 Pod 调度周期的最后调用，用于防止或延迟 Pod 的绑定。 一个允许插件可以做以下三件事之一：

- 批准
  一旦所有 Permit 插件批准 Pod 后，该 Pod 将被发送以进行绑定。
- 拒绝
  如果任何 Permit 插件拒绝 Pod，则该 Pod 将被返回到调度队列。 这将触发 Reserve 插件中的 Unreserve 阶段。
- 等待（带有超时）
  如果一个 Permit 插件返回 “等待” 结果，则 Pod 将保持在一个内部的 “等待中” 的 Pod 列表，同时该 Pod 的绑定周期启动时即直接阻塞直到得到批准。 如果超时发生，等待变成拒绝，并且 Pod 将返回调度队列，从而触发 Reserve 插件中的 Unreserve 阶段。

------

**绑定**
**PreBind**
这些插件用于执行 Pod 绑定前所需的所有工作。 例如，一个 PreBind 插件可能需要制备网络卷并且在允许 Pod 运行在该节点之前 将其挂载到目标节点上。

如果任何 PreBind 插件返回错误，则 Pod 将被拒绝并且退回到调度队列中。

**Bind**
Bind 插件用于将 Pod 绑定到节点上。直到所有的 PreBind 插件都完成，Bind 插件才会被调用。各 Bind 插件按照配置顺序被调用。Bind 插件可以选择是否处理指定的 Pod。 如果某 Bind 插件选择处理某 Pod，剩余的 Bind 插件将被跳过。

**PostBind**
这是个信息性的扩展点。 PostBind 插件在 Pod 成功绑定后被调用。这是绑定周期的结尾，可用于清理相关的资源。

**Unreserve**
这是个信息性的扩展点。 如果 Pod 被保留，然后在后面的阶段中被拒绝，则 Unreserve 插件将被通知。 Unreserve 插件应该清楚保留 Pod 的相关状态。

使用此扩展点的插件通常也使用 Reserve。





如果我们要实现自己的插件，必须向调度框架注册插件并完成配置，另外还必须实现扩展点接口，对应的扩展点接口我们可以在源码 `pkg/scheduler/framework/interface.go` 文件中找到，如下所示：

```go
// Plugin is the parent type for all the scheduling framework plugins.
type Plugin interface {
	Name() string
}

//QueueSort 扩展用于对 Pod 的待调度队列进行排序，以决定先调度哪个 Pod，QueueSort 扩展本质上只需要实现一个方法 Less(Pod1, Pod2) 用于比较两个 Pod 谁更优先获得调度即可，同一时间点只能有一个 QueueSort 插件生效
type QueueSortPlugin interface {
	Plugin
	Less(*PodInfo, *PodInfo) bool
}


//Pre-filter 扩展用于对 Pod 的信息进行预处理，或者检查一些集群或 Pod 必须满足的前提条件，如果 pre-filter 返回了 error，则调度过程终止。
type PreFilterPlugin interface {
	Plugin
	PreFilter(pc *PluginContext, p *v1.Pod) *Status
}

//Filter 扩展用于排除那些不能运行该 Pod 的节点，对于每一个节点，调度器将按顺序执行 filter 扩展；如果任何一个 filter 将节点标记为不可选，则余下的 filter 扩展将不会被执行。调度器可以同时对多个节点执行 filter 扩展
type FilterPlugin interface {
	Plugin
	Filter(pc *PluginContext, pod *v1.Pod, nodeName string) *Status
}


//Post-filter 是一个通知类型的扩展点，调用该扩展的参数是 filter 阶段结束后被筛选为可选节点的节点列表，可以在扩展中使用这些信息更新内部状态，或者产生日志或 metrics 信息。
type PostFilterPlugin interface {
	Plugin
	PostFilter(pc *PluginContext, pod *v1.Pod, nodes []*v1.Node, filteredNodesStatuses NodeToStatusMap) *Status
}

// ScorePlugin is an interface that must be implemented by "score" plugins to rank
// nodes that passed the filtering phase.
//Scoring 扩展用于为所有可选节点进行打分，调度器将针对每一个节点调用 Soring 扩展，评分结果是一个范围内的整数。在 normalize scoring 阶段，调度器将会把每个 scoring 扩展对具体某个节点的评分结果和该扩展的权重合并起来，作为最终评分结果
type ScorePlugin interface {
	Plugin
	Score(pc *PluginContext, p *v1.Pod, nodeName string) (int, *Status)
}


//Normalize scoring 扩展在调度器对节点进行最终排序之前修改每个节点的评分结果，注册到该扩展点的扩展在被调用时，将获得同一个插件中的 scoring 扩展的评分结果作为参数，调度框架每执行一次调度，都将调用所有插件中的一个 normalize scoring 扩展一次
type ScoreWithNormalizePlugin interface {
	ScorePlugin
	NormalizeScore(pc *PluginContext, p *v1.Pod, scores NodeScoreList) *Status
}

// Reserve 是一个通知性质的扩展点，有状态的插件可以使用该扩展点来获得节点上为 Pod 预留的资源，该事件发生在调度器将 Pod 绑定到节点之前，目的是避免调度器在等待 Pod 与节点绑定的过程中调度新的 Pod 到节点上时，发生实际使用资源超出可用资源的情况。（因为绑定 Pod 到节点上是异步发生的）。这是调度过程的最后一个步骤，Pod 进入 reserved 状态以后，要么在绑定失败时触发 Unreserve 扩展，要么在绑定成功时，由 Post-bind 扩展结束绑定过程。&#x20;

type ReservePlugin interface {
	Plugin
	Reserve(pc *PluginContext, p *v1.Pod, nodeName string) *Status
}


//Pre-bind 扩展用于在 Pod 绑定之前执行某些逻辑。例如，pre-bind 扩展可以将一个基于网络的数据卷挂载到节点上，以便 Pod 可以使用。如果任何一个 pre-bind 扩展返回错误，Pod 将被放回到待调度队列，此时将触发 Unreserve 扩展
type PreBindPlugin interface {
	Plugin
	PreBind(pc *PluginContext, p *v1.Pod, nodeName string) *Status
}

//Post-bind 是一个通知性质的扩展：
//1.  Post-bind 扩展在 Pod 成功绑定到节点上之后被动调用;
//2.  Post-bind 扩展是绑定过程的最后一个步骤，可以用来执行资源清理的动作
type PostBindPlugin interface {
	Plugin
	PostBind(pc *PluginContext, p *v1.Pod, nodeName string)
}

// Unreserve 是一个通知性质的扩展，如果为 Pod 预留了资源，Pod 又在被绑定过程中被拒绝绑定，则 unreserve 扩展将被调用。Unreserve 扩展应该释放已经为 Pod 预留的节点上的计算资源。在一个插件中，reserve 扩展和 unreserve 扩展应该成对出现
type UnreservePlugin interface {
	Plugin
	Unreserve(pc *PluginContext, p *v1.Pod, nodeName string)
}



//Permit 扩展用于阻止或者延迟 Pod 与节点的绑定。Permit 扩展可以做下面三件事中的一项;
//1.  approve（批准）：当所有的 permit 扩展都 approve 了 Pod 与节点的绑定，调度器将继续执行绑定过程&#x20;
//2.  deny（拒绝）：如果任何一个 permit 扩展 deny 了 Pod 与节点的绑定，Pod 将被放回到待调度队列，此时将触发 Unreserve 扩展;
//3.  wait（等待）：如果一个 permit 扩展返回了 wait，则 Pod 将保持在 permit 阶段，直到被其他扩展 approve，如果超时事件发生，wait 状态变成 deny，Pod 将被放回到待调度队列，此时将触发 Unreserve 扩展
type PermitPlugin interface {
	Plugin
	Permit(pc *PluginContext, p *v1.Pod, nodeName string) (*Status, time.Duration)
}



// Bind 扩展用于将 Pod 绑定到节点上： 只有所有的 pre-bind 扩展都成功执行了，bind 扩展才会执行 调度框架按照 bind 扩展注册的顺序逐个调用 bind 扩展 具体某个 bind 扩展可以选择处理或者不处理该 Pod 如果某个 bind 扩展处理了该 Pod 与节点的绑定，余下的 bind 扩展将被忽略;

type BindPlugin interface {
	Plugin
	Bind(pc *PluginContext, p *v1.Pod, nodeName string) *Status
}
```



要实现一个调度器插件必须满足两个条件：

1.  必须实现对应扩展点插件接口
2.  将此插件在调度框架中进行注册。

在kubernetes默认调度流程中插入自定义逻辑的大概流程是：

*   写一个KubeSchedulerConfiguration文件，将其挂载到kube-scheduler容器中。KubeSchedulerConfiguration 配置了某些阶段回调哪个插件的逻辑。
*   修改kube-scheduler容器配置，一般是/etc/kubernetes/manifests/kube-scheduler.yaml。配置--config参数指向KubeSchedulerConfiguration文件在容器中的路径。修改保存之后，kubernetes会自动重启kube-system命名空间的scheduler容器让配置生效。



## 二 开发插件

/usr/local/kubernetes-1.26.7/pkg/scheduler/framework/plugins/myplugin/main.go

将此插件在调度框架中进行注册

```go
package main

import (
	"k8s.io/kubernetes/pkg/scheduler/framework/plugins/myplugin/plugin"
	"os"

	"k8s.io/kubernetes/cmd/kube-scheduler/app"
)

func main() {
	command := app.NewSchedulerCommand(
		app.WithPlugin(plugin.Name, plugin.New),
	)		

	if err := command.Execute(); err != nil {
		os.Exit(1)
	}
}

```

/usr/local/kubernetes-1.26.7/pkg/scheduler/framework/plugins/myplugin/plugin/myplugin.go

必须实现对应扩展点插件接口

```go
package plugin

import (
	"context"
	"k8s.io/api/core/v1"
	"k8s.io/apimachinery/pkg/runtime"
	"k8s.io/kubernetes/pkg/scheduler/framework"
	"log"
)

// Name is the name of the plugin used in the plugin registry and configurations.

const Name = "sample"

// Sort is a plugin that implements QoS class based sorting.

type sample struct{}

var _ framework.FilterPlugin = &sample{}
var _ framework.PreScorePlugin = &sample{}

// New initializes a new plugin and returns it.
func New(_ runtime.Object, _ framework.Handle) (framework.Plugin, error) {
	return &sample{}, nil
}

// Name returns name of the plugin.
func (pl *sample) Name() string {
	return Name
}

func (pl *sample) Filter(ctx context.Context, state *framework.CycleState, pod *v1.Pod, nodeInfo *framework.NodeInfo) *framework.Status {
	log.Printf("filter pod: %v, node: %v", pod.Name, nodeInfo)
	log.Println(state)

	// 排除没有cpu=true标签的节点
	if nodeInfo.Node().Labels["cpu"] != "true" {
		return framework.NewStatus(framework.Unschedulable, "Node: "+nodeInfo.Node().Name)
	}
	return framework.NewStatus(framework.Success, "Node: "+nodeInfo.Node().Name)
}

func (pl *sample) PreScore(ctx context.Context, state *framework.CycleState, pod *v1.Pod, nodes []*v1.Node) *framework.Status {
	log.Println(nodes)
	return framework.NewStatus(framework.Success, "Node: "+pod.Name)
}

```

## 三 部署插件

### 3.1 生成二进制文件

```powershell
cd /usr/local/kubernetes-1.26.7/pkg/scheduler/framework/plugins/myplugin/
go build -o sample-scheduler main.go
```

Dockerfile

```dockerfile
FROM  ubuntu:20.04
WORKDIR .
COPY sample-scheduler /usr/local/bin
CMD ["sample-scheduler"]

```

```powershell
docker tag registry.cn-zhangjiakou.aliyuncs.com/jcrose-k8s/sample-scheduler:v0.0.1 .
docker push registry.cn-zhangjiakou.aliyuncs.com/jcrose-k8s/sample-scheduler:v0.0.1
```

scheduler.yaml

```yaml
kind: ClusterRole
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: sample-scheduler-clusterrole
rules:
  - apiGroups:
      - ""
    resources:
      - namespaces
    verbs:
      - create
      - get
      - list
  - apiGroups:
      - ""
    resources:
      - endpoints
      - events
    verbs:
      - create
      - get
      - update
  - apiGroups:
      - ""
    resources:
      - nodes
    verbs:
      - get
      - list
      - watch
  - apiGroups:
      - ""
    resources:
      - pods
    verbs:
      - delete
      - get
      - list
      - watch
      - update
  - apiGroups:
      - ""
    resources:
      - bindings
      - pods/binding
    verbs:
      - create
  - apiGroups:
      - ""
    resources:
      - pods/status
    verbs:
      - patch
      - update
  - apiGroups:
      - ""
    resources:
      - replicationcontrollers
      - services
    verbs:
      - get
      - list
      - watch
  - apiGroups:
      - apps
      - extensions
    resources:
      - replicasets
    verbs:
      - get
      - list
      - watch
  - apiGroups:
      - apps
    resources:
      - statefulsets
    verbs:
      - get
      - list
      - watch
  - apiGroups:
      - policy
    resources:
      - poddisruptionbudgets
    verbs:
      - get
      - list
      - watch
  - apiGroups:
      - ""
    resources:
      - persistentvolumeclaims
      - persistentvolumes
    verbs:
      - get
      - list
      - watch
  - apiGroups:
      - ""
    resources:
      - configmaps
    verbs:
      - get
      - list
      - watch
  - apiGroups:
      - "storage.k8s.io"
    resources:
      - storageclasses
      - csinodes
      - csistoragecapacities
      - csidrivers
    verbs:
      - get
      - list
      - watch
  - apiGroups:
      - "coordination.k8s.io"
    resources:
      - leases
    verbs:
      - create
      - get
      - list
      - update
  - apiGroups:
      - "events.k8s.io"
    resources:
      - events
    verbs:
      - create
      - patch
      - update
---
apiVersion: v1
kind: ServiceAccount
metadata:
  name: sample-scheduler-sa
  namespace: kube-system
---
kind: ClusterRoleBinding
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: sample-scheduler-clusterrolebinding
  namespace: kube-system
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: sample-scheduler-clusterrole
subjects:
  - kind: ServiceAccount
    name: sample-scheduler-sa
    namespace: kube-system
---
apiVersion: v1
kind: ConfigMap
metadata:
  name: scheduler-config
  namespace: kube-system
data:
  scheduler-config.yaml: |
    apiVersion: kubescheduler.config.k8s.io/v1
    kind: KubeSchedulerConfiguration
    leaderElection:
      leaderElect: false
      leaseDuration: 15s
      renewDeadline: 10s
      resourceName: sample-scheduler
      resourceNamespace: kube-system
      retryPeriod: 2s
    profiles:
      - schedulerName: sample-scheduler
        plugins:
          filter:
            enabled:
              - name: sample
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: sample-scheduler
  namespace: kube-system
  labels:
    component: sample-scheduler
spec:
  selector:
    matchLabels:
      component: sample-scheduler
  template:
    metadata:
      labels:
        component: sample-scheduler
    spec:
      serviceAccountName: sample-scheduler-sa
      priorityClassName: system-cluster-critical
      volumes:
        - name: scheduler-config
          configMap:
            name: scheduler-config
      containers:
        - name: scheduler
          image: registry.cn-zhangjiakou.aliyuncs.com/jcrose-k8s/sample-scheduler:v0.0.1
          imagePullPolicy: IfNotPresent
          command:
            - sample-scheduler
            - --config=/etc/kubernetes/scheduler-config.yaml
            - --v=3
          volumeMounts:
            - name: scheduler-config
              mountPath: /etc/kubernetes
```

这个yaml 文件共完成以下几件事：

1.  通过 `ConfigMap` 声明一个 `KubeSchedulerConfiguration` 配置
2.  创建一个 `Deployment` 对象，其中容器镜像 registry.cn-zhangjiakou.aliyuncs.com/jcrose-k8s/sample-scheduler\:v0.0.1 是前面我们开发的插件应用，对于调度器插件配置通过 `volume` 的方式存储到容器里 `/etc/kubernetes/scheduler-config.yaml`，应用启动时指定此配置文件；这里为了调试方便指定了日志 `--v=3` 等级
3.  创建一个`ClusterRole`，指定不同资源的访问权限
4.  创建一个`ServiceAccount`
5.  声明一个 `ClusterRoleBinding` 对象，绑定 `ClusterRole` 和 `ServiceAccount` 两者的关系

### 3.2 测试

创建一个pod，并指定调度器为 `sample-scheduler`

```powershell
# test-scheduler.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: test-scheduler
spec:
  selector:
    matchLabels:
      app: test-scheduler
  template:
    metadata:
      labels:
        app: test-scheduler
    spec:
      schedulerName: sample-scheduler # 指定使用的调度器，不指定使用默认的default-scheduler
      containers:
        - image: nginx
          imagePullPolicy: IfNotPresent
          name: nginx
          ports:
            - containerPort: 80
```

```pow
➜ kubectl apply -f test-scheduler.yaml

```

由于插件主要实现对节点筛选，排除那些不能运行该 Pod 的节点，运行时将检查节点是否存在 `cpu=true` 标签，如果不存在这个label标签，则说明此节点无法通过预选阶段，后面的调度与绑定步骤就不可能执行。

```powershell
root@jcrose:~# kubectl get po
NAME                                       READY   STATUS    RESTARTS     AGE
calico-kube-controllers-7c6f5c46cc-vwdxh   1/1     Running   0            51m
calico-node-75xkf                          1/1     Running   0            51m
calico-typha-d76b77b7f-bxwlm               1/1     Running   0            51m
nginx-web-4vllh                            1/1     Running   0            5d2h
nginx-web-rmtff                            1/1     Running   1 (3d ago)   5d2h
sample-scheduler-77fd75dbcc-vprkc          1/1     Running   0            11m
test-scheduler-566cc4c4c7-495wg            0/1     Pending   0            10m



root@jcrose:~# kubectl describe po test-scheduler-566cc4c4c7-495wg
  Type     Reason            Age    From              Message
  ----     ------            ----   ----              -------
  Warning  FailedScheduling  11m    sample-scheduler  0/1 nodes are available: 1 Node: jcrose. preemption: 0/1 nodes are available: 1 No preemption victims found for incoming pod..
  Warning  FailedScheduling  5m40s  sample-scheduler  0/1 nodes are available: 1 Node: jcrose. preemption: 0/1 nodes are available: 1 No preemption victims found for incoming pod..


```

#### 恢复调度

下面我们实现让Pod 恢复正常调度的效果，我们给 `node1`添加一个 `cpu=true` 标签

```powershell
kubectl label nodes jcrose cpu=true


root@jcrose:~# kubectl get po
NAME                                       READY   STATUS    RESTARTS     AGE
calico-kube-controllers-7c6f5c46cc-vwdxh   1/1     Running   0            55m
calico-node-75xkf                          1/1     Running   0            55m
calico-typha-d76b77b7f-bxwlm               1/1     Running   0            55m
nginx-web-4vllh                            1/1     Running   0            5d2h
nginx-web-rmtff                            1/1     Running   1 (3d ago)   5d2h
sample-scheduler-77fd75dbcc-vprkc          1/1     Running   0            15m
test-scheduler-75b49bc547-f5rl2            1/1     Running   0            95s
```

可以发现现在已经可以成功调度

# 总结

当前示例非常的简单，主要是为了让大家方便理解。对于开发什么样的插件，只需要看对应的插件接口就可以了。然后在配置文件里在合适的扩展点启用即可。

对于自定义调度器的实现，是在main.go文件里通过

```go
app.NewSchedulerCommand(
    app.WithPlugin(plugin.Name, plugin.New),
)
```

来注册自定义调度器插件，然后再通过`--config` 指定插件是否启用以及启用的扩展点。

本文调度器是以Pod 方式运行，并在容器里挂载一个配置 `/etc/kubernetes/scheduler-config.yaml` ，也可以直接修改集群 `kube-scheduler` 的配置文件添加一个新调度器配置来实现，不过由于对集群侵入太大了，个人不推荐这种用法。
