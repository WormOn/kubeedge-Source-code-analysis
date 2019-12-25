# kubeedge源码分析系列之edgecore（二）


本系列的源码分析是在 commit da92692baa660359bb314d89dfa3a80bffb1d26c 之上进行的。

cloudcore部分的源码分析是在[kubeedge源码分析系列之整体架构](https://juejin.im/post/5dc92c66f265da4d513359ab)基础上展开的，如果没有阅读过[kubeedge源码分析系列之整体架构](https://juejin.im/post/5dc92c66f265da4d513359ab)，直接阅读本文，会感觉比较突兀。


## 本文概述

本文对edgecore进行了展开，因为edgecore中的功能组件比较多，共包括devicetwin、edged、edgehub、eventbus、edgemesh、metamanager、servicebus、和test共8个功能模块，限于篇幅，需要多篇文章才能分析清楚，本文只对edged进行剖析，具体如下：

1. edged的具体逻辑剖析
2. edged调用容器运行时剖析

## edged的具体逻辑剖析

从edgecore模块注册函数入手：

kubeedge/edge/cmd/edgecore/app/server.go

	// registerModules register all the modules started in edgecore
	func registerModules() {
		...
		edged.Register()
		...
	}
	
进入registerModules()函数中的edged.Register()，具体如下：

kubeedge/edge/pkg/edged/edged.go

	// Register register edged
	func Register() {
		edged, err := newEdged()
		if err != nil {
			klog.Errorf("init new edged error, %v", err)
			return
		}
		core.Register(edged)
	}
	
在Register()函数中，主要做了如下2件事：

1. 初始化edged（edged, err := newEdged()）；
2. 注册将已经实例化的edged struct（core.Register(edged)）；

下面要深入剖析初始化edged过程中具体做了哪些事情，进入newEdged()函数的定义：

kubeedge/edge/pkg/edged/edged.go

	//newEdged creates new edged object and initialises it
	func newEdged() (*edged, error) {
		conf := getConfig()
		backoff := flowcontrol.NewBackOff(backOffPeriod, MaxContainerBackOff)

		podManager := podmanager.NewPodManager()
		policy := images.ImageGCPolicy{
			...
		}
		// build new object to match interface
		recorder := record.NewEventRecorder()

		ed := &edged{
			...
		}
		...
		ed.livenessManager = proberesults.NewManager()
		...
		statsProvider := edgeimages.NewStatsProvider()
		...



		//create and start the docker shim running as a grpc server
		if conf.remoteRuntimeEndpoint == DockerShimEndpoint || conf.remoteRuntimeEndpoint == DockerShimEndpointDeprecated {
			streamingConfig := &streaming.Config{}
			DockerClientConfig := &dockershim.ClientConfig{
				DockerEndpoint:            conf.DockerAddress,
				ImagePullProgressDeadline: time.Duration(conf.imagePullProgressDeadline) * time.Second,
				EnableSleep:               true,
				WithTraceDisabled:         true,
			}

			pluginConfigs := dockershim.NetworkPluginSettings{
				...
			}

			...

			ds, err := dockershim.NewDockerService(DockerClientConfig, 	conf.PodSandboxImage, streamingConfig,
			&pluginConfigs, cgroupName, cgroupDriver, DockershimRootDir, redirectContainerStream)

			if err != nil {
				return nil, err
			}

			klog.Infof("RemoteRuntimeEndpoint: %q, remoteImageEndpoint: %q",
			conf.remoteRuntimeEndpoint, conf.remoteRuntimeEndpoint)

			klog.Info("Starting the GRPC server for the docker CRI shim.")
			server := dockerremote.NewDockerServer(conf.remoteRuntimeEndpoint, ds)
			if err := server.Start(); err != nil {
				return nil, err
			}

		}
		ed.clusterDNS = convertStrToIP(conf.clusterDNS)
		ed.dnsConfigurer = kubedns.NewConfigurer(recorder, nodeRef, ed.nodeIP, ed.clusterDNS, conf.clusterDomain, ResolvConfDefault)

		containerRefManager := kubecontainer.NewRefManager()
		httpClient := &http.Client{}
		runtimeService, imageService, err := 	getRuntimeAndImageServices(conf.remoteRuntimeEndpoint, 	conf.remoteRuntimeEndpoint, conf.RuntimeRequestTimeout)
		if err != nil {
			return nil, err
		}
		if ed.os == nil {
			ed.os = kubecontainer.RealOS{}
		}

		ed.clcm, err = clcm.NewContainerLifecycleManager(DefaultRootDir)

		var machineInfo cadvisorapi.MachineInfo
		machineInfo.MemoryCapacity = uint64(conf.memoryCapacity)
		containerRuntime, err := kuberuntime.NewKubeGenericRuntimeManager(
			...
		)
	
		cadvisorInterface, err := cadvisor.New("")
		containerManager, err := cm.NewContainerManager(mount.New(""),
			cadvisorInterface,
			cm.NodeConfig{
				...
			},
			false,
			conf.devicePluginEnabled,
			recorder)
		ed.containerRuntime = containerRuntime
		ed.containerRuntimeName = RemoteContainerRuntime
		ed.containerManager = containerManager
		ed.runtimeService = runtimeService
		imageGCManager, err := images.NewImageGCManager(ed.containerRuntime, statsProvider, recorder, nodeRef, policy, conf.PodSandboxImage)
		...
		ed.imageGCManager = imageGCManager

		containerGCManager, err := kubecontainer.NewContainerGC(containerRuntime, containerGCPolicy, &containers.KubeSourcesReady{})
		...
		ed.containerGCManager = containerGCManager
		ed.server = server.NewServer(ed.podManager)
		ed.volumePluginMgr, err = NewInitializedVolumePluginMgr(ed, ProbeVolumePlugins(""))
		...

		return ed, nil
	}

从newEdged()函数的定义可以知道做很多事情，具体如下：

1. 获取edged相关配置（conf := getConfig()）；
2. 初始化podmanager（podManager := podmanager.NewPodManager()）；
3. 初始化edged struct（ed := &edged{}）；
4. 初始化 edged的livenessManager（ed.livenessManager = proberesults.NewManager()）；
5. 初始化edged的镜像存放地（statsProvider := edgeimages.NewStatsProvider()）；
6. 创建并启动dockershim的grpc server

		DockerClientConfig := &dockershim.ClientConfig{...}
		server := dockerremote.NewDockerServer(conf.remoteRuntimeEndpoint, ds)
		if err := server.Start(); err != nil {
			return nil, err
		}
		
7. 初始化运行时服务和镜像服务（runtimeService, imageService, err := getRuntimeAndImageServices(...));
8. 初始化通用容器运行时服务（containerRuntime, err := kuberuntime.NewKubeGenericRuntimeManager(...))；
9. 初始化镜像垃圾回收管理器（imageGCManager, err := images.NewImageGCManager(...））；
10. 初始化容器垃圾回收器（containerGCManager, err := kubecontainer.NewContainerGC(...））；
11. 初始化edged的server（ed.server = server.NewServer(...)）；
12. 初始化edged的volume plugin管理器（ed.volumePluginMgr, err = NewInitializedVolumePluginMgr(...））；

在以上动作中，重点分析6. ”创建并启动dockershim的grpc server“,dockershim是edged与容器运行时（container runtime）交互的管道，所以edged对容器操作在dockershim的方法中都会得到体现，进入dockershim服务的初始化函数：

k8s.io/kubernetes/pkg/kubelet/dockershim/docker_service.go

	// NewDockerService creates a new `DockerService` struct.
	// NOTE: Anything passed to DockerService should be eventually handled in another way when we switch to running the shim as a different process.
	func NewDockerService(config *ClientConfig, podSandboxImage string, streamingConfig *streaming.Config, pluginSettings *NetworkPluginSettings,
	cgroupsName string, kubeCgroupDriver string, dockershimRootDir string, startLocalStreamingServer bool) (DockerService, error) {
		...
		ds := &dockerService{
			client:          c,
			os:              kubecontainer.RealOS{},
			podSandboxImage: podSandboxImage,
			streamingRuntime: &streamingRuntime{
				client:      client,
				execHandler: &NativeExecHandler{},
			},
			containerManager:          cm.NewContainerManager(cgroupsName, client),
			checkpointManager:         checkpointManager,
			startLocalStreamingServer: startLocalStreamingServer,
			networkReady:              make(map[string]bool),
			containerCleanupInfos:     make(map[string]*containerCleanupInfo),
		}

		...
	}
	
从 NewDockerService(...)函数可以看出dockershim的真身是dockerService struct，进入dockerService struct的定义：

k8s.io/kubernetes/pkg/kubelet/dockershim/docker_service.go

	type dockerService struct {
		client           libdocker.Interface
		os               kubecontainer.OSInterface
		podSandboxImage  string
		streamingRuntime *streamingRuntime
		streamingServer  streaming.Server

		network *network.PluginManager
		// Map of podSandboxID :: network-is-ready
		networkReady     map[string]bool
		networkReadyLock sync.Mutex

		containerManager cm.ContainerManager
		// cgroup driver used by Docker runtime.
		cgroupDriver      string
		checkpointManager checkpointmanager.CheckpointManager
		// caches the version of the runtime.
		// To be compatible with multiple docker versions, we need to perform
		// version checking for some operations. Use this cache to avoid querying
		// the docker daemon every time we need to do such checks.
		versionCache *cache.ObjectCache
		// startLocalStreamingServer indicates whether dockershim should start a
		// streaming server on localhost.
		startLocalStreamingServer bool

		// containerCleanupInfos maps container IDs to the `containerCleanupInfo` structs
		// needed to clean up after containers have been started or removed.
		// (see `applyPlatformSpecificDockerConfig` and 	`performPlatformSpecificContainerCleanup`
		// methods for more info).
		containerCleanupInfos map[string]*containerCleanupInfo
	}
	
从dockerService的定义可以看出，大部分属性都是与container、pod、image相关的，从这些属性也可以推测dockerService应该是与container runtime交互的一个组件，为了进一步验证猜想，可以在同一个文件中（k8s.io/kubernetes/pkg/kubelet/dockershim/docker_service.go）查看dockerService实现的方法，由于方法太多，这里就不逐一展开了。


以上只是剖析了edged的初始化过程，下面剖析edged的启动过程：

kubeedge/edge/pkg/edged/edged.go

	func (e *edged) Start(c *context.Context) {
		e.context = c
		e.metaClient = client.New(c)

		// use self defined client to replace fake kube client
		e.kubeClient = fakekube.NewSimpleClientset(e.metaClient)

		e.statusManager = status.NewManager(e.kubeClient, e.podManager, utilpod.NewPodDeleteSafety(), e.metaClient)
		if err := e.initializeModules(); err != nil {
			klog.Errorf("initialize module error: %v", err)
			os.Exit(1)
		}

		e.volumeManager = volumemanager.NewVolumeManager(
			...
		)
		go e.volumeManager.Run(edgedutil.NewSourcesReady(), utilwait.NeverStop)
		go utilwait.Until(e.syncNodeStatus, e.nodeStatusUpdateFrequency, utilwait.NeverStop)

		e.probeManager = prober.NewManager(e.statusManager, e.livenessManager, containers.NewContainerRunner(), kubecontainer.NewRefManager(), record.NewEventRecorder())
		e.pleg = edgepleg.NewGenericLifecycleRemote(e.containerRuntime, e.probeManager, plegChannelCapacity, plegRelistPeriod, e.podManager, e.statusManager, e.podCache, clock.RealClock{}, e.interfaceName)
		e.statusManager.Start()
		e.pleg.Start()

		e.podAddWorkerRun(concurrentConsumers)
		e.podRemoveWorkerRun(concurrentConsumers)

		housekeepingTicker := time.NewTicker(housekeepingPeriod)
		syncWorkQueueCh := time.NewTicker(syncWorkQueuePeriod)
		e.probeManager.Start()
		go e.syncLoopIteration(e.pleg.Watch(), housekeepingTicker.C, syncWorkQueueCh.C)
		go e.server.ListenAndServe()

		e.imageGCManager.Start()
		e.StartGarbageCollection()

		e.pluginManager = pluginmanager.NewPluginManager(
			...
		)

		// Adding Registration Callback function for CSI Driver
		e.pluginManager.AddHandler(pluginwatcherapi.CSIPlugin, plugincache.PluginHandler(csiplugin.PluginHandler))
		// Start the plugin manager
		go e.pluginManager.Run(edgedutil.NewSourcesReady(), utilwait.NeverStop)

		e.syncPod()
	}

从启动函数Start(...）中可以看到，以go routine的方式启动很多后台处理服务，具体如下：

1. 初始化edged的kubeClient

		// use self defined client to replace fake kube client
		e.kubeClient = fakekube.NewSimpleClientset(e.metaClient)
		
2. 初始化pod status管理器

		e.statusManager = status.NewManager(e.kubeClient, e.podManager, utilpod.NewPodDeleteSafety(), e.metaClient)
		
3. 初始化edged节点的模块

		if err := e.initializeModules(); err != nil {
			klog.Errorf("initialize module error: %v", err)
			os.Exit(1)
		}
进入e.initializeModules()函数：

		func (e *edged) initializeModules() error {
			node, _ := e.initialNode()
			if err := e.containerManager.Start(node, e.GetActivePods, edgedutil.NewSourcesReady(), e.statusManager, e.runtimeService); err != nil {
				klog.Errorf("Failed to start device plugin manager %v", err)
				return err
			}
			return nil
		}
从函数initializeModules(...)定义可以看出，该函数实际上启动了容器管理器（container manager）；

4. 初始化并启动volume管理器

		e.volumeManager = volumemanager.NewVolumeManager(
		...
		)
		go e.volumeManager.Run(edgedutil.NewSourcesReady(), utilwait.NeverStop)
		
5. 初始化pod生命周期事件生成器(pleg）

		e.pleg = edgepleg.NewGenericLifecycleRemote(...)
		...
		e.pleg.Start()	
		
6. 启动pod增加和删除消息队列

		e.podAddWorkerRun(concurrentConsumers)
		e.podRemoveWorkerRun(concurrentConsumers)
		
7. 启动edged的探针管理器

		e.probeManager.Start()
		
8. 	启动监听pod事件的loop

		go e.syncLoopIteration(e.pleg.Watch(), housekeepingTicker.C, syncWorkQueueCh.C)
		
9. 启动edged的http server

		go e.server.ListenAndServe()
		
10. 启动镜像和容器的垃圾回收服务

		e.imageGCManager.Start()
		e.StartGarbageCollection()
		
11. 初始化和启动edged的插件服务

		e.pluginManager = pluginmanager.NewPluginManager（...)
		// Adding Registration Callback function for CSI Driver
		e.pluginManager.AddHandler(pluginwatcherapi.CSIPlugin, plugincache.PluginHandler(csiplugin.PluginHandler))
		// Start the plugin manager
		klog.Infof("starting plugin manager")
		go e.pluginManager.Run(edgedutil.NewSourcesReady(), utilwait.NeverStop)

12. 启动与metamanager进行事件同步的服务

		e.syncPod()
		
到此edged的具体逻辑剖析就结束了，由于edged的内容较多，本文就没有逐一展开，有感兴趣的同学可以在本文的基础上自行剖析。

## edged调用容器运行时剖析

edged与容器运行时（container runtime）的调用关系，可以总结为下图：

![edged调用容器运行时剖析](http://q0xwr06f1.bkt.clouddn.com/edge%E6%9E%B6%E6%9E%84%E5%9B%BE.png)

从上图可以看出edged首先启动dockershim的grpc server，然后edged通过调用dockershim的grpc server,来实现与容器运行时（container runtime）的交互，最后dockershim的grpc server将edged具体操作传递给容器运行时（container runtime）。

本文是“之江实验室端边云操作系统团队” kubeedge源码分析系列的第5篇，接下来会对各组件的源码进行系统分析。如果有机会我们团队也会积极解决kubeedge的issue和实现新的feature。


这是我们“之江实验室端边云操作系统团队”维护的“之江实验室kubeedge源码分析群“微信群，欢迎大家的参与！！！


![之江实验室kubeedge源码分析群二维码入口](https://user-gold-cdn.xitu.io/2019/11/14/16e68041067a8e76?imageView2/0/w/1280/h/960/format/webp/ignore-error/1)