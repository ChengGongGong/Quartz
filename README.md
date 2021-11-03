# Quartz
## 1. 核心元素：Scheduler、Trigger、Job
### Job：任务，具体要执行的业务逻辑
    1. org.quartz.Job接口，job的实现类需要实现该接口，并重写其execute()方法。
    2. JobDetailImpl类实现了JobDetail接口，用来描述一个job，定义了job所有属性：
       class：job实现类，用来绑定一个具体job；
       name：job名称。如果未指定，会自动分配一个唯一名称，如果两个job的name重复，则只有最前面的job能被调度；
       group：job所属的组名；
       description：	job描述；
       durability：是否持久化，默认为非持久，当没有活跃的trigger与之关联的时候，job会自动从scheduler中删除；
       shouldRecover：	是否可恢复。如果设置为可恢复，一旦job执行时scheduler进程崩溃或关机，当scheduler重启后，该job会被重新执行；
       jobDataMap：用户可以把任意kv数据存入jobDataMap，实现job属性的无限制扩展。
    3. org.quartz.Scheduler#scheduleJob(org.quartz.JobDetail, org.quartz.Trigger)注册任务和定时器到调度器的jobStore；通知job监听者，通知调度器线程，通知trigger监听者
        根据配置的jobStore策略存储JobDetail和Trigger，其中存储方式：
        RAMJobStore：将trigger和job存储在内存中，存取速度非常快，但是在系统被停止后所有的数据都会丢失,
        JobStoreSupport：将信息存储到数据库中，例如mysql、redis等；      
### Trigger：触发器，定义调度任务的时间规则
    用来定义Job（任务）触发条件、触发时间，触发间隔，终止时间等。
    1. 四大类型：
      SimpleTrigger：一种设置和使用简单的触发器，在指定日期/时间且可能需要重复执行n次的时机下使用的。这种触发器不适合每天定时执行任务这种场景。
	    CronTrigger：可以设定复杂的触发时间表，设置方式简单，就是需要编写cron表达式
	    DateIntervalTrigger：1.7版本后加入的触发器，该触发器适用于每小时，每几周，每几月重复执行的任务
	    NthIncludedDayTrigger：适用于在每一个间隔类型(月或年等)的第N天触发；
    2. Trigger诸类保存了trigger所有属性：
       name:	所有trigger通用	trigger名称;
       group:	所有trigger通用	trigger所属的组名;
       description:	所有trigger通用	trigger描述;
       calendarName:	所有trigger通用	日历名称，指定使用哪个Calendar类;
       misfireInstruction:	所有trigger通用	错过job（未在指定时间执行的job）的处理策略;
       priority:	所有trigger通用	优先级，默认为5。当多个trigger同时触发job时，线程池可能不够用，此时根据优先级来决定谁先触发;
       jobDataMap:	所有trigger通用	同job的jobDataMap,假如job和trigger的jobDataMap有同名key，通过getMergedJobDataMap()获取的jobDataMap，将以trigger的为准
       startTime:	所有trigger通用	触发开始时间，默认为当前时间。决定什么时间开始触发job
       endTime:	所有trigger通用	触发结束时间。决定什么时间停止触发job;
       nextFireTime：某种Trigger私有	下一次触发job的时间
       previousFireTime：	某种Trigger私有	上一次触发job的时间;
    3. 一个触发器只能绑定一个Job，但是一个Job可以有多个触发器
### Scheduler：调度器，启动Trigger去执行Job
    1. Scheduler由scheduler工厂创建：DirectSchedulerFactory 或者 StdSchedulerFactory。通常使用第二种工厂，因为第一种使用不方便且需要详细的编码设置，
       主要包括三种Scheduler：RemoteMBeanScheduler， RemoteScheduler 和 StdScheduler
    2. 详细配置属性，见：https://www.jianshu.com/p/2a5d3b6336ba
    3. org.quartz.impl.StdSchedulerFactory#getScheduler()，生成Scheduler实例。
      3.1 读取quartz配置文件，解析内容和环境变量，存入PropertiesParser对象中；
      3.2 获取调度池(维护了一个hashMap)，单例模式,从调度池中获取当前配置所用的调度器；
      3.3 如果调度池中没有当前配置的调度器，则实例化一个调度器，主要包括以下内容：
           1）初始化threadPool(线程池)：开发者可以通过org.quartz.threadPool.class配置指定使用哪个线程池类，比如SimpleThreadPool。
         先class load线程池类，接着动态生成线程池实例bean，然后通过反射，使用setXXX()方法将以org.quartz.threadPool开头的配置内容赋值给bean成员变量；
           2）初始化jobStore(任务存储方式)：开发者可以通过org.quartz.jobStore.class配置指定使用哪个任务存储类，比如RAMJobStore。
        先class load任务存储类，接着动态生成实例bean，然后通过反射，使用setXXX()方法将以org.quartz.jobStore开头的配置内容赋值给bean成员变量；
           3）初始化dataSource(数据源)：开发者可以通过org.quartz.dataSource配置指定数据源详情，比如哪个数据库、账号、密码等。jobStore要指定为JDBCJobStore，dataSource才会有效；
           4）初始化其他配置：包括SchedulerPlugins、JobListeners、TriggerListeners等；
           5）初始化threadExecutor(线程执行器)：默认为DefaultThreadExecutor；
           6）创建工作线程：根据配置创建N个工作thread，执行start()启动thread，并将N个thread顺序add进threadPool实例的空闲线程列表availWorkers中；
           7）创建调度器线程：创建QuartzSchedulerThread实例(paused=true,halted=false)，并通过threadExecutor.execute(实例)启动调度器线程；
           8）创建调度器：创建StdScheduler实例，将上面所有配置和引用组合进实例中，并将实例存入调度器池中.
    4. org.quartz.impl.StdScheduler#start，进行线程的启动，并执行相应的trigger。实际上是调用org.quartz.core.QuartzScheduler#start:
     4.1 调用JobStore的schedulerStarted()方法进行启动 ，若是集群，初始化集群管理线程JobStoreSupport.ClusterManager，非集群，需要有恢复机制，恢复任何失败或misfire的作业；
         初始化JobStoreSupport.MisfireHandler线程
     4.2 调用QuartzSchedulerThread#togglePause(paused=false)唤醒所有等待的线程；
     4.3 org.quartz.core.QuartzSchedulerThread#run，调度器线程一旦启动，将一直运行，在有可用线程的时候获取需要执行Trigger并出触发进行任务的调度；
     	1)  while()无限循环，每次循环取出时间将到的trigger，触发对应的job，直到调度器线程被关闭;
      	2)  同步加锁，循环等待，进入休眠状态，直到QuartzScheduler.start()调用了togglePause(false)唤醒线程；
	    3) 如果线程池中可用线程数大于0，获取马上到时间的trigger，且取出的trigger数不能超过一个阈值，并设置触发器状态为正在执行；
	    4) 通过JobRunShell，实例化正在执行的job，org.quartz.simpl.SimpleJobFactory#newJob，并放到线程池中运行，org.quartz.core.JobRunShell#run，最终执行Job接口中的execute方法
quartz中避免GC的方式：
    在类中创建一个ArrayList<Object>列表，如果某对象被添加进该队列，则意味着该类的实例引用了此对象，那么此对象至少在类实例存活时不会被GC。
	
## 2. 应用示例
### 2.1 引入相关pom
	<dependency>
    <groupId>org.quartz-scheduler</groupId>
    <artifactId>quartz</artifactId>
    <version>2.3.0</version>
	</dependency>
### 2.2 使用SchedulerFactoryBean 创建Scheduler
	以更具Bean风格的方式为Scheduler提供配置信息；
	让Scheduler和Spring容器的生命周期建立关联；
	通过属性配置部分或全部代替Quartz自身的配置文件。
	
	@Configuration
	@Slf4j
	public class QuartzConfig implements InitializingBean {

    @Value("${scheduler.pool.redis.cluster}")
    private String redisHost;

    @Value("${scheduler.pool.thread.count}")
    private Integer threadCount;

    @Value("${scheduler.pool.size}")
    private Integer poolSize;

    private static final String SCHEDULER_KEY_PRE = "xxx";


    //任务调度池,存储多个scheduler
    private static final Map<String, Scheduler> SCHEDULER_MAP = new HashMap<>(16);

    @Bean(name = "myJobFactory")
    public JobFactory jobFactoryBean() {
        return new MyJobFactory();
    }

    /**
     * 配置SchedulerFactoryBean,自动注入bean
     *
     * @param keyPrefix
     * @param jobFactory
     * @return
     * @throws Exception
     */
    public SchedulerFactoryBean schedulerFactoryBean(String keyPrefix, JobFactory jobFactory) throws Exception {
        SchedulerFactoryBean factory = new SchedulerFactoryBean();
        Properties quartzProperties = new Properties();
        //此处采用开源的redis存储JobDetail和Trigger
        quartzProperties.put("org.quartz.jobStore.class", "net.joelinn.quartz.jobstore.RedisJobStore");
        quartzProperties.put("org.quartz.jobStore.host", redisHost);
        quartzProperties.put("org.quartz.jobStore.redisCluster", "true");
        quartzProperties.put("org.quartz.jobStore.keyPrefix", keyPrefix);
        quartzProperties.put("org.quartz.threadPool.class", org.quartz.simpl.SimpleThreadPool.class.getName());
        quartzProperties.put("org.quartz.jobStore.misfireThreshold", "600");
        quartzProperties.put("org.quartz.threadPool.threadCount", threadCount);
        factory.setQuartzProperties(quartzProperties);
        factory.setJobFactory(jobFactory);
        factory.afterPropertiesSet();
        return factory;
    }

    public static Scheduler getRandomScheduler(String uid) {
        if (StringUtils.isEmpty(uid)) {
            log.error("scheduler uid is null");
            return null;
        }
        if (SCHEDULER_MAP.isEmpty()||SCHEDULER_MAP==null) {
            log.error("scheduler map is not init , data is empty");
            return null;
        }
        // 将调度器打散，随机获取其中某一个调度器，注意打散策略
        int index = Math.abs(MD5Utils.digest(uid).hashCode()) % SCHEDULER_MAP.size();
        String schedulerKey = generateSchedulerKey(index);
        return SCHEDULER_MAP.get(schedulerKey);
    }

    /**
     *  实现了InitializingBean接口，重写SchedulerFactoryBean中的afterPropertiesSet()方法
     *  生成自定义数目的scheduler
     * @throws Exception
     */
    @Override
    public void afterPropertiesSet() throws Exception {
        JobFactory jobFactory = new MyJobFactory();
        for (int i = 0; i < poolSize; i++) {
            String keyPrefix = generateSchedulerKey(i);
            Scheduler scheduler = schedulerFactoryBean(keyPrefix, jobFactory).getScheduler();
            scheduler.start();
            SCHEDULER_MAP.put(keyPrefix, scheduler);
            log.info("scheduler:[key pref={}] init succ", keyPrefix);
        }
        addShutdownHook();

        log.info("scheduler map init succ,size:{}", poolSize);
    }

    /**
     * 当前的scheduler还有未执行完的任务.将会挂起
     */
    private void addShutdownHook() {
        Runtime.getRuntime().addShutdownHook(new Thread(() -> {
            if (MapUtils.isEmpty(SCHEDULER_MAP)) {
                return;
            }

            for (Map.Entry<String, Scheduler> scheduleEntry : SCHEDULER_MAP.entrySet()) {
                try {
                    scheduleEntry.getValue().shutdown(true);
                } catch (SchedulerException e) {
                    log.error("scheduler pool shutdown error|{}", scheduleEntry.getKey(), e);
                }
                log.info("schedule[keyPref:{}] shutdown succ", scheduleEntry.getKey());
            }
        }, "schedulerHalt"));
    }

    private static String generateSchedulerKey(int i) {
        return SCHEDULER_KEY_PRE + (i == 0 ? StringUtils.EMPTY : String.valueOf(i)) + "_";
    }
}

   

   //自定义JobFactory，
	
   @Component  
   public class MyJobFactory extends AdaptableJobFactory { 
	
   	private static ConcurrentHashMap<String, Object> JOB_INSTANCE = new ConcurrentHashMap<>(16);
  
	//将ApplicationContext之外的一些instance实例加入到Spring Application上下文中
    @Autowired    
    private AutowireCapableBeanFactory capableBeanFactory;    
  
   	 @Override    
     protected Object createJobInstance(TriggerFiredBundle bundle) throws Exception {    
        //获取当前jobInstance实例
		String instanceKey = bundle.getJobDetail().getJobClass().getSimpleName();
        Object jobInstance = JOB_INSTANCE.get(instanceKey);
        if(jobInstance==null){	
				jobInstance = super.createJobInstance(bundle);    
        		capableBeanFactory.autowireBean(jobInstance);
				JOB_INSTANCE.put(instanceKey,jobInstance);
		}   
        	return jobInstance;    
    	}
   }
	
	//自定义Job
	
	public class MyJob implements Job {
		@Override
    	public void execute(JobExecutionContext context){
			JobDataMap dataMap = context.getMergedJobDataMap();
			//具体的业务逻辑
			//或使用Scheduler重规划任务
			Scheduler scheduler = context.getScheduler();
            JobDetail jobDetail = context.getJobDetail();
            Trigger trigger = context.getTrigger();
			//具体的业务逻辑
		}
	}
	
	//自定义trigger并触发任务
	
	public class MyTrigger{
	   private static final String TRIGGER_GROUP = "xxx-trigger";
	   private static final String TRIGGER_NAME="tg_";
	   private static final String JOB_NAME="job_"
	   public void handle(String uid,String data ){
		   Scheduler triggerScheduler = SchedulerPoolConfig.getRandomScheduler(uid);
		   TriggerKey triggerKey = new TriggerKey(TRIGGER_NAME+data,TRIGGER_GROUP);
		   Trigger quartzTrigger = TriggerBuilder
                    .newTrigger()
                    .withIdentity(triggerKey)
                    .withSchedule(TriggerUtils.buildShceduleCron())
                    .usingJobData(xxx, data))
                    .build();
		  if (!triggerScheduler.checkExists(triggerKey)) {
                JobDetail quartzJob = JobBuilder.newJob(MyJob.class)
                        .withIdentity(JOB_NAME+ data, TRIGGER_GROUP)
                        .build();
                triggerScheduler.scheduleJob(quartzJob, quartzTrigger);
		}	
	}
	
	//自定义cron生成工具类
	public class TriggerUtils{
		public static CronScheduleBuilder buildShceduleCron() {
			long startTime=System.currentTimeMillis();
			int plusHours=2;
        	LocalDateTime ldt;
        	try {
            	ldt = LocalDateTime.
                    	 ofInstant(Instant.ofEpochMilli(startTime), ZoneId.systemDefault())
                    	.plusHours(plusHours);
        	} catch (Exception e) {
            	logger.error("failed to parse local date time. startTime:[{}]", startTime);
            	ldt = LocalDateTime.now();
        	}

        	String year = String.valueOf(ldt.getYear());
        	String month = String.valueOf(ldt.getMonthValue());
        	String day = String.valueOf(ldt.getDayOfMonth());
        	String hour = String.valueOf(ldt.getHour());
        	String minute = String.valueOf(ldt.getMinute());
        	String cronExpression = CronExpressionGenerator.Builder.aCronExpression()
                .withYear(year)
                .withMonth(month)
                .withDayOfMonth(day)
                .withHour(hour)
                .withMinite(minute)
                .build().generate();
        	return CronScheduleBuilder.cronSchedule(cronExpression).withMisfireHandlingInstructionDoNothing();
    	}
	
		private CronTrigger createTrigger(final String cron) {
        	return CronScheduleBuilder.cronSchedule(cron).withMisfireHandlingInstructionDoNothing();
    	}
   }
	
	//cron生成类转换
	public class CronExpressionGenerator {
    	private static final String DELIMITER = " ";
    	private String second;
    	private String minite;
    	private String hour;
    	private String dayOfMonth;
    	private String month;
    	private String dayOfWeek;
    	private String year;

    public String generate() {
        return String.join(DELIMITER, new String[]{second, minite, hour, dayOfMonth, month, dayOfWeek, year});
    }

    public static final class Builder {
        private String second = "0";
        private String minite = "0";
        private String hour = "0";
        private String dayOfMonth = "?";
        private String month = "*";
        private String dayOfWeek = "?";
        private String year = "*";

        private Builder() {
        }

        public static Builder aCronExpression() {
            return new Builder();
        }

        public Builder withSecond(String second) {
            this.second = second;
            return this;
        }

        public Builder withMinite(String minite) {
            this.minite = minite;
            return this;
        }

        public Builder withHour(String hour) {
            this.hour = hour;
            return this;
        }

        public Builder withDayOfMonth(String dayOfMonth) {
            this.dayOfMonth = dayOfMonth;
            return this;
        }

        public Builder withMonth(String month) {
            this.month = month;
            return this;
        }

        public Builder withDayOfWeek(String dayOfWeek) {
            this.dayOfWeek = dayOfWeek;
            return this;
        }

        public Builder withYear(String year) {
            this.year = year;
            return this;
        }

        public CronExpressionGenerator build() {
            CronExpressionGenerator cronExpression = new CronExpressionGenerator();
            cronExpression.hour = this.hour;
            cronExpression.dayOfMonth = this.dayOfMonth;
            cronExpression.minite = this.minite;
            cronExpression.dayOfWeek = this.dayOfWeek;
            cronExpression.second = this.second;
            cronExpression.month = this.month;
            cronExpression.year = this.year;
            return cronExpression;
        }
    }
 }

## 2. JobStore-集群方式
### 2.1 JDBC的方式
	数据库表：https://github.com/quartz-scheduler/quartz/blob/master/quartz-core/src/main/resources/org/quartz/impl/jdbcjobstore/tables_mysql_innodb.sql
	数据库表的详解：https://www.jianshu.com/p/b94ebb8780fa
	属性配置：
		#同一集群中应用采用相同的Scheduler实例
		org.quartz.scheduler.instanceName: wenqyScheduler

		#集群节点的ID必须唯一，可由quartz自动生成
		org.quartz.scheduler.instanceId: AUTO

		#通知Scheduler实例要它参与到一个集群当中
		org.quartz.jobStore.isClustered: true

		#需持久化存储
		org.quartz.jobStore.class=org.quartz.impl.jdbcjobstore.JobStoreTX
		org.quartz.jobStore.driverDelegateClass=org.quartz.impl.jdbcjobstore.StdJDBCDelegate

		#数据源
		org.quartz.jobStore.dataSource=myDS

		#quartz表前缀
		org.quartz.jobStore.tablePrefix=QRTZ_

		#数据源配置
		org.quartz.dataSource.myDS.driver: com.mysql.jdbc.Driver
		org.quartz.dataSource.myDS.URL: jdbc:mysql://localhost:3306/ncdb
		org.quartz.dataSource.myDS.user: root
		org.quartz.dataSource.myDS.password: 123456
		org.quartz.dataSource.myDS.maxConnections: 5
		org.quartz.dataSource.myDS.validationQuery: select 0
	具体原理：
		![image](https://user-images.githubusercontent.com/41152743/140027174-6a155141-1976-40d3-9e4e-3dcba30fcaab.png)

	    1. org.quartz.impl.jdbcjobstore.JobStoreSupport#acquireNextTrigger
		   1）从QRTZ_LOCKS表中，获取数据库锁，取到行级锁TRIGGER_ACCESS,执行成功后,数据库对该行锁定。
				另外一个线程使用相同的SQL对表的数据进行查询，只能等待，直到该行锁的线程完成了相关的操作。保证了同一个集群下，只有一个quartz实例获取需要执行的trigger
	       2）从QRTZ_TRIGGER表中，获取30s内且触发状态为WAITTING的Trigger，并按优先级排序；
	       3）根据Trigger的JobKey从qrtz_job_details表中获取详细的job信息；
		   4）在QRTZ_TRIGGER表中修改Trigger的状态为ACQUIRED，
		   5）然后将待触发的Trigger插入到qrtz_fired_triggers中
	       6）提交获取Trigger的事务，行锁被释放，返回待触发的Trigeer列表；
	
	    2. org.quartz.impl.jdbcjobstore.JobStoreSupport#triggersFired
		   通知JobStore触发trigger，获取数据库锁，从QRTZ_LOCKS表中获取STATE_ACCESS行级锁;
		   从QRTZ_TRIGGER表中确认trigger的状态；
	       从qrtz_job_details表中获取job信息；
	       从qrtz_calendars表中获取trigger的calendar信息；
	       从qrtz_fired_triggers表中更新trigger的状态为EXECUTING；
	       在QRTZ_TRIGGER表中根据下次触发时间、是否允许并发等信息更新trigger的状态信息，持久化Trigger，并更新下次触发时间，
	       最后提交触发trigger的事务，行锁被释放，返回待执行的JobDetail信息；
	
	   3. 集群故障转移 org.quartz.impl.jdbcjobstore.JobStoreSupport#doCheckin
			每个服务器会定时（org.quartz.jobStore.clusterCheckinInterval这个时间）更新SCHEDULER_STATE表中的LAST_CHECK_TIME，检测集群的scheduler的实例状态信息；
			如果发现服务超时，则会尝试接替该服务为完成的作业，在qrtz_fired_triggers中更新触发器的状态，然后从qrtz_scheduler_state表中删除故障节点。
			故障节点触发器更新前状态	更新后状态
				BLOCK	                WAITING
                PAUSED_BLOCK        	PAUSED
                ACQUIRED	           WAITING
                COMPLETE	         无，删除Trigger
## 3. misfire机制
### 3.1 CronTrigger的misfire机制
	1. withMisfireHandlingInstructionDoNothing() -> misfireInstruction = 2
	不触发立即执行，等待下次Cron触发频率到达时刻开始按照cron频率依次执行；
	2. withMisfireHandlingInstructionFireAndProceed -> misfireInstruction = 1
	以当前时间为触发频率立刻触发一次执行，然后按照cron频率依次执行；
	3. withMisfireHandlingInstructionIgnoreMisfires -> misfireInstruction = -1
	以错过的第一个频率时间立刻开始执行，重做错过的所有频率周期后，当下一次触发频率发生时间大于当前时间后，按照正常的cron频率依次执行；
### 3.2 SimpleTrigger的misfire机制
	withMisfireHandlingInstructionFireNow
		——以当前时间为触发频率立即触发执行
		——执行至FinalTIme的剩余周期次数
		——以调度或恢复调度的时刻为基准的周期频率，FinalTime根据剩余次数和当前时间计算得到
		——调整后的FinalTime会略大于根据starttime计算的到的FinalTime值

	withMisfireHandlingInstructionIgnoreMisfires
		——以错过的第一个频率时间立刻开始执行
		——重做错过的所有频率周期
		——当下一次触发频率发生时间大于当前时间以后，按照Interval的依次执行剩下的频率
		——共执行RepeatCount+1次

	withMisfireHandlingInstructionNextWithExistingCount
		——不触发立即执行
		——等待下次触发频率周期时刻，执行至FinalTime的剩余周期次数
		——以startTime为基准计算周期频率，并得到FinalTime
		——即使中间出现pause，resume以后保持FinalTime时间不变


	withMisfireHandlingInstructionNowWithExistingCount
		——以当前时间为触发频率立即触发执行
		——执行至FinalTIme的剩余周期次数
		——以调度或恢复调度的时刻为基准的周期频率，FinalTime根据剩余次数和当前时间计算得到
		——调整后的FinalTime会略大于根据starttime计算的到的FinalTime值

	withMisfireHandlingInstructionNextWithRemainingCount
		——不触发立即执行
		——等待下次触发频率周期时刻，执行至FinalTime的剩余周期次数
		——以startTime为基准计算周期频率，并得到FinalTime
		——即使中间出现pause，resume以后保持FinalTime时间不变

	withMisfireHandlingInstructionNowWithRemainingCount
		——以当前时间为触发频率立即触发执行
		——执行至FinalTIme的剩余周期次数
		——以调度或恢复调度的时刻为基准的周期频率，FinalTime根据剩余次数和当前时间计算得到
		——调整后的FinalTime会略大于根据starttime计算的到的FinalTime值

	MISFIRE_INSTRUCTION_RESCHEDULE_NOW_WITH_REMAINING_REPEAT_COUNT
		——此指令导致trigger忘记原始设置的starttime和repeat-count
		——触发器的repeat-count将被设置为剩余的次数
		——这样会导致后面无法获得原始设定的starttime和repeat-count值
