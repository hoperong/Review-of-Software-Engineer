# clusterpedia

Clusterpedia 可以在与 Kubernetes OpenAPI 兼容的基础上，实现多集群资源的同步，提供更强大的搜索功能，帮助您快速、轻松地有效获取您正在寻找的任何多集群资源。Clusterpedia 是一个云原生计算基础沙盒项目。Clusterpedia 可以作为独立平台部署，也可以与Cluster API、Karmada、Clusternet和其他多云平台集成

多云平台管理的集群自动同步
集群百科可以自动同步多云平台管理的集群内的资源。

用户无需手动维护 Clusterpedia，Clusterpedia 可以与多云平台的内部组件一样工作。

进一步了解与多云平台的接口

更多检索功能和与Kubernetes OpenAPI的兼容性
支持使用kubectl,client-go或controller-runtime/client, client-go 示例检索资源
可以通过 API 或client-go/metadata检索资源元数据
丰富的检索条件：按集群/命名空间/名称/创建过滤、按父或祖先所有者搜索、多集群标签选择器、增强字段选择器、自定义搜索条件等。
支持导入 Kubernetes 1.10+
自动转换不同版本的 Kube 资源，支持多版本资源
即使你导入不同版本的 Kube，我们仍然可以使用相同的资源版本来检索资源
例如，我们可以使用v1, v1beta2, v1beta1version 来检索不同集群中的 Deployments 资源。

注意：部署的版本v1beta1在 Kubernetes 1.10 中，v1在 Kubernetes 1.24 中。