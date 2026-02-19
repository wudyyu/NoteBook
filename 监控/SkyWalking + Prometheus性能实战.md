# 基于 Spring Boot + SkyWalking + Prometheus 的性能工程闭环实战

## 从监控到压测，再到容量评估的工程化落地

>> 真正的性能工程不是“跑一次压测看看QPS”，而是建立一条可持续运转的工程闭环:
> >
> > 监控数据采集 → 性能瓶颈洞察 → 针对性压测 → 优化验证 → 容量评估 → 告警固化 → 持续回归
> >
> > 1. SkyWalking 负责 “看链路”
> >
> > 2. Prometheus 负责 “看资源和业务指标”
> >
> > 3. Grafana 负责 “统一表达事实”
>>
> > 4. 压测工具负责 “制造问题”
> >
> > 组合起来，才能构成完整的性能工程体系

> 一、性能工程闭环总体架构
> >
![197641ee3c72d41d3a21d61a97a2e4ac](../197641ee3c72d41d3a21d61a97a2e4ac.png)

> 二、监控体系搭建
>>
> > 1. Docker Compose 部署
```
version: '3.8'
services:
  prometheus:
    image: prom/prometheus
    ports:
      - "9090:9090"
    volumes:
      - ./prometheus.yml:/etc/prometheus/prometheus.yml

  grafana:
    image: grafana/grafana
    ports:
      - "3000:3000"
    environment:
      - GF_SECURITY_ADMIN_PASSWORD=admin

  skywalking-oap:
    image: apache/skywalking-oap-server
    environment:
      - SW_STORAGE=elasticsearch
    ports:
      - "11800:11800"
      - "12800:12800"

  skywalking-ui:
    image: apache/skywalking-ui
    environment:
      - SW_OAP_ADDRESS=http://skywalking-oap:12800
    ports:
      - "8080:8080"

  elasticsearch:
    image: elasticsearch:7.10.0
    environment:
      - discovery.type=single-node
```

> 三、Spring Boot 集成
>> 1. SkyWalking Agent
```
java -javaagent:/opt/skywalking/agent/skywalking-agent.jar \
 -Dskywalking.agent.service_name=order-service \
 -jar app.jar
```


>> 2. Prometheus 指标暴露（Micrometer）
```
<dependency>
  <groupId>io.micrometer</groupId>
  <artifactId>micrometer-registry-prometheus</artifactId>
</dependency>
```

```
management:
  endpoints:
    web:
      exposure:
        include: prometheus,health,info
  metrics:
    export:
      prometheus:
        enabled: true
```
>> 访问: http://localhost:8080/actuator/prometheus

> 四、Prometheus 抓取配置
>>
```
global:
  scrape_interval: 5s

scrape_configs:
  - job_name: 'springboot'
    metrics_path: '/actuator/prometheus'
    static_configs:
      - targets: ['host.docker.internal:8080']

  - job_name: 'skywalking'
    static_configs:
      - targets: ['skywalking-oap:1234']
```

> 五、Grafana 核心监控指标体系
>>
> > RED 指标（服务质量）

| 指标   | PromQL                                                                                                               |
|:------|:---------------------------------------------------------------------------------------------------------------------|
| QPS   | rate(http_server_requests_seconds_count[1m])                                                                         |
| P99   | histogram_quantile(0.99, sum(rate(http_server_requests_seconds_bucket[5m])) by (le))                                 |
| 错误率 | sum(rate(http_server_requests_seconds_count{status=~"5.."}[1m])) / sum(rate(http_server_requests_seconds_count[1m])) |

>> JVM & 系统资源
> >
```
# CPU
process_cpu_usage

# JVM 内存
jvm_memory_used_bytes{area="heap"}

# GC
rate(jvm_gc_pause_seconds_count[1m])
```

























