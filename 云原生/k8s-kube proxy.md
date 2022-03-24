# Kube Proxy

## 介绍

Kube-proxy 是 kubernetes 工作节点上的一个网络代理组件，运行在每个节点上。

Kube-proxy维护节点上的网络规则，实现了Kubernetes Service 概念的一部分 。它的作用是使发往 Service 的流量（通过ClusterIP和端口）负载均衡到正确的后端Pod。

kube-proxy 监听 API server 中 资源对象的变化情况，包括以下三种：
+ service
+ endpoint/endpointslices
+ node
然后根据监听资源变化操作代理后端来为服务配置负载均衡。

如果你的 kubernetes 使用EndpointSlice，那么kube-proxy会监听EndpointSlice，否则会监听Endpoint。

如果你启用了服务拓扑，那么 kube-proxy 也会监听 node 信息 。服务拓扑（Service Topology）可以让一个服务基于集群的 Node 拓扑进行流量路由。 例如，一个服务可以指定流量是被优先路由到一个和客户端在同一个 Node 或者在同一可用区域的端点。

## 源码简单分析

### 目录结构
```
kubernetes
|-cmd
    |-kube-proxy
        |-app
            |-conntrack.go
            |-init_others.go
            |-server_others.go //模式代码
            |-server.go //核心代码
        |-proxy.go  // 启动命令
```

### 源码

通过标准的k8s组件启动方式启动kube-proxy
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
通过查看opts.Run()，可以发现kube-proxy使用proxyServer.Run()去运行主逻辑，在proxyServer.Run()里面，使用informers，去监听service、endpoint（或者是EndpointSlices）、node（当）
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
可以看出，这是一个接口类型。kebu-proxy也是利用这个接口类型，来实现不同系统的适配与不同模式的提供。

kube-proxy模式:
+ userspace（很早版本默认）
+ iptables（默认）
+ ipvs
+ kernelspace
可以使用启动参数--proxy-mode来配置。

### userspace
在 userspace 模式下，kube-proxy 通过监听 K8s apiserver 获取关于 Service 和 Endpoint 的变化信息，在内存中维护一份从ClusterIP:Port 到后端 Endpoints 的映射关系，通过反向代理的形式，将收到的数据包转发给后端，并将后端返回的应答报文转发给客户端。该模式下，kube-proxy 会为每个 Service （每种协议，每个 Service IP，每个 Service Port）在宿主机上创建一个 Socket 套接字（监听端口随机）用于接收和转发 client 的请求。默认条件下，kube-proxy 采用 round-robin 算法从后端 Endpoint 列表中选择一个响应请求。

### iptables
kube-proxy 会监视 apiserver 对 Service 对象和 Endpoints 对象的添加和移除。对每个 Service，它会添加上 iptables 规则，从而捕获到达该 Service 的 clusterIP（虚拟 IP）和端口的请求，进而将请求重定向到 Service 的一组 backend 中的某一个 Pod 上面。

我们还可以使用 Pod readiness 探针 验证后端 Pod 可以正常工作，以便 iptables 模式下的 kube-proxy 仅看到测试正常的后端，这样做意味着可以避免将流量通过 kube-proxy 发送到已知失败的 Pod 中，所以对于线上的应用来说一定要做 readiness 探针。

iptables模式下，kube-proxy为Service后端的每个Pod创建对应的iptables规则，直接将发向Cluster IP的请求重定向到一个Pod的IP上。

该模式下kube-proxy不承担四层负载均衡器的角色，只负责创建iptables规则。该模式的优点在于较userspace模式效率更高，但是不能提供灵活的LB策略，当后端Pod不可用的时候无法进行重试。



### ipvs
IPVS是LVS的负载均衡模块，亦基于netfilter，但比iptables性能更高，具备更好的可扩展性，采用增量式更新，并可以保证 service 更新期间连接保持不断开



### kernelspace
kernelspace 也可以说是 winkernel 模式，因为它只应用于 Windows 平台。在 winkernel 模式下，kube-proxy 通过监听 K8s apiserver 获取关于 Service 和 Endpoint 的变化信息，然后通过 HNS (Host Network Service) 接口直接配置 Windows 内核的 LoadBalancer 以及 Endpoint ，实现基于 winkernel 的 Kubernetes Service 代理服务。

winkernel 模式相对 winuserspace 模式的改进与 iptables 模式相对 userspace 模式的改进类似，即避免了主动创建大量的 Socket 监听，也免去了频繁且低效的用户态-内核态数据拷贝，直接通过配置系统的 LoadBalancer 和 Endpoint 实现代理，性能上具有明显优势。


## iptables模式

## IPVS模式

