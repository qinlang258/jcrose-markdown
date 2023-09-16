# 概述

通常情况下，网络一个 Node 在一段时间内使用的网络流量也是作为生产环境中很常见的情况。例如在配置均衡的多个主机中，主机 A 作为业务拉单脚本运行，主机 B 作为寻常服务运行。因为拉单需要下载大量数据，而硬件资源占用的却很少，此时，如果有 Pod 被调度到该节点上，那么可能双方业务都会收到影响（前端代理觉得这个节点连接数少会被大量调度，而拉单脚本因为网络带宽的占用降低了效能）。

项目所在根路径

```
root@jcrose:/usr/local/kubernetes-1.26.7/pkg/scheduler/framework/plugins/networtraffic# tree 
.
├── Dockerfile
├── main.go
├── networktraffic-scheduler
└── plugin
    ├── networktraffic.go
    └── prometheus.go
 
1 directory, 5 files
 
```

main.go

```
package main
 
import (
    "k8s.io/kubernetes/cmd/kube-scheduler/app"
    "k8s.io/kubernetes/pkg/scheduler/framework/plugins/networtraffic/plugin"
    "os"
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

Dockerfile

```
FROM  ubuntu:20.04
WORKDIR .
COPY networktraffic-scheduler /usr/local/bin
CMD ["networktraffic-scheduler"]
 
```

plugin/networktraffic.go

```
package plugin
 
import (
    "context"
    "fmt"
    "k8s.io/api/core/v1"
    "k8s.io/apimachinery/pkg/runtime"
    "k8s.io/klog"
    "k8s.io/kubernetes/pkg/scheduler/framework"
    fruntime "k8s.io/kubernetes/pkg/scheduler/framework/runtime"
    "time"
)
 
const Name = "network_traffic"
 
type NetworkTraffic struct {
    prometheus *PrometheusHandle
    handle     framework.Handle
}
 
type NetworkTrafficArgs struct {
    IP         string `json:"ip"`
    DeviceName string `json:"device_name"`
    TimeRange  int    `json:"time_range"`
}
 
func New(plArgs runtime.Object, h framework.Handle) (framework.Plugin, error) {
    args := NetworkTrafficArgs{}
    if err := fruntime.DecodeInto(plArgs, args); err != nil {
        return nil, err
    }
 
    klog.Infof("[NetworkTraffic] args received. Device: %s; TimeRange: %d, Address: %s", args.DeviceName, args.TimeRange, args.IP)
 
    return &NetworkTraffic{
        handle:     h,
        prometheus: NewProme(args.IP, args.DeviceName, time.Minute*time.Duration(args.TimeRange)),
    }, nil
}
 
func (n *NetworkTraffic) Name() string {
    return Name
}
 
func (n *NetworkTraffic) ScoreExtensions() framework.ScoreExtensions {
    return n
}
 
// NormalizeScore与ScoreExtensions是固定格式
func (n *NetworkTraffic) NormalizeScore(ctx context.Context, state *framework.CycleState, p *v1.Pod, scores framework.NodeScoreList) *framework.Status {
    var higherScores int64
    for _, node := range scores {
        if higherScores < node.Score {
            higherScores = node.Score
        }
    }
 
    // 计算公式为，满分 - (当前带宽 / 最高最高带宽 * 100)
    // 公式的计算结果为，带宽占用越大的机器，分数越低
    for i, node := range scores {
        scores[i].Score = framework.MaxNodeScore - (node.Score * 100 / higherScores)
        klog.Infof("[NetworkTraffic] Nodes final score: %v", scores)
    }
 
    klog.Infof("[NetworkTraffic] Nodes final score: %v", scores)
    return nil
}
 
func (n *NetworkTraffic) Score(ctx context.Context, state *framework.CycleState, p *v1.Pod, nodeName string) (int64, *framework.Status) {
    nodeBandwidth, err := n.prometheus.GetGauge(nodeName)
    if err != nil {
        return 0, framework.NewStatus(framework.Error, fmt.Sprintf("error getting node bandwidth measure: %s", err))
    }
 
    bandWidth := int64(nodeBandwidth.Value)
    klog.Infof("[NetworkTraffic] node '%s' bandwidth: %s", nodeName, bandWidth)
    return bandWidth, nil
}
 
 
```

plugin/prometheus.go

```
package plugin
 
import (
    "github.com/prometheus/client_golang/api"
    v1 "github.com/prometheus/client_golang/api/prometheus/v1"
    "github.com/prometheus/common/model"
    "k8s.io/klog"
 
    "context"
    "fmt"
    "time"
)
 
const (
    // nodeMeasureQueryTemplate is the template string to get the query for the node used bandwidth
    // nodeMeasureQueryTemplate = "sum_over_time(node_network_receive_bytes_total{device=\"%s\"}[%ss])"
    nodeMeasureQueryTemplate = "sum_over_time(node_network_receive_bytes_total{device=\"%s\"}[%ss]) * on(instance) group_left(nodename) (node_uname_info{nodename=\"%s\"})"
)
 
type PrometheusHandle struct {
    deviceName string
    timeRange  time.Duration
    ip         string
    client     v1.API
}
 
func NewProme(ip, deviceName string, timeRace time.Duration) *PrometheusHandle {
    client, err := api.NewClient(api.Config{Address: ip})
    if err != nil {
        klog.Fatalf("[NetworkTraffic Plugin] FatalError creating prometheus client: %s", err.Error())
    }
    return &PrometheusHandle{
        deviceName: deviceName,
        ip:         ip,
        timeRange:  timeRace,
        client:     v1.NewAPI(client),
    }
}
 
func (p *PrometheusHandle) GetGauge(node string) (*model.Sample, error) {
 
    value, err := p.query(fmt.Sprintf(nodeMeasureQueryTemplate, node, p.deviceName, p.timeRange))
    fmt.Println(fmt.Sprintf(nodeMeasureQueryTemplate, p.deviceName, p.timeRange, node))
    if err != nil {
        return nil, fmt.Errorf("[NetworkTraffic Plugin] Error querying prometheus: %w", err)
    }
 
    nodeMeasure := value.(model.Vector)
    if len(nodeMeasure) != 1 {
        return nil, fmt.Errorf("[NetworkTraffic Plugin] Invalid response, expected 1 value, got %d", len(nodeMeasure))
    }
    return nodeMeasure[0], nil
}
 
func (p *PrometheusHandle) query(promQL string) (model.Value, error) {
    results, warnings, err := p.client.Query(context.Background(), promQL, time.Now())
    if len(warnings) > 0 {
        klog.Warningf("[NetworkTraffic Plugin] Warnings: %v\n", warnings)
    }
 
    return results, err
}
 
 
```

    # 制作容器
    docker build -t registry.cn-zhangjiakou.aliyuncs.com/jcrose-k8s/networktraffic-scheduler:v0.0.1 .
    docker push registry.cn-zhangjiakou.aliyuncs.com/jcrose-k8s/networktraffic-scheduler:v0.0.1

scheduler.yaml

    ---
    apiVersion: v1
    kind: ServiceAccount
    metadata:
      name:  networktraffic-scheduler-sa
      namespace: kube-system
    ---
    kind: ClusterRoleBinding
    apiVersion: rbac.authorization.k8s.io/v1
    metadata:
      name:  networtraffic-clusterrolebinding
      namespace: kube-system
    roleRef:
      apiGroup: rbac.authorization.k8s.io
      kind: ClusterRole
      name:  system:kube-scheduler #直接使用scheduler的权限
    subjects:
      - kind: ServiceAccount
        name:  networktraffic-scheduler-sa
        namespace: kube-system
    ---
    apiVersion: v1
    kind: ConfigMap
    metadata:
      name: networktraffic-scheduler-config
      namespace: kube-system
    data:
      scheduler-config.yaml: |
        apiVersion: kubescheduler.config.k8s.io/v1
        kind: KubeSchedulerConfiguration
        leaderElection:
          leaderElect: false
        profiles:
          - schedulerName: networktraffic-scheduler
            plugins:
              score:
                enabled:
                  - name: network_traffic
            pluginConfig:
              - name: "NetworkTraffic"
                args:
                  ip: "prometheus.kube-system:9090"
                  deviceName: "eth0"
                  timeRange: 60
    ---
    apiVersion: apps/v1
    kind: Deployment
    metadata:
      name: networktraffic-scheduler
      namespace: kube-system
      labels:
        component: networktraffic-scheduler
    spec:
      selector:
        matchLabels:
          component: networktraffic-scheduler
      template:
        metadata:
          labels:
            component: networktraffic-scheduler
        spec:
          serviceAccountName: networktraffic-scheduler-sa
          priorityClassName: system-cluster-critical
          volumes:
            - name: networktraffic-scheduler-config
              configMap:
                name: networktraffic-scheduler-config
          containers:
            - name: networktraffic-scheduler
              image: registry.cn-zhangjiakou.aliyuncs.com/jcrose-k8s/networktraffic-scheduler:v0.0.14
              imagePullPolicy: IfNotPresent
              command:
                - networktraffic-scheduler
                - --config=/etc/kubernetes/scheduler-config.yaml
                - --v=2
              volumeMounts:
                - name: networktraffic-scheduler-config
                  mountPath: /etc/kubernetes

这个yaml 文件共完成以下几件事：

1.  通过 `ConfigMap` 声明一个 `KubeSchedulerConfiguration` 配置
2.  创建一个 `Deployment` 对象，其中容器镜像 registry.cn-zhangjiakou.aliyuncs.com/jcrose-k8s/ networtraffic\\\\\:v0.0.1 是前面我们开发的插件应用，对于调度器插件配置通过 `volume` 的方式存储到容器里 `/etc/kubernetes/scheduler-config.yaml`，应用启动时指定此配置文件；这里为了调试方便指定了日志 `--v=3` 等级
3.  创建一个`ClusterRole`，指定不同资源的访问权限
4.  创建一个`ServiceAccount`
5.  声明一个 `ClusterRoleBinding` 对象，绑定 `ClusterRole` 和 `ServiceAccount` 两者的关系

### 3.2 测试

创建一个pod，并指定调度器为 networktraffic-scheduler

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
          schedulerName: networktraffic-scheduler # 指定使用的调度器，不指定使用默认的default-scheduler
          containers:
            - image: nginx
              imagePullPolicy: IfNotPresent
              name: nginx
              ports:
                - containerPort: 80

```
➜ kubectl apply -f test-scheduler.yaml
 
```

由于插件主要实现对节点筛选，排除那些不能运行该 Pod 的节点，运行时将检查节点是否存在 `cpu=true` 标签，如果不存在这个label标签，则说明此节点无法通过预选阶段，后面的调度与绑定步骤就不可能执行。
