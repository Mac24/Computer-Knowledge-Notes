## flannel源码解读

### 1.校验subnet-lease-renew-margin

```
    if opts.subnetLeaseRenewMargin >= 24*60 || opts.subnetLeaseRenewMargin <= 0 {
		log.Error("Invalid subnet-lease-renew-margin option, out of acceptable range")
		os.Exit(1)
	}
```

要求小于等于24h，大于0

### 2.找出使用的网络接口

通过`iface`和`ifaceRegex`两个参数来进行选取

- 如果`iface`和`ifaceRegex`都为空，则直接选取默认路由所使用的输出网卡

- 如果`iface`参数不为空，则依次遍历其中的各个实例，直到找到和该网卡名或IP匹配的实例为止

- 如果`ifaceRegex`参数不为空，操作方式和上述相同，唯一不同的是使用正则表达式去匹配

  最后，对于集群间交互的Public IP，我们同样可以通过启动参数”–public-ip”进行指定。否则，将使用上文中获取的网卡的IP作为Public IP。

  外部接口的定义如下：

  ```
  type ExternalInterface struct {
      Iface     *net.Interface
      IfaceAddr net.IP
      ExtAddr   net.IP
  }
  ```

### 3.创建SubnetManager

  ```
  func newSubnetManager() (subnet.Manager, error) {
      if opts.kubeSubnetMgr {
          return kube.NewSubnetManager(opts.kubeApiUrl, opts.kubeConfigFile)
      }
  
      cfg := &etcdv2.EtcdConfig{
          Endpoints: strings.Split(opts.etcdEndpoints, ","),
          Keyfile:   opts.etcdKeyfile,
          Certfile:  opts.etcdCertfile,
          CAFile:    opts.etcdCAFile,
          Prefix:    opts.etcdPrefix,
          Username:  opts.etcdUsername,
          Password:  opts.etcdPassword,
      }
  
      // Attempt to renew the lease for the subnet specified in the subnetFile
      prevSubnet := ReadSubnetFromSubnetFile(opts.subnetFile)
  
      return etcdv2.NewLocalManager(cfg, prevSubnet)
  }
  ```

  子网管理器负责子网的创建、更新、添加、删除、监听等，主要和 etcd 打交道,定义：

  ```
  type Manager interface {
      GetNetworkConfig(ctx context.Context) (*Config, error)
      AcquireLease(ctx context.Context, attrs *LeaseAttrs) (*Lease, error)
      RenewLease(ctx context.Context, lease *Lease) error
      WatchLease(ctx context.Context, sn ip.IP4Net, cursor interface{}) (LeaseWatchResult, error)
      WatchLeases(ctx context.Context, cursor interface{}) (LeaseWatchResult, error)
  
      Name() string
  }
  ```

  - RenewLease 续约。在lease到期之前，子网管理器调用该方法进行续约。
  - GetNetworkConfig 获取本机的subnet配置，进行一些初始化的工作。

  ### 4. 获取网络配置

  ```
  config, err := getConfig(ctx, sm)
      if err == errCanceled {
          wg.Wait()
          os.Exit(0)
      }
  ```

  这个配置主要是管理网络的配置，需要在flannel启动之前写到etcd中。例如：

  ```
  {
      "Network": "10.0.0.0/8",
      "SubnetLen": 20,
      "SubnetMin": "10.10.0.0",
      "SubnetMax": "10.99.0.0",
      "Backend": {
          "Type": "udp",
          "Port": 7890
      }
  }
  ```

  /coreos.com/network/config 保存着上面网络配置数据。

  详细解读一下：

  - SubnetLen表示每个主机分配的subnet大小，我们可以在初始化时对其指定，否则使用默认配置。在默认配置的情况下，如果集群的网络地址空间大于/24，则SubnetLen配置为24，否则它比集群网络地址空间小1，例如集群的大小为/25，则SubnetLen的大小为/26
  - SubnetMin是集群网络地址空间中最小的可分配的subnet，可以手动指定，否则默认配置为集群网络地址空间中第一个可分配的subnet。
  - SubnetMax表示最大可分配的subnet
  - BackendType为使用的backend的类型，如未指定，则默认为“udp”
  - Backend中会包含backend的附加信息，例如backend为vxlan时，其中会存储vtep设备的mac地址

  ### 5. 创建backend管理器

使用它来创建backend并使用它注册网络,然后执行run方法

  ```
  bm := backend.NewManager(ctx, sm, extIface)
      be, err := bm.GetBackend(config.BackendType)
      if err != nil {
          log.Errorf("Error fetching backend: %s", err)
          cancel()
          wg.Wait()
          os.Exit(1)
      }
  
      bn, err := be.RegisterNetwork(ctx, config)
      if err != nil {
          log.Errorf("Error registering network: %s", err)
          cancel()
          wg.Wait()
          os.Exit(1)
      }
  
  ...
  
  log.Info("Running backend.")
      wg.Add(1)
      go func() {
          bn.Run(ctx)
          wg.Done()
      }()
  ```

  backend管理器

  ```
  type manager struct {
      ctx      context.Context
      sm       subnet.Manager
      extIface *ExternalInterface
      mux      sync.Mutex
      active   map[string]Backend
      wg       sync.WaitGroup
  }
  ```

  主要是提供了GetBackend(backendType string) (Backend, error)方法，根据配置文件的设置backend标志，生产对应的backend。

  此处注意

  ```
  go func() {
          <-bm.ctx.Done()
  
          // TODO(eyakubovich): this obviosly introduces a race.
          // GetBackend() could get called while we are here.
          // Currently though, all backends' Run exit only
          // on shutdown
  
          bm.mux.Lock()
          delete(bm.active, betype)
          bm.mux.Unlock()
  
          bm.wg.Done()
      }()
  ```

  在生产backend以后，会启动一个协程，在flanneld退出运行之前，将会执行激活的backend map中删除操作。

  最后run方法：

  ```
  func (n *RouteNetwork) Run(ctx context.Context) {
      wg := sync.WaitGroup{}
  
      log.Info("Watching for new subnet leases")
      evts := make(chan []subnet.Event)
      wg.Add(1)
      go func() {
          subnet.WatchLeases(ctx, n.SM, n.SubnetLease, evts)
          wg.Done()
      }()
  
      n.routes = make([]netlink.Route, 0, 10)
      wg.Add(1)
      go func() {
          n.routeCheck(ctx)
          wg.Done()
      }()
  
      defer wg.Wait()
  
      for {
          select {
          case evtBatch := <-evts:
              n.handleSubnetEvents(evtBatch)
  
          case <-ctx.Done():
              return
          }
      }
  }
  ```

  run方法中主要是执行：

  - subnet 负责和 etcd 交互，把 etcd 中的信息转换为 flannel 的子网数据结构，并对 etcd 进行子网和网络的监听；
  - backend 接受 subnet 的监听事件，并做出对应的处理。

  事件主要是subnet.EventAdded和subnet.EventRemoved两个。

  添加子网事件发生时的处理步骤：检查参数是否正常，根据参数构建路由表项，把路由表项添加到主机，把路由表项添加到自己的数据结构中。

  删除子网事件发生时的处理步骤：检查参数是否正常，根据参数构建路由表项，把路由表项从主机删除，把路由表项从管理的数据结构中删除

