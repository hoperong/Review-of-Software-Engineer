# Kube-Proxy

## 介绍

Kube-Proxy 是 Kubernetes 工作节点上的一个网络代理组件，运行在每个节点上。

Kube-Proxy 维护节点上的网络规则，实现了 Kubernetes Service 概念的一部分 。它的作用是使发往 Service 的流量（通过ClusterIP和端口）负载均衡到正确的后端 Pod。

Kube-Proxy 监听 API Server 中 资源对象的变化情况，包括以下三种：
+ Service
+ Endpoint/EndpointSlices
+ Node
然后根据监听资源变化操作代理后端来为服务配置负载均衡。

如果你的 Kubernetes 使用 EndpointSlice，那么 Kube-Proxy 会监听 EndpointSlice，否则会监听 Endpoint。

如果你启用了服务拓扑，那么 Kube-Proxy 也会监听 Node 信息 。服务拓扑（Service Topology）可以让一个服务基于集群的 Node 拓扑进行流量路由。 例如，一个服务可以指定流量是被优先路由到一个和客户端在同一个 Node 或者在同一可用区域的端点。

## 源码简单分析

### 目录结构
```
kubernetes
|-cmd
|    |-kube-proxy
|        |-app
|            |-conntrack.go
|            |-init_others.go
|            |-server_others.go //模式代码
|            |-server.go //核心代码|
|        |-proxy.go  // 启动命令
|-pkg
    |-proxy
```

### 源码

通过标准的 Kubernetes 组件启动方式启动 Kube-Proxy
```
// app.go

func main() {
	command := app.NewProxyCommand()
	code := cli.Run(command)
	os.Exit(code)
}

// server.go

// NewProxyCommand creates a *cobra.Command object with default parameters
func NewProxyCommand() *cobra.Command {
	opts := NewOptions()

	cmd := &cobra.Command{
		Use: "kube-proxy",
		Long: `The Kubernetes network proxy runs on each node. This
reflects services as defined in the Kubernetes API on each node and can do simple
TCP, UDP, and SCTP stream forwarding or round robin TCP, UDP, and SCTP forwarding across a set of backends.
Service cluster IPs and ports are currently found through Docker-links-compatible
environment variables specifying ports opened by the service proxy. There is an optional
addon that provides cluster DNS for these cluster IPs. The user must create a service
with the apiserver API to configure the proxy.`,
		RunE: func(cmd *cobra.Command, args []string) error {
			verflag.PrintAndExitIfRequested()
			cliflag.PrintFlags(cmd.Flags())

			if err := initForOS(opts.WindowsService); err != nil {
				return fmt.Errorf("failed os init: %w", err)
			}

			if err := opts.Complete(); err != nil {
				return fmt.Errorf("failed complete: %w", err)
			}

			if err := opts.Validate(); err != nil {
				return fmt.Errorf("failed validate: %w", err)
			}

			if err := opts.Run(); err != nil {
				klog.ErrorS(err, "Error running ProxyServer")
				return err
			}

			return nil
		},
		Args: func(cmd *cobra.Command, args []string) error {
			for _, arg := range args {
				if len(arg) > 0 {
					return fmt.Errorf("%q does not take any arguments, got %q", cmd.CommandPath(), args)
				}
			}
			return nil
		},
	}

	var err error
	opts.config, err = opts.ApplyDefaults(opts.config)
	if err != nil {
		klog.ErrorS(err, "Unable to create flag defaults")
		// ACTION REQUIRED: Exit code changed from 255 to 1
		os.Exit(1)
	}

	fs := cmd.Flags()
	opts.AddFlags(fs)
	fs.AddGoFlagSet(goflag.CommandLine) // for --boot-id-file and --machine-id-file

	_ = cmd.MarkFlagFilename("config", "yaml", "yml", "json")

	return cmd
}
```
通过查看 opts.Run()，可以发现 Kube-Proxy 使用 proxyServer.Run() 去运行主逻辑，在 proxyServer.Run() 里面，使用 informers，去监听Service、Endpoint（或者是EndpointSlices）、Node
```
// server.go

// Make informers that filter out objects that want a non-default service proxy.
	informerFactory := informers.NewSharedInformerFactoryWithOptions(s.Client, s.ConfigSyncPeriod,
		informers.WithTweakListOptions(func(options *metav1.ListOptions) {
			options.LabelSelector = labelSelector.String()
		}))

	// Create configs (i.e. Watches for Services and Endpoints or EndpointSlices)
	// Note: RegisterHandler() calls need to happen before creation of Sources because sources
	// only notify on changes, and the initial update (on process start) may be lost if no handlers
	// are registered yet.
	serviceConfig := config.NewServiceConfig(informerFactory.Core().V1().Services(), s.ConfigSyncPeriod)
	serviceConfig.RegisterEventHandler(s.Proxier)
	go serviceConfig.Run(wait.NeverStop)

	if endpointsHandler, ok := s.Proxier.(config.EndpointsHandler); ok && !s.UseEndpointSlices {
		endpointsConfig := config.NewEndpointsConfig(informerFactory.Core().V1().Endpoints(), s.ConfigSyncPeriod)
		endpointsConfig.RegisterEventHandler(endpointsHandler)
		go endpointsConfig.Run(wait.NeverStop)
	} else {
		endpointSliceConfig := config.NewEndpointSliceConfig(informerFactory.Discovery().V1().EndpointSlices(), s.ConfigSyncPeriod)
		endpointSliceConfig.RegisterEventHandler(s.Proxier)
		go endpointSliceConfig.Run(wait.NeverStop)
	}

	// This has to start after the calls to NewServiceConfig and NewEndpointsConfig because those
	// functions must configure their shared informer event handlers first.
	informerFactory.Start(wait.NeverStop)

	if utilfeature.DefaultFeatureGate.Enabled(features.TopologyAwareHints) {
		// Make an informer that selects for our nodename.
		currentNodeInformerFactory := informers.NewSharedInformerFactoryWithOptions(s.Client, s.ConfigSyncPeriod,
			informers.WithTweakListOptions(func(options *metav1.ListOptions) {
				options.FieldSelector = fields.OneTermEqualSelector("metadata.name", s.NodeRef.Name).String()
			}))
		nodeConfig := config.NewNodeConfig(currentNodeInformerFactory.Core().V1().Nodes(), s.ConfigSyncPeriod)
		nodeConfig.RegisterEventHandler(s.Proxier)
		go nodeConfig.Run(wait.NeverStop)

		// This has to start after the calls to NewNodeConfig because that must
		// configure the shared informer event handler first.
		currentNodeInformerFactory.Start(wait.NeverStop)
	}
```
从中可以看出，核心主要在于s.Proxier这个对象，查看ProxyServer结构体
```
// server.go

type ProxyServer struct {
	Client                 clientset.Interface
	EventClient            v1core.EventsGetter
	IptInterface           utiliptables.Interface
	IpvsInterface          utilipvs.Interface
	IpsetInterface         utilipset.Interface
	execer                 exec.Interface
	Proxier                proxy.Provider
	Broadcaster            events.EventBroadcaster
	Recorder               events.EventRecorder
	ConntrackConfiguration kubeproxyconfig.KubeProxyConntrackConfiguration
	Conntracker            Conntracker // if nil, ignored
	ProxyMode              string
	NodeRef                *v1.ObjectReference
	MetricsBindAddress     string
	BindAddressHardFail    bool
	EnableProfiling        bool
	UseEndpointSlices      bool
	OOMScoreAdj            *int32
	ConfigSyncPeriod       time.Duration
	HealthzServer          healthcheck.ProxierHealthUpdater
}

// kubernetes/pkg/proxy/types.go

// Provider is the interface provided by proxier implementations.
type Provider interface {
	config.EndpointSliceHandler
	config.ServiceHandler
	config.NodeHandler

	// Sync immediately synchronizes the Provider's current state to proxy rules.
	Sync()
	// SyncLoop runs periodic work.
	// This is expected to run as a goroutine or as the main loop of the app.
	// It does not return.
	SyncLoop()
}

```
可以看出，这是一个接口类型。Kube-Proxy 也是利用这个接口类型，来实现不同系统的适配与不同模式的提供。

```
// server_others.go

func newProxyServer(
	config *proxyconfigapi.KubeProxyConfiguration,
	cleanupAndExit bool,
	master string) (*ProxyServer, error) {
        ...
        proxyMode := getProxyMode(string(config.Mode), canUseIPVS, iptables.LinuxKernelCompatTester{})
        ...
        if proxyMode == proxyModeIPTables {
            ...
        } else if proxyMode == proxyModeIPVS {
            ...
        } else {
            ...
        }
```
根据配置的模式不同，生成不同的 Proxier。

Kube-Proxy 模式:
+ userspace（很早版本默认）
+ iptables（默认）
+ ipvs
+ kernelspace
可以使用启动参数--proxy-mode来配置。

### userspace
在 userspace 模式下，kube-proxy 通过监听 K8s apiserver 获取关于 Service 和 Endpoint 的变化信息，在内存中维护一份从ClusterIP:Port 到后端 Endpoints 的映射关系，通过反向代理的形式，将收到的数据包转发给后端，并将后端返回的应答报文转发给客户端。该模式下，kube-proxy 会为每个 Service （每种协议，每个 Service IP，每个 Service Port）在宿主机上创建一个 Socket 套接字（监听端口随机）用于接收和转发 client 的请求。默认条件下，kube-proxy 采用 round-robin 算法从后端 Endpoint 列表中选择一个响应请求。

### iptables
在 iptables 模式下，kube-proxy 依然需要通过监听 K8s apiserver 获取关于 Service 和 Endpoint 的变化信息。不过与 userspace 模式不同的是，kube-proxy 不再为每个 Service 创建反向代理（也就是无需创建 Socket 监听），而是通过安装 iptables 规则，捕获访问 Service ClusterIP:Port 的流量，直接重定向到指定的 Endpoints 后端。默认条件下，kube-proxy 会 随机 从后端 Endpoint 列表中选择一个响应请求。ipatbles 模式与 userspace 模式的不同之处在于，数据包的转发不再通过 kube-proxy 在用户空间通过反向代理来做，而是基于 iptables/netfilter 在内核空间直接转发，避免了数据的来回拷贝，因此在性能上具有很大优势，而且也避免了大量宿主机端口被占用的问题。

但是将数据转发完全交给 iptables 来做也有个缺点，就是一旦选择的后端没有响应，连接就会直接失败了，而不会像 userspace 模式那样，反向代理可以支持自动重新选择后端重试，算是失去了一定的重试灵活性。不过，官方建议使用 Readiness 探针来解决这个问题，一旦检测到后端故障，就自动将其移出 Endpoint 列表，避免请求被代理到存在问题的后端。

### ipvs
在 ipvs 模式下，kube-proxy 通过监听 K8s apiserver 获取关于 Service 和 Endpoint 的变化信息，然后调用 netlink 接口创建 ipvs 规则，并周期性地同步 Kubernetes Service/Endpoint 与 ipvs 规则，保证状态匹配。当客户端请求 Service 时，ipvs 会将流量重定向到指定的后端。

ipvs 相比 iptables 的优势在于通过 hash table 实现了 O(1) 时间复杂度的规则匹配。当 Service 和 Endpoint 的数量急剧增加时，iptables 的匹配表项会变得十分庞大，而其采用的顺序匹配模式会严重影响其性能。相反，ipvs 无论在小规模集群还是大规模集群，都拥有几乎相同的性能表现，因此相比其他代理模式，能提供更高的网络吞吐量，更能满足大规模集群扩展性和高性能的需要。同时，ipvs 支持对后端的健康检查和连接重试，可靠性相比 iptables 更佳。此外，ipvs 模式还为用户提供了多种负载均衡策略以供选择，例如：

rr: round-robin (轮询，默认采用)
lc: least connection （最少连接数）
dh: destination hashing （根据目的哈希）
sh: source hashing （根据源哈希）
sed: shortest expected delay （最小延迟）
nq: never queue （不排队等待，有空闲 Server 直接分配给空闲 Server 处理，否则通过 sed 策略处理）

ipvs 仍然使用 iptables 实现对数据包的过滤、SNAT 或 MASQ，但使用 ipset 来存储需要执行 MASQ 或 DROP 的源地址和目的地址，从而保证在大量 Service 存在的情况下，iptables 表项数量仍能维持在常数级。

### kernelspace
kernelspace 也可以说是 winkernel 模式，因为它只应用于 Windows 平台。在 winkernel 模式下，kube-proxy 通过监听 K8s apiserver 获取关于 Service 和 Endpoint 的变化信息，然后通过 HNS (Host Network Service) 接口直接配置 Windows 内核的 LoadBalancer 以及 Endpoint ，实现基于 winkernel 的 Kubernetes Service 代理服务。

winkernel 模式相对 winuserspace 模式的改进与 iptables 模式相对 userspace 模式的改进类似，即避免了主动创建大量的 Socket 监听，也免去了频繁且低效的用户态-内核态数据拷贝，直接通过配置系统的 LoadBalancer 和 Endpoint 实现代理，性能上具有明显优势。

## iptables模式
```
// server_others.go

if proxyMode == proxyModeIPTables {
    klog.V(0).InfoS("Using iptables Proxier")
    if config.IPTables.MasqueradeBit == nil {
        // MasqueradeBit must be specified or defaulted.
        return nil, fmt.Errorf("unable to read IPTables MasqueradeBit from config")
    }

    if dualStack {
        klog.V(0).InfoS("kube-proxy running in dual-stack mode", "ipFamily", iptInterface.Protocol())
        klog.V(0).InfoS("Creating dualStackProxier for iptables")
        // Always ordered to match []ipt
        var localDetectors [2]proxyutiliptables.LocalTrafficDetector
        localDetectors, err = getDualStackLocalDetectorTuple(detectLocalMode, config, ipt, nodeInfo)
        if err != nil {
            return nil, fmt.Errorf("unable to create proxier: %v", err)
        }

        // TODO this has side effects that should only happen when Run() is invoked.
        proxier, err = iptables.NewDualStackProxier(
            ipt,
            utilsysctl.New(),
            execer,
            config.IPTables.SyncPeriod.Duration,
            config.IPTables.MinSyncPeriod.Duration,
            config.IPTables.MasqueradeAll,
            int(*config.IPTables.MasqueradeBit),
            localDetectors,
            hostname,
            nodeIPTuple(config.BindAddress),
            recorder,
            healthzServer,
            config.NodePortAddresses,
        )
    } else {
        // Create a single-stack proxier if and only if the node does not support dual-stack (i.e, no iptables support).
        var localDetector proxyutiliptables.LocalTrafficDetector
        localDetector, err = getLocalDetector(detectLocalMode, config, iptInterface, nodeInfo)
        if err != nil {
            return nil, fmt.Errorf("unable to create proxier: %v", err)
        }

        // TODO this has side effects that should only happen when Run() is invoked.
        proxier, err = iptables.NewProxier(
            iptInterface,
            utilsysctl.New(),
            execer,
            config.IPTables.SyncPeriod.Duration,
            config.IPTables.MinSyncPeriod.Duration,
            config.IPTables.MasqueradeAll,
            int(*config.IPTables.MasqueradeBit),
            localDetector,
            hostname,
            nodeIP,
            recorder,
            healthzServer,
            config.NodePortAddresses,
        )
    }

    if err != nil {
        return nil, fmt.Errorf("unable to create proxier: %v", err)
    }
    proxymetrics.RegisterMetrics()
}
```
iptables.NewProxier在kubernetes/pkg/proxy/iptables/proxier.go里面定义。通过上面使用，去监听service、endpoint，可以知道，主要的逻辑就是针对监听的回调函数操作。
```
// kubernetes/pkg/proxy/iptables/proxier.go

// OnServiceAdd is called whenever creation of new service object
// is observed.
func (proxier *Proxier) OnServiceAdd(service *v1.Service) {
	proxier.OnServiceUpdate(nil, service)
}

// OnServiceUpdate is called whenever modification of an existing
// service object is observed.
func (proxier *Proxier) OnServiceUpdate(oldService, service *v1.Service) {
	if proxier.serviceChanges.Update(oldService, service) && proxier.isInitialized() {
		proxier.Sync()
	}
}

// OnServiceDelete is called whenever deletion of an existing service
// object is observed.
func (proxier *Proxier) OnServiceDelete(service *v1.Service) {
	proxier.OnServiceUpdate(service, nil)

}

// OnServiceSynced is called once all the initial event handlers were
// called and the state is fully propagated to local cache.
func (proxier *Proxier) OnServiceSynced() {
	proxier.mu.Lock()
	proxier.servicesSynced = true
	proxier.setInitialized(proxier.endpointSlicesSynced)
	proxier.mu.Unlock()

	// Sync unconditionally - this is called once per lifetime.
	proxier.syncProxyRules()
}

// OnEndpointSliceAdd is called whenever creation of a new endpoint slice object
// is observed.
func (proxier *Proxier) OnEndpointSliceAdd(endpointSlice *discovery.EndpointSlice) {
	if proxier.endpointsChanges.EndpointSliceUpdate(endpointSlice, false) && proxier.isInitialized() {
		proxier.Sync()
	}
}

// OnEndpointSliceUpdate is called whenever modification of an existing endpoint
// slice object is observed.
func (proxier *Proxier) OnEndpointSliceUpdate(_, endpointSlice *discovery.EndpointSlice) {
	if proxier.endpointsChanges.EndpointSliceUpdate(endpointSlice, false) && proxier.isInitialized() {
		proxier.Sync()
	}
}

// OnEndpointSliceDelete is called whenever deletion of an existing endpoint slice
// object is observed.
func (proxier *Proxier) OnEndpointSliceDelete(endpointSlice *discovery.EndpointSlice) {
	if proxier.endpointsChanges.EndpointSliceUpdate(endpointSlice, true) && proxier.isInitialized() {
		proxier.Sync()
	}
}
```
可以看出，无论是add、update、delete，都会放进一个update的方法里面，update方法有两个参数，一个old_object，一个new_object，add就会把object放进new_object里面，delete就会把object放进old_object里面，update则同时放入old_object、new_object里面。并且可以看出，更新完后，他会去做一个同步操作。


```
// proxier.serviceChanges.Update(oldService, service) 
// kubernetes/pkg/proxy/service.go

// Update updates given service's change map based on the <previous, current> service pair.  It returns true if items changed,
// otherwise return false.  Update can be used to add/update/delete items of ServiceChangeMap.  For example,
// Add item
//   - pass <nil, service> as the <previous, current> pair.
// Update item
//   - pass <oldService, service> as the <previous, current> pair.
// Delete item
//   - pass <service, nil> as the <previous, current> pair.
func (sct *ServiceChangeTracker) Update(previous, current *v1.Service) bool {
	svc := current
	if svc == nil {
		svc = previous
	}
	// previous == nil && current == nil is unexpected, we should return false directly.
	if svc == nil {
		return false
	}
	metrics.ServiceChangesTotal.Inc()
	namespacedName := types.NamespacedName{Namespace: svc.Namespace, Name: svc.Name}

	sct.lock.Lock()
	defer sct.lock.Unlock()

	change, exists := sct.items[namespacedName]
	if !exists {
		change = &serviceChange{}
		change.previous = sct.serviceToServiceMap(previous)
		sct.items[namespacedName] = change
	}
	change.current = sct.serviceToServiceMap(current)
	// if change.previous equal to change.current, it means no change
	if reflect.DeepEqual(change.previous, change.current) {
		delete(sct.items, namespacedName)
	} else {
		klog.V(2).InfoS("Service updated ports", "service", klog.KObj(svc), "portCount", len(change.current))
	}
	metrics.ServiceChangesPending.Set(float64(len(sct.items)))
	return len(sct.items) > 0
}
```
可以看出来update方法，只是把service的信息整理构建成新得数据结构后，存放在items里面，并没有说去操作iptables。

看刚才的同步函数，只是启动了proxier.syncRunner，不过从Proxier的初始化中，可以看出，proxier.syncRunner绑定了一个回调方法proxier.syncProxyRules。
```
// kubernetes/pkg/proxy/iptables/proxier.go

// Sync is called to synchronize the proxier state to iptables as soon as possible.
func (proxier *Proxier) Sync() {
	if proxier.healthzServer != nil {
		proxier.healthzServer.QueuedUpdate()
	}
	metrics.SyncProxyRulesLastQueuedTimestamp.SetToCurrentTime()
	proxier.syncRunner.Run()
}

func NewProxier{
    ...
    proxier.syncRunner = async.NewBoundedFrequencyRunner("sync-runner", proxier.syncProxyRules, minSyncPeriod, time.Hour, burstSyncs)
    ...
}
```
通过看proxier.syncProxyRules方法，基本上关于iptables的操作都在里面了，并且也可以基本确定了kube-proxy的运行逻辑。通过informers监听service、endpoint资源的变化，然后构建一个items作为缓存，整理一段时间内的变化，最后通过一个定时任务，把缓存一口气更新到iptables里面。

接下来观察proxier.syncProxyRules，可以看到，kube-proxy会先确保创建好所需要的chain，然后根据service的信息与service目前是否存在endpoint，来对应向不同的chain里面写入不同的信息。

### ClusterIP
```
// iptables -S -t nat | grep dao-2048

-A KUBE-SERVICES -d 172.31.214.223/32 -p tcp -m comment --comment "default/dao-2048: cluster IP" -m tcp --dport 80 -j KUBE-MARK-MASQ
-A KUBE-SERVICES -d 172.31.214.223/32 -p tcp -m comment --comment "default/dao-2048: cluster IP" -m tcp --dport 80 -j KUBE-SVC-LXOEKJ2ZQE3MR4LO
-A KUBE-SVC-LXOEKJ2ZQE3MR4LO -m comment --comment "default/dao-2048:" -m statistic --mode random --probability 0.33333333349 -j KUBE-SEP-EJFH32X3YSSCOZZG
-A KUBE-SVC-LXOEKJ2ZQE3MR4LO -m comment --comment "default/dao-2048:" -m statistic --mode random --probability 0.50000000000 -j KUBE-SEP-J7YAF3N4OSYK74E3
-A KUBE-SVC-LXOEKJ2ZQE3MR4LO -m comment --comment "default/dao-2048:" -j KUBE-SEP-ERKGB5P7AYO7SEF4
-A KUBE-SEP-ERKGB5P7AYO7SEF4 -s 172.29.248.226/32 -m comment --comment "default/dao-2048:" -j KUBE-MARK-MASQ
-A KUBE-SEP-ERKGB5P7AYO7SEF4 -p tcp -m comment --comment "default/dao-2048:" -m tcp -j DNAT --to-destination 172.29.248.226:80
-A KUBE-SEP-J7YAF3N4OSYK74E3 -s 172.29.140.75/32 -m comment --comment "default/dao-2048:" -j KUBE-MARK-MASQ
-A KUBE-SEP-J7YAF3N4OSYK74E3 -p tcp -m comment --comment "default/dao-2048:" -m tcp -j DNAT --to-destination 172.29.140.75:80
-A KUBE-SEP-EJFH32X3YSSCOZZG -s 172.29.114.58/32 -m comment --comment "default/dao-2048:" -j KUBE-MARK-MASQ
-A KUBE-SEP-EJFH32X3YSSCOZZG -p tcp -m comment --comment "default/dao-2048:" -m tcp -j DNAT --to-destination 172.29.114.58:80
```
创建一个ClusterIP服务，有3个实例，查看iptables，可以看到kube-proxy对应创建了许多规则。
1. 会在KUBE-SERVICES下创建ClusterIP+端口的规则，把接收到的转发到“KUBE-SVC-”开头的规则下
2. 会根据endpoint的数量创建以“KUBE-SVC-”开头的规则，把流量再转给以“KUBE-SEP-”开头的规则
3. “KUBE-SEP-”开头的规则才会实际把流量转到每个pod里面

### NodePort
```
// iptables -S -t nat | grep dao-2048

-A KUBE-SERVICES -d 172.31.214.223/32 -p tcp -m comment --comment "default/dao-2048: cluster IP" -m tcp --dport 80 -j KUBE-MARK-MASQ
-A KUBE-SERVICES -d 172.31.214.223/32 -p tcp -m comment --comment "default/dao-2048: cluster IP" -m tcp --dport 80 -j KUBE-SVC-LXOEKJ2ZQE3MR4LO
-A KUBE-NODEPORTS -p tcp -m comment --comment "default/dao-2048:" -m tcp --dport 31180 -j KUBE-MARK-MASQ
-A KUBE-NODEPORTS -p tcp -m comment --comment "default/dao-2048:" -m tcp --dport 31180 -j KUBE-SVC-LXOEKJ2ZQE3MR4LO
-A KUBE-SVC-LXOEKJ2ZQE3MR4LO -m comment --comment "default/dao-2048:" -m statistic --mode random --probability 0.33333333349 -j KUBE-SEP-EJFH32X3YSSCOZZG
-A KUBE-SVC-LXOEKJ2ZQE3MR4LO -m comment --comment "default/dao-2048:" -m statistic --mode random --probability 0.50000000000 -j KUBE-SEP-J7YAF3N4OSYK74E3
-A KUBE-SVC-LXOEKJ2ZQE3MR4LO -m comment --comment "default/dao-2048:" -j KUBE-SEP-ERKGB5P7AYO7SEF4
-A KUBE-SEP-ERKGB5P7AYO7SEF4 -s 172.29.248.226/32 -m comment --comment "default/dao-2048:" -j KUBE-MARK-MASQ
-A KUBE-SEP-ERKGB5P7AYO7SEF4 -p tcp -m comment --comment "default/dao-2048:" -m tcp -j DNAT --to-destination 172.29.248.226:80
-A KUBE-SEP-J7YAF3N4OSYK74E3 -s 172.29.140.75/32 -m comment --comment "default/dao-2048:" -j KUBE-MARK-MASQ
-A KUBE-SEP-J7YAF3N4OSYK74E3 -p tcp -m comment --comment "default/dao-2048:" -m tcp -j DNAT --to-destination 172.29.140.75:80
-A KUBE-SEP-EJFH32X3YSSCOZZG -s 172.29.114.58/32 -m comment --comment "default/dao-2048:" -j KUBE-MARK-MASQ
-A KUBE-SEP-EJFH32X3YSSCOZZG -p tcp -m comment --comment "default/dao-2048:" -m tcp -j DNAT --to-destination 172.29.114.58:80
```
创建一个NodePort服务，有3个实例，查看iptables，可以看到kube-proxy对应创建了许多规则。
1. 会在KUBE-NODEPORTS下创建针对nodeport的31180的规则，把接收到的转发到“KUBE-SVC-”开头的规则下
2. 同时也会在KUBE-SERVICES下创建ClusterIP+端口的规则，把接收到的转发到“KUBE-SVC-”开头的规则下
3. 会根据endpoint的数量创建以“KUBE-SVC-”开头的规则，把流量再转给以“KUBE-SEP-”开头的规则
4. “KUBE-SEP-”开头的规则才会实际把流量转到每个pod里面