# NameSrv

## what

NameSrv是什么？

![](image/rocketmq_architecture_1.png)

从架构图看NameSrv用于储存Broker的路由数据，同时也被生产者和消费者端端作服务发现，来获取这条Message该发往哪一个broker，从哪一台broker上拉取(接受推送的)数据。

## Why

为什么需要NameSrv？

首先NameSrv是无状态的，意味着NameSrv可以无限扩容。此外，NameSrv的出现使得Broker不再关注生产者端和消费者端的链接方式，当Broker横向扩容时候，只需要告诉老大哥，老大哥自然会把话带给上游卖家(生产者)和下游买家(消费者)。

## How

NameSrv是怎么保存Broker数据的？

NameSrv是怎么同步Broker数据给生产者和消费者的？

带着疑问，我们来一探究竟。

### NameSrv的启动过程

**核心代码 --- org.apache.rocketmq.namesrv.NamesrvStartup**

1. main方法

```JAVA
    public static NamesrvController main0(String[] args) {
        try {
            // 1. 解析配置文件和命令行输入
            parseCommandlineAndConfigFile(args);
            // 2. 创建NameSrvController并启动服务
            NamesrvController controller = createAndStartNamesrvController();
            return controller;
        } catch (Throwable e) {
            e.printStackTrace();
            System.exit(-1);
        }

        return null;
    }
```

2. **NameSrvController#start**

```java
    public static NamesrvController start(final NamesrvController controller) throws Exception {
        if (null == controller) {
            throw new IllegalArgumentException("NamesrvController is null");
        }
		// 1. 初始化NameSrcController
        boolean initResult = controller.initialize();
        if (!initResult) {
            controller.shutdown();
            System.exit(-3);
        }
	    // 2. 注册优雅停机钩子
        Runtime.getRuntime().addShutdownHook(new ShutdownHookThread(log, (Callable<Void>) () -> {
            controller.shutdown();
            return null;
        }));
        // 3. 开始对外服务
        controller.start();

        return controller;
    }
```

3.**NameSrvController#initialize**

```java
    public boolean initialize() {
        // 加载KV配置
        loadConfig();
        // 初始化网络组件，NettyServer NettyClient，但没有启动
        initiateNetworkComponents();
        // 初始化线程池
        // 1. 默认线程池
        // 2. ClientRequest处理线程池
        initiateThreadExecutors();
        // 将线程池注入到对应的server中
        // 这边的设计比较不错，将网络通信和业务逻辑进行解耦
        // 采用了组合模式
        registerProcessor();
        // 启动3个定时任务
        // 1. 扫描非活跃Broker
        // 2. 打印NameSrv KV信息
        // 3. 打印NameSrv 水位
        startScheduleService();
        // 初始化SSL上下文
        initiateSslContext();
        // 向remoteServer注册一个ZoneRouteRPCHook
        initiateRpcHooks();
        return true;
    }
```

4. **NameSrvController#start()**

```java
    public void start() throws Exception {
        // 1. 启动Netty Server
        this.remotingServer.start();

        // In test scenarios where it is up to OS to pick up an available port, set the listening port back to config
        // 翻译：当NettyServer的端口没有设置时，服务端会设置一个可用的端口并反注入到NettyServerConfig中，让用户感知
        if (0 == nettyServerConfig.getListenPort()) {
            nettyServerConfig.setListenPort(this.remotingServer.localListenPort());
        }

        //  2. 启动NettyClient,NettyClient只会连接到本地NettyServer，TODO why？
        this.remotingClient.updateNameServerAddressList(Collections.singletonList(RemotingUtil.getLocalAddress()
            + ":" + nettyServerConfig.getListenPort()));
        this.remotingClient.start();

        // 3. 启动文件监听服务
        if (this.fileWatchService != null) {
            this.fileWatchService.start();
        }

        // 4. 路由管理服务
        this.routeInfoManager.start();
    }
```

5. **NameSrv开启Contoller模式(类似redis的sentinel)**，可选项目

```java
    public static ControllerManager controllerManagerMain() {
        try {
            if (namesrvConfig.isEnableControllerInNamesrv()) {
                return createAndStartControllerManager();
            }
        } catch (Throwable e) {
            e.printStackTrace();
            System.exit(-1);
        }
        return null;
    }
```

### NameSrv如何保存Broker信息

主要的实现都在**org.apache.rocketmq.namesrv.routeinfo.RouteInfoManager**中