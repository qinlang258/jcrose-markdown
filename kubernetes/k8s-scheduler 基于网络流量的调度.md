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

## 方法一 在源码所在地进行制作

注意 **k8s.io/kubernetes/pkg/scheduler/framework/plugins/networtraffic/plugin 这个位置是模仿k8s.io的引用路径进行修改**


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


## 方法二 使用自建的go.mod
注意点需要注意的是 kubernetes 包是不允许被外部调用的，所以go mod tidy直接下载是下不下来的。

**To use Kubernetes code as a library in other applications, see the list of published components. Use of the k8s.io/kubernetes module or k8s.io/kubernetes/... packages as libraries is not supported.
要将 Kubernetes 代码用作其他应用程序中的库，请参阅已发布组件列表。k8s.io/kubernetes不支持将模块或包用作k8s.io/kubernetes/...库。**

需要自行配置go.mod文件，主要是配置 replace   
```powershell
module scheduler-demo

go 1.20

require (
	github.com/360EntSecGroup-Skylar/excelize v1.4.1
	github.com/caoyingjunz/pixiulib v1.0.0
	github.com/gin-gonic/gin v1.9.1
	github.com/manifoldco/promptui v0.9.0
	k8s.io/api v0.28.0
	k8s.io/apimachinery v0.28.0
	k8s.io/client-go v0.28.0
	k8s.io/component-base v0.28.0
	k8s.io/klog v1.0.0
	k8s.io/klog/v2 v2.100.1
	k8s.io/kubernetes v1.28.0
)

require (
	github.com/Azure/go-ansiterm v0.0.0-20210617225240-d185dfc1b5a1 // indirect
	github.com/NYTimes/gziphandler v1.1.1 // indirect
	github.com/antlr/antlr4/runtime/Go/antlr/v4 v4.0.0-20230305170008-8188dc5388df // indirect
	github.com/asaskevich/govalidator v0.0.0-20190424111038-f61b66f89f4a // indirect
	github.com/beorn7/perks v1.0.1 // indirect
	github.com/blang/semver/v4 v4.0.0 // indirect
	github.com/bytedance/sonic v1.9.1 // indirect
	github.com/cenkalti/backoff/v4 v4.2.1 // indirect
	github.com/cespare/xxhash/v2 v2.2.0 // indirect
	github.com/chenzhuoyu/base64x v0.0.0-20221115062448-fe3a3abad311 // indirect
	github.com/chzyer/readline v0.0.0-20180603132655-2972be24d48e // indirect
	github.com/coreos/go-semver v0.3.1 // indirect
	github.com/coreos/go-systemd/v22 v22.5.0 // indirect
	github.com/davecgh/go-spew v1.1.1 // indirect
	github.com/docker/distribution v2.8.2+incompatible // indirect
	github.com/emicklei/go-restful/v3 v3.9.0 // indirect
	github.com/evanphx/json-patch v5.6.0+incompatible // indirect
	github.com/felixge/httpsnoop v1.0.3 // indirect
	github.com/fsnotify/fsnotify v1.6.0 // indirect
	github.com/gabriel-vasile/mimetype v1.4.2 // indirect
	github.com/gin-contrib/sse v0.1.0 // indirect
	github.com/go-logr/logr v1.2.4 // indirect
	github.com/go-logr/stdr v1.2.2 // indirect
	github.com/go-openapi/jsonpointer v0.19.6 // indirect
	github.com/go-openapi/jsonreference v0.20.2 // indirect
	github.com/go-openapi/swag v0.22.3 // indirect
	github.com/go-playground/locales v0.14.1 // indirect
	github.com/go-playground/universal-translator v0.18.1 // indirect
	github.com/go-playground/validator/v10 v10.14.0 // indirect
	github.com/goccy/go-json v0.10.2 // indirect
	github.com/gogo/protobuf v1.3.2 // indirect
	github.com/golang/groupcache v0.0.0-20210331224755-41bb18bfe9da // indirect
	github.com/golang/protobuf v1.5.3 // indirect
	github.com/google/cel-go v0.16.0 // indirect
	github.com/google/gnostic-models v0.6.8 // indirect
	github.com/google/go-cmp v0.5.9 // indirect
	github.com/google/gofuzz v1.2.0 // indirect
	github.com/google/uuid v1.3.0 // indirect
	github.com/grpc-ecosystem/go-grpc-prometheus v1.2.0 // indirect
	github.com/grpc-ecosystem/grpc-gateway/v2 v2.7.0 // indirect
	github.com/imdario/mergo v0.3.6 // indirect
	github.com/inconshreveable/mousetrap v1.1.0 // indirect
	github.com/josharian/intern v1.0.0 // indirect
	github.com/json-iterator/go v1.1.12 // indirect
	github.com/klauspost/cpuid/v2 v2.2.4 // indirect
	github.com/leodido/go-urn v1.2.4 // indirect
	github.com/mailru/easyjson v0.7.7 // indirect
	github.com/mattn/go-isatty v0.0.19 // indirect
	github.com/matttproud/golang_protobuf_extensions v1.0.4 // indirect
	github.com/moby/sys/mountinfo v0.6.2 // indirect
	github.com/moby/term v0.0.0-20221205130635-1aeaba878587 // indirect
	github.com/modern-go/concurrent v0.0.0-20180306012644-bacd9c7ef1dd // indirect
	github.com/modern-go/reflect2 v1.0.2 // indirect
	github.com/mohae/deepcopy v0.0.0-20170929034955-c48cc78d4826 // indirect
	github.com/munnerz/goautoneg v0.0.0-20191010083416-a7dc8b61c822 // indirect
	github.com/opencontainers/go-digest v1.0.0 // indirect
	github.com/opencontainers/selinux v1.10.0 // indirect
	github.com/pelletier/go-toml/v2 v2.0.8 // indirect
	github.com/pkg/errors v0.9.1 // indirect
	github.com/prometheus/client_golang v1.16.0 // indirect
	github.com/prometheus/client_model v0.4.0 // indirect
	github.com/prometheus/common v0.44.0 // indirect
	github.com/prometheus/procfs v0.10.1 // indirect
	github.com/spf13/cobra v1.7.0 // indirect
	github.com/spf13/pflag v1.0.5 // indirect
	github.com/stoewer/go-strcase v1.2.0 // indirect
	github.com/twitchyliquid64/golang-asm v0.15.1 // indirect
	github.com/ugorji/go/codec v1.2.11 // indirect
	go.etcd.io/etcd/api/v3 v3.5.9 // indirect
	go.etcd.io/etcd/client/pkg/v3 v3.5.9 // indirect
	go.etcd.io/etcd/client/v3 v3.5.9 // indirect
	go.opentelemetry.io/contrib/instrumentation/google.golang.org/grpc/otelgrpc v0.35.0 // indirect
	go.opentelemetry.io/contrib/instrumentation/net/http/otelhttp v0.35.1 // indirect
	go.opentelemetry.io/otel v1.10.0 // indirect
	go.opentelemetry.io/otel/exporters/otlp/internal/retry v1.10.0 // indirect
	go.opentelemetry.io/otel/exporters/otlp/otlptrace v1.10.0 // indirect
	go.opentelemetry.io/otel/exporters/otlp/otlptrace/otlptracegrpc v1.10.0 // indirect
	go.opentelemetry.io/otel/metric v0.31.0 // indirect
	go.opentelemetry.io/otel/sdk v1.10.0 // indirect
	go.opentelemetry.io/otel/trace v1.10.0 // indirect
	go.opentelemetry.io/proto/otlp v0.19.0 // indirect
	go.uber.org/atomic v1.10.0 // indirect
	go.uber.org/multierr v1.11.0 // indirect
	go.uber.org/zap v1.19.0 // indirect
	golang.org/x/arch v0.3.0 // indirect
	golang.org/x/crypto v0.11.0 // indirect
	golang.org/x/exp v0.0.0-20220722155223-a9213eeb770e // indirect
	golang.org/x/net v0.13.0 // indirect
	golang.org/x/oauth2 v0.8.0 // indirect
	golang.org/x/sync v0.2.0 // indirect
	golang.org/x/sys v0.10.0 // indirect
	golang.org/x/term v0.10.0 // indirect
	golang.org/x/text v0.11.0 // indirect
	golang.org/x/time v0.3.0 // indirect
	google.golang.org/appengine v1.6.7 // indirect
	google.golang.org/genproto v0.0.0-20230526161137-0005af68ea54 // indirect
	google.golang.org/genproto/googleapis/api v0.0.0-20230525234035-dd9d682886f9 // indirect
	google.golang.org/genproto/googleapis/rpc v0.0.0-20230525234030-28d5490b6b19 // indirect
	google.golang.org/grpc v1.54.0 // indirect
	google.golang.org/protobuf v1.30.0 // indirect
	gopkg.in/inf.v0 v0.9.1 // indirect
	gopkg.in/natefinch/lumberjack.v2 v2.2.1 // indirect
	gopkg.in/yaml.v2 v2.4.0 // indirect
	gopkg.in/yaml.v3 v3.0.1 // indirect
	k8s.io/apiextensions-apiserver v0.28.0 // indirect
	k8s.io/apiserver v0.28.0 // indirect
	k8s.io/cloud-provider v0.0.0 // indirect
	k8s.io/component-helpers v0.28.0 // indirect
	k8s.io/controller-manager v0.28.0 // indirect
	k8s.io/csi-translation-lib v0.0.0 // indirect
	k8s.io/dynamic-resource-allocation v0.0.0 // indirect
	k8s.io/kms v0.28.0 // indirect
	k8s.io/kube-openapi v0.0.0-20230717233707-2695361300d9 // indirect
	k8s.io/kube-scheduler v0.28.0 // indirect
	k8s.io/kubelet v0.28.0 // indirect
	k8s.io/mount-utils v0.0.0 // indirect
	k8s.io/utils v0.0.0-20230406110748-d93618cff8a2 // indirect
	sigs.k8s.io/apiserver-network-proxy/konnectivity-client v0.1.2 // indirect
	sigs.k8s.io/json v0.0.0-20221116044647-bc3834ca7abd // indirect
	sigs.k8s.io/structured-merge-diff/v4 v4.2.3 // indirect
	sigs.k8s.io/yaml v1.3.0 // indirect
)

replace (
	k8s.io/apiextensions-apiserver => k8s.io/apiextensions-apiserver v0.28.0
	k8s.io/apiserver => k8s.io/apiserver v0.28.0
	k8s.io/cli-runtime => k8s.io/cli-runtime v0.28.0
	k8s.io/cloud-provider => k8s.io/cloud-provider v0.28.0
	k8s.io/cluster-bootstrap => k8s.io/cluster-bootstrap v0.28.0
	k8s.io/component-base => k8s.io/component-base v0.28.0
	k8s.io/component-helpers => k8s.io/component-helpers v0.28.0
	k8s.io/controller-manager => k8s.io/controller-manager v0.28.0
	k8s.io/cri-api => k8s.io/cri-api v0.28.0
	k8s.io/csi-translation-lib => k8s.io/csi-translation-lib v0.28.0
	k8s.io/dynamic-resource-allocation => k8s.io/dynamic-resource-allocation v0.28.0
	k8s.io/endpointslice => k8s.io/endpointslice v0.28.0
	k8s.io/kube-aggregator => k8s.io/kube-aggregator v0.28.0
	k8s.io/kube-controller-manager => k8s.io/kube-controller-manager v0.28.0
	k8s.io/kube-proxy => k8s.io/kube-proxy v0.28.0
	k8s.io/kube-scheduler => k8s.io/kube-scheduler v0.28.0
	k8s.io/kubectl => k8s.io/kubectl v0.28.0
	k8s.io/kubelet => k8s.io/kubelet v0.28.0
	k8s.io/kubernetes => k8s.io/kubernetes v1.28.0
	k8s.io/legacy-cloud-providers => k8s.io/legacy-cloud-providers v0.28.0
	k8s.io/metrics => k8s.io/metrics v0.28.0
	k8s.io/mount-utils => k8s.io/mount-utils v0.28.0
	k8s.io/pod-security-admission => k8s.io/pod-security-admission v0.28.0
	k8s.io/sample-apiserver => k8s.io/sample-apiserver v0.28.0
)
```

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
