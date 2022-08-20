_Spring cloud+eureka是微服务常用解决方案之一，kubernetes则是广泛应用的发布工具，两者结合使用很常见。两者结合使用时如何优雅启停从而实现无感发布很重要。  
下面将从不做特殊处理时启停存在的问题、业务代码设计要求、spring cloud+eureka本身停机处理机制、k8s滚动发布如何关联spring程序的启停机制 几点分析和提出解决方案。

### 1、不做特殊处理时启停存在的问题
#### 1.1、启动服务时的问题
(1)spring boot项目有EnableDiscoveryClient注解时服务启动后会自动向eureka注册。如果服务数据多预加载耗时长，头部的部分请求会失败，需要控制注册发生在预加载后。
(2)spring程序启动和预加载数据需要时间。但k8s在启动Pod后很快将外部请求导向服务，导致异常。解决这个问题需要k8s知道服务是否就绪。

#### 1.2、停止服务时的问题

最原始的关闭服务方法是使用kill指令，常用的信号选项:
(1) kill -2 pid 向指定 pid 发送 SIGINT 中断信号, 等同于 ctrl+c.
(2) kill -9 pid, 向指定 pid 发送 SIGKILL 立即终止信号.
(3) kill -15 pid, 向指定 pid 发送 SIGTERM 终止信号.
(4) kill pid 等同于 kill 15 pid

SIGINT/SIGKILL/SIGTERM 信号的区别:
(1) SIGINT (ctrl+c) 信号(信号编号为2),信号会被当前进程树接收到,当前进程及它的子进程都会收到该信号。
(2) SIGKILL信号(信号编号为 9),程序不能捕获该信号,最粗暴最快速结束程序的方法。
(3) SIGTERM 信号 (信号编号为 15)会被当前进程接收到, 但子进程不会收到。如果当前进程被 kill 掉, 其子进程的父进程将变成 init 进程 (即pid 为 1 的进程)

应尽量使用 kill pid结束服务。如果服务提供关闭通知接口,在完全退出之前调用，让其先做完进行中的任务。

k8s关闭pod时，相当于执行kill -15指令，在一定时间（默认30s，可配置）后仍未关闭就执行kill -9 强制杀死进程。

Java提供了注册监听器接收SIGTERM信号的机制 —— [示例](https://www.cnblogs.com/nexiyi/p/java_add_ShutdownHook.html "Java接受SIGTERM信号的用法")  。

spring服务在接收到SIGTERM信号后会回收线程，终止定时任务，spring cloud服务发现模块会主动向eureka请求下线。但仍会存在问题：
(1) 处理中的任务会被直接打断，无法做妥善处理。比如sleep阻塞的线程会直接报异常 InterruptedException，执行中的定时任务会被直接终止退出。
(2) kill指令发出SIGTERM信号后，框架会自动向eureka发送下线通知。但其他微服务感应到需要一定时间，在此之前还会有请求。而关停服务spring框架底层已经在做停止处理，容易产生异常。

这两个问题前者需要提前通知服务要被关停并留足时间处理；后者需要微服务得到关停通知后主动向eureka下线。
### 2、代码设计要求
为了主动停止服务，业务代码要做到以下要求

- 时间长的定时任务拆分成多个短时子任务，每个子任务执行前都检查服务状态，如果为终止则保存数据并退出。
- 未执行或执行到一半因关停而终止的子任务后续能检测出来并恢复执行,避免无法消除异常数据。

### 3、Spring cloud+eureka的优雅启停方案
只基于spring cloud+eureka体系做开发也很常见，所以先了解这个体系本身解决优雅启停的方案。
#### 3.1微服务下线快速感知
微服务下线后需快速被其他微服务节点感知，这样才能避免微服务下线后其他服务还持续请求。

要做到下线快速感知有以下参数要注意配置。

eureka server端：
```
eureka:
  server:
    evictionIntervalTimerInMs: 5000  #启用主动失效，并且每次主动失效检测间隔为5000ms
    responseCacheUpdateIntervalMs: 5000 #从ReadWriteMap刷新节点信息到ReadOnlyMap的时间，client读取的是后者，默认30s
```
eureka server端对微服务节点信息的记录有两层，ReadWriteMap和ReadOnlyMap。最新的变化在ReadWriteMap，一段时间后才会更新到 ReadOnlyMap。client读取的是 ReadOnlyMap，所以 responseCacheUpdateIntervalMs 设置为3-5s较好。

另外很多文章解释快速感知时，会提到配置——enableSelfPreservation，即自我保护开关。大意是当eureka发现节点的心跳大面积异常时就持续一段时间（默认5分钟，可配置）不更新节点信息，期间管理页面有红字警告。因为这时可能是eureka server网络异常，保留错误数据也优于彻底瘫痪。但实测自我保护机制不会干扰服务主动下线时启停的感知，不用刻意设置。

eureka client端：
```
eureka:
  instance:
    #服务刷新时间，每隔这个时间会主动心跳一次
    leaseRenewalIntervalInSeconds: 3   
    #服务过期时间,超过这个时间没有接收到心跳EurekaServer就会将这个实例剔除
    leaseExpirationDurationInSeconds: 10   
  client:
    fetchRegistry: true   #定期更新eureka server拉去服务节点清单，快速感知服务下线
    registryFetchIntervalSeconds: 5  #eureka client刷新本地缓存时间，默认30s
```
这四个配置前两者保证服务上下线都快速被eureka server感知，后两者保证快速感知其他微服务的上下线。

但leaseRenewalIntervalInSeconds 和 leaseExpirationDurationInSeconds两个配置的时间对于主动下线的感知没有影响。后面提到的优雅启停方案都是属于主动下线，这两个配置可以自行调整。

#### 3.2启动问题处理方案
1.1提到的两个启动问题：
- 微服务自动向eureka注注册时数据预加载还未完成，是spring cloud本身就要处理的问题
- 第二个问题外部过早导入请求，则需要服务提供接口方便外部获取就绪情况

这里先说明第一个问题的解决方案。
直接将预加载代码放在static代码块或者PostConstruct标注的代码块，使预加载成为程序启动的一部分，这样自动注册必然发生在预加载后。
为了方便对外提供状态，记录加载完成状态为true，并记录时间。下面为示例代码:

```
    /**
     * 服务是否已经启动
     */
    public volatile static boolean isStart = false;
    /**
     * 服务启动的时间
     */
    public volatile static long startTime = 0;
    /**
     * 服务是否处于准备关闭的状态，这个时候一些定时任务就应该及时退出了
     */
    public volatile static boolean isPreStop = false;
    
    public static void main(String[] args) {
        SpringApplication.run(xxxApplication.class, args);
        logger.info("服务启动完毕");
        startTime = System.currentTimeMillis();
        isStart = true;
    }

    @PostConstruct
    public void preload(){
        logger.info("预加载完毕");
    }
```

接着说第二个问题，提供接口查询是否就绪。

接口逻辑：
- 如果没就绪就返回500的http code。
- 由于服务启动后还要一定时间才能注册到eureka并被其他微服务感知，即使自身已就绪也要延长一定时间。 这个时间应该至少是 responseCacheUpdateIntervalMs + registryFetchIntervalSeconds，可以再增加几秒。

示例代码如下，其中isPreStop变量在后面停止方案部分会赋值，主要是为了避免服务准备停机了ifReady仍然返回true。

```
    public boolean ifReady() {
        long timeCur = System.currentTimeMillis();
        if (isStart && !isPreStop && timeCur - startTime > 16000) {
            return true;//success
        } else {
            //TODO:throw Exception在外层做异常处理并返回500的http code
        }
    }
```

#### 3.3停止问题解决方案
停止服务两个问题的关键在于
- 关机脚本主动通知服务要停止，并预留处理时间；
- 主动向eureka发送下线通知

上面两个要求spring cloud+eureka有多种解决方案，下面只介绍最简单易用的方案。
- 服务提供一个http接口专门用于下线通知并在接口里做服务终止处理，调用接口返回成功后预留一定时间再调用kill -9杀死服务。
- 在上一步提到的http接口中向eureka发送下线通知,如下方实例。由于其他微服务感应到要一定时间，下线服务仍要运行一段时间，和启动时的等待时间类似，至少要有 responseCacheUpdateIntervalMs + registryFetchIntervalSeconds ，为了保险可以多几秒。
```
    @Autowired
    private EurekaAutoServiceRegistration autoServiceRegistration;
    
    public boolean preStop() {
        preStopStartTime = System.currentTimeMillis();
        isPreStop = true;
        autoServiceRegistration.stop();
        try {
            Thread.sleep(16000);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
       //TODO:这里才能做回收http接口要用到的资源的操作，包括数据缓存等
       return true;
    }
```
上面的方案可以通过curl指令手工调用http接口，或结合shell脚本、jenkins脚本等做自动化处理。

另外网上很多文章使用EurekaClient对象shutDown方法，但是我验证该方法无法真正下线服务，eurekaServer端会报以下异常。
```
Cancelled instance xxx（实例名称） (replication=false)
DS: Registry: cancel failed because Lease is not registered for: xxxx
Not Found (Cancel): xxx
```

#### 3.4小结

3.1提到的eureka client端fetchRegistry 、registryFetchIntervalSeconds两个配置和后面两个章节提到的就绪检测接口、prestop接口的预留时间也就是sleep的时间注意系统内保持一致。
否则微服务之间会出现实际上已经停机了，另外的程序应然未感应到的情况。

### 4、k8s发布时关联spring cloud启停方案
使用k8s发布服务默认使用的滚动发布方案，这个方案本身已经有一定机制减少发布的影响。滚动发布时发布完一个新版本的pod后才会下线一个旧的pod，并把指向sevice的请求经负载均衡指向新pod，直到所有旧的pod下线，新的pod全部发布完毕。

所以只要k8s在pod的启停时做到和微服务联动，就可以做到无感发布。关键在于探知微服务是否准备好了、通知服务将要停止、配置启停过程预留的时间。这几个方面k8s都有相关的机制，所以我们先了解这些机制，再整合得出解决思路。
#### 4.1 k8s相关机制或配置
##### 4.1.1 启动后固定较长就绪时间
通过属性 minReadySeconds 设置pod启动后多长时间才认为就绪,默认为0s。
##### 4.1.2 探针机制
k8s提供了两种探针机制，就绪探针 readinessProbe、存活探针 livenessProbe。

探针机制可以通过http接口、shell指令、tcp确认容器状态。
可以配置延迟探测时间、探测间隔、探测成功或失败条件延后时间等参数。
http接口探测如果响应状态码大于等于200 且小于 400 则诊断为成功。

- 存活探针，主要用于检测pod是否异常，如果异常会替换或重启容器
- 就绪探针，探测通过时才会将其加入到service匹配的endpoint列表，并向该容器发送请求，否则会将pod从列表移除直到就绪探针再次通过

就绪探针和存活探针比较类似，都会持续执行检测，只是检测会导致的结果不一样，前者决定外部请求何时可以分发到服务，后者决定容器是否重启或被替换。

探针机制详细介绍可以查看 [这里](https://kubernetes.io/zh/docs/concepts/workloads/pods/pod-lifecycle/#容器探针 "容器探针的说明") 。

详细的配置可以查看 [这里](https://kubernetes.io/docs/tasks/configure-pod-container/configure-liveness-readiness-startup-probes/#define-a-liveness-http-request "探针的配置")。但是要注意这个文档里还提到启动探针机制，但笔者尝试配置无发生效，不确定是否和版本有关，所以对其不做介绍。

##### 4.1.3 terminationGracePeriodSeconds 配置延迟关闭时间
该属性默认30s，只配置terminationGracePeriodSeconds属性而没有prestop时，k8s先发送SIGTERM信号给主进程，等待 terminationGracePeriodSeconds 时长后使用SIGKILL杀死进程，仍未处理完毕请求会失败。
##### 4.1.4 prestop 机制
prestop机制，容器生命周期钩子中的一种，在准备关闭pod时调用。这个接口调用至少一次，有可能多次，需要做好幂等处理。更详细资料可以参考官方文档 [这里](https://kubernetes.io/zh/docs/concepts/containers/container-lifecycle-hooks/#容器钩子 "容器钩子说明")。这个机制和就绪探针类似，也可以通过http接口、shell指令两种方式进行。

该机制跟terminationGracePeriodSeconds有关联，k8s根据prestop设置的方式通知服务要关停，GracePeriod时间后如果仍未有结果就直接发送SIGTERM信号，等待2s后使用SIGKILL杀死主进程。更详细步骤可以查看 [这里](https://kubernetes.io/docs/concepts/workloads/pods/pod/#termination-of-pods "Pod关闭流程说明")

prestop对应接口或者指令执行时间不宜过长，30s以内为宜。k8s在pod关闭过程中会先同时部署新pod，如果耗时过长会导致资源消耗较大。

##### 4.1.5 以上几类机制的配置示例
提供上述几个机制的deployment文件配置示例如下
```
apiVersion: extensions/v1beta1
kind: Deployment
metadata:
  name: review-demo
  namespace: scm
  labels:
    app: review-demo
spec:
  replicas: 3
#  minReadySeconds: 60     #滚动升级时60s后认为该pod就绪
  strategy:
    rollingUpdate:  ##由于replicas为3,则整个升级,pod个数在2-4个之间
      maxSurge: 1      #滚动升级时会先启动1个pod
      maxUnavailable: 1 #滚动升级时允许的最大Unavailable的pod个数
  template:
    metadata:
      labels:
        app: review-demo
    spec:
      terminationGracePeriodSeconds: 60 ##k8s将会给应用发送SIGTERM信号，可以用来正确、优雅地关闭应用,默认为30秒
      containers:
      - name: review-demo
        image: library/review-demo:0.0.1-SNAPSHOT
        imagePullPolicy: IfNotPresent
        lifecycle:
          preStop:
            httpGet:
              path: /prestop
              port: 8080
              scheme: HTTP            
        livenessProbe: #kubernetes认为该pod是存活的,不存活则需要重启
          httpGet:
            path: /health
            port: 8080
            scheme: HTTP
            httpHeaders:
              - name: Custom-Header
              value: Awesome               
          initialDelaySeconds: 60 ## equals to the max startup time of the application + couple of seconds
          timeoutSeconds: 10
          successThreshold: 1
          failureThreshold: 5
          periodSeconds: 5 # 多少秒执行一次检测
        readinessProbe: #kubernetes认为该pod是准备好接收http请求了的
          httpGet:
            path: /ifready
            port: 8080
            scheme: HTTP
            httpHeaders:
              - name: Custom-Header
              value: Awesome            
          initialDelaySeconds: 30 #equals to min startup time of the app
          timeoutSeconds: 10
          successThreshold: 1
          failureThreshold: 5
          periodSeconds: 5 # 多少秒执行一次检测
        resources:
          # keep request = limit to keep this container in guaranteed class
          requests:
            cpu: 50m
            memory: 200Mi
          limits:
            cpu: 500m
            memory: 500Mi
        env:
          - name: PROFILE
            value: "test"
        ports:
          - name: http
            containerPort: 8080
```
#### 4.2、基于以上机制解决启动问题
k8s的就绪探针配合3.2提到的spring cloud+eureka程序本身的启动和注册机制即可解决1.1提到的微服务启动未完成就被注册到eureka以及被k8s导入请求的问题。

#### 4.3、基于机制解决停机问题
1.2提到的k8s发布spring cloud服务时停机过程的问题结合3.3提到的方案，使用k8s preStop机制调用微服务的服务终止处理接口即可解决。

#### 4.4、更严谨的方案
4.3提供的方案可以满足绝大部分场景了，但对高要求场景仍有不足。比Pod在关闭过程中的善后处理时可能会超出预期时间，希望取消pod重启或升级方案。

如果系统要求比较高可以考虑使用k8s的ApiServer（可以通过接口管理pod等的增删查改和启停）结合WebHook机制访问自定义接口确认服务状态。具体参考[此文章](https://cloud.tencent.com/developer/article/1409225 "k8s高级管理方法") 的 另辟蹊径：解耦 Pod 删除的控制流 部分。

### 5、试验
为了验证这个优雅启停方案，基于k8s环境使用两个spring cloud+eureka服务试验。服务A为认证服务，两个实例，提供token检查接口。服务B调用A的token校验接口。

不使用以上的优雅启停方案时，使用wrk持续访问B服务接口使其调用A服务接口。期间对A服务发版，结果B服务出现大量访问A服务接口不通的日志。

使用文章提到的优雅启停方案后，A服务发版时服务B请求其接口不再失败。

### 6、总结

前面4.2、4.3提到的方案虽然在一些极端情况，可能还会产生异常数据，但对于非金融场景已经够用。

