# kubeedge源码分析系列之edgecore（一）


本系列的源码分析是在 commit da92692baa660359bb314d89dfa3a80bffb1d26c 之上进行的。

cloudcore部分的源码分析是在[kubeedge源码分析系列之整体架构](https://juejin.im/post/5dc92c66f265da4d513359ab)基础上展开的，如果没有阅读过[kubeedge源码分析系列之整体架构](https://juejin.im/post/5dc92c66f265da4d513359ab)，直接阅读本文，会感觉比较突兀。

## 本文概述

本文对edgecore进行了展开，因为edgecore中的功能组件比较多，共包括devicetwin、edged、edgehub、eventbus、edgemesh、metamanager、servicebus、和test共8个功能模块，限于篇幅，需要多篇文章才能分析清楚，本文只对devicetwin进行剖析，具体如下：

1. devicetwin的struct调用链剖析
2. devicetwin的具体逻辑剖析
3. devicetwin的缓存机制剖析

## devicetwin的struct调用链剖析

从edgecore模块注册函数入手：

kubeedge/edge/cmd/edgecore/app/server.go

	// registerModules register all the modules started in edgecore
	func registerModules() {
		devicetwin.Register()
		...
	}
	
进入registerModules()函数中的devicetwin.Register()，具体如下：

kubeedge/edge/pkg/devicetwin/devicetwin.go
	
	// Register register devicetwin
	func Register() {
		dtclient.InitDBTable()
		dt := DeviceTwin{}
		core.Register(&dt)
	}
	
在Register()函数中，主要做了如下3件事：

1. 初始化devicetwin需要的数据库表（dtclient.InitDBTable()）；
2. 实例化DeviceTwin struct（dt := DeviceTwin{}）；
3. 注册将已经实例化的DeviceTwin struct（core.Register(&dt)）；

下面要深入剖析初始化devicetwin需要的数据库表具体做了哪些事情，DeviceTwin struct具体有哪些属性组成的。

### devicetwin数据库相关struct剖析

这部分剖析初始化devicetwin需要的数据库及数据库表相关的struct。

kubeedge/edge/pkg/devicetwin/dtclient/sql.go

	//InitDBTable create table
	func InitDBTable() {
		klog.Info("Begin to register twin model")
		dbm.RegisterModel("twin", new(Device))
		dbm.RegisterModel("twin", new(DeviceAttr))
		dbm.RegisterModel("twin", new(DeviceTwin))
	}	
	
在InitDBTable()函数中，通过封装的[beego的orm](https://github.com/astaxie/beego/tree/develop/orm)创建了数据库"twin"，并初始化了"device"、"device_attr"和"device_twin"3张表，与上述三张表对应的结构体如下：

kubeedge/edge/pkg/devicetwin/dtclient/device_db.go

	//Device the struct of device
	type Device struct {
		ID          string `orm:"column(id); size(64); pk"`
		Name        string `orm:"column(name); null; type(text)"`
		Description string `orm:"column(description); null; type(text)"`
		State       string `orm:"column(state); null; type(text)"`
		LastOnline  string `orm:"column(last_online); null; type(text)"`
	}

kubeedge/edge/pkg/devicetwin/dtclient/deviceattr_db.go

	//DeviceAttr the struct of device attributes
	type DeviceAttr struct {
		ID          int64  `orm:"column(id);size(64);auto;pk"`
		DeviceID    string `orm:"column(deviceid); null; type(text)"`
		Name        string `orm:"column(name);null;type(text)"`
		Description string `orm:"column(description);null;type(text)"`
		Value       string `orm:"column(value);null;type(text)"`
		Optional    bool   `orm:"column(optional);null;type(integer)"`
		AttrType    string `orm:"column(attr_type);null;type(text)"`
		Metadata    string `orm:"column(metadata);null;type(text)"`
	}
	
kubeedge/edge/pkg/devicetwin/dtclient/devicetwin_db.go

	//DeviceTwin the struct of device twin
	type DeviceTwin struct {
		ID              int64  `orm:"column(id);size(64);auto;pk"`
		DeviceID        string `orm:"column(deviceid); null; type(text)"`
		Name            string `orm:"column(name);null;type(text)"`
		Description     string `orm:"column(description);null;type(text)"`
		Expected        string `orm:"column(expected);null;type(text)"`
		Actual          string `orm:"column(actual);null;type(text)"`
		ExpectedMeta    string `orm:"column(expected_meta);null;type(text)"`
		ActualMeta      string `orm:"column(actual_meta);null;type(text)"`
		ExpectedVersion string `orm:"column(expected_version);null;type(text)"`
		ActualVersion   string `orm:"column(actual_version);null;type(text)"`
		Optional        bool   `orm:"column(optional);null;type(integer)"`
		AttrType        string `orm:"column(attr_type);null;type(text)"`
		Metadata        string `orm:"column(metadata);null;type(text)"`
	}

在以上3个文件中除了与"device"、"device_attr"和"device_twin"3张表对应的struct定义外，还有针对这三张表的增、删、改、查方法的定义，感兴趣的同学可以在本文剖析的基础上去自己尝试剖析。

### DeviceTwin struct组成剖析

该部分对DeviceTwin struct的组成进行剖析。接着“devicetwin的struct调用链剖析”的 2. 实例化DeviceTwin struct（dt := DeviceTwin{}）往下剖析，进入DeviceTwin struct的定义：

kubeedge/edge/pkg/devicetwin/devicetwin.go

	//DeviceTwin the module
	type DeviceTwin struct {
		context      *context.Context
		dtcontroller *DTController
	}
	
DeviceTwin struct的定义由 \*context.Context 和 \*DTController两部分组成，其中\*context.Context可以参考[kubeedge源码分析系列之cloudcore](https://juejin.im/post/5dcbcd50e51d451bdf43a3a8)的“cloudcore功能模块之间通信的消息框架”,这里就不再赘述。下面重点剖析DTController，进入DTController的定义：

kubeedge/edge/pkg/devicetwin/dtcontroller.go

	//DTController controller for devicetwin
	type DTController struct {
		HeartBeatToModule map[string]chan interface{}
		DTContexts        *dtcontext.DTContext
		DTModules         map[string]dtmodule.DTModule
		Stop              chan bool
	}
	
在DTController struct定义中发现\*dtcontext.DTContext和dtmodule.DTModule还不清楚，继续进入它们的定义：

kubeedge/edge/pkg/devicetwin/dtcontext/dtcontext.go

	//DTContext context for devicetwin
	type DTContext struct {
		GroupID        string
		NodeID         string
		CommChan       map[string]chan interface{}
		ConfirmChan    chan interface{}
		ConfirmMap     *sync.Map
		ModulesHealth  *sync.Map
		ModulesContext *context.Context
		DeviceList     *sync.Map
		DeviceMutex    *sync.Map
		Mutex          *sync.RWMutex
		// DBConn *dtclient.Conn
		State string
	}
	
从DTContext struct的定义可以看出DTContext struct主要用来实现devicetwin的通信和缓存。


kubeedge/edge/pkg/devicetwin/dtmodule/dtmodule.go

	//DTModule module for devicetwin
	type DTModule struct {
		Name   string
		Worker dtmanager.DTWorker
	}

在 DTModule struct定义中dtmanager.DTWorker是interface type，其定义如下：

kubeedge/edge/pkg/devicetwin/dtmanager/dtworker.go

	//DTWorker worker for devicetwin
	type DTWorker interface {
		Start()
	}

从dtmanager.DTWorker这个interface type可以推测DTModule有多种类型，而且都实现了DTWorker interface，在kubeedge/edge/pkg/devicetwin/dtmodule/dtmodule.go中的InitWorker(...)就是来实例化DTModule的多种类型的，具体定义如下：

kubeedge/edge/pkg/devicetwin/dtmodule/dtmodule.go

	// InitWorker init worker
	func (dm *DTModule) InitWorker(recv chan interface{}, confirm chan interface{}, heartBeat chan interface{}, dtContext *dtcontext.DTContext) {
		switch dm.Name {
		case dtcommon.MemModule:
			dm.Worker = dtmanager.MemWorker{
				Group: dtcommon.MemModule,
				Worker: dtmanager.Worker{
					ReceiverChan:  recv,
					ConfirmChan:   confirm,
					HeartBeatChan: heartBeat,
					DTContexts:    dtContext,
			},
		}
	
		...
	}
	
从InitWorker(...)函数的定义中可以梳理出DTModule有MemWorker、TwinWorker、DeviceWorker和CommWorker四种类型，至于每种类型有哪些属性，用来做什么的，感兴趣的同学可以在本文梳理的基础上自己尝试剖析。

到此edgecore中devicetwin的struct调用链剖析就全部结束了，有分析的不够深入或没有覆盖到的，感兴趣的同学可以在本文的基础上自行剖析，也可以在评论区里留言。

## devicetwin的具体逻辑剖析

从devicetwin的启动函数入手：

kubeedge/edge/pkg/devicetwin/devicetwin.go

	//Start run the module
	func (dt *DeviceTwin) Start(c *context.Context) {
		controller, err := InitDTController(c)
		if err != nil {
			klog.Errorf("Start device twin failed, due to %v", err)
		}
		dt.dtcontroller = controller
		dt.context = c
		err = controller.Start()
		if err != nil {
			klog.Errorf("Start device twin failed, due to %v", err)
		}
	}
	
在启动函数中主要做了如下几件事情：

1. 初始化DTController（controller, err := InitDTController(c)）；
2. 启动已经初始化的DTController（err = controller.Start()）；

初始化DTController时把传入的beehive context消息框架实例，并在其中初始化一些devicetwin所需的channel，用来与传入的beehive context消息框架实例进行交互，本文就不展开剖析了，感兴趣的同学可以在本文梳理的基础上自己尝试剖析。

下面深入剖析已经初始化的DTController在启动过程中和启动以后所做的事，进入DTController启动函数Start()的定义：

kubeedge/edge/pkg/devicetwin/dtcontroller.go

	//Start devicetwin controller
	func (dtc *DTController) Start() error {
		err := SyncSqlite(dtc.DTContexts)
		...
		moduleNames := []string{dtcommon.MemModule, dtcommon.TwinModule, 	dtcommon.DeviceModule, dtcommon.CommModule}
		for _, v := range moduleNames {
			dtc.RegisterDTModule(v)
			go dtc.DTModules[v].Start()
		}
		...
		}
	}
	
在上面的定义中主要做了2件事：

1. 将数据库中的内容加载到内存中来（err := SyncSqlite(dtc.DTContexts)）；
2. 启动devicetwin中所有的module：

		moduleNames := []string{dtcommon.MemModule, dtcommon.TwinModule, 	dtcommon.DeviceModule, dtcommon.CommModule}
		for _, v := range moduleNames {
			dtc.RegisterDTModule(v)
			go dtc.DTModules[v].Start()
		}
		
	
到此edgecore中devicetwin的具体逻辑剖析就全部结束了，有分析的不够深入或没有覆盖到的，感兴趣的同学可以在本文的基础上自行剖析，也可以在评论区里留言。



## devicetwin的缓存机制剖析

devicetwin中缓存是利用golang本身的sync.Map来实现的，这里就不展开剖析了。如果真是这样的话，推测kubeedge在edge端的offline mode也是基于golang本身的sync.Map来实现的，这样就会带来一下问题：

1. 用golang本身的sync.Map需要处处用锁，这个在并发量大的情况下会出现堵塞；
2. 用golang本身的sync.Map，其在内存里最大限度能缓存多少数据，缓存周期怎么控制，缓存与持久存储怎么平衡；

到此“kubeedge源码分析系列之edgecore（一）”就全部结束了，有分析的不够深入或没有覆盖到的，感兴趣的同学可以在本文的基础上自行剖析，也可以在评论区里留言。

本文是“之江实验室端边云操作系统团队” kubeedge源码分析系列的第一篇，接下来会对各组件的源码进行系统分析。如果有机会我们团队也会积极解决kubeedge的issue和实现新的feature。


这是我们“之江实验室端边云操作系统团队”维护的“之江实验室kubeedge源码分析群“微信群，欢迎大家的参与！！！


![之江实验室kubeedge源码分析群二维码入口](https://user-gold-cdn.xitu.io/2019/11/14/16e68041067a8e76?imageView2/0/w/1280/h/960/format/webp/ignore-error/1)






