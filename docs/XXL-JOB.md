* [一、项目概览](#一数据类型)
    * [1.1 简介](#11-简介) 
    * [1.2 环境](#12-环境)
    * [1.3 源码及官网](#13-源码及官网)
* [二、项目使用](#二项目使用)
* [三、项目设计](#三项目设计)
    * [3.1 总体设计](#31-总体设计)
    * [3.2 关键点分析](#32-关键点分析)
        * [3.2.1 调度中心和执行器怎么通讯](#321-调度中心和执行器怎么通讯)  
        * [3.2.2 调度中心怎么按时调度任务](#322-调度中心怎么按时调度任务)  
        * [3.2.3 调度中心和执行器执行任务的整个流程](#323-调度中心和执行器执行任务的整个流程)  
        * [3.2.4 调度中心各种路由策略实现原理](#324-调度中心各种路由策略实现原理) 
        * [3.2.5 调度中心页面中查看执行日志如何实现Rolling实时日志](#325-调度中心页面中查看执行日志如何实现Rolling实时日志)
        * [3.2.6 GLUE模式(Java)任务如何实现](#326-GLUE模式Java任务如何实现)
* [四、其他](#四其他)

# 一、项目概览

## 1.1 简介
XXL-JOB是一个轻量级分布式任务调度平台，其核心设计目标是开发迅速、学习简单、轻量级、易扩展。现已开放源代码并接入多家公司线上产品线，开箱即用。

## 1.2 环境
```aidl
Maven3+
Jdk1.7+
Mysql5.6+

源码版本： 2.0.1(主) 和 2.1.0

```
## 1.3 源码及官网

[github源码](https://github.com/xuxueli/xxl-job/)

[官网](http://www.xuxueli.com/xxl-job)

# 二、项目使用
分为调度中心管理页面和执行器工程
- 调度中心管理页面
![调度中心管理页面](pic/XXL-JOB-admin.png)
默认用户admin/admin,配置执行器和任务（执行器名称和任务名称在执行器工程中配置）

- 执行器工程（如集成到springboot）

```aidl
application.properties 添加配置如下

### xxl-job admin address list, such as "http://address" or "http://address01,http://address02"
xxl.job.admin.addresses=http://127.0.0.1:8080/xxl-job-admin
### xxl-job executor address
xxl.job.executor.appname=xxl-job-executor-sample
xxl.job.executor.ip=
xxl.job.executor.port=9998
### xxl-job, access token
xxl.job.accessToken=
### xxl-job log path
xxl.job.executor.logpath=D:/xxl-job/jobhandler
### xxl-job log retention days
xxl.job.executor.logretentiondays=-1

pom文件引入依赖

<!-- xxl-job-core -->
<dependency>
    <groupId>com.xuxueli</groupId>
    <artifactId>xxl-job-core</artifactId>
    <version>${project.parent.version}</version>
</dependency>

配置springboot XxlJobSpringExecutor.
@Bean(initMethod = "start", destroyMethod = "destroy")
public XxlJobSpringExecutor xxlJobExecutor() {
    logger.info(">>>>>>>>>>> xxl-job config init.");
    XxlJobSpringExecutor xxlJobSpringExecutor = new XxlJobSpringExecutor();
    xxlJobSpringExecutor.setAdminAddresses(adminAddresses);
    xxlJobSpringExecutor.setAppName(appName);
    xxlJobSpringExecutor.setIp(ip);
    xxlJobSpringExecutor.setPort(port);
    xxlJobSpringExecutor.setAccessToken(accessToken);
    xxlJobSpringExecutor.setLogPath(logPath);
    xxlJobSpringExecutor.setLogRetentionDays(logRetentionDays);
    return xxlJobSpringExecutor;
}

编写具体定时任务IJobHandler（demoJobHandler为名称，手动添加到调度中心管理页面的任务管理中）
@JobHandler(value="demoJobHandler")
@Component
public class DemoJobHandler extends IJobHandler {
	@Override
	public ReturnT<String> execute(String param) throws Exception {
		XxlJobLogger.log("XXL-JOB, Hello World.");
		for (int i = 0; i < 5; i++) {
			XxlJobLogger.log("beat at:" + i);
			TimeUnit.SECONDS.sleep(2);
		}
		return SUCCESS;
	}
}
```

# 三、项目设计

## 3.1 总体设计
![总体架构图](pic/XXL-JOB-OverALL.png)

* 分为调度中心和执行器两个部分
* 调度中心管理定时任务，调度定时任务，和页面显示等
* 执行器主要用来执行定时任务

## 3.2 关键点分析
### 3.2.1 调度中心和执行器怎么通讯
通过XXL-RPC框架进行RPC通讯，先看下执行器访问调度中心的方式
- 执行器配置调度中心地址 http://127.0.0.1:8080/xxl-job-admin , 初始化链接spring boot方式
```
    在springboot配置XxlJobSpringExecutor这个bean进行初始化链接
    @Bean(initMethod = "start", destroyMethod = "destroy")
    public XxlJobSpringExecutor xxlJobExecutor() {
        logger.info(">>>>>>>>>>> xxl-job config init.");
        XxlJobSpringExecutor xxlJobSpringExecutor = new XxlJobSpringExecutor(); 
        xxlJobSpringExecutor.setAdminAddresses(adminAddresses);
        xxlJobSpringExecutor.setAppName(appName);
        xxlJobSpringExecutor.setIp(ip);
        xxlJobSpringExecutor.setPort(port);
        xxlJobSpringExecutor.setAccessToken(accessToken);
        xxlJobSpringExecutor.setLogPath(logPath);
        xxlJobSpringExecutor.setLogRetentionDays(logRetentionDays);
        return xxlJobSpringExecutor;
    }
    XxlJobSpringExecutor中的start方法（支持多个调度中心），初始化调用调度中心的接口AdminBiz
        private void initAdminBizList(String adminAddresses, String accessToken) throws Exception {
            if (adminAddresses!=null && adminAddresses.trim().length()>0) {
                for (String address: adminAddresses.trim().split(",")) {
                    if (address!=null && address.trim().length()>0) { 
                        String addressUrl = address.concat(AdminBiz.MAPPING);   
                        AdminBiz adminBiz = (AdminBiz) new XxlRpcReferenceBean(NetEnum.JETTY, Serializer.SerializeEnum.HESSIAN.getSerializer(), CallType.SYNC,
                                AdminBiz.class, null, 10000, addressUrl, accessToken, null).getObject();    
                        if (adminBizList == null) {
                            adminBizList = new ArrayList<AdminBiz>();
                        }
                        adminBizList.add(adminBiz); } }}
  ```  
   
- XxlRpcReferenceBean的getObject方法是个动态代理方法，根据传入的参数实例化对象，NetEnum.JETTY标识为JETTY方式的HTTP协议链接
 
 ``` 
public Object getObject() {
return Proxy.newProxyInstance(Thread.currentThread()
    .getContextClassLoader(), new Class[] { iface },
    new InvocationHandler() {
        @Override
        public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
            String className = method.getDeclaringClass().getName();
            // filter method like "Object.toString()"
            if (Object.class.getName().equals(className)) {
                logger.info(">>>>>>>>>>> xxl-rpc proxy class-method not support [{}.{}]", className, method.getName());
                throw new XxlRpcException("xxl-rpc proxy class-method not support");
            }    
            // address
            String address = routeAddress();
            if (address==null || address.trim().length()==0) {
                throw new XxlRpcException("xxl-rpc reference bean["+ className +"] address empty");
            }   
            // request
            XxlRpcRequest xxlRpcRequest = new XxlRpcRequest();
            xxlRpcRequest.setRequestId(UUID.randomUUID().toString());
            xxlRpcRequest.setCreateMillisTime(System.currentTimeMillis());
            xxlRpcRequest.setAccessToken(accessToken);
            xxlRpcRequest.setClassName(className);
            xxlRpcRequest.setMethodName(method.getName());
            xxlRpcRequest.setParameterTypes(method.getParameterTypes());
            xxlRpcRequest.setParameters(args);  	                    
            // send
            if (CallType.SYNC == callType) {
                try {
                    // future set
                    XxlRpcFutureResponse futureResponse = new XxlRpcFutureResponse(xxlRpcRequest, null);

                    // do invoke
                    client.asyncSend(address, xxlRpcRequest);

                    // future get
                    XxlRpcResponse xxlRpcResponse = futureResponse.get(timeout, TimeUnit.MILLISECONDS);
                    if (xxlRpcResponse.getErrorMsg() != null) {
                        throw new XxlRpcException(xxlRpcResponse.getErrorMsg());
                    }
                    return xxlRpcResponse.getResult();
                } catch (Exception e) {
                    logger.info(">>>>>>>>>>> xxl-job, invoke error, address:{}, XxlRpcRequest{}", address, xxlRpcRequest);

                    throw (e instanceof XxlRpcException)?e:new XxlRpcException(e);
                } finally{
                    // remove-InvokerFuture
                    XxlRpcFutureResponseFactory.removeInvokerFuture(xxlRpcRequest.getRequestId());
                }
            } else if (CallType.FUTURE == callType) {

                // thread future set
                XxlRpcInvokeFuture invokeFuture = null;
                try {
                    // future set
                    invokeFuture = new XxlRpcInvokeFuture(new XxlRpcFutureResponse(xxlRpcRequest, null));
                    XxlRpcInvokeFuture.setFuture(invokeFuture);

                    // do invoke
                    client.asyncSend(address, xxlRpcRequest);

                    return null;
                } catch (Exception e) {
                    logger.info(">>>>>>>>>>> xxl-job, invoke error, address:{}, XxlRpcRequest{}", address, xxlRpcRequest);

                    // remove-InvokerFuture
                    invokeFuture.stop();

                    throw (e instanceof XxlRpcException)?e:new XxlRpcException(e);
                }

            } else if (CallType.CALLBACK == callType) {

                // get callback
                XxlRpcInvokeCallback finalInvokeCallback = invokeCallback;
                XxlRpcInvokeCallback threadInvokeCallback = XxlRpcInvokeCallback.getCallback();
                if (threadInvokeCallback != null) {
                    finalInvokeCallback = threadInvokeCallback;
                }
                if (finalInvokeCallback == null) {
                    throw new XxlRpcException("xxl-rpc XxlRpcInvokeCallback（CallType="+ CallType.CALLBACK.name() +"） cannot be null.");
                }

                try {
                    // future set
                    XxlRpcFutureResponse futureResponse = new XxlRpcFutureResponse(xxlRpcRequest, finalInvokeCallback);

                    client.asyncSend(address, xxlRpcRequest);
                } catch (Exception e) {
                    logger.info(">>>>>>>>>>> xxl-job, invoke error, address:{}, XxlRpcRequest{}", address, xxlRpcRequest);

                    // future remove
                    XxlRpcFutureResponseFactory.removeInvokerFuture(xxlRpcRequest.getRequestId());

                    throw (e instanceof XxlRpcException)?e:new XxlRpcException(e);
                }

                return null;
            } else if (CallType.ONEWAY == callType) {
                client.asyncSend(address, xxlRpcRequest);
                return null;
            } else {
                throw new XxlRpcException("xxl-rpc callType["+ callType +"] invalid");
            }

        }
    });
}
  
 ``` 

- 通过client.asyncSend向调度中心发送请求，NetEnum.JETTY具体实现为
```
private void postRequestAsync(String address, XxlRpcRequest xxlRpcRequest) throws Exception {
    // reqURL,请求调度中心的地址为http://127.0.0.1:8080/xxl-job-admin/api
    String reqURL = address; 
    if (!address.toLowerCase().startsWith("http")) {
        reqURL = "http://" + address;	// IP:PORT, need parse to url
    }
    // serialize request 请求前先序列化对象对象xxlRpcRequest
    byte[] requestBytes = xxlRpcReferenceBean.getSerializer().serialize(xxlRpcRequest);
     // httpclient
     HttpClient httpClient = getJettyHttpClient();
     // request
     Request request = httpClient.newRequest(reqURL);
     request.method(HttpMethod.POST);
     request.timeout(xxlRpcReferenceBean.getTimeout() + 500, TimeUnit.MILLISECONDS);		// async, not need timeout
     request.content(new BytesContentProvider(requestBytes));
     // invoke
     request.send(new BufferingResponseListener() {
        @Override
        public void onComplete(Result result) {
            try {
                 // valid status
                 if (result.isFailed()) {
                     throw new XxlRpcException(result.getResponseFailure());
                 }
                 // valid HttpStatus
                 if (result.getResponse().getStatus() != HttpStatus.OK_200) {
                     throw new XxlRpcException("xxl-rpc remoting request fail, http HttpStatus["+ result.getResponse().getStatus() +"] invalid.");
                 }

                 // valid response bytes
                 byte[] responseBytes = getContent();
                 if (responseBytes == null || responseBytes.length==0) {
                     throw new XxlRpcException("xxl-rpc remoting request fail, response bytes is empty.");
                 }
                 // deserialize response反序列化得到XxlRpcResponse对象
                 XxlRpcResponse xxlRpcResponse = (XxlRpcResponse) xxlRpcReferenceBean.getSerializer().deserialize(responseBytes, XxlRpcResponse.class);
                 // notify response 响应结果
                XxlRpcFutureResponseFactory.notifyInvokerFuture(xxlRpcResponse.getRequestId(), xxlRpcResponse);
             } catch (Exception e){
                // fail, request finish, remove request
                if (result.getRequest().getContent() instanceof BytesContentProvider) {
                    try {
                         BytesContentProvider requestCp = (BytesContentProvider) result.getRequest().getContent();
                         XxlRpcRequest requestTmp = (XxlRpcRequest) xxlRpcReferenceBean.getSerializer().deserialize(requestCp.iterator().next().array(), XxlRpcRequest.class);
                        // error msg
                        String errorMsg = null;
                        if (e instanceof XxlRpcException) {
                            XxlRpcException rpcException = (XxlRpcException) e;
                            if (rpcException.getCause() != null) {
                                errorMsg = ThrowableUtil.toString(rpcException.getCause());
                            } else {
                                errorMsg = rpcException.getMessage();
                            }
                        } else {
                            errorMsg = ThrowableUtil.toString(e);
                        }
                        //  make response
                        XxlRpcResponse xxlRpcResponse = new XxlRpcResponse();
                        xxlRpcResponse.setRequestId(requestTmp.getRequestId());
                        xxlRpcResponse.setErrorMsg(errorMsg);
                        // notify response
                        XxlRpcFutureResponseFactory.notifyInvokerFuture(xxlRpcResponse.getRequestId(), xxlRpcResponse);
                     } catch (Exception e2) {
                         logger.info(">>>>>>>>>>> xxl-rpc, remoting request error, and callback error: " + e2.getMessage());
                        logger.info(e.getMessage(), e);
                     }
                 } else {
                    logger.info(">>>>>>>>>>> xxl-rpc, remoting request error.", e);
                }
             }
        }
    });
 }   
```    
    
- AdminBiz接口是在调度中心实现的，在执行器通过rpc远程调用调度中的实现的方法 
```    
    public interface AdminBiz {   
        public static final String MAPPING = "/api";
        public ReturnT<String> callback(List<HandleCallbackParam> callbackParamList);
        public ReturnT<String> registry(RegistryParam registryParam);
        public ReturnT<String> registryRemove(RegistryParam registryParam);    
    }
```
- 调度中心中提供AdminBiz.MAPPING = /api接口,用于执行器远程访问
```  
@Controller
public class JobApiController implements InitializingBean {
    @Override
    public void afterPropertiesSet() throws Exception {

    }
    @RequestMapping(AdminBiz.MAPPING)
    @PermessionLimit(limit=false)
    public void api(HttpServletRequest request, HttpServletResponse response) throws IOException, ServletException {
        XxlJobDynamicScheduler.invokeAdminService(request, response);
    }
}
调度中心实例化JettyServerHandler服务用于处理执行器的请求
private static JettyServerHandler jettyServerHandler;
private void initRpcProvider(){
    // init
    XxlRpcProviderFactory xxlRpcProviderFactory = new XxlRpcProviderFactory();
    xxlRpcProviderFactory.initConfig(NetEnum.JETTY, Serializer.SerializeEnum.HESSIAN.getSerializer(), null, 0, XxlJobAdminConfig.getAdminConfig().getAccessToken(), null, null);
    // add services 增加AdminBiz接口及实现类到XxlRpcProviderFactory中的serviceData = new HashMap<String, Object>();
    xxlRpcProviderFactory.addService(AdminBiz.class.getName(), null, XxlJobAdminConfig.getAdminConfig().getAdminBiz());
    // jetty handler
    jettyServerHandler = new JettyServerHandler(xxlRpcProviderFactory);
}
public static void invokeAdminService(HttpServletRequest request, HttpServletResponse response) throws IOException, ServletException {
    jettyServerHandler.handle(null, new Request(null, null), request, response);
}

invokeAdminService真正调用具体的实现类，并返回结果
@Override
public void handle(String target, Request baseRequest, HttpServletRequest request, HttpServletResponse response) throws IOException, ServletException {
    if ("/services".equals(target)) {	// services mapping
        StringBuffer stringBuffer = new StringBuffer("<ui>");
        for (String serviceKey: xxlRpcProviderFactory.getServiceData().keySet()) {
            stringBuffer.append("<li>").append(serviceKey).append(": ").append(xxlRpcProviderFactory.getServiceData().get(serviceKey)).append("</li>");
        }
        stringBuffer.append("</ui>");
        writeResponse(baseRequest, response, stringBuffer.toString().getBytes());
        return;
    } else {	// default remoting mapping
        // request parse
        XxlRpcRequest xxlRpcRequest = null;
        try {
            xxlRpcRequest = parseRequest(request); //从request对象的参数中等到XxlRpcRequest对象
        } catch (Exception e) {
            writeResponse(baseRequest, response, ThrowableUtil.toString(e).getBytes());
            return;
        }
        // invoke 调用具体方法，并封装返回结果到XxlRpcResponse
        XxlRpcResponse xxlRpcResponse = xxlRpcProviderFactory.invokeService(xxlRpcRequest);
        // response-serialize + response-write
        byte[] responseBytes = xxlRpcProviderFactory.getSerializer().serialize(xxlRpcResponse);
        writeResponse(baseRequest, response, responseBytes);
    }
}

通过反序列化方式得到XxlRpcRequest对象
private XxlRpcRequest parseRequest(HttpServletRequest request) throws Exception {
    // deserialize request
    byte[] requestBytes = readBytes(request);//从request读取全部的字节
    if (requestBytes == null || requestBytes.length==0) {
        throw new XxlRpcException("xxl-rpc request data is empty.");
    }
    XxlRpcRequest rpcXxlRpcRequest = (XxlRpcRequest) xxlRpcProviderFactory.getSerializer().deserialize(requestBytes, XxlRpcRequest.class);
    return rpcXxlRpcRequest;
}
通过反射方式调用xxlRpcRequest中的请求方法
public XxlRpcResponse invokeService(XxlRpcRequest xxlRpcRequest) {

    //  make response
    XxlRpcResponse xxlRpcResponse = new XxlRpcResponse();
    xxlRpcResponse.setRequestId(xxlRpcRequest.getRequestId());

    // match service bean
    String serviceKey = makeServiceKey(xxlRpcRequest.getClassName(), xxlRpcRequest.getVersion());
    //通过缓存类serviceData获取已实例化的对象
    Object serviceBean = serviceData.get(serviceKey);

    // valid
    if (serviceBean == null) {
        xxlRpcResponse.setErrorMsg("The serviceKey["+ serviceKey +"] not found.");
        return xxlRpcResponse;
    }

    if (System.currentTimeMillis() - xxlRpcRequest.getCreateMillisTime() > 3*60*1000) {
        xxlRpcResponse.setErrorMsg("The timestamp difference between admin and executor exceeds the limit.");
        return xxlRpcResponse;
    }
    if (accessToken!=null && accessToken.trim().length()>0 && !accessToken.trim().equals(xxlRpcRequest.getAccessToken())) {
        xxlRpcResponse.setErrorMsg("The access token[" + xxlRpcRequest.getAccessToken() + "] is wrong.");
        return xxlRpcResponse;
    }

    // invoke
    try {
        Class<?> serviceClass = serviceBean.getClass();
        String methodName = xxlRpcRequest.getMethodName();
        Class<?>[] parameterTypes = xxlRpcRequest.getParameterTypes();
        Object[] parameters = xxlRpcRequest.getParameters();

        Method method = serviceClass.getMethod(methodName, parameterTypes);
        method.setAccessible(true);
        Object result = method.invoke(serviceBean, parameters);

        /*FastClass serviceFastClass = FastClass.create(serviceClass);
        FastMethod serviceFastMethod = serviceFastClass.getMethod(methodName, parameterTypes);
        Object result = serviceFastMethod.invoke(serviceBean, parameters);*/

        xxlRpcResponse.setResult(result);
    } catch (Throwable t) {
        logger.error("xxl-rpc provider invokeService error.", t);
        xxlRpcResponse.setErrorMsg(ThrowableUtil.toString(t));
    }

    return xxlRpcResponse;
}


```

再看调度中心访问执行器的方式，也是XXL-RPC方式的JETTY协议

在springboot配置XxlJobSpringExecutor这个bean进行初始化链接
```  
    @Bean(initMethod = "start", destroyMethod = "destroy")
    public XxlJobSpringExecutor xxlJobExecutor() {
        logger.info(">>>>>>>>>>> xxl-job config init.");
        XxlJobSpringExecutor xxlJobSpringExecutor = new XxlJobSpringExecutor(); 
        xxlJobSpringExecutor.setAdminAddresses(adminAddresses);
        xxlJobSpringExecutor.setAppName(appName);
        xxlJobSpringExecutor.setIp(ip);
        xxlJobSpringExecutor.setPort(port);
        xxlJobSpringExecutor.setAccessToken(accessToken);
        xxlJobSpringExecutor.setLogPath(logPath);
        xxlJobSpringExecutor.setLogRetentionDays(logRetentionDays);
        return xxlJobSpringExecutor;
    }
 ```   
- 执行器提供JETTY服务端，默认端口为9888，XxlJobSpringExecutor的start中初始化xxlRpcProviderFactory
```aidl
    private void initRpcProvider(String ip, int port, String appName, String accessToken) throws Exception {
        // init invoker factory
        xxlRpcInvokerFactory = new XxlRpcInvokerFactory();
        // init, provider factory
        String address = IpUtil.getIpPort(ip, port);
        Map<String, String> serviceRegistryParam = new HashMap<String, String>();
        serviceRegistryParam.put("appName", appName);//当前执行器名称，同一组执行器一个名称
        serviceRegistryParam.put("address", address);//执行器IP和端口
        xxlRpcProviderFactory = new XxlRpcProviderFactory();
        xxlRpcProviderFactory.initConfig(NetEnum.JETTY, Serializer.SerializeEnum.HESSIAN.getSerializer(), ip, port, accessToken, ExecutorServiceRegistry.class, serviceRegistryParam);
        // add services 增加调度中心远程访问执行器的方法ExecutorBiz
        xxlRpcProviderFactory.addService(ExecutorBiz.class.getName(), null, new ExecutorBizImpl());
        // start
       xxlRpcProviderFactory.start();
    }
    ExecutorServiceRegistry中调用了
        ExecutorRegistryThread.getInstance().start(param.get("appName"), param.get("address"));
    ExecutorRegistryThread启动一个守护线程用于向调用中心不停的注册，调用的方法是ReturnT<String> registryResult = adminBiz.registry(registryParam);            
```
- 调度中心registry实现方法，把注册来的执行器的名称和地址存储
```
    @Override
    public ReturnT<String> registry(RegistryParam registryParam) {
        int ret = xxlJobRegistryDao.registryUpdate(registryParam.getRegistGroup(), registryParam.getRegistryKey(), registryParam.getRegistryValue());
        if (ret < 1) {
            xxlJobRegistryDao.registrySave(registryParam.getRegistGroup(), registryParam.getRegistryKey(), registryParam.getRegistryValue());
        }
        return ReturnT.SUCCESS;
    }
```
- 调度中心调用执行的方法根据address获取ExecutorBiz，调用ExecutorBiz的方法（还是动态代理方式）
```
    public static ExecutorBiz getExecutorBiz(String address) throws Exception {
        // valid
        if (address==null || address.trim().length()==0) {
            return null;
        }

        // load-cache
        address = address.trim();
        ExecutorBiz executorBiz = executorBizRepository.get(address);
        if (executorBiz != null) {
            return executorBiz;
        }

        // set-cache
        executorBiz = (ExecutorBiz) new XxlRpcReferenceBean(NetEnum.JETTY, Serializer.SerializeEnum.HESSIAN.getSerializer(), CallType.SYNC,
                ExecutorBiz.class, null, 10000, address, XxlJobAdminConfig.getAdminConfig().getAccessToken(), null).getObject();

        executorBizRepository.put(address, executorBiz);
        return executorBiz;
    }
```
 
### 3.2.2 调度中心怎么按时调度任务

2.0.1版本时依赖quartz调度

- 调度中心集成quartz
```aidl

配置 quartz.properties 
org.quartz.scheduler.instanceName: DefaultQuartzScheduler
org.quartz.scheduler.instanceId: AUTO
org.quartz.scheduler.rmi.export: false
org.quartz.scheduler.rmi.proxy: false
org.quartz.scheduler.wrapJobExecutionInUserTransaction: false
org.quartz.threadPool.class: org.quartz.simpl.SimpleThreadPool
org.quartz.threadPool.threadCount: 50
org.quartz.threadPool.threadPriority: 5
org.quartz.threadPool.threadsInheritContextClassLoaderOfInitializingThread: true
org.quartz.jobStore.misfireThreshold: 60000
org.quartz.jobStore.maxMisfiresToHandleAtATime: 1
# for cluster
org.quartz.jobStore.tablePrefix: XXL_JOB_QRTZ_
org.quartz.jobStore.class: org.quartz.impl.jdbcjobstore.JobStoreTX
org.quartz.jobStore.isClustered: true
org.quartz.jobStore.clusterCheckinInterval: 5000

配置 quartz的SchedulerFactoryBean 
@Configuration
public class XxlJobDynamicSchedulerConfig {

    @Bean
    public SchedulerFactoryBean getSchedulerFactoryBean(DataSource dataSource){

        SchedulerFactoryBean schedulerFactory = new SchedulerFactoryBean();
        schedulerFactory.setDataSource(dataSource);
        schedulerFactory.setAutoStartup(true);                  // 自动启动
        schedulerFactory.setStartupDelay(20);                   // 延时启动，应用启动成功后在启动
        schedulerFactory.setOverwriteExistingJobs(true);        // 覆盖DB中JOB：true、以数据库中已经存在的为准：false
        schedulerFactory.setApplicationContextSchedulerContextKey("applicationContext");
        schedulerFactory.setConfigLocation(new ClassPathResource("quartz.properties"));

        return schedulerFactory;
    }

    @Bean(initMethod = "start", destroyMethod = "destroy")
    public XxlJobDynamicScheduler getXxlJobDynamicScheduler(SchedulerFactoryBean schedulerFactory){

        Scheduler scheduler = schedulerFactory.getScheduler();

        XxlJobDynamicScheduler xxlJobDynamicScheduler = new XxlJobDynamicScheduler();
        xxlJobDynamicScheduler.setScheduler(scheduler);

        return xxlJobDynamicScheduler;
    }

}

```
添加启动定时任务
```aidl
@Override
public ReturnT<String> start(int id) {
    XxlJobInfo xxlJobInfo = xxlJobInfoDao.loadById(id);
    String group = String.valueOf(xxlJobInfo.getJobGroup());
    String name = String.valueOf(xxlJobInfo.getId());//任务id
    String cronExpression = xxlJobInfo.getJobCron();//任务 时间表达式

    try {
        boolean ret = XxlJobDynamicScheduler.addJob(name, group, cronExpression);
        return ret?ReturnT.SUCCESS:ReturnT.FAIL;
    } catch (SchedulerException e) {
        logger.error(e.getMessage(), e);
        return ReturnT.FAIL;
    }
}

public static boolean addJob(String jobName, String jobGroup, String cronExpression) throws SchedulerException {
    	// 1、job key
        TriggerKey triggerKey = TriggerKey.triggerKey(jobName, jobGroup);
        JobKey jobKey = new JobKey(jobName, jobGroup);

        // 2、valid
        if (scheduler.checkExists(triggerKey)) {
            return true;    // PASS
        }

        // 3、corn trigger
        CronScheduleBuilder cronScheduleBuilder = CronScheduleBuilder.cronSchedule(cronExpression).withMisfireHandlingInstructionDoNothing();   // withMisfireHandlingInstructionDoNothing 忽略掉调度终止过程中忽略的调度
        CronTrigger cronTrigger = TriggerBuilder.newTrigger().withIdentity(triggerKey).withSchedule(cronScheduleBuilder).build();

        // 4、job detail
		Class<? extends Job> jobClass_ = RemoteHttpJobBean.class;   // Class.forName(jobInfo.getJobClass());
		JobDetail jobDetail = JobBuilder.newJob(jobClass_).withIdentity(jobKey).build();

        /*if (jobInfo.getJobData()!=null) {
        	JobDataMap jobDataMap = jobDetail.getJobDataMap();
        	jobDataMap.putAll(JacksonUtil.readValue(jobInfo.getJobData(), Map.class));	
        	// JobExecutionContext context.getMergedJobDataMap().get("mailGuid");
		}*/
        
        // 5、schedule job
        Date date = scheduler.scheduleJob(jobDetail, cronTrigger);

        logger.info(">>>>>>>>>>> addJob success, jobDetail:{}, cronTrigger:{}, date:{}", jobDetail, cronTrigger, date);
        return true;
    }

```
quartz到时间后执行定时任务

```aidl
public class RemoteHttpJobBean extends QuartzJobBean {
	private static Logger logger = LoggerFactory.getLogger(RemoteHttpJobBean.class);
	@Override
	protected void executeInternal(JobExecutionContext context)
			throws JobExecutionException {
		// load jobId
		JobKey jobKey = context.getTrigger().getJobKey();
		Integer jobId = Integer.valueOf(jobKey.getName());
		// trigger
		JobTriggerPoolHelper.trigger(jobId, TriggerTypeEnum.CRON, -1, null, null);
	}
}

JobTriggerPoolHelper.trigger后调用下面的代码，线程池执行一个任务
public static void trigger(int jobId, TriggerTypeEnum triggerType, int failRetryCount, String executorShardingParam, String executorParam) {
     helper.addTrigger(jobId, triggerType, failRetryCount, executorShardingParam, executorParam);
}

 public void addTrigger(final int jobId, final TriggerTypeEnum triggerType, final int failRetryCount, final String executorShardingParam, final String executorParam) {
     triggerPool.execute(new Runnable() {
         @Override
         public void run() {
             XxlJobTrigger.trigger(jobId, triggerType, failRetryCount, executorShardingParam, executorParam);
         }
     });
 }
private ThreadPoolExecutor triggerPool = new ThreadPoolExecutor(32,256, 60L,TimeUnit.SECONDS,
      new LinkedBlockingQueue<Runnable>(1000));
XxlJobTrigger.trigger最后执行下面代码，访问执行器工程的RPC接口，启动任务
ExecutorBiz executorBiz = XxlJobDynamicScheduler.getExecutorBiz(address);
runResult = executorBiz.run(triggerParam);
```

2.0.1版本时自己实现任务调度

- 添加启动定时任务

```aidl
	@Override
	public ReturnT<String> start(int id) {
		XxlJobInfo xxlJobInfo = xxlJobInfoDao.loadById(id);

		// next trigger time (10s后生效，避开预读周期)
		long nextTriggerTime = 0;
		try {
			nextTriggerTime = new CronExpression(xxlJobInfo.getJobCron()).getNextValidTimeAfter(new Date(System.currentTimeMillis() + 10000)).getTime();
		} catch (ParseException e) {
			logger.error(e.getMessage(), e);
			return new ReturnT<String>(ReturnT.FAIL_CODE, I18nUtil.getString("jobinfo_field_cron_unvalid")+" | "+ e.getMessage());
		}

		xxlJobInfo.setTriggerStatus(1);
		xxlJobInfo.setTriggerLastTime(0);
		xxlJobInfo.setTriggerNextTime(nextTriggerTime); //给任务设置下次开始运行的时间
		xxlJobInfoDao.update(xxlJobInfo);
		return ReturnT.SUCCESS;
	}

启动两个线程进行定时任务触发	
public class JobScheduleHelper {
    private static Logger logger = LoggerFactory.getLogger(JobScheduleHelper.class);

    private static JobScheduleHelper instance = new JobScheduleHelper();
    public static JobScheduleHelper getInstance(){
        return instance;
    }

    private Thread scheduleThread;
    private Thread ringThread;
    private volatile boolean toStop = false;
    private volatile static Map<Integer, List<Integer>> ringData = new ConcurrentHashMap<>();

    public void start(){

        // schedule thread
        scheduleThread = new Thread(new Runnable() {
            @Override
            public void run() {

                try {
                    TimeUnit.MILLISECONDS.sleep(5000 - System.currentTimeMillis()%1000 );
                } catch (InterruptedException e) {
                    if (!toStop) {
                        logger.error(e.getMessage(), e);
                    }
                }
                logger.info(">>>>>>>>> init xxl-job admin scheduler success.");

                while (!toStop) {

                    // 扫描任务
                    long start = System.currentTimeMillis();
                    Connection conn = null;
                    PreparedStatement preparedStatement = null;
                    try {
                        if (conn==null || conn.isClosed()) {
                            conn = XxlJobAdminConfig.getAdminConfig().getDataSource().getConnection();
                        }
                        conn.setAutoCommit(false);
                        //使用数据库中的锁，多个进程时，只有一个进程能触发获取定时任务
                        preparedStatement = conn.prepareStatement(  "select * from xxl_job_lock where lock_name = 'schedule_lock' for update" );
                        preparedStatement.execute();

                        // tx start

                        // 1、预读10s内调度任务
                        long maxNextTime = System.currentTimeMillis() + 10000;
                        long nowTime = System.currentTimeMillis();
                        //获取最近10秒，要运行的任务
                        List<XxlJobInfo> scheduleList = XxlJobAdminConfig.getAdminConfig().getXxlJobInfoDao().scheduleJobQuery(maxNextTime);
                        if (scheduleList!=null && scheduleList.size()>0) {
                            // 2、推送时间轮
                            for (XxlJobInfo jobInfo: scheduleList) {

                                // 时间轮刻度计算
                                int ringSecond = -1;
                                if (jobInfo.getTriggerNextTime() < nowTime - 10000) {   // 过期超10s：本地忽略，当前时间开始计算下次触发时间
                                    ringSecond = -1;

                                    jobInfo.setTriggerLastTime(jobInfo.getTriggerNextTime());
                                    jobInfo.setTriggerNextTime(
                                            new CronExpression(jobInfo.getJobCron())
                                                    .getNextValidTimeAfter(new Date())
                                                    .getTime()
                                    );
                                } else if (jobInfo.getTriggerNextTime() < nowTime) {    // 过期10s内：立即触发一次，当前时间开始计算下次触发时间
                                    ringSecond = (int)((nowTime/1000)%60);

                                    jobInfo.setTriggerLastTime(jobInfo.getTriggerNextTime());
                                    jobInfo.setTriggerNextTime(
                                            new CronExpression(jobInfo.getJobCron())
                                                    .getNextValidTimeAfter(new Date())
                                                    .getTime()
                                    );
                                } else {    // 未过期：正常触发，递增计算下次触发时间
                                    ringSecond = (int)((jobInfo.getTriggerNextTime()/1000)%60);

                                    jobInfo.setTriggerLastTime(jobInfo.getTriggerNextTime());
                                    jobInfo.setTriggerNextTime(
                                            new CronExpression(jobInfo.getJobCron())
                                                    .getNextValidTimeAfter(new Date(jobInfo.getTriggerNextTime()))
                                                    .getTime()
                                    );
                                }
                                if (ringSecond == -1) {
                                    continue;
                                }

                                // push async ring
                                List<Integer> ringItemData = ringData.get(ringSecond);
                                if (ringItemData == null) {
                                    ringItemData = new ArrayList<Integer>();
                                    ringData.put(ringSecond, ringItemData);
                                }
                                ringItemData.add(jobInfo.getId());

                                logger.debug(">>>>>>>>>>> xxl-job, push time-ring : " + ringSecond + " = " + Arrays.asList(ringItemData) );
                            }

                            // 3、更新trigger信息
                            for (XxlJobInfo jobInfo: scheduleList) {
                                XxlJobAdminConfig.getAdminConfig().getXxlJobInfoDao().scheduleUpdate(jobInfo);
                            }

                        }

                        // tx stop

                        conn.commit();
                    } catch (Exception e) {
                        if (!toStop) {
                            logger.error(">>>>>>>>>>> xxl-job, JobScheduleHelper#scheduleThread error:{}", e);
                        }
                    } finally {
                        if (conn != null) {
                            try {
                                conn.close();
                            } catch (SQLException e) {
                            }
                        }
                        if (null != preparedStatement) {
                            try {
                                preparedStatement.close();
                            } catch (SQLException ignore) {
                            }
                        }
                    }
                    long cost = System.currentTimeMillis()-start;

                    // next second, align second
                    try {
                        if (cost < 1000) {
                            TimeUnit.MILLISECONDS.sleep(1000 - System.currentTimeMillis()%1000);
                        }
                    } catch (InterruptedException e) {
                        if (!toStop) {
                            logger.error(e.getMessage(), e);
                        }
                    }

                }
                logger.info(">>>>>>>>>>> xxl-job, JobScheduleHelper#scheduleThread stop");
            }
        });
        scheduleThread.setDaemon(true);
        scheduleThread.setName("xxl-job, admin JobScheduleHelper#scheduleThread");
        scheduleThread.start();


        // ring thread
        //这线程实现的方式 我看的一点也不好 实时性也不行 可以使用ScheduledThreadPoolExecutor方式
        ringThread = new Thread(new Runnable() {
            @Override
            public void run() {

                // align second
                try {
                    TimeUnit.MILLISECONDS.sleep(1000 - System.currentTimeMillis()%1000 );
                } catch (InterruptedException e) {
                    if (!toStop) {
                        logger.error(e.getMessage(), e);
                    }
                }

                int lastSecond = -1;
                while (!toStop) {

                    try {
                        // second data
                        List<Integer> ringItemData = new ArrayList<>();
                        int nowSecond = (int)((System.currentTimeMillis()/1000)%60);   // 避免处理耗时太长，跨过刻度；
                        if (lastSecond == -1) {
                            lastSecond = (nowSecond+59)%60;
                        }
                        for (int i = 1; i <=60; i++) {
                            int secondItem = (lastSecond+i)%60;

                            List<Integer> tmpData = ringData.remove(secondItem);
                            if (tmpData != null) {
                                ringItemData.addAll(tmpData);
                            }

                            if (secondItem == nowSecond) {
                                break;
                            }
                        }
                        lastSecond = nowSecond;

                        // ring trigger
                        logger.debug(">>>>>>>>>>> xxl-job, time-ring beat : " + nowSecond + " = " + Arrays.asList(ringItemData) );
                        if (ringItemData!=null && ringItemData.size()>0) {
                            // do trigger
                            for (int jobId: ringItemData) {
                                // do trigger
                                JobTriggerPoolHelper.trigger(jobId, TriggerTypeEnum.CRON, -1, null, null);
                            }
                            // clear
                            ringItemData.clear();
                        }
                    } catch (Exception e) {
                        if (!toStop) {
                            logger.error(">>>>>>>>>>> xxl-job, JobScheduleHelper#ringThread error:{}", e);
                        }
                    }

                    // next second, align second
                    try {
                        TimeUnit.MILLISECONDS.sleep(1000 - System.currentTimeMillis()%1000);
                    } catch (InterruptedException e) {
                        if (!toStop) {
                            logger.error(e.getMessage(), e);
                        }
                    }
                }
                logger.info(">>>>>>>>>>> xxl-job, JobScheduleHelper#ringThread stop");
            }
        });
        ringThread.setDaemon(true);
        ringThread.setName("xxl-job, admin JobScheduleHelper#ringThread");
        ringThread.start();
    }

    public void toStop(){
        toStop = true;

        // interrupt and wait
        scheduleThread.interrupt();
        try {
            scheduleThread.join();
        } catch (InterruptedException e) {
            logger.error(e.getMessage(), e);
        }

        // interrupt and wait
        ringThread.interrupt();
        try {
            ringThread.join();
        } catch (InterruptedException e) {
            logger.error(e.getMessage(), e);
        }
    }

}	
```


- 



### 3.2.3 调度中心和执行器执行任务的整个流程

- 调度中心手动执行一个任务
```aidl
@RequestMapping("/trigger")
@ResponseBody
//@PermessionLimit(limit = false)
public ReturnT<String> triggerJob(int id, String executorParam) {
    // force cover job param
    if (executorParam == null) {
        executorParam = "";
    }
    JobTriggerPoolHelper.trigger(id, TriggerTypeEnum.MANUAL, -1, null, executorParam);
    return ReturnT.SUCCESS;
}

这个任务被添加到线程池中异步执行
private ThreadPoolExecutor triggerPool = new ThreadPoolExecutor(
        32,
        256,
        60L,
        TimeUnit.SECONDS,
        new LinkedBlockingQueue<Runnable>(1000));
public void addTrigger(final int jobId, final TriggerTypeEnum triggerType, final int failRetryCount, final String executorShardingParam, final String executorParam) {
    triggerPool.execute(new Runnable() {
        @Override
        public void run() {
            XxlJobTrigger.trigger(jobId, triggerType, failRetryCount, executorShardingParam, executorParam);
        }
    });
}

public static void trigger(int jobId, TriggerTypeEnum triggerType, int failRetryCount, String executorShardingParam, String executorParam) {
    // load data
    XxlJobInfo jobInfo = XxlJobAdminConfig.getAdminConfig().getXxlJobInfoDao().loadById(jobId);
    if (jobInfo == null) {
        logger.warn(">>>>>>>>>>>> trigger fail, jobId invalid，jobId={}", jobId);
        return;
    }
    if (executorParam != null) {
        jobInfo.setExecutorParam(executorParam);
    }
    int finalFailRetryCount = failRetryCount>=0?failRetryCount:jobInfo.getExecutorFailRetryCount();
    XxlJobGroup group = XxlJobAdminConfig.getAdminConfig().getXxlJobGroupDao().load(jobInfo.getJobGroup());
    // sharding param
    int[] shardingParam = null;
    if (executorShardingParam!=null){
        String[] shardingArr = executorShardingParam.split("/");
        if (shardingArr.length==2 && StringUtils.isNumeric(shardingArr[0]) && StringUtils.isNumeric(shardingArr[1])) {
            shardingParam = new int[2];
            shardingParam[0] = Integer.valueOf(shardingArr[0]);
            shardingParam[1] = Integer.valueOf(shardingArr[1]);
        }
    }
    if (ExecutorRouteStrategyEnum.SHARDING_BROADCAST==ExecutorRouteStrategyEnum.match(jobInfo.getExecutorRouteStrategy(), null)
            && CollectionUtils.isNotEmpty(group.getRegistryList()) && shardingParam==null) {           
        for (int i = 0; i < group.getRegistryList().size(); i++) {
            //路由策略为分片广播时，给当前任务组的每个执行器机器都触发任务，每个执行器得到不一样的分片id
            processTrigger(group, jobInfo, finalFailRetryCount, triggerType, i, group.getRegistryList().size());
        }
    } else {
        if (shardingParam == null) {
            shardingParam = new int[]{0, 1};
        }
        processTrigger(group, jobInfo, finalFailRetryCount, triggerType, shardingParam[0], shardingParam[1]);
    }
}


  private static void processTrigger(XxlJobGroup group, XxlJobInfo jobInfo, int finalFailRetryCount, TriggerTypeEnum triggerType, int index, int total){

        // param 阻塞策略 路由策略 分片参数
        ExecutorBlockStrategyEnum blockStrategy = ExecutorBlockStrategyEnum.match(jobInfo.getExecutorBlockStrategy(), ExecutorBlockStrategyEnum.SERIAL_EXECUTION);  // block strategy
        ExecutorRouteStrategyEnum executorRouteStrategyEnum = ExecutorRouteStrategyEnum.match(jobInfo.getExecutorRouteStrategy(), null);    // route strategy
        String shardingParam = (ExecutorRouteStrategyEnum.SHARDING_BROADCAST==executorRouteStrategyEnum)?String.valueOf(index).concat("/").concat(String.valueOf(total)):null;

        // 1、save log-id 在保存调度中心保存 调度日志 到mysql数据库
        XxlJobLog jobLog = new XxlJobLog();
        jobLog.setJobGroup(jobInfo.getJobGroup());
        jobLog.setJobId(jobInfo.getId());
        jobLog.setTriggerTime(new Date());
        XxlJobAdminConfig.getAdminConfig().getXxlJobLogDao().save(jobLog);
        logger.debug(">>>>>>>>>>> xxl-job trigger start, jobId:{}", jobLog.getId());

        // 2、init trigger-param 设置该任务的各项参数
        TriggerParam triggerParam = new TriggerParam();
        triggerParam.setJobId(jobInfo.getId());
        //任务的处理类的名称，即@JobHandler(value="demoJobHandler")的value值
        triggerParam.setExecutorHandler(jobInfo.getExecutorHandler()); 
        triggerParam.setExecutorParams(jobInfo.getExecutorParam());
        //任务的阻塞策略
        triggerParam.setExecutorBlockStrategy(jobInfo.getExecutorBlockStrategy());
        triggerParam.setExecutorTimeout(jobInfo.getExecutorTimeout());
        triggerParam.setLogId(jobLog.getId());
        triggerParam.setLogDateTim(jobLog.getTriggerTime().getTime());
        //为BEAN的话setExecutorHandler有值，为Glue java类型时，在调度中心保存了java源码，发送执行器执行
        triggerParam.setGlueType(jobInfo.getGlueType()); 
        triggerParam.setGlueSource(jobInfo.getGlueSource());
        triggerParam.setGlueUpdatetime(jobInfo.getGlueUpdatetime().getTime());
        //分片Id
        triggerParam.setBroadcastIndex(index);
        //分片总数
        triggerParam.setBroadcastTotal(total);

        // 3、init address
        String address = null;
        ReturnT<String> routeAddressResult = null;
        if (CollectionUtils.isNotEmpty(group.getRegistryList())) {
            if (ExecutorRouteStrategyEnum.SHARDING_BROADCAST == executorRouteStrategyEnum) {
                //路由为分片广播时，给每台机器发送
                if (index < group.getRegistryList().size()) {
                    address = group.getRegistryList().get(index);
                } else {
                    address = group.getRegistryList().get(0);
                }
            } else {
                //路由为其他时，根据路由策略选取执行器地址，具体的下个问题介绍）
                routeAddressResult = executorRouteStrategyEnum.getRouter().route(triggerParam, group.getRegistryList());
                if (routeAddressResult.getCode() == ReturnT.SUCCESS_CODE) {
                    address = routeAddressResult.getContent();
                }
            }
        } else {
            routeAddressResult = new ReturnT<String>(ReturnT.FAIL_CODE, I18nUtil.getString("jobconf_trigger_address_empty"));
        }

        // 4、trigger remote executor
        ReturnT<String> triggerResult = null;
        if (address != null) {
            //发送任务参数到执行器执行并获取执行结果（执行器收到后入队列，直接返回成功，异步执行任务）
            triggerResult = runExecutor(triggerParam, address);
        } else {
            triggerResult = new ReturnT<String>(ReturnT.FAIL_CODE, null);
        }

        // 5、collection trigger info
        StringBuffer triggerMsgSb = new StringBuffer();
        triggerMsgSb.append(I18nUtil.getString("jobconf_trigger_type")).append("：").append(triggerType.getTitle());
        triggerMsgSb.append("<br>").append(I18nUtil.getString("jobconf_trigger_admin_adress")).append("：").append(IpUtil.getIp());
        triggerMsgSb.append("<br>").append(I18nUtil.getString("jobconf_trigger_exe_regtype")).append("：")
                .append( (group.getAddressType() == 0)?I18nUtil.getString("jobgroup_field_addressType_0"):I18nUtil.getString("jobgroup_field_addressType_1") );
        triggerMsgSb.append("<br>").append(I18nUtil.getString("jobconf_trigger_exe_regaddress")).append("：").append(group.getRegistryList());
        triggerMsgSb.append("<br>").append(I18nUtil.getString("jobinfo_field_executorRouteStrategy")).append("：").append(executorRouteStrategyEnum.getTitle());
        if (shardingParam != null) {
            triggerMsgSb.append("("+shardingParam+")");
        }
        triggerMsgSb.append("<br>").append(I18nUtil.getString("jobinfo_field_executorBlockStrategy")).append("：").append(blockStrategy.getTitle());
        triggerMsgSb.append("<br>").append(I18nUtil.getString("jobinfo_field_timeout")).append("：").append(jobInfo.getExecutorTimeout());
        triggerMsgSb.append("<br>").append(I18nUtil.getString("jobinfo_field_executorFailRetryCount")).append("：").append(finalFailRetryCount);

        triggerMsgSb.append("<br><br><span style=\"color:#00c0ef;\" > >>>>>>>>>>>"+ I18nUtil.getString("jobconf_trigger_run") +"<<<<<<<<<<< </span><br>")
                .append((routeAddressResult!=null&&routeAddressResult.getMsg()!=null)?routeAddressResult.getMsg()+"<br><br>":"").append(triggerResult.getMsg()!=null?triggerResult.getMsg():"");

        // 6、save log trigger-info 再次根据执行器返回的信息，跟新调度日志
        jobLog.setExecutorAddress(address);
        jobLog.setExecutorHandler(jobInfo.getExecutorHandler());
        jobLog.setExecutorParam(jobInfo.getExecutorParam());
        jobLog.setExecutorShardingParam(shardingParam);
        jobLog.setExecutorFailRetryCount(finalFailRetryCount);
        //jobLog.setTriggerTime();
        jobLog.setTriggerCode(triggerResult.getCode());
        jobLog.setTriggerMsg(triggerMsgSb.toString());
        XxlJobAdminConfig.getAdminConfig().getXxlJobLogDao().updateTriggerInfo(jobLog);

        // 7、monitor trigger 监控该任务执行情况
        JobFailMonitorHelper.monitor(jobLog.getId());
        logger.debug(">>>>>>>>>>> xxl-job trigger end, jobId:{}", jobLog.getId());
    }

监控任务
public static void monitor(int jobLogId){
    getInstance().queue.offer(jobLogId); //对该调度日志的任务进行如队列监控
}
private Thread monitorThread; 
//该监控线程对queue队列进行监控，判断队列中的任务是否成功
//成功的话，移除队列
//失败的话，进行报警和重试运行任务（调用JobTriggerPoolHelper.trigger重试，有次数限制）


public static ReturnT<String> runExecutor(TriggerParam triggerParam, String address){
    ReturnT<String> runResult = null;
    try {
        //获取执行器的RPC方法，调用run方法给执行器
        //XxlJobDynamicScheduler.getExecutorBiz有缓存，没有的话实例化ExecutorBiz
        ExecutorBiz executorBiz = XxlJobDynamicScheduler.getExecutorBiz(address);
        runResult = executorBiz.run(triggerParam);
    } catch (Exception e) {
        logger.error(">>>>>>>>>>> xxl-job trigger error, please check if the executor[{}] is running.", address, e);
        runResult = new ReturnT<String>(ReturnT.FAIL_CODE, ThrowableUtil.toString(e));
    }
    StringBuffer runResultSB = new StringBuffer(I18nUtil.getString("jobconf_trigger_run") + "：");
    runResultSB.append("<br>address：").append(address);
    runResultSB.append("<br>code：").append(runResult.getCode());
    runResultSB.append("<br>msg：").append(runResult.getMsg());

    runResult.setMsg(runResultSB.toString());
    return runResult;
}

实例化ExecutorBiz,并放入缓存
private static ConcurrentHashMap<String, ExecutorBiz> executorBizRepository = new ConcurrentHashMap<String, ExecutorBiz>();
executorBiz = (ExecutorBiz) new XxlRpcReferenceBean(NetEnum.JETTY, Serializer.SerializeEnum.HESSIAN.getSerializer(), CallType.SYNC,
        ExecutorBiz.class, null, 10000, address, XxlJobAdminConfig.getAdminConfig().getAccessToken(), null).getObject();
executorBizRepository.put(address, executorBiz);

```
- 执行器接受定时任务，并运行

```aidl
run方法是执行器提供远程RPC方法
@Override
public ReturnT<String> run(TriggerParam triggerParam) {
    // load old：jobHandler + jobThread 
    // JobThread是线程类，每来一个新的任务id生成一个新线程运行该任务，相同任务运行入该JobThread中的运行队列
    // 根据任务id,从缓存类中获取jobThread类
    JobThread jobThread = XxlJobExecutor.loadJobThread(triggerParam.getJobId());
    IJobHandler jobHandler = jobThread!=null?jobThread.getHandler():null;
    String removeOldReason = null;
    // valid：jobHandler + jobThread
    GlueTypeEnum glueTypeEnum = GlueTypeEnum.match(triggerParam.getGlueType());
    if (GlueTypeEnum.BEAN == glueTypeEnum) {
        // new jobhandler 
        IJobHandler newJobHandler = XxlJobExecutor.loadJobHandler(triggerParam.getExecutorHandler());
        // valid old jobThread
        if (jobThread!=null && jobHandler != newJobHandler) {
            // change handler, need kill old thread
            removeOldReason = "change jobhandler or glue type, and terminate the old job thread.";
            jobThread = null;
            jobHandler = null;
        }
        // valid handler
        if (jobHandler == null) {
            jobHandler = newJobHandler;
            if (jobHandler == null) {
                return new ReturnT<String>(ReturnT.FAIL_CODE, "job handler [" + triggerParam.getExecutorHandler() + "] not found.");
            }
        }
    } else if (GlueTypeEnum.GLUE_GROOVY == glueTypeEnum) {
        // valid old jobThread
        if (jobThread != null &&
                !(jobThread.getHandler() instanceof GlueJobHandler
                    && ((GlueJobHandler) jobThread.getHandler()).getGlueUpdatetime()==triggerParam.getGlueUpdatetime() )) {
            // change handler or gluesource updated, need kill old thread
            removeOldReason = "change job source or glue type, and terminate the old job thread.";
            jobThread = null;
            jobHandler = null;
        }
        // valid handler
        if (jobHandler == null) {
            try {
                IJobHandler originJobHandler = GlueFactory.getInstance().loadNewInstance(triggerParam.getGlueSource());
                jobHandler = new GlueJobHandler(originJobHandler, triggerParam.getGlueUpdatetime());
            } catch (Exception e) {
                logger.error(e.getMessage(), e);
                return new ReturnT<String>(ReturnT.FAIL_CODE, e.getMessage());
            }
        }
    } else if (glueTypeEnum!=null && glueTypeEnum.isScript()) {
        // valid old jobThread
        if (jobThread != null &&
                !(jobThread.getHandler() instanceof ScriptJobHandler
                        && ((ScriptJobHandler) jobThread.getHandler()).getGlueUpdatetime()==triggerParam.getGlueUpdatetime() )) {
            // change script or gluesource updated, need kill old thread
            removeOldReason = "change job source or glue type, and terminate the old job thread.";
            jobThread = null;
            jobHandler = null;
        }
        // valid handler
        if (jobHandler == null) {
            jobHandler = new ScriptJobHandler(triggerParam.getJobId(), triggerParam.getGlueUpdatetime(), triggerParam.getGlueSource(), GlueTypeEnum.match(triggerParam.getGlueType()));
        }
    } else {
        return new ReturnT<String>(ReturnT.FAIL_CODE, "glueType[" + triggerParam.getGlueType() + "] is not valid.");
    }
    // executor block strategy
    if (jobThread != null) {
        ExecutorBlockStrategyEnum blockStrategy = ExecutorBlockStrategyEnum.match(triggerParam.getExecutorBlockStrategy(), null);
        //阻塞策略为丢弃最新任务，新任务来时
        //如果jobThread中正在运行或运行队列中有待运行的任务，那么丢弃该新任务，并返回给调用中心失败
        if (ExecutorBlockStrategyEnum.DISCARD_LATER == blockStrategy) { 
            // discard when running
            if (jobThread.isRunningOrHasQueue()) {
                return new ReturnT<String>(ReturnT.FAIL_CODE, "block strategy effect："+ExecutorBlockStrategyEnum.DISCARD_LATER.getTitle());
            }
         //阻塞策略为覆盖之前的任务，新任务来时
        //如果jobThread中正在运行或运行队列中有待运行的任务，那么jobThread = null，以后的代码会新创建jobThread，并执行新任务
        } else if (ExecutorBlockStrategyEnum.COVER_EARLY == blockStrategy) {
            // kill running jobThread
            if (jobThread.isRunningOrHasQueue()) {
                removeOldReason = "block strategy effect：" + ExecutorBlockStrategyEnum.COVER_EARLY.getTitle();
                jobThread = null;
            }
        } else {
            // just queue trigger
        }
    }
    // replace thread (new or exists invalid) 
    if (jobThread == null) {
        //该任务第一次运行时，在执行器启动jobThread线程并启动运行
        //如果之前该任务id的jobThread，调用tostop方法执行之前的jobThread线程
        jobThread = XxlJobExecutor.registJobThread(triggerParam.getJobId(), jobHandler, removeOldReason);
    }
    // push data to queue
    // 把该次运行任务的参数，放入jobThread线程的运行队列中，等待运行
    ReturnT<String> pushResult = jobThread.pushTriggerQueue(triggerParam);
    return pushResult;
}

jobThread类真正运行定时任务的

public class JobThread extends Thread{
	private static Logger logger = LoggerFactory.getLogger(JobThread.class);

	private int jobId;
	private IJobHandler handler;
	private LinkedBlockingQueue<TriggerParam> triggerQueue;
	private Set<Integer> triggerLogIdSet;		// avoid repeat trigger for the same TRIGGER_LOG_ID

	private volatile boolean toStop = false;
	private String stopReason;

    private boolean running = false;    // if running job
	private int idleTimes = 0;			// idel times


	public JobThread(int jobId, IJobHandler handler) {
		this.jobId = jobId;
		this.handler = handler;
		this.triggerQueue = new LinkedBlockingQueue<TriggerParam>();
		this.triggerLogIdSet = Collections.synchronizedSet(new HashSet<Integer>());
	}
	public IJobHandler getHandler() {
		return handler;
	}

    /**
     * new trigger to queue
     *
     * @param triggerParam
     * @return
     */
	public ReturnT<String> pushTriggerQueue(TriggerParam triggerParam) {
		// avoid repeat
		if (triggerLogIdSet.contains(triggerParam.getLogId())) {
			logger.info(">>>>>>>>>>> repeate trigger job, logId:{}", triggerParam.getLogId());
			return new ReturnT<String>(ReturnT.FAIL_CODE, "repeate trigger job, logId:" + triggerParam.getLogId());
		}

		triggerLogIdSet.add(triggerParam.getLogId());
		triggerQueue.add(triggerParam);
        return ReturnT.SUCCESS;
	}

    /**
     * kill job thread
     *
     * @param stopReason
     */
	public void toStop(String stopReason) {
		/**
		 * Thread.interrupt只支持终止线程的阻塞状态(wait、join、sleep)，
		 * 在阻塞出抛出InterruptedException异常,但是并不会终止运行的线程本身；
		 * 所以需要注意，此处彻底销毁本线程，需要通过共享变量方式；
		 */
		this.toStop = true;
		this.stopReason = stopReason;
	}

    /**
     * is running job
     * @return
     */
    public boolean isRunningOrHasQueue() {
        return running || triggerQueue.size()>0;
    }

    @Override
	public void run() {

    	// init
    	try {
			handler.init();
		} catch (Throwable e) {
    		logger.error(e.getMessage(), e);
		}

		// execute
		while(!toStop){
			running = false;
			idleTimes++;

            TriggerParam triggerParam = null;
            ReturnT<String> executeResult = null;
            try {
				// to check toStop signal, we need cycle, so wo cannot use queue.take(), instand of poll(timeout)
				triggerParam = triggerQueue.poll(3L, TimeUnit.SECONDS);
				if (triggerParam!=null) {
					running = true;
					idleTimes = 0;
					triggerLogIdSet.remove(triggerParam.getLogId());

					// log filename, like "logPath/yyyy-MM-dd/9999.log"
					String logFileName = XxlJobFileAppender.makeLogFileName(new Date(triggerParam.getLogDateTim()), triggerParam.getLogId());
					//给当前线程设置，本地日志文件名称，本线程记录日志到这个文件中
					XxlJobFileAppender.contextHolder.set(logFileName);
					//给当前线程设置，分片参数
					ShardingUtil.setShardingVo(new ShardingUtil.ShardingVO(triggerParam.getBroadcastIndex(), triggerParam.getBroadcastTotal()));

					// execute
					//记录日志，内部使用XxlJobFileAppender.contextHolder.get()获取本线程对应的文件名称，进行日志记录
					XxlJobLogger.log("<br>----------- xxl-job job execute start -----------<br>----------- Param:" + triggerParam.getExecutorParams());

                    //如果有超时时间的话，使用FutureTask来先实现超时功能，并futureThread.interrupt();停止该任务
					if (triggerParam.getExecutorTimeout() > 0) {
						// limit timeout
						Thread futureThread = null;
						try {
							final TriggerParam triggerParamTmp = triggerParam;
							FutureTask<ReturnT<String>> futureTask = new FutureTask<ReturnT<String>>(new Callable<ReturnT<String>>() {
								@Override
								public ReturnT<String> call() throws Exception {
									return handler.execute(triggerParamTmp.getExecutorParams());
								}
							});
							futureThread = new Thread(futureTask);
							futureThread.start();

							executeResult = futureTask.get(triggerParam.getExecutorTimeout(), TimeUnit.SECONDS);
						} catch (TimeoutException e) {

							XxlJobLogger.log("<br>----------- xxl-job job execute timeout");
							XxlJobLogger.log(e);

							executeResult = new ReturnT<String>(IJobHandler.FAIL_TIMEOUT.getCode(), "job execute timeout ");
						} finally {
							futureThread.interrupt();
						}
					} else {
						// just execute
						//执行该任务
						executeResult = handler.execute(triggerParam.getExecutorParams());
					}

					if (executeResult == null) {
						executeResult = IJobHandler.FAIL;
					}
					XxlJobLogger.log("<br>----------- xxl-job job execute end(finish) -----------<br>----------- ReturnT:" + executeResult);

				} else {
				    //空闲次数大于30时，停止并移除该线程
					if (idleTimes > 30) {
						XxlJobExecutor.removeJobThread(jobId, "excutor idel times over limit.");
					}
				}
			} catch (Throwable e) {
				if (toStop) {
					XxlJobLogger.log("<br>----------- JobThread toStop, stopReason:" + stopReason);
				}

				StringWriter stringWriter = new StringWriter();
				e.printStackTrace(new PrintWriter(stringWriter));
				String errorMsg = stringWriter.toString();
				executeResult = new ReturnT<String>(ReturnT.FAIL_CODE, errorMsg);

				XxlJobLogger.log("<br>----------- JobThread Exception:" + errorMsg + "<br>----------- xxl-job job execute end(error) -----------");
			} finally {
                if(triggerParam != null) {
                    // callback handler info
                    if (!toStop) {
                        // commonm
                        //把执行结果返回给调度中心，先入 LinkedBlockingQueue<HandleCallbackParam> callBackQueue = new LinkedBlockingQueue<HandleCallbackParam>(); 队列
                        //再由callback线程，执行ReturnT<String> callbackResult = adminBiz.callback(callbackParamList);给调度中心
                        TriggerCallbackThread.pushCallBack(new HandleCallbackParam(triggerParam.getLogId(), triggerParam.getLogDateTim(), executeResult));
                    } else {
                        // is killed
                        ReturnT<String> stopResult = new ReturnT<String>(ReturnT.FAIL_CODE, stopReason + " [job running，killed]");
                        TriggerCallbackThread.pushCallBack(new HandleCallbackParam(triggerParam.getLogId(), triggerParam.getLogDateTim(), stopResult));
                    }
                }
            }
        }

		// callback trigger request in queue
		while(triggerQueue !=null && triggerQueue.size()>0){
			TriggerParam triggerParam = triggerQueue.poll();
			if (triggerParam!=null) {
				// is killed
				ReturnT<String> stopResult = new ReturnT<String>(ReturnT.FAIL_CODE, stopReason + " [job not executed, in the job queue, killed.]");
				TriggerCallbackThread.pushCallBack(new HandleCallbackParam(triggerParam.getLogId(), triggerParam.getLogDateTim(), stopResult));
			}
		}

		// destroy
		try {
			handler.destroy();
		} catch (Throwable e) {
			logger.error(e.getMessage(), e);
		}

		logger.info(">>>>>>>>>>> xxl-job JobThread stoped, hashCode:{}", Thread.currentThread());
	}
}
//移除空闲的线程
public static void removeJobThread(int jobId, String removeOldReason){
    JobThread oldJobThread = jobThreadRepository.remove(jobId);
    if (oldJobThread != null) {
        oldJobThread.toStop(removeOldReason);
        oldJobThread.interrupt();
    }
}


```

### 3.2.4 调度中心各种路由策略实现原理


- 路由策略：当执行器集群部署时，提供丰富的路由策略，包括

```aidl
FIRST（第一个）：固定选择第一个机器；
LAST（最后一个）：固定选择最后一个机器；
ROUND（轮询）：；
RANDOM（随机）：随机选择在线的机器；
CONSISTENT_HASH（一致性HASH）：每个任务按照Hash算法固定选择某一台机器，且所有任务均匀散列在不同机器上。
LEAST_FREQUENTLY_USED（最不经常使用）：使用频率最低的机器优先被选举；
LEAST_RECENTLY_USED（最近最久未使用）：最久为使用的机器优先被选举；
FAILOVER（故障转移）：按照顺序依次进行心跳检测，第一个心跳检测成功的机器选定为目标执行器并发起调度；
BUSYOVER（忙碌转移）：按照顺序依次进行空闲检测，第一个空闲检测成功的机器选定为目标执行器并发起调度；
SHARDING_BROADCAST(分片广播)：广播触发对应集群中所有机器执行一次任务，同时系统自动传递分片参数；可根据分片参数开发分片任务；

public enum ExecutorRouteStrategyEnum {
    FIRST(I18nUtil.getString("jobconf_route_first"), new ExecutorRouteFirst()),
    LAST(I18nUtil.getString("jobconf_route_last"), new ExecutorRouteLast()),
    ROUND(I18nUtil.getString("jobconf_route_round"), new ExecutorRouteRound()),
    RANDOM(I18nUtil.getString("jobconf_route_random"), new ExecutorRouteRandom()),
    CONSISTENT_HASH(I18nUtil.getString("jobconf_route_consistenthash"), new ExecutorRouteConsistentHash()),
    LEAST_FREQUENTLY_USED(I18nUtil.getString("jobconf_route_lfu"), new ExecutorRouteLFU()),
    LEAST_RECENTLY_USED(I18nUtil.getString("jobconf_route_lru"), new ExecutorRouteLRU()),
    FAILOVER(I18nUtil.getString("jobconf_route_failover"), new ExecutorRouteFailover()),
    BUSYOVER(I18nUtil.getString("jobconf_route_busyover"), new ExecutorRouteBusyover()),
    SHARDING_BROADCAST(I18nUtil.getString("jobconf_route_shard"), null);
}

FIRST（第一个）实现原理是取执行器地址列表的第一个
    public class ExecutorRouteFirst extends ExecutorRouter {
        @Override
        public ReturnT<String> route(TriggerParam triggerParam, List<String> addressList){
            return new ReturnT<String>(addressList.get(0));
        }
    }
LAST（最后一个）实现原理是取执行器地址列表的最后一个
    public class ExecutorRouteLast extends ExecutorRouter {
        @Override
        public ReturnT<String> route(TriggerParam triggerParam, List<String> addressList) {
            return new ReturnT<String>(addressList.get(addressList.size()-1));
        }
    }
ROUND（轮询）实现原理是本地调度中心保存任务id的调度次数，下次+1进行轮训选择执行器
    public class ExecutorRouteRound extends ExecutorRouter {
    
        private static ConcurrentHashMap<Integer, Integer> routeCountEachJob = new ConcurrentHashMap<Integer, Integer>();
        private static long CACHE_VALID_TIME = 0;
        private static int count(int jobId) {
            // cache clear
            if (System.currentTimeMillis() > CACHE_VALID_TIME) {
                routeCountEachJob.clear();
                CACHE_VALID_TIME = System.currentTimeMillis() + 1000*60*60*24;
            }
            // count++
            Integer count = routeCountEachJob.get(jobId);
            count = (count==null || count>1000000)?(new Random().nextInt(100)):++count;  // 初始化时主动Random一次，缓解首次压力
            routeCountEachJob.put(jobId, count);
            return count;
        }
        @Override
        public ReturnT<String> route(TriggerParam triggerParam, List<String> addressList) {
            String address = addressList.get(count(triggerParam.getJobId())%addressList.size());
            return new ReturnT<String>(address);
        }
    }
RANDOM（随机）：随机选择在线的机器；
    public class ExecutorRouteRandom extends ExecutorRouter {
        private static Random localRandom = new Random();
        @Override
        public ReturnT<String> route(TriggerParam triggerParam, List<String> addressList) {
            String address = addressList.get(localRandom.nextInt(addressList.size()));
            return new ReturnT<String>(address);
        }
    }    
CONSISTENT_HASH（一致性HASH）：每个任务按照Hash算法固定选择某一台机器，且所有任务均匀散列在不同机器上。
    一致性HASH环，每个地址在环上对应多个位置（虚拟节点解决不均衡问题），任务id 的HASH值落在环上，顺时针找到第一个
    执行器地址返回。
    /**
     * 分组下机器地址相同，不同JOB均匀散列在不同机器上，保证分组下机器分配JOB平均；且每个JOB固定调度其中一台机器；
     *      a、virtual node：解决不均衡问题
     *      b、hash method replace hashCode：String的hashCode可能重复，需要进一步扩大hashCode的取值范围
     * Created by xuxueli on 17/3/10.
     */
    public class ExecutorRouteConsistentHash extends ExecutorRouter {
        private static int VIRTUAL_NODE_NUM = 5;
        /**
         * get hash code on 2^32 ring (md5散列的方式计算hash值)
         * @param key
         * @return
         */
        private static long hash(String key) {
            // md5 byte
            MessageDigest md5;
            try {
                md5 = MessageDigest.getInstance("MD5");
            } catch (NoSuchAlgorithmException e) {
                throw new RuntimeException("MD5 not supported", e);
            }
            md5.reset();
            byte[] keyBytes = null;
            try {
                keyBytes = key.getBytes("UTF-8");
            } catch (UnsupportedEncodingException e) {
                throw new RuntimeException("Unknown string :" + key, e);
            }
            md5.update(keyBytes);
            byte[] digest = md5.digest();
            // hash code, Truncate to 32-bits
            long hashCode = ((long) (digest[3] & 0xFF) << 24)
                    | ((long) (digest[2] & 0xFF) << 16)
                    | ((long) (digest[1] & 0xFF) << 8)
                    | (digest[0] & 0xFF);
            long truncateHashCode = hashCode & 0xffffffffL;
            return truncateHashCode;
        }
        public String hashJob(int jobId, List<String> addressList) {
            // ------A1------A2-------A3------
            // -----------J1------------------
            TreeMap<Long, String> addressRing = new TreeMap<Long, String>();
            for (String address: addressList) {
                for (int i = 0; i < VIRTUAL_NODE_NUM; i++) {
                    long addressHash = hash("SHARD-" + address + "-NODE-" + i);
                    addressRing.put(addressHash, address);
                }
            }
            long jobHash = hash(String.valueOf(jobId));
            SortedMap<Long, String> lastRing = addressRing.tailMap(jobHash);
            if (!lastRing.isEmpty()) {
                return lastRing.get(lastRing.firstKey());
            }
            return addressRing.firstEntry().getValue();
        }
        @Override
        public ReturnT<String> route(TriggerParam triggerParam, List<String> addressList) {
            String address = hashJob(triggerParam.getJobId(), addressList);
            return new ReturnT<String>(address);
        }
    }   
LEAST_FREQUENTLY_USED（最不经常使用）：使用频率最低的机器优先被选举；
    每个任务ID -> 所有执行器地址和执行次数，每次获取执行次数最少的那个执行器地址返回
    public class ExecutorRouteLFU extends ExecutorRouter {
        private static ConcurrentHashMap<Integer, HashMap<String, Integer>> jobLfuMap = new ConcurrentHashMap<Integer, HashMap<String, Integer>>();
        private static long CACHE_VALID_TIME = 0;
        public String route(int jobId, List<String> addressList) {
            // cache clear
            if (System.currentTimeMillis() > CACHE_VALID_TIME) {
                jobLfuMap.clear();
                CACHE_VALID_TIME = System.currentTimeMillis() + 1000*60*60*24;
            }
            // lfu item init
            HashMap<String, Integer> lfuItemMap = jobLfuMap.get(jobId);     // Key排序可以用TreeMap+构造入参Compare；Value排序暂时只能通过ArrayList；
            if (lfuItemMap == null) {
                lfuItemMap = new HashMap<String, Integer>();
                jobLfuMap.putIfAbsent(jobId, lfuItemMap);   // 避免重复覆盖
            }
            for (String address: addressList) {
                if (!lfuItemMap.containsKey(address) || lfuItemMap.get(address) >1000000 ) {
                    lfuItemMap.put(address, new Random().nextInt(addressList.size()));  // 初始化时主动Random一次，缓解首次压力
                }
            }
            // load least userd count address
            List<Map.Entry<String, Integer>> lfuItemList = new ArrayList<Map.Entry<String, Integer>>(lfuItemMap.entrySet());
            Collections.sort(lfuItemList, new Comparator<Map.Entry<String, Integer>>() {
                @Override
                public int compare(Map.Entry<String, Integer> o1, Map.Entry<String, Integer> o2) {
                    return o1.getValue().compareTo(o2.getValue());
                }
            });
            Map.Entry<String, Integer> addressItem = lfuItemList.get(0);
            String minAddress = addressItem.getKey();
            addressItem.setValue(addressItem.getValue() + 1);
            return addressItem.getKey();
        }
        @Override
        public ReturnT<String> route(TriggerParam triggerParam, List<String> addressList) {
            String address = route(triggerParam.getJobId(), addressList);
            return new ReturnT<String>(address);
        }
    }
    
LEAST_RECENTLY_USED（最近最久未使用）：最久未使用的机器优先被选举；
     LinkedHashMap有顺序的 ，可以实现访问排序，get的key被放到最后，链表最前的key是最久未使用
    /**
     * 单个JOB对应的每个执行器，最久为使用的优先被选举
     *      a、LFU(Least Frequently Used)：最不经常使用，频率/次数
     *      b(*)、LRU(Least Recently Used)：最近最久未使用，时间
     */
    public class ExecutorRouteLRU extends ExecutorRouter {
        private static ConcurrentHashMap<Integer, LinkedHashMap<String, String>> jobLRUMap = new ConcurrentHashMap<Integer, LinkedHashMap<String, String>>();
        private static long CACHE_VALID_TIME = 0;
        public String route(int jobId, List<String> addressList) {
            // cache clear
            if (System.currentTimeMillis() > CACHE_VALID_TIME) {
                jobLRUMap.clear();
                CACHE_VALID_TIME = System.currentTimeMillis() + 1000*60*60*24;
            }
            // init lru
            LinkedHashMap<String, String> lruItem = jobLRUMap.get(jobId);
            if (lruItem == null) {
                /**
                 * LinkedHashMap
                 *      a、accessOrder：ture=访问顺序排序（get/put时排序）；false=插入顺序排期；
                 *      b、removeEldestEntry：新增元素时将会调用，返回true时会删除最老元素；可封装LinkedHashMap并重写该方法，比如定义最大容量，超出是返回true即可实现固定长度的LRU算法；
                 */
                lruItem = new LinkedHashMap<String, String>(16, 0.75f, true);
                jobLRUMap.putIfAbsent(jobId, lruItem);
            }
            // put
            for (String address: addressList) {
                if (!lruItem.containsKey(address)) {
                    lruItem.put(address, address);
                }
            }
            // load
            String eldestKey = lruItem.entrySet().iterator().next().getKey();
            String eldestValue = lruItem.get(eldestKey);
            return eldestValue;
        }
        @Override
        public ReturnT<String> route(TriggerParam triggerParam, List<String> addressList) {
            String address = route(triggerParam.getJobId(), addressList);
            return new ReturnT<String>(address);
        }
    }

FAILOVER（故障转移）：按照顺序依次进行心跳检测，第一个心跳检测成功的机器选定为目标执行器并发起调度；
    使用ExecutorBiz的RPC方法去执行器发送心跳包，执行器返回成功的话就获取这个机器运行
    public class ExecutorRouteFailover extends ExecutorRouter {
        @Override
        public ReturnT<String> route(TriggerParam triggerParam, List<String> addressList) {
            StringBuffer beatResultSB = new StringBuffer();
            for (String address : addressList) {
                // beat
                ReturnT<String> beatResult = null;
                try {
                    ExecutorBiz executorBiz = XxlJobDynamicScheduler.getExecutorBiz(address);
                    beatResult = executorBiz.beat();
                    //      public ReturnT<String> beat() { 执行器实现类中直接返回成功
                    //          return ReturnT.SUCCESS;
                    //      }                                     
                } catch (Exception e) {
                    logger.error(e.getMessage(), e);
                    beatResult = new ReturnT<String>(ReturnT.FAIL_CODE, ""+e );
                }
                beatResultSB.append( (beatResultSB.length()>0)?"<br><br>":"")
                        .append(I18nUtil.getString("jobconf_beat") + "：")
                        .append("<br>address：").append(address)
                        .append("<br>code：").append(beatResult.getCode())
                        .append("<br>msg：").append(beatResult.getMsg());
                // beat success
                if (beatResult.getCode() == ReturnT.SUCCESS_CODE) {
                    beatResult.setMsg(beatResultSB.toString());
                    beatResult.setContent(address);
                    return beatResult;
                }
            }
            return new ReturnT<String>(ReturnT.FAIL_CODE, beatResultSB.toString());
        }
    }

BUSYOVER（忙碌转移）：按照顺序依次进行空闲检测，第一个空闲检测成功的机器选定为目标执行器并发起调度；
    使用ExecutorBiz的RPC方法给执行器判断执行器上的对应线程是否在运行，没有运行的话就获取这个机器运行
    public class ExecutorRouteBusyover extends ExecutorRouter {
        @Override
        public ReturnT<String> route(TriggerParam triggerParam, List<String> addressList) {
            StringBuffer idleBeatResultSB = new StringBuffer();
            for (String address : addressList) {
                // beat
                ReturnT<String> idleBeatResult = null;
                try {
                    ExecutorBiz executorBiz = XxlJobDynamicScheduler.getExecutorBiz(address);
                    idleBeatResult = executorBiz.idleBeat(triggerParam.getJobId());
                        // public ReturnT<String> idleBeat(int jobId) { 执行器中的实现类，判断任务ID对应的线程是否在运行
                        //      boolean isRunningOrHasQueue = false;
                        //      JobThread jobThread = XxlJobExecutor.loadJobThread(jobId);
                       //       if (jobThread != null && jobThread.isRunningOrHasQueue()) {
                        //         isRunningOrHasQueue = true;
                       //       }      
                      //        if (isRunningOrHasQueue) {
                       //           return new ReturnT<String>(ReturnT.FAIL_CODE, "job thread is running or has trigger queue.");
                      //        }
                      //        return ReturnT.SUCCESS;                      
                } catch (Exception e) {
                    logger.error(e.getMessage(), e);
                    idleBeatResult = new ReturnT<String>(ReturnT.FAIL_CODE, ""+e );
                }
                idleBeatResultSB.append( (idleBeatResultSB.length()>0)?"<br><br>":"")
                        .append(I18nUtil.getString("jobconf_idleBeat") + "：")
                        .append("<br>address：").append(address)
                        .append("<br>code：").append(idleBeatResult.getCode())
                        .append("<br>msg：").append(idleBeatResult.getMsg());
                // beat success
                if (idleBeatResult.getCode() == ReturnT.SUCCESS_CODE) {
                    idleBeatResult.setMsg(idleBeatResultSB.toString());
                    idleBeatResult.setContent(address);
                    return idleBeatResult;
                }
            }
            return new ReturnT<String>(ReturnT.FAIL_CODE, idleBeatResultSB.toString());
        }
    }   
 SHARDING_BROADCAST(分片广播)：广播触发对应集群中所有机器执行一次任务，同时系统自动传递分片参数；可根据分片参数开发分片任务；    
    当路由策略为SHARDING_BROADCAST时，给当前任务组的所有执行器地址发送任务进行执行，每个执行器得到分片ID和分片总数两个参数
     if (ExecutorRouteStrategyEnum.SHARDING_BROADCAST==ExecutorRouteStrategyEnum.match(jobInfo.getExecutorRouteStrategy(), null)
             && CollectionUtils.isNotEmpty(group.getRegistryList()) && shardingParam==null) {
         for (int i = 0; i < group.getRegistryList().size(); i++) {
             processTrigger(group, jobInfo, finalFailRetryCount, triggerType, i, group.getRegistryList().size());//触发任务执行
         }
     }
 
```

### 3.2.5 调度中心页面中查看执行日志如何实现Rolling实时日志

- 调度中心页面使用joblog.detail.1.js定时执行请求，获取执行器中最新的日志内容并显示到页面

```aidl
    $(function() {
        // trigger fail, end
        if ( !(triggerCode == 200 || handleCode != 0) ) {
            $('#logConsoleRunning').hide();
            $('#logConsole').append('<span style="color: red;">'+ I18n.joblog_rolling_log_triggerfail +'</span>');
            return;
        }
        // pull log
        var fromLineNum = 1;    // [from, to], start as 1
        var pullFailCount = 0;
        function pullLog() {
            // pullFailCount, max=20
            if (pullFailCount++ > 20) {
                logRunStop('<span style="color: red;">'+ I18n.joblog_rolling_log_failoften +'</span>');
                return;
            }
            // load
            console.log("pullLog, fromLineNum:" + fromLineNum);
            $.ajax({
                type : 'POST',
                async: false,   // sync, make log ordered
                url : base_url + '/joblog/logDetailCat',
                data : {
                    "executorAddress":executorAddress,
                    "triggerTime":triggerTime,
                    "logId":logId,
                    "fromLineNum":fromLineNum  //从多少行开始
                },
                dataType : "json",
                success : function(data){
                    if (data.code == 200) {
                        if (!data.content) {
                            console.log('pullLog fail');
                            return;
                        }
                        if (fromLineNum != data.content.fromLineNum) {
                            console.log('pullLog fromLineNum not match');
                            return;
                        }
                        if (fromLineNum > data.content.toLineNum ) { //请求日志到结束了
                            console.log('pullLog already line-end');
                            // valid end
                            if (data.content.end) {
                                //停止当前js的定时器
                                logRunStop('<br><span style="color: green;">[Rolling Log Finish]</span>');
                                return;
                            }
                            return;
                        }
                        // append content
                        fromLineNum = data.content.toLineNum + 1;
                        $('#logConsole').append(data.content.logContent);
                        pullFailCount = 0;
                        // scroll to bottom
                        scrollTo(0, document.body.scrollHeight);        // $('#logConsolePre').scrollTop( document.body.scrollHeight + 300 );
                    } else {
                        console.log('pullLog fail:'+data.msg);
                    }
                }
            });
        }
        // pull first page
        pullLog();
        // handler already callback, end
        if (handleCode > 0) {
            logRunStop('<br><span style="color: green;">[Load Log Finish]</span>');
            return;
        }
        // round until end
        var logRun = setInterval(function () { //每3秒钟请求一次日志
            pullLog()
        }, 3000);
        function logRunStop(content){
            $('#logConsoleRunning').hide();
            logRun = window.clearInterval(logRun);
            $('#logConsole').append(content);
        }
    });
```

- 调度中心接口接受页面JS的请求转发给执行器并获取结果

```aidl
	@RequestMapping("/logDetailCat")
	@ResponseBody
	public ReturnT<LogResult> logDetailCat(String executorAddress, long triggerTime, int logId, int fromLineNum){
		try {
			ExecutorBiz executorBiz = XxlJobDynamicScheduler.getExecutorBiz(executorAddress);
			//远程RPC调用
			ReturnT<LogResult> logResult = executorBiz.log(triggerTime, logId, fromLineNum);
			// is end
            if (logResult.getContent()!=null && logResult.getContent().getFromLineNum() > logResult.getContent().getToLineNum()) {
                XxlJobLog jobLog = xxlJobLogDao.load(logId);
                //当任务执行完成后，并且读取日志开始行数是日志最后一行了、就可以设置日志读取完成了，js也会停止定时器
                if (jobLog.getHandleCode() > 0) {
                    logResult.getContent().setEnd(true);
                }
            }
			return logResult;
		} catch (Exception e) {
			logger.error(e.getMessage(), e);
			return new ReturnT<LogResult>(ReturnT.FAIL_CODE, e.getMessage());
		}
	}	
```

- 执行器获取执行器的方式
```aidl
    @Override
    public ReturnT<LogResult> log(long logDateTim, int logId, int fromLineNum) {
        // log filename: logPath/yyyy-MM-dd/9999.log
        //执行器日志文件存储在本地，名称使用makeLogFileName方法名称
        String logFileName = XxlJobFileAppender.makeLogFileName(new Date(logDateTim), logId);
        //从fromLineNum开始返回最新的执行日志，
        LogResult logResult = XxlJobFileAppender.readLog(logFileName, fromLineNum);
        return new ReturnT<LogResult>(logResult);
    }	
```

### 3.2.6 GLUE模式(Java)任务如何实现

- 调度中心页面中写java代码，可以在执行器中运行该java代码

```aidl
在页面中写下面的代码，保存在调度中心的数据库中
    package com.xxl.job.service.handler;
    import com.xxl.job.core.log.XxlJobLogger;
    import com.xxl.job.core.biz.model.ReturnT;
    import com.xxl.job.core.handler.IJobHandler;
    public class DemoGlueJobHandler extends IJobHandler {
        @Override
        public ReturnT<String> execute(String param) throws Exception {
            XxlJobLogger.log("XXL-JOB, Hello World.");
            return ReturnT.SUCCESS;
        }
    }

调度中心执行该任务，任务类型为GLUE模式(Java)，发送执行参数给执行器
    TriggerParam triggerParam = new TriggerParam();
    triggerParam.setJobId(jobInfo.getId());
    triggerParam.setGlueType(jobInfo.getGlueType());
    triggerParam.setGlueSource(jobInfo.getGlueSource()); //内容为在上面页面的源码
    runExecutor(triggerParam, address);
```

- 执行器根据GlueType类型执行Glue的任务
```aidl
类型为GLUE模式(Java)的执行逻辑，把triggerParam.getGlueSource()加载成一个jobHandler类进行运行
 if (GlueTypeEnum.GLUE_GROOVY == glueTypeEnum) {
            // valid old jobThread
            if (jobThread != null &&
                    !(jobThread.getHandler() instanceof GlueJobHandler
                        && ((GlueJobHandler) jobThread.getHandler()).getGlueUpdatetime()==triggerParam.getGlueUpdatetime() )) {
                // change handler or gluesource updated, need kill old thread
                removeOldReason = "change job source or glue type, and terminate the old job thread.";
                jobThread = null;
                jobHandler = null;
            }
            // valid handler
            if (jobHandler == null) {
                try {
                    IJobHandler originJobHandler = GlueFactory.getInstance().loadNewInstance(triggerParam.getGlueSource());
                    jobHandler = new GlueJobHandler(originJobHandler, triggerParam.getGlueUpdatetime());
                } catch (Exception e) {
                    logger.error(e.getMessage(), e);
                    return new ReturnT<String>(ReturnT.FAIL_CODE, e.getMessage());
                }
            }
        }
把triggerParam.getGlueSource()加载成一个jobHandler类
 	public IJobHandler loadNewInstance(String codeSource) throws Exception{
 		if (codeSource!=null && codeSource.trim().length()>0) {
 			Class<?> clazz = groovyClassLoader.parseClass(codeSource);
 			if (clazz != null) {
 				Object instance = clazz.newInstance();
 				if (instance!=null) {
 					if (instance instanceof IJobHandler) {
 						this.injectService(instance);
 						return (IJobHandler) instance;
 					} else {
 						throw new IllegalArgumentException(">>>>>>>>>>> xxl-glue, loadNewInstance error, "
 								+ "cannot convert from instance["+ instance.getClass() +"] to IJobHandler");
 					}
 				}
 			}
 		}
 		throw new IllegalArgumentException(">>>>>>>>>>> xxl-glue, loadNewInstance error, instance is null");
 	}       

```
- GlueType类型为脚本类（shell、Python、php、Nodejs、powershell）的执行逻辑

```aidl

GlueType类型
    public enum GlueTypeEnum {
        BEAN("BEAN", false, null, null),
        GLUE_GROOVY("GLUE(Java)", false, null, null),
        GLUE_SHELL("GLUE(Shell)", true, "bash", ".sh"),
        GLUE_PYTHON("GLUE(Python)", true, "python", ".py"),
        GLUE_PHP("GLUE(PHP)", true, "php", ".php"),
        GLUE_NODEJS("GLUE(Nodejs)", true, "node", ".js"),
        GLUE_POWERSHELL("GLUE(PowerShell)", true, "powershell ", ".ps1");
    }
    if (glueTypeEnum!=null && glueTypeEnum.isScript()) {
                // valid old jobThread
                if (jobThread != null &&
                        !(jobThread.getHandler() instanceof ScriptJobHandler
                                && ((ScriptJobHandler) jobThread.getHandler()).getGlueUpdatetime()==triggerParam.getGlueUpdatetime() )) {
                    // change script or gluesource updated, need kill old thread
                    removeOldReason = "change job source or glue type, and terminate the old job thread.";
                    jobThread = null;
                    jobHandler = null;
                }
                // valid handler
                if (jobHandler == null) {
                    jobHandler = new ScriptJobHandler(triggerParam.getJobId(), triggerParam.getGlueUpdatetime(), triggerParam.getGlueSource(), GlueTypeEnum.match(triggerParam.getGlueType()));
                }
     }     
 ScriptJobHandler中执行方法最后调用execToFile执行脚本
    //cmd为bash或python或php或node或powershell
    //scriptFile为脚本执行内容
    public static int execToFile(String command, String scriptFile, String logFile, String... params) throws IOException {
            // 标准输出：print （null if watchdog timeout）
            // 错误输出：logging + 异常 （still exists if watchdog timeout）
            // 标准输入  
            FileOutputStream fileOutputStream = null;   //
            try {
                fileOutputStream = new FileOutputStream(logFile, true);
                PumpStreamHandler streamHandler = new PumpStreamHandler(fileOutputStream, fileOutputStream, null); 
                // command
                CommandLine commandline = new CommandLine(command); //使用的是org.apache.commons.exe包
                commandline.addArgument(scriptFile);
                if (params!=null && params.length>0) {
                    commandline.addArguments(params);
                }   
                // exec
                DefaultExecutor exec = new DefaultExecutor();
                exec.setExitValues(null);
                exec.setStreamHandler(streamHandler);
                int exitValue = exec.execute(commandline);  // exit code: 0=success, 1=error
                return exitValue;
            } catch (Exception e) {
                XxlJobLogger.log(e);
                return -1;
            } finally {
                if (fileOutputStream != null) {
                    try {
                        fileOutputStream.close();
                    } catch (IOException e) {
                        XxlJobLogger.log(e);
                    }
    
                }
            }
        }
    
```



# 四、其他
