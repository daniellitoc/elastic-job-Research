# ZK节点

    /namespace/jobName/
        config
            jobClass                            // （否）实现类名称
            shardingTotalCount			        // （否）分片总数
            shardingItemParameters              // （否）序列号和个性化参数对照表
            jobParameter                        // （否）自定义参数
            cron                                // （否）启动时间的cron表达式
            monitorExecution                    // （否）监控任务执行时状态
            processCountIntervalSeconds         // （否）统计任务处理数据数量的间隔时间
            concurrentDateProcessThreadCount    // （否）同时处理数据的并发线程数
            fetchDataCount                      // （否）每次抓取的数据量
            maxTimeDiffSeconds                  // （否）允许的本机与注册中心的时间误差秒数
            failover                            // （否）是否开启失效转移
            misfire                             // （否）是否开启错过任务重新执行
            jobShardingStrategyClass            // （否）分片策略类
            description                         // （否）描述
            monitorPort                         // （否）监控端口

        servers
            127.0.0.1
                hostName                        // （否）实例名称
                status 					        // （是）实例状态（READY/RUNNING）。用于表示服务器在等待执行任务还是正在执行任务，如果status节点不存在则表示任务服务器未上线
                disabled				        // （否）实例状态是否禁用。可用于部署任务时，先禁止启动，部署结束后统一启动
                sharding 				        // （否）该实例分到的分片项。多个分片项用逗号分隔，如：0, 1, 2代表该服务器执行第1, 2, 3片分片
                processSuccessCount             // （否）统计processCountIntervalSeconds间隔内处理成功的数量
                processFailureCount             // （否）统计processCountIntervalSeconds间隔内处理失败的数量
                stoped                          // （否）停止实例的标记
            127.0.0.2
                ...

        execution
            1
                running                         // （是）分片项正在运行的状态。如果没有此节点，并且没有completed节点，表示该分片未运行
                completed                       // （否）分片项运行完成的状态。下次任务开始执行时会清理
                failover                        // （是）如果该分片项被失效转移分配给其他任务服务器，则此节点值记录执行此分片的任务服务器IP
                lastBeginTime                   // （否）分片项最近一次的开始执行时间
                nextFireTime                    // （否）分片项下次触发时间
                lastCompleteTime                // （否）分片项最近一次的结束执行时间
            2
                ...
        leader
            election/host			            // （是）主节点服务器IP地址。一旦该节点被删除将会触发重新选举，重新选举的过程中一切主节点相关的操作都将阻塞
            election/latch                      // （否）主节点选举的分布式锁
            sharding/necessary		            // （否）是否需要重新分片的标记。如果分片总数变化，或任务服务器节点上下线或启用/禁用，以及主节点选举，会触发设置重分片标记，任务在下次执行时使用主节点重新分片，且中间不会被打断，任务执行时不会触发分片
            sharding/processing                 // （是）主节点在分片时持有的节点。如果有此节点，所有的任务执行都将阻塞，直至分片结束，主节点分片结束或主节点崩溃会删除此临时节点
            execution/necessary                 // （否）是否需要修正任务执行时分片项信息的标记，如果分片总数变化，会触发设置修正分片项信息标记，任务在下次执行时会增加或减少分片项数量
            execution/cleaning                  // （是）主节点在清理上次任务运行时状态时所持有的节点，每次开始新任务都需要清理上次运行完成的任务信息，如果有此节点，所有的任务执行都将阻塞，直至清理结束，主节点分片结束或主节点崩溃会删除此临时节点
            failover/items/1                    // （否）一旦有任务崩溃，则会向此节点记录。当有空闲任务服务器时，会从此节点抓取需失效转移的任务项
            failover/items/latch                // （否）分配失效转移分片项时占用的分布式锁

        systemTime/current

        offset
            1
            2

# ZK监听器

    leader/election/host节点被删除且不存在leader/election/host节点时
    	重新选举主节点（leader/election/latch），成功被选举的机器
    	    不存在创建主节点（leader/election/host），设置节点值等于主节点服务器IP地址
    	    关闭当前leader                                                  // 关闭后其他机器也会进行选举，但主节点已存在，所以无意义
    	标记需要重新分片（不存在创建leader/sharding/necessary节点）

    config/shardingTotalCount节点添加或删除或内容更新时
        标记需要重新分片（不存在创建leader/sharding/necessary节点）
        标记需要修正任务执行时分片项信息（不存在创建leader/execution/necessary节点）

    servers/{IP}/status节点添加或删除时
        标记需要重新分片（不存在创建leader/sharding/necessary节点）

    servers/{IP}/disabled节点添加或删除或内容更新时
        标记需要重新分片（不存在创建leader/sharding/necessary节点）

    config/monitorExecution节点内容更新且内容等于false时
        存在删除execution节点

    execution/{item}/running节点被删除且execution/{item}/completed不存在且config/failover节点值等于true时           // 运行失败
        如果execution/{item}/failover节点不存在，标记任务崩溃（不存在创建leader/failover/items/{item}节点）
        获取实例的分片项（servers/{IP}/sharding节点值），如果没有分片项在运行（不存在/execution/{shardItem}/running节点）
            如果leader/failover/items节点存在且存在子节点且实例处于READY状态（servers/{IP}/disabled不存在，servers/{IP}/stoped不存在，servers/{IP}/status节点值等于READY）
                选举分配失效转移主节点（leader/failover/latch），成功被选举的机器
                    如果有失败的分片项（leader/failover/items存在子节点），取第一个失败的分片项并删除节点（leader/failover/items/{firstItem}节点）
                        标记分片项被失效转移分配给的实例（execution/{firstItem}/failover节点值等于leader的IP）        // 将失败分片想划归到leader上
                        执行jobScheduler.triggerJob()                                                          // 触发Job，会优先执行失败分片项
                    关闭当前leader                                        // 关闭后其他机器也会进行选举，但需要校验leader/failover/items下是否有子节点

    execution/{item}/failover节点被删除且execution/{item}/completed不存在且config/failover节点值等于true时
        如果execution/{item}/failover节点不存在，标记任务崩溃（不存在创建leader/failover/items/{item}节点）
        获取实例的分片项（servers/{IP}/sharding节点值），如果没有分片项在运行（不存在/execution/{shardItem}/running节点）
            如果leader/failover/items节点存在且存在子节点且实例处于READY状态（servers/{IP}/disabled不存在，servers/{IP}/stoped不存在，servers/{IP}/status节点值等于READY）
                选举分配失效转移主节点（leader/failover/latch），成功被选举的机器
                    如果有失败的分片项（leader/failover/items存在子节点），取第一个失败的分片项并删除节点（leader/failover/items/{firstItem}节点）
                        标记分片项被失效转移分配给的实例（execution/{firstItem}/failover节点值等于leader的IP）        // 将失败分片想划归到leader上
                        执行jobScheduler.triggerJob()                                                          // 触发Job，会优先执行失败分片项
                    关闭当前leader                                        // 关闭后其他机器也会进行选举，但需要校验leader/failover/items下是否有子节点

    config/failover节点内容更新且内容等于false时
        清理分片项失效转移标记（存在删除execution/.../failover类节点）

    连接状态等于LOST
        执行jobScheduler.stopJob()

    连接状态等于RECONNECTED
        执行jobScheduler.resumeCrashedJob()       // 和resumeManualStopedJob不同在于，此时可能因为网络问题，需要确认主节点信息和重置实例节点信息

    servers/{IP}/stoped节点被添加时
        执行jobScheduler.stopJob()

    servers/{IP}/stoped节点被删除时
        执行jobScheduler.resumeManualStopedJob()

    config/cron节点内容更新时
        执行jobScheduler.rescheduleJob(cronExpression)

# Quartz Trigger监听器

    触发misfire
        获取实例的分片项（servers/{IP}/sharding节点值），标记错过任务的分片项（不存在创建execution/{shardItem1}/misfire，execution/{shardItem2}/misfire节点）

# JobScheduler初始化

    选举主节点（leader/election/latch），成功被选举的机器
        不存在创建主节点（leader/election/host），设置节点值等于主节点服务器IP地址
        关闭当前leader                                                  // 关闭后其他机器也会进行选举，但主节点已存在，所以无意义
    持久化Job配置
        不存在或不等创建config/jobClass
        不存在或不等创建config/shardingTotalCount
        不存在或不等创建config/shardingItemParameters
        不存在或不等创建config/jobParameter
        不存在或不等创建config/cron
        不存在或不等创建config/monitorExecution
        不存在或不等创建config/processCountIntervalSeconds
        不存在或不等创建config/concurrentDataProcessThreadCount
        不存在或不等创建config/fetchDataCount
        不存在或不等创建config/maxTimeDiffSeconds
        不存在或不等创建config/failover
        不存在或不等创建config/misfire
        不存在或不等创建config/jobShardingStrategyClass
        不存在或不等创建config/description
        不存在或不等创建config/monitorPort
    标识实例READY状态
        不存在或不等创建servers/{IP}/hostName节点，设置节点值等于当前服务器hostname
        如果jobConfig.disabled等于true
            不存在或不等创建servers/{IP}/disabled节点，设置节点值等于""
        否则
            存在删除servers/{IP}/disabled节点
        创建servers/{IP}/status临时节点，设置节点值READY
    清理停止状态（存在删除servers/{IP}/stoped节点）
    定时统计实例执行状态
        根据/config/processCountIntervalSeconds节点值启动定时任务，设置servers/{IP}/processSuccessCount和servers/{IP}/processFailureCount节点值
    标记需要重新分片（不存在创建leader/sharding/necessary节点）
    开启DEBUG功能
        启动ServerSocket实例，绑定config/monitorPort节点值，接受客户端请求，请求内容为DUMP，发送Job下所有节点数据给对方
    启动Quartz任务
        使用StdSchedulerFactory单独创建Job实例，根据config/cron节点值创建Trigger，进行Job调度

# Quartz Job执行

    execute
        校验系统时间和ZK时间差（systemTime/current），如果大于config/maxTimeDiffSeconds节点值抛异常
        开始分片
            检测是否开始（leader/sharding/necessary节点不存在），退出开始分片逻辑
            循环等待直到leader/election/host节点存在，之后判断如果节点值不等于当前服务器IP                      // 不是主节点对应的实例
                // 等到自己成为主节点或者主节点已经分片完成
                循环等待直到leader/election/host节点存在且节点值等于当前服务器IP或leader/sharding/necessary，leader/sharding/processing节点不存在
                退出开始分片逻辑
            如果config/monitorExecution节点值等于true                                               // 主节点对应的实例在执行
                循环等待，直到execution节点下，所有分片项节点下的running节点不存在                        // 主节点对应的实例在执行
            创建leader/sharding/processing临时节点                                                  // 主节点对应的实例在执行
            删除servers节点下，所有实例下的sharding节点                                               // 主节点对应的实例在执行
            使用config/jobShardingStrategyClass策略，根据                                           // 主节点对应的实例在执行
                config/shardingTotalCount节点值
                config/shardingItemParameters节点值
                可用的服务节点（servers节点下所有所有实例下status节点存在disabled节点不存在的节气）
                                                       生成实例分片项信息。             // Map<IP, List<Item>>

            遍历实例分片项信息，在servers/{IP}/sharding节点，设置节点值等于List<Item>使用,拼接的字符串     // 主节点对应的实例在执行
            删除leader/sharding/necessary节点                                                      // 主节点对应的实例在执行
            删除leader/sharding/processing节点                                                     // 主节点对应的实例在执行

        创建JobExecutionShardingContext                                                           // 主要是获取各自应该执行的分片项，优先执行failover
            填充JobName
            填充config/shardingTotalCount节点值
            填充config/jobParameter节点值
            填充config/fetchDataCount节点值
            填充config/monitorExecution节点值

            如果config/failover节点值等于true
                获取execution节点下，所有分片项节点下的failover节点存在且节点值等于当前IP的节点列表（List<Item>），如果列表不为空设置给shardingItems属性                  // 本来就应该是我执行的failover列表
                如果为空
                    获取servers/{IP}/sharding节点值，使用,分隔，转成List<Item>，排除掉在execution节点下指定分片项节点下的failover节点存在的元素                        // 我预期执行的减去别人应该执行的failover列表，剩下的才是我应该执行的
                    在排除掉存在execution节点下指定分片项节点下的running节点存在的元素，设置给shardingItems属性                                                      // 在排除正在执行的
            否则
                获取servers/{IP}/sharding节点值，使用,分隔，转成List<Item>，排除掉在execution节点下指定分片项节点下的running节点存在的元素，设置给shardingItems属性      // 我预期执行的减去正在执行的，剩下的才是我应该执行的
            如果shardingItems属性不等于空
                根据config/shardingItemParameters节点值和shardingItems属性，设置shardingItemParameters属性
                根据shardingItems属性，获取对应分片项的offset（offset/1、offset/2），设置offsets属性

        如果context.shardingItems属性中，存在在execution节点下指定分片项节点下的running节点存在，标记shardingItems属性中的分片项全部重新执行（不存在创建execution/2/misfire节点）
            退出Job

        执行executeJobInternal

        循环
            如果config/misfire节点值等于true并且在execution节点下存在context.shardingItems属性中的misfire节点并且stoped属性等于false并且leader/sharding/necessary节点不存在
                遍历context.shardingItems属性，删除execution/1/misfire节点
                执行executeJobInternal

        如果config/failover节点值等于true并且stoped属性等于false
            如果leader/failover/items节点存在且存在子节点且实例处于READY状态（servers/{IP}/disabled不存在，servers/{IP}/stoped不存在，servers/{IP}/status节点值等于READY）
                选举分配失效转移主节点（leader/failover/latch），成功被选举的机器
                    如果有失败的分片项（leader/failover/items存在子节点），取第一个失败的分片项并删除节点（leader/failover/items/{firstItem}节点）
                        标记分片项被失效转移分配给的实例（execution/{firstItem}/failover节点值等于leader的IP）        // 将失败分片想划归到leader上
                        执行jobScheduler.triggerJob()                                                          // 触发Job，会优先执行失败分片项
                    关闭当前leader                                        // 关闭后其他机器也会进行选举，但需要校验leader/failover/items下是否有子节点

    executeJobInternal
        如果context.shardingItems属性不等于空
            如果config/monitorExecution节点值等于true
                设置servers/{IP}/status临时节点值等于RUNNING
            遍历context.shardingItems
                删除execution/{shardingItem}/completed节点
                创建execution/{shardingItem}/running节点
                设置execution/{shardingItem}/lastBeginTime节点值等于当前时间戳
                计算距离现在最近的下一次执行时间，设置execution/{shardingItem}/nextFireTime节点值等于该执行时间
            try {
                executeJob(shardingContext)
            } finally {
                如果config/monitorExecution节点值等于true
                    设置servers/{IP}/status临时节点值等于READY
                遍历context.shardingItems
                    创建execution/{shardingItem}/completed节点
                    删除execution/{shardingItem}/running节点
                    设置execution/{shardingItem}/lastCompleteTime节点值等于当前时间戳
                如果config/failover节点值等于true
                    遍历context.shardingItems
                        删除execution/{shardingItem}/failover节点
            }

# Elastic Job执行

    executeJob                      // 业务Job

# 简述

    启动或删除一个节点的时候，会重新触发主节点选举，对于第一个选举成功的，设置election/host节点值等于当前节点ip，并创建necessary节点
    注册job配置到config节点
    设置servers/{IP}/status节点值等于READY，删除servers/{IP}/stopped节点和servers/{IP}/disabled节点
    创建necessary节点（标记需要重新分片）
    根据config/cron节点值，启动Quartz（每个Job创建单独的StdSchedulerFactory）
    	Quartz Job
    		对于非主节点，会检测直到主节点值等于当前节点ip或者necessary节点不存在
    		对于主节点
    			等待所有execution/{shardItem}/running节点不存在
    			删除和重新分配servers/{IP}/sharding节点，对应的节点值为应该执行的分片项，使用,拼接形式
    			删除necessary节点
    		创建SharingContext
    			获取实际分片项
    				优先获取execution/{shardItem}/failover节点值等于当前节点ip的（这些节点表示当{shardItem}节点下的running或failover被删除同时completed不存在时，表示任务执行失败，添加failover/items/{item}节点并进行选举，第一个选举成功的删除failover/items下的第一个节点和设置对应的execution/{shardItem}/failover节点值等于选举成功的节点ip
    				如果没有，则根据servers/{IP}/sharding节点值，减去其中{shardItem}/running存在的
    			根据分片项参数和实际的分片项，设置当前节点的分片项参数
    		如果ShardingContext中的实际分片项中，其中有{shardItem}/running存在的，遍历全部实际分片项，创建{shardItem}/misfire（标记需要重新执行）
    			退出Job
    		否则
    			设置servers/{IP}/status节点值等于RUNNING
    			删除所有分片项的completed节点，添加running节点
    			try
    				执行真正Job
    			finally
    				设置servers/{IP}/status节点值等于READY
    				创建所有分片项的completed节点，删除running和failover节点




