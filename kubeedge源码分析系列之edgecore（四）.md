# kubeedge源码分析系列之edgecore（四）

本系列的源码分析是在 commit da92692baa660359bb314d89dfa3a80bffb1d26c 之上进行的。

cloudcore部分的源码分析是在[kubeedge源码分析系列之整体架构](https://juejin.im/post/5dc92c66f265da4d513359ab)基础上展开的，如果没有阅读过[kubeedge源码分析系列之整体架构](https://juejin.im/post/5dc92c66f265da4d513359ab)，直接阅读本文，会感觉比较突兀。

## 本文概述

本文对edgecore的eventbus模块进行剖析，eventbus作为kubeedge的edge部分与MQTT进行交互的门户，有必要将eventbus相关内容彻底分析清楚，为使用过程中的故障排查和未来的功能扩展与性能优化都会有很大的帮助。eventbus的具体业务逻辑主要集中在启动过程中，本节就侧重分析eventbus启动流程：

1. eventbus的struct调用链剖析
2. eventbus的具体逻辑剖析

## eventbus的struct调用链剖析

从eventbus的模块注册函数入手：

kubeedge/edge/pkg/eventbus/event_bus.go

	// Register register eventbus
	func Register() {
		mode, err := config.CONFIG.GetValue("mqtt.mode").ToInt()
		if err != nil || mode > externalMqttMode || mode < internalMqttMode {
			mode = internalMqttMode
		}
		edgeEventHubModule := eventbus{mqttMode: mode}
		core.Register(&edgeEventHubModule)
	}
	
注册函数中做了2件事：

1. 从配置文件中获取mqtt.mode，并对其进行判断

		mode, err := config.CONFIG.GetValue("mqtt.mode").ToInt()
		if err != nil || mode > externalMqttMode || mode < internalMqttMode {
			mode = internalMqttMode
		}
		
mqtt.mode的具体定义如下：

kubeedge/edge/pkg/eventbus/event_bus.go

	const (
		internalMqttMode = iota // 0: launch an internal mqtt broker.
		bothMqttMode            // 1: launch an internal and external mqtt broker.
		externalMqttMode        // 2: launch an external mqtt broker.
		...
	)
	
mqtt.mode定义分internalMqttMode、bothMqttMode和externalMqttMode三种：（1）externalMqttMode 启动内部mqtt代理；（2）bothMqttMode 同时启动内部和外部mqtt代理；（3）externalMqttMode 启动外部mqtt代理。

2. 实例化eventbus并将其注册

		edgeEventHubModule := eventbus{mqttMode: mode}
		core.Register(&edgeEventHubModule)
		
进入eventbus定义：

kubeedge/edge/pkg/eventbus/event_bus.go

	// eventbus struct
	type eventbus struct {
		context  *context.Context
		mqttMode int
	}
eventbus包括context、mqttMode两个属性，context负责与edgecore其他模块的通信，具体参考[kubeedge源码分析系列之cloudcore中的“cloudcore功能模块之间通信的消息框架”部分](https://juejin.im/post/5dcbcd50e51d451bdf43a3a8)，mqttMode用来区分eventbus连接mqtt的不同方式，下面会具体分析。

## eventbus的具体逻辑剖析

从eventbus的启动函数切入分析具体逻辑：

kubeedge/edge/pkg/eventbus/event_bus.go

	func (eb *eventbus) Start(c *context.Context) {
	// no need to call TopicInit now, we have fixed topic
	eb.context = c

		nodeID := config.CONFIG.GetConfigurationByKey("edgehub.controller.node-id")
		...

		mqttBus.NodeID = nodeID.(string)
		mqttBus.ModuleContext = c

		if eb.mqttMode >= bothMqttMode {
		// launch an external mqtt server
			externalMqttURL := config.CONFIG.GetConfigurationByKey("mqtt.server")
			...
			hub := &mqttBus.Client{
				MQTTUrl: externalMqttURL.(string),
			}
			mqttBus.MQTTHub = hub
			hub.InitSubClient()
			hub.InitPubClient()
		}

		if eb.mqttMode <= bothMqttMode {
			internalMqttURL := config.CONFIG.GetConfigurationByKey("mqtt.internal-server")
			...
			qos := config.CONFIG.GetConfigurationByKey("mqtt.qos")
			...
			retain := config.CONFIG.GetConfigurationByKey("mqtt.retain")
			...

			sessionQueueSize := config.CONFIG.GetConfigurationByKey("mqtt.session-queue-size")
			...

			if qos.(int) < int(packet.QOSAtMostOnce) || qos.(int) > int(packet.QOSExactlyOnce) || sessionQueueSize.(int) <= 0 {
				klog.Errorf("mqtt.qos must be one of [0,1,2] or mqtt.session-queue-size must > 0")
				os.Exit(1)
			}
			// launch an internal mqtt server only
			mqttServer = mqttBus.NewMqttServer(sessionQueueSize.(int), internalMqttURL.(string), retain.(bool), qos.(int))
			mqttServer.InitInternalTopics()
			err := mqttServer.Run()
			...
		}

		eb.pubCloudMsgToEdge()
	}
	
eventbus的启动函数做了如下3件事：

1. 处理eventbus模块的公共配置

		eb.context = c

		nodeID := config.CONFIG.GetConfigurationByKey("edgehub.controller.node-id")
		...

		mqttBus.NodeID = nodeID.(string)
		mqttBus.ModuleContext = c

接收并存储与edgecore其他模块通信的管道，从配置文件中获取所在节点的唯一标识。

2. 根据不同mqttMode启动与mqtt交互的不同实例

		if eb.mqttMode >= bothMqttMode {
			...
				}

		if eb.mqttMode <= bothMqttMode {
			
			...
		}
		
当 eb.mqttMode >= bothMqttMode将mqtt代理启动在eventbus之外，eventbus作为独立启动的mqtt代理的客户端与其交互；当eb.mqttMode <= bothMqttMode时，在eventbus内启动一个mqtt代理，负责与终端设备交互。对于两种情况的具体逻辑感兴趣的同学可以在本文的基础上自行分析。

3. 将云部分的指令和事件下发到与eventbus相连的设备

	
		eb.pubCloudMsgToEdge()
		
这部分的具体逻辑跟mqtt本身密切相关，感兴趣的同学可以在本文的基础上自行分析。


本文是“之江实验室端边云操作系统团队” kubeedge源码分析系列的第8篇，接下来会对各组件的源码进行系统分析。如果有机会我们团队也会积极解决kubeedge的issue和实现新的feature。


这是我们“之江实验室端边云操作系统团队”维护的“之江实验室kubeedge源码分析群“微信群，欢迎大家的参与！！！


![之江实验室kubeedge源码分析群二维码入口](https://user-gold-cdn.xitu.io/2019/11/25/16ea0c4cf2da9aa3?imageView2/0/w/1280/h/960/format/webp/ignore-error/1)