# TLog轻量级分布式日志标记追踪
> TLog提供了一种最简单的方式来解决日志追踪问题，它不收集日志，也不需要另外的存储空间，它只是自动的对你的日志进行打标签，自动生成TraceId贯穿你微服务的一整条链路。并且提供上下游节点信息。适合中小型企业以及想快速解决日志追踪问题的公司项目使用

> 支持dubbo，dubbox，spring cloud三大RPC框架

## TLog安装
> 在SpringBoot项目引入全量依赖
```
<dependency>
    <groupId>com.yomahub</groupId>
    <artifactId>tlog-all-spring-boot-starter</artifactId>
    <version>1.3.6</version>
</dependency>
```

## 适配方式
> 支持Log4j、Logback、Log4j2三大日志框架

> 1. Logback框架适配器
> > 1.1 同步日志
换掉encoder的实现类或者换掉layout的实现类就可以了
```
<?xml version="1.0" encoding="UTF-8"?>
<configuration debug="false">
    <property name="APP_NAME" value="logtest"/>
    <property name="LOG_HOME" value="./logs" />
    <appender name="STDOUT" class="ch.qos.logback.core.ConsoleAppender">
        <!--这里替换成AspectLogbackEncoder-->
        <encoder class="com.yomahub.tlog.core.enhance.logback.AspectLogbackEncoder">
              <pattern>%d{yyyy-MM-dd HH:mm:ss.SSS} [%thread] %-5level %logger{50} - %msg%n</pattern>
        </encoder>
    </appender>
    <appender name="FILE"  class="ch.qos.logback.core.rolling.RollingFileAppender">
        <File>${LOG_HOME}/${APP_NAME}.log</File>
        <rollingPolicy class="ch.qos.logback.core.rolling.SizeAndTimeBasedRollingPolicy">
            <FileNamePattern>${LOG_HOME}/${APP_NAME}.log.%d{yyyy-MM-dd}.%i.log</FileNamePattern>
            <MaxHistory>30</MaxHistory>
            <maxFileSize>1000MB</maxFileSize>
        </rollingPolicy>
        <!--这里替换成AspectLogbackEncoder-->
        <encoder class="com.yomahub.tlog.core.enhance.logback.AspectLogbackEncoder">
            <pattern>%d{yyyy-MM-dd HH:mm:ss.SSS} [%thread] %-5level %logger{50} - %msg%n</pattern>
        </encoder>
    </appender>

    <root level="INFO">
        <appender-ref ref="STDOUT" />
        <appender-ref ref="FILE" />
    </root>
</configuration>
```
> > 1.2. 异步日志
替换掉appender的实现类就可以了
```
<?xml version="1.0" encoding="UTF-8"?>
<configuration debug="false">
    <property name="APP_NAME" value="logback-dubbo-provider"/>
    <property name="LOG_HOME" value="./logs" />
    <appender name="STDOUT" class="ch.qos.logback.core.ConsoleAppender">
        <encoder class="ch.qos.logback.classic.encoder.PatternLayoutEncoder">
            <pattern>%d{yyyy-MM-dd HH:mm:ss.SSS} [%thread] %-5level %logger{50} - %msg%n</pattern>
        </encoder>
    </appender>
    <appender name="FILE"  class="ch.qos.logback.core.rolling.RollingFileAppender">
        <File>${LOG_HOME}/${APP_NAME}.log</File>
        <rollingPolicy class="ch.qos.logback.core.rolling.SizeAndTimeBasedRollingPolicy">
            <FileNamePattern>${LOG_HOME}/${APP_NAME}.log.%d{yyyy-MM-dd}.%i.log</FileNamePattern>
            <MaxHistory>30</MaxHistory>
            <maxFileSize>1000MB</maxFileSize>
        </rollingPolicy>
        <encoder class="ch.qos.logback.classic.encoder.PatternLayoutEncoder">
            <pattern>%d{yyyy-MM-dd HH:mm:ss.SSS} [%thread] %-5level %logger{50} - %msg%n</pattern>
        </encoder>
    </appender>

    <!-- 这里替换成AspectLogbackAsyncAppender -->
    <appender name="ASYNC_FILE" class="com.yomahub.tlog.core.enhance.logback.async.AspectLogbackAsyncAppender">
        <discardingThreshold>0</discardingThreshold>
        <queueSize>2048</queueSize>
        <includeCallerData>true</includeCallerData>
        <appender-ref ref="FILE"/>
    </appender>

    <root level="INFO">
        <appender-ref ref="STDOUT" />
        <appender-ref ref="ASYNC_FILE" />
    </root>
</configuration>
```

## 日志标签模板自定义
> TLog默认只打出spanId和traceId，以<$spanId><$traceId>这种模板打出，当然你能自定义其模板。还能加入其它的标签头  
> 你只需要在springboot的application.properties里如下定义：
```
tlog.pattern=[$preApp][$preIp][$spanId][$traceId]
```
>$preApp：上游微服务节点名称  
> $preHost： 上游微服务的Host Name  
> $preIp：上游微服务的IP地址  
> $spanId：链路spanId  
> $traceId：全局唯一跟踪ID

这样日志的打印就能按照你定义模板进行打印

## 标签位置自定义
> TLog默认的标签都是跟在具体日志信息体的前面的，但是如果想自定义位置怎么办，比如放在[INFO]的前面，有没有办法？
> TLog从1.1.5开始，适配了slf4j的mdc，从而能够让你的标签在一行日志的任意位置打印出。但是只支持日志框架配置方式接入
> TLog中log4j和logback的占位符为%X{tl}，而在log4j2中的占位符为%TX{tl}，请注意不要写错
> 你可以在任意日志框架的pattern里定义。例如在logback里，你可以这样定义pattern：

```
<?xml version="1.0" encoding="UTF-8"?>
<configuration debug="false">
    <!--定义日志文件的存储地址 勿在 LogBack 的配置中使用相对路径-->
    <property name="LOG_HOME" value="./logs" />
    <!--控制台日志-->
    <appender name="Console" class="ch.qos.logback.core.ConsoleAppender">
        <!--这里替换成AspectLogbackEncoder-->
        <encoder class="com.yomahub.tlog.core.enhance.logback.AspectLogbackEncoder">
            <pattern>%d{yyyy-MM-dd HH:mm:ss.SSS} %X{tl} [%thread] %-5level %logger{50} - %msg%n</pattern>
        </encoder>
    </appender>

    <!--文件日志-->
    <!--同步日志-->
    <appender name="SyncLogFile"  class="ch.qos.logback.core.rolling.RollingFileAppender">
        <File>${LOG_HOME}/logback-sync-rolling-mdc.log</File>
        <rollingPolicy class="ch.qos.logback.core.rolling.SizeAndTimeBasedRollingPolicy">
            <!--日志文件输出的文件名-->
            <FileNamePattern>${LOG_HOME}/logback-rolling-mdc.log.%d{yyyy-MM-dd}.%i.log</FileNamePattern>
            <!--日志文件保留天数-->
            <MaxHistory>30</MaxHistory>
            <!--日志文件大小-->
            <maxFileSize>1000MB</maxFileSize>
        </rollingPolicy>
        <encoder class="com.yomahub.tlog.core.enhance.logback.AspectLogbackEncoder">
            <!--格式化输出：%d表示日期，%thread表示线程名，%-5level：级别从左显示5个字符宽度%msg：日志消息，%n是换行符-->
            <pattern>%d{yyyy-MM-dd HH:mm:ss.SSS} %X{tl} [%thread] %-5level %logger{50} - %msg%n</pattern>
        </encoder>
    </appender>

    <!-- 日志输出级别 -->
    <root level="INFO">
        <appender-ref ref="Console" />
        <appender-ref ref="SyncLogFile" />
    </root>
</configuration>
```

那么出来的日志范例为：

```
2020-11-07 17:01:26.647 <0.1><7455616963803840> [Thread-8] INFO  c.y.tlog.example.dubbox.service.impl.AsynDomain - 这是测试日志哦
```

在log4j和log4j2中同理。

## SpanId的生成规则
> TLog默认的标签打印模板是<$spanId><$traceId>

> TLog 中的 SpanId 代表本次调用在整个调用链路树中的位置，假设一个 Web 系统 A 接收了一次用户请求，那么在这个系统的日志中，记录下的 SpanId 是 0，代表是整个调用的根节点，如果 A 系统处理这次请求，需要通过 RPC 依次调用 B，C，D 三个系统，那么在 A 系统的客户端日志中，SpanId 分别是 0.1，0.2 和 0.3，在 B，C，D 三个系统的服务端日志中，SpanId 也分别是 0.1，0.2 和 0.3

> 如果 C 系统在处理请求的时候又调用了 E，F 两个系统，那么 C 系统中对应的客户端日志是 0.2.1 和 0.2.2，E，F 两个系统对应的服务端日志也是 0.2.1 和 0.2.2。根据上面的描述，我们可以知道，如果把一次调用中所有的 SpanId 收集起来，可以组成一棵完整的链路树

> 我们假设一次分布式调用中产生的 TraceId 是 0a1234（实际不会这么短），那么根据上文 SpanId 的产生过程，有下图：

![image](../image.png)

## 业务自定义标签
> 很多公司的系统在打日志的时候，每打一个日志里都会带入一些业务信息，比如记录ID，会员CODE，方便业务日志的定位。现在有了TLog，不仅能做分布式链路标签追加，还能自动帮你做业务标签的添加。这样在定位日志的时候可以更加方便的搜索
>
> Tlog支持方法级别的自定义业务标签。你可以在方法上定义简单的标注，来实现在某一个方法的日志里，统一加入业务的指标标签，用于更加细致的定位
>
> 使用@TLogAspect
> 示例一 打印入参
```
@TLogAspect(value = {"id","name"},pattern = "<-{}->",joint = "_")
public void demo(String id,String name){
  log.info("加了patter和joint的示例");
}
```
> 日志打出来的样子如下，其中前面为框架spanId+traceId：
```
2020-02-08 22:09:40.103 [main] INFO  Demo - <0.2><7205781616706048><-NO1234_jenny-> 加了patter和joint的示例
```

> 示例二 常量标签
```
@TLogAspect(str = "XYZ")
public void demo1(String name){
  log.info("这是第一条日志");
  log.info("这是第二条日志");
  log.info("这是第三条日志");
  new Thread(() -> log.info("这是异步日志")).start();
}
```
```
2020-02-08 20:22:33.945 [main] INFO  Demo - <0.2><7205781616706048>[XYZ] 这是第一条日志
2020-02-08 20:22:33.945 [main] INFO  Demo - <0.2><7205781616706048>[XYZ] 这是第二条日志
2020-02-08 20:22:33.945 [main] INFO  Demo - <0.2><7205781616706048>[XYZ] 这是第三条日志
2020-02-08 20:22:33.948 [Thread-3] INFO  Demo - <0.2><7205781616706048>[XYZ] 这是异步日志
```
> 示例三 点操作符
> 支持类型:
> > Bean对象Map对象  
> > Json格式的字符串  
> > Fastjson的JSONObject对象
```
@TLogAspect({"person.id","person.age","person.company.department.dptId"})
public void demo(Person person){
  log.info("多参数加多层级示例");
}
```

```
2020-02-08 22:09:40.110 [main] INFO  Demo - <0.2><7205781616706048>[31-25-80013] 多参数加多层级示例
```
> 可以用点操作符操作符合Json格式的字符串
```
@TLogAspect({"person.id","person.age","person.company.department.dptId"})
public void demo(String person){
  log.info("多参数加多层级示例");
}
```
> 甚至可以用下标[num]去访问Json格式的字符串中的数组
```
@TLogAspect({"person.id","person.age","person.company.title[1]"})
public void demo(String person){
  log.info("多参数加多层级示例");
}
```

> 示例四 自定义Convertor
> 自定义Convert，适用于更复杂的业务场景
```
@TLogAspect(convert = CustomAspectLogConvert.class)
public void demo(Person person){
  log.info("自定义Convert示例");
}
```

```
public class CustomAspectLogConvert implements AspectLogConvert {
    @Override
    public String convert(Object[] args) {
        Person person = (Person)args[0];
        return "PERSON(" + person.getId() + ")";
    }
}

```

```
2020-02-20 17:05:12.414 [main] INFO  Demo - <0.2><7205781616706048>[PERSON(31] 自定义Convert示例
```

## 线程支持
### 1. 一般异步线程
> 异步线程的定义为：执行好之后线程会自动销毁，如new Thread(){…}.start这种内部类线程，你自定定义的类继承Thread或实现Runnerable接口的
> 对于一般异步线程，不需要你做任何事，TLog天然支持在异步线程中打印标签。
>

### 2. 线程池
> 但是对于使用了线程池的场景，由于线程池中的线程不会被销毁，会被复用。需要你用TLogInheritableTask替换Runnable，否则标签数据会重复
```
ExecutorService pool = Executors.newFixedThreadPool(5);
pool.submit(new TLogInheritableTask() {
    @Override
    public void runTask() {
      log.info("我是异步线程日志");
    }
});
```

### 3.MDC模式的异步线程
> 如果你用了标签位置自定义功能(这个功能是用slf4j的MDC来实现的)，那么你有可能发现，在MDC模式中的异步线程好像都没法正常打印标签。这是因为MDC中使用了ThreadLocal，而ThreadLocal中的信息没法传递给子线程。为此TLog提供了线程的包装类，能帮助你解决这个问题
> 还是同样的类TLogInheritableTask，如果你声明一个线程，你需要去继承TLogInheritableTask：
```
public class AsynDomain extends TLogInheritableTask {

    private Logger log = LoggerFactory.getLogger(this.getClass());

    @Override
    public void runTask() {
        log.info("这是异步方法哦");
        log.info("异步方法开始");
        log.info("异步方法结束");
    }
}
```
> 执行的时候和普通继承了Thread一样：
> new AsynDomain().start();
> 如果你要在线程池中执行，同样以AsynDomain为例，则为：
> ExecutorService pool = Executors.newFixedThreadPool(5);pool.submit(new AsynDomain());

## 对HTTPClient的支持
> 目前TLog对于Httpclient客户端的支持是有侵入性的，对于请求端需要加入拦截器，把日志标签信息记录用过Http Header传递到服务器
> HttpClient 4.X 添加拦截器TLogHttpClientInterceptor
```
String url = "http://127.0.0.1:2111/hi?name=2323";
CloseableHttpClient client = HttpClientBuilder.create()
            .addInterceptorFirst(new TLogHttpClientInterceptor())
            .build();
HttpGet get = new HttpGet(url);
try {
    CloseableHttpResponse response = client.execute(get);
    HttpEntity entity  = response.getEntity();
    log.info("http response code:{}", response.getStatusLine().getStatusCode());
    log.info("http response result:{}",entity == null ? "" : EntityUtils.toString(entity));
} catch (IOException e) {
    e.printStackTrace();
}
```

> HttpClient 5.X 添加拦截器TLogHttpClient5Interceptor
>
```
String url = "http://127.0.0.1:2111/hi?name=2323";
CloseableHttpClient client = HttpClientBuilder.create()
            .addInterceptorFirst(new TLogHttpClient5Interceptor())
            .build();
HttpGet get = new HttpGet(url);
try {
    CloseableHttpResponse response = client.execute(get);
    HttpEntity entity  = response.getEntity();
    log.info("http response code:{}", response.getStatusLine().getStatusCode());
    log.info("http response result:{}",entity == null ? "" : EntityUtils.toString(entity));
} catch (IOException e) {
    e.printStackTrace();
}
```

## 对mq中间件的支持
> 目前Tlog对于消息中间件的支持是有侵入性的，暂时无法做到完全无侵入
> 对于客户端，你只需要用TLogMqWrapBean包装你的业务bean就可以了
> TLogMqWrapBean<BizBean> tLogMqWrap = new TLogMqWrapBean(bizBean);mqClient.send(tLogMqWrap);
>对于消费者端，你需要这么做：

```
//从mq里接受到tLogMqWrapBean
TLogMqConsumerProcessor.process(tLogMqWrapBean, new TLogMqRunner<BizBean>() {
    @Override
    public void mqConsume(BizBean o) {
        //业务操作
    }
});
```

## 对SpringCloud Gateway的支持
> TLog从1.3.0开始，对spring cloud gateway也进行了支持
> 你只需在gateway的server项目中引入tlog-web-spring-boot-starter和tlog-gateway-spring-boot-starter模块即可
> TLog会在gateway启动时进行自动装载适配。
> 如果进行了全需依赖导入，则无需再进行按需依赖导入
>

## 自动打印调用参数和时间
> 对于springboot应用来说，你只需在springboot的配置文件中作如下配置
```
# 不配默认为false
tlog.enable-invoke-time-print=true
```
> 当你进行RPC调用时，会自动打印出参数信息和调用时间信息：
```
2020-12-01 19:20:07.768 [DubboServerHandler-127.0.0.1:30900-thread-2] INFO  c.y.tlog.dubbo.filter.TLogDubboInvokeTimeFilter - <0.1><7592057736843136> [TLOG]开始调用接口[DemoService]的方法[sayHello],参数为:["jack"]
2020-12-01 19:20:07.787 [DubboServerHandler-127.0.0.1:30900-thread-2] INFO  c.y.t.example.dubbo.service.impl.DemoServiceImpl - <0.1><7592057736843136> logback-dubbox-provider:invoke method sayHello,name=jack
2020-12-01 19:20:07.788 [Thread-14] INFO  c.y.tlog.example.dubbo.service.impl.AsynDomain - <0.1><7592057736843136> 这是异步方法哦
2020-12-01 19:20:07.788 [Thread-14] INFO  c.y.tlog.example.dubbo.service.impl.AsynDomain - <0.1><7592057736843136> 异步方法开始
2020-12-01 19:20:07.789 [Thread-14] INFO  c.y.tlog.example.dubbo.service.impl.AsynDomain - <0.1><7592057736843136> 异步方法结束
2020-12-01 19:20:07.795 [DubboServerHandler-127.0.0.1:30900-thread-2] INFO  c.y.tlog.dubbo.filter.TLogDubboInvokeTimeFilter - <0.1><7592057736843136> [TLOG]结束接口[DemoService]中方法[sayHello]的调用,耗时为:90毫秒
```

## 自定义TraceId生成器
> TLog默认采用snowflake算法生成traceId，当然你也可以去更换traceId的生成算法。
> 定义自己的traceId生成类去继承TLogIdGenerator接口：
```
public class TestIdGenerator extends TLogIdGenerator {
   @Override
   public String generateTraceId() {
      return String.valueOf(System.nanoTime());
   }
}
```
> 然后在springboot的配置类里声明：
> tlog.id-generator=com.yomahub.tlog.example.dubbo.id.TestIdGenerator

## SpringCloud的Openfeign
> 如果你的RPC是spring cloud，需要在spring xml里如下配置

```
<bean class="com.yomahub.tlog.feign.filter.TLogFeignFilter"/>
<bean class="com.yomahub.tlog.core.aop.AspectLogAop"/>
```
> 同时需要在spring mvc的xml里做如下配置
```
<mvc:interceptors>
  <bean class="com.yomahub.tlog.web.interceptor.TLogWebInterceptor" />
</mvc:interceptors>
```



