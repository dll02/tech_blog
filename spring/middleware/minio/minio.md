
# Minio服务中间件

Minio是目前githug上star最多的Object Storage框架，这里Object Storage对象存储，minio可以用来搭建分布式存储服务，企业级开源对象存储。
大部分云厂商都有对象存储服务。
## Minio优点
* 对象存储的主动、多站点复制是任务关键型生产环境的关键要求。MinIO是目前唯一提供它的供应商。MinIO 提供存储桶级粒度，并支持同步和近同步复制，具体取决于架构选择和数据变化率。
* 在对象存储领域，需要强大的加密才能在谈判桌上占有一席之地。MinIO 通过最高级别的加密以及广泛的优化提供更多功能，几乎消除了通常与存储加密操作相关的开销。
* 自动化数据管理界面



## Minio资料
官网地址：https://minio.org.cn/
github地址： https://github.com/minio/minio

## Minio 环境搭建

### docker 启动
```bash
docker pull minio/minio
docker images
mkdir -p /home/minio/config
mkdir -p /home/minio/data
# run docker
docker run -p 9000:9000 -p 9090:9090 \
     --net=host \
     --name minio \
     -d --restart=always \
     -e "MINIO_ACCESS_KEY=minioadmin" \
     -e "MINIO_SECRET_KEY=minioadmin" \
     -v /home/minio/data:/data \
     -v /home/minio/config:/root/.minio \
     minio/minio server \
     /data --console-address ":9090" -address ":9000"

docker run -p 9000:9000 -p 9090:9090 \
     --name minio \
     -d --restart=always \
     -e "MINIO_ACCESS_KEY=minioadmin" \
     -e "MINIO_SECRET_KEY=minioadmin" \
     -v /Users/linglingdai/opt/minio/data:/data \
     -v /Users/linglingdai/opt/minio/config:/root/.minio \
     minio/minio server \
     /data --console-address ":9090" -address ":9000"     
```

## Minio Java 中间件封装代码解析


## Minio 源码 
### Main 函数

采用自行封装的 cmd 功能,注入serverCmd和gatewayCmd,给每个命令挂载对应 Action,实际的调用函数

```go
minio/cmd/main.go
 
func newApp(name string) *cli.App {
 // Collection of minio commands currently supported are.
 commands := []cli.Command{}
 //省略无关代码
 // registerCommand registers a cli command.
 registerCommand := func(command cli.Command) {
 commands = append(commands, command)
        commandsTree.Insert(command.Name)
    }
 //省略无关代码
 // Register all commands.
  registerCommand(serverCmd)
  // ->server_main.go::serverMain
 registerCommand(gatewayCmd)
  // ->server_main.go::gatewayMain
 // Set up app.
 cli.HelpFlag = cli.BoolFlag{
        Name:  "help, h",
        Usage: "show help",
    }
 app := cli.NewApp()
 app.Name = name
 //省略无关代码
 app.Commands = commands
 app.CustomAppHelpTemplate = minioHelpTemplate
 //省略无关代码
 return app
}
// Main main for minio server.
func Main(args []string) {
 // Set the minio app name.
 appName := filepath.Base(args[0])
 
 // Run the app - exit on error.
 if err := newApp(appName).Run(args); err != nil {
        os.Exit(1)
    }
}
```



### 模式

![img](https://pic4.zhimg.com/v2-9a57c6edd41a8dd048e66bbecf44839b_r.jpg)

**裸机（server）模式**：minio存储系统的后端可以是磁盘，根据挂载的磁盘数量来决定是单机模式还是纠偏码模式。

**网关（gateway）模式**：minio存储系统后端也可以作为云网关，对接第三方的NAS系统、分布式文件系统或公有云存储资源，并为业务系统转换提供标准的对象访问接口。根据后面的命令参数来决定是使用什么代理模式进行，目前MinIO支持Google 云存储、HDFS、阿里巴巴OSS、亚马逊S3, 微软Azure Blob 存储等第三方存储资源。

从以上架构可以看出，从终端发起的S3 API都是通过网关这一层的 S3 API Router提供的，通过S3 API Router统一了后端的API，也就是提供了统一的S3 兼容API。

S3 API Router的具体实现又是通过ObjectLayer这一层实现的，ObjectLayer是个接口，它定义了MinIO对象存储服务针对对象操作的所有API。ObjectLayer接口不止每个具体的网关会实现（比如GCS），MinIO本身作为存储服务器也会实现，这样对于对象的操作通过ObjectLayer接口就统一了（面向接口编程），具体的实现可以定义来实现不同的功能，比如MinIO 单点存储、分布式存储（纠察模式）、各个具体的网关存储，都是接口ObjectLayer的具体实现。

当每个具体的网关（ 比如GCS）实现了ObjectLayer接口后，它对于具体后端存储的操作就是通过各个第三方存储SDK实现了。以GCS网关为例，终端通过S3 APi获取存储桶列表，那么最终的实现会通过GCS SDK访问GCS服务获取存储桶列表，然后包装成S3标准的结构返回给终端。

### server初始化重点步骤:

MinIO server启动有两种模式，一个是**单点模式**，一种是**纠察码模式**，其中单点模式就是只传了一个endpoint给minio，使用的是文件系统的操作方式，更详细的可以研究FSObjects的源代码实现。

启动命令

> minio server PATH

```shell
1. 加入DNS的Cache的停止的hook
2. 注册系统关闭的信号量
3. 设置分配多少字节进行内存采样的阀植，关闭 mutex prof，关闭统计阻塞的event统计
4. 初始化全局console日志，并作为target加入
5. 处理命令行参数
6. 处理环境变量
7. 设置分布式节点名称
8. 处理所有的帮助信息
9. 初始化以下子系统
   1.  healState 
   2.  notification 
   3.  BucketMetadata 
   4.  BucketMonitor 
   5.  ConfigSys 
   6.  IAM 
   7.  Policy
   8.  Lifecycle
   9.  BucketSSEConfig
   10. BucketObjectLock
   11. BucketQuota
   12. BucketVersioning
   13. BucketTarget
10. https 启用后的证书检查
11. 升级检查
12. 根据操作系统进程的最大内存，fd，线程数设置
13. 配置路由
    1.  分布式模式下，注册以下路由
        1.  StorageREST
        2.  PeerREST
        3.  BootstrapREST
        4.  LockREST
    2. STS 相关路由
    3. ADMIN 相关路由
    4. HealthCheck 相关路由
    5. Metrics 相关路由
    6. Web 相关路由
    7. API 相关路由
14. 注册以下hook
	// 处理未初始化object 层的重定向
	setRedirectHandler,
	// 设置 请求头 x-amz-request-id 字段.
	addCustomHeaders,
	// 添加头部安全字段例如 Content-Security-Policy.
	addSecurityHeaders,
	// 转发path style 请求到真正的主机上
	setBucketForwardingHandler,
	// 验证请求
	setRequestValidityHandler,
	// 统计
	setHTTPStatsHandler,
	// 限制请求大小
	setRequestSizeLimitHandler,
	// 限制请求头大小
	setRequestHeaderSizeLimitHandler,
	// 添加 'crossdomain.xml' 策略来处理 legacy flash clients.
	setCrossDomainPolicy,
	// 重定向一些预定义的浏览器请求到静态路由上
	setBrowserRedirectHandler,
	// 如果请求是restricted buckets 则验证
	setReservedBucketHandler,
	// 为所有浏览器请求添加cache
	setBrowserCacheControlHandler,
	// 验证所有请求流量，以便有有效期的标头
	setTimeValidityHandler,
	// 验证所有的url，以便使客户端收到不受支持的url的报错
	setIgnoreResourcesHandler,
	// 验证授权
	setAuthHandler,
	// 一些针对ssl特殊的处理
	setSSETLSHandler,
	// 筛选http头，这些标记作为meta信息保留，仅供内部使用
	filterReservedMetadata,
15. 注册最外层hook
    1.  criticalErrorHandler(处理panic)
    2.  corsHandler(处理CORS)
16. 如果是纠偏码模式
    1.  验证配置
17. 初始化 Object 层
18. 设置 deploment 的id
19. 如果是纠偏码模式
    1.  初始化自动 Heal
    2.  初始化后台 Replication
    3.  初始化后台 Transition
20. 初始化 DataCrawler
21. 初始化server
22. 如果启用缓存，初始化缓存层
23. 打印启动信息
24. 验证认证信息是否是默认认证信息，如果是，提示修改
# 博客内容: https://fengmeng.xyz/p/go-minio/
# code 注解基于新的 version: RELEASE.2023-09-04T19-57-37Z-2-g812f5a02d,内容有出入
```


### serverm_main 主函数分析

```go
// serverMain handler called for 'minio server' command.
func serverMain(ctx *cli.Context) {
  //注册系统关闭信号量
	signal.Notify(globalOSSignalCh, os.Interrupt, syscall.SIGTERM, syscall.SIGQUIT)

	go handleSignals()
	// 设置性能分析
	setDefaultProfilerRates()

	// Handle all server environment vars.
  // 处理环境变量设置 各类参数,serverURL,globalFSOSync,rootDiskSize,domains,domainIPs
	serverHandleEnvVars()

	// Handle all server command args.
  // 处理命令行参数
  // 跟踪serverHandleCmdArgs的执行->ctx 中携带的参数,并设置ttl定期刷新 DNS 缓存
	bootstrapTrace("serverHandleCmdArgs", func() {
    // 并生成Endpoint结构
		serverHandleCmdArgs(ctx)
	})

	// Initialize globalConsoleSys system
	bootstrapTrace("newConsoleLogger", func() {
		globalConsoleSys = NewConsoleLogger(GlobalContext)
		logger.AddSystemTarget(GlobalContext, globalConsoleSys)

		// Set node name, only set for distributed setup.
    // 设置分布式节点名称, globalLocalNodeName在serverHandleCmdArgs(ctx) 函数中处理了
		globalConsoleSys.SetNodeName(globalLocalNodeName)
	})

	// Perform any self-tests
  //自检查
	bootstrapTrace("selftests", func() {
    // 执行自检以确保 bitrot算法计算正确的校验和
		bitrotSelfTest()
    // 执行自检以确保纠察算法计算预期的纠察码
		erasureSelfTest()
    // 执行自检以确保压缩算法完成往返
		compressSelfTest()
	})

	// Initialize KMS configuration
	bootstrapTrace("handleKMSConfig", handleKMSConfig)

	// Initialize all help
  // 处理所有的帮助信息
	bootstrapTrace("initHelp", initHelp)

	// Initialize all sub-systems
	bootstrapTrace("initAllSubsystems", func() {
    // 初始化子系统
		initAllSubsystems(GlobalContext)
	})

// Is distributed setup, error out if no certificates are found for HTTPS endpoints.
// 如果是分布式设置，如果没有找到 HTTPS 端点的证书，则产生错误。
if globalIsDistErasure {
    // 如果全局设置为分布式 Erasure 模式

    // 如果全局端点要求使用 HTTPS 但未启用 TLS 加密
    if globalEndpoints.HTTPS() && !globalIsTLS {
        // 产生一个致命错误，指示未找到证书并且无法启动服务器。
        logger.Fatal(config.ErrNoCertsAndHTTPSEndpoints(nil), "Unable to start the server")
    }

    // 如果全局端点要求使用 HTTP 但启用了 TLS 加密
    if !globalEndpoints.HTTPS() && globalIsTLS {
        // 产生一个致命错误，指示找到了证书但无法启动服务器。
        logger.Fatal(config.ErrCertsAndHTTPEndpoints(nil), "Unable to start the server")
    }
}


// Check for updates in non-blocking manner.
// 在非阻塞方式下检查更新。

go func() {
    // 如果全局 CLI 上下文不是安静模式且未禁用就地更新
    if !globalCLIContext.Quiet && !globalInplaceUpdateDisabled {
        // 检查来自 dl.min.io 的新更新。
        bootstrapTrace("checkUpdate", func() {
            // 使用 getMinioMode() 获取 Minio 的运行模式，并检查是否有新的更新。
            checkUpdate(getMinioMode())
        })
    }
}()


	// Set system resources to maximum.
  // 设置最大系统资源 根据操作系统进程的最大内存，fd，线程数设置
	bootstrapTrace("setMaxResources", func() {
		_ = setMaxResources()
	})

	// Verify kernel release and version.
  // 检查 kenl 版本
	if oldLinux() {
		logger.Info(color.RedBold("WARNING: Detected Linux kernel version older than 4.0.0 release, there are some known potential performance problems with this kernel version. MinIO recommends a minimum of 4.x.x linux kernel version for best performance"))
	}

maxProcs := runtime.GOMAXPROCS(0)
// 获取当前系统的 GOMAXPROCS 数量，并将其赋值给 maxProcs 变量。
cpuProcs := runtime.NumCPU()
// 获取当前系统的 CPU 核心数量，并将其赋值给 cpuProcs 变量。

if maxProcs < cpuProcs {
    // 如果当前的 GOMAXPROCS 数量小于 CPU 核心数量，发出警告信息。
    logger.Info(color.RedBoldf("WARNING: Detected GOMAXPROCS(%d) < NumCPU(%d), please make sure to provide all PROCS to MinIO for optimal performance", maxProcs, cpuProcs))
}
// 此段代码用于检查当前的 GOMAXPROCS 数量是否小于 CPU 核心数量，如果是，会发出警告信息。
// 这个警告信息提醒用户可以通过适当增加 GOMAXPROCS 数量来提高 MinIO 服务器的性能。

var getCert certs.GetCertificateFunc
// 声明一个名为 getCert 的变量，其类型为 certs.GetCertificateFunc。

if globalTLSCerts != nil {
    // 如果全局变量 globalTLSCerts 不为空（即已经配置了 TLS 证书），
    // 则将 globalTLSCerts.GetCertificate 赋值给 getCert 变量。
    getCert = globalTLSCerts.GetCertificate
}


// Configure server.
bootstrapTrace("configureServer", func() {
    // 配置服务器处理程序,传入之前处理好的 globalEndpoints
    handler, err := configureServerHandler(globalEndpoints)
    if err != nil {
        logger.Fatal(config.ErrUnexpectedError(err), "Unable to configure one of server's RPC services")
    }

    // 创建 HTTP 服务器。
    httpServer := xhttp.NewServer(getServerListenAddrs()).
 				 // 设置错误处理器,跨域处理器
        UseHandler(setCriticalErrorHandler(corsHandler(handler))).
        UseTLSConfig(newTLSConfig(getCert)).
        UseShutdownTimeout(ctx.Duration("shutdown-timeout")).
        UseIdleTimeout(ctx.Duration("idle-timeout")).
        UseReadHeaderTimeout(ctx.Duration("read-header-timeout")).
        UseBaseContext(GlobalContext).
        UseCustomLogger(log.New(io.Discard, "", 0)). // 关闭 Go stdlib 的随机日志记录
  			// globalTCPOptions设置 tcp
        UseTCPOptions(globalTCPOptions)

    // 设置 HTTP 服务器的跟踪选项。
    httpServer.TCPOptions.Trace = bootstrapTraceMsg

    // 启动 HTTP 服务器的主 goroutine。
    go func() {
      // 返回一个function: serveFn
      // http server 重点是配置 控制器 handler => handler.ServeHTTP(w, r)
        serveFn, err := httpServer.Init(GlobalContext, func(listenAddr string, err error) {
            logger.LogIf(GlobalContext, fmt.Errorf("Unable to listen on `%s`: %v", listenAddr, err))
        })
        if err != nil {
            globalHTTPServerErrorCh <- err
            return
        }
      // serveFn 在这里真正执行
        globalHTTPServerErrorCh <- serveFn()
    }()

    // 设置全局 HTTP 服务器。
    setHTTPServer(httpServer)
})

// 根据特定条件执行相应的系统配置验证和服务冻结操作，以确保服务器在启动时具备必要的配置和准备状态。
if globalIsDistErasure {
    bootstrapTrace("verifying system configuration", func() {
        // 在分布式Erasure模式下，验证系统配置和设置。
        if err := verifyServerSystemConfig(GlobalContext, globalEndpoints); err != nil {
            logger.Fatal(err, "Unable to start the server")
        }
    })
}

if !globalDisableFreezeOnBoot {
    // 冻结服务，直到桶通知子系统初始化完成。
    bootstrapTrace("freezeServices", freezeServices)
}

  // 创建新的对象层
	var newObject ObjectLayer
	bootstrapTrace("newObjectLayer", func() {
		var err error
    // 其中单点模式就是只传了一个Endpoint给minio，使用的是文件系统的操作方式，即minio自带的单点模式下的文件对象操作结构体FSObjects
		newObject, err = newObjectLayer(GlobalContext, globalEndpoints)
		if err != nil {
			logFatalErrs(err, Endpoint{}, true)
		}
	})
  // 设置全局部署ID和MinIO版本信息
	xhttp.SetDeploymentID(globalDeploymentID)
	xhttp.SetMinIOVersion(Version)
	// 处理全局节点信息
	for _, n := range globalNodes {
		nodeName := n.Host
		if n.IsLocal {
			nodeName = globalLocalNodeName
		}
    // 在globalNodeNamesHex map 中对 非自身的节点的构建一个空struct{}{}
    // 用来存储对其他节点操作的信息
		nodeNameSum := sha256.Sum256([]byte(nodeName + globalDeploymentID))
		globalNodeNamesHex[hex.EncodeToString(nodeNameSum[:])] = struct{}{}
	}

	bootstrapTrace("newSharedLock", func() {
    // 创建全局共享锁: leader.lock 用来选举 leader
		globalLeaderLock = newSharedLock(GlobalContext, newObject, "leader.lock")
	})

	// Enable background operations on
	//
	// - Disk auto healing
	// - MRF (most recently failed) healing
	// - Background expiration routine for lifecycle policies
	bootstrapTrace("initAutoHeal", func() {
    // 初始化 Disk自动修复（Disk auto healing）
		initAutoHeal(GlobalContext, newObject)
	})
  // 初始化 most recently failed 的 healing策略
	bootstrapTrace("initHealMRF", func() {
		initHealMRF(GlobalContext, newObject)
	})
  // 初始化 后台生命周期策略过期例行程序
	bootstrapTrace("initBackgroundExpiry", func() {
		initBackgroundExpiry(GlobalContext, newObject)
	})

	var err error
	bootstrapTrace("initServerConfig", func() {
		if err = initServerConfig(GlobalContext, newObject); err != nil {
		... 错误处理
		}
		// 严格的AWS S3兼容性 警告
		if !globalCLIContext.StrictS3Compat {
			logger.Info(color.RedBold("WARNING: Strict AWS S3 compatible incoming PUT, POST content payload validation is turned off, caution is advised do not use in production"))
		}
	})
	// 默认凭证（Default Credentials）警告
	if globalActiveCred.Equal(auth.DefaultCredentials) {
		msg := fmt.Sprintf("WARNING: Detected default credentials '%s', we recommend that you change these values with 'MINIO_ROOT_USER' and 'MINIO_ROOT_PASSWORD' environment variables",
			globalActiveCred)
		logger.Info(color.RedBold(msg))
	}

	// Initialize users credentials and policies in background right after config has initialized.
	go func() {
    // 配置初始化后立即初始化用户凭证和策略 -- IAM
		bootstrapTrace("globalIAMSys.Init", func() {
			globalIAMSys.Init(GlobalContext, newObject, globalEtcdClient, globalRefreshIAMInterval)
		})

		// Initialize Console UI
    // 初始化控制台 server
	...
				srv, err := initConsoleServer()
				setConsoleSrv(srv)
				newConsoleServerFn().Serve()
		...

		// if we see FTP args, start FTP if possible
    go startFTPServer(ctx)
		
		// If we see SFTP args, start SFTP if possible
    go startSFTPServer(ctx)

		// Initialize data scanner.
    initDataScanner(GlobalContext, newObject)

		// Initialize background replication
    initBackgroundReplication(GlobalContext, newObject)
		globalTransitionState.Init(newObject)
    
		// Initialize batch job pool.
    globalBatchJobPool = newBatchJobPool(GlobalContext, newObject, 100)

		// Initialize the license update job
    initLicenseUpdateJob(GlobalContext, newObject)
    
			// Initialize transition tier configuration manager
      // 初始化 层级配置管理器
      globalTierConfigMgr.Init(GlobalContext, newObject)
 
		// initialize the new disk cache objects.
    // 磁盘的 cache 对象层
		if globalCacheConfig.Enabled {
			logger.Info(color.Yellow("WARNING: Drive caching is deprecated for single/multi drive MinIO setups."))
			var cacheAPI CacheObjectLayer
			cacheAPI, err = newServerCacheObjects(GlobalContext, globalCacheConfig)
			logger.FatalIf(err, "Unable to initialize drive caching")

			setCacheObjectLayer(cacheAPI)
		}

		// Initialize bucket notification system.
    // 初始化 bucket 的事件系统
		bootstrapTrace("initBucketTargets", func() {
			logger.LogIf(GlobalContext, globalEventNotifier.InitBucketTargets(GlobalContext, newObject))
		})

		var buckets []BucketInfo
		// List buckets to initialize bucket metadata sub-sys.
    // 初始化 bucket的元数据的子系统
			buckets, err = newObject.ListBuckets(GlobalContext, BucketOptions{})
      // Initialize bucket metadata sub-system.
      globalBucketMetadataSys.Init(GlobalContext, buckets, newObject)
 
		// initialize replication resync state. 复制重新同步状态
			globalReplicationPool.initResync(GlobalContext, buckets, newObject)

		// Initialize site replication manager after bucket metadata
    // 备份管理器
			globalSiteReplicationSys.Init(GlobalContext, newObject)
 
		// Initialize quota manager.配额管理器
    globalBucketQuotaSys.Init(newObject)
    
		// Populate existing buckets to the etcd backend 流行的 etcd 客户端
			// Background this operation. 初始化Federator后端
     // 将现有存储桶的信息写入 etcd 后端，以支持联合命名空间的操作
      go initFederatorBackend(buckets, newObject)

		// Prints the formatted startup message, if err is not nil then it prints additional information as well.用于打印格式化的启动消息
		printStartupMessage(getAPIEndpoints(), err)

		// Print a warning at the end of the startup banner so it is more noticeable
    // 打印警告信息，表示标准奇偶校验设置为 0 可能会导致数据丢失
		if newObject.BackendInfo().StandardSCParity == 0 {
			logger.Error("Warning: The standard parity is set to 0. This can lead to data loss.")
		}
	}()
	// 获取 region
	region := globalSite.Region
	if region == "" {
		region = "us-east-1"
	}
	bootstrapTrace("globalMinioClient", func() {
    // 客户端用于与其他 MinIO 节点通信。它使用服务器配置中的访问密钥、安全设置和区域信息进行初始化
		globalMinioClient, err = minio.New(globalLocalNodeName, &minio.Options{
			Creds:     credentials.NewStaticV4(globalActiveCred.AccessKey, globalActiveCred.SecretKey, ""),
			Secure:    globalIsTLS,
			Transport: globalProxyTransport,
			Region:    region,
		})
		logger.FatalIf(err, "Unable to initialize MinIO client")
	})

	// Add User-Agent to differentiate the requests.
  // 置应用程序信息，以区分请求的用户代理
	globalMinioClient.SetAppInfo("minio-perf-test", ReleaseTag)

	if serverDebugLog {
    // 打印调试模式已启用的信息以及当前环境变量的设置，但不会显示敏感的凭据信息
		logger.Info("== DEBUG Mode enabled ==")
		logger.Info("Currently set environment settings:")
		ks := []string{
			config.EnvAccessKey,
			config.EnvSecretKey,
			config.EnvRootUser,
			config.EnvRootPassword,
		}
		for _, v := range os.Environ() {
			// Do not print sensitive creds in debug.
			if slices.Contains(ks, strings.Split(v, "=")[0]) {
				continue
			}
			logger.Info(v)
		}
		logger.Info("======")
	}
	// 发送通知，表示 MinIO 服务器已准备好接受连接。
	daemon.SdNotify(false, daemon.SdNotifyReady)
	// 使用 <-globalOSSignalCh 阻塞主 goroutine，直到收到操作系统的信号，以便服务器能够正确地维护和退出
	<-globalOSSignalCh
}



```

#### 性能分析设置

```go
// 性能分析设置
func setDefaultProfilerRates() {
  // 分配 128kb 内存后采样
	runtime.MemProfileRate = 128 << 10 // 512KB -> 128K - Must be constant throughout application lifetime.
  // 关闭互斥锁分析
	runtime.SetMutexProfileFraction(0) // Disable until needed
  // 关闭阻塞分析
	runtime.SetBlockProfileRate(0)     // Disable until needed
}

```





#### 监控 chan 执行关闭操作

```go
// 在 server 命令里启动新的协程监控 chan
func serverMain(ctx *cli.Context) {
	signal.Notify(globalOSSignalCh, os.Interrupt, syscall.SIGTERM, syscall.SIGQUIT)

	go handleSignals()
}

func handleSignals() {
	// Custom exit function
  // 构建 exit 函数
	exit := func(success bool) {
		// If global profiler is set stop before we exit.
    // 关闭  global profiler
		globalProfilerMu.Lock()
		defer globalProfilerMu.Unlock()
		for _, p := range globalProfiler {
			p.Stop()
		}

		if success {
			os.Exit(0)
		}

		os.Exit(1)
	}

	stopProcess := func() bool {
		// send signal to various go-routines that they need to quit.
    // 向相关的需要 quit 的子 context 传播 cancel: 轮询子 ctx 的 chan,发送值到 chan
		cancelGlobalContext()
		// 锁住globalObjLayerMutex 关闭 [ http server, ObjectLayer, ConsoleServer]
		if httpServer := newHTTPServerFn(); httpServer != nil {
			if err := httpServer.Shutdown(); err != nil && !errors.Is(err, http.ErrServerClosed) {
				logger.LogIf(context.Background(), err)
			}
		}
		// 关闭 ObjectLayer
		if objAPI := newObjectLayerFn(); objAPI != nil {
			logger.LogIf(context.Background(), objAPI.Shutdown(context.Background()))
		}
		// 关闭 控制管理的 server
		if srv := newConsoleServerFn(); srv != nil {
			logger.LogIf(context.Background(), srv.Shutdown())
		}

		if globalEventNotifier != nil {
			globalEventNotifier.RemoveAllBucketTargets()
		}

		return true
	}

	for {
		select {
      // 对globalHTTPServerErrorCh ,globalOSSignalCh直接关
		case err := <-globalHTTPServerErrorCh:
			logger.LogIf(context.Background(), err)
			exit(stopProcess())
		case osSignal := <-globalOSSignalCh:
			logger.Info("Exiting on signal: %s", strings.ToUpper(osSignal.String()))
			daemon.SdNotify(false, daemon.SdNotifyStopping)
			exit(stopProcess())
		case signal := <-globalServiceSignalCh:
			switch signal {
			case serviceRestart:
				logger.Info("Restarting on service signal")
				daemon.SdNotify(false, daemon.SdNotifyReloading)
				stop := stopProcess()
				rerr := restartProcess()
				if rerr == nil {
					daemon.SdNotify(false, daemon.SdNotifyReady)
				}
				logger.LogIf(context.Background(), rerr)
				exit(stop && rerr == nil)
			case serviceStop:
				logger.Info("Stopping on service signal")
				daemon.SdNotify(false, daemon.SdNotifyStopping)
				exit(stopProcess())
			}
		}
	}
}

```

#### 命令行参数解析

```go
func serverHandleCmdArgs(ctx *cli.Context) {
    // 处理通用命令参数
    handleCommonCmdArgs(ctx)

    // 检查本地服务器地址是否有效
    logger.FatalIf(CheckLocalServerAddr(globalMinioAddr), "无法验证传递的参数")

    var err error
    var setupType SetupType

    // 检查和加载TLS证书
    globalPublicCerts, globalTLSCerts, globalIsTLS, err = getTLSConfig()
    logger.FatalIf(err, "无法加载TLS配置")

    // 检查和加载根证书颁发机构
    globalRootCAs, err = certs.GetRootCAs(globalCertsCADir.Get())
    logger.FatalIf(err, "无法读取根证书颁发机构 (%v)", err)

    // 将全局公共证书添加到全局根证书颁发机构中
    for _, publicCrt := range globalPublicCerts {
        globalRootCAs.AddCert(publicCrt)
    }

    // 注册根证书颁发机构供远程环境使用
    env.RegisterGlobalCAs(globalRootCAs)

    // 创建服务器端点和设置类型
    globalEndpoints, setupType, err = createServerEndpoints(globalMinioAddr, serverCmdArgs(ctx)...)
    logger.FatalIf(err, "无效的命令行参数")
    globalNodes = globalEndpoints.GetNodes()

    // 获取本地节点的对等节点的值, 并计算其哈希值
    globalLocalNodeName = GetLocalPeer(globalEndpoints, globalMinioHost, globalMinioPort)
    nodeNameSum := sha256.Sum256([]byte(globalLocalNodeName))
    globalLocalNodeNameHex = hex.EncodeToString(nodeNameSum[:])

    // 初始化并检查服务运行的网络接口
    setGlobalInternodeInterface(ctx.String("interface"))

    // 允许传输为HTTP/1.1以用于代理
    globalProxyTransport = NewCustomHTTPProxyTransport()()
    globalProxyEndpoints = GetProxyEndpoints(globalEndpoints)
    globalInternodeTransport = NewInternodeHTTPTransport()()
    globalRemoteTargetTransport = NewRemoteTargetHTTPTransport(false)()

    // 创建请求转发器
    globalForwarder = handlers.NewForwarder(&handlers.Forwarder{
        PassHost:     true,
        RoundTripper: NewHTTPTransportWithTimeout(1 * time.Hour),
        Logger: func(err error) {
            if err != nil && !errors.Is(err, context.Canceled) {
                logger.LogIf(GlobalContext, err)
            }
        },
    })

    // 设置TCP选项
    globalTCPOptions = xhttp.TCPOptions{
        UserTimeout: int(ctx.Duration("conn-user-timeout").Milliseconds()),
        Interface:   ctx.String("interface"),
    }

    // 检查端口的可用性
    logger.FatalIf(xhttp.CheckPortAvailability(globalMinioHost, globalMinioPort, globalTCPOptions), "无法启动服务器")

    // 设置全局Erasure标志 
    globalIsErasure = (setupType == ErasureSetupType)
    globalIsDistErasure = (setupType == DistErasureSetupType)
    if globalIsDistErasure {
        globalIsErasure = true
    }
    globalIsErasureSD = (setupType == ErasureSDSetupType)

    // 设置连接读取和写入的截止时间
    globalConnReadDeadline = ctx.Duration("conn-read-deadline")
    globalConnWriteDeadline = ctx.Duration("conn-write-deadline")
}	
```



#### 加入DNS的Cache的刷新和停止hook

```go


func handleCommonCmdArgs(){
  ...
// Check if we have configured a custom DNS cache TTL.
// 
	dnsTTL := ctx.Duration("dns-cache-ttl")
	if dnsTTL <= 0 {
		dnsTTL = 10 * time.Minute
	}

	// Call to refresh will refresh names in cache.
	go func() {
		// Baremetal setups set DNS refresh window up to dnsTTL duration.
    // 定时触发器
		t := time.NewTicker(dnsTTL)
		defer t.Stop()
		for {
			select {
			case <-t.C:
				globalDNSCache.Refresh()
			// 一旦接受 done 停止 for-select,则停止 dns 的刷新
			case <-GlobalContext.Done():
				return
			}
    }
	}()
}
```

#### server.Init

```go
// Init - init HTTP server
func (srv *Server) Init(listenCtx context.Context, listenErrCallback func(listenAddr string, err error)) (serve func() error, err error) {
	... config
	handler := srv.Handler // if srv.Handler holds non-synced state -> possible data race

	// Create new HTTP listener.
  // 设置tcp socket 服务端启动的 addrs 
	var listener *httpListener
	listener, listenErrs := newHTTPListener(
		listenCtx,
		srv.Addrs,
		srv.TCPOptions,
	)
	... error处理 -> listenErrCallback(srv.Addrs[i], listenErrs[i])...
	

	// Wrap given handler to do additional
	// * return 503 (service unavailable) if the server in shutdown.
	wrappedHandler := http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
		// If server is in shutdown.
    // 包装 handler 增加inShutdown的专门处理
		if atomic.LoadUint32(&srv.inShutdown) != 0 {
			// To indicate disable keep-alives
			w.Header().Set("Connection", "close")
			w.WriteHeader(http.StatusServiceUnavailable)
			w.Write([]byte(http.ErrServerClosed.Error()))
			return
		}

		atomic.AddInt32(&srv.requestCount, 1)
		defer atomic.AddInt32(&srv.requestCount, -1)

		// Handle request using passed handler.
		handler.ServeHTTP(w, r)
	})
  // 再设置 srv
	srv.listenerMutex.Lock()
	srv.Handler = wrappedHandler
	srv.listener = listener
	srv.listenerMutex.Unlock()

	var l net.Listener = listener
	if tlsConfig != nil {
    // 创建自定义的网络监听器，该监听器用于接受来自内部监听器的连接请求。
    // 返回的监听器对象实现了 net.Listener 接口，可以用于接受和处理客户端连接请求
		l = tls.NewListener(listener, tlsConfig)
	}
	// 返回的函数
	serve = func() error {
    // 走到 http 包的func (srv *Server) Serve(l net.Listener) 
    // 真正启动一个 http server 服务
		return srv.Server.Serve(l)
	}

	return
}
```

#### 获取共享锁newSharedLock

```go
type sharedLock struct {
	lockContext chan LockContext
}
// LockContext带一个常规的 ctx 和设置CancelFunc
type LockContext struct {
	ctx    context.Context
	cancel context.CancelFunc
}

// 将获取到的锁上下文 lkctx 发送到 lockContext 通道中，
// 以便其他协程可以获取到共享锁的上下文，从而进行需要锁的操作。
func (ld sharedLock) backgroundRoutine(ctx context.Context, objAPI ObjectLayer, lockName string) {
	for {
    // 获取到  objAPI.NewNSLock 对象层的空间锁
		locker := objAPI.NewNSLock(minioMetaBucket, lockName)
    // 获得其互斥锁,得不到会阻塞,并返回获取锁的上下文 lkctx，以及可能出现的错误 err
		lkctx, err := locker.GetLock(ctx, sharedLockTimeout)
    // 错误
		if err != nil {
      // 失败了会进入下一轮的 for 循环获取锁
			continue
		}
	// 保持锁在使用的状态
	keepLock:
		for {
			select {
			case <-ctx.Done():
        // 如果传入的上下文 ctx 被取消，退出循环，结束协程
				return
			case <-lkctx.Context().Done():
				// 如果锁的上下文被取消，这可能是因为锁在分布式环境中失去了一致性
				// 在这种情况下，跳出内部循环，尝试重新获取锁
				break keepLock
			case ld.lockContext <- lkctx:
        // 将获取到的锁上下文传递给需要锁的代码块,发送后又回到第二次 for-select 等待锁的释放并退出该协程
				// Send the lock context to anyone asking for it
			}
		}
	}
}

// 工厂函数，用于创建共享锁的实例
func newSharedLock(ctx context.Context, objAPI ObjectLayer, lockName string) *sharedLock {
    // 创建 sharedLock 结构的实例
    l := &sharedLock{
        lockContext: make(chan LockContext),
    }
    
    // 启动后台协程，用于处理锁的后台任务
    go l.backgroundRoutine(ctx, objAPI, lockName)

    // 返回创建的 sharedLock 实例
    return l
}

```

### ObjectLayer 对象存储层

定义了MinIO对象存储服务针对对象操作的所有API,是混合云的重点实现,不同的云存储会有不同的基于 SDK 的实现.

这个接口定义了用于管理对象存储系统的多种操作，实现了对象存储的核心功能。不同的对象存储系统可以通过实现这个接口来提供不同的后端存储支持。

比如 :  Google 提供的云存储服务Google Cloud Storage  会通过 GCS的sdk获取实际存储在gcs的桶,包装成 s3标准, 供ObjectLayer的gcs实现包装使用,再提供给 minio 的调用方使用

* ObjectLayer封装的接口,重点在于描述一个ObjectLayer必须提供的动作|操作
* 只要实现了接口函数的结构体都是ObjectLayer

```go
// ObjectLayer implements primitives for object API layer.
type ObjectLayer interface {
	// Locking operations on object.
  // 对象上的空间锁
	NewNSLock(bucket string, objects ...string) RWLocker

	// Storage operations. 存储层面的操作
	Shutdown(context.Context) error
	NSScanner(ctx context.Context, updates chan<- DataUsageInfo, wantCycle uint32, scanMode madmin.HealScanMode) error
	BackendInfo() madmin.BackendInfo
	StorageInfo(ctx context.Context) StorageInfo
	LocalStorageInfo(ctx context.Context) StorageInfo

	// Bucket operations.桶操作
  // 增删改查 bucket
	MakeBucket(ctx context.Context, bucket string, opts MakeBucketOptions) error
	GetBucketInfo(ctx context.Context, bucket string, opts BucketOptions) (bucketInfo BucketInfo, err error)
	ListBuckets(ctx context.Context, opts BucketOptions) (buckets []BucketInfo, err error)
	DeleteBucket(ctx context.Context, bucket string, opts DeleteBucketOptions) error
	ListObjects(ctx context.Context, bucket, prefix, marker, delimiter string, maxKeys int) (result ListObjectsInfo, err error)
  // 列出存储桶中的对象（版本2）
	ListObjectsV2(ctx context.Context, bucket, prefix, continuationToken, delimiter string, maxKeys int, fetchOwner bool, startAfter string) (result ListObjectsV2Info, err error)
	ListObjectVersions(ctx context.Context, bucket, prefix, marker, versionMarker, delimiter string, maxKeys int) (result ListObjectVersionsInfo, err error)
	// Walk lists all objects including versions, delete markers.
  // 对包含的版本对象遍历
	Walk(ctx context.Context, bucket, prefix string, results chan<- ObjectInfo, opts ObjectOptions) error

	// Object operations.对象操作

	// GetObjectNInfo returns a GetObjectReader that satisfies the
	// ReadCloser interface. The Close method runs any cleanup
	// functions, so it must always be called after reading till EOF
	//
	// IMPORTANTLY, when implementations return err != nil, this
	// function MUST NOT return a non-nil ReadCloser.
  // GetObjectNInfo 返回一个满足 ReadCloser 接口的 GetObjectReader
	GetObjectNInfo(ctx context.Context, bucket, object string, rs *HTTPRangeSpec, h http.Header, opts ObjectOptions) (reader *GetObjectReader, err error)
	GetObjectInfo(ctx context.Context, bucket, object string, opts ObjectOptions) (objInfo ObjectInfo, err error)
	PutObject(ctx context.Context, bucket, object string, data *PutObjReader, opts ObjectOptions) (objInfo ObjectInfo, err error)
	CopyObject(ctx context.Context, srcBucket, srcObject, destBucket, destObject string, srcInfo ObjectInfo, srcOpts, dstOpts ObjectOptions) (objInfo ObjectInfo, err error)
	DeleteObject(ctx context.Context, bucket, object string, opts ObjectOptions) (ObjectInfo, error)
	DeleteObjects(ctx context.Context, bucket string, objects []ObjectToDelete, opts ObjectOptions) ([]DeletedObject, []error)
	TransitionObject(ctx context.Context, bucket, object string, opts ObjectOptions) error
	RestoreTransitionedObject(ctx context.Context, bucket, object string, opts ObjectOptions) error

	// Multipart operations. 分段处理操作
  // 列出所有分段上传任务
	ListMultipartUploads(ctx context.Context, bucket, prefix, keyMarker, uploadIDMarker, delimiter string, maxUploads int) (result ListMultipartsInfo, err error)
  //  创建一个新的分段上传任务
	NewMultipartUpload(ctx context.Context, bucket, object string, opts ObjectOptions) (result *NewMultipartUploadResult, err error)
  // 复制分段上传的部分
	CopyObjectPart(ctx context.Context, srcBucket, srcObject, destBucket, destObject string, uploadID string, partID int,
		startOffset int64, length int64, srcInfo ObjectInfo, srcOpts, dstOpts ObjectOptions) (info PartInfo, err error)
  // 存储分段上传的部分
	PutObjectPart(ctx context.Context, bucket, object, uploadID string, partID int, data *PutObjReader, opts ObjectOptions) (info PartInfo, err error)
  // 获取分段上传任务的信息
	GetMultipartInfo(ctx context.Context, bucket, object, uploadID string, opts ObjectOptions) (info MultipartInfo, err error)
  // 列出对象的分段。
	ListObjectParts(ctx context.Context, bucket, object, uploadID string, partNumberMarker int, maxParts int, opts ObjectOptions) (result ListPartsInfo, err error)
  // 中止分段上传任务
	AbortMultipartUpload(ctx context.Context, bucket, object, uploadID string, opts ObjectOptions) error
  //  完成分段上传任务
	CompleteMultipartUpload(ctx context.Context, bucket, object, uploadID string, uploadedParts []CompletePart, opts ObjectOptions) (objInfo ObjectInfo, err error)

  // 获取属于指定池和集合的磁盘
	GetDisks(poolIdx, setIdx int) ([]StorageAPI, error) // return the disks belonging to pool and set.
  // 获取每个池的纠察码横跨大小列表
	SetDriveCounts() []int                              // list of erasure stripe size for each pool in order.
 
	// Healing operations.
  // 格式化存储系统
	HealFormat(ctx context.Context, dryRun bool) (madmin.HealResultItem, error)
  // 修复存储桶
	HealBucket(ctx context.Context, bucket string, opts madmin.HealOpts) (madmin.HealResultItem, error)
  // 修复对象
	HealObject(ctx context.Context, bucket, object, versionID string, opts madmin.HealOpts) (madmin.HealResultItem, error)
  // 修复多个对象
	HealObjects(ctx context.Context, bucket, prefix string, opts madmin.HealOpts, fn HealObjectFn) error
  // 检查废弃的分段
	CheckAbandonedParts(ctx context.Context, bucket, object string, opts madmin.HealOpts) error

	// Returns health of the backend
  // 返回后端的健康状况
	Health(ctx context.Context, opts HealthOptions) HealthResult
	ReadHealth(ctx context.Context) bool

	// Metadata operations
  // 存储对象的元数据
	PutObjectMetadata(context.Context, string, string, ObjectOptions) (ObjectInfo, error)
  // 解除分层对象
	DecomTieredObject(context.Context, string, string, FileInfo, ObjectOptions) error

	// ObjectTagging operations
	PutObjectTags(context.Context, string, string, string, ObjectOptions) (ObjectInfo, error)
	GetObjectTags(context.Context, string, string, ObjectOptions) (*tags.Tags, error)
	DeleteObjectTags(context.Context, string, string, ObjectOptions) (ObjectInfo, error)
}
```






#### 擦除集Erasure Set

擦除集（Erasure Set）：是指一组纠察码集合，最大为32个驱动器，纠察码作为一种数据冗余技术相比于多副本以较低的数据冗余度提供足够的数据可靠性。擦除集中包含数据块与校验块，并且随机均匀的分布在各个节点上 

- 非通过备分完整副本来达到可靠性实现,如 hdfs
- 丢失的块通过存在的数据集和校验块计算能重新恢复出丢失块的数据, 通过位运算的奇偶校验等复杂的算法实现.

![img](https://img-blog.csdnimg.cn/img_convert/0ac5510e648bc3e10f64db614270080e.png)

>  data block: 数据块  parity block: 校验块
>
> 10 个 disk driver=> 6 data block  + 4 parity block
>
> 随即分布在磁盘中

#### 服务池Server Pool

服务器池（Server Pool）: 由一组MinIO节点组成一个存储池，池子中的所有节点以相同的命令启动 

![img](https://img-blog.csdnimg.cn/img_convert/086b4db87114732d2ebe65f2af012bee.png)

> 一个pool池子由3个节点和6个驱动disk driver，共18个驱动器组成9+9的擦除集；
>
> 池子中可能会包含多个擦除集 

#### Endpoint

在Minio中，端点（endpoints）是用于指定对象存储服务的位置和访问方式的配置元素。Minio支持不同类型的端点，包括URL样式端点（URL-style endpoints）和路径样式端点（path-style endpoints）。

1. **URL样式端点（URL-style endpoints）**：这种端点的配置类似于常见的URL，例如：`https://minio.example.com`。URL样式端点通常用于在生产环境中使用，支持HTTPS等安全协议。这是Minio推荐的端点配置方式。
2. **路径样式端点（path-style endpoints）**：这种端点的配置类似于文件系统路径，例如：`minio.example.com/bucket/object`。路径样式端点通常用于本地开发或测试环境中，也可以用于访问Minio服务

![img](https://pic4.zhimg.com/80/v2-76664bcf305925cdc123bb2b97669767_1440w.webp)

```go
// Endpoint 表示任何类型的端点（Endpoint）。
type Endpoint struct {
    // URL 包含端点的 URL 信息，包括协议、主机名、端口和路径等。
    *url.URL

    // IsLocal 是一个布尔值字段，用于指示端点是否是本地端点。
    // 如果 IsLocal 为 true，则表示端点位于本地主机上，否则表示端点是远程的。
    IsLocal bool

    // PoolIdx 端点池（Pool）索引,指示端点属于哪个池
    PoolIdx int

    // SetIdx  表示端点所属的 擦除 set集（Set）索引,从Endpoints[idx]获取endpoint结构
    SetIdx int

    // DiskIdx  表示端点所属的磁盘（Disk）索引.属于哪个磁盘
    // 磁盘是数据的物理存储位置。
    DiskIdx int
}

// CreateEndpoints - 验证并创建给定参数的新端点。
func CreateEndpoints(serverAddr string, args ...[]string) (Endpoints, SetupType, error) {
	var endpoints Endpoints
	var setupType SetupType
	var err error

	// 检查服务器地址是否对此主机有效。
	if err = CheckLocalServerAddr(serverAddr); err != nil {
		return endpoints, setupType, err
	}

	_, serverAddrPort := mustSplitHostPort(serverAddr)
  // 单点模式
	// 对于单个参数，返回单个驱动器设置。
	if len(args) == 1 && len(args[0]) == 1 {
		var endpoint Endpoint
		endpoint, err = NewEndpoint(args[0][0])
		if err != nil {
			return endpoints, setupType, err
		}
    // 设置为本地
		if err := endpoint.UpdateIsLocal(); err != nil {
			return endpoints, setupType, err
		}
		if endpoint.Type() != PathEndpointType {
			return endpoints, setupType, config.ErrInvalidEndpoint(nil).Msg("使用路径样式端点进行单节点设置")
		}
		// 单点就一个
		endpoint.SetPoolIndex(0)
		endpoint.SetSetIndex(0)
		endpoint.SetDiskIndex(0)

		endpoints = append(endpoints, endpoint)
    // ErasureSDSetupType 数据存储和冗余配置的类型
		setupType = ErasureSDSetupType

		// 检查是否存在跨设备挂载。
		if err = checkCrossDeviceMounts(endpoints); err != nil {
			return endpoints, setupType, config.ErrInvalidEndpoint(nil).Msg(err.Error())
		}

		return endpoints, setupType, nil
	}
  // 纠察码模式
	// 返回Endpoints 集合
	for setIdx, iargs := range args {
		// 将参数转换为端点
		eps, err := NewEndpoints(iargs...)
	 ...
		// 检查是否存在跨设备挂载。
		if err = checkCrossDeviceMounts(eps); err != nil {
			return endpoints, setupType, config.ErrInvalidErasureEndpoints(nil).Msg(err.Error())
		}

		for diskIdx := range eps {
			eps[diskIdx].SetSetIndex(setIdx)
			eps[diskIdx].SetDiskIndex(diskIdx)
		}

		endpoints = append(endpoints, eps...)
	}
...
  	setupType = DistErasureSetupType
	return endpoints, setupType, nil
}

```
#### server 命令中初始化newObjectLayer

```go
// Initialize object layer with the supplied disks, objectLayer is nil upon any error.
// 使用提供的磁盘初始化对象层
// endpointServerPools之前通过参数构建的 多个 endpoint终端存储点服务池
func newObjectLayer(ctx context.Context, endpointServerPools EndpointServerPools) (newObject ObjectLayer, err error) {
	return newErasureServerPools(ctx, endpointServerPools)
}


```

#### erasureServerPools

serverPools: <img src="/Users/linglingdai/Library/Application Support/typora-user-images/image-20230907041211040.png" alt="image-20230907041211040" style="zoom:50%;" />

serverpool: <img src="/Users/linglingdai/Library/Application Support/typora-user-images/image-20230907041438659.png" alt="image-20230907041438659" style="zoom:50%;" />

> serverpools[i] => erasureSets



```go
// 擦除集|纠察集的 服务池
type erasureServerPools struct {
  // poolMeta元数据的读写锁
	poolMetaMutex sync.RWMutex
  //擦除数据集服务池的元数据
	poolMeta      poolMeta
	// 再平衡元数据
	rebalMu   sync.RWMutex
	rebalMeta *rebalanceMeta
  // 部署的唯一标识符
	deploymentID     [16]byte
  // 存储分布式算法
	distributionAlgo string
  // 擦除集的集合
	serverPools []*erasureSets

	// Active decommission canceler 
  // 存储取消函数的切片，用于取消正在进行的存储节点退役操作
	decommissionCancelers []context.CancelFunc
  // 与其他endpoints连接通信的客户端池
	s3Peer *S3PeerSys
}

// Initialize new pool of erasure sets.
// 初始化 erasure sets 纠察集组成的存储池
func newErasureServerPools(ctx context.Context, endpointServerPools EndpointServerPools) (ObjectLayer, error) {
  // 定义相关的变量
	var (
		deploymentID       string
		distributionAlgo   string
		commonParityDrives int
		err                error
	
		formats      = make([]*formatErasureV3, len(endpointServerPools))
		storageDisks = make([][]StorageAPI, len(endpointServerPools))
		z            = &erasureServerPools{
			serverPools: make([]*erasureSets, len(endpointServerPools)),
			s3Peer:      NewS3PeerSys(endpointServerPools),
		}
	)

	var localDrives []StorageAPI
  // 第一个 endpoint 是不是本地的
	local := endpointServerPools.FirstLocal()
	for i, ep := range endpointServerPools {
    // 遍历纠察集存储池
		// If storage class is not set during startup, default values are used
		// -- Default for Reduced Redundancy Storage class is, parity = 2
		// -- Default for Standard Storage class is, parity = 2 - disks 4, 5
		// -- Default for Standard Storage class is, parity = 3 - disks 6, 7
		// -- Default for Standard Storage class is, parity = 4 - disks 8 to 16
    // 0: 默认奇偶磁盘数量尚未设置
		if commonParityDrives == 0 {
      // 计算默认奇偶磁盘数量
			commonParityDrives, err = ecDrivesNoConfig(ep.DrivesPerSet)
		}

    // 验证奇偶磁盘数量是否合法
		if err = storageclass.ValidateParity(commonParityDrives, ep.DrivesPerSet); err != nil {
			return nil, fmt.Errorf("parity validation returned an error: %w <- (%d, %d), for pool(%s)", err, commonParityDrives, ep.DrivesPerSet, humanize.Ordinal(i+1))
		}
    // 等待格式化
    // storageDisks[i]=> 存储磁盘的信息集
    // formats[i]=>存储池的格式化信息集 ，包括节点信息、部署ID、分布算法等。
		storageDisks[i], formats[i], err = waitForFormatErasure(local, ep.Endpoints, i+1,
			ep.SetCount, ep.DrivesPerSet, deploymentID, distributionAlgo)

		for _, storageDisk := range storageDisks[i] {
      // 根据storageDisk计算当前节点的本地纠察集的 driver
			if storageDisk != nil && storageDisk.IsLocal() {
				localDrives = append(localDrives, storageDisk)
			}
		}
    // all pools should have same deployment ID
    // 设置部署 id
		deploymentID = formats[i].ID

    // 分布算法
		if distributionAlgo == "" {
			distributionAlgo = formats[i].Erasure.DistributionAlgo
		}

		// Validate if users brought different DeploymentID pools.
		if deploymentID != formats[i].ID {
      // 验证所有存储池是否具有相同的部署ID，确保它们属于同一部署
			return nil, fmt.Errorf("all pools must have same deployment ID - expected %s, got %s for pool(%s)", deploymentID, formats[i].ID, humanize.Ordinal(i+1))
		}
		// 对每个 纠察集存储池 初始化它持有的  纠察集ErasureSet们
		z.serverPools[i], err = newErasureSets(ctx, ep, storageDisks[i], formats[i], commonParityDrives, i)
		if err != nil {
			return nil, err
		}

		if deploymentID != "" && bytes.Equal(z.deploymentID[:], []byte{}) {
			z.deploymentID = uuid.MustParse(deploymentID)
		}

		if distributionAlgo != "" && z.distributionAlgo == "" {
			z.distributionAlgo = distributionAlgo
		}
	}

	z.decommissionCancelers = make([]context.CancelFunc, len(z.serverPools))

	// initialize the object layer.
  // 全局对象层（Object Layer）为z，以便后续的操作能够访问对象层的功能
	setObjectLayer(z)

	r := rand.New(rand.NewSource(time.Now().UnixNano()))
	for {
    // 初始化 all pools 直到成功为止
		err := z.Init(ctx) // Initializes all pools.
		if err != nil {
			if !configRetriableErrors(err) {
				logger.Fatal(err, "Unable to initialize backend")
			}
			retry := time.Duration(r.Float64() * float64(5*time.Second))
			logger.LogIf(ctx, fmt.Errorf("Unable to initialize backend: %w, retrying in %s", err, retry))
			time.Sleep(retry)
			continue
		}
		break
	}

	globalLocalDrivesMu.Lock()
  // 加锁设置全局本地 derivers 为localDrives
	globalLocalDrives = localDrives
	defer globalLocalDrivesMu.Unlock()
 
	return z, nil
}

```

#### newErasureSets

- 1个纠察集ErasureSet有 N 个磁盘驱动器的dirver 
- 每个 driver 操作一个存储介质

```go
// Initialize new set of erasure coded sets.
// 初始化一组纠察集ErasureSet
func newErasureSets(ctx context.Context, endpoints PoolEndpoints, storageDisks []StorageAPI, format *formatErasureV3, defaultParityCount, poolIdx int) (*erasureSets, error) {
  // 总量
	setCount := len(format.Erasure.Sets)
  // deriver 数
	setDriveCount := len(format.Erasure.Sets[0])
	// endpoint 的字符串集合
	endpointStrings := make([]string, len(endpoints.Endpoints))
	for i, endpoint := range endpoints.Endpoints {
		endpointStrings[i] = endpoint.String()
	}

	// Initialize the erasure sets instance.
  // 声明变量
	s := &erasureSets{
		sets:               make([]*erasureObjects, setCount),
		erasureDisks:       make([][]StorageAPI, setCount),
		erasureLockers:     make([][]dsync.NetLocker, setCount),
		erasureLockOwner:   globalLocalNodeName,// 全局本地节点的锁名
		endpoints:          endpoints,// 终端存储器集合
		endpointStrings:    endpointStrings,
		setCount:           setCount,
		setDriveCount:      setDriveCount,// 纠察集ErasureSet的驱动器种类
		defaultParityCount: defaultParityCount,//默认的奇偶磁盘数量，用于配置纠察码的奇偶校验
		format:             format,
		setReconnectEvent:  make(chan int),//重连事件
		distributionAlgo:   format.Erasure.DistributionAlgo,
		deploymentID:       uuid.MustParse(format.ID),
		poolIndex:          poolIdx,
	}
  // 空间锁
	mutex := newNSLock(globalIsDistErasure)

	// Number of buffers, max 2GB 缓冲区大小
	n := (2 * humanize.GiByte) / (blockSizeV2 * 2)

	// Initialize byte pool once for all sets, bpool size is set to
	// setCount * setDriveCount with each memory upto blockSizeV2.
  // blockSizeV2 一次性字节池=> 直接写入内存,避免 2 次 io 
  // 新缓冲区
	bp := bpool.NewBytePoolCap(n, blockSizeV2, blockSizeV2*2)

	// Initialize byte pool for all sets, bpool size is set to
	// setCount * setDriveCount with each memory upto blockSizeV1
	//
	// Number of buffers, max 10GiB
  // blockSizeV1 blockSize
	m := (10 * humanize.GiByte) / (blockSizeV1 * 2)
	// 老缓冲区,可能存储生命周期不一的字节 
	bpOld := bpool.NewBytePoolCap(m, blockSizeV1, blockSizeV1*2)
  
	for i := 0; i < setCount; i++ {
    // 声明 当前纠察集ErasureSet 对应的存储驱动器 's的结构
    // 一个纠察集ErasureSet 会有种存储驱动器
		s.erasureDisks[i] = make([]StorageAPI, setDriveCount)
	}
  
  // 纠察集Erasure's总 的每个 host 的远程网络锁 => [host: lock]
	erasureLockers := map[string]dsync.NetLocker{}
  // 遍历终端存储器 endpoints
	for _, endpoint := range endpoints.Endpoints {
		if _, ok := erasureLockers[endpoint.Host]; !ok {
			erasureLockers[endpoint.Host] = newLockAPI(endpoint)
		}
	}
   // setcount:  纠察集ErasureSet的 size
	for i := 0; i < setCount; i++ {
		lockerEpSet := set.NewStringSet()
		for j := 0; j < setDriveCount; j++ {
      // 找到 当前纠察集ErasureSet的 的 当前驱动器 的 endpoint
			endpoint := endpoints.Endpoints[i*setDriveCount+j]
      // 每个端点和每个擦除集只能添加一个锁。
			// Only add lockers only one per endpoint and per erasure set.
			if locker, ok := erasureLockers[endpoint.Host]; ok && !lockerEpSet.Contains(endpoint.Host) {
        // 先获取host 远程net锁[host] 再操作
        // 对单个Erasure的每个 host 获取一个远程net锁
				lockerEpSet.Add(endpoint.Host)
				s.erasureLockers[i] = append(s.erasureLockers[i], locker)
			}
		}
	}
	// 这些锁用来在后面想要远程对其他 host 进行原子操作时上锁
  // 等待一组协程执行完成
	var wg sync.WaitGroup
	for i := 0; i < setCount; i++ {
		wg.Add(1)
    // 对每个纠察集ErasureSet开启一个协程任务
		go func(i int) {
			defer wg.Done()
			// 内部的WaitGroup
			var innerWg sync.WaitGroup
      // 遍历当前纠察集ErasureSet的驱动器
			for j := 0; j < setDriveCount; j++ {
        // 获取当前驱动操作的存储器 disk
				disk := storageDisks[i*setDriveCount+j]
				if disk == nil {
					continue
				}
				innerWg.Add(1)
				go func(disk StorageAPI, i, j int) {
					defer innerWg.Done()
          // 初始化存储器 disk
					diskID, err := disk.GetDiskID()
					if err != nil {
						if !errors.Is(err, errUnformattedDisk) {
							logger.LogIf(ctx, err)
						}
						return
					}
					if diskID == "" {
						return
					}
					m, n, err := findDiskIndexByDiskID(format, diskID)
					if err != nil {
						logger.LogIf(ctx, err)
						return
					}
					if m != i || n != j {
            // 校验报错
						logger.LogIf(ctx, fmt.Errorf("Detected unexpected drive ordering refusing to use the drive - poolID: %s, found drive mounted at (set=%s, drive=%s) expected mount at (set=%s, drive=%s): %s(%s)", humanize.Ordinal(poolIdx+1), humanize.Ordinal(m+1), humanize.Ordinal(n+1), humanize.Ordinal(i+1), humanize.Ordinal(j+1), disk, diskID))
						s.erasureDisks[i][j] = &unrecognizedDisk{storage: disk}
						return
					}
          // 反向设置 disk 持有当前 纠察集存储池的 index,是第几个纠察集,第几个 driver
					disk.SetDiskLoc(s.poolIndex, m, n)
          // 设置endpointStrings,erasureDisks对应的 disk 信息
					s.endpointStrings[m*setDriveCount+n] = disk.String()
					s.erasureDisks[m][n] = disk
				}(disk, i, j)
			}
      // 直到所有并发任务结束
			innerWg.Wait()

			// Initialize erasure objects for a given set.
      // 初始化纠察集对象的数组中 => 当前的 纠察集对象
			s.sets[i] = &erasureObjects{
				setIndex:           i,
				poolIndex:          poolIdx,
				setDriveCount:      setDriveCount,
				defaultParityCount: defaultParityCount,// 校验块数
				getDisks:           s.GetDisks(i),//磁盘disk数组
				getLockers:         s.GetLockers(i),
				getEndpoints:       s.GetEndpoints(i),// 终端存储器数组
				nsMutex:            mutex,// 空间锁
				bp:                 bp,// 新线程池
				bpOld:              bpOld,// 老线程池
			}
		}(i)
	}

	wg.Wait()

	// start cleanup stale uploads go-routine.
  // 清理过期上传的协程
	go s.cleanupStaleUploads(ctx)

	// start cleanup of deleted objects.
  // 清理已删除对象的协程
	go s.cleanupDeletedObjects(ctx)

	// Start the disk monitoring and connect routine.
  // 非测试环境: //启动硬盘监控并连接协程
	if !globalIsTesting {
		go s.monitorAndConnectEndpoints(ctx, defaultMonitorConnectEndpointInterval)
	}

	return s, nil
}


```

### 核心流程

#### 数据上传

数据是如何上传到ObjectLayer

![img](https://img-blog.csdnimg.cn/img_convert/bef6e0def4b288d3ecb605c2bf634de7.png)

> 选择pool => 选择set => 上传 => 数据写入=> 元数据写入

##### 选择pool 

- pool只有一个，直接返回
- pool有多个，这里分两步：
  - 第一步会去查询之前是否存在此数据（bucket+object），如果存在，则返回对应的pool，如果不存在则进入下一步；
  - 第二步根据object 哈希计算落在每个pool的set单元，然后根据每个pool对应set的可用容量进行选择，会高概率选择上可用容量大的pool

```go
// PutObject - 将对象写入最不常用的纠察码池中。
func (z *erasureServerPools) PutObject(ctx context.Context, bucket string, object string, data *PutObjReader, opts ObjectOptions) (ObjectInfo, error) {
	// 验证放置对象的输入参数。
	if err := checkPutObjectArgs(ctx, bucket, object, z); err != nil {
		return ObjectInfo{}, err
	}

	// 对目录对象名称进行编码。
	object = encodeDirObject(object)

	// 如果只有一个纠察集池，则绕过进一步检查并直接写入。
	if z.SinglePool() {
		// 如果 bucket 不是 MinIO 的元数据存储桶
		if !isMinioMetaBucketName(bucket) {
			// 检查磁盘是否有足够的空间来容纳数据。
			avail, err := hasSpaceFor(getDiskInfos(ctx, z.serverPools[0].getHashedSet(object).getDisks()...), data.Size())
			if err != nil {
        // 纠察集写入的投票错误
				logger.LogOnceIf(ctx, err, "erasure-write-quorum")
				return ObjectInfo{}, toObjectErr(errErasureWriteQuorum)
			}
      // // 没有足够的空间
			if !avail {
				return ObjectInfo{}, toObjectErr(errDiskFull)
			}
		}
		// 将对象写入单个池
		return z.serverPools[0].PutObject(ctx, bucket, object, data, opts)
	}

	// 如果不是单一池，且未禁用锁定（opts.NoLock 为 false），则获取一个命名空间锁。
	if !opts.NoLock {
    // 对 bucket 设置一个空间锁
		ns := z.NewNSLock(bucket, object)
    // 获取锁
		lkctx, err := ns.GetLock(ctx, globalOperationTimeout)
		if err != nil {
			return ObjectInfo{}, err
		}
		ctx = lkctx.Context()
		defer ns.Unlock(lkctx)
    // 切换opts锁状态
		opts.NoLock = true
	}

	// 获取应该写入对象的池的索引，选择具有最少已使用磁盘的池，以确保均衡写入。
  // 是否存在此数据（bucket+object）: 高概率选择上可用容量大的pool
	idx, err := z.getPoolIdxNoLock(ctx, bucket, object, data.Size())
	if err != nil {
		return ObjectInfo{}, err
	}
  // idx: 已存在对象的 pool 的 idx 或者是分配到的 pool 的 idx
	// 对象=> 覆盖 或 插入
	return z.serverPools[idx].PutObject(ctx, bucket, object, data, opts)
}

```

##### 选择纠察集 erasure  set

其实在选择pool的时候已经计算过一次对应object会落在那个set中，这里会有两种哈希算法：

- crcHash，计算对象名对应的crc值 % set大小
- sipHash，计算对象名、deploymentID哈希得到 % set大小，当前版本默认为该算法

```go
// PutObject - writes an object to hashedSet based on the object name.
func (s *erasureSets) PutObject(ctx context.Context, bucket string, object string, data *PutObjReader, opts ObjectOptions) (objInfo ObjectInfo, err error) {
	set := s.getHashedSet(object)
	return set.PutObject(ctx, bucket, object, data, opts)
}
```

##### 上传数据

确定数据块、校验块个数及写入Quorum

![img](https://img-blog.csdnimg.cn/img_convert/3e630791f981a2da556174ee503e597d.png)

1. 根据用户配置的`x-amz-storage-class` 值确定校验块个数parityDrives 

   - RRS，集群初始化时如果有设置`MINIO_STORAGE_CLASS_RRS`则返回对应的校验块数，否则为2

   - 其他情况，如果设置了`MINIO_STORAGE_CLASS_STANDARD`则返回对应的校验块数，否则返回默认值

   - | Erasure Set Size | Default Parity (EC:N) |
     | :--------------- | :-------------------- |
     | 5 or fewer       | EC:2                  |
     | 6-7              | EC:3                  |
     | 8 or more        | EC:4                  |

2. `parityDrives+=统计set中掉线或者不存在的磁盘数`，如果`parityDrives`大于set磁盘数的一半，则设置校验块个数为set的磁盘半数，也就是说校验块的个数是不定的

3. `dataDrives:=set中drives个数-partyDrives`

4. writeQuorum := dataDrives，如果数据块与校验块的个数相等，则writeQuorum++


**写入流程**

1. 重排set中磁盘，根据对象的key进行crc32哈希得到分布关系 

   ![img](https://img-blog.csdnimg.cn/img_convert/9d13b00bd8729d4dda3093d2d65de67d.png)

2. 根据对象大小确定ec计算的buffer大小，最大为1M，即一个blockSize大小

3. ec构建数据块与校验块，即上面提到的buffer大小

![img](https://img-blog.csdnimg.cn/img_convert/302e94217449d5fc46d0e54494735951.png)

- `BlockSize`：表示纠删码计算的数据块大小，可以简单理解有1M的用户数据则会根据纠删码规则计算得到数据块+校验块
- `ShardSize`：纠删码块的实际shard大小，比如blockSize=1M，数据块个数dataBlocks为5，那么单个shard大小为209716字节（blockSize/dataBlocks向上取整），是指ec的每个数据小块大小
- `ShardFileSize`：最终纠删码数据shard大小，比如blockSize=1M，数据块个数dataBlocks为5，用户上传一个5M的对象，那么这里会将其分五次进行纠删码计算，最终得到的单个shard的实际文件大小为5*shardSize

1. 写入数据到对应节点，根据shardFileSize的大小会有不同的策略

   - 小文件：以上图为例，假定对象大小为10KB，blockSize=1M，data block与parity block个数均为5，则shardFileSize=2048Bytes，满足小文件的条件，小文件的数据会存在元数据中，后面会详细介绍：

     - 桶为开启多版本且shardFileSize小于128K;
     - 或者shardFileSize大小小于16K。

   - 大文件：如图所示，假定object大小为1.5MB，blockSize=1M，data block与parity block个数均为5，磁盘中对应文件则会分成两个block，每满1M数据会进行一次ec计算并写入数据，最后一个block大小为0.5MB，shardFileSize为209716+104858=314574（详细计算方法见附录 `shardFileSize计算`）。

   - 数据在写入时会有数据bit位保护机制，可以有效检查出磁盘静默或者比特位衰减等问题，保证读取到的数据一定是正确的，比特位保护有两种策略：

     - streaming-bitrot，这种模式每个block会计算一个哈希值并写入到对应的数据文件中；

     - whole-bitrot，这种模式下是针对driver中的一个文件进行计算的，比如上面3小结中的图所示，针对block1+block6计算一个哈希值并将其写入到元数据中。可以看到第二种方式的保护粒度要粗一些，当前默认采用了第一种策略。 

       ![img](https://img-blog.csdnimg.cn/img_convert/a9da0a80b053afb8f92a91b4fca6b60a.png)

   - 对于小于128K的文件走普通IO；大文件则是采用directIO，这里根据文件大小确定写入buffer，64M以上的数据，buffer为4M；其他大文件为2M，如果是4K对齐的数据，则会走drectIO，否则普通io（数据均会调用fdatasync落盘，`cmd/xl-storage.go`中的`CreteFile`方法）

```go
// PutObject - writes an object to hashedSet based on the object name.
func (s *erasureSets) PutObject(ctx context.Context, bucket string, object string, data *PutObjReader, opts ObjectOptions) (objInfo ObjectInfo, err error) {
	set := s.getHashedSet(object)
	return set.PutObject(ctx, bucket, object, data, opts)
}
// Returns always a same erasure coded set for a given input.
func (s *erasureSets) getHashedSet(input string) (set *erasureObjects) {
	return s.sets[s.getHashedSetIndex(input)]
}

// putObject wrapper for erasureObjects PutObject
// object 对象名
// PutObjReader 对象的读入流
// objInfo:返回的对象的信息
func (er erasureObjects) putObject(ctx context.Context, bucket string, object string, r *PutObjReader, opts ObjectOptions) (objInfo ObjectInfo, err error) {
  // 审计功能:  把操作系统输出到日志中
	auditObjectErasureSet(ctx, object, &er)
	// 读入数据
	data := r.Reader
   // 如果提供了条件检查函数（opts.CheckPrecondFn），则执行条件检查。
	if opts.CheckPrecondFn != nil {
		if !opts.NoLock {
      // 没锁就获取一个该 object 的空间锁
			ns := er.NewNSLock(bucket, object)
			lkctx, err := ns.GetLock(ctx, globalOperationTimeout)
			if err != nil {
				return ObjectInfo{}, err
			}
			ctx = lkctx.Context()
			defer ns.Unlock(lkctx)
			opts.NoLock = true
		}
		// 获取对象的信息，以便执行条件检查。
    // ObjectInfo[bucket,name,..., DataBlocks, ParityBlocks]
		obj, err := er.getObjectInfo(ctx, bucket, object, opts)
		if err == nil && opts.CheckPrecondFn(obj) {
			return objInfo, PreConditionFailed{}
		}
		if err != nil && !isErrVersionNotFound(err) && !isErrObjectNotFound(err) && !isErrReadQuorum(err) {
			return objInfo, err
		}
	}

	// Validate input data size and it can never be less than -1.
  // 验证输入数据大小，不能小于 -1
	if data.Size() < -1 {
		logger.LogIf(ctx, errInvalidArgument, logger.Application)
		return ObjectInfo{}, toObjectErr(errInvalidArgument)
	}
	// 复制用户定义的元数据。
	userDefined := cloneMSS(opts.UserDefined)
	// 获取 erasure set 的存储磁盘。
	storageDisks := er.getDisks()
	// 计算 校验块 的数量
	parityDrives := len(storageDisks) / 2
  // 用户没有设置最大的校验块数
  if !opts.MaxParity {
    // 根据存储类AmzStorageClass  获取奇偶校验和数据磁盘数量parityDrives
    parityDrives = globalStorageClass.GetParityForSC(userDefined[xhttp.AmzStorageClass])
    if parityDrives < 0 {
      // 有问题设置为默认值
      parityDrives = er.defaultParityCount
    }

    // 如果有离线磁盘，增加此对象的parityDrives
    parityOrig := parityDrives

    // 创建 原子变量 
    atomicParityDrives := uatomic.NewInt64(0)
    // 计算离线 drivers
    atomicOfflineDrives := uatomic.NewInt64(0)

    atomicParityDrives.Store(int64(parityDrives))

    var wg sync.WaitGroup
    for _, disk := range storageDisks {
      // 遍历 存储disk
      if disk == nil {
        // 为空
        // 增加ParityDrives和离线磁盘数量
        atomicParityDrives.Inc()
        atomicOfflineDrives.Inc()
        continue
      }
      if !disk.IsOnline() {
      	// disk 掉线 不可用
        atomicParityDrives.Inc()
        atomicOfflineDrives.Inc()
        continue
      }
      wg.Add(1)
      go func(disk StorageAPI) {
        defer wg.Done()
        di, err := disk.DiskInfo(ctx, false)
        if err != nil || di.ID == "" {
          //检查是否有别的问题
          atomicOfflineDrives.Inc()
          atomicParityDrives.Inc()
        }
      }(disk)
    }
    wg.Wait()

    // 如果离线磁盘数量超过磁盘总数的一半，无法满足写入要求，返回 errErasureWriteQuorum 错误
    if int(atomicOfflineDrives.Load()) >= (len(storageDisks)+1)/2 {
      return ObjectInfo{}, toObjectErr(errErasureWriteQuorum, bucket, object)
    }

    // 根据计算结果更新纠删码数量
    parityDrives = int(atomicParityDrives.Load())
    if parityDrives >= len(storageDisks)/2 {
      // 超过了存储 disk 的一般,只最多只能用存储 disk 的一半
      parityDrives = len(storageDisks) / 2
    }

    // 如果更新了纠删码数量，将此信息记录在用户定义的元数据中
    if parityOrig != parityDrives {
      userDefined[minIOErasureUpgraded] = strconv.Itoa(parityOrig) + "->" + strconv.Itoa(parityDrives)
    }
  }
  // 计算数据块 => 总的存储块 - 用于校验的块
	dataDrives := len(storageDisks) - parityDrives

	// we now know the number of blocks this object needs for data and parity.
	// writeQuorum is dataBlocks + 1
	writeQuorum := dataDrives
	if dataDrives == parityDrives {
    // 必须保证 data 块多于校验块一块
		writeQuorum++
	}

	// Initialize parts metadata
  // 初始化 块元数据 数组
	partsMetadata := make([]FileInfo, len(storageDisks))
	// 文件块 信息 
	fi := newFileInfo(pathJoin(bucket, object), dataDrives, parityDrives)
	fi.VersionID = opts.VersionID
	if opts.Versioned && fi.VersionID == "" {
		fi.VersionID = mustGetUUID()
	}
	// 数据 dir 编号
	fi.DataDir = mustGetUUID()
  // 校验和
	fi.Checksum = opts.WantChecksum.AppendTo(nil, nil)
	if opts.EncryptFn != nil {
		fi.Checksum = opts.EncryptFn("object-checksum", fi.Checksum)
	}
  // 唯一 id
	uniqueID := mustGetUUID()
	tempObj := uniqueID

	// Initialize erasure metadata.
  //  partsMetadata数组 都 指向同一个 FileInfo
	for index := range partsMetadata {
		partsMetadata[index] = fi
	}

  // Order disks according to erasure distribution
  // 根据erasure分布对可用磁盘进行排序
  var onlineDisks []StorageAPI
  // 重排set中磁盘，根据对象的key进行crc32哈希得到分布关系
  onlineDisks, partsMetadata = shuffleDisksAndPartsMetadata(storageDisks, partsMetadata, fi)

  // 创建纠删码编解码器
  erasure, err := NewErasure(ctx, fi.Erasure.DataBlocks, fi.Erasure.ParityBlocks, fi.Erasure.BlockSize)
  if err != nil {
    return ObjectInfo{}, toObjectErr(err, bucket, object)
  }

  // Fetch buffer for I/O, returns from the pool if not allocates a new one and returns.
  // 获取用于 I/O 操作的缓冲区，如果没有则从池中获取，如果没有则分配一个新的缓冲区
  // 根据对象大小确定ec计算的buffer大小，最大为1M，即一个blockSize大小
  var buffer []byte
  switch size := data.Size(); {
  case size == 0:
    buffer = make([]byte, 1) // 至少分配一个字节以达到 EOF
  case size >= fi.Erasure.BlockSize || size == -1:
    // 大于 fi.Erasure.BlockSize
    // 从er.bp.Get()线程池获取缓冲区
    buffer = er.bp.Get()
    defer er.bp.Put(buffer)
  case size < fi.Erasure.BlockSize:
    // No need to allocate fully blockSizeV1 buffer if the incoming data is smaller.
    // 如果传入数据较小，无需分配完整的 blockSizeV1 缓冲区
    // 直接分配一个 byte 数组
    buffer = make([]byte, size, 2*size+int64(fi.Erasure.ParityBlocks+fi.Erasure.DataBlocks-1))
  }

  if len(buffer) > int(fi.Erasure.BlockSize) {
    // 截断多余 buffer 部分
    buffer = buffer[:fi.Erasure.BlockSize]
  }
  // 设置块名字
  partName := "part.1"
  //  fi.DataDir: 存储对象统一的uuid,作为文件夹名
  // tempErasureObj: 临时纠察集对象
  tempErasureObj := pathJoin(uniqueID, fi.DataDir, partName)
	// 清楚 临时的minioMetaTmpBucket
  defer er.deleteAll(context.Background(), minioMetaTmpBucket, tempObj)

  // 计算分片文件大小
  shardFileSize := erasure.ShardFileSize(data.Size())

  // 创建写入器切片
  writers := make([]io.Writer, len(onlineDisks))

  // 创建内联缓冲区切片
  var inlineBuffers []*bytes.Buffer
  if shardFileSize >= 0 {
    if !opts.Versioned && shardFileSize < smallFileThreshold {
      inlineBuffers = make([]*bytes.Buffer, len(onlineDisks))
    } else if shardFileSize < smallFileThreshold/8 {
      inlineBuffers = make([]*bytes.Buffer, len(onlineDisks))
    }
  } else {
    // 如果数据已经压缩，则使用实际大小来确定
    if sz := erasure.ShardFileSize(data.ActualSize()); sz > 0 {
      if !opts.Versioned && sz < smallFileThreshold {
        inlineBuffers = make([]*bytes.Buffer, len(onlineDisks))
      } else if sz < smallFileThreshold/8 {
        inlineBuffers = make([]*bytes.Buffer, len(onlineDisks))
      }
    }
  }
  for i, disk := range onlineDisks {
    if disk == nil {
      continue
    }

    if !disk.IsOnline() {
      continue
    }

    if len(inlineBuffers) > 0 {
      // 分片大小
      sz := shardFileSize
      if sz < 0 {
        sz = data.ActualSize()
      }
      //设置分片的 buffer
      inlineBuffers[i] = bytes.NewBuffer(make([]byte, 0, sz))
      // 流式写入器: 构造 每个分片的写入器
      // 在内存中缓冲数据，然后将其写入到目标存储介质
      // 直写
      writers[i] = newStreamingBitrotWriterBuffer(inlineBuffers[i], DefaultBitrotAlgorithm, erasure.ShardSize())
      continue
    }
    // 小文件的数据会存在元数据中 => minioMetaTmpBucket
    // 普通 io
    writers[i] = newBitrotWriter(disk, minioMetaTmpBucket, tempErasureObj, shardFileSize, DefaultBitrotAlgorithm, erasure.ShardSize())
  }
  // io 读对象=> toEncode
  toEncode := io.Reader(data)
  if data.Size() > bigFileThreshold {
    // 使用2个缓冲区，以便始终有一个完整的输入缓冲区
    // We use 2 buffers, so we always have a full buffer of input.
    bufA := er.bp.Get()
    bufB := er.bp.Get()
    defer er.bp.Put(bufA)
    defer er.bp.Put(bufB)
    ra, err := readahead.NewReaderBuffer(data, [][]byte{bufA[:fi.Erasure.BlockSize], bufB[:fi.Erasure.BlockSize]})
    if err == nil {
      // 读取大的完整对象
      toEncode = ra
      defer ra.Close()
    }
    logger.LogIf(ctx, err)
  }
  // 对读取器的数据toEncode进行编码，对数据进行 擦除集编码 并写入 写入器writers。
  n, erasureErr := erasure.Encode(ctx, toEncode, writers, buffer, writeQuorum)
  // 关闭writers写入器
  closeBitrotWriters(writers)
  if erasureErr != nil {
    return ObjectInfo{}, toObjectErr(erasureErr, minioMetaTmpBucket, tempErasureObj)
  }

  // 如果读取的字节数小于请求头中指定的字节数，则返回 IncompleteBody 错误
  if n < data.Size() {
    return ObjectInfo{}, IncompleteBody{Bucket: bucket, Object: object}
  }

  var compIndex []byte
  if opts.IndexCB != nil {
    compIndex = opts.IndexCB()
  }
  if !opts.NoLock {
    lk := er.NewNSLock(bucket, object)
    lkctx, err := lk.GetLock(ctx, globalOperationTimeout)
    if err != nil {
      return ObjectInfo{}, err
    }
    ctx = lkctx.Context()
    defer lk.Unlock(lkctx)
  }

  modTime := opts.MTime
  if opts.MTime.IsZero() {
    modTime = UTCNow()
  }

  for i, w := range writers {
    if w == nil {
      onlineDisks[i] = nil
      continue
    }
    if len(inlineBuffers) > 0 && inlineBuffers[i] != nil {
      partsMetadata[i].Data = inlineBuffers[i].Bytes()
    } else {
      partsMetadata[i].Data = nil
    }
    // 无需在分片上添加校验和，对象已包含校验和
    partsMetadata[i].AddObjectPart(1, "", n, data.ActualSize(), modTime, compIndex, nil)
    partsMetadata[i].Versioned = opts.Versioned || opts.VersionSuspended
  }

  userDefined["etag"] = r.MD5CurrentHexString()
  kind, _ := crypto.IsEncrypted(userDefined)
  if opts.PreserveETag != "" {
    if !opts.ReplicationRequest {
      userDefined["etag"] = opts.PreserveETag
    } else if kind != crypto.S3 {
      // 如果是复制请求并且指定了 SSE-S3，则不保留传入的 ETag
      userDefined["etag"] = opts.PreserveETag
    }
  }

  // 如果未指定 content-type，则尝试从文件扩展名猜测内容类型
  if userDefined["content-type"] == "" {
    userDefined["content-type"] = mimedb.TypeByExtension(path.Ext(object))
  }

  // 填充所有必要的元数据，更新每个磁盘上的 `xl.meta` 内容
  for index := range partsMetadata {
    partsMetadata[index].Metadata = userDefined
    partsMetadata[index].Size = n
    partsMetadata[index].ModTime = modTime
  }

  if len(inlineBuffers) > 0 {
    // 当数据内联时设置额外的标头
    for index := range partsMetadata {
      partsMetadata[index].SetInlineData()
    }
  }

  // 将成功写入的临时对象重命名为最终位置
  onlineDisks, versionsDisparity, err := renameData(ctx, onlineDisks, minioMetaTmpBucket, tempObj, partsMetadata, bucket, object, writeQuorum)
  if err != nil {
    if errors.Is(err, errFileNotFound) {
      return ObjectInfo{}, toObjectErr(errErasureWriteQuorum, bucket, object)
    }
    logger.LogOnceIf(ctx, err, "erasure-object-rename-"+bucket+"-"+object)
    return ObjectInfo{}, toObjectErr(err, bucket, object)
  }

  for i := 0; i < len(onlineDisks); i++ {
    if onlineDisks[i] != nil && onlineDisks[i].IsOnline() {
      // 所有磁盘中的对象信息相同，因此我们可以从在线磁盘中选择第一个元数据
      fi = partsMetadata[i]
      break
    }
  }

  // 对于速度测试对象，不尝试修复它们
  if !opts.Speedtest {
    // 无论初始还是在上传过程中，如果磁盘离线，都将其添加到 MRF 列表
    for i := 0; i < len(onlineDisks); i++ {
      if onlineDisks[i] != nil && onlineDisks[i].IsOnline() {
        continue
      }

      er.addPartial(bucket, object, fi.VersionID)
      break
    }

    if versionsDisparity {
      // 如果版本不一致，则将对象添加到 MRF 列表
      globalMRFState.addPartialOp(partialOperation{
        bucket:      bucket,
        object:      object,
        queued:      time.Now(),
        allVersions: true,
        setIndex:    er.setIndex,
        poolIndex:   er.poolIndex,
      })
    }
  }

  // 设置对象信息，标记为最新版本
  fi.ReplicationState = opts.PutReplicationState()
  fi.IsLatest = true

  return fi.ToObjectInfo(bucket, object, opts.Versioned || opts.VersionSuspended), nil

```

##### 元数据写入

元数据主要包含以下内容（详细定义见附录`对象元数据信息`）

- Volume：桶名
- Name：文件名
- VersionID：版本号
- Erasure：对象的ec信息，包括ec算法，数据块个数，校验块个数，block大小、数据分布状态、以及校验值（whole-bitrot方式的校验值存在这里）
- DataDir：对象存储目录，UUID
- Data：用于存储小对象数据
- Parts：分片信息，包含分片号，etag，大小以及实际大小信息，根据分片号排序
- Metadata：用户定义的元数据，如果是小文件则会添加一条`x-minio-internal-inline-data: true`元数据
- Size：数据存储大小，会大于等于真实数据大小
- ModTime：数据更新时间

```go
{
    "volume":"lemon",
    "name":"temp.2M",
    "data_dir":"8366601f-8d64-40e8-90ac-121864c79a45",
    "mod_time":"2021-08-12T01:46:45.320343158Z",
    "size":2097152,
    "metadata":{
        "content-type":"application/octet-stream",
        "etag":"b2d1236c286a3c0704224fe4105eca49"
    },
    "parts":[
        {
            "number":1,
            "size":2097152,
            "actualSize":2097152
        }
    ],
    "erasure":{
        "algorithm":"reedsolomon",
        "data":2,
        "parity":2,
        "blockSize":1048576,
        "index":4,
        "distribution":[
            4,
            1,
            2,
            3
        ],
        "checksum":[
            {
                "PartNumber":1,
                "Algorithm":3,
                "Hash":""
            }
        ]
    },
    ...
}
 
```

**数据在机器的组织结构**

```shell
.
├── GitKrakenSetup.exe #文件名
│   ├── 449e2259-fb0d-48db-97ed-0d71416c33a3 #datadir，存放数据，分片上传的话会有多个part
│   │   ├── part.1
│   │   ├── part.2
│   │   ├── part.3
│   │   ├── part.4
│   │   ├── part.5
│   │   ├── part.6
│   │   ├── part.7
│   │   └── part.8
│   └── xl.meta #存放对象的元数据信息
├── java_error_in_GOLAND_28748.log #可以看到这个文件没有datadir，因为其为小文件将数据存放到了xl.meta中
│   └── xl.meta
├── temp.1M
│   ├── bc58f35c-d62e-42e8-bd79-8e4a404f61d8
│   │   └── part.1
│   └── xl.meta
├── tmp.8M
│   ├── 1eca8474-2739-4316-9307-12fac3a3ccd9
│   │   └── part.1
│   └── xl.meta
└── worker.conf
    └── xl.meta
 
```



#### 数据下载



![img](https://img-blog.csdnimg.cn/img_convert/7541d9421df274bdc65a158b7faa6efd.png)

**选择pool**

- 单pool直接请求对应pool
- 多个pool
  - 向所有pool发起对象查询请求，并对结果根据文件修改时间降序排列，如果时间相同则pool索引小的在前
  - 遍历结果，获取正常对象所在的pool信息（对应pool获取对象信息没有失败）

**选择set**

与上传对象类似，对对象名进行哈希得到具体存储的set

**读元信息**

- 向所有节点发起元数据读取请求，如果失败节点超过一半，则返回读失败
- 根据元数据信息确定对象读取readQuorum（datablocks大小，即数据块个数）
- 根据第一步返回的错误信息判断元数据是否满足quorum机制，如果不满足则会判断是否为垃圾数据，针对垃圾数据执行数据删除操作
- 如果满足quorum，则会校验第一步读到的元数据信息正确性，如果满足quorum机制，则读取元信息成功
- 如果第一步返回的信息中有磁盘掉线信息，则不会发起数据修复流程，直接返回元数据信息
- 判断对象是否有缺失的block，如果有则后台异步发起修复（文件缺失修复）

**读数据**

- 根据数据分布对disk进行排序
- 读取数据并进行ec重建
- 如果读取到期望数据大小但读取过程中发现有数据缺失或损坏，则会后台异步发起修复，不影响数据的正常读取
  - 文件缺失：修复类型为`HealNormalScan`
  - 数据损坏：修复类型为`HealDeepScan`

##### GetObjectNInfo

```go

func (z *erasureServerPools) GetObjectNInfo(ctx context.Context, bucket, object string, rs *HTTPRangeSpec, h http.Header, opts ObjectOptions) (gr *GetObjectReader, err error) {

  // 单个SinglePool
	object = encodeDirObject(object)
	if z.SinglePool() {
		return z.serverPools[0].GetObjectNInfo(ctx, bucket, object, rs, h, opts)
	}
  ....
  // 加读锁
  lock := z.NewNSLock(bucket, object)
	lkctx, err := lock.GetRLock(ctx, globalOperationTimeout)
  // 获取最新的objInfo, zIdx
	objInfo, zIdx, err := z.getLatestObjectInfoWithIdx(ctx, bucket, object, opts)
	... 
    // 错误返回objInfo
    return &GetObjectReader{
      ObjInfo: objInfo,
    }, toObjectErr(errFileNotFound, bucket, object)

  // 去指定的 pool 获取对象 info
	gr, err = z.serverPools[zIdx].GetObjectNInfo(ctx, bucket, object, rs, h, opts)
	return gr, nil
} 

// GetObjectNInfo - returns object info and locked object ReadCloser
func (s *erasureSets) GetObjectNInfo(ctx context.Context, bucket, object string, rs *HTTPRangeSpec, h http.Header, opts ObjectOptions) (gr *GetObjectReader, err error) {
	set := s.getHashedSet(object)
	return set.GetObjectNInfo(ctx, bucket, object, rs, h, opts)
}

// GetObjectNInfo - 返回对象信息和对象的读取器(Closer)。当 err != nil 时，返回的读取器始终为 nil。
func (er erasureObjects) GetObjectNInfo(ctx context.Context, bucket, object string, rs *HTTPRangeSpec, h http.Header, opts ObjectOptions) (gr *GetObjectReader, err error) {
	// 跟踪对象的Erasure Set，用于审计。
	auditObjectErasureSet(ctx, object, &er)

	// 这是一个特殊的调用，首先尝试检查SOS-API调用。
	gr, err = veeamSOSAPIGetObject(ctx, bucket, object, rs, opts)
  ...
	// 获取锁
		lock := er.NewNSLock(bucket, object)
		lkctx, err := lock.GetRLock(ctx, globalOperationTimeout)
  ...
		// 在元数据验证完毕且读取器准备好读取时释放锁。
		//
		// 这是可能的，因为：
		// - 对于内联对象，xl.meta 已经将数据读取到内存中，随后对 xl.meta 的任何变异都对总体读取操作无关紧要。
		// - xl.meta 元数据仍在锁()下验证冗余，但是写入响应不需要串行化并发写入者。
		unlockOnDefer = true
		nsUnlocker = func() { lock.RUnlock(lkctx) }
 ...

	// 获取对象的文件信息、元数据数组和在线磁盘，如果出现错误则返回对象错误。
	fi, metaArr, onlineDisks, err := er.getObjectFileInfo(ctx, bucket, object, opts, true)

	// 如果数据分片不固定，则获取数据分片的磁盘修改时间，并检查是否需要将某些磁盘标记为离线。
	if !fi.DataShardFixed() {
		diskMTime := pickValidDiskTimeWithQuorum(metaArr, fi.Erasure.DataBlocks)
		if !diskMTime.Equal(timeSentinel) && !diskMTime.IsZero() {
			for index := range onlineDisks {
				if onlineDisks[index] == OfflineDisk {
					continue
				}
				if !metaArr[index].IsValid() {
					continue
				}
				if !metaArr[index].AcceptableDelta(diskMTime, shardDiskTimeDelta) {
					// 如果磁盘 mTime 不匹配，则被视为过时。
					// https://github.com/minio/minio/pull/13803
					//
					// 仅当我们能够找到跨冗余中最多出现的磁盘 mtime 大致相同时，才会激活此检查。
					// 允许跳过我们可能认为是错误的那些分片。
					onlineDisks[index] = OfflineDisk
				}
			}
		}
	}

	// 根据文件信息创建对象信息对象。
	objInfo := fi.ToObjectInfo(bucket, object, opts.Versioned || opts.VersionSuspended)

	// 如果对象是删除标记，则根据 `VersionID` 判断是否返回文件未找到或不允许的错误信息。
	if objInfo.DeleteMarker {
		if opts.VersionID == "" {
			return &GetObjectReader{
				ObjInfo: objInfo,
			}, toObjectErr(errFileNotFound, bucket, object)
		}
		// 确保返回对象信息以提供额外信息。
		return &GetObjectReader{
			ObjInfo: objInfo,
		}, toObjectErr(errMethodNotAllowed, bucket, object)
	}

	// 如果对象位于远程存储，则获取过渡的对象读取器。
	if objInfo.IsRemote() {
		gr, err := getTransitionedObjectReader(ctx, bucket, object, rs, h, objInfo, opts)
		if err != nil {
			return nil, err
		}
		unlockOnDefer = false
		return gr.WithCleanupFuncs(nsUnlocker), nil
	}

	// 如果对象大小为0，零字节对象甚至不需要进一步初始化管道等。
	if objInfo.Size == 0 {
		return NewGetObjectReaderFromReader(bytes.NewReader(nil), objInfo, opts)
	}

	// 根据HTTP Range规范和对象信息创建对象读取器。
	fn, off, length, err := NewGetObjectReader(rs, objInfo, opts)
	if err != nil {
		return nil, err
	}

	if unlockOnDefer {
		unlockOnDefer = fi.InlineData()
	}

	// 创建等待管道。
	pr, pw := xioutil.WaitPipe()

	// 启动一个 goroutine 用于读取对象数据。
	go func() {
    // 这里执行读数据
		pw.CloseWithError(er.getObjectWithFileInfo(ctx, bucket, object, off, length, pw, fi, metaArr, onlineDisks))
	}()

	// 用于在出现不完整读取时导致上面的 goroutine 退出的清理函数。
	pipeCloser := func() {
		pr.CloseWithError(nil)
	}

	if !unlockOnDefer {
		return fn(pr, h, pipeCloser, nsUnlocker)
	}

	return fn(pr, h, pipeCloser)
}


func (er erasureObjects) getObjectFileInfo(ctx context.Context, bucket, object string, opts ObjectOptions, readData bool) (fi FileInfo, metaArr []FileInfo, onlineDisks []StorageAPI, err error) {
	disks := er.getDisks()

	var errs []error

	// Read metadata associated with the object from all disks.
	if opts.VersionID != "" {
    // 向所有节点发起元数据读取请求，如果失败节点超过一半，则返回读失败
		metaArr, errs = readAllFileInfo(ctx, disks, bucket, object, opts.VersionID, readData)
	} else {
		metaArr, errs = readAllXL(ctx, disks, bucket, object, readData, opts.InclFreeVersions, true)
	}
  // 根据元数据信息确定对象读取readQuorum（datablocks大小，即数据块个数）
	readQuorum, _, err := objectQuorumFromMeta(ctx, metaArr, errs, er.defaultParityCount)
	if err != nil {
    // 根据错误信息判断元数据是否满足quorum机制，
		if errors.Is(err, errErasureReadQuorum) && !strings.HasPrefix(bucket, minioMetaBucket) {
			_, derr := er.deleteIfDangling(ctx, bucket, object, metaArr, errs, nil, opts)
			if derr != nil {
				err = derr
			}
		}
		return fi, nil, nil, toObjectErr(err, bucket, object)
	}
  
	if reducedErr := reduceReadQuorumErrs(ctx, errs, objectOpIgnoredErrs, readQuorum); reducedErr != nil {
		if errors.Is(reducedErr, errErasureReadQuorum) && !strings.HasPrefix(bucket, minioMetaBucket) {
      // 如果不满足则会判断是否为垃圾数据 针对垃圾数据执行数据删除操作
			_, derr := er.deleteIfDangling(ctx, bucket, object, metaArr, errs, nil, opts)
			if derr != nil {
				reducedErr = derr
			}
		}
		return fi, nil, nil, toObjectErr(reducedErr, bucket, object)
	}

	// List all online disks.
	onlineDisks, modTime, etag := listOnlineDisks(disks, metaArr, errs, readQuorum)

	// Pick latest valid metadata.
  // 如果满足quorum机制，则读取元信息成功
	fi, err = pickValidFileInfo(ctx, metaArr, modTime, etag, readQuorum)
	if err != nil {
		return fi, nil, nil, err
	}

	if !fi.Deleted && len(fi.Erasure.Distribution) != len(onlineDisks) {
		err := fmt.Errorf("unexpected file distribution (%v) from online disks (%v), looks like backend disks have been manually modified refusing to heal %s/%s(%s)",
			fi.Erasure.Distribution, onlineDisks, bucket, object, opts.VersionID)
		logger.LogOnceIf(ctx, err, "get-object-file-info-manually-modified")
		return fi, nil, nil, toObjectErr(err, bucket, object, opts.VersionID)
	}

	filterOnlineDisksInplace(fi, metaArr, onlineDisks)

	// if one of the disk is offline, return right here no need
	// to attempt a heal on the object.
	if countErrs(errs, errDiskNotFound) > 0 {
		return fi, metaArr, onlineDisks, nil
	}
	// 判断对象是否有缺失的block，
	var missingBlocks int
	for i, err := range errs {
		if err != nil && errors.Is(err, errFileNotFound) {
			missingBlocks++
			continue
		}

		// verify metadata is valid, it has similar erasure info
		// as well as common modtime, if modtime is not possible
		// verify if it has common "etag" atleast.
		if metaArr[i].IsValid() && metaArr[i].Erasure.Equal(fi.Erasure) {
			ok := metaArr[i].ModTime.Equal(modTime)
			if modTime.IsZero() || modTime.Equal(timeSentinel) {
				ok = etag != "" && etag == fi.Metadata["etag"]
			}
			if ok {
				continue
			}
		} // in all other cases metadata is corrupt, do not read from it.

		metaArr[i] = FileInfo{}
		onlineDisks[i] = nil
		missingBlocks++
	}

	// if missing metadata can be reconstructed, attempt to reconstruct.
	// additionally do not heal delete markers inline, let them be
	// healed upon regular heal process.
  // 如果可修复 且有missingBlocks则后台异步发起修复（文件缺失修复） 不修复Deleted 
	if !fi.Deleted && missingBlocks > 0 && missingBlocks < readQuorum {
		globalMRFState.addPartialOp(partialOperation{
			bucket:    bucket,
			object:    object,
			versionID: fi.VersionID,
			queued:    time.Now(),
			setIndex:  er.setIndex,
			poolIndex: er.poolIndex,
		})
	}

	return fi, metaArr, onlineDisks, nil
}

```

##### getObjectWithFileInfo

```go
func (er erasureObjects) getObjectWithFileInfo(ctx context.Context, bucket, object string, startOffset int64, length int64, writer io.Writer, fi FileInfo, metaArr []FileInfo, onlineDisks []StorageAPI) error {
	// Reorder online disks based on erasure distribution order.
	// Reorder parts metadata based on erasure distribution order.
  // 根据数据分布对disk进行排序
	onlineDisks, metaArr = shuffleDisksAndPartsMetadataByIndex(onlineDisks, metaArr, fi)

	// For negative length read everything.
	if length < 0 {
		length = fi.Size - startOffset
	}

	// Reply back invalid range if the input offset and length fall out of range.
	if startOffset > fi.Size || startOffset+length > fi.Size {
		logger.LogIf(ctx, InvalidRange{startOffset, length, fi.Size}, logger.Application)
		return InvalidRange{startOffset, length, fi.Size}
	}

	// Get start part index and offset.
  // 获取开始部分索引和偏移量。
	partIndex, partOffset, err := fi.ObjectToPartOffset(ctx, startOffset)
	if err != nil {
		return InvalidRange{startOffset, length, fi.Size}
	}

	// Calculate endOffset according to length
  // 计算 endoffset
	endOffset := startOffset
	if length > 0 {
		endOffset += length - 1
	}

	// Get last part index to read given length.
  // 获取最后一部分索引来读取给定的长度。
	lastPartIndex, _, err := fi.ObjectToPartOffset(ctx, endOffset)
	if err != nil {
		return InvalidRange{startOffset, length, fi.Size}
	}
  // 读取数据并进行ec重建
	var totalBytesRead int64
	erasure, err := NewErasure(ctx, fi.Erasure.DataBlocks, fi.Erasure.ParityBlocks, fi.Erasure.BlockSize)
	if err != nil {
		return toObjectErr(err, bucket, object)
	}

	var healOnce sync.Once

	for ; partIndex <= lastPartIndex; partIndex++ {
		if length == totalBytesRead {
			break
		}

		partNumber := fi.Parts[partIndex].Number

		// Save the current part name and size.
		partSize := fi.Parts[partIndex].Size

		partLength := partSize - partOffset
		// partLength should be adjusted so that we don't write more data than what was requested.
		if partLength > (length - totalBytesRead) {
			partLength = length - totalBytesRead
		}

		tillOffset := erasure.ShardFileOffset(partOffset, partLength, partSize)
		// Get the checksums of the current part.
		readers := make([]io.ReaderAt, len(onlineDisks))
		prefer := make([]bool, len(onlineDisks))
		for index, disk := range onlineDisks {
			if disk == OfflineDisk {
				continue
			}
			if !metaArr[index].IsValid() {
				continue
			}
			if !metaArr[index].Erasure.Equal(fi.Erasure) {
				continue
			}
			checksumInfo := metaArr[index].Erasure.GetChecksumInfo(partNumber)
			partPath := pathJoin(object, metaArr[index].DataDir, fmt.Sprintf("part.%d", partNumber))
      // 直接读取内存构造 reader
			readers[index] = newBitrotReader(disk, metaArr[index].Data, bucket, partPath, tillOffset,
				checksumInfo.Algorithm, checksumInfo.Hash, erasure.ShardSize())

			// Prefer local disks
			prefer[index] = disk.Hostname() == ""
		}

		written, err := erasure.Decode(ctx, writer, readers, partOffset, partLength, partSize, prefer)
		// Note: we should not be defer'ing the following closeBitrotReaders() call as
		// we are inside a for loop i.e if we use defer, we would accumulate a lot of open files by the time
		// we return from this function.
		closeBitrotReaders(readers)
		if err != nil {
			// If we have successfully written all the content that was asked
			// by the client, but we still see an error - this would mean
			// that we have some parts or data blocks missing or corrupted
			// - attempt a heal to successfully heal them for future calls.
			if written == partLength {
				var scan madmin.HealScanMode
				switch {
				case errors.Is(err, errFileNotFound):
          // 文件缺失：修复类型为HealNormalScan
					scan = madmin.HealNormalScan
				case errors.Is(err, errFileCorrupt):
          // 数据损坏：修复类型为HealDeepScan
					scan = madmin.HealDeepScan
				}
				switch scan {
				case madmin.HealNormalScan, madmin.HealDeepScan:
					healOnce.Do(func() {
            // 如果读取到期望数据大小但读取过程中发现有数据缺失或损坏，
            // 则会后台异步发起修复，不影响数据的正常读取
						globalMRFState.addPartialOp(partialOperation{
							bucket:    bucket,
							object:    object,
							versionID: fi.VersionID,
							queued:    time.Now(),
							setIndex:  er.setIndex,
							poolIndex: er.poolIndex,
						})
					})
					// Healing is triggered and we have written
					// successfully the content to client for
					// the specific part, we should `nil` this error
					// and proceed forward, instead of throwing errors.
					err = nil
				}
			}
			if err != nil {
				return toObjectErr(err, bucket, object)
			}
		}
		for i, r := range readers {
			if r == nil {
				onlineDisks[i] = OfflineDisk
			}
		}
		// Track total bytes read from disk and written to the client.
		totalBytesRead += partLength
		// partOffset will be valid only for the first part, hence reset it to 0 for
		// the remaining parts.
		partOffset = 0
	} // End of read all parts loop.
	// Return success.
	return nil
}

```



### 数据巡检

数据巡检主要做以下事情：

- 发现缺失的数据，并尝试将其修复，无法修复的数据（垃圾数据）则会进行清理
- 统计计量信息，如文件数、存储量、桶个数等

巡检时候会在每块磁盘上对所有bucket中的数据进行巡检，这里主要介绍下巡检是如何发现待修复数据并执行修复？

- 扫描对象信息时：如果发现数据缺失或数据损坏则会快速或深度修复（深度扫描会校验数据文件是否完整，而快速扫描则是检查数据是否缺失，巡检时是否发起深度巡检是在服务启动配置中设置的），不是每一次的巡检都会发起修复，通常是每巡检一定轮数会发起一次，这里的修复是立即执行的；
- 跟上一次巡检结果对比：比如上次巡检发现有文件A，这次巡检却没有找到文件A，满足一定条件则会发起修复操作，这里的巡检是先投递修补消息，异步修复。

> 每次巡检都会将巡检的结果缓存在本地，下次巡检与之对比

```go
// cmd/data-scanner.go的runDataScanner方法
// runDataScanner 将启动一个数据扫描器。
// 该函数将阻塞，直到上下文被取消。
// 每个集群只能运行一个扫描器。
func runDataScanner(ctx context.Context, objAPI ObjectLayer) {
	ctx, cancel := globalLeaderLock.GetLock(ctx)
	defer cancel()

	// 加载当前的布隆周期信息
	var cycleInfo currentScannerCycle
  // 读配置
	buf, _ := readConfig(ctx, objAPI, dataUsageBloomNamePath)
	if len(buf) == 8 {
		cycleInfo.next = binary.LittleEndian.Uint64(buf)
	} else if len(buf) > 8 {
		cycleInfo.next = binary.LittleEndian.Uint64(buf[:8])
		buf = buf[8:]
		_, err := cycleInfo.UnmarshalMsg(buf)
		logger.LogIf(ctx, err)
	}
  
	scannerTimer := time.NewTimer(scannerCycle.Load())
	defer scannerTimer.Stop()
	defer globalScannerMetrics.setCycle(nil)

	for {
		select {
		case <-ctx.Done():
			return
		case <-scannerTimer.C:
			// 重置计时器以进行下一个周期。
			// 如果扫描器需要更长时间，我们会立即开始。
			scannerTimer.Reset(scannerCycle.Load())

			stopFn := globalScannerMetrics.log(scannerMetricScanCycle)
			cycleInfo.current = cycleInfo.next
			cycleInfo.started = time.Now()
			globalScannerMetrics.setCycle(&cycleInfo)

			// 读取后台修复信息
      // backgroundHealInfo[
      //		bitrotStartTime,bitrotStartCycle,currentScanMode{
      // 		HealNormalScan,HealDeepScan
      // }]
			bgHealInfo := readBackgroundHealInfo(ctx, objAPI)
			// 获取当前扫描模式
			scanMode := getCycleScanMode(cycleInfo.current, bgHealInfo.BitrotStartCycle, bgHealInfo.BitrotStartTime)
			if bgHealInfo.CurrentScanMode != scanMode {
				// 如果当前扫描模式与新的扫描模式不同，则更新后台修复信息
				newHealInfo := bgHealInfo
				newHealInfo.CurrentScanMode = scanMode
				if scanMode == madmin.HealDeepScan {
					newHealInfo.BitrotStartTime = time.Now().UTC()
					newHealInfo.BitrotStartCycle = cycleInfo.current
				}
        // 更新健康扫描模式
				saveBackgroundHealInfo(ctx, objAPI, newHealInfo)
			}

			// 在启动下一个周期前等待一段时间
			results := make(chan DataUsageInfo, 1)
      // 将存储在gui通道results上发送的所有对象，直到关闭 => saveConfig
      // 每次巡检都会将巡检的结果缓存在本地，下次巡检与之对比
			go storeDataUsageInBackend(ctx, objAPI, results)
      // 走 objAPI 实现 ->server 启动的: erasureServerPools
      // 对桶 对 disk 做扫描,并更新结果,通过 results
			err := objAPI.NSScanner(ctx, results, uint32(cycleInfo.current), scanMode)
			logger.LogIf(ctx, err)
			res := map[string]string{"cycle": strconv.FormatUint(cycleInfo.current, 10)}
			if err != nil {
				res["error"] = err.Error()
			}
			stopFn(res)
			if err == nil {
				// 存储新的周期信息
				cycleInfo.next++
				cycleInfo.current = 0
				cycleInfo.cycleCompleted = append(cycleInfo.cycleCompleted, time.Now())
				if len(cycleInfo.cycleCompleted) > dataUsageUpdateDirCycles {
					cycleInfo.cycleCompleted = cycleInfo.cycleCompleted[len(cycleInfo.cycleCompleted)-dataUsageUpdateDirCycles:]
				}
				globalScannerMetrics.setCycle(&cycleInfo)
				tmp := make([]byte, 8, 8+cycleInfo.Msgsize())
				// 为了向后兼容，存储周期信息
				binary.LittleEndian.PutUint64(tmp, cycleInfo.next)
				tmp, _ = cycleInfo.MarshalMsg(tmp)
				err = saveConfig(ctx, objAPI, dataUsageBloomNamePath, tmp)
				logger.LogIf(ctx, err)
			}
		}
	}
}

```

### api 层

 ![image-20230907215811827](https://img-blog.csdnimg.cn/a20957bf4e8741c38d7bebc44430b841.png)

api层调用层级结构如图，从图中我们可以看出,

1. 无论是 `gateway` 还是 `server` 模式都是通过实现 `ObjectAPI` 这个interface来进行服务
2. 在objectAPIHandlers这一层面，主要是做了一些检查，实际针对内容处理是放在ObjectAPI这个interface的实现层，以putObject为例,做了以下内容
   1. 检查 `http` 头字段
   2. 验证签名
   3. bucket容量检查

```go
main->cmd->server_main -> handler, err := configureServerHandler(globalEndpoints)
-> 注册 router
// configureServer handler returns final handler for the http server.
func configureServerHandler(endpointServerPools EndpointServerPools) (http.Handler, error) {
	// Initialize router. `SkipClean(true)` stops minio/mux from
	// normalizing URL path minio/minio#3256
	router := mux.NewRouter().SkipClean(true).UseEncodedPath()

	// Initialize distributed NS lock.
	if globalIsDistErasure {
		registerDistErasureRouters(router, endpointServerPools)
	}

	// Add Admin router, all APIs are enabled in server mode.
	registerAdminRouter(router, true)

	// Add healthCheck router
	registerHealthCheckRouter(router)

	// Add server metrics router
	registerMetricsRouter(router)

	// Add STS router always.
	registerSTSRouter(router)

	// Add KMS router
	registerKMSRouter(router)

	// Add API router
  // 注册怎么操作 object
	registerAPIRouter(router)

	router.Use(globalMiddlewares...)

	return router, nil
}

// objectAPIHandler implements and provides http handlers for S3 API.
type objectAPIHandlers struct {
	ObjectAPI func() ObjectLayer
	CacheAPI  func() CacheObjectLayer
}

// registerAPIRouter - registers S3 compatible APIs.
// 符合 s3 协议
func registerAPIRouter(router *mux.Router) {
	// Initialize API.
  // 初始化objectAPIHandler
	api := objectAPIHandlers{
    //  挂载实现的ObjectLayer <= 在初始化ObjectLayer后,会 setObjectLayer(o ObjectLayer)
		ObjectAPI: newObjectLayerFn,
		CacheAPI:  newCachedObjectLayerFn,
	}

	// API Router
  // '/' 分割 uri
	apiRouter := router.PathPrefix(SlashSeparator).Subrouter()

	var routers []*mux.Router
	for _, domainName := range globalDomainNames {
		if IsKubernetes() {
			routers = append(routers, apiRouter.MatcherFunc(func(r *http.Request, match *mux.RouteMatch) bool {
				host, _, err := net.SplitHostPort(getHost(r))
				if err != nil {
					host = r.Host
				}
				// Make sure to skip matching minio.<domain>` this is
				// specifically meant for operator/k8s deployment
				// The reason we need to skip this is for a special
				// usecase where we need to make sure that
				// minio.<namespace>.svc.<cluster_domain> is ignored
				// by the bucketDNS style to ensure that path style
				// is available and honored at this domain.
				//
				// All other `<bucket>.<namespace>.svc.<cluster_domain>`
				// makes sure that buckets are routed through this matcher
				// to match for `<bucket>`
				return host != minioReservedBucket+"."+domainName
			}).Host("{bucket:.+}."+domainName).Subrouter())
		} else {
                // 读取 path 里的匹配的数据到 bucket 参数里
                // 注册新的 router,以 domainName
			routers = append(routers, apiRouter.Host("{bucket:.+}."+domainName).Subrouter())
		}
	}
     // 最后的匹配{bucket}的 router
	routers = append(routers, apiRouter.PathPrefix("/{bucket}").Subrouter())

	gz, err := gzhttp.NewWrapper(gzhttp.MinSize(1000), gzhttp.CompressionLevel(gzip.BestSpeed))
	if err != nil {
		// Static params, so this is very unlikely.
		logger.Fatal(err, "Unable to initialize server")
	}

	for _, router := range routers {
		// Register all rejected object APIs
		for _, r := range rejectedObjAPIs {
			t := router.Methods(r.methods...).
				HandlerFunc(collectAPIStats(r.api, httpTraceAll(notImplementedHandler))).
				Queries(r.queries...)
			t.Path(r.path)
		}

		// Object operations
	  .... 
		// GetObject
    // 如果判断出 apistats 是 getobject => 请求走该处理链
    // path匹配到的参数到object变量里
		router.Methods(http.MethodGet).Path("/{object:.+}").HandlerFunc(
			collectAPIStats("getobject", maxClients(gz(httpTraceHdrs(api.GetObjectHandler)))))
		// PutObject
		router.Methods(http.MethodPut).Path("/{object:.+}").HandlerFunc(
			collectAPIStats("putobject", maxClients(gz(httpTraceHdrs(api.PutObjectHandler)))))
    ....
	}

```

#### http中间件

这里的请求 middleware 是采用一层套一层来实现的

```go
func middleware1(f http.HandlerFunc) http.HandlerFunc {
  // 返回一个包装后的中间件HandlerFunc函数
    return func(w http.ResponseWriter, r *http.Request) {
     ... 自己的逻辑
      f(w,r) 
     ...
    }
}
```

##### maxclients

```go
// maxClients throttles the S3 API calls
func maxClients(f http.HandlerFunc) http.HandlerFunc {
    return func(w http.ResponseWriter, r *http.Request) {
        // 记录全局的HTTP统计信息，增加S3请求的计数
        globalHTTPStats.incS3RequestsIncoming()

        // 检查HTTP请求头中是否包含名为globalObjectPerfUserMetadata的字段
        if r.Header.Get(globalObjectPerfUserMetadata) == "" {
            // 如果不包含该字段，检查全局服务是否被冻结
            if val := globalServiceFreeze.Load(); val != nil {
                if unlock, ok := val.(chan struct{}); ok && unlock != nil {
                    // 等待解冻，直到服务解冻为止
                    select {
                    case <-unlock:
                    case <-r.Context().Done():
                        // 如果客户端取消了请求，就不需要一直等待
                        return
                    }
                }
            }
        }

        // 获取用于请求的池和请求的截止时间
        pool, deadline := globalAPIConfig.getRequestsPool()
        if pool == nil {
            // 说明没有最大客户端限制
            // 如果请求池为空，直接调用处理函数并返回
            f.ServeHTTP(w, r)
            return
        }

        // 增加等待队列中的请求数
        globalHTTPStats.addRequestsInQueue(1)

        // 设置请求跟踪信息
        if tc, ok := r.Context().Value(mcontext.ContextTraceKey).(*mcontext.TraceCtxt); ok {
            tc.FuncName = "s3.MaxClients"
        }

        // 创建一个截止时间计时器
        deadlineTimer := time.NewTimer(deadline)
        defer deadlineTimer.Stop()

        select {
          // 等待 pool 中 的 chan <- 令牌桶
        case pool <- struct{}{}:
            // 如果成功从池中获取了令牌，释放令牌并处理请求
            defer func() { <-pool }()
            globalHTTPStats.addRequestsInQueue(-1)
            f.ServeHTTP(w, r)
        case <-deadlineTimer.C:
            // 如果在截止时间内没有获取到令牌，返回HTTP请求超时错误响应
            writeErrorResponse(r.Context(), w,
                errorCodes.ToAPIErr(ErrTooManyRequests),
                r.URL)
            globalHTTPStats.addRequestsInQueue(-1)
            return
        case <-r.Context().Done():
            // 当客户端在获取S3处理程序状态码响应之前断开连接时，将状态码设置为499
            // 这样可以正确记录和跟踪此请求
            w.WriteHeader(499)
            globalHTTPStats.addRequestsInQueue(-1)
            return
        }
    }
}

```

##### api.GetObjectHandler

猜测:肯定会去调真正的存储层实现:  ObjectLayer 或者ObjectCacheLayer

```go
// GetObjectHandler - GET Object
// ----------
// This implementation of the GET operation retrieves object. To use GET,
// you must have READ access to the object.
func (api objectAPIHandlers) GetObjectHandler(w http.ResponseWriter, r *http.Request) {
	ctx := newContext(r, w, "GetObject")

	defer logger.AuditLog(ctx, w, r, mustGetClaimsFromToken(r))
	bucket := vars["bucket"]
	object, err := unescapePath(vars["object"])
	objectAPI := api.ObjectAPI()
	getObjectNInfo := objectAPI.GetObjectNInfo
	if api.CacheAPI() != nil {
		getObjectNInfo = api.CacheAPI().GetObjectNInfo
	}
  // 对读取到的对象返回的响应,做层层封装处理
}
```


## Minio的Java服务中间件

> **通过MinIO整合SpringBoot实现OSS服务器组件搭建和功能实现**
   
 - Minio是Apache License v2.0下发布的对象存储服务器。它与Amazon S3云存储服务兼容。它最适合存储非结构化数据，如照片，视频，日志文件，备份和容器/ VM映像。对象的大小可以从几KB到最大5TB。
   
- Minio服务器足够轻，可以与应用程序堆栈捆绑在一起，类似于NodeJS，Redis和MySQL。
- github 地址: https://github.com/dll02/assemble-platform/tree/main/assemble-platform-minioClient

 
 ```java
 //使用了 client 包
       <dependency>
       <groupId>com.jvm123</groupId>
       <artifactId>minio-spring-boot-starter</artifactId>
       <version>1.2.1</version>
       <exclusions>
         <exclusion>
           <artifactId>guava</artifactId>
           <groupId>com.google.guava</groupId>
         </exclusion>
       </exclusions>
     </dependency>
         
 // 其他代码很简单
 @Slf4j
 @Service
 public class MinioHttpOssService {
 
 
     @Autowired
     MinioFileService fileStoreService;
 
     /**
      * bucket
      * @param bucketName
      * @return
      */
     public ResultResponse create(@RequestParam("bucketName") String bucketName){
         return fileStoreService.createBucket(bucketName)? ResultResponse.success(): ResultResponse.failure("创建oss bucket失败！");
     }
 
 
     /**
      * 存储文件
      * @param file
      * @param bucketName
      * @return
      */
     public ResultResponse upload(@RequestParam("file") MultipartFile file, @RequestParam("bucketName") String bucketName){
         try {
             fileStoreService.save(bucketName,file.getInputStream(),file.getOriginalFilename());
         } catch (IOException e) {
             log.error("upload the file is error",e);
             return ResultResponse.failure("upload the file is error");
         }
         return ResultResponse.success();
     }
 
 
     /**
      * 删除文件
      * @param bucketName
      * @param bucketName
      * @return
      */
     public ResultResponse delete(@RequestParam("bucketName") String bucketName, @RequestParam("fileName") String fileName){
         return fileStoreService.delete(bucketName,fileName)? ResultResponse.success(): ResultResponse.failure("删除oss bucket文件失败！");
     }
 
 
 
     /**
      * 下载文件
      * @param bucketName
      * @param bucketName
      * @return
      */
     public void download(HttpServletResponse httpServletResponse, @RequestParam("bucketName") String bucketName, @RequestParam("fileName") String fileName){
         try (InputStream inputStream = fileStoreService.getStream(bucketName, fileName)){
             httpServletResponse.addHeader("Content-Disposition","attachment;filename="+fileName);
             ServletOutputStream os = httpServletResponse.getOutputStream();
             fileStoreService.writeTo(bucketName, fileName, os);
         } catch (IOException e) {
             log.error("download file is failure!",e);
         }
     }
 
 }
 
 // 走到 client 包
     public String save(String bucket, InputStream is, String destFileName) {
         if (bucket != null && bucket.length() > 0) {
             try {
               // 获取一个MinioClient链接
                 MinioClient minioClient = this.connect();
                 this.checkBucket(minioClient, bucket);
                 minioClient.putObject(bucket, destFileName, is, (Long)null, (Map)null, (ServerSideEncryption)null, (String)null);
                 return destFileName;
             } catch (NoSuchAlgorithmException | IOException | XmlPullParserException | InvalidKeyException | MinioException var5) {
                 LOGGER.error("error: {}", var5.getMessage());
                 return null;
             }
         } else {
             LOGGER.error("Bucket name cannot be blank.");
             return null;
         }
     }
 
 //  minioClient.putObject 最后一定会发起一个 tcp 请求到 minio 服务
 // 封装为符合 s3 协议的请求
 HttpResponse response = execute(Method.PUT, region, bucketName, objectName,
                                 headerMap, queryParamMap,
                                 data, length);
 Response response = this.httpClient.newCall(request).execute();
 ```




## 感言&&参考资料:
minio 的项目是很庞大复杂的,尤其是关于对给类云的协议的兼容解析封装,对Erasure Code擦除码底层存储的实现,都非常的晦涩难懂,功力有限,暂时更新到这里,后面有时间精力和兴趣再更新,有缘再见.
* [浅谈对象之MinIO源码篇](https://blog.csdn.net/u011436273/article/details/123833319)
* [minIO server源码分析](https://zhuanlan.zhihu.com/p/459854239)
* [Erasure-Code-擦除码-1-原理篇](https://blog.openacid.com/storage/ec-1/)

