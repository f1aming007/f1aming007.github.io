# Serverless Openfaas


# Serverless 介绍
serverless 指的是由开发者实现的服务端逻辑运行在无状态的计算容器中, 它由事件触发，完全由第三方管理，其业务层面的状态则被开发者使用的数据库和存储资源所记录 
<!--more-->


![](media/17034703209918.jpg)

![](media/17034706615678.jpg)

Baas： 后端及服务， 一般是一个个的API调用后端或者别人已经别人已经实现好的逻辑

![](media/17034706843961.jpg)

Faas： 函数即服务， 本质上是一种事件驱动的由消息触发的服务， Faas供应商一般会集成各种同步和异步的事件源， 通过订阅这些事件源， 可以突发或者定期的触发函数运行

### Faas的优势和不足
**优势**
* 降低人力成本
* 降低风险
* 减少资源开销
* 增加伸缩的灵活性
* 缩短创新周期

**不足**
* 不适合长时间运行的应用
* 状态管理
* 冷启动时间
* 缺乏调试和开发工具
* 构建复杂

### 应用场景
* 发送通知
* webhhok
* 轻量级API
* 物联网
* 数据统计分析
* Trigger和定时任务
* 聊天机器人


## Faas 蓝图
![](media/17034714952107.jpg)

# OpenFaas
OpenFaaS一款高人气的开源的faas框架，可以直接在Kubernetes上运行，也可以基于Swarm或容器运行

openFaas的架构图
![](media/17034718685089.jpg)
用到的镜像如下：

functions/faas-netesd:0.3.4
functions/gateway:0.6.14
functions/prometheus:latest-k8s
functions/alertmanager:latest-k8s

### 基础组件
* Gateway
* Provider
* queue-worker
* watchdog

**Gateway**：
* 为函数调用提供一个路由， 起到一个代理转发的作用
* 内置UI 界面， 可访问函数商店
* Prometheus 手机监控指标
*  接收AlertManager 通知， 自动伸缩


![](media/17034735721802.jpg)

![](media/17034737447497.jpg)

### 自动伸缩函数
* 根据每秒请求数
* 内存和CPU 使用量
* 最大和最小副本数
    * com.openfaas.sacle.min 最小
    * com.openfaas.sacle.max 最大
    * com.openfaas.sacle.factor 步长
* 手动调用 API 伸缩
* AlertManaer 触发
    * 获取现在副本数
    * 计算新副本数： min+(max/100)*factor
    * 调用 k8s API 设置新副本数

    
### watchdog
* 职责是调用函数， healthcheck 和超时
* 任何二进制都会被watchdog 变成一个函数
* 小型的http server

![](media/17034741867546.jpg)

### queue-worker
* 同步函数
    * 路由是/function/:name
    * 等待
    * 结束时得到结果
    * 明确知道是成功还是失败
* 异步函数
    * 路由是 /async-function/:name
    * http的状态码是202，  即时响应码
    * 从queue-worker中调用函数
    * 默认情况下，函数的执行结果是被丢弃的
