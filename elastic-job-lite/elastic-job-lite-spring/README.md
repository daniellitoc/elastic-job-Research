# xml配置

## reg:zookeeper

    获取元素的serverLists属性，设置给serverLists变量
    获取元素的namespace属性，设置给namespace变量
    获取元素的baseSleepTimeMilliseconds属性，设置给baseSleepTimeMilliseconds变量
    获取元素的maxSleepTimeMilliseconds属性，设置给maxSleepTimeMilliseconds变量
    获取元素的maxRetries属性，设置给maxRetries变量
    创建new SpringZookeeperConfigurationDto(serverLists, namespace, baseSleepTimeMilliseconds, maxSleepTimeMilliseconds, maxRetries)实例，设置给result变量
    获取元素的sessionTimeoutMilliseconds属性，设置给result变量的sessionTimeoutMilliseconds属性
    获取元素的connectionTimeoutMilliseconds属性，设置给result变量的connectionTimeoutMilliseconds属性
    获取元素的digest属性，设置给result变量的digest属性
    获取元素的localPropertiesPath属性，设置给result变量的localPropertiesPath属性
    获取元素的overwrite属性，设置给result变量的overwrite属性

    创建new GenericBeanDefinition()实例，设置给definition变量，设置其beanClass属性等于SpringZookeeperRegistryCenter
    设置definition变量创建bean实例的构造参数等于result变量
    设置definition变量的destroyMethodName属性等于close
    获取元素的id属性设置给id变量，如果id变量等于null，抛异常Id is required for element...
    创建new BeanDefinitionHolder(definition, id)实例，添加到Spring的Registry中

### com.dangdang.ddframe.reg.spring.namespace.SpringZookeeperRegistryCenter         // 实现了BeanFactoryPostProcessor

    public SpringZookeeperRegistryCenter(SpringZookeeperConfigurationDto configDto)
       执行super(new ZookeeperConfiguration())
       设置configDto属性等于configDto参数
    public void postProcessBeanFactory(ConfigurableListableBeanFactory beanFactory)           // 将dto的属性通过属性置换器进行处理，设置给zkConfig的对应属性
       设置zkConfig.serverLists属性等于使用spring的属性置换器处理过的configDto.serverLists属性
       设置zkConfig.namespace属性等于使用spring的属性置换器处理过的configDto.namespace属性
       设置zkConfig.baseSleepTimeMilliseconds属性等于使用spring的属性置换器处理过的configDto.baseSleepTimeMilliseconds属性
       设置zkConfig.maxSleepTimeMilliseconds属性等于使用spring的属性置换器处理过的configDto.maxSleepTimeMilliseconds属性
       设置zkConfig.maxRetries属性等于使用spring的属性置换器处理过的configDto.maxRetries属性
       如果使用spring的属性置换器处理过的configDto.sessionTimeoutMilliseconds属性不等于null，设置给zkConfig.sessionTimeoutMilliseconds属性
       如果使用spring的属性置换器处理过的configDto.connectionTimeoutMilliseconds属性不等于null，设置给zkConfig.connectionTimeoutMilliseconds属性
       如果使用spring的属性置换器处理过的configDto.digest属性不等于null，设置给zkConfig.digest属性
       如果使用spring的属性置换器处理过的configDto.localPropertiesPath属性不等于null，设置给zkConfig.localPropertiesPath属性
       如果使用spring的属性置换器处理过的configDto.overwrite属性不等于null，设置给zkConfig.overwrite属性
       执行init()

## reg:placeholder

    创建new RootBeanDefinition()实例，设置给sourceDefinition变量，设置beanClass属性等于RegistryPropertySources
    获取元素的registerRef属性，作为sourceDefinition变量创建bean实例的构造参数引用

    创建new RootBeanDefinition()实例，设置给definition变量，设置其beanClass属性等于PropertySourcesPlaceholderConfigurer
    设置definition变量的propertyValues中ignoreUnresolvablePlaceholders属性等于true
    设置definition变量的propertyValues中propertySources属性等于sourceDefinition变量
    设置id变量等于org.springframework.context.support.PropertySourcesPlaceholderConfigurer，判断在Spring的Registry中是否存在，如果存在通过 id = id + # + counter来保证唯一
    创建new BeanDefinitionHolder(definition, id)实例，添加到Spring的Registry中

### com.dangdang.ddframe.reg.spring.placeholder.RegistryPropertySources

    public RegistryPropertySources(RegistryCenter registryCenter)
        执行addLast(new RegistryPropertySource(registryCenter))           // 添加基于注册中心的属性置换源

### com.dangdang.ddframe.reg.spring.placeholder.RegistryPropertySource

    public RegistryPropertySource(RegistryCenter source)           // 基于注册中心的属性置换源：提供读取注册中心数据
        设置source属性等于source参数
    public Object getProperty(String name)
        返回source.get(name)

## job:bean

    创建new GenericBeanDefinition()实例，设置给jobDefinition变量，设置其beanClass属性等于JobConfiguration
    分别获取元素的id、class、shardingTotalCount、cron作为jobDefinition变量创建bean实例的构造参数
    如果元素的shardingItemParameters属性不等于null，获取属性值，添加到jobDefinition变量的propertyValues中
    如果元素的jobParameter属性不等于null，获取属性值，添加到jobDefinition变量的propertyValues中
    如果元素的monitorExecution属性不等于null，获取属性值，添加到jobDefinition变量的propertyValues中
    如果元素的monitorPort属性不等于null，获取属性值，添加到jobDefinition变量的propertyValues中
    如果元素的processCountIntervalSeconds属性不等于null，获取属性值，添加到jobDefinition变量的propertyValues中
    如果元素的concurrentDataProcessThreadCount属性不等于null，获取属性值，添加到jobDefinition变量的propertyValues中
    如果元素的fetchDataCount属性不等于null，获取属性值，添加到jobDefinition变量的propertyValues中
    如果元素的maxTimeDiffSeconds属性不等于null，获取属性值，添加到jobDefinition变量的propertyValues中
    如果元素的failover属性不等于null，获取属性值，添加到jobDefinition变量的propertyValues中
    如果元素的misfire属性不等于null，获取属性值，添加到jobDefinition变量的propertyValues中
    如果元素的jobShardingStrategyClass属性不等于null，获取属性值，添加到jobDefinition变量的propertyValues中
    如果元素的description属性不等于null，获取属性值，添加到jobDefinition变量的propertyValues中
    如果元素的disabled属性不等于null，获取属性值，添加到jobDefinition变量的propertyValues中
    如果元素的overwrite属性不等于null，获取属性值，添加到jobDefinition变量的propertyValues中
    如果元素的processCountIntervalSeconds属性不等于null，获取属性值，添加到jobDefinition变量的propertyValues中
    设置jobName变量等于元素的id属性 + Conf，添加jobName变量和jobDefinition变量到Spring的Registry中

    创建new RootBeanDefinition()实例，设置给definition变量，设置其beanClass属性等于SpringJobScheduler
    设置definition变量的initMethodName和destroyMethodName分别等于init和shutdown
    获取元素的regCenter属性，设置给regCenter变量
    设置definition变量创建bean实例的构造参数引用分别为regCenter变量和jobName变量
    设置id变量等于com.dangdang.ddframe.job.spring.schedule.SpringJobScheduler，判断在Spring的Registry中是否存在，如果存在通过 id = id + # + counter来保证唯一
    创建new BeanDefinitionHolder(definition, id)实例，添加到Spring的Registry中

### com.dangdang.ddframe.job.api.JobConfiguration

    final String jobName;                           // 作业名称.
    final Class<? extends ElasticJob> jobClass;     // 作业实现类名称.
    final int shardingTotalCount;                   // 作业分片总数.
    final String cron;                              // 作业启动时间的cron表达式.
    /**
     * 分片序列号和个性化参数对照表.
     * 分片序列号和参数用等号分隔, 多个键值对用逗号分隔. 类似map. 分片序列号从0开始, 不可大于或等于作业分片总数. 如: 0=a,1=b,2=c
     */
    String shardingItemParameters = "";
    String jobParameter = "";                       // 作业自定义参数. 可以配置多个相同的作业, 但是用不同的参数作为不同的调度实例.
    /**
     * 监控作业执行时状态.
     * 每次作业执行时间和间隔时间均非常短的情况, 建议不监控作业运行时状态以提升效率, 因为是瞬时状态, 所以无必要监控. 请用户自行增加数据堆积监控. 并且不能保证数据重复选取, 应在作业中实现幂等性. 也无法实现作业失效转移.
     * 每次作业执行时间和间隔时间均较长短的情况, 建议监控作业运行时状态, 可保证数据不会重复选取.
     */
    boolean monitorExecution = true;
    int processCountIntervalSeconds = 300;          // 统计作业处理数据数量的间隔时间. 单位: 秒. 只对处理数据流类型作业起作用.
    int concurrentDataProcessThreadCount = 1;       // 处理数据的并发线程数. 只对高吞吐量处理数据流类型作业起作用.
    int fetchDataCount = 1;                         // 每次抓取的数据量. 可在不重启作业的情况下灵活配置抓取数据量.
    /**
     * 最大容忍的本机与注册中心的时间误差秒数.
     * 如果时间误差超过配置秒数则作业启动时将抛异常.
     * 配置为-1表示不检查时间误差.
     */
    int maxTimeDiffSeconds = -1;
    boolean failover;                               // 是否开启失效转移. 只有对monitorExecution的情况下才可以开启失效转移.
    boolean misfire = true;                         // 是否开启misfire.
    int monitorPort = -1;                           // 作业辅助监控端口.
    String jobShardingStrategyClass = "";           // 作业分片策略实现类全路径. 默认使用{@code com.dangdang.ddframe.job.plugin.sharding.strategy.AverageAllocationJobShardingStrategy}.
    String description = "";                        // 作业描述信息.
    boolean disabled;                               // 作业是否禁止启动. 可用于部署作业时, 先禁止启动, 部署结束后统一启动.
    boolean overwrite;                              // 本地配置是否可覆盖注册中心配置. 如果可覆盖, 每次启动作业都以本地配置为准.

    public JobConfiguration(String jobName, Class<? extends ElasticJob> jobClass, int shardingTotalCount, String cron)
        复制参数到属性中

### com.dangdang.ddframe.job.spring.schedule.SpringJobScheduler     // 实现了ApplicationContextAware

    public SpringJobScheduler(CoordinatorRegistryCenter registryCenter, JobConfiguration jobConfig)
        super(registryCenter, jobConfig)
    public void setApplicationContext(ApplicationContext applicationContext)
        设置applicationContext属性等于applicationContext参数

    protected void prepareEnvironments(Properties props)
        设置SpringJobFactory的静态变量applicationContext等于applicationContext属性
        添加org.quartz.scheduler.jobFactory.class和SpringJobFactory到props参数中           // 如果是通过Spring注解启动，配置JobFactory使用SpringJobFactory

### com.dangdang.ddframe.job.spring.schedule.SpringJobFactory

    public Job newJob(TriggerFiredBundle bundle, Scheduler scheduler) throws SchedulerException
        获取applicationContext中所有的Job类型的实例，并进行遍历
            如果bundle.getJobDetail().getJobClass()和当前job的targetClass相等
                设置给job变量
        如果job变量等于null，设置job变量等于super.newJob(bundle, scheduler)                  // 如果Spring中存在对应targetClass的bean，使用此bean作为Job，否则使用Quartz创建Job

        创建new JobDataMap()实例设置给jobDataMap变量，添加scheduler.getContext()、bundle.getJobDetail().getJobDataMap()、bundle.getTrigger().getJobDataMap()到其中
        如果job变量是代理对象，则获取被代理对象设置给job变量
        执行setBeanProps(job, jobDataMap)                                                // setter方法
        返回job
        