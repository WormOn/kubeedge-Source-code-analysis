# kubeedge源码分析系列之edgecore（六）

本系列的源码分析是在 commit da92692baa660359bb314d89dfa3a80bffb1d26c 之上进行的。

cloudcore部分的源码分析是在[kubeedge源码分析系列之整体架构](https://juejin.im/post/5dc92c66f265da4d513359ab)基础上展开的，如果没有阅读过[kubeedge源码分析系列之整体架构](https://juejin.im/post/5dc92c66f265da4d513359ab)，直接阅读本文，会感觉比较突兀。

## 本文概述

本文对edgecore的edgemesh模块进行剖析，kubeedge官网目前没有edgemesh相关介绍，根据华为近期的边缘计算视频分享课程获得edgemsh的相关信息。edgemesh作为edgecore中节点级别的代理解决方案，主要实现了节点内的流量代理、节点间的流量代理和节点内的DNS解析3块功能，本文就来剖析这三块功能的具体实现：

1. edgemesh Struct组成及注册
2. edgemesh业务逻辑剖析

## edgemesh Struct组成及注册

从edgemesh模块的注册函数入手：

kubeedge/edgemesh/pkg/module.go

	// Register register edgemesh
	func Register() {
		core.Register(&EdgeMesh{})
	}
注册函数只干了一件事，就是将实例化的edgemesh struct加入到一个全局map中去，进入EdgeMesh struct的定义：

kubeedge/edgemesh/pkg/module.go

	//EdgeMesh defines EdgeMesh object structure
	type EdgeMesh struct {
		context *context.Context
	}
EdgeMesh struct的定义比较简单，只有context一个属性，该属性用来与edgecore中的其他模块进行通信，具体可以参考[kubeedge源码分析系列之cloudcore](https://juejin.im/post/5dcbcd50e51d451bdf43a3a8)中的“cloudcore功能模块之间通信的消息框架”部分，本文就不对其进行展开剖析。

## edgemesh业务逻辑剖析

从edgemesh模块的启动函数入手：

kubeedge/edgemesh/pkg/module.go

	//Start sets context and starts the controller
	func (em *EdgeMesh) Start(c *context.Context) {
		em.context = c
		proxy.Init()
		go server.Start()
		// we need watch message to update the cache of instances
		for {
			if msg, ok := em.context.Receive(constant.ModuleNameEdgeMesh); ok == nil {
				proxy.MsgProcess(msg)
				klog.Infof("get message: %v", msg)
				continue
			}
		}
	}
	
启动函数Start(...)做了如下几件事：

1. 接收并保存通信管道

		em.context = c
		
2. 初始化porxy

		proxy.Init()
		
3. 启动服务

		go server.Start()
		
4. 通过一个for循环接收通信管道中关于edgemesh的信息并处理

	for {...}

下面对 “2. 初始化porxy”、“3. 启动服务”和“4. 通过一个for循环接收通信管道中关于edgemesh的信息并处理”展开剖析：

### 初始化porxy

进入proxy.Init()定义：

kubeedge/edgemesh/pkg/proxy/proxy.go

	// Init: init the proxy. create virtual device and assign ips, etc..
	func Init() {
		go func() {
			unused = make([]string, 0)
			addrByService = &addrTable{}
			c := context.GetContext(context.MsgCtxTypeChannel)
			metaClient = client.New(c)
			//create virtual network device
			for {
				err := vdev.CreateDevice()
				if err == nil {
					break
				}
				klog.Warningf("[L4 Proxy] create Device is failed : %s", err)
				//there may have some exception need to be fixed on OS
				time.Sleep(2 * time.Second)
			}
			//configure vir ip
			ipPoolSize = 0
			expandIpPool()
			//open epoll
			ep, err := poll.CreatePoll(pollCallback)
			if err != nil {
				vdev.DestroyDevice()
				klog.Errorf("[L4 Proxy] epoll is open failed : %s", err)
				return
			}
			epoll = ep
			go epoll.Loop()
			klog.Infof("[L4 Proxy] proxy is running now")
		}()
	}
Init()函数主要做3块事情：

1. 创建获得pod原数据的client

		unused = make([]string, 0)
		addrByService = &addrTable{}
		c := context.GetContext(context.MsgCtxTypeChannel)
		metaClient = client.New(c)
		
2. 创建和维护虚拟网络设备

		//create virtual network device
		for {
			err := vdev.CreateDevice()
			if err == nil {
				break
			}
			klog.Warningf("[L4 Proxy] create Device is failed : %s", err)
			//there may have some exception need to be fixed on OS
			time.Sleep(2 * time.Second)
		}
		//configure vir ip
		ipPoolSize = 0
		expandIpPool()
		
3. 通过epoll管理代理进程

		//open epoll
		ep, err := poll.CreatePoll(pollCallback)
		if err != nil {
			vdev.DestroyDevice()
			klog.Errorf("[L4 Proxy] epoll is open failed : %s", err)
			return
		}
		epoll = ep
		go epoll.Loop()
		klog.Infof("[L4 Proxy] proxy is running now")
		
进入创建虚拟网络设备的定义vdev.CreateDevice()：

kubeedge/edgemesh/pkg/proxy/virtualdevice/virtualdevice.go

		func CreateDevice() error {
		//if device is exist,delete and create a new one
		//make sure the latest configuration
		DestroyDevice()

		edge0 := &netlink.Dummy{
			LinkAttrs: netlink.LinkAttrs{
				Name: DeviceNameDefault,
			},
		}
		return nh.LinkAdd(edge0)
	}
CreateDevice()函数先判断虚拟网络设备edge0是否存在，如果存在就将其删除并重建；如果不存在，就直接创建虚拟网络设备edge0，至于虚拟网络设备edge0具体是什么，怎么创建的，目前官方还没有实现。

### 启动服务

进入server.Start()定义：

kubeedge/edgemesh/pkg/server/server.go

	func Start() {
		//Initialize the resolvers
		r := &resolver.MyResolver{"http"}
		resolver.RegisterResolver(r)
		//Initialize the handlers
	
		config.GlobalDefinition = &model.GlobalCfg{}
		config.GlobalDefinition.Panel.Infra = "fake"
		opts := control.Options{
			Infra:   config.GlobalDefinition.Panel.Infra,
			Address: config.GlobalDefinition.Panel.Settings["address"],
		}
		config.GlobalDefinition.Ssl = make(map[string]string)

		control.Init(opts)
		opt := registry.Options{}
		registry.DefaultServiceDiscoveryService = 	edgeregistry.NewServiceDiscovery(opt)
		myStrategy := mconfig.CONFIG.GetConfigurationByKey("mesh.loadbalance.strategy-name").(string)
		loadbalancer.InstallStrategy(myStrategy, func() loadbalancer.Strategy {
			switch myStrategy {
			case "RoundRobin":
				return &loadbalancer.RoundRobinStrategy{}
			case "Random":
				return &loadbalancer.RandomStrategy{}
			default:
				return &loadbalancer.RoundRobinStrategy{}
			}
		})
		//Start dns server
		go DnsStart()
		//Start server
		StartTCP()
	}
	
Start()函数做了3块事情：

1. 初始化resolvers、handlers

	初始化的resolvers、handlers在TCP服务处理连接的过程中被使用。
	
2. 启动 DNS服务
	
		go DnsStart()
	
	该DNS服务主要为节点内的应用做域名解析。
	
3. 启动TCP服务

	StartTCP()
	
	该TCP服务在节点间做流量代理。
	

### 通过一个for循环接收通信管道中关于edgemesh的信息并处理

kubeedge/edgemesh/pkg/module.go

	//Start sets context and starts the controller
	func (em *EdgeMesh) Start(c *context.Context) {
		...
		// we need watch message to update the cache of instances
		for {
			if msg, ok := em.context.Receive(constant.ModuleNameEdgeMesh); ok == nil {
				proxy.MsgProcess(msg)
				klog.Infof("get message: %v", msg)
				continue
			}
		}
	}
	
进入proxy.MsgProcess(...)函数定义：

kubeedge/edgemesh/pkg/proxy/proxy.go

	
	// MsgProcess process from metaManager and start a proxy server
	func MsgProcess(msg model.Message) {
		svcs := filterResourceType(msg)
		...
		for _, svc := range svcs {
			svcName := svc.Namespace + "." + svc.Name
			if !IsL4Proxy(&svc) {
				// when server protocol update to http
				delServer(svcName)
				continue
			}

			port := make([]int32, 0)
			targetPort := make([]int32, 0)
			for _, p := range svc.Spec.Ports {
				// this version will support TCP only
				if p.Protocol == "TCP" {
					port = append(port, p.Port)
					// this version will not support string type
					targetPort = append(targetPort, p.TargetPort.IntVal)
				}
			}
			if len(port) == 0 || len(targetPort) == 0 {
				continue
			}
			switch msg.GetOperation() {
			case "insert":
				addServer(svcName, port)
			case "delete":
				delServer(svcName)
			case "update":
				updateServer(svcName, port)
			default:
				klog.Infof("[L4 proxy] Unknown operation")
			}
			st := addrByService.getAddrTable(svcName)
			if st != nil {
				st.targetPort = targetPort
			}
		}
	}
MsgProcess(...）函数处理的是metamanager模块发送过来的service信息，首先，根据service的命名规范、支持的协议对收到的service信息进行过滤；然后，根据信息操作类型（insert、delete、update）对信息进行具体操作，这些操作都是在节点的缓存中进行的；最后，对相应service的targetPort进行设置。

到此，edgecore的edgemesh模块的剖析就结束了。


本文是“之江实验室端边云操作系统团队” kubeedge源码分析系列的第10篇，接下来会对各组件的源码进行系统分析。如果有机会我们团队也会积极解决kubeedge的issue和实现新的feature。


这是我们“之江实验室端边云操作系统团队”维护的“之江实验室kubeedge源码分析群“微信群，欢迎大家的参与！！！


![之江实验室kubeedge源码分析群二维码入口](https://user-gold-cdn.xitu.io/2019/11/25/16ea0c4cf2da9aa3?imageView2/0/w/1280/h/960/format/webp/ignore-error/1)