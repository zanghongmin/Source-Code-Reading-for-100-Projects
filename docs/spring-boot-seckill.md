* [一、项目概览](#一数据类型)
    * [1.1 简介](#11-简介) 
    * [1.2 环境](#12-环境)
    * [1.3 源码及官网](#13-源码及官网)
* [二、项目使用](#二项目使用)
* [三、项目设计](#三项目设计)
    * [3.1 总体设计](#31-总体设计)
    * [3.2 关键点分析](#32-关键点分析)
        * [3.2.1 单机秒杀方案分析](#321-单机秒杀方案分析)  
        * [3.2.2 分布式秒杀方案分析](#322-分布式秒杀方案分析)  
        * [3.2.3 使用WebSocket技术分析](#323-使用WebSocket技术分析)  
* [四、其他](#四其他)

# 一、项目概览

## 1.1 简介
- spring-boot-seckill是分布式秒杀的测试案例，单纯为了学习，某些案例并不适用于生产环境，大家根据所需自行调整。

## 1.2 环境
```aidl
JDK1.7、Maven、Mysql、Eclipse、SpringBoot1.5.10
zookeeper3.4.6、kafka_2.11、redis-2.8.4、curator-2.10.0
```
## 1.3 源码及官网

[gitee源码](https://gitee.com/52itstyle/spring-boot-seckill)

# 二、项目使用


# 三、项目设计

## 3.1 总体设计
无

## 3.2 关键点分析

### 3.2.1 单机秒杀方案分析

- 秒杀一(超卖) 最简单的查询更新操作，不涉及各种锁，会出现超卖情况。 
```aidl
    	@ApiOperation(value="秒杀一(最low实现)",nickname="科帮网")
    	@PostMapping("/start")
    	public Result start(long seckillId){
    		seckillService.deleteSeckill(seckillId);
    		final long killId =  seckillId;
    		LOGGER.info("开始秒杀一(会出现超卖)");
    		for(int i=0;i<1000;i++){
    			final long userId = i;
    			Runnable task = new Runnable() {
    				@Override
    				public void run() {
    					Result result = seckillService.startSeckil(killId, userId);
    					if(result!=null){
    						LOGGER.info("用户:{}{}",userId,result.get("msg"));
    					}else{
    						LOGGER.info("用户:{}{}",userId,"哎呦喂，人也太多了，请稍后！");
    					}
    				}
    			};
    			executor.execute(task);
    		}
    		try {
    			Thread.sleep(10000);
    			Long  seckillCount = seckillService.getSeckillCount(seckillId);
    			LOGGER.info("一共秒杀出{}件商品",seckillCount);
    		} catch (InterruptedException e) {
    			e.printStackTrace();
    		}
    		return Result.ok();
    	}
	
  	@Override
  	@ServiceLimit  //使用AOP+注解方式限流
  	@Transactional
  	public Result startSeckil(long seckillId,long userId) {
  		//校验库存
  		String nativeSql = "SELECT number FROM seckill WHERE seckill_id=?";
  		Object object =  dynamicQuery.nativeQueryObject(nativeSql, new Object[]{seckillId});
  		Long number =  ((Number) object).longValue();
  		if(number>0){  //多个线程同时到这里后，进行扣库存，会出现多扣的情况
  			//扣库存
  			nativeSql = "UPDATE seckill  SET number=number-1 WHERE seckill_id=?";
  			dynamicQuery.nativeExecuteUpdate(nativeSql, new Object[]{seckillId});
  			//创建订单
  			SuccessKilled killed = new SuccessKilled();
  			killed.setSeckillId(seckillId);
  			killed.setUserId(userId);
  			killed.setState((short)0);
  			killed.setCreateTime(new Timestamp(new Date().getTime()));
  			dynamicQuery.save(killed);
  			//支付
  			return Result.ok(SeckillStatEnum.SUCCESS);
  		}else{
  			return Result.error(SeckillStatEnum.END);
  		}
  	}  

    @Component
    @Scope
    @Aspect
    public class LimitAspect {
        ////每秒只发出5个令牌，此处是单进程服务的限流,内部采用令牌捅算法实现
        private static   RateLimiter rateLimiter = RateLimiter.create(5.0);
        
        //Service层切点  限流
        @Pointcut("@annotation(com.itstyle.seckill.common.aop.ServiceLimit)")  
        public void ServiceAspect() {
            
        }
        
        @Around("ServiceAspect()")
        public  Object around(ProceedingJoinPoint joinPoint) { 
            Boolean flag = rateLimiter.tryAcquire();
            Object obj = null;
            try {
                if(flag){
                    obj = joinPoint.proceed();
                }
            } catch (Throwable e) {
                e.printStackTrace();
            } 
            return obj;
        } 
    }


```


- 秒杀二(超卖) 使用ReentrantLock重入锁，由于事物提交和锁释放的先后顺序也会导致超卖

```aidl
	@ApiOperation(value="秒杀二(程序锁)",nickname="科帮网")
	@PostMapping("/startLock")
	public Result startLock(long seckillId){
		seckillService.deleteSeckill(seckillId);
		final long killId =  seckillId;
		LOGGER.info("开始秒杀二(正常)");
		for(int i=0;i<1000;i++){
			final long userId = i;
			Runnable task = new Runnable() {
				@Override
				public void run() {
					Result result = seckillService.startSeckilLock(killId, userId);
					LOGGER.info("用户:{}{}",userId,result.get("msg"));
				}
			};
			executor.execute(task);
		}
		try {
			Thread.sleep(10000);
			Long  seckillCount = seckillService.getSeckillCount(seckillId);
			LOGGER.info("一共秒杀出{}件商品",seckillCount);
		} catch (InterruptedException e) {
			e.printStackTrace();
		}
		return Result.ok();
	}
	
    @Override
    @Transactional //出现超卖的原因是释放锁在事务提交前
    public Result  startSeckilLock(long seckillId, long userId) {
         try {
            lock.lock();
            //这里、不清楚为啥、总是会被超卖101、难道锁不起作用、lock是同一个对象
            //来自热心网友 zoain 的细心测试思考、然后自己总结了一下
            //事物未提交之前，锁已经释放(事物提交是在整个方法执行完)，导致另一个事物读取到了这个事物未提交的数据，也就是传说中的脏读。建议锁上移
            //给自己留个坑思考：为什么分布式锁(zk和redis)没有问题？(事实是有问题的，由于redis释放锁需要远程通信，不那么明显而已)
            String nativeSql = "SELECT number FROM seckill WHERE seckill_id=?";
            Object object =  dynamicQuery.nativeQueryObject(nativeSql, new Object[]{seckillId});
            Long number =  ((Number) object).longValue();
            if(number>0){
                nativeSql = "UPDATE seckill  SET number=number-1 WHERE seckill_id=?";
                dynamicQuery.nativeExecuteUpdate(nativeSql, new Object[]{seckillId});
                SuccessKilled killed = new SuccessKilled();
                killed.setSeckillId(seckillId);
                killed.setUserId(userId);
                killed.setState(Short.parseShort(number+""));
                killed.setCreateTime(new Timestamp(new Date().getTime()));
                dynamicQuery.save(killed);
            }else{
                return Result.error(SeckillStatEnum.END);
            }
        } catch (Exception e) {
            e.printStackTrace();
        }finally {
            lock.unlock();
        }
        return Result.ok(SeckillStatEnum.SUCCESS);
    }
    
    
```

- 秒杀三(正常) 基于秒杀二场景的修复，锁上移，事物提交后再释放锁，不会超卖。
```aidl

	@Override
	@Servicelock //使用AOP+注解方式，先处理事物提交后再释放锁
	@Transactional
	public Result startSeckilAopLock(long seckillId, long userId) {
		//来自码云码友<马丁的早晨>的建议 使用AOP + 锁实现
		String nativeSql = "SELECT number FROM seckill WHERE seckill_id=?";
		Object object =  dynamicQuery.nativeQueryObject(nativeSql, new Object[]{seckillId});
		Long number =  ((Number) object).longValue();
		if(number>0){
			nativeSql = "UPDATE seckill  SET number=number-1 WHERE seckill_id=?";
			dynamicQuery.nativeExecuteUpdate(nativeSql, new Object[]{seckillId});
			SuccessKilled killed = new SuccessKilled();
			killed.setSeckillId(seckillId);
			killed.setUserId(userId);
			killed.setState(Short.parseShort(number+""));
			killed.setCreateTime(new Timestamp(new Date().getTime()));
			dynamicQuery.save(killed);
		}else{
			return Result.error(SeckillStatEnum.END);
		}
		return Result.ok(SeckillStatEnum.SUCCESS);
	}
	
	@Component
    @Scope
    @Aspect
    @Order(1)
    //order越小越是最先执行，但更重要的是最先执行的最后结束。order默认值是2147483647
    public class LockAspect {
    	/**
         * 思考：为什么不用synchronized
         * service 默认是单例的，并发下lock只有一个实例
         */
    	private static  Lock lock = new ReentrantLock(true);//互斥锁 参数默认false，不公平锁      	
    	//Service层切点     用于记录错误日志
    	@Pointcut("@annotation(com.itstyle.seckill.common.aop.Servicelock)")  
    	public void lockAspect() {}
   	
        @Around("lockAspect()")
        public  Object around(ProceedingJoinPoint joinPoint) { 
        	lock.lock();
        	Object obj = null;
    		try {
    			obj = joinPoint.proceed();
    		} catch (Throwable e) {
    			e.printStackTrace();
    		} finally{
    			lock.unlock();
    		}
        	return obj;
        } 
    }
```


- 秒杀四(少买) 基于数据库悲观锁实现，查询加锁，然后更新，由于使用了 限流 注解(可自行注释)，这里会出现少买。如果想正常，去掉startSeckil方法上的@ServiceLimit注解即可。
```aidl

	//注意这里 限流注解 可能会出现少买 自行调整
	@Override
	@ServiceLimit
	@Transactional
	public Result startSeckilDBPCC_ONE(long seckillId, long userId) {
		//单用户抢购一件商品或者多件都没有问题
		String nativeSql = "SELECT number FROM seckill WHERE seckill_id=? FOR UPDATE";
		Object object =  dynamicQuery.nativeQueryObject(nativeSql, new Object[]{seckillId});
		Long number =  ((Number) object).longValue();
		if(number>0){
			nativeSql = "UPDATE seckill  SET number=number-1 WHERE seckill_id=?";
			dynamicQuery.nativeExecuteUpdate(nativeSql, new Object[]{seckillId});
			SuccessKilled killed = new SuccessKilled();
			killed.setSeckillId(seckillId);
			killed.setUserId(userId);
			killed.setState((short)0);
			killed.setCreateTime(new Timestamp(new Date().getTime()));
			dynamicQuery.save(killed);
			return Result.ok(SeckillStatEnum.SUCCESS);
		}else{
			return Result.error(SeckillStatEnum.END);
		}
	}

```

- 秒杀五(正常) 基于数据库悲观锁实现，更新加锁并判断剩余数量
```aidl
    /**
     * SHOW STATUS LIKE 'innodb_row_lock%'; 
     * 如果发现锁争用比较严重，如InnoDB_row_lock_waits和InnoDB_row_lock_time_avg的值比较高
     */
	@Override
	@Transactional
	public Result startSeckilDBPCC_TWO(long seckillId, long userId) {
		//单用户抢购一件商品没有问题、但是抢购多件商品不建议这种写法
		String nativeSql = "UPDATE seckill  SET number=number-1 WHERE seckill_id=? AND number>0";//UPDATE锁表
		int count = dynamicQuery.nativeExecuteUpdate(nativeSql, new Object[]{seckillId});
		if(count>0){
			SuccessKilled killed = new SuccessKilled();
			killed.setSeckillId(seckillId);
			killed.setUserId(userId);
			killed.setState((short)0);
			killed.setCreateTime(new Timestamp(new Date().getTime()));
			dynamicQuery.save(killed);
			return Result.ok(SeckillStatEnum.SUCCESS);
		}else{
			return Result.error(SeckillStatEnum.END);
		}
	}

```
- 秒杀六(正常) 基于数据库乐观锁实现，先查询商品版本号，然后根据版本号更新，判断更新数量。少量用户抢购的时候会出现 少买 的情况。
```aidl
    //这里使用的乐观锁、可以自定义抢购数量、如果配置的抢购人数比较少、比如120:100(人数:商品) 会出现少买的情况
    //用户同时进入会出现更新失败的情况
	@Override
	@Transactional
	public Result startSeckilDBOCC(long seckillId, long userId, long number) {
		Seckill kill = seckillRepository.findOne(seckillId);
		//if(kill.getNumber()>0){
		if(kill.getNumber()>=number){//剩余的数量应该要大于等于秒杀的数量
			//乐观锁
			String nativeSql = "UPDATE seckill  SET number=number-?,version=version+1 WHERE seckill_id=? AND version = ?";
			int count = dynamicQuery.nativeExecuteUpdate(nativeSql, new Object[]{number,seckillId,kill.getVersion()});
			if(count>0){
				SuccessKilled killed = new SuccessKilled();
				killed.setSeckillId(seckillId);
				killed.setUserId(userId);
				killed.setState((short)0);
				killed.setCreateTime(new Timestamp(new Date().getTime()));
				dynamicQuery.save(killed);
				return Result.ok(SeckillStatEnum.SUCCESS);
			}else{
				return Result.error(SeckillStatEnum.END);
			}
		}else{
			return Result.error(SeckillStatEnum.END);
		}
	}
```

- 秒杀七 基于进程内队列 LinkedBlockingQueue 实现，同步消费。
```aidl
    @ApiOperation(value="秒杀柒(进程内队列)",nickname="科帮网")
	@PostMapping("/startQueue")
	public Result startQueue(long seckillId){
		seckillService.deleteSeckill(seckillId);
		final long killId =  seckillId;
		LOGGER.info("开始秒杀柒(正常)");
		for(int i=0;i<1000;i++){
			final long userId = i;
			Runnable task = new Runnable() {
				@Override
				public void run() {
					SuccessKilled kill = new SuccessKilled();
					kill.setSeckillId(killId);
					kill.setUserId(userId);
					try {
						Boolean flag = SeckillQueue.getMailQueue().produce(kill); //放入内部队列，
						if(flag){
							LOGGER.info("用户:{}{}",kill.getUserId(),"秒杀成功");
						}else{
							LOGGER.info("用户:{}{}",userId,"秒杀失败");
						}
					} catch (InterruptedException e) {
						e.printStackTrace();
						LOGGER.info("用户:{}{}",userId,"秒杀失败");
					}
				}
			};
			executor.execute(task);
		}
		try {
			Thread.sleep(10000);
			Long  seckillCount = seckillService.getSeckillCount(seckillId);
			LOGGER.info("一共秒杀出{}件商品",seckillCount);
		} catch (InterruptedException e) {
			e.printStackTrace();
		}
		return Result.ok();
	}
	
public class SeckillQueue {
	 //队列大小
    static final int QUEUE_MAX_SIZE   = 100;
    /** 用于多线程间下单的队列 */
    static BlockingQueue<SuccessKilled> blockingQueue = new LinkedBlockingQueue<SuccessKilled>(QUEUE_MAX_SIZE);   
    /**
     * 私有的默认构造子，保证外界无法直接实例化
     */
    private SeckillQueue(){};
    /**
     * 类级的内部类，也就是静态的成员式内部类，该内部类的实例与外部类的实例
     * 没有绑定关系，而且只有被调用到才会装载，从而实现了延迟加载
     */
    private static class SingletonHolder{
        /**
         * 静态初始化器，由JVM来保证线程安全
         */
		private  static SeckillQueue queue = new SeckillQueue();
    }
    //单例队列
    public static SeckillQueue getMailQueue(){
        return SingletonHolder.queue;
    }
    /**
     * 生产入队
     * @param kill
     * @throws InterruptedException
     * add(e) 队列未满时，返回true；队列满则抛出IllegalStateException(“Queue full”)异常——AbstractQueue 
     * put(e) 队列未满时，直接插入没有返回值；队列满时会阻塞等待，一直等到队列未满时再插入。
     * offer(e) 队列未满时，返回true；队列满时返回false。非阻塞立即返回。
     * offer(e, time, unit) 设定等待的时间，如果在指定时间内还不能往队列中插入数据则返回false，插入成功返回true。 
     */
    public  Boolean  produce(SuccessKilled kill) throws InterruptedException {
    	return blockingQueue.offer(kill);
    }
    /**
     * 消费出队
     * poll() 获取并移除队首元素，在指定的时间内去轮询队列看有没有首元素有则返回，否者超时后返回null
     * take() 与带超时时间的poll类似不同在于take时候如果当前队列空了它会一直等待其他线程调用notEmpty.signal()才会被唤醒
     */
    public  SuccessKilled consume() throws InterruptedException {
        return blockingQueue.take();
    }
    // 获取队列大小
    public int size() {
        return blockingQueue.size();
    }
}	
	
@Component
public class TaskRunner implements ApplicationRunner{
	
	@Autowired
	private ISeckillService seckillService;
	
	@Override
    public void run(ApplicationArguments var) throws Exception{
		while(true){
			//进程内队列
			SuccessKilled kill = SeckillQueue.getMailQueue().consume();
			if(kill!=null){
				seckillService.startSeckil(kill.getSeckillId(), kill.getUserId()); //一个一个的消费队列中内容，进行扣库存
			}
		}
    }
}	
```

- 秒杀八(少买) 基于高性能队列 Disruptor实现，同步消费，由于使用了 限流 注解(可自行注释)，这里会出现少买。如果想正常，去掉startSeckil方法上的@ServiceLimit注解即可。


### 3.2.2 分布式秒杀方案分析

- 秒杀一(超卖) 基于 Rediss 分布式锁 实现，由于锁和事物执行顺序问题，会出现超卖的情况(建议锁上移)。
```aidl
	@ApiOperation(value="秒杀一(Rediss分布式锁)",nickname="科帮网")
	@PostMapping("/startRedisLock")
	public Result startRedisLock(long seckillId){
		seckillService.deleteSeckill(seckillId);
		final long killId =  seckillId;
		LOGGER.info("开始秒杀一");
		for(int i=0;i<1000;i++){
			final long userId = i;
			Runnable task = new Runnable() {
				@Override
				public void run() {
					Result result = seckillDistributedService.startSeckilRedisLock(killId, userId);
					LOGGER.info("用户:{}{}",userId,result.get("msg"));
				}
			};
			executor.execute(task);
		}
		try {
			Thread.sleep(15000);
			Long  seckillCount = seckillService.getSeckillCount(seckillId);
			LOGGER.info("一共秒杀出{}件商品",seckillCount);
		} catch (InterruptedException e) {
			e.printStackTrace();
		}
		return Result.ok();
	}
    @Override
	@Transactional
	public Result startSeckilRedisLock(long seckillId,long userId) {
		boolean res=false;
		try {
			/**
			 * 尝试获取锁，最多等待3秒，上锁以后20秒自动解锁（实际项目中推荐这种，以防出现死锁）、这里根据预估秒杀人数，设定自动释放锁时间.
			 * 看过博客的朋友可能会知道(Lcok锁与事物冲突的问题)：https://blog.52itstyle.com/archives/2952/
			 * 分布式锁的使用和Lock锁的实现方式是一样的，但是测试了多次分布式锁就是没有问题，当时就留了个坑
			 * 闲来咨询了《静儿1986》，推荐下博客：https://www.cnblogs.com/xiexj/p/9119017.html
			 * 先说明下之前的配置情况：Mysql在本地，而Redis是在外网。
			 * 回复是这样的：
			 * 这是因为分布式锁的开销是很大的。要和锁的服务器进行通信，它虽然是先发起了锁释放命令，涉及网络IO，延时肯定会远远大于方法结束后的事务提交。
			 * ==========================================================================================
			 * 分布式锁内部都是Runtime.exe命令调用外部，肯定是异步的。分布式锁的释放只是发了一个锁释放命令就算完活了。真正其作用的是下次获取锁的时候，要确保上次是释放了的。
			 * 就是说获取锁的时候耗时比较长，那时候事务肯定提交了就是说获取锁的时候耗时比较长，那时候事务肯定提交了。
			 * ==========================================================================================
			 * 周末测试了一下，把redis配置在了本地，果然出现了超卖的情况；或者还是使用外网并发数增加在10000+也是会有问题的，之前自己没有细测，我的锅。
			 * 所以这钟实现也是错误的，事物和锁会有冲突，建议AOP实现。
			 */
			res = RedissLockUtil.tryLock(seckillId+"", TimeUnit.SECONDS, 3, 20);
			if(res){
				String nativeSql = "SELECT number FROM seckill WHERE seckill_id=?";
				Object object =  dynamicQuery.nativeQueryObject(nativeSql, new Object[]{seckillId});
				Long number =  ((Number) object).longValue();
				if(number>0){
					SuccessKilled killed = new SuccessKilled();
					killed.setSeckillId(seckillId);
					killed.setUserId(userId);
					killed.setState((short)0);
					killed.setCreateTime(new Timestamp(new Date().getTime()));
					dynamicQuery.save(killed);
					nativeSql = "UPDATE seckill  SET number=number-1 WHERE seckill_id=? AND number>0";
					dynamicQuery.nativeExecuteUpdate(nativeSql, new Object[]{seckillId});
				}else{
					return Result.error(SeckillStatEnum.END);
				}
			}else{
			    return Result.error(SeckillStatEnum.MUCH);
			}
		} catch (Exception e) {
			e.printStackTrace();
		} finally{
			if(res){//释放锁
				RedissLockUtil.unlock(seckillId+"");
			}
		}
		return Result.ok(SeckillStatEnum.SUCCESS);
	}
	
    /**
     * 尝试获取锁
     * @param lockKey
     * @param unit 时间单位
     * @param waitTime 最多等待时间
     * @param leaseTime 上锁后自动释放锁时间
     * @return
     */
    public static boolean tryLock(String lockKey, TimeUnit unit, int waitTime, int leaseTime) {
    	RLock lock = redissonClient.getLock(lockKey);
		try {
			return lock.tryLock(waitTime, leaseTime, unit);
		} catch (InterruptedException e) {
			return false;
		}
    }		

    /**
     * 单机模式自动装配
     * @return
     */
    @Bean
    @ConditionalOnProperty(name="redisson.address")
    RedissonClient redissonSingle() {
        Config config = new Config();
        SingleServerConfig serverConfig = config.useSingleServer()
                .setAddress(redssionProperties.getAddress())
                .setTimeout(redssionProperties.getTimeout())
                .setConnectionPoolSize(redssionProperties.getConnectionPoolSize())
                .setConnectionMinimumIdleSize(redssionProperties.getConnectionMinimumIdleSize());
        if(StringUtils.isNotBlank(redssionProperties.getPassword())) {
            serverConfig.setPassword(redssionProperties.getPassword());
        }

        return Redisson.create(config);
    }

    /**
     * 装配locker类，并将实例注入到RedissLockUtil中
     * @return
     */
    @Bean
    RedissLockUtil redissLockUtil(RedissonClient redissonClient) {
    	RedissLockUtil redissLockUtil = new RedissLockUtil();
    	redissLockUtil.setRedissonClient(redissonClient);
		return redissLockUtil;
    }	
    
```
- 秒杀二(超卖) 基于 Zookeeper 分布式锁 实现，同上一个意思。
```aidl
    @Override
	@Transactional
	public Result startSeckilZksLock(long seckillId, long userId) {
		boolean res=false;
		try {
			//基于redis分布式锁 基本就是上面这个解释 但是 使用zk分布式锁 使用本地zk服务 并发到10000+还是没有问题，谁的锅？
			res = ZkLockUtil.acquire(3,TimeUnit.SECONDS);
			if(res){
				String nativeSql = "SELECT number FROM seckill WHERE seckill_id=?";
				Object object =  dynamicQuery.nativeQueryObject(nativeSql, new Object[]{seckillId});
				Long number =  ((Number) object).longValue();
				if(number>0){
					SuccessKilled killed = new SuccessKilled();
					killed.setSeckillId(seckillId);
					killed.setUserId(userId);
					killed.setState((short)0);
					killed.setCreateTime(new Timestamp(new Date().getTime()));
					dynamicQuery.save(killed);
					nativeSql = "UPDATE seckill  SET number=number-1 WHERE seckill_id=? AND number>0";
					dynamicQuery.nativeExecuteUpdate(nativeSql, new Object[]{seckillId});
				}else{
					return Result.error(SeckillStatEnum.END);
				}
			}else{
			    return Result.error(SeckillStatEnum.MUCH);
			}
		} catch (Exception e) {
			e.printStackTrace();
		} finally{
			if(res){//释放锁
				ZkLockUtil.release();
			}
		}
		return Result.ok(SeckillStatEnum.SUCCESS);
	}
	
public class ZkLockUtil{
	
	private static String address = "192.168.1.180:2181";
	
	public static CuratorFramework client;
	
	static{
		RetryPolicy retryPolicy = new ExponentialBackoffRetry(1000, 3); 
        client = CuratorFrameworkFactory.newClient(address, retryPolicy); 
        client.start();
	}
	/**
     * 私有的默认构造子，保证外界无法直接实例化
     */
    private ZkLockUtil(){};
    /**
     * 类级的内部类，也就是静态的成员式内部类，该内部类的实例与外部类的实例
     * 没有绑定关系，而且只有被调用到才会装载，从而实现了延迟加载
     * 针对一件商品实现，多件商品同时秒杀建议实现一个map
     */
    private static class SingletonHolder{
        /**
         * 静态初始化器，由JVM来保证线程安全
         * 参考：http://ifeve.com/zookeeper-lock/
         * 这里建议 new 一个
         */
    	private  static InterProcessMutex mutex = new InterProcessMutex(client, "/curator/lock"); 
    }
    public static InterProcessMutex getMutex(){
        return SingletonHolder.mutex;
    }
    //获得了锁
    public static boolean acquire(long time, TimeUnit unit){
    	try {
			return getMutex().acquire(time,unit);
		} catch (Exception e) {
			e.printStackTrace();
			return false;
		}
    }
    //释放锁
    public static void release(){
    	try {
			getMutex().release();
		} catch (Exception e) {
			e.printStackTrace();
		}
    }
}  	
```
- 秒杀三 基于 Redis 分布式队列-订阅监听。
```aidl
    @ApiOperation(value="秒杀三(Redis分布式队列-订阅监听)",nickname="科帮网")
	@PostMapping("/startRedisQueue")
	public Result startRedisQueue(long seckillId){
		redisUtil.cacheValue(seckillId+"", null);//秒杀结束
		seckillService.deleteSeckill(seckillId);
		final long killId =  seckillId;
		LOGGER.info("开始秒杀三");
		for(int i=0;i<1000;i++){
			final long userId = i;
			Runnable task = new Runnable() {
				@Override
				public void run() {
					if(redisUtil.getValue(killId+"")==null){
						//思考如何返回给用户信息ws
						redisSender.sendChannelMess("seckill",killId+";"+userId);
					}else{
						//秒杀结束
					}
				}
			};
			executor.execute(task);
		}
		try {
			Thread.sleep(10000);
			redisUtil.cacheValue(killId+"", null);
			Long  seckillCount = seckillService.getSeckillCount(seckillId);
			LOGGER.info("一共秒杀出{}件商品",seckillCount);
		} catch (InterruptedException e) {
			e.printStackTrace();
		}
		return Result.ok();
	}
	
@Service
public class RedisSender {
    @Autowired
    private StringRedisTemplate stringRedisTemplate;
    //向通道发送消息的方法
    public void sendChannelMess(String channel, String message) {
        stringRedisTemplate.convertAndSend(channel, message);
    }
}
@Service
public class RedisConsumer {
	
	@Autowired
	private ISeckillService seckillService;
	@Autowired
	private RedisUtil redisUtil;
	
    public void receiveMessage(String message) {
        //收到通道的消息之后执行秒杀操作(超卖)
    	String[] array = message.split(";"); 
    	if(redisUtil.getValue(array[0])==null){//control层已经判断了，其实这里不需要再判断了
    		Result result = seckillService.startSeckilDBPCC_TWO(Long.parseLong(array[0]), Long.parseLong(array[1]));
    		if(result.equals(Result.ok(SeckillStatEnum.SUCCESS))){
    			WebSocketServer.sendInfo(array[0].toString(), "秒杀成功");//推送给前台 ,使用WebSocket技术
    		}else{
    			WebSocketServer.sendInfo(array[0].toString(), "秒杀失败");//推送给前台 ,使用WebSocket技术
    			redisUtil.cacheValue(array[0], "ok");//秒杀结束
    		}
    	}else{
    		WebSocketServer.sendInfo(array[0].toString(), "秒杀失败");//推送给前台 ,使用WebSocket技术
    	}
    }
}
/**
 * 群发自定义消息
 * */
public static void sendInfo(String message,@PathParam("userId") String userId){
    log.info("推送消息到窗口"+userId+"，推送内容:"+message);
    for (WebSocketServer item : webSocketSet) {
        try {
            //这里可以设定只推送给这个userId的，为null则全部推送
            if(userId==null) {
                item.sendMessage(message);
            }else if(item.userId.equals(userId)){
                item.sendMessage(message);
            }
        } catch (IOException e) {
            continue;
        }
    }
}
	
```

- 秒杀四基于 Kafka 分布式消息队列。
```aidl
@Component
public class KafkaSender {
    @Autowired
    private KafkaTemplate<String,String> kafkaTemplate;
    /**
     * 发送消息到kafka
     */
    public void sendChannelMess(String channel, String message){
        kafkaTemplate.send(channel,message);
    }
}
@Component
public class KafkaConsumer {
	@Autowired
	private ISeckillService seckillService;
	
	private static RedisUtil redisUtil = new RedisUtil();
    /**
     * 监听seckill主题,有消息就读取
     * @param message
     */
    @KafkaListener(topics = {"seckill"})
    public void receiveMessage(String message){
    	//收到通道的消息之后执行秒杀操作
    	String[] array = message.split(";"); 
    	if(redisUtil.getValue(array[0])==null){//control层已经判断了，其实这里不需要再判断了，这个接口有限流 注意一下
    		Result result = seckillService.startSeckil(Long.parseLong(array[0]), Long.parseLong(array[1]));
    		//可以注释掉上面的使用这个测试
    	    //Result result = seckillService.startSeckilDBPCC_TWO(Long.parseLong(array[0]), Long.parseLong(array[1]));
    		if(result.equals(Result.ok())){
    			WebSocketServer.sendInfo(array[0].toString(), "秒杀成功");//推送给前台 ,使用WebSocket技术
    		}else{
    			WebSocketServer.sendInfo(array[0].toString(), "秒杀失败");//推送给前台 ,使用WebSocket技术
    			redisUtil.cacheValue(array[0], "ok");//秒杀结束
    		}
    	}else{
    		WebSocketServer.sendInfo(array[0].toString(), "秒杀失败");//推送给前台 ,使用WebSocket技术
    	}
    }
}
```

### 3.2.3 使用WebSocket技术分析

```aidl
    @Configuration  
    public class WebSocketConfig {  
        @Bean  
        public ServerEndpointExporter serverEndpointExporter() {  
            return new ServerEndpointExporter();  
        }  
    }  
    
    
    
@ServerEndpoint("/websocket/{userId}")  
@Component  
public class WebSocketServer {  
	private final static Logger log = LoggerFactory.getLogger(WebSocketServer.class);
    //静态变量，用来记录当前在线连接数。应该把它设计成线程安全的。
    private static int onlineCount = 0;
    //concurrent包的线程安全Set，用来存放每个客户端对应的MyWebSocket对象。
    private static CopyOnWriteArraySet<WebSocketServer> webSocketSet = new CopyOnWriteArraySet<WebSocketServer>();

    //与某个客户端的连接会话，需要通过它来给客户端发送数据
    private Session session;

    //接收userId
    private String userId="";
    /**
     * 连接建立成功调用的方法*/
    @OnOpen
    public void onOpen(Session session,@PathParam("userId") String userId) {
        this.session = session;
        webSocketSet.add(this);     //加入set中
        addOnlineCount();           //在线数加1
        log.info("有新窗口开始监听:"+userId+",当前在线人数为" + getOnlineCount());
        this.userId=userId;
        try {
             sendMessage("连接成功");
        } catch (IOException e) {
            log.error("websocket IO异常");
        }
    }

    /**
     * 连接关闭调用的方法
     */
    @OnClose
    public void onClose() {
        webSocketSet.remove(this);  //从set中删除
        subOnlineCount();           //在线数减1
        log.info("有一连接关闭！当前在线人数为" + getOnlineCount());
    }

    /**
     * 收到客户端消息后调用的方法
     * @param message 客户端发送过来的消息*/
    @OnMessage
    public void onMessage(String message, Session session) {
        log.info("收到来自窗口"+userId+"的信息:"+message);
        //群发消息
        for (WebSocketServer item : webSocketSet) {
            try {
                item.sendMessage(message);
            } catch (IOException e) {
                e.printStackTrace();
            }
        }
    }

    /**
     * @param session
     * @param error
     */
    @OnError
    public void onError(Session session, Throwable error) {
        log.error("发生错误");
        error.printStackTrace();
    }
    /**
     * 实现服务器主动推送
     */
    public void sendMessage(String message) throws IOException {
        this.session.getBasicRemote().sendText(message);
    }
    /**
     * 群发自定义消息
     * */
    public static void sendInfo(String message,@PathParam("userId") String userId){
        log.info("推送消息到窗口"+userId+"，推送内容:"+message);
        for (WebSocketServer item : webSocketSet) {
            try {
                //这里可以设定只推送给这个userId的，为null则全部推送
                if(userId==null) {
                    item.sendMessage(message);
                }else if(item.userId.equals(userId)){
                    item.sendMessage(message);
                }
            } catch (IOException e) {
                continue;
            }
        }
    }

    public static synchronized int getOnlineCount() {
        return onlineCount;
    }

    public static synchronized void addOnlineCount() {
        WebSocketServer.onlineCount++;
    }

    public static synchronized void subOnlineCount() {
        WebSocketServer.onlineCount--;
    }
  
}  


客户端连接：
        var socket = new SockJS('/ws');
        stompClient = Stomp.over(socket);
        stompClient.connect({}, onConnected, onError);
android连接：  
    public class WebClient extends WebSocketClient{
        public static final String ACTION_RECEIVE_MESSAGE = "com.jinuo.mhwang.servermanager";
        public static final String KEY_RECEIVED_DATA = "data";
        private static WebClient mWebClient;
        private Context mContext;
        /**
         *  路径为ws+服务器地址+服务器端设置的子路径+参数（这里对应服务器端机器编号为参数）
         *  如果服务器端为https的，则前缀的ws则变为wss
         */
        private static final String mAddress = "ws://服务器地址：端口/mhwang7758/websocket/";
        private void showLog(String msg){
            Log.d("WebClient---->", msg);
        }
        private WebClient(URI serverUri, Context context){
            super(serverUri, new Draft_6455());
            mContext = context;
            showLog("WebClient");
        }   
        @Override
        public void onOpen(ServerHandshake handshakedata) {
            showLog("open->"+handshakedata.toString());
        }  
        @Override
        public void onMessage(String message) {
            showLog("onMessage->"+message);
            sendMessageBroadcast(message);
        } 
        @Override
        public void onClose(int code, String reason, boolean remote) {
            showLog("onClose->"+reason);
        }  
        @Override
        public void onError(Exception ex) {
            showLog("onError->"+ex.toString());
        }
        /** 初始化
         * @param vmc_no
         */
        public static void initWebSocket(final Context context, final long vmc_no){
            new Thread(new Runnable() {
                @Override
                public void run() {
                    try {
                        mWebClient = new WebClient(new URI(mAddress+vmc_no), context);
                        try {
                            mWebClient.connectBlocking();
                        } catch (InterruptedException e) {
                            e.printStackTrace();
                        }
                    } catch (URISyntaxException e) {
                        e.printStackTrace();
                    }
                }
            }).start();
        }
    
        /** 发送消息广播
         * @param message
         */
        private void sendMessageBroadcast(String message){
            if (!message.isEmpty()){
                Intent intent = new Intent();
                intent.setAction(ACTION_RECEIVE_MESSAGE);
                intent.putExtra(KEY_RECEIVED_DATA,message);
                showLog("发送收到的消息");
                mContext.sendBroadcast(intent);
            }
        }
    
    }

    
```

# 四、其他
