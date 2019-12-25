#  kubeedge源码分析系列之配置文件读取

本系列的源码分析是在 commit da92692baa660359bb314d89dfa3a80bffb1d26c 之上进行的。

cloudcore部分的源码分析是在[kubeedge源码分析系列之整体架构](https://juejin.im/post/5dc92c66f265da4d513359ab)基础上展开的，如果没有阅读过[kubeedge源码分析系列之整体架构](https://juejin.im/post/5dc92c66f265da4d513359ab)，直接阅读本文，会感觉比较突兀。

## 本文概述

截至目前，已经分析了[kubeedge整体架构](https://juejin.im/post/5dc92c66f265da4d513359ab)，[kubeedge的cloudcore部分](https://juejin.im/post/5dcbcd50e51d451bdf43a3a8)，[kubeedge edgecore部分的devicetwin](https://juejin.im/post/5dcd374ee51d45080d2bdd36)，[kubeedge edgecore部分的edged](https://juejin.im/post/5dd2970ee51d4561e73c373d)，[kubeedge edgecore部分的edgehub](https://juejin.im/post/5dd3cdfd5188254eed5b2859),但却忽略了一个重要环节，那就是配置文件的读取。由于该环节的缺失，导致很多同学在阅读以上源码分析时有一些疑惑，应同学们的强烈要求，本文对kubeedge的配置文件读取流程进行剖析。

1. 配置文件读取流程剖析

kubeedge各模块配置文件读取使用的库是相同，流程也是高度相似的，所以本文只对kubeedge其中一个模块读取配置文件的流程进行深入剖析，其他模块的配置文件读取，同学们可以在本文的基础上自行剖析。



## 配置文件读取流程剖析

本文在[kubeedge的cloudcore部分](https://juejin.im/post/5dcbcd50e51d451bdf43a3a8)“cloudhub剖析”的“初始化cloudhub的配置”基础上展开，进入“初始化cloudhub的配置”：

kubeedge/cloud/edge/pkg/cloudhub/cloudhub.go

	func (a *cloudHub) Start(c *beehiveContext.Context) {
		...
		initHubConfig()
		...
	}
	
进入initHubConfig()函数：

kubeedge/cloud/pkg/cloudhub/cloudhub.go

	import(
		...
		"github.com/kubeedge/beehive/pkg/common/config"
		...
	)

	func initHubConfig() {
		cafile, err := config.CONFIG.GetValue("cloudhub.ca").ToString()
		...
		certfile, err := config.CONFIG.GetValue("cloudhub.cert").ToString()
		...
		keyfile, err := config.CONFIG.GetValue("cloudhub.key").ToString()
		...

		util.HubConfig.ProtocolUDS, _ = config.CONFIG.GetValue("cloudhub.enable_uds").ToBool()

		util.HubConfig.Address, _ = config.CONFIG.GetValue("cloudhub.address").ToString()
		util.HubConfig.Port, _ = config.CONFIG.GetValue("cloudhub.port").ToInt()
		util.HubConfig.QuicPort, _ = config.CONFIG.GetValue("cloudhub.quic_port").ToInt()
		util.HubConfig.MaxIncomingStreams, _ = config.CONFIG.GetValue("cloudhub.max_incomingstreams").ToInt()
		util.HubConfig.UDSAddress, _ = config.CONFIG.GetValue("cloudhub.uds_address").ToString()
		util.HubConfig.KeepaliveInterval, _ = config.CONFIG.GetValue("cloudhub.keepalive-interval").ToInt()
		util.HubConfig.WriteTimeout, _ = config.CONFIG.GetValue("cloudhub.write-timeout").ToInt()
		util.HubConfig.NodeLimit, _ = config.CONFIG.GetValue("cloudhub.node-limit").ToInt()

		...
	}
	

根据initHubConfig()函数定义和相关import导入，config是import进来的package，真正起作用的是config.CONFIG，进入config.CONFIG的定义：

kubeedge/beehive/pkg/common/config/config.go

	// CONFIG conf
	var CONFIG archaius.ConfigurationFactory
	
从定义中可以知道，config.CONFIG是定义的一个archaius.ConfigurationFactory类型的全局变量，剖析到此，同学们肯定会疑惑，只定义怎么一个全局变量怎么来读取配置文件呢？目测，肯定会有函数对这个全局变量进行赋值，根据以往经验，这样的全局变量会被所在文件的init()函数初始化，检查该变量所在文件的init()函数：

kubeedge/beehive/pkg/common/config/config.go

	func init() {
		InitializeConfig()
	}
	
进入InitializeConfig()函数定义：

kubeedge/beehive/pkg/common/config/config.go

	import (
		...
		archaius "github.com/go-chassis/go-archaius"
		...
	)
	// config file  only support .yml or .yaml  !
	func InitializeConfig() {
		once.Do(func() {
			err := archaius.Init()
			...
			CONFIG = archaius.GetConfigFactory()
			ms := memoryconfigsource.NewMemoryConfigurationSource()
			CONFIG.AddSource(ms)

			cmdSource := commandlinesource.NewCommandlineConfigSource()
			CONFIG.AddSource(cmdSource)

			envSource := envconfigsource.NewEnvConfigurationSource()
			CONFIG.AddSource(envSource)
			confLocation := getConfigDirectory() + "/conf"
			_, err = os.Stat(confLocation)
			if !os.IsExist(err) {
				os.Mkdir(confLocation, os.ModePerm)
			}
			err = filepath.Walk(confLocation, func(location string, f os.FileInfo, err error) error {
				if f == nil {
					return err
				}
				if f.IsDir() {
					return nil
				}
				ext := strings.ToLower(path.Ext(location))
				if ext == ".yml" || ext == ".yaml" {
					archaius.AddFile(location)
				}
				return nil
			})
			...
		})
	}
	
从函数内容来看， InitializeConfig()函数对config.CONFIG进行了赋值，也是读取配置文件内容的真身，下面剖析一下InitializeConfig()具体做了什么事情：

1. 配置模块初始化

		err := archaius.Init()
		...
		CONFIG = archaius.GetConfigFactory()
	
2. 获取内存配置源
		
		
		ms := memoryconfigsource.NewMemoryConfigurationSource()
		CONFIG.AddSource(ms)
		
3. 获取命令行配置源

		cmdSource := commandlinesource.NewCommandlineConfigSource()
		CONFIG.AddSource(cmdSource)
		
4. 获取环境变量配置源

		envSource := envconfigsource.NewEnvConfigurationSource()
		CONFIG.AddSource(envSource)
		
5. 根据配置文件路径获取配置源

			confLocation := getConfigDirectory() + "/conf"
			_, err = os.Stat(confLocation)
			if !os.IsExist(err) {
				os.Mkdir(confLocation, os.ModePerm)
			}
			err = filepath.Walk(confLocation, func(location string, f os.FileInfo, err error) error {
				if f == nil {
					return err
				}
				if f.IsDir() {
					return nil
				}
				ext := strings.ToLower(path.Ext(location))
				if ext == ".yml" || ext == ".yaml" {
					archaius.AddFile(location)
				}
				return nil
			})
			...
		})
		
kubeedge目前所用的读取配置文件的方式是“5.根据配置文件路径获取配置源”，下面深入剖析该配置文件读取方式，首先确定配置所在路径confLocation，进入getConfigDirectory()函数定义：

kubeedge/beehive/pkg/common/config/config.go

	//constants to define config paths
	const (
		ParameterConfigPath     = "config-path"
		EnvironmentalConfigPath = "GOARCHAIUS_CONFIG_PATH"
	)
	...
	// get the configuration file path
	func getConfigDirectory() string {
		if config, err := CONFIG.GetValue(ParameterConfigPath).ToString(); err == nil {
			return config
		}

		if config, err := CONFIG.GetValue(EnvironmentalConfigPath).ToString(); err == nil {
			return config
		}

		return util.GetCurrentDirectory()
	}
	
getConfigDirectory()函数中获取配置文件所在目录的方式有3种：

1. 根据ParameterConfigPath获取

		// get the configuration file path
		func getConfigDirectory() string {
			if config, err := CONFIG.GetValue(ParameterConfigPath).ToString(); err == nil {
				return config
			}
			
2. 根据EnvironmentalConfigPath获取

		if config, err := CONFIG.GetValue(EnvironmentalConfigPath).ToString(); err == nil {
			return config
		}

3. 根据可执行文件运行所在的当前目录下获取

	return util.GetCurrentDirectory()
	
kubeedge目前获取配置文件的方式是“3. 根据可执行文件运行所在的当前目录下获取”，到此就确定了kubeedge读取配置文件的路径，接着分析怎么读入配置文件的：

kubeedge/beehive/pkg/common/config/config.go

			confLocation := getConfigDirectory() + "/conf"
			_, err = os.Stat(confLocation)
			if !os.IsExist(err) {
				os.Mkdir(confLocation, os.ModePerm)
			}
			err = filepath.Walk(confLocation, func(location string, f os.FileInfo, err error) error {
				if f == nil {
					return err
				}
				if f.IsDir() {
					return nil
				}
				ext := strings.ToLower(path.Ext(location))
				if ext == ".yml" || ext == ".yaml" {
					archaius.AddFile(location)
				}
				return nil
			})
			...
		})	
在获取了配置文件所在的目录confLocation后，干了下面几件事：

1. 判断目录是否存在，如果不存在创建该目录
	
		confLocation := getConfigDirectory() + "/conf"
		_, err = os.Stat(confLocation)
		if !os.IsExist(err) {
			os.Mkdir(confLocation, os.ModePerm)
		}
2. 遍历配置文件目录下符合条件的文件，加入配置信息源

		err = filepath.Walk(confLocation, func(location string, f os.FileInfo, err error) error {
				if f == nil {
					return err
				}
				if f.IsDir() {
					return nil
				}
				ext := strings.ToLower(path.Ext(location))
				if ext == ".yml" || ext == ".yaml" {
					archaius.AddFile(location)
				}
				return nil
			})
			...
		})	
		
到此 config.CONFIG就获取了kubeedge所需配置文件内容。

通过上面的分析，kubeedge是通过kubeedge/beehive/pkg/common/config来读去所需的配置文件的，而kubeedge/beehive/pkg/common/config又封装了[go-archaius](https://github.com/go-chassis/go-archaius)。

kubeedge各模块配置文件读取使用的库是相同，流程也是高度相似的，所以本文只对kubeedge其中一个模块读取配置文件的流程进行深入剖析，其他模块的配置文件读取，同学们可以在本文的基础上自行剖析。



本文是“之江实验室端边云操作系统团队” kubeedge源码分析系列的第7篇，接下来会对各组件的源码进行系统分析。如果有机会我们团队也会积极解决kubeedge的issue和实现新的feature。


这是我们“之江实验室端边云操作系统团队”维护的“之江实验室kubeedge源码分析群“微信群，欢迎大家的参与！！！


![之江实验室kubeedge源码分析群二维码入口](https://user-gold-cdn.xitu.io/2019/11/25/16ea0c4cf2da9aa3?imageView2/0/w/1280/h/960/format/webp/ignore-error/1)