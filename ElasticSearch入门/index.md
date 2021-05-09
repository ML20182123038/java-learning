---
title: ElasticSearch入门
description: ElasticSearch介绍，环境搭建以及基本操作。
date: 2021-05-08
tags: 
    - ElasticSearch
categories: 
    - 开发者
---

##  一、ElasticSearch介绍

Elasticsearch 是一个`基于JSON的分布式、高扩展、高实时、RESTful 风格的搜索和数据分析引擎`，能够解决不断涌现出的各种用例。 作为 Elastic Stack 的核心，它集中存储您的数据，帮助您发现意料之中以及意料之外的情况。

Elasticsearch是与名为Logstash的数据收集和日志解析引擎以及名为Kibana的分析和可视化平台一起开发的。这三个产品被设计成一个集成解决方案，称为“Elastic Stack”（以前称为“ELK stack”）。

Elasticsearch可以用于搜索各种文档。它提供可扩展的搜索，具有接近实时的搜索，并支持多租户。Elasticsearch是分布式的，这意味着索引可以被分成分片，每个分片可以有0个或多个副本。每个节点托管一个或多个分片，并充当协调器将操作委托给正确的分片。再平衡和路由是自动完成的。“相关数据通常存储在同一个索引中，该索引由一个或多个主分片和零个或多个复制分片组成。一旦创建了索引，就不能更改主分片的数量。

Elasticsearch使用Lucene，并通过JSON和Java API提供其所有特性。它支持facetting和percolating，如果新文档与注册查询匹配，这对于通知非常有用。另一个特性称为“网关”，处理索引的长期持久性；例如，在服务器崩溃的情况下，可以从网关恢复索引。Elasticsearch支持实时GET请求，适合作为NoSQL数据存储，但缺少分布式事务。



## 二、ElasticSearch 7环境搭建

**ps：**全文基于 **ElasticSearch 7.12.1** ，需要java 8及以上环境，安装ElasticSearch相关软件/插件时，注意版本要一致。

### 1、安装ElasticSearch

官网下载[ElasticSearch](https://www.elastic.co/cn/start)

直接解压，运行bin目录下`elasticsearch.bat`

如果一切正常，浏览器访问`127.0.0.1:9200`,就会看到一下信息：

```json
{
  "name" : "DESKTOP-V4GSUJH",
  "cluster_name" : "elasticsearch",
  "cluster_uuid" : "4tnI-jAtTXqXbMDJ8CVRjQ",
  "version" : {
    "number" : "7.12.1",
    "build_flavor" : "default",
    "build_type" : "zip",
    "build_hash" : "3186837139b9c6b6d23c3200870651f10d3343b7",
    "build_date" : "2021-04-20T20:56:39.040728659Z",
    "build_snapshot" : false,
    "lucene_version" : "8.8.0",
    "minimum_wire_compatibility_version" : "6.8.0",
    "minimum_index_compatibility_version" : "6.0.0-beta1"
  },
  "tagline" : "You Know, for Search"
}
```

elasticsearch文件目录介绍：

```java
elasticsearch
    bin      :可执行文件。我们用来启动elasticsearch的脚本就在这里面
    config   :elastic-search的全局设置和你的具体设置，如果你需要更改JVM，数据路径，日志路径等，就需                要改这里。同时端口设置等也都在这里。
    data     :你的索引数据，即你存放具体用来搜索数据的地方
    jdk      :自带的JDK，不重要可忽略
    lib      :存放源码jar包
    logs     :存放一些日志文件
    modules  :自带的一些模块，不可删除。比如x-pack模块等（对我们学习不重要，可忽略）
    plugins  :放置插件的地方，比如第三方的分词器等
    
```



### 2、安装ElasticSearch-head插件

安装[ElasticSearch-head](https://github.com/mobz/elasticsearch-head)插件

解压head插件，进入ElasticSearch-head目录，在cmd下运行下面命令启动：

```bash
npm install  // 安装插件
npm run start  // 启动插件
```

启动成功后访问 `http://localhost:9100`

**解决跨域问题**

> 修改 Elasticsearch 配置文件 config/elasticsearch.yml，在文件末尾加上：
>
> http.cors.enabled: true
> 	     http.cors.allow-origin: "*"



### 3、安装Kibana

下载[Kibana](https://www.elastic.co/cn/downloads/kibana)

Kibana 是为 Elasticsearch设计的开源分析和可视化平台。你可以使用 Kibana 来搜索，查看存储在 Elasticsearch 索引中的数据并与之交互。你可以很容易实现高级的数据分析和可视化，以图表的形式展现出来。

① 解压Kibana

② 进入kibana/bin下，运行`kibana.bat`

**国际化**进入config/kibana.yml,在文末加上`i18n.locale: "zh-CN"`,修改kibana界面为中文。



### 4、安装 **ik** 中文分词器

下载[elasticsearch-analysis-ik](https://github.com/medcl/elasticsearch-analysis-ik/releases)

直接解压到`Elasticsearch/elasticsearch-7.12.1/plugins/`下就行（先在plugins下建个ik文件夹）。

可以在Kibana控制台里测试：

- **ik_smart：**最少切分
- **ik_max_word：**最细粒度切分

```json
GET _analyze
{
  "analyzer": "ik_smart",
  "text": ["我是好学生"]
}

GET _analyze
{
  "analyzer": "ik_max_word",
  "text": ["我是好学生"]
}
```

可以去plugins中写自己的字典my.dic  ，多词典用分号分隔。



## 三、基本操作

#### Rest风格说明

一种软件架构风格,而不是标准,只是提供了一组设计原则和约束条件。它主要用于客户端和服务器交互类的软件。基于这个风格设计的软件可以更简洁。更有层次,更易于实现缓存等机制。
**基本Rest命令说明:（从es7开始弃用类型types，所以在url中可以不用再写类型名称，或者写成_doc）**

| method | url地址                                           |          描述          |
| :----: | :------------------------------------------------ | :--------------------: |
|  PUT   | localhost:9200/索引\|名称/类型名称/文档id         | 创建文档（指定文档id） |
|  POST  | localhost:9200/索引\|名称/类型名称                | 创建文档（随机文档id） |
|  POST  | localhost:9200/索引\|名称/类型名称/文档id/_update |        修改文档        |
| DELETE | localhost:9200/索引\|名称/类型名称/文档id         |        删除文档        |
|  GET   | localhost:9200/索引\|名称/类型名称/文档id         |  查询文档，通过文档id  |
|  POST  | localhost:9200/索引\|名称/类型名称_search         |      查询所有数据      |

#### 基本概念

#####  Node 与 Cluster

Elastic 本质上是一个分布式数据库，允许多台服务器协同工作，每台服务器可以运行多个 Elastic 实例。

单个 Elastic 实例称为一个节点（node）。一组节点构成一个集群（cluster）。

##### Index

Elastic 会索引所有字段，经过处理后写入一个反向索引（Inverted Index）。查找数据的时候，直接查找该索引。

所以，Elastic 数据管理的顶层单位就叫做 Index（索引）。它是单个数据库的同义词。每个 Index （即数据库）的名字必须是小写。

下面的命令可以查看当前节点的所有 Index。

```bash
GET _cat/indices?v
```

##### Document

Index 里面单条的记录称为 Document（文档）。许多条 Document 构成了一个 Index。

Document 使用 JSON 格式表示，下面是一个例子。

 ```json
 {
   "user": "张三",
   "title": "工程师",
   "desc": "数据库管理"
 }
 ```

同一个 Index 里面的 Document，不要求有相同的结构（scheme），但是最好保持相同，这样有利于提高搜索效率。

##### Type

Document 可以分组，比如`weather`这个 Index 里面，可以按城市分组（北京和上海），也可以按气候分组（晴天和雨天）。这种分组就叫做 Type，它是虚拟的逻辑分组，用来过滤 Document。

不同的 Type 应该有相似的结构（schema），举例来说，`id`字段不能在这个组是字符串，在另一个组是数值。这是与关系型数据库的表的一个区别。性质完全不同的数据（比如`products`和`logs`）应该存成两个 Index，而不是一个 Index 里面的两个 Type（虽然可以做到）。

下面的命令可以列出每个 Index 所包含的 Type。

 ```bash
 GET _mapping?pretty=true
 ```

根据规划，Elastic 6.x 版只允许每个 Index 包含一个 Type，7.x 版将会彻底移除 Type。



#### 关于索引的操作

> 创建索引

```json
PUT /test1/_doc/1
{
  "name": "老王",
  "age": 18
}
```

_doc为默认类型，自动推断类型。

>  数据类型：

- 字符串：text、keyword
- 数值类型：long、integer、short、byte、double、float、half float、scaled float
- 日期类型：date
- 布尔类型：boolean
- 二进制：binary
- 等等

> 创建规则

```json
PUT /test2
{
  "mappings": {
    "properties": {
      "name": {
        "type": "text"
      },
      "age": {
        "type": "long"
      }
    }
  }
}
```

> 获取索引情况,通过  _cat 命令

```json
GET _cat/plugins
```

> 修改索引

**1、方法1：**

```json
PUT /test1/_doc/1
{
  "name": "老王",
  "age": 19
}
```

直接更改值，但是漏掉某个值，该值会为空。



**2、方法2：(不加_update，其他属性会置空 )**

```json
POST /test1/_doc/1/_update
{
  "doc":{
    "age": 20
  }
}
```



#### 关于文档的操作

#####  返回所有记录

使用 GET 方法，直接请求`/Index/Type/_search`，就会返回所有记录。

```bash
GET accounts/person/_search
{
  "took":2,
  "timed_out":false,
  "_shards":{"total":5,"successful":5,"failed":0},
  "hits":{
    "total":2,
    "max_score":1.0,
    "hits":[
      {
        "_index":"accounts",
        "_type":"person",
        "_id":"AV3qGfrC6jMbsbXb6k1p",
        "_score":1.0,
        "_source": {
          "user": "李四",
          "title": "工程师",
          "desc": "系统管理"
        }
      },
      {
        "_index":"accounts",
        "_type":"person",
        "_id":"1",
        "_score":1.0,
        "_source": {
          "user" : "张三",
          "title" : "工程师",
          "desc" : "数据库管理，软件开发"
        }
      }
    ]
  }
}
```

上面代码中，返回结果的 `took`字段表示该操作的耗时（单位为毫秒），`timed_out`字段表示是否超时，`hits`字段表示命中的记录，里面子字段的含义如下。

- `total`：返回记录数，本例是2条。
- `max_score`：最高的匹配程度，本例是`1.0`。
- `hits`：返回的记录组成的数组。

返回的记录中，每条记录都有一个`_score`字段，表示匹配的程序，默认是按照这个字段降序排列。

#####  全文搜索

**更多[查询语法](https://www.elastic.co/guide/en/elasticsearch/reference/current/query-dsl.html)，见官网**

Elastic 的查询非常特别，使用自己的[查询语法](https://www.elastic.co/guide/en/elasticsearch/reference/current/query-dsl.html)，要求 GET 请求带有数据体。

```bash
GET accounts/person/_search
{
  "query" : { "match" : { "desc" : "软件" }}
}'
```

上面代码使用 [Match 查询](https://www.elastic.co/guide/en/elasticsearch/reference/current/query-dsl-match-query.html)，指定的匹配条件是`desc`字段里面包含"软件"这个词。返回结果如下。

```javascript
{
  "took":3,
  "timed_out":false,
  "_shards":{"total":5,"successful":5,"failed":0},
  "hits":{
    "total":1,
    "max_score":0.28582606,
    "hits":[
      {
        "_index":"accounts",
        "_type":"person",
        "_id":"1",
        "_score":0.28582606,
        "_source": {
          "user" : "张三",
          "title" : "工程师",
          "desc" : "数据库管理，软件开发"
        }
      }
    ]
  }
}
```

Elastic 默认一次返回10条结果，可以通过`size`字段改变这个设置。

```bash
GET accounts/person/_search
{
  "query" : { "match" : { "desc" : "管理" }},
  "size": 1
}'
```

上面代码指定，每次只返回一条结果。

还可以通过`from`字段，指定位移。

```bash
GET accounts/person/_search
{
  "query" : { "match" : { "desc" : "管理" }},
  "from": 1,
  "size": 1
}'
```

上面代码指定，从位置1开始（默认是从位置0开始），只返回一条结果。

#####  逻辑运算

如果有多个搜索关键字， Elastic 认为它们是`or`关系。

```bash
GET accounts/person/_search
{
  "query" : { "match" : { "desc" : "软件 系统" }}
}'
```

上面代码搜索的是`软件 or 系统`。

如果要执行多个关键词的`and`搜索，必须使用布尔查询。

```bash
GET accounts/person/_search
{
  "query": {
    "bool": {
      "must": [
        { "match": { "desc": "软件" } },
        { "match": { "desc": "系统" } }
      ]
    }
  }
}'
```



**更多[查询语法](https://www.elastic.co/guide/en/elasticsearch/reference/current/query-dsl.html)，见官网**

### [Documentation](https://www.elastic.co/guide/index.html)



























