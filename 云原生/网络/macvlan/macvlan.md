# MacVlan

## 什么是MacVlan

## 数据包流转

## 虚拟网卡

## MacVlan运作过程

# 源码分析
## 添加
添加的时候，会先从调用cmdAdd开始
```
func cmdAdd(args *skel.CmdArgs) error {
  // 1.获取配置
	n, cniVersion, err := loadConf(args, args.Args)
  // 2.获取网络空间
	netns, err := ns.GetNS(args.Netns)
  // 3.创建一个Macvlan接口
  macvlanInterface, err := createMacvlan(n, args.IfName, netns)
  // 4.判断是否使用IPAM，如果有，则使用IPAM来获取IP。如果没有，则自动分配
	if isLayer3 {
    // 获取IPAM分配的网络信息
    r, err := ipam.ExecAdd(n.IPAM.Type, args.StdinData)
    ......
    // 把IPAM返回的数据流转换成CNI的格式
    ipamResult, err := current.NewResultFromResult(r)
    ......
    for _, ipc := range result.IPs {
			// All addresses apply to the container macvlan interface
			ipc.Interface = current.Int(0)
		}
    ......
    err = netns.Do(func(_ ns.NetNS) error {
      // 启用macvlan网路接口的arp通知
			_, _ = sysctl.Sysctl(fmt.Sprintf("net/ipv4/conf/%s/arp_notify", args.IfName), "1")
      // 配置网卡
			return ipam.ConfigureIface(args.IfName, result)
		})
  }else{
    ......
  }
}
```
Macvlan CNI 接受的网络信息格式
```
type Result struct {
	CNIVersion string         `json:"cniVersion,omitempty"`
	Interfaces []*Interface   `json:"interfaces,omitempty"`
	IPs        []*IPConfig    `json:"ips,omitempty"`
	Routes     []*types.Route `json:"routes,omitempty"`
	DNS        types.DNS      `json:"dns,omitempty"`
}

type IPConfig struct {
	// Index into Result structs Interfaces list
	Interface *int
	Address   net.IPNet
	Gateway   net.IP
}
```
配置网卡逻辑
```
func ConfigureIface(ifName string, res *current.Result) error {
  ......
	for _, ipc := range res.IPs {
    ......
    // 写IP
    addr := &netlink.Addr{IPNet: &ipc.Address, Label: ""}
    if err = netlink.AddrAdd(link, addr); err != nil {
			return fmt.Errorf("failed to add IP addr %v to %q: %v", ipc, ifName, err)
		}
    ......
    if err := netlink.LinkSetUp(link); err != nil {
		  return fmt.Errorf("failed to set %q UP: %v", ifName, err)
	  }
  }
  ......
  for _, r := range res.Routes {
		routeIsV4 := r.Dst.IP.To4() != nil
		gw := r.GW
		if gw == nil {
			if routeIsV4 && v4gw != nil {
				gw = v4gw
			} else if !routeIsV4 && v6gw != nil {
				gw = v6gw
			}
		}
    // 创建路由信息
		route := netlink.Route{
			Dst:       &r.Dst,
			LinkIndex: link.Attrs().Index,
			Gw:        gw,
		}
		if err = netlink.RouteAddEcmp(&route); err != nil {
			return fmt.Errorf("failed to add route '%v via %v dev %v': %v", r.Dst, gw, ifName, err)
		}
	}
  return nil
}
```