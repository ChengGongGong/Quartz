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
public class SchedulerPoolConfig implements InitializingBean {

    private static final Logger LOGGER = LoggerFactory.getLogger(SchedulerPoolConfig.class);

    @Value("${scheduler.pool.redis.cluster}")
    private String redisHost;

    @Value("${scheduler.pool.thread.count}")
    private String threadCount;

    @Value("${scheduler.pool.size}")
    private Integer poolSize;

    private static final String SCHEDULER_KEY_PRE = "xxx";

    //任务调度池,存储多个scheduler
    private static final Map<String, Scheduler> SCHEDULER_MAP = new HashMap<>();

    private static int SCHEDULER_MAP_SIZE = 0;

    public static Scheduler getRandomScheduler(String uid) {
        if (StringUtils.isEmpty(uid)) {
            LOGGER.error("scheduler uid is null");
            return null;
        }
        if (MapUtils.isEmpty(SCHEDULER_MAP)) {
            LOGGER.error("scheduler map is not init , data is empty");
            return null;
        }
        // 将调度器打散，随机获取其中某一个调度器，注意打散策略
        int index = Math.abs(MD5Utils.digest(uid).hashCode()) % SCHEDULER_MAP_SIZE;
        String schedulerKey = generateSchedulerKey(index);
        return SCHEDULER_MAP.get(schedulerKey);
    }

	//配置ShedulerFactoryBean
    public SchedulerFactoryBean schedulerFactoryBean(String keyPrefix, JobFactory jobFactory) throws Exception {
        SchedulerFactoryBean factory = new SchedulerFactoryBean();
        Properties quartzProperties = new Properties();
		//此处采用开源的redis存储JobDetail和Trigger
        quartzProperties.put("org.quartz.jobStore.class", "net.joelinn.quartz.jobstore.RedisJobStore");
        quartzProperties.put("org.quartz.jobStore.host", redisHost);
        quartzProperties.put("org.quartz.jobStore.redisCluster", "true");
        quartzProperties.put("org.quartz.jobStore.keyPrefix", keyPrefix);
        quartzProperties.put("org.quartz.threadPool.class", "org.quartz.simpl.SimpleThreadPool");
        quartzProperties.put("org.quartz.jobStore.misfireThreshold", "600");
        quartzProperties.put("org.quartz.threadPool.threadCount", threadCount);
        factory.setQuartzProperties(quartzProperties);
        factory.setJobFactory(jobFactory);
        factory.afterPropertiesSet();
        return factory;
    }

	//实现了InitializingBean接口，重写SchedulerFactoryBean中的afterPropertiesSet()方法
    @Override
    public void afterPropertiesSet() throws Exception {
        JobFactory jobFactory = new MyJobFactory();
		//自定义scheduler的数目
        for (int i = 0; i < poolSize; i++) {
            String keyPrefix = generateSchedulerKey(i);
            Scheduler scheduler = schedulerFactoryBean(keyPrefix, jobFactory).getScheduler();
            scheduler.start();
            SCHEDULER_MAP.put(keyPrefix, scheduler);
            LOGGER.info("scheduler:[key pref={}] init succ", keyPrefix);
        }
        SCHEDULER_MAP_SIZE = SCHEDULER_MAP.size();
        addShutdownHook();

        LOGGER.info("scheduler map init succ,size:{}", size);
    }

    private void addShutdownHook() {
        Runtime.getRuntime().addShutdownHook(new Thread(() -> {
            LOGGER.info("scheduler pool shutdown begin...");
            if (MapUtils.isEmpty(SCHEDULER_MAP)) {
                return;
            }

            for (Map.Entry<String, Scheduler> scheduleEntry : SCHEDULER_MAP.entrySet()) {
                try {
                    scheduleEntry.getValue().shutdown(true);
                } catch (SchedulerException e) {
                    LOGGER.error("scheduler pool shutdown error|{}", scheduleEntry.getKey(), e);
                }
                LOGGER.info("schedule[keyPref:{}] shutdown succ", scheduleEntry.getKey());
            }
            LOGGER.info("scheduler pool shutdown end.");
        }, "SchedulerPoolShutdownHook"));
    }

    private static String generateSchedulerKey(int i) {
        return SCHEDULER_KEY_PRE + (i == 0 ? StringUtils.EMPTY : String.valueOf(i)) + "_";
    }
	
	@Bean(name = "myJobFactory")
    public JobFactory jobFactoryBean {
        return new MyJobFactory();
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
}  
