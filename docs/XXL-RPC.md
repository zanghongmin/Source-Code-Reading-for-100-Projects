* [一、项目概览](#一数据类型)
    * [1.1 简介](#1-1简介) 
    * [1.2 环境](#1-2环境)
    * [1.3 源码及官网](#1-3源码及官网)
* [二、项目使用](#二项目使用)
* [三、项目设计](#三项目设计)
    * [3.1 总体设计](#3-1总体设计)
    * [3.2 关键点分析](#3-2关键点分析)
        * [3.2.1 调度中心和执行器怎么通讯](#3-2-1调度中心和执行器怎么通讯)  
        * [3.2.2 调度中心怎么按时调度任务](#3-2-2调度中心怎么按时调度任务)  
        * [3.2.3 调度中心调度任务和执行器执行任务的整个流程](#3-2-3调度中心调度任务和执行器执行任务的整个流程)  
* [四、其他](#四其他)

# 一、项目概览

## 1.1 简介

## 1.2 环境

## 1.3 源码及官网

# 二、项目使用


# 三、项目设计

## 3.1 总体设计
![总体架构图](../pic/XXL-JOB-OverALL.png)

* 分为调度中心和执行器两个部分
* 调度中心管理定时任务，调度定时任务，和页面显示等
* 执行器主要用来执行定时任务

## 3.2 关键点分析
### 3.2.1 调度中心和执行器怎么通讯
 1. 通过XXL-RPC框架进行RPC通讯，先看下执行器访问调度中心的方式
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

 
### 3.2.2 调度中心怎么按时调度任务
### 3.2.3 调度中心调度任务和执行器执行任务的整个流程


# 四、其他
