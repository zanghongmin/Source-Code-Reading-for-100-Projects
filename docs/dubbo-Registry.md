* [一、项目概览](#一数据类型)
    * [1.1 简介](#11-简介) 
    * [1.2 环境](#12-环境)
    * [1.3 源码及官网](#13-源码及官网)
* [二、项目使用](#二项目使用)
* [三、项目设计](#三项目设计)
    * [3.1 总体设计](#31-总体设计)
    * [3.2 关键点分析](#32-关键点分析)
        * [3.2.1 基本的XxlRegistryBaseClient与服务端各种交互实现原理](#321-基本的XxlRegistryBaseClient与服务端各种交互实现原理)  
        * [3.2.2 增强客户端的XxlRegistryClient实现原理](#322-增强客户端的XxlRegistryClient实现原理)
        * [3.2.3 长轮询方式long-polling实现原理](#323-长轮询方式long-polling实现原理)

* [四、其他](#四其他)

# 一、项目概览

## 1.1 简介
注册中心，多种实现方式，推荐使用zookeeper  
## 1.2 环境
    Dubbo2.6.7
## 1.3 源码及官网

[官网](http://dubbo.apache.org/zh-cn/docs/user/references/registry/introduction.html)

# 二、项目使用

- duddo中服务提供方使用注册中心


![duddo中服务提供方使用注册中心](pic/dubbo-Registry-user-provider.png)

```aidl
服务提供方配置文件增加注册中心，以zookeeper为例
    <beans xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
           xmlns:dubbo="http://dubbo.apache.org/schema/dubbo"
           xmlns="http://www.springframework.org/schema/beans"
           xsi:schemaLocation="http://www.springframework.org/schema/beans http://www.springframework.org/schema/beans/spring-beans-4.3.xsd
           http://dubbo.apache.org/schema/dubbo http://dubbo.apache.org/schema/dubbo/dubbo.xsd">
        <dubbo:application  name="validation-provider" />
        <dubbo:registry  address="zookeeper://10.130.0.237:2181"/>
        <dubbo:protocol  name="dubbo" port="20880"/>
        <bean id="validationService" class="com.alibaba.dubbo.examples.validation.impl.ValidationServiceImpl"/>
        <dubbo:service interface="com.alibaba.dubbo.examples.validation.api.ValidationService" ref="validationService"/>
    </beans>
服务提供者启动时: 向 /dubbo/com.alibaba.dubbo.examples.validation.api.ValidationService/providers 目录下写入自己的 URL 地址
    URL ：dubbo://10.6.1.250:20880/com.alibaba.dubbo.examples.validation.api.ValidationService?anyhost=true&application=validation-provider&dubbo=2.0.0&
    generic=false&interface=com.alibaba.dubbo.examples.validation.api.ValidationService&methods=save,update,delete&pid=243220&side=provider&timestamp=1573550757133
服务提供方服务加载时，调用RegistryProtocol的export方法中register向注册中心进行注册
     public void register(URL registryUrl, URL registedProviderUrl) {
         Registry registry = registryFactory.getRegistry(registryUrl);
         registry.register(registedProviderUrl);
     } 
Zookeeper中具体实现（向目录写入一个临时节点数据url）
    protected void doRegister(URL url) {
        try {
            zkClient.create(toUrlPath(url), url.getParameter(Constants.DYNAMIC_KEY, true));
        } catch (Throwable e) {
            throw new RpcException("Failed to register " + url + " to zookeeper " + getUrl() + ", cause: " + e.getMessage(), e);
        }
    }          
```

- duddo中服务消费方使用注册中心

![duddo中服务消费方使用注册中心](pic/dubbo-Registry-user-consumer.png)

```aidl
服务消费方配置文件增加注册中心，以zookeeper为例
    <beans xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
           xmlns:dubbo="http://dubbo.apache.org/schema/dubbo"
           xmlns="http://www.springframework.org/schema/beans"
           xsi:schemaLocation="http://www.springframework.org/schema/beans http://www.springframework.org/schema/beans/spring-beans-4.3.xsd
           http://dubbo.apache.org/schema/dubbo http://dubbo.apache.org/schema/dubbo/dubbo.xsd">
        <dubbo:application name="validation-consumer"/>
        <dubbo:registry  address="zookeeper://10.130.0.237:2181"/>
        <dubbo:reference id="validationService" interface="com.alibaba.dubbo.examples.validation.api.ValidationService"
                         validation="true"/>
    </beans>
服务消费者启动时: 订阅 /dubbo/com.alibaba.dubbo.examples.validation.api.ValidationService/providers 目录下的提供者 URL 地址。
并向 /dubbo/com.alibaba.dubbo.examples.validation.api.ValidationService/consumers 目录下写入自己的 URL 地址
服务消费者使用服务时，会调用RegistryProtocol的refre方法向注册中心进行注册和订阅
    private <T> Invoker<T> doRefer(Cluster cluster, Registry registry, Class<T> type, URL url) {
        RegistryDirectory<T> directory = new RegistryDirectory<T>(type, url);
        ...
        if (!Constants.ANY_VALUE.equals(url.getServiceInterface())
                && url.getParameter(Constants.REGISTER_KEY, true)) {
            registry.register(subscribeUrl.addParameters(Constants.CATEGORY_KEY, Constants.CONSUMERS_CATEGORY,
                    Constants.CHECK_KEY, String.valueOf(false))); //注册
        }
        directory.subscribe(subscribeUrl.addParameter(Constants.CATEGORY_KEY,
                Constants.PROVIDERS_CATEGORY
                        + "," + Constants.CONFIGURATORS_CATEGORY
                        + "," + Constants.ROUTERS_CATEGORY));//订阅
       ...
       
    }
Zookeeper中订阅具体实现（添加/dubbo/com.alibaba.dubbo.examples.validation.api.ValidationService/providers目录下的监听，数据有变化时，通过NotifyListener进行本地处理）
        for (String path : toCategoriesPath(url)) {
            ConcurrentMap<NotifyListener, ChildListener> listeners = zkListeners.get(url);
            if (listeners == null) {
                zkListeners.putIfAbsent(url, new ConcurrentHashMap<NotifyListener, ChildListener>());
                listeners = zkListeners.get(url);
            }
            ChildListener zkListener = listeners.get(listener);
            if (zkListener == null) {
                listeners.putIfAbsent(listener, new ChildListener() {
                    @Override
                    public void childChanged(String parentPath, List<String> currentChilds) {
                        ZookeeperRegistry.this.notify(url, listener, toUrlsWithEmpty(url, parentPath, currentChilds));
                    }
                });
                zkListener = listeners.get(listener);
            }
            zkClient.create(path, false);
            List<String> children = zkClient.addChildListener(path, zkListener);
            if (children != null) {
                urls.addAll(toUrlsWithEmpty(url, path, children));
            }
        }

```

- duddo admin中使用注册中心
![duddo admin中使用注册中心](pic/dubbo-Registry-user-admin.png)
```aidl
    admin启动时: 先订阅Zookeeper的/dubbo目录下的所有URL地址。再通过notify(List<URL> urls) 监听所有URL的变化  
    RegistryServerSync类中订阅时URL为： admin://10.6.1.250?category=providers,consumers,routers,configurators&check=false&classifier=*&enabled=*&group=*&interface=*&version=*
    Zookeeper中订阅具体实现
    protected void doSubscribe(final URL url, final NotifyListener listener) {
         try {
             if (Constants.ANY_VALUE.equals(url.getServiceInterface())) { // url中interface=*时走这个
                 String root = toRootPath();
                 ConcurrentMap<NotifyListener, ChildListener> listeners = zkListeners.get(url);
                 if (listeners == null) {
                     zkListeners.putIfAbsent(url, new ConcurrentHashMap<NotifyListener, ChildListener>());
                     listeners = zkListeners.get(url);
                 }
                 ChildListener zkListener = listeners.get(listener);
                 if (zkListener == null) {
                     listeners.putIfAbsent(listener, new ChildListener() { // /dubbo目录下有服务变化时的监听类，订阅新增加服务类
                         @Override
                         public void childChanged(String parentPath, List<String> currentChilds) { 
                             for (String child : currentChilds) { 
                                 child = URL.decode(child);
                                 if (!anyServices.contains(child)) {
                                     anyServices.add(child);
                                     subscribe(url.setPath(child).addParameters(Constants.INTERFACE_KEY, child,
                                             Constants.CHECK_KEY, String.valueOf(false)), listener);
                                 }
                             }
                         }
                     });
                     zkListener = listeners.get(listener);
                 }
                 zkClient.create(root, false);
                 List<String> services = zkClient.addChildListener(root, zkListener);
                 if (services != null && !services.isEmpty()) {
                     for (String service : services) { // /dubbo目录下具体服务类
                         service = URL.decode(service);
                         anyServices.add(service);
                         subscribe(url.setPath(service).addParameters(Constants.INTERFACE_KEY, service,
                                 Constants.CHECK_KEY, String.valueOf(false)), listener);// 订阅dubbo目录下具体服务类，走else逻辑，服务类中有变化时，走ZookeeperRegistry.this.notify(url, listener, toUrlsWithEmpty(url, parentPath, currentChilds));通知
                     }
                 }
             } else { 
                 List<URL> urls = new ArrayList<URL>();
                 for (String path : toCategoriesPath(url)) {
                     ConcurrentMap<NotifyListener, ChildListener> listeners = zkListeners.get(url);
                     if (listeners == null) {
                         zkListeners.putIfAbsent(url, new ConcurrentHashMap<NotifyListener, ChildListener>());
                         listeners = zkListeners.get(url);
                     }
                     ChildListener zkListener = listeners.get(listener);
                     if (zkListener == null) {
                         listeners.putIfAbsent(listener, new ChildListener() {
                             @Override
                             public void childChanged(String parentPath, List<String> currentChilds) {
                                 ZookeeperRegistry.this.notify(url, listener, toUrlsWithEmpty(url, parentPath, currentChilds));
                             }
                         });
                         zkListener = listeners.get(listener);
                     }
                     zkClient.create(path, false);
                     List<String> children = zkClient.addChildListener(path, zkListener);
                     if (children != null) {
                         urls.addAll(toUrlsWithEmpty(url, path, children));
                     }
                 }
                 notify(url, listener, urls);
             }
         } catch (Throwable e) {
             throw new RpcException("Failed to subscribe " + url + " to zookeeper " + getUrl() + ", cause: " + e.getMessage(), e);
         }
     }
     
 admin中 NotifyListener listener  具体实现，根据变化的URL 按类目放入本地缓存registryCache中
     public void notify(List<URL> urls) {
             if (urls == null || urls.isEmpty()) {
                 return;
             }
             // Map<category, Map<servicename, Map<Long, URL>>>
             final Map<String, Map<String, Map<Long, URL>>> categories = new HashMap<String, Map<String, Map<Long, URL>>>();
             String interfaceName = null;
             for (URL url : urls) {
                 String category = url.getParameter(Constants.CATEGORY_KEY, Constants.PROVIDERS_CATEGORY);
                 if (Constants.EMPTY_PROTOCOL.equalsIgnoreCase(url.getProtocol())) { // NOTE: group and version in empty protocol is *
                     ConcurrentMap<String, Map<Long, URL>> services = registryCache.get(category);
                     if (services != null) {
                         String group = url.getParameter(Constants.GROUP_KEY);
                         String version = url.getParameter(Constants.VERSION_KEY);
                         // NOTE: group and version in empty protocol is *
                         if (!Constants.ANY_VALUE.equals(group) && !Constants.ANY_VALUE.equals(version)) {
                             services.remove(url.getServiceKey());
                         } else {
                             for (Map.Entry<String, Map<Long, URL>> serviceEntry : services.entrySet()) {
                                 String service = serviceEntry.getKey();
                                 if (Tool.getInterface(service).equals(url.getServiceInterface())
                                         && (Constants.ANY_VALUE.equals(group) || StringUtils.isEquals(group, Tool.getGroup(service)))
                                         && (Constants.ANY_VALUE.equals(version) || StringUtils.isEquals(version, Tool.getVersion(service)))) {
                                     services.remove(service);
                                 }
                             }
                         }
                     }
                 } else {
                     if (StringUtils.isEmpty(interfaceName)) {
                         interfaceName = url.getServiceInterface();
                     }
                     Map<String, Map<Long, URL>> services = categories.get(category);
                     if (services == null) {
                         services = new HashMap<String, Map<Long, URL>>();
                         categories.put(category, services);
                     }
                     String service = url.getServiceKey();
                     Map<Long, URL> ids = services.get(service);
                     if (ids == null) {
                         ids = new HashMap<Long, URL>();
                         services.put(service, ids);
                     } 
                     // Make sure we use the same ID for the same URL
                     if (URL_IDS_MAPPER.containsKey(url.toFullString())) {
                         ids.put(URL_IDS_MAPPER.get(url.toFullString()), url);
                     } else {
                         long currentId = ID.incrementAndGet();
                         ids.put(currentId, url);
                         URL_IDS_MAPPER.putIfAbsent(url.toFullString(), currentId);
                     }
                 }
             }
             if (categories.size() == 0) {
                 return;
             }
             for (Map.Entry<String, Map<String, Map<Long, URL>>> categoryEntry : categories.entrySet()) {
                 String category = categoryEntry.getKey();
                 ConcurrentMap<String, Map<Long, URL>> services = registryCache.get(category);
                 if (services == null) {
                     services = new ConcurrentHashMap<String, Map<Long, URL>>();
                     registryCache.put(category, services);
                 } else {// Fix map can not be cleared when service is unregistered: when a unique “group/service:version” service is unregistered, but we still have the same services with different version or group, so empty protocols can not be invoked.
                     Set<String> keys = new HashSet<String>(services.keySet());
                     for (String key : keys) {
                         if (Tool.getInterface(key).equals(interfaceName) && !categoryEntry.getValue().entrySet().contains(key)) {
                             services.remove(key);
                         }
                     }
                 }
                 services.putAll(categoryEntry.getValue());
             }
         }
     
    
```

# 三、项目设计


## 3.1 总体设计-dubbo注册中心接口层分析

![代码结构](pic/dubbo-Registry-ALL.png)

- 接口类RegistryService和NotifyListener
```aidl
注册中心实现5个功能方法，注册、取消注册、订阅、取消订阅和查找,
dubbo 有多种注册中心实现，zookeeper实现、multicast实现、Redis实现等

接口RegistryService5个功能定义
    public interface RegistryService {
        void register(URL url); 
        void unregister(URL url);
        void subscribe(URL url, NotifyListener listener);
        void unsubscribe(URL url, NotifyListener listener);
        List<URL> lookup(URL url); -- 被调用的地方还没看
    }
接口NotifyListener,用来在订阅或订阅数据变化时，通知订阅方进行相应的处理（如新增或删除服务提供者，订阅者收到服务提供者列表变化通知，进行本地处理）
    public interface NotifyListener {
        void notify(List<URL> urls);
    }
```
- 抽象类FailbackRegistry
```aidl
抽象类FailbackRegistry定义模板方法，具体实现由zookeeper、multicast、Redis等实现
FailbackRegistry继承AbstractRegistry，先看下AbstractRegistry（实现RegistryService5个方法）
AbstractRegistry有3个变量，保存本地数据
    private final Set<URL> registered = new ConcurrentHashSet<URL>(); //保存注册数据，如当服务提供方注册时，会在本地这个变量中保存服务URL 
    private final ConcurrentMap<URL, Set<NotifyListener>> subscribed = new ConcurrentHashMap<URL, Set<NotifyListener>>();//保存订阅数据，如当服务消费方订阅时，会在本地这个变量中保存订阅URL和监听类
    private final ConcurrentMap<URL, Map<String, List<URL>>> notified = new ConcurrentHashMap<URL, Map<String, List<URL>>>();//保存服务消费订阅URL 及 远程提供方UAL对应关系

AbstractRegistry的register方法
    @Override
    public void register(URL url) {
        if (url == null) {
            throw new IllegalArgumentException("register url == null");
        }
        if (logger.isInfoEnabled()) {
            logger.info("Register: " + url);
        }
        registered.add(url); ////保存注册数据
    }

AbstractRegistry的subscribe方法
    @Override
    public void subscribe(URL url, NotifyListener listener) {
        Set<NotifyListener> listeners = subscribed.get(url);
        if (listeners == null) {
            subscribed.putIfAbsent(url, new ConcurrentHashSet<NotifyListener>());
            listeners = subscribed.get(url);
        }
        listeners.add(listener);//保存订阅数据
    }

AbstractRegistry的lookup方法 - 这个没看到哪块使用这个方法
    @Override
    public List<URL> lookup(URL url) {
        List<URL> result = new ArrayList<URL>();
        Map<String, List<URL>> notifiedUrls = getNotified().get(url);
        if (notifiedUrls != null && notifiedUrls.size() > 0) {
            for (List<URL> urls : notifiedUrls.values()) {
                for (URL u : urls) {
                    if (!Constants.EMPTY_PROTOCOL.equals(u.getProtocol())) {
                        result.add(u);
                    }
                }
            }
        } else {
            final AtomicReference<List<URL>> reference = new AtomicReference<List<URL>>();
            NotifyListener listener = new NotifyListener() {
                @Override
                public void notify(List<URL> urls) {
                    reference.set(urls);
                }
            };
            subscribe(url, listener); // Subscribe logic guarantees the first notify to return
            List<URL> urls = reference.get();
            if (urls != null && !urls.isEmpty()) {
                for (URL u : urls) {
                    if (!Constants.EMPTY_PROTOCOL.equals(u.getProtocol())) {
                        result.add(u);
                    }
                }
            }
        }
        return result;
    }
    
 AbstractRegistry的notify方法(消费方和提供方都有使用)
   1. 消费方从注册中心获取所有服务URL或服务URL有变化时，执行这个监听方法，生成（或重新生成）本地调用远程的方法
   2. 提供方从注册中心获取服务URL的Configurator目录的变化，用来重新根据配置信息生成提供方方法
   protected void notify(URL url, NotifyListener listener, List<URL> urls) { // urls为从注册中心获取所有的提供方URL
          Map<String, List<URL>> result = new HashMap<String, List<URL>>();
          for (URL u : urls) {
              if (UrlUtils.isMatch(url, u)) {
                  String category = u.getParameter(Constants.CATEGORY_KEY, Constants.DEFAULT_CATEGORY);
                  List<URL> categoryList = result.get(category);
                  if (categoryList == null) {
                      categoryList = new ArrayList<URL>();
                      result.put(category, categoryList);
                  }
                  categoryList.add(u);
              }
          }
          if (result.size() == 0) {
              return;
          }
          Map<String, List<URL>> categoryNotified = notified.get(url);
          if (categoryNotified == null) {
              notified.putIfAbsent(url, new ConcurrentHashMap<String, List<URL>>());
              categoryNotified = notified.get(url);
          }
          for (Map.Entry<String, List<URL>> entry : result.entrySet()) {
              String category = entry.getKey();
              List<URL> categoryList = entry.getValue();
              categoryNotified.put(category, categoryList);
              saveProperties(url);
              listener.notify(categoryList);
              //listener.notify的消费方实现 ： 具体实现在RegistryDirectory类型，Map<String, Invoker<T>> newUrlInvokerMap = toInvokers(invokerUrls); 主要是把URL变化Invoker，这块在消费方调用时，再介绍
              //listener.notify的提供方实现 :  具体实现在RegistryProtocol中OverrideListener类型，List<Configurator> configurators = RegistryDirectory.toConfigurators(matchedUrls);RegistryProtocol.this.doChangeLocalExport(originInvoker, newUrl); 主要是根据配置信息，重新生成本地提供方方法
          }
      }  
 

FailbackRegistry继承AbstractRegistry
    5个变量，注册失败，订阅失败，通知失败等（注册中心重连，注册中心掉线，本地监听通知异常等）的URL数据，用于在retryExecutor线程池中执行重新注册，订阅等操作
    private final Set<URL> failedRegistered = new ConcurrentHashSet<URL>();  
    private final Set<URL> failedUnregistered = new ConcurrentHashSet<URL>();  
    private final ConcurrentMap<URL, Set<NotifyListener>> failedSubscribed = new ConcurrentHashMap<URL, Set<NotifyListener>>();  
    private final ConcurrentMap<URL, Set<NotifyListener>> failedUnsubscribed = new ConcurrentHashMap<URL, Set<NotifyListener>>(); 
    private final ConcurrentMap<URL, Map<NotifyListener, List<URL>>> failedNotified = new ConcurrentHashMap<URL, Map<NotifyListener, List<URL>>>();//通知失败的记录结果 和父类notified有关系
    
    retryExecutor线程池（每5秒运行一次）
        this.retryFuture = retryExecutor.scheduleWithFixedDelay(new Runnable() {
            @Override
            public void run() {
                // Check and connect to the registry
                try {
                    retry(); //判断5个变量是否为空，如failedRegistered有值时，就执行doRegister，failedSubscribed有值时就执行doSubscribe
                            //failedNotified有值时就执行NotifyListener.notify(urls);
                } catch (Throwable t) { // Defensive fault tolerance
                    logger.error("Unexpected error occur at failed retry, cause: " + t.getMessage(), t);
                }
            }
        }, retryPeriod, retryPeriod, TimeUnit.MILLISECONDS);
     
     
     //监控通知( 调用父类AbstractRegistry的notify方法)
     /调用父类的方法，进行本地监控通知，如注册中心，提供方数据有变化时，本地消费方根据提供方URL，重新生成远程调用地址
     protected void doNotify(URL url, NotifyListener listener, List<URL> urls) {
             super.notify(url, listener, urls);
     }

  
     // ==== Template method ====    具体实现在子类 zookeeper,redis,multicast实现等
     protected abstract void doRegister(URL url);
     protected abstract void doUnregister(URL url);
     protected abstract void doSubscribe(URL url, NotifyListener listener);
     protected abstract void doUnsubscribe(URL url, NotifyListener listener); 
           
        
FailbackRegistry的register方法
    @Override
    public void register(URL url) {
        super.register(url); //调用AbstractRegistry的register
        failedRegistered.remove(url);
        failedUnregistered.remove(url);
        try {
            // Sending a registration request to the server side
            // 具体实现在子类 zookeeper,redis,multicast实现等
            doRegister(url);
        } catch (Exception e) {
            Throwable t = e;
            // If the startup detection is opened, the Exception is thrown directly.
            boolean check = getUrl().getParameter(Constants.CHECK_KEY, true)
                    && url.getParameter(Constants.CHECK_KEY, true)
                    && !Constants.CONSUMER_PROTOCOL.equals(url.getProtocol());
            boolean skipFailback = t instanceof SkipFailbackWrapperException;
            if (check || skipFailback) {
                if (skipFailback) {
                    t = t.getCause();
                }
                throw new IllegalStateException("Failed to register " + url + " to registry " + getUrl().getAddress() + ", cause: " + t.getMessage(), t);
            } else {
                logger.error("Failed to register " + url + ", waiting for retry, cause: " + t.getMessage(), t);
            }
            // Record a failed registration request to a failed list, retry regularly
            failedRegistered.add(url); //注册失败时，保存下，retryExecutor线程池 5秒重试
        }
    }
    
FailbackRegistry的subscribe方法   
   @Override
    public void subscribe(URL url, NotifyListener listener) {
        super.subscribe(url, listener);
        removeFailedSubscribed(url, listener);
        try {
            // Sending a subscription request to the server side
            // 具体实现在子类 zookeeper,redis,multicast实现等
            doSubscribe(url, listener);
        } catch (Exception e) {
            Throwable t = e;
            List<URL> urls = getCacheUrls(url);
            if (urls != null && !urls.isEmpty()) {
                notify(url, listener, urls);
                logger.error("Failed to subscribe " + url + ", Using cached list: " + urls + " from cache file: " + getUrl().getParameter(Constants.FILE_KEY, System.getProperty("user.home") + "/dubbo-registry-" + url.getHost() + ".cache") + ", cause: " + t.getMessage(), t);
            } else {
                // If the startup detection is opened, the Exception is thrown directly.
                boolean check = getUrl().getParameter(Constants.CHECK_KEY, true)
                        && url.getParameter(Constants.CHECK_KEY, true);
                boolean skipFailback = t instanceof SkipFailbackWrapperException;
                if (check || skipFailback) {
                    if (skipFailback) {
                        t = t.getCause();
                    }
                    throw new IllegalStateException("Failed to subscribe " + url + ", cause: " + t.getMessage(), t);
                } else {
                    logger.error("Failed to subscribe " + url + ", waiting for retry, cause: " + t.getMessage(), t);
                }
            }
            // Record a failed registration request to a failed list, retry regularly
            //订阅 失败时，保存下，retryExecutor线程池 5秒重试
            addFailedSubscribed(url, listener);
        }
    }
    
FailbackRegistry的unregister 和 unsubscribe方法类似
```

- 接口类RegistryFactory

```aidl
RegistryFactory注册中心工厂，用来生成RegistryFactory
    @SPI("dubbo") //spi是dubbo的加载扩展的注解，根据URL中的参数或其他配置加载对应的具体实现类，spi在其他文章再介绍
    public interface RegistryFactory {
        @Adaptive({"protocol"})
        Registry getRegistry(URL url);
    }
 
抽象类AbstractRegistryFactory   
    private static final ReentrantLock LOCK = new ReentrantLock();
    // 保存创建过的注册中心类，同一个注册中心地址，只有一个实现类
    private static final Map<String, Registry> REGISTRIES = new ConcurrentHashMap<String, Registry>();
    @Override
    public Registry getRegistry(URL url) {
        //zookeeper://10.130.0.237:2181/com.alibaba.dubbo.registry.RegistryService?application=validation-provider&compiler=jdk&dubbo=2.0.2&interface=com.alibaba.dubbo.registry.RegistryService&pid=198260&timestamp=1574667468400
        url = url.setPath(RegistryService.class.getName())
                .addParameter(Constants.INTERFACE_KEY, RegistryService.class.getName())
                .removeParameters(Constants.EXPORT_KEY, Constants.REFER_KEY);
        //key为zookeeper://10.130.0.237:2181/com.alibaba.dubbo.registry.RegistryService
        String key = url.toServiceStringWithoutResolving();
        // Lock the registry access process to ensure a single instance of the registry
        LOCK.lock();
        try {
            Registry registry = REGISTRIES.get(key);
            if (registry != null) {
                return registry;
            }
            registry = createRegistry(url);
            if (registry == null) {
                throw new IllegalStateException("Can not create registry " + url);
            }
            REGISTRIES.put(key, registry);
            return registry;
        } finally {
            // Release the lock
            LOCK.unlock();
        }
    }
    // 具体实现在子类 zookeeper,redis,multicast实现等
    protected abstract Registry createRegistry(URL url);
    
    //RegistryFactory在RegistryProtocol中通过SPI自动注入方式实例化（自适应类）
    public class RegistryProtocol implements Protocol {  
            ...
            private RegistryFactory registryFactory;
            ...
            //registryFactory = {RegistryFactory$Adaptive@2829} 
            public void setRegistryFactory(RegistryFactory registryFactory) {
                this.registryFactory = registryFactory;
            }
            //注册
            public void register(URL registryUrl, URL registedProviderUrl) {
                Registry registry = registryFactory.getRegistry(registryUrl);
                registry.register(registedProviderUrl);
            }
    
    }

```

## 3.2 dubbo注册中心-zookeeper实现分析

 - zookeeper介绍
 
 ![duddo中服务消费方使用注册中心](pic/dubbo-Registry-zookeeper.png)
 ```aidl
Zookeeper 是 Apacahe Hadoop 的子项目，是一个树型的目录服务，支持变更推送，适合作为 Dubbo 服务的注册中心，工业强度较高，可用于生产环境，并推荐使用 [1]。

流程说明：
    服务提供者启动时: 向 /dubbo/com.foo.BarService/providers 目录下写入自己的 URL 地址
    服务消费者启动时: 订阅 /dubbo/com.foo.BarService/providers 目录下的提供者 URL 地址。并向 /dubbo/com.foo.BarService/consumers 目录下写入自己的 URL 地址
    监控中心启动时: 订阅 /dubbo/com.foo.BarService 目录下的所有提供者和消费者 URL 地址。

支持以下功能：
    当提供者出现断电等异常停机时，注册中心能自动删除提供者信息
    当注册中心重启时，能自动恢复注册数据，以及订阅请求
    当会话过期时，能自动恢复注册数据，以及订阅请求
    当设置 <dubbo:registry check="false" /> 时，记录失败注册和订阅请求，后台定时重试
    可通过 <dubbo:registry username="admin" password="1234" /> 设置 zookeeper 登录信息
    可通过 <dubbo:registry group="dubbo" /> 设置 zookeeper 的根节点，不配置将使用默认的根节点。
    支持 * 号通配符 <dubbo:reference group="*" version="*" />，可订阅服务的所有分组和所有版本的提供者

```

- ZookeeperRegistryFactory

```aidl

//SPI接口com.alibaba.dubbo.registry.RegistryFactory
//zookeeper=com.alibaba.dubbo.registry.zookeeper.ZookeeperRegistryFactory
//当Registry registry = registryFactory.getRegistry(registryUrl);（自适应类），根据URL的协议加载zookeeper或别的实现
public class ZookeeperRegistryFactory extends AbstractRegistryFactory {
    private ZookeeperTransporter zookeeperTransporter;
    //自适应类SPI自动注入
    //zookeeperTransporter = {ZookeeperTransporter$Adaptive@3476} 
    public void setZookeeperTransporter(ZookeeperTransporter zookeeperTransporter) {
        this.zookeeperTransporter = zookeeperTransporter;
    }
    //直接new对象，在AbstractRegistryFactory中有缓存，只会运行一次这个
    @Override
    public Registry createRegistry(URL url) {
        return new ZookeeperRegistry(url, zookeeperTransporter);
    }
}

```
- ZookeeperRegistry

```aidl
    //默认端口
    private final static int DEFAULT_ZOOKEEPER_PORT = 2181;
    private final static String DEFAULT_ROOT = "dubbo";
    //在Zookeeper上存储数据的路径，默认是/dubbo
    private final String root;
    private final Set<String> anyServices = new ConcurrentHashSet<String>();
    private final ConcurrentMap<URL, ConcurrentMap<NotifyListener, ChildListener>> zkListeners = new ConcurrentHashMap<URL, ConcurrentMap<NotifyListener, ChildListener>>();
    //链接Zookeeper的客户端
    private final ZookeeperClient zkClient;
    //ZookeeperTransporter为SPI接口，zookeeperTransporter为自适应类，根据URL的参数获取具体实现，默认实现为curator
    //ZookeeperTransporter为com.alibaba.dubbo.remoting的内容，具体看dubbo-remoting的介绍
    public ZookeeperRegistry(URL url, ZookeeperTransporter zookeeperTransporter) {
        //调用父类，主要有启动retryExecutor线程池（每5秒运行一次重试注册失败等）
        super(url);
        if (url.isAnyHost()) {
            throw new IllegalStateException("registry address == null");
        }
        String group = url.getParameter(Constants.GROUP_KEY, DEFAULT_ROOT);
        if (!group.startsWith(Constants.PATH_SEPARATOR)) {
            group = Constants.PATH_SEPARATOR + group;
        }
        //在Zookeeper上存储数据的路径，默认是/dubbo
        this.root = group;
        //初始化链接
        zkClient = zookeeperTransporter.connect(url);
        //增加监听
        zkClient.addStateListener(new StateListener() {
            @Override
            public void stateChanged(int state) {
                //和zookeeprt重连时进行recover()。（重新进行服务注册和服务订阅）
                if (state == RECONNECTED) {
                    try {
                        recover();
                    } catch (Exception e) {
                        logger.error(e.getMessage(), e);
                    }
                }
            }
        });
    }
    @Override
    protected void recover() throws Exception {
        // register 把当前已注册的服务设置为失败状态，failedRegistered（retryExecutor线程池（每5秒运行一次重试注册失败等））
        Set<URL> recoverRegistered = new HashSet<URL>(getRegistered());
        if (!recoverRegistered.isEmpty()) {
            if (logger.isInfoEnabled()) {
                logger.info("Recover register url " + recoverRegistered);
            }
            for (URL url : recoverRegistered) {
                failedRegistered.add(url);
            }
        }
        // subscribe 把当前已订阅的服务设置为失败状态，addFailedSubscribed（retryExecutor线程池（每5秒运行一次重试订阅失败等））
        Map<URL, Set<NotifyListener>> recoverSubscribed = new HashMap<URL, Set<NotifyListener>>(getSubscribed());
        if (!recoverSubscribed.isEmpty()) {
            if (logger.isInfoEnabled()) {
                logger.info("Recover subscribe url " + recoverSubscribed.keySet());
            }
            for (Map.Entry<URL, Set<NotifyListener>> entry : recoverSubscribed.entrySet()) {
                URL url = entry.getKey();
                for (NotifyListener listener : entry.getValue()) {
                    addFailedSubscribed(url, listener);
                }
            }
        }
    }
    
    // 注册实现
    @Override
    protected void doRegister(URL url) {
        try {
            //目录， 是否为持久节点
            zkClient.create(toUrlPath(url), url.getParameter(Constants.DYNAMIC_KEY, true));
        } catch (Throwable e) {
            throw new RpcException("Failed to register " + url + " to zookeeper " + getUrl() + ", cause: " + e.getMessage(), e);
        }
    }
    //目录， 是否为临时节点，这个是curator中的实现
    @Override
    public void create(String path, boolean ephemeral) {
        if (!ephemeral) {
            if (checkExists(path)) {
                return;
            }
        }
        int i = path.lastIndexOf('/');
        if (i > 0) {
            create(path.substring(0, i), false);
        }
        if (ephemeral) {
            createEphemeral(path);
        } else {
            createPersistent(path);
        }
    }
    
    //去掉注册实现
    @Override
    protected void doUnregister(URL url) {
        try {
            //删除对应的目录
            zkClient.delete(toUrlPath(url));
        } catch (Throwable e) {
            throw new RpcException("Failed to unregister " + url + " to zookeeper " + getUrl() + ", cause: " + e.getMessage(), e);
        }
    }

    //订阅实现 
    // 消费者端例子
    //url : consumer://10.6.1.250/com.alibaba.dubbo.examples.validation.api.ValidationService?application=validation-consumer&category=providers,configurators,routers&dubbo=2.0.2&interface=com.alibaba.dubbo.examples.validation.api.ValidationService&methods=save,update,delete&pid=217112&side=consumer&timestamp=1574731839267&validation=true
    //listener = {RegistryDirectory@2938} 
    @Override
    protected void doSubscribe(final URL url, final NotifyListener listener) {
        try {
            //这个url中interface=*时走这个
            //dubbo admin走这个逻辑
            if (Constants.ANY_VALUE.equals(url.getServiceInterface())) {
                String root = toRootPath();
                ConcurrentMap<NotifyListener, ChildListener> listeners = zkListeners.get(url);
                if (listeners == null) {
                    zkListeners.putIfAbsent(url, new ConcurrentHashMap<NotifyListener, ChildListener>());
                    listeners = zkListeners.get(url);
                }
                ChildListener zkListener = listeners.get(listener);
                if (zkListener == null) {
                    listeners.putIfAbsent(listener, new ChildListener() {
                        @Override
                        public void childChanged(String parentPath, List<String> currentChilds) {
                            for (String child : currentChilds) {
                                child = URL.decode(child);
                                if (!anyServices.contains(child)) {
                                    anyServices.add(child);
                                     // /dubbo目录下有服务变化时的监听类，订阅新增加服务类
                                    subscribe(url.setPath(child).addParameters(Constants.INTERFACE_KEY, child,
                                            Constants.CHECK_KEY, String.valueOf(false)), listener);
                                }
                            }
                        }
                    });
                    zkListener = listeners.get(listener);
                }
                zkClient.create(root, false);
                List<String> services = zkClient.addChildListener(root, zkListener);
                if (services != null && !services.isEmpty()) {
                    for (String service : services) {
                        service = URL.decode(service);
                        anyServices.add(service);
                        subscribe(url.setPath(service).addParameters(Constants.INTERFACE_KEY, service,
                                Constants.CHECK_KEY, String.valueOf(false)), listener);
                    }
                }
            } else {
                //服务订阅时走这个逻辑
                List<URL> urls = new ArrayList<URL>();
                //path 为/dubbo/com.alibaba.dubbo.examples.validation.api.ValidationService/providers
                //path 为 /dubbo/com.alibaba.dubbo.examples.validation.api.ValidationService/configurators
                //path 为 /dubbo/com.alibaba.dubbo.examples.validation.api.ValidationService/routers
                //消费端会订阅这3个目录
                for (String path : toCategoriesPath(url)) {
                    ConcurrentMap<NotifyListener, ChildListener> listeners = zkListeners.get(url);
                    if (listeners == null) {
                        zkListeners.putIfAbsent(url, new ConcurrentHashMap<NotifyListener, ChildListener>());
                        listeners = zkListeners.get(url);
                    }
                    ChildListener zkListener = listeners.get(listener);
                    if (zkListener == null) {
                        listeners.putIfAbsent(listener, new ChildListener() {
                            @Override
                            public void childChanged(String parentPath, List<String> currentChilds) {
                                //当监听路径的子目录有变化时，调用该方法
                                //调用的是FailbackRegistry中的notify方法，进行本地服务方法重新注册或调阅
                                ZookeeperRegistry.this.notify(url, listener, toUrlsWithEmpty(url, parentPath, currentChilds));
                            }
                        });
                        zkListener = listeners.get(listener);
                    }
                    zkClient.create(path, false);
                    //给path路径添加监听
                    //返回的值为dubbo%3A%2F%2F10.6.1.250%3A20880%2Fcom.alibaba.dubbo.examples.validation.api.ValidationService%3Faccesslog%3Dtrue%26anyhost%3Dtrue%26application%3Dvalidation-provider%26bean.name%3Dcom.alibaba.dubbo.examples.validation.api.ValidationService%26compiler%3Djdk%26dubbo%3D2.0.2%26generic%3Dfalse%26interface%3Dcom.alibaba.dubbo.examples.validation.api.ValidationService%26methods%3Dsave%2Cupdate%2Cdelete%26pid%3D198160%26side%3Dprovider%26timestamp%3D1574668172852
                    //返回的是当前目录下，所有的URL形式的目录字符串
                    List<String> children = zkClient.addChildListener(path, zkListener);
                    if (children != null) {
                        urls.addAll(toUrlsWithEmpty(url, path, children));
                    }
                    //urls为,children为0时URL协议为empty
                    //0 = {URL@3137} "dubbo://10.6.1.250:20880/com.alibaba.dubbo.examples.validation.api.ValidationService?accesslog=true&anyhost=true&application=validation-provider&bean.name=com.alibaba.dubbo.examples.validation.api.ValidationService&compiler=jdk&dubbo=2.0.2&generic=false&interface=com.alibaba.dubbo.examples.validation.api.ValidationService&methods=save,update,delete&pid=198160&side=provider&timestamp=1574668172852"
                    //1 = {URL@3138} "empty://10.6.1.250/com.alibaba.dubbo.examples.validation.api.ValidationService?application=validation-consumer&category=configurators&dubbo=2.0.2&interface=com.alibaba.dubbo.examples.validation.api.ValidationService&methods=save,update,delete&pid=223972&side=consumer&timestamp=1574732691830&validation=true"
                    //2 = {URL@3322} "empty://10.6.1.250/com.alibaba.dubbo.examples.validation.api.ValidationService?application=validation-consumer&category=routers&dubbo=2.0.2&interface=com.alibaba.dubbo.examples.validation.api.ValidationService&methods=save,update,delete&pid=223972&side=consumer&timestamp=1574732691830&validation=true"
                }
                //根据返回的urls，第一次调用的是FailbackRegistry中的notify方法，进行本地服务方法重新注册或调阅
                notify(url, listener, urls);
            }
        } catch (Throwable e) {
            throw new RpcException("Failed to subscribe " + url + " to zookeeper " + getUrl() + ", cause: " + e.getMessage(), e);
        }
    }

    //去掉订阅实现 
    @Override
    protected void doUnsubscribe(URL url, NotifyListener listener) {
        ConcurrentMap<NotifyListener, ChildListener> listeners = zkListeners.get(url);
        if (listeners != null) {
            ChildListener zkListener = listeners.get(listener);
            if (zkListener != null) {
                if (Constants.ANY_VALUE.equals(url.getServiceInterface())) {
                    String root = toRootPath();
                    zkClient.removeChildListener(root, zkListener);
                } else {
                    for (String path : toCategoriesPath(url)) {
                        zkClient.removeChildListener(path, zkListener);
                    }
                }
            }
        }
    }
    //查找实现
    @Override
    public List<URL> lookup(URL url) {
        if (url == null) {
            throw new IllegalArgumentException("lookup url == null");
        }
        try {
            List<String> providers = new ArrayList<String>();
            for (String path : toCategoriesPath(url)) {
                List<String> children = zkClient.getChildren(path);
                if (children != null) {
                    providers.addAll(children);
                }
            }
            return toUrlsWithoutEmpty(url, providers);
        } catch (Throwable e) {
            throw new RpcException("Failed to lookup " + url + " from zookeeper " + getUrl() + ", cause: " + e.getMessage(), e);
        }
    }
    private List<URL> toUrlsWithoutEmpty(URL consumer, List<String> providers) {
        List<URL> urls = new ArrayList<URL>();
        if (providers != null && !providers.isEmpty()) {
            for (String provider : providers) {
                provider = URL.decode(provider);
                if (provider.contains("://")) {
                    URL url = URL.valueOf(provider);
                    if (UrlUtils.isMatch(consumer, url)) {
                        urls.add(url);
                    }
                }
            }
        }
        return urls;
    }
    
    private List<URL> toUrlsWithEmpty(URL consumer, String path, List<String> providers) {
        List<URL> urls = toUrlsWithoutEmpty(consumer, providers);
        if (urls == null || urls.isEmpty()) {
            int i = path.lastIndexOf('/');
            String category = i < 0 ? path : path.substring(i + 1);
            URL empty = consumer.setProtocol(Constants.EMPTY_PROTOCOL).addParameter(Constants.CATEGORY_KEY, category);
            urls.add(empty);
        }
        return urls;
    }
```




## 3.3 dubbo注册中心-multicast实现分析

 - multicast介绍
 
 ![duddo中服务消费方使用注册中心](pic/dubbo-Registry-multicast.png)

```aidl
Multicast 注册中心不需要启动任何中心节点，只要广播地址一样，就可以互相发现。
    提供方启动时广播自己的地址
    消费方启动时广播订阅请求
    提供方收到订阅请求时，单播自己的地址给订阅者，如果设置了 unicast=false，则广播给订阅者
    消费方收到提供方地址时，连接该地址进行 RPC 调用。
组播受网络结构限制，只适合小规模应用或开发阶段使用。组播地址段: 224.0.0.0 - 239.255.255.255

配置
<dubbo:registry address="multicast://224.5.6.7:1234" />
或
<dubbo:registry protocol="multicast" address="224.5.6.7:1234" />
为了减少广播量，Dubbo 缺省使用单播发送提供者地址信息给消费者，如果一个机器上同时启了多个消费者进程，消费者需声明 unicast=false，
否则只会有一个消费者能收到消息;当服务者和消费者运行在同一台机器上，消费者同样需要声明unicast=false，
否则消费者无法收到消息，导致No provider available for the service异常：
<dubbo:registry address="multicast://224.5.6.7:1234?unicast=false" />
或
<dubbo:registry protocol="multicast" address="224.5.6.7:1234">
    <dubbo:parameter key="unicast" value="false" />
</dubbo:registry>
```

- MulticastRegistry

```aidl
配置 <dubbo:registry address="multicast://224.5.6.7:1234"/>

//SPI接口com.alibaba.dubbo.registry.RegistryFactory
//multicast=com.alibaba.dubbo.registry.multicast.MulticastRegistryFactory
//当Registry registry = registryFactory.getRegistry(registryUrl);（自适应类），根据URL的协议加载multicast或别的实现
public class MulticastRegistryFactory extends AbstractRegistryFactory {
    //直接new对象，在AbstractRegistryFactory中有缓存，只会运行一次这个
    @Override
    public Registry createRegistry(URL url) {
        return new MulticastRegistry(url);
    }
}

//以消费端向注册中心订阅，列出服务提供方的信息
public class MulticastRegistry extends FailbackRegistry {
    private static final Logger logger = LoggerFactory.getLogger(MulticastRegistry.class);
    //组播协议默认端口
    private static final int DEFAULT_MULTICAST_PORT = 1234;
    private final InetAddress multicastAddress;
    private final MulticastSocket multicastSocket;
    private final int multicastPort;
    //
    private final ConcurrentMap<URL, Set<URL>> received = new ConcurrentHashMap<URL, Set<URL>>();
    private final ScheduledExecutorService cleanExecutor = Executors.newScheduledThreadPool(1, new NamedThreadFactory("DubboMulticastRegistryCleanTimer", true));
    private final ScheduledFuture<?> cleanFuture;
    private final int cleanPeriod;
    private volatile boolean admin = false;
    public MulticastRegistry(URL url) {
        super(url);
        if (url.isAnyHost()) {
            throw new IllegalStateException("registry address == null");
        }
        try {
            multicastAddress = InetAddress.getByName(url.getHost());
            checkMulticastAddress(multicastAddress);
            multicastPort = url.getPort() <= 0 ? DEFAULT_MULTICAST_PORT : url.getPort();
            //初始化组播
            multicastSocket = new MulticastSocket(multicastPort);
            //加入224.5.6.7:1234组，消费端和提供端在一个组中，可相互接受到信息
            NetUtils.joinMulticastGroup(multicastSocket, multicastAddress);
            Thread thread = new Thread(new Runnable() {
                @Override
                public void run() {
                    byte[] buf = new byte[2048];
                    DatagramPacket recv = new DatagramPacket(buf, buf.length);
                    while (!multicastSocket.isClosed()) {
                        try {
                            //接受消息，消费端和提供端发送组播信息后，加入该组播地址的消费端和提供端都会收到该信息
                            multicastSocket.receive(recv);
                            String msg = new String(recv.getData()).trim();
                            int i = msg.indexOf('\n');
                            if (i > 0) {
                                msg = msg.substring(0, i).trim();
                            }
                            //信息给该方法进行处理
                            MulticastRegistry.this.receive(msg, (InetSocketAddress) recv.getSocketAddress());
                            Arrays.fill(buf, (byte) 0);
                        } catch (Throwable e) {
                            if (!multicastSocket.isClosed()) {
                                logger.error(e.getMessage(), e);
                            }
                        }
                    }
                }
            }, "DubboMulticastRegistryReceiver");
            thread.setDaemon(true);
            thread.start();
        } catch (IOException e) {
            throw new IllegalStateException(e.getMessage(), e);
        }
        this.cleanPeriod = url.getParameter(Constants.SESSION_TIMEOUT_KEY, Constants.DEFAULT_SESSION_TIMEOUT);
        if (url.getParameter("clean", true)) {
            this.cleanFuture = cleanExecutor.scheduleWithFixedDelay(new Runnable() {
                @Override
                public void run() {
                    try {
                        clean(); // Remove the expired
                    } catch (Throwable t) { // Defensive fault tolerance
                        logger.error("Unexpected exception occur at clean expired provider, cause: " + t.getMessage(), t);
                    }
                }
            }, cleanPeriod, cleanPeriod, TimeUnit.MILLISECONDS);
        } else {
            this.cleanFuture = null;
        }
    }
    private void checkMulticastAddress(InetAddress multicastAddress) {
        if (!multicastAddress.isMulticastAddress()) {
            String message = "Invalid multicast address " + multicastAddress;
            if (!(multicastAddress instanceof Inet4Address)) {
                throw new IllegalArgumentException(message + ", " +
                        "ipv4 multicast address scope: 224.0.0.0 - 239.255.255.255.");
            } else {
                throw new IllegalArgumentException(message + ", " + "ipv6 multicast address must start with ff, " +
                        "for example: ff01::1");
            }
        }
    }
    private static boolean isMulticastAddress(String ip) {
        int i = ip.indexOf('.');
        if (i > 0) {
            String prefix = ip.substring(0, i);
            if (StringUtils.isInteger(prefix)) {
                int p = Integer.parseInt(prefix);
                return p >= 224 && p <= 239;
            }
        }
        return false;
    }
    private void clean() {
        if (admin) {
            for (Set<URL> providers : new HashSet<Set<URL>>(received.values())) {
                for (URL url : new HashSet<URL>(providers)) {
                    if (isExpired(url)) {
                        if (logger.isWarnEnabled()) {
                            logger.warn("Clean expired provider " + url);
                        }
                        doUnregister(url);
                    }
                }
            }
        }
    }
   
    private void receive(String msg, InetSocketAddress remoteAddress) {
        if (logger.isInfoEnabled()) {
            logger.info("Receive multicast message: " + msg + " from " + remoteAddress);
        }
        if (msg.startsWith(Constants.REGISTER)) {
            URL url = URL.valueOf(msg.substring(Constants.REGISTER.length()).trim());
            registered(url);
        } else if (msg.startsWith(Constants.UNREGISTER)) {
            URL url = URL.valueOf(msg.substring(Constants.UNREGISTER.length()).trim());
            unregistered(url);
        } else if (msg.startsWith(Constants.SUBSCRIBE)) {
            //消费者订阅时
            //consumer://10.6.1.250/com.alibaba.dubbo.examples.validation.api.ValidationService?application=validation-consumer&category=providers,configurators,routers&dubbo=2.0.2&interface=com.alibaba.dubbo.examples.validation.api.ValidationService&methods=save,update,delete&pid=244128&side=consumer&timestamp=1574755164222&validation=true
            URL url = URL.valueOf(msg.substring(Constants.SUBSCRIBE.length()).trim());
            //提供者的已注册数据
            // dubbo://10.6.1.250:20880/com.alibaba.dubbo.examples.validation.api.ValidationService?accesslog=true&anyhost=true&application=validation-provider&bean.name=com.alibaba.dubbo.examples.validation.api.ValidationService&compiler=jdk&dubbo=2.0.2&generic=false&interface=com.alibaba.dubbo.examples.validation.api.ValidationService&methods=save,update,delete&pid=154884&side=provider&timestamp=1574754854128
            Set<URL> urls = getRegistered();
            if (urls != null && !urls.isEmpty()) {
                for (URL u : urls) {
                    //url为consumer时，u为提供者时（如dubbo）
                    if (UrlUtils.isMatch(url, u)) {
                        //127.0.0.1 消费者和提供者在一台机器上时
                        String host = remoteAddress != null && remoteAddress.getAddress() != null
                                ? remoteAddress.getAddress().getHostAddress() : url.getIp();
                        //单播发送（消费者和提供者不在一台机器上并消费者机器只有一个消费者）
                        if (url.getParameter("unicast", true) // Whether the consumer's machine has only one process
                                && !NetUtils.getLocalHost().equals(host)) { // Multiple processes in the same machine cannot be unicast with unicast or there will be only one process receiving information
                            unicast(Constants.REGISTER + " " + u.toFullString(), host);
                        } else {
                           //组播发送 （消费者和提供者在一台机器上时）
                            broadcast(Constants.REGISTER + " " + u.toFullString());
                        }
                    }
                }
            }
        }/* else if (msg.startsWith(UNSUBSCRIBE)) {
        }*/
    }
    //组播发送（如224.5.6.7:1234组都能收到信息）
    private void broadcast(String msg) {
        if (logger.isInfoEnabled()) {
            logger.info("Send broadcast message: " + msg + " to " + multicastAddress + ":" + multicastPort);
        }
        try {
            byte[] data = (msg + "\n").getBytes();
            DatagramPacket hi = new DatagramPacket(data, data.length, multicastAddress, multicastPort);
            multicastSocket.send(hi);
        } catch (Exception e) {
            throw new IllegalStateException(e.getMessage(), e);
        }
    }
    //单播发送（指定host地址和端口）
    private void unicast(String msg, String host) {
        if (logger.isInfoEnabled()) {
            logger.info("Send unicast message: " + msg + " to " + host + ":" + multicastPort);
        }
        try {
            byte[] data = (msg + "\n").getBytes();
            DatagramPacket hi = new DatagramPacket(data, data.length, InetAddress.getByName(host), multicastPort);
            multicastSocket.send(hi);
        } catch (Exception e) {
            throw new IllegalStateException(e.getMessage(), e);
        }
    }
    
    //注册实现，组播发送 如 
    // 
    @Override
    protected void doRegister(URL url) {
        broadcast(Constants.REGISTER + " " + url.toFullString());
    }
    @Override
    protected void doUnregister(URL url) {
        broadcast(Constants.UNREGISTER + " " + url.toFullString());
    }
    //订阅实现，组播发送 如 
    //  subscribe consumer://10.6.1.250/com.alibaba.dubbo.examples.validation.api.ValidationService?application=validation-consumer&category=providers,configurators,routers&dubbo=2.0.2&interface=com.alibaba.dubbo.examples.validation.api.ValidationService&methods=save,update,delete&pid=244128&side=consumer&timestamp=1574755164222&validation=true
    @Override
    protected void doSubscribe(URL url, NotifyListener listener) {
        if (Constants.ANY_VALUE.equals(url.getServiceInterface())) {
            admin = true;
        }
        broadcast(Constants.SUBSCRIBE + " " + url.toFullString());
        synchronized (listener) {
            try {
                listener.wait(url.getParameter(Constants.TIMEOUT_KEY, Constants.DEFAULT_TIMEOUT));
            } catch (InterruptedException e) {
            }
        }
    }
    ...
    protected void registered(URL url) {
        for (Map.Entry<URL, Set<NotifyListener>> entry : getSubscribed().entrySet()) {
            URL key = entry.getKey();
            if (UrlUtils.isMatch(key, url)) {
                Set<URL> urls = received.get(key);
                if (urls == null) {
                    received.putIfAbsent(key, new ConcurrentHashSet<URL>());
                    urls = received.get(key);
                }
                urls.add(url);
                List<URL> list = toList(urls);
                for (NotifyListener listener : entry.getValue()) {
                    notify(key, listener, list);
                    synchronized (listener) {
                        listener.notify();
                    }
                }
            }
        }
    }
   ...
}

```


## 3.4 dubbo注册中心-Redis实现分析

- Redis介绍

 ![duddo中Redis注册中心](pic/dubbo-Registry-Redis.png)
 

```aidl
使用 Redis 的 Key/Map 结构存储数据结构：
    主 Key 为服务名和类型
    Map 中的 Key 为 URL 地址
    Map 中的 Value 为过期时间，用于判断脏数据，脏数据由监控中心删除 [3]
使用 Redis 的 Publish/Subscribe 事件通知数据变更：
    通过事件的值区分事件类型：register, unregister, subscribe, unsubscribe
    普通消费者直接订阅指定服务提供者的 Key，只会收到指定服务的 register, unregister 事件
    监控中心通过 psubscribe 功能订阅 /dubbo/*，会收到所有服务的所有变更事件
调用过程：
    服务提供方启动时，向 Key:/dubbo/com.foo.BarService/providers 下，添加当前提供者的地址
    并向 Channel:/dubbo/com.foo.BarService/providers 发送 register 事件
    服务消费方启动时，从 Channel:/dubbo/com.foo.BarService/providers 订阅 register 和 unregister 事件
    并向 Key:/dubbo/com.foo.BarService/consumers 下，添加当前消费者的地址
    服务消费方收到 register 和 unregister 事件后，从 Key:/dubbo/com.foo.BarService/providers 下获取提供者地址列表
    服务监控中心启动时，从 Channel:/dubbo/* 订阅 register 和 unregister，以及 subscribe 和unsubsribe事件
    服务监控中心收到 register 和 unregister 事件后，从 Key:/dubbo/com.foo.BarService/providers 下获取提供者地址列表
    服务监控中心收到 subscribe 和 unsubsribe 事件后，从 Key:/dubbo/com.foo.BarService/consumers 下获取消费者地址列表
    
    

```



# 四、其他
