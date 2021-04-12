Eureka的一些**概念**
Register-服务注册
当Eureka Client向Eureka Server注册时，client提供自身的元数据，如IP地址、端口、运行状态指标的Url、主页地址等信息。
**Renew-服务续约**
Eureka Client默认情况下每隔30秒发送一次心跳来进行服务续约。如果Eureka Server在90秒内没有收到Eureka Client的心跳，Eureka Server会将Eureka Client实例从注册列表中剔除。
**Fetch Registries-获取服务注册列表信息**
Eureka Client从Eureka Server获取服务注册表信息，并缓存在本地，可根据注册表信息查找服务，从而进行远程调用。该注册表信息每30秒更新一次。如果由于某种原因导致服务注册列表信息不能及时匹配，Eureka Client会重新获取整个注册表信息。
**Cancel-服务下线**
Eureka Client在程序关闭时可以向Eureka Server发送下线信息。发送请求后，该客户端的实例信息将从Eureka Server的服务注册列表中删除。该下线请求不会自动完成，需要在程序关闭时调用以下代码：

**Eviction-服务剔除**
默认情况下，当Eureka Client连续90秒没有向Eureka Server发送服务续约（即心跳）时，Eureka Server会将该服务实例从服务注册列表删除。
**Eureka的自我保护模式**
当一个新的Eureka Server出现时，会尝试从相邻的Peer节点获取所有服务实例的注册信息。如果从相邻的Peer节点获取信息时出现了故障，Eureka Server会尝试其它的Peer节点。如果Eureka Server能够成功获取所有的服务实例信息，则根据配置信息设置服务续约的阈值。在任何时间，如果Eureka Server接收到的服务续约低于该配置的百分比（默认为15分钟内低于85%），则服务器开启自我保护模式，即不在剔除注册列表的信息。这样做的好处是如果是Eureka Server自身的网络问题而导致Eureka Client无法续约，这样在注册列表中不会剔除注册信息，这样Eureka Client还可以被其他服务消费。同时这个功能也是有坑的，如果在服务较少时，服务由于意外情况挂掉后很容易阈值就低于85%，由于自我保护导致挂掉的服务在注册列表中还是存在，但其它服务无法消费，这会迷惑开发人员排错的方向。


下面直接进行**源码分析**, 首先看**客户端**的代码.
````java
public class EurekaBootStrap implements ServletContextListener {
    // ... 省略部分代码
    @Override
    public void contextInitialized(ServletContextEvent event) {
        try {
            initEurekaEnvironment();
            initEurekaServerContext();

            ServletContext sc = event.getServletContext();
            sc.setAttribute(EurekaServerContext.class.getName(), serverContext);
        } catch (Throwable e) {
            logger.error("Cannot bootstrap eureka server :", e);
            throw new RuntimeException("Cannot bootstrap eureka server :", e);
        }
    }
}
````

````java
   protected void initEurekaServerContext() throws Exception {
        EurekaServerConfig eurekaServerConfig = new DefaultEurekaServerConfig();

        // For backward compatibility
        JsonXStream.getInstance().registerConverter(new V1AwareInstanceInfoConverter(), XStream.PRIORITY_VERY_HIGH);
        XmlXStream.getInstance().registerConverter(new V1AwareInstanceInfoConverter(), XStream.PRIORITY_VERY_HIGH);

        logger.info("Initializing the eureka client...");
        logger.info(eurekaServerConfig.getJsonCodecName());
        ServerCodecs serverCodecs = new DefaultServerCodecs(eurekaServerConfig);

        ApplicationInfoManager applicationInfoManager = null;

        if (eurekaClient == null) {
            EurekaInstanceConfig instanceConfig = isCloud(ConfigurationManager.getDeploymentContext())
                    ? new CloudInstanceConfig()
                    : new MyDataCenterInstanceConfig();
            
            applicationInfoManager = new ApplicationInfoManager(
                    instanceConfig, new EurekaConfigBasedInstanceInfoProvider(instanceConfig).get());
            
            EurekaClientConfig eurekaClientConfig = new DefaultEurekaClientConfig();
            // 初始化一个DiscoveryClient
            eurekaClient = new DiscoveryClient(applicationInfoManager, eurekaClientConfig);
        } else {
            applicationInfoManager = eurekaClient.getApplicationInfoManager();
        }
        // 构造了PeerAwareInstanceRegistry对象，用来感知eureka server集群的服务实例注册表   
        PeerAwareInstanceRegistry registry;
        if (isAws(applicationInfoManager.getInfo())) {
            registry = new AwsInstanceRegistry(
                    eurekaServerConfig,
                    eurekaClient.getEurekaClientConfig(),
                    serverCodecs,
                    eurekaClient
            );
            awsBinder = new AwsBinderDelegate(eurekaServerConfig, eurekaClient.getEurekaClientConfig(), registry, applicationInfoManager);
            awsBinder.start();
        } else {
            registry = new PeerAwareInstanceRegistryImpl(
                    eurekaServerConfig,
                    eurekaClient.getEurekaClientConfig(),
                    serverCodecs,
                    eurekaClient
            );
        }
        // 构造了PeerEurekaNodes对象，用于处理复制所有集群更新操作(注册、更新、取消、过期和状态更改) 
        PeerEurekaNodes peerEurekaNodes = getPeerEurekaNodes(
                registry,
                eurekaServerConfig,
                eurekaClient.getEurekaClientConfig(),
                serverCodecs,
                applicationInfoManager
        );

        serverContext = new DefaultEurekaServerContext(
                eurekaServerConfig,
                serverCodecs,
                registry,
                peerEurekaNodes,
                applicationInfoManager
        );

        EurekaServerContextHolder.initialize(serverContext);

        serverContext.initialize();
        logger.info("Initialized server context");

        // Copy registry from neighboring eureka node
        // 从相邻的一个eureka server节点拷贝注册表的信息，如果拷贝失败，就找下一个
        int registryCount = registry.syncUp();
        registry.openForTraffic(applicationInfoManager, registryCount);

        // Register all monitoring statistics.
        // 监控机制，注册所有监测数据
        EurekaMonitors.registerAllStats();
    }

````
DiscoveryClient类初始化相关功能如下
（1）读取EurekaClientConfig，包括TransportConfig
（2）保存EurekaInstanceConfig和InstanceInfo
（3）处理了是否要注册以及抓取注册表，如果不要的话，释放一些资源
（4）支持调度的线程池
（5）支持心跳的线程池
（6）支持缓存刷新的线程池
（7）EurekaTransport，支持底层的eureka client跟eureka server进行网络通信的组件，对网络通信组件进行了一些初始化的操作
（8）抓取注册表
````java
  @Inject
    DiscoveryClient(ApplicationInfoManager applicationInfoManager, EurekaClientConfig config, AbstractDiscoveryClientOptionalArgs args,
                    Provider<BackupRegistry> backupRegistryProvider, EndpointRandomizer endpointRandomizer) {
        if (args != null) {
            this.healthCheckHandlerProvider = args.healthCheckHandlerProvider;
            this.healthCheckCallbackProvider = args.healthCheckCallbackProvider;
            this.eventListeners.addAll(args.getEventListeners());
            this.preRegistrationHandler = args.preRegistrationHandler;
        } else {
            this.healthCheckCallbackProvider = null;
            this.healthCheckHandlerProvider = null;
            this.preRegistrationHandler = null;
        }
        
        this.applicationInfoManager = applicationInfoManager;
        InstanceInfo myInfo = applicationInfoManager.getInfo();

        clientConfig = config;
        staticClientConfig = clientConfig;
        transportConfig = config.getTransportConfig();
        instanceInfo = myInfo;
        if (myInfo != null) {
            appPathIdentifier = instanceInfo.getAppName() + "/" + instanceInfo.getId();
        } else {
            logger.warn("Setting instanceInfo to a passed in null value");
        }

        this.backupRegistryProvider = backupRegistryProvider;
        this.endpointRandomizer = endpointRandomizer;
        this.urlRandomizer = new EndpointUtils.InstanceInfoBasedUrlRandomizer(instanceInfo);
        localRegionApps.set(new Applications());

        fetchRegistryGeneration = new AtomicLong(0);

        remoteRegionsToFetch = new AtomicReference<String>(clientConfig.fetchRegistryForRemoteRegions());
        remoteRegionsRef = new AtomicReference<>(remoteRegionsToFetch.get() == null ? null : remoteRegionsToFetch.get().split(","));

        if (config.shouldFetchRegistry()) {
            this.registryStalenessMonitor = new ThresholdLevelsMetric(this, METRIC_REGISTRY_PREFIX + "lastUpdateSec_", new long[]{15L, 30L, 60L, 120L, 240L, 480L});
        } else {
            this.registryStalenessMonitor = ThresholdLevelsMetric.NO_OP_METRIC;
        }

        if (config.shouldRegisterWithEureka()) {
            this.heartbeatStalenessMonitor = new ThresholdLevelsMetric(this, METRIC_REGISTRATION_PREFIX + "lastHeartbeatSec_", new long[]{15L, 30L, 60L, 120L, 240L, 480L});
        } else {
            this.heartbeatStalenessMonitor = ThresholdLevelsMetric.NO_OP_METRIC;
        }

        logger.info("Initializing Eureka in region {}", clientConfig.getRegion());

        if (!config.shouldRegisterWithEureka() && !config.shouldFetchRegistry()) {
            logger.info("Client configured to neither register nor query for data.");
            scheduler = null;
            heartbeatExecutor = null;
            cacheRefreshExecutor = null;
            eurekaTransport = null;
            instanceRegionChecker = new InstanceRegionChecker(new PropertyBasedAzToRegionMapper(config), clientConfig.getRegion());

            // This is a bit of hack to allow for existing code using DiscoveryManager.getInstance()
            // to work with DI'd DiscoveryClient
            DiscoveryManager.getInstance().setDiscoveryClient(this);
            DiscoveryManager.getInstance().setEurekaClientConfig(config);

            initTimestampMs = System.currentTimeMillis();
            logger.info("Discovery Client initialized at timestamp {} with initial instances count: {}",
                    initTimestampMs, this.getApplications().size());

            return;  // no need to setup up an network tasks and we are done
        }
        // 心跳检测线程池初始化、本地服务刷新线程池初始化
        try {
            // default size of 2 - 1 each for heartbeat and cacheRefresh
            scheduler = Executors.newScheduledThreadPool(2,
                    new ThreadFactoryBuilder()
                            .setNameFormat("DiscoveryClient-%d")
                            .setDaemon(true)
                            .build());

            heartbeatExecutor = new ThreadPoolExecutor(
                    1, clientConfig.getHeartbeatExecutorThreadPoolSize(), 0, TimeUnit.SECONDS,
                    new SynchronousQueue<Runnable>(),
                    new ThreadFactoryBuilder()
                            .setNameFormat("DiscoveryClient-HeartbeatExecutor-%d")
                            .setDaemon(true)
                            .build()
            );  // use direct handoff

            cacheRefreshExecutor = new ThreadPoolExecutor(
                    1, clientConfig.getCacheRefreshExecutorThreadPoolSize(), 0, TimeUnit.SECONDS,
                    new SynchronousQueue<Runnable>(),
                    new ThreadFactoryBuilder()
                            .setNameFormat("DiscoveryClient-CacheRefreshExecutor-%d")
                            .setDaemon(true)
                            .build()
            );  // use direct handoff

            eurekaTransport = new EurekaTransport();
            scheduleServerEndpointTask(eurekaTransport, args);

            AzToRegionMapper azToRegionMapper;
            if (clientConfig.shouldUseDnsForFetchingServiceUrls()) {
                azToRegionMapper = new DNSBasedAzToRegionMapper(clientConfig);
            } else {
                azToRegionMapper = new PropertyBasedAzToRegionMapper(clientConfig);
            }
            if (null != remoteRegionsToFetch.get()) {
                azToRegionMapper.setRegionsToFetch(remoteRegionsToFetch.get().split(","));
            }
            instanceRegionChecker.getAzToRegionMapper = new InstanceRegionChecker(azToRegionMapper, clientConfig.getRegion());
        } catch (Throwable e) {
            throw new RuntimeException("Failed to initialize DiscoveryClient!", e);
        }

        if (clientConfig.shouldFetchRegistry() && !fetchRegistry(false)) {
            fetchRegistryFromBackup();
        }

        // call and execute the pre registration handler before all background tasks (inc registration) is started
        if (this.preRegistrationHandler != null) {
            this.preRegistrationHandler.beforeRegistration();
        }

        if (clientConfig.shouldRegisterWithEureka() && clientConfig.shouldEnforceRegistrationAtInit()) {
            try {
                // 进行服务注册
                if (!register() ) {
                    throw new IllegalStateException("Registration error at startup. Invalid server response.");
                }
            } catch (Throwable th) {
                logger.error("Registration error at startup: {}", th.getMessage());
                throw new IllegalStateException(th);
            }
        }

        // finally, init the schedule tasks (e.g. cluster resolvers, heartbeat, instanceInfo replicator, fetch
        // 心跳检测、 服务注册、 本地注册信息刷新
        initScheduledTasks();

        try {
            Monitors.registerObject(this);
        } catch (Throwable e) {
            logger.warn("Cannot register timers", e);
        }

        // This is a bit of hack to allow for existing code using DiscoveryManager.getInstance()
        // to work with DI'd DiscoveryClient
        DiscoveryManager.getInstance().setDiscoveryClient(this);
        DiscoveryManager.getInstance().setEurekaClientConfig(config);

        initTimestampMs = System.currentTimeMillis();
        logger.info("Discovery Client initialized at timestamp {} with initial instances count: {}",
                initTimestampMs, this.getApplications().size());
    }
````

接下来重点关注 initScheduledTasks()
````java
    private void initScheduledTasks() {
        // 抓取注册信息到本地
        if (clientConfig.shouldFetchRegistry()) {
            // registry cache refresh timer
            int registryFetchIntervalSeconds = clientConfig.getRegistryFetchIntervalSeconds();
            int expBackOffBound = clientConfig.getCacheRefreshExecutorExponentialBackOffBound();
            cacheRefreshTask = new TimedSupervisorTask(
                    "cacheRefresh",
                    scheduler,
                    cacheRefreshExecutor,
                    registryFetchIntervalSeconds,
                    TimeUnit.SECONDS,
                    expBackOffBound,
                    new CacheRefreshThread()
            );
            scheduler.schedule(
                    cacheRefreshTask,
                    registryFetchIntervalSeconds, TimeUnit.SECONDS);
        }
        // 心跳检测
        if (clientConfig.shouldRegisterWithEureka()) {
            int renewalIntervalInSecs = instanceInfo.getLeaseInfo().getRenewalIntervalInSecs();
            int expBackOffBound = clientConfig.getHeartbeatExecutorExponentialBackOffBound();
            logger.info("Starting heartbeat executor: " + "renew interval is: {}", renewalIntervalInSecs);

            // Heartbeat timer
            heartbeatTask = new TimedSupervisorTask(
                    "heartbeat",
                    scheduler,
                    heartbeatExecutor,
                    renewalIntervalInSecs,
                    TimeUnit.SECONDS,
                    expBackOffBound,
                    new HeartbeatThread()
            );
            scheduler.schedule(
                    heartbeatTask,
                    renewalIntervalInSecs, TimeUnit.SECONDS);

            // InstanceInfo replicator
            instanceInfoReplicator = new InstanceInfoReplicator(
                    this,
                    instanceInfo,
                    clientConfig.getInstanceInfoReplicationIntervalSeconds(),
                    2); // burstSize

            statusChangeListener = new ApplicationInfoManager.StatusChangeListener() {
                @Override
                public String getId() {
                    return "statusChangeListener";
                }

                @Override
                public void notify(StatusChangeEvent statusChangeEvent) {
                    if (InstanceStatus.DOWN == statusChangeEvent.getStatus() ||
                            InstanceStatus.DOWN == statusChangeEvent.getPreviousStatus()) {
                        // log at warn level if DOWN was involved
                        logger.warn("Saw local status change event {}", statusChangeEvent);
                    } else {
                        logger.info("Saw local status change event {}", statusChangeEvent);
                    }
                    instanceInfoReplicator.onDemandUpdate();
                }
            };

            if (clientConfig.shouldOnDemandUpdateStatusChange()) {
                applicationInfoManager.registerStatusChangeListener(statusChangeListener);
            }
            // 注册服务到注册中心
            instanceInfoReplicator.start(clientConfig.getInitialInstanceInfoReplicationIntervalSeconds());
        } else {
            logger.info("Not registering with Eureka server per configuration");
        }
    }

````

抓取注册的服务信息到本地, 追踪代码new CacheRefreshThread()
````java
    class CacheRefreshThread implements Runnable {
        public void run() {
            refreshRegistry();
        }
    }
````

refreshRegistry方法
````java
    @VisibleForTesting
    void refreshRegistry() {
        try {
            boolean isFetchingRemoteRegionRegistries = isFetchingRemoteRegionRegistries();

            boolean remoteRegionsModified = false;
            // This makes sure that a dynamic change to remote regions to fetch is honored.
            String latestRemoteRegions = clientConfig.fetchRegistryForRemoteRegions();
            if (null != latestRemoteRegions) {
                String currentRemoteRegions = remoteRegionsToFetch.get();
                if (!latestRemoteRegions.equals(currentRemoteRegions)) {
                    // Both remoteRegionsToFetch and AzToRegionMapper.regionsToFetch need to be in sync
                    synchronized (instanceRegionChecker.getAzToRegionMapper()) {
                        if (remoteRegionsToFetch.compareAndSet(currentRemoteRegions, latestRemoteRegions)) {
                            String[] remoteRegions = latestRemoteRegions.split(",");
                            remoteRegionsRef.set(remoteRegions);
                            instanceRegionChecker.getAzToRegionMapper().setRegionsToFetch(remoteRegions);
                            remoteRegionsModified = true;
                        } else {
                            logger.info("Remote regions to fetch modified concurrently," +
                                    " ignoring change from {} to {}", currentRemoteRegions, latestRemoteRegions);
                        }
                    }
                } else {
                    // Just refresh mapping to reflect any DNS/Property change
                    instanceRegionChecker.getAzToRegionMapper().refreshMapping();
                }
            }
            // 调用获取注册的方法
            boolean success = fetchRegistry(remoteRegionsModified);
            if (success) {
                registrySize = localRegionApps.get().size();
                lastSuccessfulRegistryFetchTimestamp = System.currentTimeMillis();
            }

            if (logger.isDebugEnabled()) {
                StringBuilder allAppsHashCodes = new StringBuilder();
                allAppsHashCodes.append("Local region apps hashcode: ");
                allAppsHashCodes.append(localRegionApps.get().getAppsHashCode());
                allAppsHashCodes.append(", is fetching remote regions? ");
                allAppsHashCodes.append(isFetchingRemoteRegionRegistries);
                for (Map.Entry<String, Applications> entry : remoteRegionVsApps.entrySet()) {
                    allAppsHashCodes.append(", Remote region: ");
                    allAppsHashCodes.append(entry.getKey());
                    allAppsHashCodes.append(" , apps hashcode: ");
                    allAppsHashCodes.append(entry.getValue().getAppsHashCode());
                }
                logger.debug("Completed cache refresh task for discovery. All Apps hash code is {} ",
                        allAppsHashCodes);
            }
        } catch (Throwable e) {
            logger.error("Cannot fetch registry from server", e);
        }
    }
````


````java
    private boolean fetchRegistry(boolean forceFullRegistryFetch) {
        Stopwatch tracer = FETCH_REGISTRY_TIMER.start();

        try {
            // If the delta is disabled or if it is the first time, get all
            // applications
            Applications applications = getApplications();

            if (clientConfig.shouldDisableDelta()
                    || (!Strings.isNullOrEmpty(clientConfig.getRegistryRefreshSingleVipAddress()))
                    || forceFullRegistryFetch
                    || (applications == null)
                    || (applications.getRegisteredApplications().size() == 0)
                    || (applications.getVersion() == -1)) //Client application does not have latest library supporting delta
            {
                logger.info("Disable delta property : {}", clientConfig.shouldDisableDelta());
                logger.info("Single vip registry refresh property : {}", clientConfig.getRegistryRefreshSingleVipAddress());
                logger.info("Force full registry fetch : {}", forceFullRegistryFetch);
                logger.info("Application is null : {}", (applications == null));
                logger.info("Registered Applications size is zero : {}",
                        (applications.getRegisteredApplications().size() == 0));
                logger.info("Application version is -1: {}", (applications.getVersion() == -1));
                // 全量获取注册信息
                getAndStoreFullRegistry();
            } else {
                // 增量获取注册信息
                getAndUpdateDelta(applications);
            }
            applications.setAppsHashCode(applications.getReconcileHashCode());
            logTotalInstances();
        } catch (Throwable e) {
            logger.error(PREFIX + "{} - was unable to refresh its cache! status = {}", appPathIdentifier, e.getMessage(), e);
            return false;
        } finally {
            if (tracer != null) {
                tracer.stop();
            }
        }

        // Notify about cache refresh before updating the instance remote status
        onCacheRefreshed();

        // Update remote status based on refreshed data held in the cache
        updateInstanceRemoteStatus();

        // registry was fetched successfully, so return true
        return true;
    }
````

全量获取注册信息getAndStoreFullRegistry()代码:
````java
    /**
     * 发送get请求/eureka/apps
     */
    private void getAndStoreFullRegistry() throws Throwable {
        long currentUpdateGeneration = fetchRegistryGeneration.get();

        logger.info("Getting all instance registry info from the eureka server");

        Applications apps = null;
        EurekaHttpResponse<Applications> httpResponse = clientConfig.getRegistryRefreshSingleVipAddress() == null
                ? eurekaTransport.queryClient.getApplications(remoteRegionsRef.get())
                : eurekaTransport.queryClient.getVip(clientConfig.getRegistryRefreshSingleVipAddress(), remoteRegionsRef.get());
        if (httpResponse.getStatusCode() == Status.OK.getStatusCode()) {
            apps = httpResponse.getEntity();
        }
        logger.info("The response status is {}", httpResponse.getStatusCode());

        if (apps == null) {
            logger.error("The application is null for some reason. Not storing this information");
        } else if (fetchRegistryGeneration.compareAndSet(currentUpdateGeneration, currentUpdateGeneration + 1)) {
            localRegionApps.set(this.filterAndShuffle(apps));
            logger.debug("Got full registry with apps hashcode {}", apps.getAppsHashCode());
        } else {
            logger.warn("Not updating applications as another thread is updating it already");
        }
    }
````

增量获取注册信息getAndUpdateDelta()代码:
````java
    /**
     * 送get请求/eureka/apps/delta
     */
    private void getAndUpdateDelta(Applications applications) throws Throwable {
        long currentUpdateGeneration = fetchRegistryGeneration.get();

        Applications delta = null;
        EurekaHttpResponse<Applications> httpResponse = eurekaTransport.queryClient.getDelta(remoteRegionsRef.get());
        if (httpResponse.getStatusCode() == Status.OK.getStatusCode()) {
            delta = httpResponse.getEntity();
        }

        if (delta == null) {
            logger.warn("The server does not allow the delta revision to be applied because it is not safe. "
                    + "Hence got the full registry.");
            getAndStoreFullRegistry();
        } else if (fetchRegistryGeneration.compareAndSet(currentUpdateGeneration, currentUpdateGeneration + 1)) {
            logger.debug("Got delta update with apps hashcode {}", delta.getAppsHashCode());
            String reconcileHashCode = "";
            if (fetchRegistryUpdateLock.tryLock()) {
                try {
                    updateDelta(delta);
                    reconcileHashCode = getReconcileHashCode(applications);
                } finally {
                    fetchRegistryUpdateLock.unlock();
                }
            } else {
                logger.warn("Cannot acquire update lock, aborting getAndUpdateDelta");
            }
            // There is a diff in number of instances for some reason
            if (!reconcileHashCode.equals(delta.getAppsHashCode()) || clientConfig.shouldLogDeltaDiff()) {
                reconcileAndLogDifference(delta, reconcileHashCode);  // this makes a remoteCall
            }
        } else {
            logger.warn("Not updating application delta as another thread is updating it already");
            logger.debug("Ignoring delta update with apps hashcode {}, as another thread is updating it already", delta.getAppsHashCode());
        }
    }
````

接下来看心跳检测代码
````java
    private class HeartbeatThread implements Runnable {

        public void run() {
            if (renew()) {
                lastSuccessfulHeartbeatTimestamp = System.currentTimeMillis();
            }
        }
    }
````

````java
    /**
     * 发送put请求到/eureka/apps/{appName}/{id}
     */
    boolean renew() {
        EurekaHttpResponse<InstanceInfo> httpResponse;
        try {
            httpResponse = eurekaTransport.registrationClient.sendHeartBeat(instanceInfo.getAppName(), instanceInfo.getId(), instanceInfo, null);
            logger.debug(PREFIX + "{} - Heartbeat status: {}", appPathIdentifier, httpResponse.getStatusCode());
            if (httpResponse.getStatusCode() == Status.NOT_FOUND.getStatusCode()) {
                REREGISTER_COUNTER.increment();
                logger.info(PREFIX + "{} - Re-registering apps/{}", appPathIdentifier, instanceInfo.getAppName());
                long timestamp = instanceInfo.setIsDirtyWithTime();
                boolean success = register();
                if (success) {
                    instanceInfo.unsetIsDirty(timestamp);
                }
                return success;
            }
            return httpResponse.getStatusCode() == Status.OK.getStatusCode();
        } catch (Throwable e) {
            logger.error(PREFIX + "{} - was unable to send heartbeat!", appPathIdentifier, e);
            return false;
        }
    }
````

接下来看服务注册相关代码
````java
class InstanceInfoReplicator implements Runnable {
    // 这里只看run方法                                                                                              
    public void run() {
        try {
            discoveryClient.refreshInstanceInfo();

            Long dirtyTimestamp = instanceInfo.isDirtyWithTime();
            if (dirtyTimestamp != null) {
                discoveryClient.register();
                instanceInfo.unsetIsDirty(dirtyTimestamp);
            }
        } catch (Throwable t) {
            logger.warn("There was a problem with the instance info replicator", t);
        } finally {
            Future next = scheduler.schedule(this, replicationIntervalSeconds, TimeUnit.SECONDS);
            scheduledPeriodicRef.set(next);
        }
    }

}

````

register相关方法
````java
    /**
     * 发送post请求到/eureka/apps/{appName}
     */
    boolean register() throws Throwable {
        logger.info(PREFIX + "{}: registering service...", appPathIdentifier);
        EurekaHttpResponse<Void> httpResponse;
        try {
            httpResponse = eurekaTransport.registrationClient.register(instanceInfo);
        } catch (Exception e) {
            logger.warn(PREFIX + "{} - registration failed {}", appPathIdentifier, e.getMessage(), e);
            throw e;
        }
        if (logger.isInfoEnabled()) {
            logger.info(PREFIX + "{} - registration status: {}", appPathIdentifier, httpResponse.getStatusCode());
        }
        return httpResponse.getStatusCode() == Status.NO_CONTENT.getStatusCode();
    }
````

spring.factories是core
````java
org.springframework.boot.autoconfigure.EnableAutoConfiguration=\
org.springframework.cloud.netflix.eureka.config.EurekaClientConfigServerAutoConfiguration,\
org.springframework.cloud.netflix.eureka.config.EurekaDiscoveryClientConfigServiceAutoConfiguration,\
org.springframework.cloud.netflix.eureka.EurekaClientAutoConfiguration,\
org.springframework.cloud.netflix.ribbon.eureka.RibbonEurekaAutoConfiguration,\
org.springframework.cloud.netflix.eureka.EurekaDiscoveryClientConfiguration,\
org.springframework.cloud.netflix.eureka.reactive.EurekaReactiveDiscoveryClientConfiguration,\
org.springframework.cloud.netflix.eureka.loadbalancer.LoadBalancerEurekaAutoConfiguration
````
EurekaClientConfigServerAutoConfiguration类中缺少ConfigServerProperties所以不会生效, 
EurekaDiscoveryClientConfigServiceAutoConfiguration中缺少spring.cloud.config.discovery.enabled属性, 所以也不会生效.
接下来我们看EurekaClientAutoConfiguration配置类, 也是客户端启动的核心配置类

````java
@Configuration(proxyBeanMethods = false)
@EnableConfigurationProperties
@ConditionalOnClass(EurekaClientConfig.class)
@Import(DiscoveryClientOptionalArgsConfiguration.class)
@ConditionalOnProperty(value = "eureka.client.enabled", matchIfMissing = true)
@ConditionalOnDiscoveryEnabled
@AutoConfigureBefore({ NoopDiscoveryClientAutoConfiguration.class,
        CommonsClientAutoConfiguration.class, ServiceRegistryAutoConfiguration.class })
@AutoConfigureAfter(name = {
        "org.springframework.cloud.autoconfigure.RefreshAutoConfiguration",
        "org.springframework.cloud.netflix.eureka.EurekaDiscoveryClientConfiguration",
        "org.springframework.cloud.client.serviceregistry.AutoServiceRegistrationAutoConfiguration" })
public class EurekaClientAutoConfiguration {
    // 其他bean
    @Configuration(proxyBeanMethods = false)
    @ConditionalOnMissingRefreshScope
    protected static class EurekaClientConfiguration {

        @Autowired
        private ApplicationContext context;

        @Autowired
        private AbstractDiscoveryClientOptionalArgs<?> optionalArgs;

        @Bean(destroyMethod = "shutdown")
        @ConditionalOnMissingBean(value = EurekaClient.class,
                search = SearchStrategy.CURRENT)
        public EurekaClient eurekaClient(ApplicationInfoManager manager,
                EurekaClientConfig config) {
            return new CloudEurekaClient(manager, config, this.optionalArgs,
                    this.context);
        }
        // 其他bean
    }
}
````

接下来看的EurekaClient的shutdown方法, EurekaClient该bean销毁的会行shutdown()方法,
shutdown()方法会注销注册信息、关闭一些线程池、取消一些任务、关闭监视器等
````java
    @PreDestroy
    @Override
    public synchronized void shutdown() {
        if (isShutdown.compareAndSet(false, true)) {
            logger.info("Shutting down DiscoveryClient ...");

            if (statusChangeListener != null && applicationInfoManager != null) {
                applicationInfoManager.unregisterStatusChangeListener(statusChangeListener.getId());
            }
            // 该方法会关闭心跳检测线程池、本地注册信息缓存刷新线程池、服务注册停止, 
            // 并将心跳检测、缓存刷新取消
            cancelScheduledTasks();

            // If APPINFO was registered
            if (applicationInfoManager != null
                    && clientConfig.shouldRegisterWithEureka()
                    && clientConfig.shouldUnregisterOnShutdown()) {
                applicationInfoManager.setInstanceStatus(InstanceStatus.DOWN);
                unregister();
            }

            if (eurekaTransport != null) {
                eurekaTransport.shutdown();
            }

            heartbeatStalenessMonitor.shutdown();
            registryStalenessMonitor.shutdown();

            Monitors.unregisterObject(this);

            logger.info("Completed shut down of DiscoveryClient");
        }
    }
````

该方法会关闭心跳检测线程池、本地注册信息缓存刷新线程池、服务注册停止, 
并将心跳检测、缓存刷新取消       
````java
    private void cancelScheduledTasks() {
        if (instanceInfoReplicator != null) {
            instanceInfoReplicator.stop();
        }
        if (heartbeatExecutor != null) {
            heartbeatExecutor.shutdownNow();
        }
        if (cacheRefreshExecutor != null) {
            cacheRefreshExecutor.shutdownNow();
        }
        if (scheduler != null) {
            scheduler.shutdownNow();
        }
        if (cacheRefreshTask != null) {
            cacheRefreshTask.cancel();
        }
        if (heartbeatTask != null) {
            heartbeatTask.cancel();
        }
    }
````

删除注册信息
````java
    /**
     * 发送delete请求到/eureka/apps/{appName}/{id};
     */
    void unregister() {
        // It can be null if shouldRegisterWithEureka == false
        if(eurekaTransport != null && eurekaTransport.registrationClient != null) {
            try {
                logger.info("Unregistering ...");
                EurekaHttpResponse<Void> httpResponse = eurekaTransport.registrationClient.cancel(instanceInfo.getAppName(), instanceInfo.getId());
                logger.info(PREFIX + "{} - deregister  status: {}", appPathIdentifier, httpResponse.getStatusCode());
            } catch (Exception e) {
                logger.error(PREFIX + "{} - de-registration failed{}", appPathIdentifier, e.getMessage(), e);
            }
        }
    }
````


下面是eureka服务端的源码分析

首先看EnableEurekaServer注解, 该注解导入了一个EurekaServerMarkerConfiguration配置类
````java
    @Target(ElementType.TYPE)
    @Retention(RetentionPolicy.RUNTIME)
    @Documented
    @Import(EurekaServerMarkerConfiguration.class)
    public @interface EnableEurekaServer {

    }
````

接下来看EurekaServerMarkerConfiguration配置类, 该配置类配置了一个Marker的类型的类
````java
@Configuration(proxyBeanMethods = false)
public class EurekaServerMarkerConfiguration {

    @Bean
    public Marker eurekaServerMarkerBean() {
        return new Marker();
    }

    class Marker {

    }

}
````

Maker类的配置的bean会引出另一个配置类EurekaServerAutoConfiguration, 该类继承了WebMvcConfigurer,

````java
@Configuration(proxyBeanMethods = false)
@Import(EurekaServerInitializerConfiguration.class)
@ConditionalOnBean(EurekaServerMarkerConfiguration.Marker.class)
@EnableConfigurationProperties({ EurekaDashboardProperties.class,
        InstanceRegistryProperties.class })
@PropertySource("classpath:/eureka/server.properties")
public class EurekaServerAutoConfiguration implements WebMvcConfigurer {
    // ...该处省略一部分代码
    /**
     * 这里配置FilterRegistrationBean的过滤器会拦截/eureka/**的相关路径
     */
    @Bean
    public FilterRegistrationBean<?> jerseyFilterRegistration(
            javax.ws.rs.core.Application eurekaJerseyApp) {
        FilterRegistrationBean<Filter> bean = new FilterRegistrationBean<Filter>();
        bean.setFilter(new ServletContainer(eurekaJerseyApp));
        bean.setOrder(Ordered.LOWEST_PRECEDENCE);
        bean.setUrlPatterns(
                Collections.singletonList(EurekaConstants.DEFAULT_PREFIX + "/*"));

        return bean;
    }

    /**
     * 扫描出所有@Path和@Provider的类放入DefaultResourceConfig
     */
    @Bean
    public javax.ws.rs.core.Application jerseyApplication(Environment environment,
            ResourceLoader resourceLoader) {

        ClassPathScanningCandidateComponentProvider provider = new ClassPathScanningCandidateComponentProvider(
                false, environment);

        // Filter to include only classes that have a particular annotation.
        //
        provider.addIncludeFilter(new AnnotationTypeFilter(Path.class));
        provider.addIncludeFilter(new AnnotationTypeFilter(Provider.class));

        // Find classes in Eureka packages (or subpackages)
        // 
        Set<Class<?>> classes = new HashSet<>();
        for (String basePackage : EUREKA_PACKAGES) {
            Set<BeanDefinition> beans = provider.findCandidateComponents(basePackage);
            for (BeanDefinition bd : beans) {
                Class<?> cls = ClassUtils.resolveClassName(bd.getBeanClassName(),
                        resourceLoader.getClassLoader());
                classes.add(cls);
            }
        }

        // Construct the Jersey ResourceConfig
        Map<String, Object> propsAndFeatures = new HashMap<>();
        propsAndFeatures.put(
                // Skip static content used by the webapp
                ServletContainer.PROPERTY_WEB_PAGE_CONTENT_REGEX,
                EurekaConstants.DEFAULT_PREFIX + "/(fonts|images|css|js)/.*");

        DefaultResourceConfig rc = new DefaultResourceConfig(classes);
        rc.setPropertiesAndFeatures(propsAndFeatures);

        return rc;
    }

}
````

@Import(EurekaServerInitializerConfiguration.class)类中会引入EurekaServerInitializerConfiguration
配置
````java
@Configuration(proxyBeanMethods = false)
public class EurekaServerInitializerConfiguration
        implements ServletContextAware, SmartLifecycle, Ordered {
    // 省略部分代码

    @Override
    public void start() {
        new Thread(() -> {
            try {
                // TODO: is this class even needed now?
                // 上下文初始化
                eurekaServerBootstrap.contextInitialized(
                        EurekaServerInitializerConfiguration.this.servletContext);
                log.info("Started Eureka Server");

                publish(new EurekaRegistryAvailableEvent(getEurekaServerConfig()));
                EurekaServerInitializerConfiguration.this.running = true;
                publish(new EurekaServerStartedEvent(getEurekaServerConfig()));
            }
            catch (Exception ex) {
                // Help!
                log.error("Could not initialize Eureka servlet context", ex);
            }
        }).start();
    }
}
````

contextInitialized()方法
````java
    /**
     * 初始化方法
     */
    public void contextInitialized(ServletContext context) {
        try {
            // 初始化环境
            initEurekaEnvironment();
            // 初始化上下文
            initEurekaServerContext();

            context.setAttribute(EurekaServerContext.class.getName(), this.serverContext);
        }
        catch (Throwable e) {
            log.error("Cannot bootstrap eureka server :", e);
            throw new RuntimeException("Cannot bootstrap eureka server :", e);
        }
    }
````

````java
    protected void initEurekaServerContext() throws Exception {
        // For backward compatibility
        JsonXStream.getInstance().registerConverter(new V1AwareInstanceInfoConverter(),
                XStream.PRIORITY_VERY_HIGH);
        XmlXStream.getInstance().registerConverter(new V1AwareInstanceInfoConverter(),
                XStream.PRIORITY_VERY_HIGH);

        if (isAws(this.applicationInfoManager.getInfo())) {
            this.awsBinder = new AwsBinderDelegate(this.eurekaServerConfig,
                    this.eurekaClientConfig, this.registry, this.applicationInfoManager);
            this.awsBinder.start();
        }

        EurekaServerContextHolder.initialize(this.serverContext);

        log.info("Initialized server context");

        // Copy registry from neighboring eureka node
        // 从相邻的节点复制注册信息
        int registryCount = this.registry.syncUp();
        this.registry.openForTraffic(this.applicationInfoManager, registryCount);

        // Register all monitoring statistics.
        EurekaMonitors.registerAllStats();
    }
````

从相邻的节点服务注册信息
````java
    @Override
    public int syncUp() {
        // Copy entire entry from neighboring DS node
        int count = 0;

        for (int i = 0; ((i < serverConfig.getRegistrySyncRetries()) && (count == 0)); i++) {
            if (i > 0) {
                try {
                    Thread.sleep(serverConfig.getRegistrySyncRetryWaitMs());
                } catch (InterruptedException e) {
                    logger.warn("Interrupted during registry transfer..");
                    break;
                }
            }
            Applications apps = eurekaClient.getApplications();
            // 获取注册信息并注册在本地
            for (Application app : apps.getRegisteredApplications()) {
                for (InstanceInfo instance : app.getInstances()) {
                    try {
                        if (isRegisterable(instance)) {
                            register(instance, instance.getLeaseInfo().getDurationInSecs(), true);
                            count++;
                        }
                    } catch (Throwable t) {
                        logger.error("During DS init copy", t);
                    }
                }
            }
        }
        return count;
    }
````

````java
    public void register(InstanceInfo registrant, int leaseDuration, boolean isReplication) {
        try {
            read.lock();
            Map<String, Lease<InstanceInfo>> gMap = registry.get(registrant.getAppName());
            REGISTER.increment(isReplication);
            if (gMap == null) {
                final ConcurrentHashMap<String, Lease<InstanceInfo>> gNewMap = new ConcurrentHashMap<String, Lease<InstanceInfo>>();
                gMap = registry.putIfAbsent(registrant.getAppName(), gNewMap);
                if (gMap == null) {
                    gMap = gNewMap;
                }
            }
            Lease<InstanceInfo> existingLease = gMap.get(registrant.getId());
            // Retain the last dirty timestamp without overwriting it, if there is already a lease
            if (existingLease != null && (existingLease.getHolder() != null)) {
                Long existingLastDirtyTimestamp = existingLease.getHolder().getLastDirtyTimestamp();
                Long registrationLastDirtyTimestamp = registrant.getLastDirtyTimestamp();
                logger.debug("Existing lease found (existing={}, provided={}", existingLastDirtyTimestamp, registrationLastDirtyTimestamp);

                // this is a > instead of a >= because if the timestamps are equal, we still take the remote transmitted
                // InstanceInfo instead of the server local copy.
                if (existingLastDirtyTimestamp > registrationLastDirtyTimestamp) {
                    logger.warn("There is an existing lease and the existing lease's dirty timestamp {} is greater" +
                            " than the one that is being registered {}", existingLastDirtyTimestamp, registrationLastDirtyTimestamp);
                    logger.warn("Using the existing instanceInfo instead of the new instanceInfo as the registrant");
                    registrant = existingLease.getHolder();
                }
            } else {
                // The lease does not exist and hence it is a new registration
                synchronized (lock) {
                    if (this.expectedNumberOfClientsSendingRenews > 0) {
                        // Since the client wants to register it, increase the number of clients sending renews
                        this.expectedNumberOfClientsSendingRenews = this.expectedNumberOfClientsSendingRenews + 1;
                        updateRenewsPerMinThreshold();
                    }
                }
                logger.debug("No previous lease information found; it is new registration");
            }
            Lease<InstanceInfo> lease = new Lease<InstanceInfo>(registrant, leaseDuration);
            if (existingLease != null) {
                lease.setServiceUpTimestamp(existingLease.getServiceUpTimestamp());
            }
            gMap.put(registrant.getId(), lease);
            recentRegisteredQueue.add(new Pair<Long, String>(
                    System.currentTimeMillis(),
                    registrant.getAppName() + "(" + registrant.getId() + ")"));
            // This is where the initial state transfer of overridden status happens
            if (!InstanceStatus.UNKNOWN.equals(registrant.getOverriddenStatus())) {
                logger.debug("Found overridden status {} for instance {}. Checking to see if needs to be add to the "
                                + "overrides", registrant.getOverriddenStatus(), registrant.getId());
                if (!overriddenInstanceStatusMap.containsKey(registrant.getId())) {
                    logger.info("Not found overridden id {} and hence adding it", registrant.getId());
                    overriddenInstanceStatusMap.put(registrant.getId(), registrant.getOverriddenStatus());
                }
            }
            InstanceStatus overriddenStatusFromMap = overriddenInstanceStatusMap.get(registrant.getId());
            if (overriddenStatusFromMap != null) {
                logger.info("Storing overridden status {} from map", overriddenStatusFromMap);
                registrant.setOverriddenStatus(overriddenStatusFromMap);
            }

            // Set the status based on the overridden status rules
            InstanceStatus overriddenInstanceStatus = getOverriddenInstanceStatus(registrant, existingLease, isReplication);
            registrant.setStatusWithoutDirty(overriddenInstanceStatus);

            // If the lease is registered with UP status, set lease service up timestamp
            if (InstanceStatus.UP.equals(registrant.getStatus())) {
                lease.serviceUp();
            }
            registrant.setActionType(ActionType.ADDED);
            recentlyChangedQueue.add(new RecentlyChangedItem(lease));
            registrant.setLastUpdatedTimestamp();
            invalidateCache(registrant.getAppName(), registrant.getVIPAddress(), registrant.getSecureVipAddress());
            logger.info("Registered instance {}/{} with status {} (replication={})",
                    registrant.getAppName(), registrant.getId(), registrant.getStatus(), isReplication);
        } finally {
            read.unlock();
        }
    }
````

其实注册信息就是存在一个ConcurrentHashMap中
````java
    private final ConcurrentHashMap<String, Map<String, Lease<InstanceInfo>>> registry
            = new ConcurrentHashMap<String, Map<String, Lease<InstanceInfo>>>();
````

openForTraffic剔除
````java
    @Override
    public void openForTraffic(ApplicationInfoManager applicationInfoManager, int count) {
        // Renewals happen every 30 seconds and for a minute it should be a factor of 2.
        this.expectedNumberOfClientsSendingRenews = count;
        updateRenewsPerMinThreshold();
        logger.info("Got {} instances from neighboring DS node", count);
        logger.info("Renew threshold is: {}", numberOfRenewsPerMinThreshold);
        this.startupTime = System.currentTimeMillis();
        if (count > 0) {
            this.peerInstancesTransferEmptyOnStartup = false;
        }
        DataCenterInfo.Name selfName = applicationInfoManager.getInfo().getDataCenterInfo().getName();
        boolean isAws = Name.Amazon == selfName;
        if (isAws && serverConfig.shouldPrimeAwsReplicaConnections()) {
            logger.info("Priming AWS connections for all replicas..");
            primeAwsReplicas(applicationInfoManager);
        }
        logger.info("Changing status to UP");
        applicationInfoManager.setInstanceStatus(InstanceStatus.UP);
        super.postInit();
    }
`````

````java
    protected void postInit() {
        renewsLastMin.start();
        if (evictionTaskRef.get() != null) {
            evictionTaskRef.get().cancel();
        }
        evictionTaskRef.set(new EvictionTask());
        evictionTimer.schedule(evictionTaskRef.get(),
                serverConfig.getEvictionIntervalTimerInMs(),
                serverConfig.getEvictionIntervalTimerInMs());
    }
````

这里是一个内部类
````java
    /* visible for testing */ class EvictionTask extends TimerTask {

        private final AtomicLong lastExecutionNanosRef = new AtomicLong(0l);

        @Override
        public void run() {
            try {
                long compensationTimeMs = getCompensationTimeMs();
                logger.info("Running the evict task with compensationTime {}ms", compensationTimeMs);
                evict(compensationTimeMs);
            } catch (Throwable e) {
                logger.error("Could not run the evict task", e);
            }
        }
    }
````

剔除过期的服务
````java
    public void evict(long additionalLeaseMs) {
        logger.debug("Running the evict task");

        if (!isLeaseExpirationEnabled()) {
            logger.debug("DS: lease expiration is currently disabled.");
            return;
        }

        // We collect first all expired items, to evict them in random order. For large eviction sets,
        // if we do not that, we might wipe out whole apps before self preservation kicks in. By randomizing it,
        // the impact should be evenly distributed across all applications.
        List<Lease<InstanceInfo>> expiredLeases = new ArrayList<>();
        for (Entry<String, Map<String, Lease<InstanceInfo>>> groupEntry : registry.entrySet()) {
            Map<String, Lease<InstanceInfo>> leaseMap = groupEntry.getValue();
            if (leaseMap != null) {
                for (Entry<String, Lease<InstanceInfo>> leaseEntry : leaseMap.entrySet()) {
                    Lease<InstanceInfo> lease = leaseEntry.getValue();
                    if (lease.isExpired(additionalLeaseMs) && lease.getHolder() != null) {
                        expiredLeases.add(lease);
                    }
                }
            }
        }

        // To compensate for GC pauses or drifting local time, we need to use current registry size as a base for
        // triggering self-preservation. Without that we would wipe out full registry.
        int registrySize = (int) getLocalRegistrySize();
        int registrySizeThreshold = (int) (registrySize * serverConfig.getRenewalPercentThreshold());
        int evictionLimit = registrySize - registrySizeThreshold;

        int toEvict = Math.min(expiredLeases.size(), evictionLimit);
        if (toEvict > 0) {
            logger.info("Evicting {} items (expired={}, evictionLimit={})", toEvict, expiredLeases.size(), evictionLimit);

            Random random = new Random(System.currentTimeMillis());
            for (int i = 0; i < toEvict; i++) {
                // Pick a random item (Knuth shuffle algorithm)
                int next = i + random.nextInt(expiredLeases.size() - i);
                Collections.swap(expiredLeases, i, next);
                Lease<InstanceInfo> lease = expiredLeases.get(i);

                String appName = lease.getHolder().getAppName();
                String id = lease.getHolder().getId();
                EXPIRED.increment();
                logger.warn("DS: Registry: expired lease for {}/{}", appName, id);
                internalCancel(appName, id, false);
            }
        }
    }
````

下面我们以注册的逻辑去分析代码
````java
    @POST
    @Consumes({"application/json", "application/xml"})
    public Response addInstance(InstanceInfo info,
                                @HeaderParam(PeerEurekaNode.HEADER_REPLICATION) String isReplication) {
        logger.debug("Registering instance {} (replication={})", info.getId(), isReplication);
        // 信息校验
        if (isBlank(info.getId())) {
            return Response.status(400).entity("Missing instanceId").build();
        } else if (isBlank(info.getHostName())) {
            return Response.status(400).entity("Missing hostname").build();
        } else if (isBlank(info.getAppName())) {
            return Response.status(400).entity("Missing appName").build();
        } else if (!appName.equals(info.getAppName())) {
            return Response.status(400).entity("Mismatched appName, expecting " + appName + " but was " + info.getAppName()).build();
        } else if (info.getDataCenterInfo() == null) {
            return Response.status(400).entity("Missing dataCenterInfo").build();
        } else if (info.getDataCenterInfo().getName() == null) {
            return Response.status(400).entity("Missing dataCenterInfo Name").build();
        }

        // 处理客户端可能使用错误的数据中心信息注册缺少数据的情况
        DataCenterInfo dataCenterInfo = info.getDataCenterInfo();
        if (dataCenterInfo instanceof UniqueIdentifier) {
            String dataCenterInfoId = ((UniqueIdentifier) dataCenterInfo).getId();
            if (isBlank(dataCenterInfoId)) {
                boolean experimental = "true".equalsIgnoreCase(serverConfig.getExperimental("registration.validation.dataCenterInfoId"));
                if (experimental) {
                    String entity = "DataCenterInfo of type " + dataCenterInfo.getClass() + " must contain a valid id";
                    return Response.status(400).entity(entity).build();
                } else if (dataCenterInfo instanceof AmazonInfo) {
                    AmazonInfo amazonInfo = (AmazonInfo) dataCenterInfo;
                    String effectiveId = amazonInfo.get(AmazonInfo.MetaDataKey.instanceId);
                    if (effectiveId == null) {
                        amazonInfo.getMetadata().put(AmazonInfo.MetaDataKey.instanceId.getName(), info.getId());
                    }
                } else {
                    logger.warn("Registering DataCenterInfo of type {} without an appropriate id", dataCenterInfo.getClass());
                }
            }
        }

        //这里是具体的注册逻辑。
        registry.register(info, "true".equals(isReplication));
        return Response.status(204).build();  // 204 to be backwards compatible
    }
````

注册的主要逻辑
````java
    @Override
    public void register(final InstanceInfo info, final boolean isReplication) {
        int leaseDuration = Lease.DEFAULT_DURATION_IN_SECS;
        if (info.getLeaseInfo() != null && info.getLeaseInfo().getDurationInSecs() > 0) {
            leaseDuration = info.getLeaseInfo().getDurationInSecs();
        }
        
        //信息注册
        super.register(info, leaseDuration, isReplication);
        
        //同步信息给其他节点
        replicateToPeers(Action.Register, info.getAppName(), info.getId(), info, null, isReplication);
    }
````

下面再看下同步信息到其他节点
````java
    private void replicateToPeers(Action action, String appName, String id,
                                  InstanceInfo info /* optional */,
                                  InstanceStatus newStatus /* optional */, boolean isReplication) {
        Stopwatch tracer = action.getTimer().start();
        try {
            if (isReplication) {
                numberOfReplicationsLastMin.increment();
            }
            // If it is a replication already, do not replicate again as this will create a poison replication
            if (peerEurekaNodes == Collections.EMPTY_LIST || isReplication) {
                return;
            }

            for (final PeerEurekaNode node : peerEurekaNodes.getPeerEurekaNodes()) {
                // If the url represents this host, do not replicate to yourself.
                if (peerEurekaNodes.isThisMyUrl(node.getServiceUrl())) {
                    continue;
                }
                //同步信息到其他节点，这里node里面保存很多其他节点的信息，主要用于和其他节点交互
                replicateInstanceActionsToPeers(action, appName, id, info, newStatus, node);
            }
        } finally {
            tracer.stop();
        }
    }
````

继续追踪注册实例, replicateInstanceActionsToPeers这个方法我们大致就能猜到其他的cancel
、heartbeat等操作也是通过http请求通知到其他节点
````java
    private void replicateInstanceActionsToPeers(Action action, String appName,
                                                 String id, InstanceInfo info, InstanceStatus newStatus,
                                                 PeerEurekaNode node) {
        try {
            InstanceInfo infoFromRegistry;
            CurrentRequestVersion.set(Version.V2);
            switch (action) {
                case Cancel:
                    node.cancel(appName, id);
                    break;
                case Heartbeat:
                    InstanceStatus overriddenStatus = overriddenInstanceStatusMap.get(id);
                    infoFromRegistry = getInstanceByAppAndId(appName, id, false);
                    node.heartbeat(appName, id, infoFromRegistry, overriddenStatus, false);
                    break;
                case Register:
                    node.register(info);
                    break;
                case StatusUpdate:
                    infoFromRegistry = getInstanceByAppAndId(appName, id, false);
                    node.statusUpdate(appName, id, newStatus, infoFromRegistry);
                    break;
                case DeleteStatusOverride:
                    infoFromRegistry = getInstanceByAppAndId(appName, id, false);
                    node.deleteStatusOverride(appName, id, infoFromRegistry);
                    break;
            }
        } catch (Throwable t) {
            logger.error("Cannot replicate information to {} for action {}", node.getServiceUrl(), action.name(), t);
        } finally {
            CurrentRequestVersion.remove();
        }
    }
````

注册系统同步这里会有一个延时expiryTime
````java
    public void register(final InstanceInfo info) throws Exception {
        long expiryTime = System.currentTimeMillis() + getLeaseRenewalOf(info);
        batchingDispatcher.process(
                taskId("register", info),
                new InstanceReplicationTask(targetHost, Action.Register, info, null, true) {
                    public EurekaHttpResponse<Void> execute() {
                        return replicationClient.register(info);
                    }
                },
                expiryTime
        );
    }
````

最终还是用http的形式发送注册的请求到各个节点
````java
    @Override
    public EurekaHttpResponse<Void> register(InstanceInfo info) {
        String urlPath = "apps/" + info.getAppName();
        ClientResponse response = null;
        try {
            Builder resourceBuilder = jerseyClient.resource(serviceUrl).path(urlPath).getRequestBuilder();
            addExtraHeaders(resourceBuilder);
            response = resourceBuilder
                    .header("Accept-Encoding", "gzip")
                    .type(MediaType.APPLICATION_JSON_TYPE)
                    .accept(MediaType.APPLICATION_JSON)
                    .post(ClientResponse.class, info);
            return anEurekaHttpResponse(response.getStatus()).headers(headersOf(response)).build();
        } finally {
            if (logger.isDebugEnabled()) {
                logger.debug("Jersey HTTP POST {}/{} with instance {}; statusCode={}", serviceUrl, urlPath, info.getId(),
                        response == null ? "N/A" : response.getStatus());
            }
            if (response != null) {
                response.close();
            }
        }
    }
````

相关心跳检测、服务注销的代码, 已经客户端获取服务注册信息这里就不在给出相关源码分析。

参考博客
https://www.cnblogs.com/nijunyang/p/10745730.html

https://www.cnblogs.com/nijunyang/p/10805759.html