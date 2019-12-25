# kubeedge源码分析系列之edgecore（三）

本系列的源码分析是在 commit da92692baa660359bb314d89dfa3a80bffb1d26c 之上进行的。

cloudcore部分的源码分析是在[kubeedge源码分析系列之整体架构](https://juejin.im/post/5dc92c66f265da4d513359ab)基础上展开的，如果没有阅读过[kubeedge源码分析系列之整体架构](https://juejin.im/post/5dc92c66f265da4d513359ab)，直接阅读本文，会感觉比较突兀。

## 本文概述

本文对edgecore的edgehub模块进行剖析，edgehub作为edge部分与cloud部分进行交互的门户，有必要将edgehub相关内容彻底分析清楚，为使用过程中的故障排查和未来的功能扩展与性能优化都会有很大的帮助。对edgehub的剖析，具体包括如下几部分：

1. edgehub的struct调用链剖析
2. edgehub的具体逻辑剖析

## edgehub的struct组成剖析

从edgecore模块注册函数入手：

kubeedge/edge/cmd/edgecore/app/server.go

	// registerModules register all the modules started in edgecore
	func registerModules() {
		...
		edgehub.Register()
		...
	}
	
进入registerModules()函数中的edgehub.Register()，具体如下：

kubeedge/edge/pkg/edgehub/module.go

	// Register register edgehub
	func Register() {
		core.Register(&EdgeHub{
			controller: NewEdgeHubController(),
		})
	}
	
顺着Register()函数中EdgeHub struct的实例化语句进入EdgeHub struct定义：

kubeedge/edge/pkg/edgehub/module.go

	//EdgeHub defines edgehub object structure
	type EdgeHub struct {
		context    *context.Context
		controller *Controller
	}

EdgeHub struct中包含\*context.Context和\*Controller 两个属性：

1. \*context.Context 在[kubeedge源码分析系列之cloudcore](https://juejin.im/post/5dcbcd50e51d451bdf43a3a8)中，笔者已经分析过\*context.Context，它是一个基于go-channels的消息框架，edgecore用它作为各功能模块之间通信的消息管道。

2. \*Controller edgehub的主要功能载体；

进入Controller struct的定义：

kubeedge/edge/pkg/edgehub/controller.go

	//Controller is EdgeHub controller object
	type Controller struct {
		context    *context.Context
		chClient   clients.Adapter
		config     *config.ControllerConfig
		stopChan   chan struct{}
		syncKeeper map[string]chan model.Message
		keeperLock sync.RWMutex
	}
	
从Controller struct的定义可以确定，Controller struct是edgehub核心功能载体没错了。

到此，edgehub的struct组成剖析就结束了，接下来剖析edgehub的具体逻辑。

## edgehub的具体逻辑剖析

回到edgehub的注册函数，开始剖析edgehub相关的逻辑：

kubeedge/edge/pkg/edgehub/module.go

	// Register register edgehub
	func Register() {
		core.Register(&EdgeHub{
			controller: NewEdgeHubController(),
		})
	}

在Register()函数中对EdgeHub struct的初始化只是对EdgeHub struct中的controller进行了初始化，进入controller的初始化函数：

kubeedge/edge/pkg/edgehub/controller.go

	//NewEdgeHubController creates and returns a EdgeHubController object
	func NewEdgeHubController() *Controller {
		return &Controller{
			config:     &config.GetConfig().CtrConfig,
			stopChan:   make(chan struct{}),
			syncKeeper: make(map[string]chan model.Message),
		}
	}
NewEdgeHubController()函数中嵌套了一个获取配置信息的函数调用：

	config:     &config.GetConfig().CtrConfig
	
进入config.GetConfig()函数定义：

kubeedge/edge/pkg/edgehub/config/config.go

	var edgeHubConfig EdgeHubConfig
	...
	//GetConfig returns the EdgeHub configuration object
	func GetConfig() *EdgeHubConfig {
		return &edgeHubConfig
	}
	
GetConfig()函数只返回了&edgeHubConfig，而edgeHubConfig是一个EdgeHubConfig struct类型的全局变量，至于该变量是在哪里被赋值，怎么赋值的，暂且立个疑问flag:

* EdgeHubConfig赋值疑问

到此，EdgeHub struct的初始化就告一段落了，下面分析EdgeHub的启动逻辑：

kubeedge/edge/pkg/edgehub/module.go

	//Start sets context and starts the controller
	func (eh *EdgeHub) Start(c *context.Context) {
		eh.context = c
		eh.controller.Start(c)
	}
	

EdgeHub启动函数Start(...)只做了2件事：

1. 接收并存储传入的消息管道

		eh.context = c
		
2. 启动EdgeHub的controller

		eh.controller.Start(c)
		
由前面的分析可知EdgeHub的controller作为EdgeHub功能的主要载体，其启动函数也必将囊括EdgeHub绝大部分启动逻辑，继续进入controller的启动函数定义：

kubeedge/edge/pkg/edgehub/controller.go

	//Start will start EdgeHub
	func (ehc *Controller) Start(ctx *context.Context) {
		config.InitEdgehubConfig()
		for {
			err := ehc.initial(ctx)
			...
			err = ehc.chClient.Init()
			...
			// execute hook func after connect
			ehc.pubConnectInfo(true)
			go ehc.routeToEdge()
			go ehc.routeToCloud()
			go ehc.keepalive()

			// wait the stop singal
			// stop authinfo manager/websocket connection
			<-ehc.stopChan
			ehc.chClient.Uninit()

			// execute hook fun after disconnect
			ehc.pubConnectInfo(false)

			// sleep one period of heartbeat, then try to connect cloud hub again
			time.Sleep(ehc.config.HeartbeatPeriod * 2)

			// clean channel
		clean:
			for {
				select {
				case <-ehc.stopChan:
				default:
					break clean
				}
			}
		}
	}
	
从Controller的启动函数Start(...)的定义，可以清楚地看到，其中包含了EdgeHub的初始化、各种业务go routine的启动和最后的退出清理，下面逐个深入剖析：

1. 初始化EdgehubConfig

		config.InitEdgehubConfig()
		
进入config.InitEdgehubConfig()函数定义：

kubeedge/edge/pkg/edgehub/config/config.go

	// InitEdgehubConfig init edgehub config
	func InitEdgehubConfig() {
		err := getControllerConfig()
		...
		if edgeHubConfig.CtrConfig.Protocol == protocolWebsocket {
			err = getWebSocketConfig()
			...
		} else if edgeHubConfig.CtrConfig.Protocol == protocolQuic {
			err = getQuicConfig()
			...
		} else {
			...
		}
	}
	
InitEdgehubConfig()函数首先通过err := getControllerConfig()获得EdgeHub Controller的配置信息，然后通过获得的配置信息中的Protocol字段来判断是哪个协议，最后根据判断结果获取相应的协议绑定的配置信息或报错。

针对以上获取配置的操作，重点分析获得EdgeHub Controller的配置信息，进入getControllerConfig()：

kubeedge/edge/pkg/edgehub/config/config.go

	
	var edgeHubConfig EdgeHubConfig
	...
	func getControllerConfig() error {
		protocol, err := config.CONFIG.GetValue("edgehub.controller.protocol").ToString()
		...
		edgeHubConfig.CtrConfig.Protocol = protocol

		heartbeat, err := config.CONFIG.GetValue("edgehub.controller.heartbeat").ToInt()
		...
		edgeHubConfig.CtrConfig.HeartbeatPeriod = time.Duration(heartbeat) * time.Second

		projectID, err := config.CONFIG.GetValue("edgehub.controller.project-id").ToString()
		...
		edgeHubConfig.CtrConfig.ProjectID = projectID

		nodeID, err := config.CONFIG.GetValue("edgehub.controller.node-id").ToString()
		...
		edgeHubConfig.CtrConfig.NodeID = nodeID

		return nil
	}
	

getControllerConfig()获取edgehub.controller.*相关的配置信息并赋值给变量edgeHubConfig，到此前面立的“EdgeHubConfig赋值疑问”flag也得到了解答。

2. EdgeHub Controller初始化 

		err := ehc.initial(ctx)
		
进入ehc.initial(...)函数定义：

kubeedge/edge/pkg/edgehub/controller.go

	func (ehc *Controller) initial(ctx *context.Context) (err error) {
		config.GetConfig().WSConfig.URL, err = bhconfig.CONFIG.GetValue("edgehub.websocket.url").ToString()
		...
		cloudHubClient, err := clients.GetClient(ehc.config.Protocol, config.GetConfig())
		...
		ehc.context = ctx
		ehc.chClient = cloudHubClient
		return nil
	}
	
1. 第一行单独获取edgehub.websocket.url感觉在前面“初始化EdgehubConfig“中的websocket配置信息初始化部分重复，在此立一个源码问题flag:

* 获取websocket配置信息重复

2. 获取cloudhub client

		cloudHubClient, err := clients.GetClient(ehc.config.Protocol, config.GetConfig())
		
进入clients.GetClient(...)定义：

kubeedge/edge/pkg/edgehub/factory.go

		//GetClient returns an Adapter object with new web socket
	func GetClient(clientType string, config *config.EdgeHubConfig) (Adapter, error) {

		switch clientType {
		case ClientTypeWebSocket:
			websocketConf := wsclient.WebSocketConfig{
				...
			}
			return wsclient.NewWebSocketClient(&websocketConf), nil
		case ClientTypeQuic:
			quicConfig := quicclient.QuicConfig{
				...
			}
			return quicclient.NewQuicClient(&quicConfig), nil
		default:
			klog.Errorf("Client type: %s is not supported", clientType)
		}
		return nil, ErrorWrongClientType
	}
	
从GetClient(...)函数定义可以知道，该函数定义了ClientTypeWebSocket、ClientTypeQuic两种client类型，两者都实现了Adapter interface，下面遇到Adapter类型的client变量时，记得对应此处的ClientTypeWebSocket、ClientTypeQuic。

3. cloud client初始化

		err = ehc.chClient.Init()
		
ehc.chClient.Init()函数对应”获取cloudhub client“中ClientTypeWebSocket、ClientTypeQuic的Init()方法，想要了解Init()具体做了哪些事情，感兴趣的同学可以在本文的基础上自行剖析。

4. 向edgecore各模块广播已经连接成功的消息

		// execute hook func after connect
		ehc.pubConnectInfo(true)
		
5. 将从cloud部分收到的消息转发给指定edge部分的指定模块

		go ehc.routeToEdge()
		
6. 将从edge部分的消息转发给cloud部分

		go ehc.routeToCloud()

7. 向cloud部分发送心跳信息

		go ehc.keepalive()
		
8. 剩下的步骤都是在edgehub模块退出时的一些清理操作。

到此edgecore组件的edgehub模块的剖析就结束了，由于篇幅原因，很多思路比较清楚的就没有展开分析，感兴趣的同学可以在本文的基础上自习剖析。

本文是“之江实验室端边云操作系统团队” kubeedge源码分析系列的第6篇，接下来会对各组件的源码进行系统分析。如果有机会我们团队也会积极解决kubeedge的issue和实现新的feature。


这是我们“之江实验室端边云操作系统团队”维护的“之江实验室kubeedge源码分析群“微信群，欢迎大家的参与！！！


![之江实验室kubeedge源码分析群二维码入口](https://user-gold-cdn.xitu.io/2019/11/14/16e68041067a8e76?imageView2/0/w/1280/h/960/format/webp/ignore-error/1)