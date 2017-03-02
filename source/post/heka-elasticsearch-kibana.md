```toml
title = "基于Heka，ElasticSearch和Kibana的分布式后端日志架构（一）"
slug = "heka-elasticserach-kibana-1"
desc = "基于Heka，ElasticSearch和Kibana的分布式后端日志架构（一）"
date = "2017-03-02 23:17:33"
update_date = "2017-03-22 23:17:33"
author = "libi"
thumb = ""
tags = ["thrift","laravel"]
```

目前主流的后端日志都采用的标准的elk模式（Elasticsearch，Logstash，Kinaba），分别负责日志存储，收集和日志可视化。

不过介于我们的日志文件多样，分布在各个不同的服务器，各种不同的日志，为了日后方便二次开发定制。所以采用了Mozilla仿照Logstash使用golang开源实现的Heka。

#### 整体架构图

采用Heka，ElasticSearch和Kibana后的整体架构如下图所示

 ![日志系统](http://libisky.com/static/2017030201.png)

#### Heka篇

##### 简介

Heka对日志的处理流程为输入 分割 解码 过滤 编码 输出。单个Heka服务内部的数据流了通过Heka定义的Message数据模型在各个模块内进行流转。

heka内置了常用的大多数模块插件，比如

- 输入插件有Logstreamer Input可以将日志文件作为输入源，
- 解码插件Nginx Access Log Decoder可以将nginx访问日志解码为标准的键值对数据交给后边的模块插件进行处理。

得益于输入输出的灵活配置，可以将分散各地的Heka收集到的日志数据加工后统一输出到日志中心的Heka进行统一编码后交给ElasticSearch存储。

#### 安装

源码安装的方式较为繁琐这里就不再介绍，有需要可以参考官网文档。http://hekad.readthedocs.io/en/v0.10.0/installing.html

这里我们的linux发行版用的centos所以使用rpm包的安装方式。

下载rpm安装包

`wget https://github.com/mozilla-services/heka/releases/download/v0.10.0/heka-0_10_0-linux-amd64.rpm`

使用`rpm -i heka-0_10_0-linux-amd64.rpm`进行安装。

安装后执行 `hekad -version`输出版本号即安装成功。

#### 使用说明

在安装好之后使用 `heka -config xxx.toml`指定配置文件即可启动单个Heka服务。

配置文件内至少需要包括 输入 ，编码和输出配置。

#### 输入配置

因为需要收集的日志类型包括apphttp上报日志和服务器各类服务器日志，所以需要使用到输入插件包括Http Listen Input和Logstreamer Input。配置如下

##### Http Listen Input 

http listen input可以根据配置指定监听端口启动一个http服务，将日志数据post到该端口即可进行日志收集。

一般配置如下

```toml
[HttpListenInput]
address = "0.0.0.0:8325" #监听当前服务器的所有接口的8325端口
auth_type = "API" #接口认证方式为api,需要在请求头部加入X-API-KEY方可请求成功
api_key = "xxxx" #设定api_key
```

##### Logstreamer Input

Logstreamer Input可以根据配置监控指定目录内的指定日志文件，在日志产生变动时会增量将新增日志数据发送给Heka服务进行日志收集。

一般配置如下

```toml
[accesslogs]
type = "LogstreamerInput" 

#记录当前文件的读取位置保存目录
journal_directory = "/tmp/heka" 

#被监听日志文件目录 
log_directory = "/var/log/nginx" 

#正则匹配路径此处是匹配log_directory后面的路径,例如现在监听的文件路径为 #/var/log/nginx/2015/05/23/test.log 
file_match = '(?P<Year>\d+)/(?P<Month>\d+)/(?P<Day>\d+)/(?P<FileName>[^/]+)\.log' 

#排序,以match匹配到的年月日对文件进行排序依次监听读取 
priority = ["Year","Month","Day"] 

#日志的最后修改时间在oldest_duration时间后,则不会监听 
#heka 0.9.1及以前版本此处有bug,该设置无效 
oldest_duration = "1h" 

#分类名设置,内部是修改全局变量 Logger,以备后面对日志流做来源区分,默认则为数据处理类名 
differentiator = ["FileName","-","access"]

```

#### 解码设置

主要是对nginx日志和自己个性化的日志格式解码为标准的数据对象。这里用到了Nginx Access Log Decoder。

一般配置如下

```toml
#需要在输入设置指定解码器 decoder = "CombinedLogDecoder" 
[CombinedLogDecoder]
type = "SandboxDecoder"
filename = "lua_decoders/nginx_access.lua"

[CombinedLogDecoder.config]
type = "combined"
payload_keep = true
user_agent_transform = true
log_format = '$remote_addr - $remote_user'
```

#### 编码设置

##### ElasticSearch JSON Encoder

该插件是将之前流程处理好的数据编码后准备给后端的ElasticSearch使用。

配置如下

```toml
[ESJsonEncoder]
index = "%{Type}-%{%Y.%m.%d}" //设置写入ElasticSearch的索引值
es_index_from_timestamp = true
type_name = "%{Type}"//设置写入ElasticSearch的分类名
    [ESJsonEncoder.field_mappings]	//映射Heka内的数据键为es json格式的key值
    Timestamp = "@timestamp"
    Severity = "level"
```

#### 输出设置

##### **ElasticSearchOutput**

最后需要将Heka收集加工过的日志数据传入ElasticSearch。

配置如下

```toml
[ElasticSearchOutput]
message_matcher = "Type == 'sync.log'" #设置过滤条件 无不需要过滤 设置为true即可
server = "http://es-server:9200" #ElasticSearch 服务地址
flush_interval = 5000 #刷新间隔
flush_count = 10 #刷新次数
encoder = "ESJsonEncoder" #指定编码插件
```

#### 多个Heka串联设置

##### 介绍

在实际使用中因为各个服务器日志格式等不同，需要将多个heka收集后的日志数据汇总到es服务器中，我们当然可以使用直接将输出设置为es服务器地址存储，但是灵活性会大打折扣。在将来需要增加队列中间件将会无比复杂。

所有可以利用单个Heka作为中转，收集所有heka节点传入的日志输入统一处理输出到后端的es服务器。具体参见上面的架构图。

对于Heka节点间的通信使用何种协议，可以按照具体情况进行实施。

- 如对日志数据一致性要求高，并且不允许有遗漏，可以使用TCP Output和TCP Input分别作为输出和下一节点的输入进行对接。
- 如果为了追求最大性能，允许数据有少量遗漏，则可以使用UDP Output和UDP Input分别作为输出和下一节点的输入进行对接。

目前我们的日志系统主要是了收集错误及访问等日志信息，没有计费等方面的需求，所以为了节省资源。使用udp协议进行heka节点间的通讯协议。

##### 输出与输入配置

输出配置如下

```toml
[aggregator_output]
type = "TcpOutput"
address = "next-heka-server:55" #发送至后一heka节点地址
local_address = "127.0.0.1" #本机ip
message_matcher = "TRUE" #允许任何数据
```

输入配置如下

```toml
[UdpOutput]
address = "127.0.0.1:4880" #监听端口

[UdpInput.signer.ops_0] #用来标记消息的哈希值
hmac_key = "4865ey9urgkidls xtb0[7lf9rzcivthkm" 
[UdpInput.signer.ops_1]
hmac_key = "xdd908lfcgikauexdi8elogusridaxoalf"

[UdpInput.signer.dev_1]
hmac_key = "haeoufyaiofeugdsnzaogpi.ua,dp.804u"
```











