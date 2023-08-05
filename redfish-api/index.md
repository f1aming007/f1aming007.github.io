# Redfish Api


# Redfish Api 
* Redfish 利用常见的互联网和web 服务标准将信息直接提供给相关的工具链

* IPMI 是一种较早的带外管理管理工具，仅限`最小公共集`命令集（开机/关机/重启/温度值/文本控制台）还是需要带内管理软件
* 当管控设备多的时候， 需要统一管理， 需要对接不同供应商的API， 基础的IMPI功能不好满足横向扩展环境，需要更便捷的方式调用服务器高级管理功能新的需求
* `Redfish`可扩展平台管理API是一种新的规范，使用RESTful语句来访问定义在模型格式的数据，用于执行带外系统管理

## Redfish应用场景
用户期望能够批量管理服务器，比如笔者想一次性给100个服务器安装系统，并且这100个服务器并不都是同一厂商，不同厂商的IPMI操作都不一样，比如Dell是iDRAC，你还需要专门学习iDRAC使用和各种对接，这会带来很多困扰。而Redfish标准的出现彻底改变这种情况，它是凌驾于所有服务器之上的一个标准，对服务器的基本操作都是统一的，并且是基于Restful API的方式实现

## 选择REST、HTTP以及JSON
除了REST、HTTP和JSON之外，Redfish还采用常见的OData v4约定来描述模式、URL约定和命名，以及JSON有效负载中常见属性的结构。越来越多的通用客户端库、应用程序和工具生态系统使用Redfish。
它有多简单?下面显示了使用Redfish从服务器检索序列号的示例Python代码：此示例中的输出如下所示
```python
awData= urllib.urlopen(‘http://192.168.1.135/redfish/v1/Systems/1’) 
jsonData=json.loads(rawData)
print(jsonData[‘SerialNumber’])

1A87CA442K
```

## golang 的SDK
https://github.com/Nordix/go-redfish

* 封装客户端
```go
cfg := &redfish.Configuration{
    BasePath:      "http://localhost:8000",
    DefaultHeader: make(map[string]string),
    UserAgent:     "go-redfish/client",
  }

redfishApi := redfish.NewAPIClient(cfg).DefaultApi
```

* 列出可用的计算机系统
```go
sl, _, _ := redfishApi.ListSystems(context.Background())
```

* 重置计算机系统
```go
system_id := "dd9fd064-263b-469c-91d4-d45f341fe2c5"
systemReq := redfish.ComputerSystem{ResetType: "ForceRestart"}
reset_resp, _, _ = redfishApi.ResetSystem(context.Background(), system_id, reset_type)
```

* 给计算器插入虚拟媒体
```go
manager_id := "58893887-8974-2487-2389-841168418919"
insertReq := redfish.InsertMediaRequestBody{}
insertReq.Image = "http://releases.ubuntu.com/19.04/ubuntu-19.04-live-server-amd64.iso"
insertReq.Inserted = true
redfishApi.InsertVirtualMedia(context.Background(), manager_id, "Cd", insertReq)
```

* 为下次启动设置启动设备
```go
system_id := "dd9fd064-263b-469c-91d4-d45f341fe2c5"
systemReq := redfish.ComputerSystem{}
systemReq.Boot.BootSourceOverrideTarget = "Cd"
redfishApi.SetSystem(context.Background(), system_id, systemReq)
```

## Redfish实践
* [The python-redfish project](https://pythonhosted.org/python-redfish/readme.html)

* [Python环境redfish接口获取泰山服务器和鲲鹏CPU信息](https://bbs.huaweicloud.com/forum/thread-40434-1-1.html)
* redfish是当前主流的服务器监控协议，通过redfish协议可以通过带外管理通道获取服务器状态和详细硬件信息


```python
pip install python-redfish

import redfish
 
login_host="https://10.93.20.10"
login_account="ADMIN"
login_password="ADMIN"
REDFISH_OBJ = redfish.redfish_client(base_url=login_host, username=login_account, password=login_password, default_prefix='/redfish/v1')
REDFISH_OBJ.login(auth="session")
response = REDFISH_OBJ.get("/redfish/v1/Systems/1", None)
print(response)
REDFISH_OBJ.logout()
```

[参考链接](https://www.supermicro.org.cn/zh_cn/solutions/management-software/redfish)
## Supermicro Redfish 特点
* 获取系统/机箱信息
* 管理用户账号和权限
* BMC配置（AD、LDAP、SNMP、SMTP、RADIUS、Fan mode、Mouse mode、NTP、Snooping 等等）
* BIOS 配置
* 开机顺序变更
* RAID 储存装置管理（For Broadcom 3108, 3008, 3216, 3616 & Marvel SE9230）
* 磁盘管理
* 获取 NIC MAC 信息（NIC 资讯）
* BMC／BIOS／3108韧体更新
* 获取温度／电源／传感器讯息


