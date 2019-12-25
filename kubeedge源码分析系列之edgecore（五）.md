# kubeedge源码分析系列之edgecore（五）

本系列的源码分析是在 commit da92692baa660359bb314d89dfa3a80bffb1d26c 之上进行的。

cloudcore部分的源码分析是在[kubeedge源码分析系列之整体架构](https://juejin.im/post/5dc92c66f265da4d513359ab)基础上展开的，如果没有阅读过[kubeedge源码分析系列之整体架构](https://juejin.im/post/5dc92c66f265da4d513359ab)，直接阅读本文，会感觉比较突兀。

## 本文概述

本文对edgecore的metamanager模块进行剖析，metamanager作为edgecore中的edged模块与edgehub模块进行交互的桥梁，除了将edgehub的消息转发给edged，还对一些必要的数据通过SQLite进行了缓存，在某种程度上实现了kubeedge的offline mode。本文就对metamanager所涉及的SQLite数据库相关逻辑和业务逻辑进行剖析：

1. metamanager数据库相关逻辑剖析
2. metamanager业务逻辑剖析



## metamanager数据库相关逻辑剖析

从metamanager的模块注册函数入手：

kubeedge/edge/pkg/metamanager/module.go

	//constant metamanager module name
	const (
		MetaManagerModuleName = "metaManager"
	)
	...

	// Register register metamanager
	func Register() {
		dbm.RegisterModel(MetaManagerModuleName, new(dao.Meta))
		core.Register(&metaManager{})
	}
	
在注册函数Register()中，做了2件事：

1. 在SQLite中的数据库中初始化metaManager表

		dbm.RegisterModel(MetaManagerModuleName, new(dao.Meta))
		
2. 注册已经初始化的metamanager

		core.Register(&metaManager{})	
		
下面深入剖析 “1.在SQLite中的数据库中初始化metaManager表” 相关内容，进入dbm.RegisterModel(...)：

kubeedge/edge/pkg/common/dbm/db.go

	//RegisterModel registers the defined model in the orm if model is enabled
	func RegisterModel(moduleName string, m interface{}) {
		if isModuleEnabled(moduleName) {
			orm.RegisterModel(m)
			...
		} else {
			...
		}
	}
	
 RegisterModel(...)函数是对[github.com/astaxie/beego/orm](https://github.com/astaxie/beego/tree/develop/orm)的封装，本文就不跟进去剖析了，感兴趣的同学可以在本文的基础上自行剖析。
 
 回到“1.在SQLite中的数据库中初始化metaManager表”，深入剖析metaManager表的具体定义dao.Meta，进入dao.Meta定义：
 
 kubeedge/edge/pkg/metamanager/dao/meta.go
 
 	// Meta metadata object
	type Meta struct {
		// ID    int64  `orm:"pk; auto; column(id)"`
		Key   string `orm:"column(key); size(256); pk"`
		Type  string `orm:"column(type); size(32)"`
		Value string `orm:"column(value); null; type(text)"`
	}

metaManager表的具体定义包含	Key、Type和Value三个字段，具体含义如下：

1. Key meta的名字；
2. Type meta对应的操作类型；
3. Value 具体的meta值；

与Meta Struct的定义在同一文件内，还有对metaManager表的一些操作定义，如SaveMeta、DeleteMetaByKey、UpdateMeta、InsertOrUpdate、UpdateMetaField、UpdateMetaFields、QueryMeta、QueryAllMeta，本文不对具体操作的定义进行深入剖析，感兴趣的同学可以在本文的基础上自行剖析。

## metamanager业务逻辑剖析

从metamanager的模块启动函数入手：

kubeedge/edge/pkg/metamanager/module.go

	func (m *metaManager) Start(c *context.Context) {
		m.context = c
		InitMetaManagerConfig()
		go func() {
			period := getSyncInterval()
			timer := time.NewTimer(period)
			for {
				select {
				case <-timer.C:
					timer.Reset(period)
					msg := model.NewMessage("").BuildRouter(MetaManagerModuleName, GroupResource, model.ResourceTypePodStatus, OperationMetaSync)
					m.context.Send(MetaManagerModuleName, *msg)
				}
			}
		}()
		m.mainLoop()
	}

启动函数Start(...)做了如下4件事：

1. 接收并保存模块启动时传入的\*context.Context实例
	
		m.context = c	
		
2. 初始化metamanager配置

		InitMetaManagerConfig()
		
3. 启动一个goroutine同步心跳信息

		go func() {...}
		
4. 启动一个循环处理各种事件

		m.mainLoop()	
		
接下来展开分析2、3、4。

### 初始化metamanager配置

进入InitMetaManagerConfig()定义：

kubeedge/edge/pkg/metamanager/msg_processor.go

	// InitMetaManagerConfig init meta config
	func InitMetaManagerConfig() {
		var err error
		groupName, err := config.CONFIG.GetValue("metamanager.context-send-group").ToString()
		...

		edgeSite, err := config.CONFIG.GetValue("metamanager.edgesite").ToBool()
		...

		moduleName, err := config.CONFIG.GetValue("metamanager.context-send-module").ToString()
		...
	}
	
在初始化metamanager配置时，从配置文件中获取了metamanager.context-send-group、metamanager.edgesite、metamanager.context-send-module，根据获取的值对相关变量进行设置。如果对配置文件的读取过程有疑惑可以参考[kubeedge源码分析系列之配置文件读取](https://juejin.im/post/5ddb564351882572fd108cc7)。

### 启动一个goroutine同步心跳信息


kubeedge/edge/pkg/metamanager/module.go

	go func() {
		period := getSyncInterval()
		timer := time.NewTimer(period)
		for {
			select {
			case <-timer.C:
				timer.Reset(period)
				msg := model.NewMessage("").BuildRouter(MetaManagerModuleName, GroupResource, model.ResourceTypePodStatus, OperationMetaSync)
				m.context.Send(MetaManagerModuleName, *msg)
			}
		}
	}

在同步心跳信息的goroutine中，做了如下2件事：

1. 获取通信心跳的时间间隔

		period := getSyncInterval()
		
2. 创建定时器，并定时发送心跳信息

		timer := time.NewTimer(period)
		for {...}
		
## 启动一个循环处理各种事件

进入m.mainLoop()	定义：

kubeedge/edge/pkg/metamanager/msg_processor.go

	func (m *metaManager) mainLoop() {
		go func() {
			for {
				if msg, err := m.context.Receive(m.Name()); err == nil {
				...
				m.process(msg)
			} else {
				...
			}
		}
		}()
	}
 mainLoop()函数启动了一个for循环，在循环中主要做了2件事：
 
 1. 接收信息

 		msg, err := m.context.Receive(m.Name())
 		
 2. 对接收到的信息进行处理

 		m.process(msg)
 		
 想弄明白对信息的处理过程，需要进入m.process(...)的定义：
 
 kubeedge/edge/pkg/metamanager/msg_processor.go
 
 		func (m *metaManager) process(message model.Message) {
			operation := message.GetOperation()
			switch operation {
			case model.InsertOperation:
				m.processInsert(message)
			case model.UpdateOperation:
				m.processUpdate(message)
			case model.DeleteOperation:
				m.processDelete(message)
			case model.QueryOperation:
				m.processQuery(message)
			case model.ResponseOperation:
				m.processResponse(message)
			case messagepkg.OperationNodeConnection:
				m.processNodeConnection(message)
			case OperationMetaSync:
				m.processSync(message)
			case OperationFunctionAction:
				m.processFunctionAction(message)
			case OperationFunctionActionResult:
				m.processFunctionActionResult(message)
			case constants.CSIOperationTypeCreateVolume,
				constants.CSIOperationTypeDeleteVolume,
				constants.CSIOperationTypeControllerPublishVolume,
				constants.CSIOperationTypeControllerUnpublishVolume:
				m.processVolume(message)
			}
		}
		
process(...)函数中主要做了如下2件事：

1. 获取消息的操作的类型

		operation := message.GetOperation()
		
2. 根据信息操作类型对信息进行相应处理

	switch operation {
		...
	}
	
信息的操作类型包括insert、update、delete、query、response、publish、meta-internal-sync、action、action_result等，本文不对信息的具体处理过程剖析，感兴趣的同学可以在本文的基础上自行剖析。

到此，对edgecore中metamanager模块的剖析就结束了。


本文是“之江实验室端边云操作系统团队” kubeedge源码分析系列的第9篇，接下来会对各组件的源码进行系统分析。如果有机会我们团队也会积极解决kubeedge的issue和实现新的feature。


这是我们“之江实验室端边云操作系统团队”维护的“之江实验室kubeedge源码分析群“微信群，欢迎大家的参与！！！


![之江实验室kubeedge源码分析群二维码入口](https://user-gold-cdn.xitu.io/2019/11/25/16ea0c4cf2da9aa3?imageView2/0/w/1280/h/960/format/webp/ignore-error/1)