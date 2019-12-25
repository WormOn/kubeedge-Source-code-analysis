# kubeedge源码分析系列之cloudcore

本系列的源码分析是在 commit da92692baa660359bb314d89dfa3a80bffb1d26c 之上进行的。

cloudcore部分的源码分析是在[kubeedge源码分析系列之整体架构](https://juejin.im/post/5dc92c66f265da4d513359ab)基础上展开的，如果没有阅读过[kubeedge源码分析系列之整体架构](https://juejin.im/post/5dc92c66f265da4d513359ab)，直接阅读本文，会感觉比较突兀。

## 本文概述

本文对cloudcore进行了展开，对cloudcore组件中功能模块共用的消息框架和各功能模块的具体功能进行深入剖析，具体如下：

1. cloudcore功能模块之间通信的消息框架
2. cloudhub剖析
3. edgecontroller剖析
4. devicecontroller剖析


## cloudcore功能模块之间通信的消息框架

cloudcore组件中各个功能模块之间是通过Beehive来组织和管理的，Beehive是一个基于go-channels的消息框架，但本文不对Beehive做全面的分析，只会分析cloudcore组件中用到的Beehive的相关功能。

Beehive的消息框架是在cloudcore的功能模块启动的时候一如的，具体如下：

kubeedge/beehive/pkg/core/core.go

	import (
		...
		"github.com/kubeedge/beehive/pkg/core/context"
	)

	// StartModules starts modules that are registered
	func StartModules() {
		coreContext := context.GetContext(context.MsgCtxTypeChannel)

		modules := GetModules()
		for name, module := range modules {
			//Init the module
			coreContext.AddModule(name)
			//Assemble typeChannels for sendToGroup
			coreContext.AddModuleGroup(name, module.Group())
			go module.Start(coreContext)
			klog.Infof("Starting module %v", name)
		}
	}
	
从上面的函数定义中可以看到，在cloudcore的功能模块（model）启动之前，首先实例化了一个beehive的context，然后再获得各个功能模块，最后用一个for循环逐个启动功能模块，并将已实例化的beehive的context作为参数，传入每个功能模块的启动函数（Start(...)）中。这样cloudcore中独立的功能模块就被beehive的context给组织成了一个统一的整体，至于beehive的context是怎么做到的，还需要进入beehive的context的实例化函数context.GetContext(...)一探究竟，具体如下：

kubeedge/beehive/pkg/core/context/contex_factory.go

	// GetContext gets global context instance
	func GetContext(contextType string) *Context {
		once.Do(func() {
			context = &Context{}
			switch contextType {
			case MsgCtxTypeChannel:
				channelContext := NewChannelContext()
				context.messageContext = channelContext
				context.moduleContext = channelContext
			default:
				klog.Warningf("Do not support context type:%s", contextType)
			}
		})
		return context
	}

上面的函数定义可以中的第3行context = &Context{}实例化了一个空context，进入该实例化context的定义，探究一下它的具体内容：

kubeedge/beehive/pkg/core/context/context.go

	// Context is global context object
	type Context struct {
		moduleContext  ModuleContext
		messageContext MessageContext
	}

	//ModuleContext is interface for context module management
	type ModuleContext interface {
		AddModule(module string)
		AddModuleGroup(module, group string)
		Cleanup(module string)
	}
	
	//MessageContext is interface for message syncing
	type MessageContext interface {
		// async mode
		Send(module string, message model.Message)
		Receive(module string) (model.Message, error)
		// sync mode
		SendSync(module string, message model.Message, timeout time.Duration) (		model.Message, error)
		SendResp(message model.Message)
		// group broadcast
		SendToGroup(moduleType string, message model.Message)
		SendToGroupSync(moduleType string, message model.Message, timeout 		time.Duration) error
	}

从上面的Context Struct定义可以看出该Context的两个属性moduleContext，messageContext都是interface类型的，所以可以断定该Context肯定不是真身，在函数GetContext(...)（kubeedge/beehive/pkg/core/context/context.go）继续往下看，在第6行channelContext := NewChannelContext()，又一个context实例化函数NewChannelContext()，进入该函数的定义去看一下，它是不是真身：

kubeedge/beehive/pkg/core/context/context.go

	// ChannelContext is object for Context channel
	type ChannelContext struct {
		//ConfigFactory goarchaius.ConfigurationFactory
		channels     map[string]chan model.Message
		chsLock      sync.RWMutex
		typeChannels map[string]map[string]chan model.Message
		typeChsLock  sync.RWMutex
		anonChannels map[string]chan model.Message
		anonChsLock  sync.RWMutex
	}

在该struct的定义文件中，会看到ChannelContext struct实现了Context struct（kubeedge/beehive/pkg/core/context/context.go）属性包含的所有interface（ModuleContext，MessageContext），毫无疑问ChannelContext struct就是cloudcore中各功能模块相互通信的消息队列框架的真身了。

至于ChannelContext struct（kubeedge/beehive/pkg/core/context/context.go）具体是通过什么手段来实现cloudcore中各功能模块相互通信的，感兴趣的同学可以根据本文的梳理自己去研究一下，如果想让我们继续深入分析，可以在评论区留言。

## cloudhub剖析

从“cloudcore功能模块之间通信的消息框架”已经分析到各功能模块的启动了，cloudhub功能模块启动函数的具体内容如下：

kubeedge/cloud/edge/pkg/cloudhub/cloudhub.go

	func (a *cloudHub) Start(c *beehiveContext.Context) {
		var ctx context.Context
		a.context = c
		ctx, a.cancel = context.WithCancel(context.Background())

		initHubConfig()

		messageq := channelq.NewChannelMessageQueue(c)

		// start dispatch message from the cloud to edge node
		go messageq.DispatchMessage(ctx)

		// start the cloudhub server
		if util.HubConfig.ProtocolWebsocket {
			go servers.StartCloudHub(servers.ProtocolWebsocket, messageq, c)
		}

		if util.HubConfig.ProtocolQuic {
			go servers.StartCloudHub(servers.ProtocolQuic, messageq, c)
		}

		if util.HubConfig.ProtocolUDS {
			// The uds server is only used to communicate with csi driver from kubeedge on cloud.
			// It is not used to communicate between cloud and edge.
			go udsserver.StartServer(util.HubConfig, c)
		}

	}

从以上cloudhub的启动函数Start(...)定义中，可以清晰地看出cloudhub在启动时主要做了如下几件事：

1. 接受beehiveContext.Context的通信框架实例，并保存（a.context = c）；
2. 初始化cloudhub的配置（initHubConfig()）；
3. 将接受到的beehiveContext.Context的通信框架实例进行修饰（messageq := channelq.NewChannelMessageQueue(c)），在原用通信框架实例的基础上加入了缓存功能；
4. 启动一个消息分发go routine（go messageq.DispatchMessage(ctx)），监听云端的事件下发到edge端去；
5. 如果设置了websocket启动，就启动websocket服务器的go routine（go servers.StartCloudHub(servers.ProtocolWebsocket, messageq, c)
）；
6. 如果设置了quic启动，就启动quic服务器的go routine（go servers.StartCloudHub(servers.ProtocolQuic, messageq, c)）；
7. 如果设置了unix domain socket启动，就启动unix domain socket服务器的go routine（go udsserver.StartServer(util.HubConfig, c)）；

需要对以上另外说明的是: 

1. websocket server和quic server的功能是相同，也就是说两者可以选其一，如果条件允许的话，建议选quic server，速度更快一些；
2. unix domain socket是用来与kubeedge的csi（container storage interface） 通信的；

以上就是cloudcore中cloudhub功能模块的剖析，如果有同学对于cloudhub具体都做了那些事，是怎么做的感兴趣，可以在本文的基础上自行剖析，如果想让笔者分析，请评论区留言。

## edgecontroller剖析

从“cloudcore功能模块之间通信的消息框架”已经分析到各功能模块的启动了，edgecontroller功能模块启动函数的具体内容如下：

kubeedge/cloud/pkg/edgecontroller/controller.go

	// Start controller
	func (ctl *Controller) Start(c *beehiveContext.Context) {
		var ctx context.Context

		config.Context = c
		ctx, ctl.cancel = context.WithCancel(context.Background())

		initConfig()

		upstream, err := controller.NewUpstreamController()
		if err != nil {
			klog.Errorf("new upstream controller failed with error: %s", err)
			os.Exit(1)
		}
		upstream.Start(ctx)

		downstream, err := controller.NewDownstreamController()
		if err != nil {
			klog.Warningf("new downstream controller failed with error: %s", err)
			os.Exit(1)
		}
		downstream.Start(ctx)

	}
	
从以上edgecontroller的启动函数Start(...)定义中，可以清晰地看出cloudhub在启动时主要做了如下几件事：

1. 接受beehiveContext.Context的通信框架实例，并保存（config.Context = c）；
2. 初始化edgecontroller的配置（initHubConfig()）；
3. 实例化并启动UpstreamController
	
		upstream, err := controller.NewUpstreamController()
		if err != nil {
			klog.Errorf("new upstream controller failed with error: %s", err)
			os.Exit(1)
		}
		upstream.Start(ctx)
4. 实例化并启动DownstreamController

		downstream, err := controller.NewDownstreamController()
		if err != nil {
			klog.Warningf("new downstream controller failed with error: %s", err)
			os.Exit(1)
		}
		downstream.Start(ctx)	

下面深入分析UpstreamController和DownstreamController都具体做了哪些事：

### UpstreamController

顺着UpstreamController的实例化函数找到UpstreamController struct定义：

kubeedge/cloud/pkg/edgecontroller/upstream.go

	// UpstreamController subscribe messages from edge and sync to k8s api server
	type UpstreamController struct {
		kubeClient   *kubernetes.Clientset
		messageLayer messagelayer.MessageLayer

		// message channel
		nodeStatusChan            chan model.Message
		podStatusChan             chan model.Message
		secretChan                chan model.Message
		configMapChan             chan model.Message
		serviceChan               chan model.Message
		endpointsChan             chan model.Message
		persistentVolumeChan      chan model.Message
		persistentVolumeClaimChan chan model.Message
		volumeAttachmentChan      chan model.Message
		queryNodeChan             chan model.Message
		updateNodeChan            chan model.Message
	}	
	
看到上面UpstreamController的定义，估计正在阅读本文的同学已经在猜，UpstreamController是不是负责处理edge节点上报的nodeStatus、podStatus、secret、configMap、service、endpoints、persistentVolume、persistentVolumeClaim、volumeAttachment等资源的信息的，恭喜你猜对了，UpstreamController就是干这个事的。

以上就是UpstreamController功能模块的剖析，如果有同学对于UpstreamController具体怎么处理edge节点上报的nodeStatus、podStatus、secret、configMap、service、endpoints、persistentVolume、persistentVolumeClaim、volumeAttachment等资源的信息的的感兴趣，可以在本文的基础上自行剖析，如果想让笔者分析，请评论区留言。

### DownstreamController

顺着DownstreamController的实例化函数找到DownstreamController struct定义：

kubeedge/cloud/pkg/edgecontroller/downstream.go

	// DownstreamController watch kubernetes api server and send change to edge
	type DownstreamController struct {
		kubeClient   *kubernetes.Clientset
		messageLayer messagelayer.MessageLayer

		podManager *manager.PodManager

		configmapManager *manager.ConfigMapManager

		secretManager *manager.SecretManager

		nodeManager *manager.NodesManager

		serviceManager *manager.ServiceManager

		endpointsManager *manager.EndpointsManager

		lc *manager.LocationCache
	}
	
看到上面DownstreamController的定义，和UpstreamController的功能，同学们应该也可以对DownstreamController的具体功能猜到几分了，对DownstreamController就是监听cloud端pod、configmap、secret、node、service和endpoints等资源的事件，并下发到edge节点的。


以上就是DownstreamController功能模块的剖析，如果有同学对于DownstreamController具体怎么处理edge节点上报的pod、configmap、secret、node、service和endpoints等资源的信息的的感兴趣，可以在本文的基础上自行剖析，如果想让笔者分析，请评论区留言。


## devicecontroller剖析

从“cloudcore功能模块之间通信的消息框架”已经分析到各功能模块的启动了，devicecontroller功能模块启动函数的具体内容如下：

kubeedge/cloud/pkg/devicecontroller/module.go

	// Start controller
	func (dctl *DeviceController) Start(c *beehiveContext.Context) {
		var ctx context.Context
		config.Context = c

		ctx, dctl.cancel = context.WithCancel(context.Background())

		initConfig()

		downstream, err := controller.NewDownstreamController()
		if err != nil {
			klog.Errorf("New downstream controller failed with error: %s", err)
			os.Exit(1)
		}
		upstream, err := controller.NewUpstreamController(downstream)
		if err != nil {
			klog.Errorf("new upstream controller failed with error: %s", err)
			os.Exit(1)
		}

		downstream.Start(ctx)
		// wait for downstream controller to start and load deviceModels and devices
		// TODO think about sync
		time.Sleep(1 * time.Second)
		upstream.Start(ctx)
	}
	
看上面devicecontroller的启动函数是否似曾相识相识，那是因为它和edgecontroller的启动函数的逻辑基本相同，所以对于devicecontroller的剖析，同学可以参考“edgecontrller剖析“。


到此“kubeedge源码分析系列之cloudcore”就全部结束了，大家在根据本文去阅读kubeedge的源码时，一定要时刻提醒自己，cloudcore中的个model之间时可以通过“beehive的context消息通信框架”进行相互通信的，否则，会产生一些困惑。


本文是“之江实验室端边云操作系统团队” kubeedge源码分析系列的第一篇，接下来会对各组件的源码进行系统分析。如果有机会我们团队也会积极解决kubeedge的issue和实现新的feature。


这是我们“之江实验室端边云操作系统团队”维护的“之江实验室kubeedge源码分析群“微信群，欢迎大家的参与！！！

[之江实验室kubeedge源码分析群二维码入口](https://pan.baidu.com/s/1x1EAfIeKcSyAsBAQTn3ZIA)
