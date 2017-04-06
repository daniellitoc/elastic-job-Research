# com.dangdang.ddframe.reg.zookeeper.ZookeeperRegistryCenter

    public ZookeeperRegistryCenter(ZookeeperConfiguration zkConfig)
        设置zkConfig属性等于zkConfig参数
    public void init()
        创建zk客户端
            连接地址设置为zkConfig.serverLists属性
            命名空间设置为zkConfig.namespace属性（根节点）
            设置重试策略的初始间隔时间、重试次数、最大重试时间分别为zkConfig.baseSleepTimeMilliseconds、zkConfig.maxRetries、zkConfig.maxSleepTimeMilliseconds
            如果zkConfig.sessionTimeoutMilliseconds属性不等0，设置作为zk的会话超时时间
            如果zkConfig.connectionTimeoutMilliseconds属性不等0，设置作为zk的连接超时时间
            如果zkConfig.digest不等于null，设置作为zk的权限令牌
        如果zkConfig属性的localPropertiesPath属性不等于null
            创建Properties实例加载localPropertiesPath文件遍历                                         // 只有读取，无保存
                如果key节点不存在
                    创建key持久节点，设置节点值等于value
                否则如果zkConfig.overwrite是否等于true
                    设置key节点值等于value
    public void close()
        遍历caches属性
            调用entry.value.close()方法
        关闭zk客户端
    public void addCacheData(String cachePath)                                                    // 监听cachePath节点和所有子节点，进行节点和数据同步
        设置cache变量等于new TreeCache(client, cachePath)实例
        执行cache.start()
        添加cachePath和cache变量到caches属性中
    public String get(String key)                                                                 // 获取节点值
        遍历caches属性，
            如果key.startWith(entry.key)等于true，设置cache变量等于entry.value
        如果cache变量等于null，获取key节点的节点值并返回
        设置resultIncache变量等于cache.getCurrentData(key)
        如果resultIncache变量不等于null
            如果resultIncache.getData()不等于null，则返回，否则返回null
        否则
            获取key节点的节点值并返回

# com.dangdang.ddframe.job.api.JobScheduler

    public JobScheduler(CoordinatorRegistryCenter registryCenter, JobConfiguration config)
        设置registryCenter属性等于registryCenter参数
        设置config属性等于config参数
        创建new ListenerManager(registryCenter, config)实例，设置给listenerManager属性
        创建new ConfigurationService(registryCenter, config)实例，设置给configService属性
        创建new LeaderElectionService(registryCenter, config)实例，设置给leaderElectionService属性
        创建new ServerService(registryCenter, config)实例，设置给serverService属性
        创建new ShardingService(registryCenter, config)实例，设置给shardingService属性
        创建new ExecutionContextService(registryCenter, config)实例，设置给executionContextService属性
        创建new ExecutionService(registryCenter, config)实例，设置给executionService属性
        创建new FailoverService(registryCenter, config)实例，设置给failoverService属性
        创建new StatisticsService(registryCenter, config)实例，设置给statisticsService属性
        创建new OffsetService(registryCenter, config)实例，设置给offsetService属性
        创建new MonitorService(registryCenter, config)实例，设置给monitorService属性
        设置jobDetail属性等于创建JobBuilder.newJob(conf.getJobClass()).withIdentity(config.getJobName()).build()        // Quartz的JobDetail
    public void init()
        执行registryCenter.addCacheData("/{jobName}")

        执行listenerManager.startAllListeners()
        执行leaderElectionService.leaderElection()
        执行configService.persistJobConfiguration()
        执行serverService.persistServerOnline()
        执行serverService.clearJobStopedStatus()
        执行statisticsService.startProcessCountJob()
        执行shardingService.setReshardingFlag()
        执行monitorService.listen()

        添加configService和configService属性到jobDetail.jobDataMap属性中
        添加shardingService和shardingService属性到jobDetail.jobDataMap属性中
        添加executionContextService和executionContextService属性到jobDetail.jobDataMap属性中
        添加executionService和executionService属性到jobDetail.jobDataMap属性中
        添加failoverService和failoverService属性到jobDetail.jobDataMap属性中
        添加offsetService和offsetService属性到jobDetail.jobDataMap属性中

        StdSchedulerFactory factory = new StdSchedulerFactory();
        Properties properties = new Properties();
        properties.put("org.quartz.threadPool.class", "org.quartz.simpl.SimpleThreadPool");
        properties.put("org.quartz.threadPool.threadCount", "1");
        properties.put("org.quartz.scheduler.instanceName", "{jobName}_Scheduler");
        if (!configService.isMisfire())
            // 任务实际执行时间和任务预期时间间隔小于此值，继续执行任务，并将下一次预期执行时间以当前时间为基准；如果大于此值，不执行此次任务，等待下一次预期
            properties.put("org.quartz.jobStore.misfireThreshold", "1");
        prepareEnvironments(result)
        factory.initialize(properties);
        scheduler = factory.getScheduler();
        scheduler.getListenerManager().addTriggerListener(new JobTriggerListener(executionService, shardingService));
        CronScheduleBuilder cronScheduleBuilder = CronScheduleBuilder.cronSchedule(configService.getCron());
        if (configService.isMisfire())
            cronScheduleBuilder = cronScheduleBuilder.withMisfireHandlingInstructionFireAndProceed();
        else
            cronScheduleBuilder = cronScheduleBuilder.withMisfireHandlingInstructionDoNothing();
        Trigger trigger = TriggerBuilder.newTrigger().withIdentity("{jobName}_Trigger").withSchedule(cronScheduleBuilder).build();
        if (!scheduler.checkExists(jobDetail.getKey())) {
            scheduler.scheduleJob(jobDetail, trigger);
        }
        scheduler.start();
        JobRegistry.getInstance().addJobScheduler("{jobName}", this)
    protected void prepareEnvironments(Properties props)

    public void shutdown()
        monitorService.close()
        statisticsService.stopProcessCountJob()
        scheduler.shutdown()
    public void triggerJob()
        scheduler.triggerJob(jobDetail.getKey())
    public void stopJob()
        JobRegistry.getInstance().getJobInstance("{jobName}").stop()
        scheduler.pauseAll()
    public void resumeCrashedJob()
        serverService.persistServerOnline()
        executionService.clearRunningInfo(shardingService.getLocalHostShardingItems());
        if (!serverService.isJobStopedManually()) {
            JobRegistry.getInstance().getJobInstance("{jobName}").resume();
            scheduler.resumeAll();
        }
    public void resumeManualStopedJob()
        if (!scheduler.isShutdown()) {
            JobRegistry.getInstance().getJobInstance("{jobName}").resume();
            scheduler.resumeAll();
        }
    public void rescheduleJob(String cronExpression)
        CronScheduleBuilder cronScheduleBuilder = CronScheduleBuilder.cronSchedule(cronExpression);
        if (configService.isMisfire())
            cronScheduleBuilder = cronScheduleBuilder.withMisfireHandlingInstructionFireAndProceed();
        else
            cronScheduleBuilder = cronScheduleBuilder.withMisfireHandlingInstructionDoNothing();
        Trigger trigger = TriggerBuilder.newTrigger().withIdentity("{jobName}_Trigger").withSchedule(cronScheduleBuilder).build();
        scheduler.rescheduleJob(TriggerKey.triggerKey("{jobName}_Trigger"), trigger);
    public Date getNextFireTime()               // 如果存在多个触发器，获取计算后距离现在最近的一个，实际不会有多个
        List<? extends Trigger> triggers = triggers = scheduler.getTriggersOfJob(jobDetail.getKey())
        Date result = null;
        for (Trigger each : triggers) {
            Date nextFireTime = each.getNextFireTime();
            if (null == nextFireTime) {
                continue;
            }
            if (null == result) {
                result = nextFireTime;
            } else if (nextFireTime.getTime() < result.getTime()) {
                result = nextFireTime;
            }
        }
        return result;
## com.dangdang.ddframe.job.internal.listener.ListenerManager

    public void startAllListeners()
        执行electionListenerManager.start()
        执行shardingListenerManager.start()
        执行executionListenerManager.start()
        执行failoverListenerManager.start()
        执行jobOperationListenerManager.start()
        执行configurationListenerManager.start()

### com.dangdang.ddframe.job.internal.election.ElectionListenerManager

    public void start()
        当/{jobName}节点本身或子节点发生变化时，执行监听器
            如果节点路径等于/{jobName}/leader/election/host并且事件类型等于NODE_REMOVED并且leaderElectionService.hasLeader()等于false
                执行leaderElectionService.leaderElection()
                执行shardingService.setReshardingFlag()

### com.dangdang.ddframe.job.internal.sharding.ShardingListenerManager

    public void start()
        当/{jobName}节点本身或子节点发生变化时，执行监听器
            如果节点路径等于/{jobName}/config/shardingTotalCount
                执行shardingService.setReshardingFlag()
                执行executionService.setNeedFixExecutionInfoFlag()
        当/{jobName}节点本身或子节点发生变化时，执行监听器
            如果节点路径以/{jobName}/servers开头且以status结尾且事件类型不等于NODE_UPDATED或者节点路径以/{jobName}/servers开头且以disabled结尾
                执行shardingService.setReshardingFlag()

### com.dangdang.ddframe.job.internal.execution.ExecutionListenerManager

    public void start()
        当/{jobName}节点本身或子节点发生变化时，执行监听器
            如果节点路径等于/{jobName}/config/monitorExecution并且事件类型等于NODE_UPDATED并且节点值不等于true
                执行executionService.removeExecutionInfo()

### com.dangdang.ddframe.job.internal.failover.FailoverListenerManager

    public void start()
        当/{jobName}节点本身或子节点发生变化时，执行监听器
            如果节点路径以/{jobName}/execution开头并且以running结尾，获取/{jobName}/execution/和/running中间的分片号，设置给item变量，否则设置item变量等于null
            如果item变量不等于null并且事件类型等于NODE_REMOVED并且executionService.isCompleted(item)等于false并且configService.isFailover()等于true
                执行failoverService.setCrashedFailoverFlag(item)
                如果executionService.hasRunningItems(shardingService.getLocalHostShardingItems())等于false
                    执行failoverService.failoverIfNecessary()
        当/{jobName}节点本身或子节点发生变化时，执行监听器
            如果节点路径以/{jobName}/execution开头并且以failover结尾，获取/{jobName}/execution/和/failover中间的分片号，设置给item变量，否则设置item变量等于null
            如果item变量不等于null并且事件类型等于NODE_REMOVED并且executionService.isCompleted(item)等于false并且configService.isFailover()等于true
                执行failoverService.setCrashedFailoverFlag(item)
                如果executionService.hasRunningItems(shardingService.getLocalHostShardingItems())等于false
                    执行failoverService.failoverIfNecessary()
        当/{jobName}节点本身或子节点发生变化时，执行监听器
            如果节点路径等于/{jobName}/config/failover并且事件类型等于NODE_UPDATED并且节点值不等于true
                执行failoverService.removeFailoverInfo()

### com.dangdang.ddframe.job.internal.server.JobOperationListenerManager

    public void start()
        添加连接状态改变监听器
            如果状态等于LOST
                执行JobRegistry.getInstance().getJobScheduler(jobName).stopJob()
            否则如果状态等于RECONNECTED，
                执行JobRegistry.getInstance().getJobScheduler(jobName).resumeCrashedJob()
        当/{jobName}节点本身或子节点发生变化时，执行监听器
            如果节点路径以/{jobName}/servers/{ip}/stoped开头
                设置jobScheduler变量等于JobRegistry.getInstance().getJobScheduler(jobName)
                如果jobScheduler变量不等于null
                    如果事件类型等于NODE_ADDED
                        执行jobScheduler.stopJob()
                    如果事件类型等于NODE_REMOVED
                        执行jobScheduler.resumeManualStopedJob()

### com.dangdang.ddframe.job.internal.config.ConfigurationListenerManager

    public void start()
        当/{jobName}节点本身或子节点发生变化时，执行监听器
            如果节点路径等于/{jobName}/config/cron并且事件类型等于NODE_UPDATED
                设置cronExpression变量等于节点值
                设置jobScheduler变量等于JobRegistry.getInstance().getJobScheduler(jobName)
                如果jobScheduler变量不等于null
                    执行jobScheduler.rescheduleJob(cronExpression)

## com.dangdang.ddframe.job.internal.election.LeaderElectionService

    public void leaderElection()
        LeaderLatch latch = new LeaderLatch(client, "/{jobName}/leader/election/latch")      // 以/{jobName}/leader/election/latch作为选举节点
        latch.start();                                                                       // 开始选举
        latch.await();                                                                       // 等待，直到当前节点作为选举后的主节点
        如果/{jobName}/leader/election/host节点不存在，创建临时节点，设置节点值等于{ip}
        latcher.close()                        // 关闭后，其他leader会再次选举成功，但此时leader/election/host节点已存在且存储了第一次选举成功leader的IP
    public Boolean isLeader()
        循环
            如果hasLeader()等于true，跳出循环，否则睡眠100毫秒
        如果/{jobName}/leader/election/host节点值等于{ip}，返回true，否则返回false
    public boolean hasLeader()
        如果/{jobName}/leader/election/host节点不存在，返回false，否则返回true

## com.dangdang.ddframe.job.internal.config.ConfigurationService

    public void persistJobConfiguration()
        如果/{jobName}/config/jobClass节点存在，获取节点值，如果和{jobClass}不等，抛异常JobConflictException
        如果/{jobName}/config/jobClass节点不存在或者{overwrite}等于true并且节点值不等于{jobClass}，创建永久节点，设置节点值等于{jobClass}
        如果/{jobName}/config/shardingTotalCount节点不存在或者{overwrite}等于true并且节点值不等于{shardingTotalCount}，创建永久节点，设置节点值等于{shardingTotalCount}
        如果/{jobName}/config/shardingItemParameters节点不存在或者{overwrite}等于true并且节点值不等于{shardingItemParameters}，创建永久节点，设置节点值等于{shardingItemParameters}
        如果/{jobName}/config/jobParameter节点不存在或者{overwrite}等于true并且节点值不等于{jobParameter}，创建永久节点，设置节点值等于{jobParameter}
        如果/{jobName}/config/cron节点不存在或者{overwrite}等于true并且节点值不等于{cron}，创建永久节点，设置节点值等于{cron}
        如果/{jobName}/config/monitorExecution节点不存在或者{overwrite}等于true并且节点值不等于{monitorExecution}，创建永久节点，设置节点值等于{monitorExecution}
        如果/{jobName}/config/processCountIntervalSeconds节点不存在或者{overwrite}等于true并且节点值不等于{processCountIntervalSeconds}，创建永久节点，设置节点值等于{processCountIntervalSeconds}
        如果/{jobName}/config/concurrentDataProcessThreadCount节点不存在或者{overwrite}等于true并且节点值不等于{concurrentDataProcessThreadCount}，创建永久节点，设置节点值等于{concurrentDataProcessThreadCount}
        如果/{jobName}/config/fetchDataCount节点不存在或者{overwrite}等于true并且节点值不等于{fetchDataCount}，创建永久节点，设置节点值等于{fetchDataCount}
        如果/{jobName}/config/maxTimeDiffSeconds节点不存在或者{overwrite}等于true并且节点值不等于{maxTimeDiffSeconds}，创建永久节点，设置节点值等于{maxTimeDiffSeconds}
        如果/{jobName}/config/failover节点不存在或者{overwrite}等于true并且节点值不等于{failover}，创建永久节点，设置节点值等于{failover}
        如果/{jobName}/config/misfire节点不存在或者{overwrite}等于true并且节点值不等于{misfire}，创建永久节点，设置节点值等于{misfire}
        如果/{jobName}/config/jobShardingStrategyClass节点不存在或者{overwrite}等于true并且节点值不等于{jobShardingStrategyClass}，创建永久节点，设置节点值等于{jobShardingStrategyClass}
        如果/{jobName}/config/description节点不存在或者{overwrite}等于true并且节点值不等于{description}，创建永久节点，设置节点值等于{description}
        如果/{jobName}/config/monitorPort节点不存在或者{overwrite}等于true并且节点值不等于{monitorPort}，创建永久节点，设置节点值等于{monitorPort}
    public void checkMaxTimeDiffSecondsTolerable()
        设置maxTimeDiffSeconds变量等于/{jobName}/config/maxTimeDiffSeconds节点值
        如果maxTimeDiffSeconds变量等于-1
            退出
        获取当前时间和/{jobName}/systemTime/current节点的ctime时间差，如果时间差大于maxTimeDiffSeconds * 1000            // 节点类型为EPHEMERAL_SEQUENTIAL，所以每次ctime都死最新时间
            抛异常TimeDiffIntolerableException
    public boolean isMisfire()
        返回/{jobName}/config/misfire节点值
    public String getCron()
        返回/{jobName}/config/cron节点值
    public boolean isFailover()
        如果isMonitorExecution()等于true并且/{jobName}/config/failover节点值等于true
    public boolean isMonitorExecution()
        返回/{jobName}/config/monitorExecution节点值
    public int getShardingTotalCount()
        返回/{jobName}/config/shardingTotalCount节点值
    public String getJobParameter()
        返回/{jobName}/config/jobParameter节点值
    public int getFetchDataCount()
        返回/{jobName}/config/fetchDataCount节点值
    public Map<Integer, String> getShardingItemParameters()
        设置value变量等于/{jobName}/config/shardingItemParameters节点值
        如果value变量长度等于0
            返回空Map
        否则
            使用,分隔value变量成为元素数据，使用=分隔元素提取出key/value添加到map元素中
            返回map
    public String getJobShardingStrategyClass()
        返回/{jobName}/config/jobShardingStrategyClass节点值

## com.dangdang.ddframe.job.internal.failover.FailoverService

    public void setCrashedFailoverFlag(int item)
        如果/{jobName}/execution/{item}/failover节点不存在
            如果/{jobName}/leader/failover/items/{item}节点不存在，创建永久节点
    public void failoverIfNecessary()
        如果/{jobName}/leader/failover/items节点存在且其存在子节点且serverService.isServerReady()等于true
            LeaderLatch latch = new LeaderLatch(client, "/{jobName}/leader/failover/latch")      // 以/{jobName}/leader/failover/latch作为选举节点
            latch.start();                                                                       // 开始选举
            latch.await();                                                                       // 等待，直到当前节点作为选举后的主节点
            如果/{jobName}/leader/failover/items节点存在且其存在子节点且serverService.isServerReady()等于true
                获取/{jobName}/leader/failover/items节点下第一个子节点名称，设置给crashedItem变量
                创建/{jobName}/execution/{crashedItem}/failover临时节点，设置其值等于{ip}
                如果/{jobName}/leader/failover/items/{crashedItem}节点存在，删除节点
                执行JobRegistry.getInstance().getJobScheduler({jobName}).triggerJob()
            latch.close();                  // 关闭后，其他leader会再次选举成功，但此时leader/failover/items节点下的第一个子节点已被删除
    public void removeFailoverInfo()
        获取/{jobName}/execution节点下的所有子节点，并进行遍历
            如果/{jobName}/execution/{executionChildren.i}/failover节点存在，删除节点
    public List<Integer> getLocalHostFailoverItems()
        获取/{jobName}/execution节点下的所有子节点，设置给items变量，遍历items变量
            如果/{jobName}/execution/{items.i}/failover节点不存在或者节点值不等于{ip}，删除元素
        返回items变量
    public List<Integer> getLocalHostTakeOffItems()
        设置items变量等于shardingService.getLocalHostShardingItems()，遍历items变量
            如果/{jobName}/execution/{items.i}/failover节点不存在，删除元素
        返回items变量
    public void updateFailoverComplete(List<Integer> items)
        遍历items参数
            如果/{jobName}/execution/{items.i}/failover节点存在，删除节点

## com.dangdang.ddframe.job.internal.server.ServerService

    public void persistServerOnline()
        如果leaderElectionService.hasLeader()等于false
            执行leaderElectionService.leaderElection()
        如果/{jobName}/servers/{ip}/hostName节点不存在或者{overwrite}等于true并且节点值不等于{hostname}，创建永久节点，设置节点值等于{hostname}
        如果{overwrite}等于true
            如果{disabled}等于true
                如果/{jobName}/servers/{ip}/disabled节点不存在或者{overwrite}等于true并且节点值不等于""，创建永久节点，设置节点值等于""
            否则
                如果/{jobName}/servers/{ip}/disabled节点存在，删除节点
        创建/{jobName}/servers/{ip}/status临时节点，设置其值等于READY
    public void clearJobStopedStatus()
        如果/{jobName}/servers/{ip}/stoped节点存在，删除节点
    public boolean isServerReady()
        如果/{jobName}/servers/{ip}/disabled节点存在，返回false
        如果/{jobName}/servers/{ip}/stoped节点存在，返回false
        如果/{jobName}/servers/{ip}/status节点存在且节点值等于READY，返回true，否则返回false
    public boolean isJobStopedManually()
        如果/{jobName}/servers/{ip}/stoped节点存在，返回true，否则返回false
    public List<String> getAllServers()
        返回/{jobName}/servers节点下的所有子节点
    public List<String> getAvailableServers()
        设置servers变量等于getAllServers()，遍历servers变量
            如果/{jobName}/servers/{servers.i}/status节点不存在或者/{jobName}/servers/{servers.i}/disabled节点存在，删除元素
        返回servers变量
    public void updateServerStatus(ServerStatus status)
        更新/{jobName}/servers/{ip}/status节点值等于status

## com.dangdang.ddframe.job.internal.statistics.StatisticsService

    public void startProcessCountJob()
        设置processCountIntervalSeconds变量等于/{jobName}/config/processCountIntervalSeconds节点值
        如果processCountIntervalSeconds变量大于0
            启动定时任务，任务间隔时间为processCountIntervalSeconds
                设置/{jobName}/servers/{ip}/processSuccessCount节点值等于任务成功次数
                设置/{jobName}/servers/{ip}/processFailureCount节点值等于任务失败次数
                重置任务成功次数和任务失败次数
    public void stopProcessCountJob()
        停止定时任务

## com.dangdang.ddframe.job.internal.sharding.ShardingService

    public void setReshardingFlag()
        如果/{jobName}/leader/sharding/necessary节点不存在，创建永久节点，设置节点值等于""
    public boolean isNeedSharding()
        如果/{jobName}/leader/sharding/necessary节点存在，返回true，否则返回false
    public void shardingIfNecessary()
        如果isNeedSharding()等于false
            退出
        如果leaderElectionService.isLeader()等于false
            循环
                如果leaderElectionService.isLeader()等于true或者/{jobName}/leader/sharding/necessary节点不存在且/{jobName}/leader/sharding/processing节点不存在
                    跳出循环
                否则，睡眠100毫秒
            退出
        如果configService.isMonitorExecution()等于true
            循环
                如果executionService.hasRunningItems()等于false
                    跳出循环
                否则，睡眠100毫秒
        创建/{jobName}/leader/sharding/processing临时节点，设置节点值等于""
        执行事务
            设置servers变量等于serverService.getAllServers()，遍历servers
                删除/{jobName}/servers/{servers.i}/sharding节点
        设置jobShardingStrategy变量等于configService.getJobShardingStrategyClass()类型的实例，默认为AverageAllocationJobShardingStrategy
        设置option变量等于new JobShardingStrategyOption("{jobName}", configService.getShardingTotalCount(), configService.getShardingItemParameters())
        设置shardingItems变量等于jobShardingStrategy.sharding(serverService.getAvailableServers(), option)                    // 类型：Map<String, List<Integer>>
        执行事务
            遍历shardingItems变量
                创建/{jobName}/servers/{entry.key}/sharding节点，设置节点值等于{entry.value}使用,进行合并的字符串
            删除/{jobName}/leader/sharding/necessary节点
            删除/{jobName}/leader/sharding/processing节点
    public List<Integer> getLocalHostShardingItems()
        如果/{jobName}/servers/{ip}/sharding节点存在，获取节点值使用,分隔，转成List<Integer>类型返回，否则返回空

### com.dangdang.ddframe.job.plugin.sharding.strategy.AverageAllocationJobShardingStrategy

## com.dangdang.ddframe.job.internal.monitor.MonitorService

    public void listen()
        设置port变量等于/{jobName}/config/monitorPort节点值
        如果port变量大于0
            启动一个ServerSocket实例，端口使用port变量，使用一个线程接收所有Socket，获取Socket发送的请求内容，如果是dump
                获取/{jobName}下所有节点和子节点信息发送给Socket
    public void close()
        关闭serverSocket

## com.dangdang.ddframe.job.internal.schedule.JobTriggerListener

    public void triggerMisfired(Trigger trigger)
        执行executionService.setMisfire(shardingService.getLocalHostShardingItems())

## com.dangdang.ddframe.job.internal.execution.ExecutionService

    public void clearRunningInfo(List<Integer> items)
        遍历items参数
            如果/{jobName}/execution/{items.i}/running节点存在，删除节点
    public void setNeedFixExecutionInfoFlag()
        如果/{jobName}/leader/execution/necessary节点不存在，创建永久节点，设置节点值等于""
    public void removeExecutionInfo()
        如果节点/{jobName}/execution存在，删除节点
    public boolean isCompleted(int item)
        如果/{jobName}/execution/{item}/completed节点存在，返回true，否则返回false
    public boolean hasRunningItems(List<Integer> items)
        如果configService.isMonitorExecution()等于false
            返回false
        如果存在/{jobName}/execution/{items.i}/running节点
            返回true
        返回false
    public boolean hasRunningItems()
        获取/{jobName}/execution下的所有子节点，设置给items变量
        返回hasRunningItems(items)
    public void setMisfire(List<Integer> items)
        如果configService.isMonitorExecution()等于true，遍历items
            如果/{jobName}/execution/{items.i}/misfire节点不存在，创建节点，设置节点值等于""
    public boolean misfireIfNecessary(List<Integer> items)
        如果hasRunningItems(items)等于true
            执行setMisfire(items)，返回true
        否则，返回false
    public List<Integer> getMisfiredJobItems(List<Integer> items)
        遍历items参数
            如果/{jobName}/execution/{items.i}/misfire节点不存在，删除元素
        返回items参数
    public void clearMisfire(List<Integer> items)
        遍历items参数
            删除/{jobName}/execution/{items.i}/misfire节点
    public void registerJobBegin(JobExecutionMultipleShardingContext context)
        如果context.getShardingItems()元素个数大于0并且configService.isMonitorExecution()等于true
            执行serverService.updateServerStatus(ServerStatus.RUNNING)
            遍历context.getShardingItems()
                如果/{jobName}/execution/{items.i}/completed节点存在，删除节点
                设置/{jobName}/execution/{items.i}running节点值等于""
                设置/{jobName}/execution/{items.i}/lastBeginTime节点值等于当前时间戳
                设置jobScheduler变量等于JobRegistry.getInstance().getJobScheduler("{jobName}")
                如果jobScheduler变量不等于null
                    设置nextFireTime变量等于jobScheduler.getNextFireTime()
                    如果nextFireTime变量不等于null
                        设置/{jobName}/execution/{items.i}/nextFireTime节点值等于nextFireTime.getTime()
    public void registerJobCompleted(JobExecutionMultipleShardingContext context)
        如果configService.isMonitorExecution()等于true
            执行serverService.updateServerStatus(ServerStatus.READY)
            遍历context.getShardingItems()
                如果/{jobName}/execution/{items.i}/completed节点不存在，创建永久节点，设置节点值等于""
                删除/{jobName}/execution/{items.i}running节点
                设置/{jobName}/execution/{items.i}/lastCompleteTime节点值等于当前时间戳

## com.dangdang.ddframe.job.internal.schedule.JobRegistry

    public void addJobScheduler(String jobName, JobScheduler jobScheduler)
       添加jobName和jobScheduler到schedulerMap属性中
    public JobScheduler getJobScheduler(String jobName)
       返回schedulerMap属性中jobName的实例

    public void addJobInstance(String jobName, ElasticJob job)
       添加jobName和job到instanceMap属性中
    public ElasticJob getJobInstance(String jobName)
       返回instanceMap属性中jobName的实例

# com.dangdang.ddframe.job.internal.execution.ExecutionContextService

    public JobExecutionMultipleShardingContext getJobExecutionShardingContext()
        设置result变量等于new JobExecutionMultipleShardingContext()
        设置result.jobName属性等于{jobName}
        设置result.shardingTotalCount属性等于configService.getShardingTotalCount()
        设置result.jobParameter属性等于configService.getJobParameter()
        设置result.fetchDataCount属性等于configService.getFetchDataCount()
        设置result.monitorExecution属性等于configService.isMonitorExecution()

        设置shardingItems变量等于shardingService.getLocalHostShardingItems()
        如果configService.isFailover()等于true
            设置failoverItems变量等于failoverService.getLocalHostFailoverItems()
            如果failoverItems变量元素个数大于0
                设置result.shardingItems属性等于failoverItems变量
            否则
                移除shardingItems变量中failoverService.getLocalHostTakeOffItems()列表
                设置给result.shardingItems属等于shardingItems变量
        否则
            设置给result.shardingItems属等于shardingItems变量
        如果result.isMonitorExecution()等于true
            遍历shardingItems变量
                如果/{jobName}/execution/{items.i}/running节点存在，移除{items.i}元素
        如果result.shardingItems属性的元素个数等于0
            返回result
        设置shardingItemParameters变量等于configService.getShardingItemParameters()
        遍历result.shardingItems属性
            如果shardingItemParameters中包含{items.key}
                添加{items.key}和{items.value}到result.shardingItemParameters属性中
        执行result.setOffsets(offsetService.getOffsets(result.getShardingItems()))
        返回result变量

# com.dangdang.ddframe.job.internal.offset.OffsetService

    public Map<Integer, String> getOffsets(List<Integer> items)
        遍历items
            如果/{jobName}/offset/{items.i}节点值的长度大于0，添加items.i和节点值到map中
        返回map

# com.dangdang.ddframe.job.internal.job.AbstractElasticJob              // 实现了Job接口

    public void execute(JobExecutionContext context) throws JobExecutionException       // Quartz定义
        执行configService.checkMaxTimeDiffSecondsTolerable()
        执行shardingService.shardingIfNecessary()
        设置shardingContext变量等于executionContextService.getJobExecutionShardingContext()
        如果executionService.misfireIfNecessary(shardingContext.getShardingItems())等于true
            退出
        执行executeJobInternal(shardingContext)
        循环
            如果configService.isMisfire()等于true并且executionService.getMisfiredJobItems(shardingContext.getShardingItems())元素个数大于0并且stoped等于false并且shardingService.isNeedSharding()等于false
                执行executionService.clearMisfire(shardingContext.getShardingItems())
                执行executeJobInternal(shardingContext)
        如果configService.isFailover()等于true并且stoped等于false
            执行failoverService.failoverIfNecessary()
    private void executeJobInternal(JobExecutionMultipleShardingContext shardingContext) throws JobExecutionException
        如果shardingContext.getShardingItems()元素个数等于0
            退出
        执行executionService.registerJobBegin(shardingContext);
        try {
            执行executeJob(shardingContext)
        } finally
            执行executionService.registerJobCompleted(shardingContext)
            如果configService.isFailover()等于true
                执行failoverService.updateFailoverComplete(shardingContext.getShardingItems())
        }
    public void stop()
        设置stop属性等于true
    public void resume()
        设置stop属性等于false
    protected abstract void executeJob(JobExecutionMultipleShardingContext shardingContext)

## com.dangdang.ddframe.job.plugin.job.type.simple.AbstractSimpleElasticJob

## com.dangdang.ddframe.job.plugin.job.type.dataflow.AbstractBatchSequenceDataFlowElasticJob        // 多线程处理；每个线程处理一个shardItem得到的数据

## com.dangdang.ddframe.job.plugin.job.type.dataflow.AbstractBatchThroughputDataFlowElasticJob      // 多线程处理；将最终的数据重新均分，每个线程平均处理一份
