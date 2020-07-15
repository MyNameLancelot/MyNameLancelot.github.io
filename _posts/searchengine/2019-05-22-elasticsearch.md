---
layout: post
title: "elasticsearch技术"
date: 2019-05-22 14:43:02
categories: searchengine
keywords: "elasticsearch,搜索"
description: "elasticsearch的技术"
---

## 一、ElasticSearch概念

​	ElasticSearch是一个基于Lucene的搜索服务器。它提供了一个分布式多用户能力的全文搜索引擎，基于RESTful web接口。Elasticsearch是用Java开发的，并作为Apache许可条款下的开放源码发布，是当前流行的企业级搜索引擎。设计用于云计算中，能够达到实时搜索，稳定，可靠，快速，安装使用方便。

**ElasticSearch的特点**

（1）可以作为一个大型分布式集群技术，处理PB级数据，服务大公司。也可以运行在单机上，服务小公司

（2）对用户而言，是开箱即用的，非常简单

（3）弥补数据库在全文检索，同义词处理，相关度排名，复杂数据分析，海量数据的近实时处理的不足

### ElasticSearch的相关概念

- **cluster**

  代表一个集群，集群中有多个节点，其中有一个为主节点，这个主节点是通过选举产生的，主从节点是对于集群内部来说的。ElasticSearch的一个概念就是去中心化，字面上理解就是无中心节点，这是对于集群外部来说的，因为从外部来看ElasticSearch集群，在逻辑上是个整体，与任何一个节点的通信和与整个ElasticSearch集群通信是等价的。

- **shards**

  代表索引分片，ElasticSearch可以把一个完整的索引分成多个分片，这样的好处是可以把一个大的索引拆分成多个，分布到不同的节点上。构成分布式搜索。分片的数量只能在索引创建前指定，并且索引创建后不能更改（primary shard）。

  水平扩容时，需要重新设置分片数量，重新导入数据

- **replicas**

  代表索引副本（replica shard），ElasticSearch可以设置多个索引副本，副本的作用一是提高系统的容错性，当某个节点某个分片损坏或丢失时可以从副本中恢复。二是提高ElasticSearch的查询效率，ElasticSearch会自动对搜索请求进行负载均衡。

- **recovery**

  代表数据恢复或叫数据重新分布，ElasticSearch在有节点加入或退出时会根据机器的负载对索引分片进行重新分配，挂掉的节点重新启动时也会进行数据恢复。

- **river**

  代表ElasticSearch的一个数据源，也是其它存储方式（如：数据库）同步数据ElasticSearch的一个方法。它是以插件方式存在的一个ElasticSearch服务，通过读取river中的数据并把它索引到ElasticSearch中，官方的river有couchDB的，RabbitMQ的，Twitter的，Wikipedia的

- **gateway**

  代表ElasticSearch索引快照的存储方式，ElasticSearch默认是先把索引存放到内存中，当内存满了时再持久化到本地硬盘。gateway对索引快照进行存储，当这个ElasticSearch集群关闭再重新启动时就会从gateway中读取索引备份数据。ElasticSearch支持多种类型的gateway，有本地文件系统（默认），分布式文件系统，Hadoop的HDFS和amazon的s3云存储服务。

- **discovery.zen**

  代表ElasticSearch的自动发现节点机制，ElasticSearch是一个基于p2p的系统，它先通过广播寻找存在的节点，再通过多播协议来进行节点之间的通信，同时也支持点对点的交互。

- **Transport**

  代表ElasticSearch内部节点或集群与客户端的交互方式，默认内部是使用tcp协议进行交互，同时它支持http协议（json格式）、thrift、servlet、memcached、zeroMQ等的传输协议（通过插件方式集成）。

### ElasticSearch的核心概念

- **Near Realtime (NRT)**

  从写入数据到可搜索数据有一个延迟（1秒左右，分析写入数据的过程）

- **Cluster&Node**

  Cluster-集群，包含多个节点，每个节点通过配置来决定属于哪一个集群（默认集群命为“elasticsearch”）

  Node-节点，集群中的一个节点，节点的名字默认是随机分配的。节点名字在运维管理时很重要，节点默认会自动加入一个命名为“elasticsearch”的集群，如果直接启动多个节点，则自动组成一个名为“elasticsearch”的集群。单节点启动也是一个集群。

- **Document**

  ElasticSearch中的最小数据单元。一个Document就是一条数据，一般使用JSON数据结构表示。每个Index下的Type中都可以存储多个Document。一个Document中有多个field，field就是数据字段。

  不要在Document中描述java对象的双向关联关系。在转换为JSON字符串的时候会出现无限递归问题。

- **Index**

  索引，**物理分类**。包含若干相似结构的Document数据。如：客户索引，订单索引，商品索引等。一个Index包含多个Document，也代表一类相似的或相同的Document。

- **Type**

  类型。每个索引中都可以有若干Type，Type是Index中的一个**逻辑分类**，同一个Type中的Document都有相同的field。

  6.x版本之后，type概念被弱化，一个index中只能有唯一的一个type。在7.x版本之后，会删除type。

> 一个index默认10个shard，5个primary shard，5个replica shard
>
> 一个index分配到多个shard上，primary shard和他对应的replica shard不能在同一个节点中存储

## 二、Linux环境下安装

```txt
ElasticSearch
 |
 |--bin【可执行文件包】
 |
 |--config【配置相关目录】
 |
 |--lib【ES需要依赖的jar包，ES自开发的jar包】
 |
 |--modules【组件包】
 |
 |--plugins【插件目录包，三方插件或自主开发插件】
 |
 |--logs【日志文件相关目录】
 |
 |--data【在ES启动后，会自动创建的目录，内部保存ES运行过程中需要保存的数据】
```

### 安装ElasticSearch

第一步、ElasticSearch不允许root用户启动，所以需要先创建用户

```shell
useradd -U es					#添加用户和用户组为es
chown -R es:es elasticsearch	#修改elasticsearch所属主所属组
```

第二步、修改文件打开限制。至少需要65536的文件创建权限，修改限制信息`/etc/security/limits.conf`

```conf
es    hard    nofile    65536   #number open file
es    soft    nofile    65536
```

第三步、修改线程开启限制。Linux默认用户进程有开启1024个线程权限，至少需要4096的线程池预备，修改`/etc/security/limits.d/20-nproc.conf`

```conf
*          soft    nproc     4096
root       soft    nproc     unlimited
```

第四步、修改系统虚拟内存权限。Linux默认不允许任何用户和应用直接开辟虚拟内存，修改`/etc/sysctl.conf`

```conf
vm.max_map_count=655360
```

第五步、修改客户端访问控制【可选】。修改`config/elasticsearch.yml`

```yaml
network.host: 0.0.0.0
```

第六步、切换用户并启动ElasticSearch

```shell
su es
./bin/elasticsearch
```

### 安装Kibana

第一步、修改客户端访问控制。修改`config/kibana.yml`

```yaml
#默认localhost，非本机不可访问
server.host:  "0.0.0.0"
#elasticsearch.hosts: ["http://localhost:9200"]  有必要时修改
```

第二步、切换用户并启动elasticsearch

```conf
su es
./bin/kibana
```

## 三、Kibana简单命令

### 查看健康状态

命令：`GET _cat/health?v`

返回：![health-info](/img/searchengine/health-info.png)

| 参数    | 含义                                                         |
| ------- | ------------------------------------------------------------ |
| shards  | 分片总个数                                                   |
| pri     | primary shard个数                                            |
| relo    | replica shard个数                                            |
| unssign | 未分配个数                                                   |
| status  | green【每个索引的primary shard和replica shard都是active的】<br />yellow【每个索引的primary shard都是active的，但部分的replica shard不是active的】<br />red【不是所有的索引都是primary shard都是active状态的】 |


### 检查分片信息

命令：`GET _cat/shards?v`

返回：

![shared-info](/img/searchengine/shared-info.png)

显示了每个索引的分片信息

### 查看索引信息

命令：`GET _cat/indices?v`

返回：

![indices-info](/img/searchengine/indices-info.png)

### 索引相关

- **新增索引**

  默认创建索引的时候，会分配5个primary shard，并为每个primary shard分配一个replica shard。

  默认限制：如果磁盘空间不足15%的时候，不分配replica shard。如果磁盘空间不足5%的时候，不再分配任何的primary shard。

  ```txt
  PUT /test_index
  {
    "settings":{
      "number_of_shards" : 2,    //指定primary shard个数
      "number_of_replicas" : 1   //指定每个primary shard的replica shard个数
    }
  }
  ```

- **修改索引**

  索引一旦创建，primary shard数量不可变化，可以改变replica shard数量

  ```txt
  PUT /test_index/_settings
  {
     "number_of_replicas" : 1    //指定每个primary shard的replica shard个数
  }
  ```

- **删除索引**

  命令`DELETE /test_index [, other_index]`

> ElasticSearch尽可能保证primary shard平均分布在多个节点上。Replica shard会保证不和它自身备份的那个primary shard分配在同一个节点上

**Primary Shared不可变的优缺点**

优点：不需要锁，提升并发能力，避免锁问题。数据不变，可以缓存在OS cache中（前提是cache足够大）。filter cache始终在内存中，因为数据是不可变的。可以通过压缩技术来节省CPU和IO的开销。

缺点：只要index的结构发生任何变化，都必须重建索引。

### 文档CRUD

- **创建文档**

  - **PUT语法**

    此操作为手工指定id的Document新增方式【7以后必须使用POST新增】

    ```txt
    PUT /test_index/my_type/1
    {
       "name":"test_doc_01",
       "remark":"first test elastic search",
       "order_no":1
    }
    ```

    返回内容：

    ```txt
    {
      "_index" : "test_index",   #新增的document在什么index中
      "_type" : "my_type",       #新增的document在什么type中
      "_id" : "1",               #指定的id是多少
      "_version" : 1,            #document的版本是多少，版本从1开始递增，每次写操作都会+1
      "result" : "created",      #created创建，updated修改，deleted删除
      "_shards" : {					
        "total" : 2,             #分片数量只提示primary shard
        "successful" : 1,        #成功几个
        "failed" : 0             #失败几个
      },						
      "_seq_no" : 0,             #执行的序列号
      "_primary_term" : 1        #词条比对
    }
    ```

  - **POST语法**

    此操作为ElasticSearch自动生成id的新增Document方式

    ```txt
    POST /test_index/my_type
    {
       "name":"test_doc_04",
       "remark":"forth test elastic search",
       "order_no":4
    }
    ```

  一个index中的所有type类型的Document是存储在一起的，如果index中的不同的type之间的field差别太大，也会影响到磁盘的存储结构和存储空间的占用

  ```txt
  test_index:test_type1-->{"_id":"1","f1":"v1","f2":"v2"}
  test_index:test_type2-->{"_id":"2","f3":"v3","f4":"v4"}
  
  实际上存储格式为
  {"id":"1","f1":"v1","f2":"v2","f3":"","f4":""},
  {"id":"2","f1":"","f2":"","f3":"v3","f4","v4"}
  
  建议，每个index中存储的document结构不要有太大的差别。尽量控制在总计字段数据的10%以内
  ```

- **查询Document**

  - **GET查询**

    语法：`GET /index_name/type_name/id`

    例如：GET /test_index/my_type/1

  - **GET /_mget查询**

    ```txt
    GET /_mget
    {
      "docs" : [
        {
          "_index" : "value",
          "_type" : "value",
          "_id" : "value"
        }, {}, {}
      ]
    }
    =====================================示例======================================
    GET /_mget
    {
      "docs":[
        {
          "_index":"test_index",
          "_type":"my_type",
          "_id":1
        },
        {
          "_index":"test_index",
          "_type":"my_type",
          "_id":2
        },
        {
          "_index":"test_index",
          "_type":"my_type",
          "_id":3
        }
      ]
    }
        
    ------------------------------------------------------------------------------
    GET /test_index/my_type/_mget
    {
      "docs":[
        {"_id":1},
        {"_id":2},
        {"_id":3}
      ]
    }
        
    ------------------------------------------------------------------------------
    GET /test_index/my_type/_mget
    {
      "ids":[1,2,3,4]
    }
    ```
  

- **修改Document**

  - **替换Document（全量替换）**

    语法：和新增PUT语法一致【`PUT /index_name/type_name/id{field_name:new_field_value}`】

    说明：如果有原文档此操作相当于删除原文档，并新增文档，和以前原文档没有任何关系

    ```txt
    PUT /test_index/my_type/1
    {
       "name":"new_test_doc_01",
       "remark":"first test elastic search",
       "order_no":1
    }
    ```

  - **PUT语法强制新增**

    语法：`PUT /index_name/type_name/id/_create `或`PUT /index_name/type_name/id?op_type=create`

    说明：使用强制新增语法时，如果Document的id已存在，则会报错。（version conflict, document already exists）

    ```txt
    PUT /test_index/my_type/1/_create
    {
       "name":"new_test_doc_01",
       "remark":"first test elastic search",
       "order_no":1
    }
    ```

  - **更新Document（partial update）**

    语法：`POST /index_name/type_name/id/_update{field_name:field_value_for_update}`

    说明：只更新某Document中的部分字段，文档必须存在。对比全量替换而言，只是操作上的方便，在底层执行上几乎没有区别

    ```txt
    POST /test_index/my_type/1/_update
    {
       "doc":{
          "name":" test_doc_01_for_update"
       }
    }
    ```

  - **删除Document**

    语法：`DELETE /index_name/type_name/id`

    ```txt
    DELETE /test_index/my_type/1
    ```

- **bulk批量增删改**

  语法：

  ```txt
  POST /_bulk
  { "action_type" : { "metadata_name" : "metadata_value" } }
  { document datas | action datas }
  ```

  action_type可选值为:

  1）create : 强制创建，相当于PUT /index_name/type_name/id/_create

  2）index: 普通的PUT操作，相当于创建Document或全量替换

  3）update: 更新操作（partial update）,相当于 POST /index_name/type_name/id/_update

  4）delete: 删除操作

  ```txt
  POST /_bulk
  { "create" : { "_index" : "test_index" , "_type" : "my_type", "_id" : "10" } }
  { "field_name" : "field value" }
  { "index" : { "_index" : "test_index", "_type" : "my_type" , "_id" : "20" } }
  { "field_name" : "field value 2" }
  { "update" : { "_index" : "test_index", "_type" : "my_type" , "_id" : 20 } }
  { "doc" : { "field_name" : "partial update field value" } }
  { "delete" : { "_index" : "test_index", "_type" : "my_type", "_id" : "2" } }
  ```

  注意：bulk语法中要求一个完整的json串不能有换行

- **Document routing 机制**

  ​	Document的路由算法决定了Document存放在哪一个primary shard中。算法为：primary shard = hash(routing) % number_of_primary_shards，其中的routing默认为Document中的元数据_id，也可以手工指定routing的值，指定方式为：`PUT /index_name/type_name/id?routing=xxx`

  ```txt
  PUT /test_index/my_type/5?routing=1001
  {
    "detpno":1001,
    "name":"Tom"
  }
  ```

  ​	手工指定routing在海量数据中非常有用，通过手工指定的routing值，会将相关的Document存储在同shard中，方便后期进行应用级别的负载均衡并可以提高数据检索的效率。如：存电商中的商品，使用商品类型的编号作为routing，ElasticSearch会把同一个类型的商品document数据，存在同一个shard中。查询的时候，同一个类型的商品，在一个shard上查询，效率最高。

- **Document增删改原理简图**

![；document-cud](/img/searchengine/document-cud.png)

​	执行步骤：

​	1）客户端发起请求，执行增删改操作。所有的增删改操作都由primary shard直接处理，replica shard只被动的备份数据。此操作请求到节点2（请求发送到的节点随机），这个节点称为协调节点（coordinate node）

​	2）协调节点通过路由算法，计算出本次操作的Document所在的shard。假设本次操作的Document所在shard为 primary shard 0。协调节点计算后，会将操作请求转发到节点1

​	3）节点1中的primary shard 0在处理请求后，会将数据的变化同步到对应的replica shard 0中，也就是发送一个同步数据的请求到节点3中

​	4）replica shard 0在同步数据后，会响应通知请求这同步成功，也就是响应给primary shard 0（节点1）

​	5）primary shard 0（节点1）接收到replica shard 0的同步成功响应后，会响应请求者，本次操作完成。也就是响应给协调节点（节点2）

​	6）协调节点返回响应给客户端，通知操作结果

- **Document查询简图**

![document-find](/img/searchengine/document-find.png)

​	执行步骤：

​	1）客户端发起请求，执行查询操作。查询操作都由所有shard共同处理。假设此操作请求到节点2（随机），这个节点称为协调节点（coordinate node）

​	2）协调节点通过路由算法，计算出本次查询的Document所在的shard。假设本次查询的Document所在shard为 shard 0。协调节点计算后，会将操作请求转发到节点1或节点3。分配请求到节点1还是节点3通过随机算法计算

​	3）节点1或节点3中在处理请求后，会将查询结果返回给协调节点（节点2）

​	4）协调节点得到查询结果后，再将查询结果返回给客户端

## 四、元数据

- **_index**

  代表document存放在哪个index中，_index就是索引的名字。生成环境中，类似的Document应该存放在一个index中。index名称必须是小写，且不能以【+，-，\_】开头。

- **_type**

  代表document属于index中的哪个type（类别）。6.x版本中，一个index只能定义一个type。Type命名要求：字符大小写无要求，不能下划线开头，不能包含逗号。

- **_id**

  代表document的唯一标识。使用index、type和id可以定位唯一的一个document。id可以在新增document时手工指定，也可以由ElasticSearch自动创建。

- **_version**

  代表的是document的版本。在ElasticSearch中，为document定义了版本信息，document数据每次变化，代表一次版本的变更。版本变更可以避免数据错误问题（并发问题，乐观锁），同时提供ES的搜索效率。第一次创建Document时，\_version版本号为1，默认情况下，后续每次对Document执行修改或删除操作都会对\_version数据自增1。

  删除Document也会使\_version自增。当使用PUT命令再次增加同id的Document，_version会继续之前的版本继续自增。

## 五、分布式架构分析

### 分布式机制的透明隐藏特性

​	ElasticSearch本身就是一个分布式系统，就是为了处理海量数据的应用。隐藏了复杂的分布式机制，简化了配置和操作的复杂度。ElasticSearch在现在的互联网环境中，盛行的原因，主要的核心就是分布式的简化。

​	ElasticSearch隐藏的分布式内容：分片机制、集群发现、负载均衡、路由请求、集群扩容、shard重分配等。

### 节点变化时的数据重平衡

​	在节点发生变化时，ES的cluster会自动调整每个节点中的数据，也就是shard的重新分配。这个过程叫做rebalance。rebalance目的是为了提高性能，理论上，当数据量达到一定级别的时候，每个shard分片上的数据量是均等的。那么每个节点管理的shard数量是均等的情况，ES集群才能提供最好的服务性能。

### master节点作用

​	维护集群元数据的节点。如：索引的变更（创建和删除）对应的元数据保存在master节点中。

​	master不是请求的唯一接收节点。只是维护元数据的特殊节点。不会导致单点瓶颈。

​	集群在默认配置中，会自动的选举出master节点。元数据是一个收集管理的过程。不是一个必要的不可替代的数据。Master如果宕机，集群会选举出新的master，新的master会在集群的所有节点中重新收集集群元数据并管理。

### 节点平等的分布式架构

- 节点平等

  每个节点都能接收客户端请求。不会造成单点压力。

- 自动请求路由

  节点在接收到客户端请求时，会自动的识别应该由哪个节点处理本次请求，并实现请求的转发。

- 响应收集

  在自动请求路由后，接收客户端请求的节点会自动的收集其他节点的处理结果，并统一响应给客户端。

​	![node-equality](/img/searchengine/node-equality.png)

1）客户端请求ElasticSearch集群，搜索“张三”数据。请求发送到节点4。此操作代表节点的平等性。集群中每个节点的功能都是一致的。

2）节点4将请求路由到节点1上。实现数据的搜索。这是自动请求路径的过程。

3）节点1处理结束后，搜索结果会发送给节点4。这是响应收集的过程。

4）节点4将最终结果响应给客户端，这种操作就屏蔽了集群对客户端的影响。

### 并发冲突

- **使用内置_version**【7.x版本以后废弃】

  全量替换（乐观锁）：ElasticSearch会检查请求中的version和存储中的version是否相等。如果不相等报错，如果相等则修改。

  ```txt
  PUT /test_index/my_type/1?version=4
  {
    "name":"doc_new_01"
  }
  ```

- **使用external version**

  ElasticSearch提供了一个特性，可以自定义version实现ES内置的\_version元数据版本管理功能。与元数据\_version的区别是：元数据比对必须完全一致，且external version必须比元数据_version数据大。成功后\_version修改为提供的external version数值。

  ```txt
  PUT /test_index/my_type/1?version=9&version_type=external
  {
    "name":"doc_new_new_01"
  }
  ```

- **使用partial update乐观锁**

  在ElasticSearch中，使用partial update更新数据时，内部自动使用乐观锁控制数据的并发安全。如果出现版本不一致的时候，ElasticSearch会提示更新失败。可以通过命令中的参数实现手工管理乐观锁。【retry_on_conflict和version参数不能同时使用】

  ```txt
  POST /test_index/my_type/1/_update?retry_on_conflict=3  #retry_on_conflict尝试次数
  {
    "name":"doc_new_post_01"
  }
  ```

- **使用`if_seq_no`和`if_primary_term`替代内置_version**

  ```txt
  PUT /test_index/my_type/1?if_seq_no=6&if_primary_term=3
  {
    "name":"doc_new_01"
  }
  ```

## 六、Query String搜索

语法：`GET [/index_name/type_name/]_search[?parameter_name=parameter_value&...]`

```txt
GET /current_index/_search?q=remark:test+AND+name:doc&sort=order_no:desc
```

此搜索操作一般只用在快速检索数据使用，如果查询条件复杂，很难构建query string，生产环境中很少使用

### 全数据查询

```txt
//查询所有索引下的数据，timeout为超时时长定义，不影响响应的正常返回，只影响返回条数
GET _search?timeout=10ms
```

### _all元数据搜索

`_all`是一个内置元数据，可是内置变量名。代表全部的含义。【很少使用】

```txt
GET /_all/_search                             //在所有索引中搜索数据

GET /index_name/type_name/_search?q=key_words	//在指定的索引中搜素包含key_words的信息
```

> 在ES维护Document的时候，会将Document中的所有字段数据连接作为一个_all元数据

### 多索引搜索

所谓的multi-index就是指从多个index中搜索数据。相对使用较少，只有在复合数据搜索的时候，可能出现。一般来说，如果真使用复合数据搜索，都会使用_all。

如：搜索引擎中的无条件搜索。（现在的应用中都被屏蔽了。使用的是默认搜索条件，或不执行搜索）

```txt
GET /products_es,products_index/_search   //在两个索引中搜索

GET /products_*/_search                   //使用通配符匹配

GET /_all/_search                         //_all代表所有索引
```

### 分页搜索与Deep Paging问题

```txt
GET /_search?from=0&size=10               // from从第几条开始查询，size返回的条数
```

​		执行搜索时请求发送到协调节点中，协调节点将搜索请求发送给所有的节点，而数据可能分部在多个节点中，所以会造成以下现象：集群有4个节点，每个节点有1个分片（primary shard），发起GET /_search?from=100&size=50时，每个节点会返回150条数据，协调节点一共接收到600条数据，进行排序并取其中第100~150条数据，如果页数过深会造成效率低下问题

### `-`,`+`符号条件**

```txt
GET /products_index/phone_type/_search?q=+name:plus		//搜索的关键字必须包含plus

GET /products_index/phone_type/_search?q=-name:plus		//搜索的关键字必须不能包含plus
```

`+`：和不定义符号含义一样，就是搜索指定的字段中包含key words的数据

`-` ：与`+`符号含义相反，就是搜索指定的字段中不包含key words的数据

## 七、Query DSL搜索

query DSL【Domain Specified Language】

### 查询所有数据

```txt
GET /current_index/_search?timeout=1ms	
{
  "query": {"match_all": {}}
}
```

### 全文检索

```txt
GET /es/doc/_search
{
  "query": {
    "match": {            //单字段模式
      "note": "only life" //会根据field对应的analyzer对搜索条件做分词
    }
  }
}

GET /es/doc/_search
{
  "query": {
    "multi_match": {                //多字段模式
      "query": "only life",         //会根据field对应的analyzer对搜索条件做分词
      "fields": ["note","content"]  //指定field
    }
  }
}
```

### 词元搜索

搜索条件`不`分词，使用搜索条件进行精确匹配。如果搜索字段被分词，搜索条件是一段话则无法匹配这种情况下适合匹配不分词字段。

```txt
GET /es/doc/_search
{
  "query": {
    "term": {
      "note": {						//虽然note被分词，但是只匹配一个词元会匹配上
        "value": "people"
      }
    }
  }
}
==================================================================================
GET /es/doc/_search
{
  "query" : {
    "terms" : {						//进行多个词元匹配
      "note" : [ "people",  "necessarily" ]
    }
  }
}
```

### 排序

当字段类型为text时，使用字符串类型的字段作为排序依据会有问题【ES对字段数据做分词，建立倒排索引。分词后，先使用哪一个单词做排序是不固定的】，此时需要在此字段上建立keyword类型的子字段，或者建立fielddata。

```txt
GET /kibana_sample_data_ecommerce/_search
{
  "sort": [
    {
      "total_quantity": {
        "order": "desc"
      }
    }
  ]
}
```

### 分页

```txt
GET /kibana_sample_data_ecommerce/_search
{
  "from": 100,
  "size": 20
}
```

### 范围搜索

```txt
GET /_search
{
  "query" : {
    "range" : {
      "emps.age" : {					//数字类型的范围搜索
        "gt" : 21, "lte" : 45
      }
    }
  }
}
==================================================================================
GET /_search
{
  "query": {
    "range": {
      "emps.join_date": {				//日期类型的范围搜索
        "gte": "2018-08-10||-40d"
      }
    }
  }
}
```

### 只返回部分字段

```txt
GET /kibana_sample_data_ecommerce/_search
{
  "_source": ["customer_id","customer_first_name","customer_full_name"]
}
```

### 组合搜索

bool - 用于组合多条件，相当于java中的布尔表达式。

must - 必须符合要求，相当于java中的逻辑运算符 ==或&&。

must_not - 必须不符合要求，相当于java中的逻辑运算符 ！

should - 有任意条件符合要求即可，相当于java中的逻辑运算符 ||

```txt
GET /emp_index/emp_type/_search
{
  "query": {
    "bool": {									//布尔条件开始
      "must": [									//必须满足
        {
          "match": {
            "dept_name": "Sales Department"
          }
        }
      ],
      "must_not": [								//不需不满足
        {
          "range": {
            "num_of_emps": {
              "gt": 10
            }
          }
        }
      ],
      "should": [								//满足一个即可
        {
          "match": {
            "emps.name": "wangwu"
          }
        },
        {
          "range": {
            "num_of_emps": {
              "lt": 20
            }
          }
        }
      ]
    }
  }
}
```

### scroll搜索

​	如果需要一次性搜索出大量数据，那么执行效率一定不会很高，这个时候可以使用scroll滚动搜索的方式实现搜索。scroll滚动搜索类似分页，是在搜索的时候，先查询一部分，之后再查询下一部分，分批查询总体数据。可以实现一个高效的响应。scroll搜索会在第一次搜索的时候保存一个快照，这个快照保存的时长由请求指定，后续的查询会依据这个快照执行再次查询。如果这个过程中，ES中的document发生了变化，是不会影响到原搜索结果的。

- scroll搜索时，需要指定一个排序，可以使用_doc实现排序，这种排序性能更高一些。

- scroll搜索的2次以后请求，必须在指定的快照时长内发起，且每次请求都需要指定这个快照保存时长。

- scroll搜索的返回结果一定会包含一个特殊的结果数据_scroll_id。这个数据就是scroll搜索的快照ID。scroll搜索的2次以后的请求，必须携带上这个ID。

scroll不是分页，不是用来替换分页技术。分页为用户提供数据的，scroll为系统内部处理分批提供数据的

```txt
=====================================第一次搜索=====================================
GET /es/doc/_search?scroll=1m
{
  "size": 1,
  "query": {
    "match_all": {}
  }
}
=====================================第二次搜索=====================================
GET /_search/scroll
{
  "scroll": "1m",
  "scroll_id" : "【上次搜索返回的scroll_id】"
}
```

### 搜索精度控制

搜索精度的精确控制，条件是否是同时满足还是只需满足一个即可

```txt
=======================只需满足remark含有java或者developer即可=======================
GET /student/java/_search
{
  "query": {
    "match": {
      "remark": "java developer"
    }
  }
}
==========================必须满足remark含有java和developer=========================
GET /student/java/_search
{
  "query": {
    "match": {
      "remark": {
        "query": "java developer",	//operator为or时，与第一个案例搜索语法效果一致
        "operator": "and"
      }
    }
  }
}
=============必须满足remark含有一定数量的java、developer、architect===================
GET /student/java/_search
{
  "query": {
    "match": {
      "remark": {
        "query":  "java developer architect",
        "minimum_should_match": 2			//相当于67%=>66.66%需要向前进位
      }
    }
  }
}
```

### boost权重控制

设置指定字段的权重值，控制相关度分数

```txt
GET /student/java/_search
{
  "query": {
    "bool": {
      "should": [
        {
          "term": {
            "remark": {
              "value": "developer",
              "boost":2				//默认值为1
            }
          }
        },
        {
          "term": {
            "remark": {
              "value": "architect"
            }
        }
        }
      ]
    }
  }
}
```

**多shard环境中相关度分数不准确问题**

​	在ElasticSearch的搜索结果中，相关度分数不是一定准确的。相同的数据，使用相同的搜索条件搜索，得到的相关度分数可能有误差。只要数据量达到一定程度，相关度分数误差就会逐渐趋近于0。

![relevance](/img/searchengine/relevance.png)

​	在shard0中，有100个document中包含java词组。在shard1中，有10个document中包含java词组，在执行搜索的时候，ES计算相关度分数时，就会出现计算不准确的问题。因为ES计算相关度分数是在shard本地计算的。根据TF/IDF算法，在shard0中的document相关度分数会低于shard1中的相关度分数。

### 搜索策略

**基于dis_max实现best fields策略进行多字段搜索**

best fields策略：如果某一个field中匹配到了尽可能多的关键词，那么就应被排在前面，即执行若干查询条件以若干分数中最高分为实际的得分进行排序

```txt
GET /student/java/_search
{
  "query": {
    "dis_max": {
      "queries": [
        {
          "match" : {
            "name": "james gosling"
          }
        },
        {
          "match": {
            "remark": "java developer"
          }  
        }
      ]
    }
  }
}
```

**使用tie_breaker参数优化dis_max搜索效果**

​	dis_max是将多个搜索query条件中相关度分数最高的用于结果排序，忽略其他query分数。这时候可以使用tie_breaker参数来优化dis_max搜索。tie_breaker参数代表的含义是：将其他query搜索条件的相关度分数乘以参数值，再参与到结果排序中。如果不定义此参数，相当于参数值为0。所以其他query条件的相关度分数被忽略。

```txt
GET /student/java/_search
{
  "query": {
    "dis_max": {
      "tie_breaker": 0.7, 
      "queries": [
        {
          "match" : {
            "name": "james gosling"
          }
        },
        {
          "match": {
            "remark": "java developer"
          }  
        }
      ]
    }
  }
}
```

**使用multi_match简化dis_max+tie_breaker**

multi_match匹配多个字段，使用best_fields策略

```txt
GET /student/java/_search
{
  "query": {
    "multi_match": {
      "query": "james gosling java developer",
      "fields": ["name", "remark^2"], 			//remark的boost为2
      "type": "best_fields", 
      "tie_breaker": 0.7, 
      "minimum_should_match": 2,
      "analyzer": "standard" 
    }
  }
}
```

**multi_match+most fields实现multi_field搜索**

most fields策略：指搜索结果匹配了更多的字段的document分数高

```txt
GET /student/java/_search
{
  "query": {
    "multi_match": {
      "query": "james gosling java developer",
      "fields": ["name", "remark"],
      "type": "most_fields",
      "analyzer": "standard" 
    }
  }
}
```

**cross fields搜索**

cross-fields搜索，一个唯一标识，跨了多个field。比如是地址可以散落在province，city，street中。

```txt
GET student/java/_search
{
  "query": {
    "multi_match": {
      "query": "joshua developer",
      "fields": ["name", "remark"],
      "type": "cross_fields",
      "analyzer": "standard",   //注意要使用分词器
      "operator" : "and"        //（name必须包含joshua且remark必须包含developer）或者
    }                           //（remark必须包含joshua且name必须包含developer）
  }
}
```

**most fields策略是尽可能匹配更多的字段，所以会导致精确搜索结果排序问题。又因为cross fields搜索，不能使用minimum_should_match来去除长尾数据。所以在使用most fields和cross fields策略搜索数据的时候，都有不同的缺陷。所以商业项目开发中，都推荐使用best fields策略实现搜索。**

### Suggest搜索建议

实现suggest的时，其构建的不是倒排索引，也不是正排索引，是纯的用于进行前缀搜索的一种特殊的数据结构，而且会全部放在内存中，所以suggest search进行的前缀搜索提示，性能非常高。

需要使用suggest的时候，必须在定义index时，为其mapping指定开启suggest

```txt
========================================建立索引=======================================
PUT /suggest
{
  "mappings": {
    "doc": {
      "properties": {
        "title": {
          "type": "text",
          "analyzer": "ik_max_word",
          "fields": {
            "suggest": {
              "type": "completion",
              "analyzer": "ik_max_word"
            }
          }
        },
        "content": {
          "type": "text",
          "analyzer": "ik_max_word"
        }
      }
    }
  }
}
====================================应用suggest搜索====================================
GET /suggest/doc/_search
{
  "suggest": {
    "suggest_by_title": {
      "prefix": "大话",
      "completion": {
        "field": "title.suggest"
      }
    }
  }
}
```

### 近似匹配

**短语搜索**

```txt
GET /es/doc/_search
{
  "query": {
    "match_phrase": {
      "note": {
        "query": "importance people", //必须满足此短语
        "slop": 4                     //可以移动4次实现短语匹配，移动次数越少分数越高
      }
    }
  }
}
```

​	match phrase原理：在进行短语搜索的时候其实进行了分词操作，将短语进行分词之后在倒排索引中检索数据，检查匹配到的索引在同一个文档中是否连续（利用position判断）。【使用`GET _analyze`可以查看分词信息】

**召回率和精准度平衡**

召回率：搜索结果比例

精准度：搜索结果的准确率

如果在搜索的时候，只使用match phrase语法，会导致召回率底下，因为搜索结果中必须包含短语（包括proximity search）。如果在搜索的时候，只使用match语法，会导致精准度底下，因为搜索结果排序是根据相关度分数算法计算得到。可将两者同时使用，平衡召回率和精准度。

```txt
GET /es/doc/_search
{
  "query": {
    "bool": {
      "must": [
        {
          "match": {
            "note": "importance people"
          }
        },
        {
          "match_phrase": {
            "note": {
              "query": "importance people",
              "slop": 50
            }
          }
        }
      ]
    }
  }
}
```

**搜索的优化**

match一般来说，比match phrase效率高10倍左右，比proximity search高20倍左右

优化的思路是：尽量减少proximity search搜索的document数量。即先用match来搜索出需要的数据，再用proximity search来提高term距离较近的document的相关度分数，从而影响结果排序，这个过程也叫rescore（重计分）。再rescore的时候，尽量使用分页，毕竟用户通常只会查看搜索数据的前几页。

```txt
GET /es/doc/_search
{
  "query": {
    "match": {
      "note": "importance people"
    }
  },
  "rescore": {					//开始重计分
    "query": {
      "rescore_query": {		//重新计分查询
        "match_phrase": {
            "note": {
              "query": "importance people",
              "slop": 50
            }
          }
      }
    },
    "window_size": 50			//对match搜索结果的前多少条数据执行rescore操作
  },
  "from": 0,
  "size": 2 
}
```

**前缀匹配搜索**

针对前缀搜索，是对keyword类型字段而言。而keyword类型字段数据大小写敏感。前缀搜索效率比较低。前缀搜索不会计算相关度分数。前缀越短，效率越低。不推荐使用。

```txt
GET /es/doc/_search
{
  "query": {
    "prefix": {
      "note.keyword": {
        "value": "May"
      }
    }
  }
}
```

**通配符搜索**

通配符可以在倒排索引中使用，也可以在keyword类型字段中使用。性能也很低，需要扫描完整的索引。不推荐使用。

```txt
GET /es/doc/_search
{
  "query": {
    "wildcard": {
      "note": {
        "value": "*oug?"
      }
    }
  }
}
```

**正则搜索**

ES支持正则表达式。可以在倒排索引或keyword类型字段中使用。性能也很低，需要扫描完整索引。不推荐使用。

```txt
GET /es/doc/_search
{
  "query": {
    "regexp": {
      "note": {
        "value": "[A-z]noug.?"
      }
    }
  }
}
```

**搜索推荐**

其原理和match phrase类似，是先使用match匹配term数据，然后在指定的slop移动次数范围内，前缀匹配。max_expansions是用于指定prefix最多匹配多少个term，超过这个数量就不再匹配了。

```txt
GET /es/doc/_search
{
  "query": {
    "match_phrase_prefix": {
      "note": {
        "query": "happiest of peo",
        "slop": 3,
        "max_expansions": 3			//匹配出3个以后就不再进行匹配
      }
    }
  }
}
```

这种语法的限制是只有最后一个term会执行前缀搜索，因为效率较低，如果使用一定要使用max_expansions限定

**ngram分词机制**

ngram是在指定长度下，对一个单词再次实现拆分的机制，如单词hello。

```txt
ngram length = 1  --  h  e  l  l  o
ngram length = 2  --  he  el  ll  lo
ngram length = 3  --  hel  ell  llo
ngram length = 4  --  hell  ello
ngram length = 5  --  hello
```

使用ngram搜索

```txt
=================================创建edge ngram索引====================================
PUT /engram
{
  "settings": {
    "analysis": {
      "filter": {
        "engram_analyzer_filter": {		//定义edge_ngram过滤器
          "type": "ngram",
          "min_gram" : 1,
          "max_gram" : 30
        }
      },
      "analyzer": {
        "engram_analyzer": {			//创建分词器
          "type" : "custom",
          "tokenizer" : "standard",
          "filter" : [
            "lowercase",
            "engram_analyzer_filter"
          ]

        }
      }
    }    
  }, 
  "mappings": {
    "doc" : {
      "properties": {
        "note" : {
          "type": "text",
          "analyzer": "engram_analyzer"		//使用自定义的engram_analyzer
        }
      }
    }
  }
}
```

edge ngram是使用首字母向后实现单词内的逐字符拆分。如单词hello。可提高前缀搜索或搜索推荐效率

```txt
h
he
hel
hell
hello
```

使用edge ngram搜索

```txt
=================================创建edge ngram索引====================================
PUT /engram
{
  "settings": {
    "analysis": {
      "filter": {
        "engram_analyzer_filter": {		//定义edge_ngram过滤器
          "type": "edge_ngram",
          "min_gram" : 1,
          "max_gram" : 30
        }
      },
      "analyzer": {
        "engram_analyzer": {			//创建分词器
          "type" : "custom",
          "tokenizer" : "standard",
          "filter" : [
            "lowercase",
            "engram_analyzer_filter"
          ]

        }
      }
    }    
  }, 
  "mappings": {
    "doc" : {
      "properties": {
        "note" : {
          "type": "text",
          "analyzer": "engram_analyzer"		//使用自定义的engram_analyzer
        }
      }
    }
  }
}
```

**模糊搜索**

fuzzy技术就是用于解决错误拼写的（在英文中很有效，在中文中几乎无效）

```txt
GET /es/doc/_search
{
  "query": {
    "fuzzy": {
      "note": {
        "value":  "peple",
        "fuzziness": 2			//允许错误的个数
      }
    }
  }
}
```

### 高亮显示

**plain highlighting**

默认的高亮搜索模式，也是lucene底层实现的highlighting

```txt
GET /es/doc/_search
{
 "query": {
   "match": {
      "note": "life"
    }
 },
 "highlight": {
   "pre_tags": "<color='red'>",		//高亮前缀
   "post_tags": "</color>",			//高亮后缀
   "fields": {
     "note": {
       "fragment_size": 20,			/返回每段的文本长度
       "number_of_fragments": 2		//最大片段数,影响返回的片段数
     }
   }
 }
}
```

**posting highlighting**

posting highlighting需要在创建索引的时候，在mapping中手工开启

```txt
PUT /posthightling
{
  "mappings": {
    "doc": {
      "properties": {
        "remark": {
          "type": "text",
          "analyzer": "standard",
          "index_options": "offsets"
        }
      }
    }
  }
}
```

这种高亮搜索模式设定后，不会对高亮搜索语法有任何的影响。只是在底层实现逻辑不同。

①性能比plain highlight要高，因为不需要重新对高亮文本进行分词。

②对磁盘的消耗更少。

如果索引常需要搜索且需要高亮显示建议使用posting highlighting

**fast vector highlighting**

fast vector highlighting模式和term vector有关，是借助index-time term vector来实现高亮显示。对于大field数据【大于1M】的高亮搜索性能更高。高亮搜索语法不变。

```txt
PUT /posthightling
{
  "mappings": {
      "properties": {
        "remark": {
          "type": "text",
          "analyzer": "standard",
          "term_vector": "with_positions_offsets"
        }
    }
  }
}
```

### 地理位置搜索

- 定义geo point mapping

```txt
PUT /hotel_app
{
  "mappings": {
    "hotels": {
      "properties": {
        "name": {
          "type": "text",
          "analyzer": "ik_smart"
        },
        "pin": {
          "type": "geo_point"
        }
      }
    }
  }
}
```

- **搜索指定区域范围内的数据**

  - 矩形范围搜索【遵循左上右下】

  ```txt
  GET /hotel_app/hotels/_search
  {
    "query": {
      "bool": {
        "filter": {
          "geo_bounding_box": {
            "pin": {
              "top_left": {
                "lat": 40.73,				#latitude纬度
                "lon": -74.1				#longitude经度
              },
              "bottom_right": {
                "lat": 40.717,
                "lon": -70.99
              }
            }
          }
        }
      }
    }
  }
  ```

  - 多边形搜索【不需要注意多边形点的顺序】

  ```txt
  GET /hotel_app/hotels/_search
  {
    "query": {
      "bool": {
        "filter": {
          "geo_polygon": {
            "pin": {
              "points": [
                {"lat" : 40.73, "lon" : -74.1},
                {"lat" : 40.01, "lon" : -71.12},
                {"lat" : 50.56, "lon" : -90.58}
              ]
            }
          }
        }
      }
    }
  }
  ```

- **搜索某地点附近的数据**

```txt
GET /hotel_app/hotels/_search
{
  "query": {
    "bool": {
      "filter": {
        "geo_distance": {
          "distance": "100km",
          "pin": {
            "lat": 40.73,
            "lon": -71.1
          }
        }
      }
    }
  }
}
```

### Span查询

​	跨度查询是low-level查询，可以对指定术语的顺序和接近度进行专业控制。这些通常用于对法律文件或专利实施非常具体的查询。跨度查询不能与非跨度查询混合（span_multi查询除外）。跨度查询是非常专业的查询方式，在一般的商业应用中并不多见。

**span-term**【只能作用在分词字段】

如果单独使用，效果跟term查询差不多，但是一般还是用于其他的span查询的子查询。

```txt
GET cars/_search
{
  "query": {
    "span_term": {
      "remark": {
        "value": "大众"
      }
    }
  }
}
```

**span-multi**

可以嵌套的跨度查询，可以将其他的query搜索方案包装为一个span query跨度查询。如：通配符、前缀搜索、正则搜索、模糊搜索（fuzzy）等。

```txt
========================================前缀搜索=======================================
GET cars/_search
{
  "query": {
    "span_multi": {
      "match": {
        "prefix": {
          "remark": {
            "value": "大众"
          }
        }
      }
    }
  }
}
========================================正则搜索=======================================
GET cars/_search
{
  "query": {
    "span_multi": {
      "match": {
        "regexp": {
          "remark": {
            "value": "神*"
          }
        }
      }
    }
  }
}
=======================================通配符搜索======================================
GET cars/_search
{
  "query": {
    "span_multi": {
      "match": {
        "wildcard": {
          "remark": {
            "value": "大?"
          }
        }
      }
    }
  }
}
========================================模糊搜索=======================================
GET cars/_search
{
  "query": {
    "span_multi": {
      "match": {
        "fuzzy": {
          "remark": {
            "value": "高档车",
            "fuzziness": 1
          }
        }
      }
    }
  }
}
```

**span_first**

跨度查询中匹配字段开头数据，其中的match对应任意一个span query类型。end代表在这个span query匹配过程中，搜索条件数据在字段中最大的匹配范围。即（position<end）

```txt
GET cars/_search
{
  "query": {
    "span_first": {
      "match": {
        "span_term": {
          "remark": "中档车"
        }
      },
      "end": 3
    }
  }
}
```

**span_near**

跨度搜索中的近似匹配，类似match_phrase+slop组成的近似匹配。

```txt
GET cars/_search
{
  "query": {
    "span_near": {
      "clauses": [
        {
          "span_term": {
            "remark": {
              "value": "大众"
            }
          }
        },
        {
          "span_term": {
            "remark": {
              "value": "中档"
            }
          }
        }
      ],
      "slop": 2,						//最大的跨度
      "in_order": false					//顺序是否绝对
    }
  }
}
```

**span_or**

类似should逻辑。搜索span_or中的多个搜索条件的结果并集。

```txt
GET cars/_search
{
  "query": {
    "span_or": {
      "clauses": [
        {
          "span_term": {
            "remark": {
              "value": "专用"
            }
          }
        },
        {
          "span_term": {
            "remark": {
              "value": "小资"
            }
          }
        }
      ]
    }
  }
}
```

**span_not**

搜索条件的差集，不是搜索结果的差集。【大众中档车依旧会出现】

```txt
GET cars/_search
{
  "query": {
    "span_not": {
      "include": {
        "span_term": {
          "remark": {
            "value": "大众"
          }
        }
      },
      "exclude": {
        "span_term": {
          "remark": {
            "value": "中档车"
          }
        }
      }
    }
  }
}
```

**span_containing**

这个查询内部会有多个子查询，但是会设定某个子查询优先级更高，作用更大，通过关键字little和big来指定。Big条件是一个更加精确的条件，little条件是一个相对宽泛条件。Big条件必须包含little条件。

```txt
GET cars/_search
{
  "query": {
    "span_containing": {
      "little": {
        "span_term": {
          "remark": {
            "value": "大众"
          }
        }
      },
      "big": {
        "span_near": {
          "clauses": [
            {
              "span_term": {
                "remark": {
                  "value": "肝疼"
                }
              }
            },
            {
              "span_term": {
                "remark": {
                  "value": "大众"
                }
              }
            }
          ],
          "slop": 12,
          "in_order": false
        }
      }
    }
  }
}
```

**span_within**

这个查询与span_containing查询作用差不多，不过span_containing是基于lucene中的SpanContainingQuery，而span_within则是基于SpanWithinQuery。

### 搜索模板

**一次性调用的template**

此方式没太大意义，只能调用一层的template是没有意义的

```txt
===================================方式一【简单参数】===================================
GET /cars/sales/_search/template
{
  "source": {
    "query": {
      "match": {
        "remark": "{{kw}}"						//使用{{tag}}表示变量
      }
    },
    "from": "{{from}}{{^from}}100{{/from}}", 	//默认值设置
    "size": "{{size}}"
  },
  "params": {									//传入变量
    "kw": "大众",
    "size": 2
  }
}
===================================方式二【json方式】===================================
GET /cars/sales/_search/template
{
  "source": "{ \"query\": { \"match\": {{#toJson}}parameter{{/toJson}} }}",	//字符串
  "params": {
    "parameter" : {
      "remark" : "大众"
    }
  }
}
===================================方式三【join方式】===================================
GET /cars/sales/_search/template
{
  "source" : {
    "query" : {
      "match" : {
        "remark" : "{{#join delimiter=' '}}kw{{/join delimiter=' '}}"
      }
    }
  },
  "params": {
    "kw" : ["大众", "标致"]
  }
}
```

**可重复调用的template**

```txt
=====================================创建template=====================================
POST _scripts/test
{
  "script": {
    "lang": "mustache",
    "source": {
      "query": {
        "match" : {
          "remark" : "{{kw}}"
        }
      }
    }
  }
}
=====================================调用template=====================================
GET /cars/sales/_search/template
{
  "id": "test",
  "params": {
    "kw": "大众"
  }
}
```

**查询已定义的template**

```txt
GET _scripts/test
```

**删除template**

```txt
DELETE _scripts/test
```

## 八、查询过滤

### 过滤语法

过滤的时候，不进行任何的匹配分数计算，且filter内置cache自动缓存常用的filter数据，有效提升过滤速度，相对于query来说，filter相对效率较高。Query要计算搜索匹配相关度分数。Query更加适合复杂的条件搜索。Query和Filter并行执行。

```text
GET /es/doc/_search
{
  "query": {
    "bool": {
     "filter": {		//过滤，在已有的搜索结果中进行过滤。满足条件的返回。
        "range": {
          "qt": {
            "gt": 16
          }
        }
      },
      "must": [
        {
          "match": {
            "name": "test_doc_2"
          }
        }
      ]
    }
  }
}
```

如果单纯的过滤数据，不需要增加额外的搜索匹配，也可以使用下述语句：

```txt
GET /es/doc/_search
{
  "query": {
    "constant_score": {				//不进行分数判定
      "filter": {
        "match": {
          "note": "people"
        }
      }
    }
  }
}
```

### filter 底层执行原理

- 构建bitset

  bitset是一个二进制的数组，数组下标对应的是当前index中document的搜索顺序。在bitset中，0代表不符合搜索条件，1代表符合搜索条件。ES使用bitset来标记document是否符合搜索条件，也是为了用最简单的数据结构来描述搜索结果，来节省内存的开销。

- 遍历bitset

  因为一个搜索中可以有多个filter条件，搜索在ES执行后，filter对应的bitset可能有多个，ES会遍历每个filter对应的bitset，且从最稀疏的开始遍历【1最少的】，以达到效率最佳。当所有的bitset遍历结束后，所有的filter条件过滤完成。得到过滤结果。

- caching bitset

  ES会缓存bitset数据到内存中，如果后续的搜索语法中，有相同的filter条件，那么从缓存中直接使用对应的bitset来过滤数据，效率最优。其缓存机制是：最近的256个filter中，如果某个filter执行次数达到一定的数量和一定条件，则缓存bitset。【如果数据少于1000则不缓存，如果索引分段数据不足索引整体数据的3%也不缓存】

- 执行特性

  - query和filter的执行顺序

    一般情况，ES会先执行filter，再执行query，因为filter效率高，能够过滤掉不符合搜索条件的数据。

  - bitset cache 自动更新

    如果document的数据修改，或新增、删除了document，那么缓存的bitset会自动更新，保证缓存bitset的数据有效性。

  - bitset cache的应用时机

    只要在执行query的时候包含filter过滤条件，会先检查cache中是否有这个filter的bitset cache，如果有，则直接使用，如果没有，则搜索过滤index中的document。

## 九、搜索相关

### match底层转换

​	ElasticSearch执行match搜索时，通常都会对搜索条件进行底层转换，来实现最终的搜索结果。

```txt
=====================================================================================
GET /student/java/_search
{
  "query": {
    "match": {
      "remark": "java developer"
    }
  }
}
//转换后为：
GET /student/java/_search
{
  "query": {
    "bool": {
      "should": [
        {
          "term": {
            "remark": {
              "value": "java"
            }
          }
        },
        {
          "term": {
            "remark": {
              "value": "developer"
            }
          }
        }
      ]
    }
  }
}
====================================================================================
GET /student/java/_search
{
  "query": {
    "match": {
      "remark": {
        "query": "java developer",
        "operator": "and"
      }
    }
  }  
}
//转换后为：
GET /student/java/_search
{
  "query": {
    "bool": {
      "must": [
        {
          "term": {
            "remark": {
              "value": "java"
            }
          }
        },
        {
          "term": {
            "remark": {
              "value": "developer"
            }
          }
        }
      ]
    }
  }
}
====================================================================================
GET /student/java/_search
{
  "query": {
    "match": {
      "remark": {
        "query": "java developer architect", 
        "minimum_should_match": "67%"  
      }
    }
  }
}
//转换后为：
GET /student/java/_search
{
  "query": {
    "bool": {
      "should": [
        {
          "term": {
            "remark": {
              "value": "java"
            }
          }
        },
        {
          "term": {
            "remark": {
              "value": "developer"
            }
          }
        },
        {
          "term": {
            "remark": {
              "value": "architect"
            }
          }
        }
      ],
      "minimum_should_match": 2
    }
  }
}
```

*建议，如果不怕麻烦，尽量使用转换后的语法执行搜索，效率更高。*

### 搜索语法分析

​	validate语法可以对搜索语法进行分析。检查是否有错误语法。可以辅助搜索语法分析，排除错误。通常用于复杂搜索，检查语法格式。

```txt
==================================对所有index进行校验===================================
GET _validate/query?explain			//_validate==>进行校验   explain==>具有说明性的解释
{
  "query": {
    "match": {
      "note": "people"
    }
  }
}
==================================对指定index进行校验===================================
GET /es/doc/_validate/query?explain
{
  "query": {
    "match": {
      "note": "people"
    }
  }
}
```

### Term Vector

term vector是用于获取document中的某个field内的各个term的统计信息。这些统计信息包含下述内容：

- term information: 
  - term frequency in the field（term在一个field中出现次数）
  - term positions（term在field中的下标）
  -  start and end offsets（起始结束下标）
  - term payloads（term编号，由ES维护）

- term statistics: 【term_statistics=true】
  -  total term frequency（一个term在所有document中出现的频率）
  - document frequency（有多少document包含这个term）

- field statistics: 
  - document count（有多少document包含这个field）
  - sum of document frequency（一个field中所有term的document frequency之和）
  -  sum of total term frequency（一个field中的所有term的term frequency in the field之和）

term statistics和field statistics并不精准，不会被考虑有的doc可能被删除的情况，因为不是即时删除document数据，所以在统计上会有误差。

通常来说，term vector很少使用，如果使用都是用于对某些数据进行数据探查。

**term vector数据的出现时机**

- index-time

  在创建index的时候，通过mapping开启term vector统计，在document录入index的过程中就会自动完成统计信息的记录。适合频繁进行term vector数据探查的index使用。

  ```txt
  PUT /vector
  {
    "mappings": {
      "doc": {
        "properties": {
          "text": {
            "type": "text",
            "term_vector": "with_positions_offsets_payloads",
            "store": true,
            "analyzer": "english"
          },
          "fullname": {
            "type": "text",
            "analyzer": "english"
          }
        }
      }
    }
  }
  ```

- query-time

  在查询term vector数据的时候，现场进行数据统计并返回结果，这种方式也称为on the fly。适合在很少进行term vector数据探查的index使用。

  ```txt
  GET /vector/doc/1/_termvectors
  {
    "fields" : ["text"],
    "offsets" : true,
    "payloads" : true,
    "positions" : true,
    "term_statistics" : true,
    "field_statistics" : true
  }
  ```

**multi term vector**

一次性查看若干个document中的term vector

```txt
GET _mtermvectors
{
   "docs": [
      {
         "_index": "vector",
         "_type": "doc",
         "_id": "2",
         "term_statistics": true
      },
      {
         "_index": "vector",
         "_type": "doc",
         "_id": "1",
         "fields": ["text"]
      }
   ]
}
```

**探查指定term的term vector**

构建文档探查指定term的vector信息

```txt
GET /vector/doc/_termvectors
{
  "doc" : {
    "fullname" : "Leo Li",
    "text" : "hello test"
  },
  "fields" : ["text", "fullname"],
  "offsets" : true,
  "payloads" : true,
  "positions" : true,
  "term_statistics" : true,
  "field_statistics" : true,
  "per_field_analyzer" : {
    "text": "english"					//指定分词器
  }
}
```

**term vector filter**

```txt
GET /vector/doc/_termvectors
{
  "doc" : {
    "fullname" : "Leo Li",
    "text" : "hello testing"
  },
  "fields" : ["text", "fullname"],
  "offsets" : true,
  "payloads" : true,
  "positions" : true,
  "term_statistics" : true,
  "field_statistics" : true,
  "per_field_analyzer" : {
    "text": "english"
  },
  "filter" : {
    "max_num_terms" : 3,				//最多现实多少个term的统计数据
    "min_term_freq" : 1,				//term在一个field中最少出现次数
    "min_doc_freq" : 1					//term在一个document中最少出现次数
  }
}
```

## 十、聚合搜索

前序：分析，分组，聚合，字符串排序等操作需要设置正排索引【子字段keyword】或者字段的fielddata为true(建立正排索引)

> 正排索引：类似数据库中的普通索引。依赖倒排索引中的数据，不做二次解析，将倒排索引内容重设一份正排索引，用于内存计算

```txt
PUT /products_index/phone_type/_mapping
{
  "properties": {
    "tags":{              //设置的字段
      "type": "text", 
      "fielddata": true   //fielddata开启	
    }
  }
}
```

### 词元统计

在ElasticSearch中默认为分组数据做排序使用的是`_count`[统计个数]执行降序排列。也可以使用`_key`[统计列的内容]元数据

```txt
GET /cars/sales/_search
{
  "size": 0,              //返回查询命中记录，如果为0则不返回
  "aggs": {
    "group_by_color": {   //聚合返回结果集的标签名
      "terms": {          //词元统计
        "field": "color", //统计color每个词元总数
        "order": {
          "_count": "asc"
        }
      }
    }
  }
}
```

### 计算平均值

```txt
GET /products_index/phone_type/_search
{
  "size": 0,              //返回查询命中记录，如果为0则不返回
  "aggs": {
    "avg_price": {        //聚合返回结果集的标签名
      "avg": {            //计算平均值
        "field": "price"  //计算的字段
      }
    }
  }
}
```

### 最大值&最小值&总计

```txt
GET /cars/sales/_search
{
  "size": 0, 
  "aggs": {
    "max_price": {
      "max": {						//最大值
        "field": "price"
      }
    },
    "min_price": {
      "min": {						//最小值
        "field": "price"
      }
    },
    "sum_price": {					//总计
      "sum": {
        "field": "price"
      }
    }
  }
}
```

### 聚合嵌套【下钻】

聚合是可以嵌套的，内层聚合是依托于外层聚合的结果之上实现聚合计算。

```txt
=================================父聚合不使用子聚合字段==================================
GET /products_index/phone_type/_search
{
  "query": {
    "match": {
      "name": "plus"
    }
  },
  "aggs": {
    "count_term_tags": {     //聚合进行词元统计
      "terms": {
        "field": "tags"
      },
      "aggs": {              //在词元统计的结果之下进行求平均值
        "avg_price": {
          "avg": {
            "field": "price"
          }
        }
      }
    }
  }
}
================================父聚合使用子聚合字段排序=================================
GET /cars/sales/_search
{
  "size": 0, 
  "aggs": {
    "group_by_color": {
      "terms": {
        "field": "color",
        "order": {					//只能使用直接子聚合，不能使用孙子代
          "avg_by_price": "asc"
        }
      },
      "aggs": {
        "avg_by_price": {
          "avg": {
            "field": "price"
          }
        }
      }
    }
  }
}
```

### 聚合平铺

聚合类嵌套的聚合条件是平级关系

```txt
GET /cars/sales/_search
{
  "size": 0, 
  "aggs": {
    "group_by_color": {
      "terms": {
        "field": "color"
      },
      "aggs": {
        "avg_by_price_color": {    //和group_by_brand平级
          "avg": {
            "field": "price"
          }
        },
        "group_by_brand": {        //和avg_by_price_color平级
          "terms": {
            "field": "brand"
          }
        }
      }
    }
  }
}
```

### 聚合排序

聚合中如果使用order排序的话，要求排序字段必须是一个聚合相关字段【聚合下的子聚合命名】

```txt
GET /products_index/phone_type/_search
{
  "size": 0, 
  "aggs": {
    "count_term_tags": {
      "terms": {
        "field": "tags",
        "order": {
          "avg_price": "asc"  //使用子聚合avg_price排序
        }
      },
      "aggs": {
        "avg_price": {        //子聚合avg_price
          "avg": {
            "field": "price"
          }
        }
      }
    }
  }
}
```

### 范围区间

指定分组区间，统计区间数据个数

```txt
======================================range划分=======================================
GET /products_index/phone_type/_search
{
  "query": {
    "match_all": {}
  },
  "_source": "price",
  "aggs": {
    "rang_price": {				
      "range": {            //范围分组
        "field": "price",   //分组字段
        "ranges": [         //分组区间
          {
            "from": 500000,
            "to": 700000
          },
          {
            "from": 700000,
            "to": 900000
          }
        ]
      }
    }
  }
}
====================================histogram划分=====================================
GET /cars/_search
{
  "size": 0, 
  "aggs": {
    "histogarm_by_price": {
      "histogram": {				//以10w区间划分
        "field": "price",
        "interval": 100000,			//区间间隔
        "min_doc_count" : 1			//区间最少要满足多少数据才能显示，默认0
      }
    }
  }
}
==================================date_histogram划分==================================
GET /cars/_search
{
  "size": 0, 
  "aggs": {
    "histogarm_by_date": {
      "date_histogram": {         //时间区间划分
        "field": "sold_date",
        "interval": "month",      //划分规则	
        "min_doc_count": 1,
        "format": "yyyy-MM-dd",   //最大、最小时间格式
        "extended_bounds": {      //时间起始值和终止值
          "min": "2017-01-01",
          "max": "2018-12-31"
        }
      }
    }
  }
}
===================================地理范围聚合划分=====================================
GET /hotel_app/hotels/_search
{
  "size": 0,
  "aggs": {
    "agg_by_pin": {
      "geo_distance": {
        "field": "pin",
        "distance_type": "arc",   #sloppy_arc默认算法、arc最高精度、plane最高效率
        "origin": {               #原点
          "lat": 52.376,
          "lon": 4.894
        },
        "unit": "km",             #距离单位
        "ranges": [               #距离原点距离
          {
            "to": 100

          },
          {
            "from": 100,
            "to": 300
          },
          {
            "from": 300,
            "to": 3000
          }
        ]
      }
    }
  }
}
```

### top_hits聚合

对组内的数据进行排序，并选择其中排名高的数据，那么可以使用top_hits来实现。

- top_hits中的属性size代表取组内多少条数据（默认为10）

- sort代表组内使用什么字段什么规则排序（默认使用`_doc`的asc规则排序）

- source代表结果中包含document中的那些字段（默认包含全部字段）

```txt
GET /cars/_search
{
  "size": 0,
  "aggs": {
    "group_by_brank": {               //先根据brank分组
      "terms": {
        "field": "brand"
      },
      "aggs": {
        "price_rank_two": {						//计算各分组类价钱最高的2个
          "top_hits": {
            "size": 2,
            "sort": [{"price": "desc"}],  //排序字段
            "_source": {                  //返回doc的信息
              "includes": ["model", "price"]
            }
          }
        }
      }
    }
  }
}
```

### Global Bucket

在聚合统计数据的时候，有些时候需要对比部分数据和总体数据。global是用于定义一个全局bucket，这个bucket会忽略query的条件，检索所有document进行对应的聚合统计。

```txt
GET /cars/_search
{
  "size": 0, 
  "query": {
    "term": {
      "brand": "大众"
    }
  },
  "aggs": {
    "w_avg_by_price": {
      "avg": {
        "field": "price"
      }
    },
    "aggs": {
      "global": {},         //不使用query搜索条件进行统计
      "aggs": {
        "all_avg_by_price": {
          "avg": {
            "field": "price"
          }
        }
      }
    }
  }
}
```

### 聚合内filter

这个过滤器只对query搜索得到的结果执行filter过滤。如果filter放在aggs外部，过滤器则会过滤所有的数据。

```txt
GET /cars/_search
{
  "size": 0,
  "aggs": {
    "group_by_brand_not_w": {
      "filter": {       //过滤Query之后的条件
        "terms": {
          "brand": ["大众","奥迪"]
        }
      },
      "aggs": {
        "group_by_brand": {
          "terms": {
            "field": "brand"
          }
        }
      }
    }
  }
}
```

### 去除重复

使用`cardinality`语法去除重复数据后，统计数据量。这种语法的计算类似SQL中的distinct count计算。

```txt
GET /cars/sales/_search
{
  "size": 0, 
  "aggs": {
    "count_of_province": {
      "cardinality": {
        "field": "brand",
        "precision_threshold": 100
      }
    }
  }
}
```

cardinality使用的是Hyperloglog++算法，具有一定的错误率，可使用`precision_threshold`设置预估唯一值数量提高准确度（默认100）。

**优化cardinality**

HLL算法的实现是：对所有`cardinality`指定字段取hash值，通过hash和bitset数据结构计算cardinality结果。可使用`mapper-murmur3 plugin`插件，在index_time时就计算hash值提高效率。【mapper-murmur3 plugin安装见官网】

```txt
PUT /cars
{
  "mappings": {
    "sales": {
      "properties": {
        "price": {
          "type": "long"
        },
        "color": {
          "type": "keyword"
        },
        "brand": {
          "type": "keyword",
          "fields": {
            "hash": {				//在创建索引时计算hash值,进行cardinality操作使用此字段
              "type": "murmur3"
            }
          }
        },
        "model": {
          "type": "keyword"
        },
        "sold_date": {
          "type": "date"
        },
        "remark": {
          "type": "text",
          "analyzer": "ik_max_word"
        }
      }
    }
  }
}
```

单个文档节省的时间是非常少的，但是如果聚合一亿数据，每个字段多花费10纳秒的时间，那么在每次查询时都会额外增加1秒，如果我们要在非常大量的数据里面使用 cardinality ，我们可以权衡使用。

### 百分比算法

**percentiles**

将数字字段从小到大排序，取百分比位置数据的值。用于计算百分比数据的。如：PT50、PT90、PT99等

```txt
GET /percentiles/logs/_search
{
  "aggs": {
    "r": {
      "percentiles": {
        "field": "latency",
        "percents": [
          50,					//取50%位置数据值
          90,					//取90%位置数据值
          99					//取99%位置数据值
        ]
      }
    }
  }
}
```

**percentile_rank**

可用于统计SLA【提供的服务的标准】如网站访问延迟的SLA是：确保所有的请求的访问延时都在200毫秒以内（大型公司一般都是这种标准）

如果延时超过1秒，升级到A级故障，代表网站性能有严重问题。

```txt
GET /percentiles/logs/_search
{
  "aggs": {
    "group_by_province": {
      "terms": {
        "field": "province"
      },
      "aggs": {
        "percentile_ranks_latency": {
          "percentile_ranks": {
            "field": "latency",
            "values": [
              200,
              1000
            ],
            "keyed" : false
          }
        }
      }
    }
  }
}
```

**优化percentiles和percentile_ranks**

percentiles和percentile_ranks底层采用的都是TDigest算法，是用很多的节点来执行百分比计算，计算过程也是一种近似估计，有一定的错误率。节点越多，结果越精确（内存消耗越大）。

参数compression用于限制节点的最大数目，限制为：20 * compression。这个参数的默认值为100。即默认提供2000个节点。一个节点大约使用 32 字节的内存，所以在最坏的情况下（例如，大量数据有序存入），默认设置会生成一个大小约为 64KB 的 TDigest算法空间。 在实际应用中，数据会更随机，所以 TDigest 使用的内存会更少。

```txt
GET /test_percentiles/logs/_search
{
  "aggs": {
    "r": {
      "percentiles": {
        "field": "latency",
        "percents": [
          50,
          90,
          99
        ],
        "tdigest": {				//percentile_ranks同理
          "compression": 200
        }
      }
    }
  }
}
```

### 海量bucket优化

默认ElasticSearch执行聚合分析时，是按照top_hits深度优先的方式去执行的。会计算出每一项结果。但是遇到计算销量前五的品牌中前十的车型这种问题时，使用深度优先计算不太合适，应该逐层聚合bucket，并过滤后，在当前层的结果基础上再次执行下一层的聚合（广度优先）。

```txt
GET /cars/_search
{
  "size": 0,
  "aggs": {
    "group_by_brand": {
      "terms": {
        "field": "brand",
        "size": 5,
        "collect_mode": "breadth_first"
      },
      "aggs": {
        "group_of_model": {
          "terms": {
            "field": "model"
          }
        }
      }
    }
  }
}
```

### 聚合算法简介

- **易并行聚合算法**

  若干节点同时计算一个聚合结果，再返回给协调节点做最终计算，得到最终结果的方式。如聚合计算中的max，min等

- **近似聚合算法**

  有些聚合分析算法是很难使用易并行算法解决的，如：count(distinct)。这个时候ES会采取近似聚合的方式来进行计算。近似聚合结果不完全准确，但是效率非常高，一般效率是精准算法的数十倍。

  近似聚合： 延时一般在100毫秒左右，有5%左右的错误率。

  精准算法： 延时一般在若干秒到若干小时之间，不会有任何错误。（批处理）

**三角选择原则**

精准度 + 实时性 ：一定是数据量很小的时候，通常都在单机中执行，可以随时调用。

精准度 + 大数据 ： 这是非实时的计算，是在大数据存在的情况下，保证数据计算的精准度，一次计算可能需要若干小时才能完成，通常都是批处理程序。如：hadoop

大数据 + 实时性 ： 这是一种不精准的计算，近似估计，一般都会有一定错误率（一般5%以内）。如ES中的近似聚合算法。

## 十一、同义词与分词器

### 概念

同义词：

- 为key_words提供更加完整的倒排索引。如：时态转化，单复数转化，全写简写，同义词等。

- normalization是为了提升召回率（recall），提升搜索能力的。

- normalization是配合分词器(analyzer)完成其功能

分词器：

- 分词器的功能是处理Document中的field即创建倒排索引过程中用于切分field数据

### 默认提供的常见分词器

- `standard analyzer`：是ES中的默认分词器。标准分词器，处理英语语法的分词器。这种分词器也是ES中默认的分词器。切分过程中会忽略停止词等。会进行单词的大小写转换。过滤连接符或括号等常见符号。

- `simple analyzer`：简单分词器。就是将数据切分成一个个的单词。使用较少，经常会破坏英语语法。

- `whitespace analyzer`：空白符分词器。就是根据空白符号切分数据。使用较少，经常会破坏英语语法。

- `language analyzer`：对应语言的分词器，会忽略停止词、转换大小写、单复数转换、时态转换等。

搜索条件中的key_words也需要经过分词，且搜索条件中的条件数据使用的分词器与对应的字段使用的分词器是统一的。否则会导致搜索结果丢失。

### IK分词器

**安装步骤**

第一步：下载源码包编译打包或者下载编译完成的代码包

第二步：在elasticsearch的plugins目录下创建ik目录

第三步：将编译完成的压缩包解压到plugins的ik目录下

**IK分词器种类**

`ik_max_word`：会将文本做最细粒度的拆分，会穷尽各种可能的组合，适合 Term Query

`ik_smart`：会做最粗粒度的拆分，适合 Phrase 查询

**IK分词器配置文件**

IK的所有的dic词库文件，必须使用UTF-8字符集

```txt
config
  |
  +- extra_main.dic【自定义词库，配置方式为相对于IKAnalyzer.cfg.xml相对路径，非热更新】
  |	 
  +- extra_single_word.dic【自定义单字词，配置相对于IKAnalyzer.cfg.xml相对路径，非热更新】
  |
  +- extra_single_word_full.dic【和extra_single_word.dic内容相同】
  |
  +- extra_single_word_low_freq.dic【自定义单字低频词，非热更新】
  |
  +- extra_stopword.dic【自定义停用词库，配置方式为相对于IKAnalyzer.cfg.xml相对路径，非热更新】
  |
  +- IKAnalyzer.cfg.xml【配置加载外部扩展词文件】
  |
  +- main.dic【内置词典，一行一词，未记录的单词无法实现分词，不建议修改】
  |
  +- preposition.dic【内置的中文停用词】
  |
  +- quantifier.dic【内置数据单位词典】
  |
  +- stopword.dic【内置英文停用词】
  |
  +- suffix.dic【内置后缀】
  |
  +- surname.dic【内置姓氏词典】
```

**IK词库热更新**

修改`IKAnalyzer.cfg.xml`文件

```xml
<!--这里配置远程扩展字典 -->
<entry key="remote_ext_dict">location</entry>
<!--这里配置远程扩展停止词字典-->
<entry key="remote_ext_stopwords">location</entry>
```

 `location` 指url，比如 `http://yoursite.com/getCustomDict`，该请求需满足以下条件

- 该 http 请求需要返回两个头部(header)，一个是 `Last-Modified`，一个是 `ETag`，这两者都是字符串类型，只要有一个发生变化，该插件就会去抓取新的分词进而更新词库。

- 该 http 请求返回的内容格式是一行一个分词，换行符用 `\n` 

**测试IK分词器**

```txt
GET /_analyze
{
  "analyzer": "ik_max_word",
  "text": ["中国人民共和国国歌"]
}

GET /_analyze
{
  "analyzer": "ik_smart",
  "text": ["中国人民共和国国歌"]
}
```

## 十二、Mapping

​	Mapping决定了一个index中的field使用什么数据格式存储，使用什么分词器解析，是否有子字段，是否需要copy to其他字段等。Mapping决定了index中的field的特征。

### 查询当前索引mapping信息

```txt
GET /book_index/_mapping

=======================================返回信息=======================================
{
  "book_index" : {          //索引名称
    "mappings" : {          //mapping开始
      "book_type" : {       //类型名称
        "properties" : {			//具体信息
          "author_id" : {			//字段名
            "type" : "long"			//类型名
          },
          "content" : {
            "type" : "text",				
            "fields" : {			//子字段列表默认为text类型字段提供的子字段名称为keyword
              "keyword" : {			//不分词的keyword
                "type" : "keyword",	
                "ignore_above" : 256	//最长长度
              }
            }
          },
          "post_date" : {
            "type" : "date"
          },
          "title" : {
            "type" : "text",
            "fields" : {
              "keyword" : {
                "type" : "keyword",
                "ignore_above" : 256
              }
            }
          }
        }
      }
    }
  }
}
```

字段类型为text，分词器为standard，一定创建xxx.keyword子字段，类型是keyword类型，长度为256个字符

### mapping核心数据类型

| 类型名称 | 关键字                     |
| -------- | -------------------------- |
| 字符串   | text（string）             |
| 整数     | byte、short、integer、long |
| 浮点型   | float、double              |
| 布尔型   | boolean                    |
| 日期型   | date                       |

### dynamic mapping

​	ES自动建立index，创建type，以及type对应的mapping，mapping中包含了每个field对应的数据类型，以及如何分词等设置

dynamic mapping对字段的类型分配

| 数据          | 类型                                 |
| ------------- | ------------------------------------ |
| true or false | boolean                              |
| 123           | long                                 |
| 123.123       | double                               |
| 2018-01-01    | date                                 |
| hello world   | text（string）【默认standard分词器】 |

### custom mapping

手工创建mapping时，只能新增mapping设置，不能对已有的mapping进行修改

```txt
PUT /book_index
{
  "settings": {						//定义分片信息
    "number_of_shards": 5,
    "number_of_replicas": 1
  },
  "mappings": {						//定义mapping
    "book_type": {					//类型名称
      "properties": {				//mapping信息
        "author_id": {				//字段名
          "type": "integer",		//索引类型
          "index": false			//是否索引，默认true
        },
        "title": {
          "type": "text",
          "analyzer": "standard",
          "fields": {				//子字段类型
            "keyword": {
              "type": "keyword",
              "ignore_above": 256
            }
          }
        },
        "content": {
          "type": "text",
          "analyzer": "ik_smart"
        },
        "post_date": {
          "type": "date"
        }
      }
    }
  }
}
====================================增加mapping定义====================================
PUT /book_index/book_type/_mapping
{
  "properties": {
    "new_field": {				//字段名
      "type": "text",
      "analyzer": "standard",
      "store": false			//是否存储，默认true
    }
  }
}
=====================================测试mapping======================================
GET /book_index/_analyze
{
  "field": "content",
  "text": ["中国人民共和国国歌"]
}
```

### mapping的复杂定义

- **multi field**

  数组类型和普通数据类型没有区别。只是要求字段中的多个数据的类型必须相同。

  ```txt
  PUT /multi_index/multi_type/1
  {
    "tags": ["tagA", "tagB", "tagC"]
  }
  =================================手工对应mapping=================================
  PUT /multi_index
  {
    "mappings": {
      "multi_type": {
        "properties": {
          "tags": {
            "analyzer": "standard",
            "type": "text"
          }
        }
      }
    }
  }
  ```

- **empty field**

  空数据 ：`null`，`[]`，`[null]`

  空数据如果直接保存到index中，由ES为index自动创建mapping，那么此空数据对应的field将不会创建mapping映射信息。而任意的mapping定义都可以保存空数据

  ```txt
  PUT /empty_index/empty/1
  {
    "filed_1": null,
    "filed_2": [],
    "filed_3": [null]
  }
  ==================================mapping映射信息==================================
  GET /empty_index/empty/_mapping
  
  //返回结果
  {
    "empty_index" : {
      "mappings" : {
        "empty" : { }				//无field_1、field_2、field_3信息
      }
    }
  }
  ```

- **object field**

  ES在底层存储对象数据时实际上使用`对象.属性`方式进行存储

  ```txt
  PUT /object_index/object_type/1
  {
    "name" : "张三" ,
    "age" : 20,
    "address" : {
      "province" : "北京",
      "city" : "北京",
      "street" : "永泰庄东路"
    }
  }
  ===============================对应的手动mapping映射===============================
  PUT /object_index
  {
    "mappings": {
      "object_type": {
        "properties": {
          "name": {
            "type": "text",
            "analyzer": "ik_max_word"
          },
          "age": {
            "type": "short"
          },
          "address": {
            "properties": {
              "province": {
                "type": "text",
                "analyzer": "ik_max_word"
              },
              "city": {
                "type": "text",
                "analyzer": "ik_max_word"
              },
              "street": {
                "type": "text",
                "analyzer": "ik_max_word"
              }
            }
          }
        }
      }
    }
  }
  ```

- **数组+对象**

  可将包含对象的数组视为对象

  ```txt
  PUT /arr_object_index/arr_object_type/1
  {
    "dept_name" : "销售部", 
    "emps" : [
      { "name" : "张三", "age" : 20 },
      { "name" : "李四", "age" : 21 },
      { "name" : "王五", "age" : 22 }
    ]
  }
  ===============================对应的手动mapping映射===============================
  PUT /arr_object_index
  {
    "mappings": {
      "arr_object_type": {
        "properties": {
          "dept_name": {
            "type": "text",
            "analyzer": "ik_max_word"
          },
          "emps": {
            "properties": {
              "name": {
                "type": "text",
                "analyzer": "ik_max_word"
              },
              "age": {
                "type": "short"
              }
            }
          }
        }
      }
    }
  }
  ```

- **copy_to**

  就是将多个字段，复制到一个字段中，实现一个多字段组合。

  ```txt
  =====================================创建索引======================================
  PUT /products_index
  {
    "mappings": {
      "doc": {
        "properties": {
          "name": {
            "type": "text",
            "analyzer": "standard",
            "copy_to": "search_key"
          },
          "remark": {
            "type": "text",
            "analyzer": "standard",
            "copy_to": "search_key"
          },
          "price": {
            "type": "integer"
          },
          "producer": {
            "type": "text",
            "analyzer": "standard",
            "copy_to": "search_key"
          },
          "tags": {
            "type": "text",
            "analyzer": "standard",
            "copy_to": "search_key"
          },
          "search_key": {					//此字段需要和被copy类型分词器相同
            "type": "text",
            "analyzer": "standard"
          }
        }
      }
    }
  }
  =======================================搜索=======================================
  GET /products_index/doc/_search
  {
    "query": {
      "match": {
        "search_key": "IPHONE"
      }
    }
  }
  ```

  copy_to字段【上述的search_key】未必存储，但是在逻辑上是一定存在

### root object

​	mapping的root object就是指设置index的mapping时，一个type对应的json数据。包括的内容有：properties， metadata（_id, _source, _all）等。

```txt
PUT /test_index9
{
  "settings" : {
    "number_of_shards" : 2,
    "number_of_replicas" : 1
  },
  "mappings" : {
    "test_type" : {				//==============root object开始
      "properties" : {
        "post_date" : { "type" : "date" },
        "title" : { "type" : "text", "index" : false },
        "content" : { "type" : "text" , "analyzer" : "english" },
        "author_id" : { "type" : "integer" }
      },
      "_all" : { "enabled" : false },
      "_source" : { "enabled" : false }
    }							//==============root object结束
  }
}
```

### 定制dynamic策略

​	ES中支持在自定义mapping时，为type定制dynamic mapping策略，让index更加的友好。定制dynamic mapping策略时，可选值有：true（默认值）-遇到陌生字段自动进行dynamic mapping， false-遇到陌生字段，不进行dynamic mapping（会保存数据，但是不做倒排索引，无法实现任何的搜索），strict-遇到陌生字段，直接报错。

```txt
PUT /dynamic_strategy
{
  "mappings": {
     "dynamic_type": {
        "dynamic": "strict",
        "properties": {
        "f1" :{
          "type": "text",
          "analyzer": "standard"
        },
        "f2": {
          "type": "object",
          "dynamic" :false
        }
      }
     }
  }
}
=======================================插入测试=======================================
PUT /dynamic_strategy/dynamic_type/1
{
  "f1": "f1 field",
  "f2": {
    "f21": "f21 field"			//只会存储不会分词，不能索引
  },
  "f3": "f3 field"				//f3字段无法插入，会直接报错
}
```

定制dynamic mapping，使用较少，因为很难去分析出一套完整的，有扩展能力的结构。如果使用，一般在固定的，几乎不会改变的数据结构中使用。如：身份证信息

## 十三、Document写入原理

​	ElasticSearch为了实现搜索的近实时，结合了内存buffer、OS cache、disk三种存储，尽可能的提升搜索能力。ElasticSearch的底层使用lucene实现。在lucene中一个index是分为若干个segment（分段）的，每个segment都会存放index中的部分数据。在ElasticSearch中，是将一个index先分解成若干shard，在shard中，使用若干segment来存储具体的数据。

![document-write-process](/img/searchengine/document-write-process.png)

1）客户端发起增删改请求

2）将请求的document写入到buffer。并默认每秒刷新一次buffer

3）在将document写入buffer的同时也会将本次操作的具体内容写入到translog中，这个日志文件记录了所有操作过程。其意义在于保证异常宕机尽可能少的丢失数据

4）ElasticSearch每秒中都会刷新buffer，将buffer中的数据保存到index segment中

5）在index segment创建并写入数据后，ElasticSearch会立刻将其写入到系统缓存（OS cache）中，在写入后index segment立刻被打开，可以为客户端的搜索请求提供服务【不必等待index segment写入到磁盘后再打开】

6）translog记录的是操作过程，ElasticSearch宕机重启后会读取系统磁盘中的数据和translog中的日志，实现数据恢复。默认每5秒钟执行一次translog持久化，如果在持久化的时候刚好有document写操作在执行，那么这次持久化操作会等待写操作彻底完成后才执行。可以通过设置修改translog的持久化为异步。

7）随着时间的推移，translog文件会不断增大。当其大到一定程度的时候或一定时间后【默认30分钟】，会触发commit操作。commit操作具体内容有：①将buffer中的数据刷新到一个新的index segment中，将index segment写入到OS cache中并打开index segment为搜索提供服务②清空buffer③执行一个commit point操作，将OS cache中所有的index segment标识记录在这个commit point中并持久化到系统磁盘DISK中④commit point中记录的index segment会被持久化到DISK中⑤清空本次持久化的index segment对应的translog文件中的日志内容

8）ElasticSearch会自动的执行segment merge操作。被标记为deleted状态的document会在此时被物理删除。merge的流程是：①选择一些大小相近的segment文件，merge成一个大的segment文件②将merge后的文件持久化到磁盘中③执行一个commit操作，commit point除记录要持久化的OS cache中的index segment外，还记录了merge后的segment文件和要删除的原segment文件④commit操作执行成功后，将merge后的segment打开为搜索提供服务，将旧的segment关闭并删除。

## 十四、相关度评分算法

### 算法介绍

​	ES中使用的是term frequency / inverse document frequency算法，简称TF/IDF算法。是ElasticSearch相关度评分算法的一部分，也是最重要的部分。

`TF `：计算搜索条件分词后，各词条在document的field中出现了多少次，出现次数越多，相关度越高。

`IDF`：计算搜索条件分词后，各词元在整个index中的所有document中出现的次数越多，相关度越低。

`Field-length Norm`：在匹配成功后，计算field字段数据的长度，长度越大，相关度越低。【TF算法的一部分】

ES中底层计算相关度的时候，不是简单的加减乘除，有其特有的算法

```txt
GET /es/doc/_search
{
  "query": {
    "match": {
      "note": "people"
    }
  },
  "explain": true				//使用explain查看评分的详细解释
}
```

### 相关**度分数计算步骤**

**第一步：boolean model**

​	ElasticSearch搜索的时候，首先根据搜索条件，过滤符合条件的document。这个时候ES是不做相关度分数计算的，只是记录true或false，标记document是否符合搜索要求。

**第二步：TF/IDF**

​	用于计算单个term在document中的相关度分数

**第三步：**

​	向量空间模型算法。用于计算多个term对一个document的搜索相关度分数。

​	ElasticSearch会根据一个term对于所有document的相关度分数，来计算得出一个query vector。在根据多个term对于一个document的相关度分数，来计算得出若干document vector。将这些计算结果放入一个向量空间，再通过一些数学算法来计算document vector相对于query vector的相似度，来得到最终相关度分数。

### 调节、优化相关对评分的方式

**query-time boost**

指搜索的时候，提供boost，来影响相关度评分

```txt
GET /student/java/_search
{
  "query": {
    "bool": {
      "should": [
        {
          "term": {
            "remark": {
              "value": "developer",
              "boost":2				//默认值为1
            }
          }
        },
        {
          "term": {
            "remark": {
              "value": "architect"
            }
        }
        }
      ]
    }
  }
}
```

**negative boost**

用于降低一个term的相关度分数比例，使之往后排列

```txt
GET /es/doc/_search
{
  "query": {
    "boosting": {
      "positive": {
        "match": {
          "note": "people"
        }
      }, 
      "negative": {
        "match": {
          "note": "enough"
        }
      },
      "negative_boost": 0.2
    }
  }
}
```

**constant_score**

​	不计算相关度评分，执行的时候会忽略相关度评分过程，所有的document的相关度分数都是1。在6.x中，constant_score内不能再使用query语法。只能通过filter来实现数据的过滤。所以这种影响相关度分数的方式已经没有太大的意义了。毕竟filter本来就不计算相关度分数。

**使用function score自定义相关分数算法**

​	ElasticSearch支持自定义相关度分数计算函数，这个自定义的相关度分数可以参与到ES的相关度分数计算结果中，甚至可以替换相关度评分，但无法跳过相关度分数计算步骤。很少使用。

```txt
GET /fscore/doc/_search
{
  "query": {
    "function_score": {				//
      "query": {
        "match": {
          "f": "hello spark"
        }
      },
      "field_value_factor": {
      	  //number_of_votes所在字段
          "field": "fc",
          //field字段的计算公式log(1 + number_of_votes)
          "modifier": "log1p", 	
          //可以进一步影响分数，增加后，对fc字段的计算公式为log(1 + factor * number_of_votes)
          "factor": 1.2			
        },
        //决定ES计算的相关度分数与指定字段的值如何计算
        "boost_mode": "sum",
        //限制字段fc最终计算出来的分数的最大值
        "max_boost": 0.3
    }
  }
}
```

## 十五、数据建模

### 模拟关系型数据库数据建模

```txt
PUT /product
{
  "mappings": {
    "doc": {
      "properties": {
        "pid": {
          "type": "integer"
        },
        "productName": {
          "type": "text",
          "analyzer": "ik_max_word"
        },
        "remark": {
          "type": "text",
          "analyzer": "ik_max_word"
        },
        "price": {
          "type": "integer"
        },
        "cid": {							//和category建立关联关系的字段
          "type": "integer"
        }
      }
    }
  }
}

PUT /category
{
  "mappings": {
    "category": {
      "properties": {
        "cid": {
          "type": "integer"
        },
        "categoryName": {
          "type": "text",
          "analyzer": "ik_max_word"
        },
         "pid": {							//和product建立关联关系的字段
          "type": "integer"
        }
      }
    }
  }
}
```

###  一对一数据建模

一般对数据进行组合存储。将某一个数据结构作为一部分（对象类型属性）实现数据的存储。

```txt
PUT person_index
{
  "mappings": {
    "persons": {
      "properties": {
        "last_name": {
          "type": "keyword"
        },
        "first_name": {
          "type": "keyword"
        },
        "age": {
          "type": "byte"
        },
        "identification_id": {
          "properties": {
            "id_no": {
              "type": "keyword"
            },
            "address": {
              "type": "text",
              "analyzer": "ik_max_word",
              "fields": {
                "keyword": {
                  "type": "keyword"
                }
              }
            }
          }
        }
      }
    }
  }
}
```

### 一对多数据建模

```txt
==================================一个用户对应多个地址==================================
PUT /user
{
  "mappings": {
    "doc" : {
      "properties": {
        "login_name" : {
          "type" : "keyword"
        },
        "age " : {
          "type" : "short"
        },
        "address" : {
          "properties": {
            "province" : {
              "type" : "keyword"
            },
            "city" : {
              "type" : "keyword"
            },
            "street" : {
              "type" : "keyword"
            }
          }
        }
      }
    }
  }
}
==================================多个地址对应一个用户==================================
PUT address
{
  "mappings": {
    "doc" : {
      "properties": {
        "province" : {
          "type" : "keyword"
        },
        "city" : {
          "type" : "keyword"
        },
        "street" : {
          "type" : "keyword"
        },
        "user" : {
          "properties": {
            "login_name" : {
              "type" : "keyword"
            },
            "nick_name" : {
              "type" : "keyword"
            }
          }
        }
      }
    }
  }
}
```

优点：思想简单，建模方便

缺点：数据大量冗余、耦合程度高、不易修改与维护

**nested object**

如果使用`一个用户对应多个地址`的存储方式，查询市和街道同时满足条件会返回大量不是我们想要的数据【返回中包含市匹配而街道不匹配的和街道匹配而市不匹配的】,原因是ElasticSearch对底层对象做了扁平化处理。

```txt
{
  "login_name" : "jack",
  "address.province" : [ "北京", "天津" ],
  "address.city" : [ "北京", "天津" ]
  "address.street" : [ "西三旗东路", "古文化街" ]
}

//查询北京、建材城西路上述数据也会返回但是并不是我们需要的
```

建立nested索引使ElasticSearch不进行扁平化处理

```txt
PUT /user
{
  "mappings": {
    "doc" : {
      "properties": {
        "login_name" : {
          "type" : "keyword"
        },
        "age " : {
          "type" : "short"
        },
        "address" : {
          "type": "nested", 
          "properties": {
            "province" : {
              "type" : "keyword"
            },
            "city" : {
              "type" : "keyword"
            },
            "street" : {
              "type" : "keyword"
            }
          }
        }
      }
    }
  }
}
```

ElasticSearch内部存储结构会变为

```txt
{
  "login_name" : "jack"
}
{
  "address.province" : "北京",
  "address.city" : "北京"，
  "address.street" : "西三旗东路"
}
{
  "address.province" : "北京",
  "address.city" : "北京",
  "address.street" : "西三旗东路",
}
```

**nested查询&聚合语法**

```txt
=========================================查询=========================================
GET /user_index/doc/_search
{
  "query": {
    "nested": {					//指定nested查询
      "path": "address",		//指定nested字段
      "query": {
        "bool": {
          "must": [
            {
              "match": {
                "address.province": "北京"
              }
            },
            {
              "match": {
                "address.street": "建材城西路"
              }
            }
          ]
        }
      }
    }
  }
}
================================聚合条件只包含nested字段================================
GET /user_index/user/_search
{
  "size": 0, 
  "aggs": {
    "group_by_address": {
      "nested": {						//指定nested字段信息
        "path": "address"
      },
      "aggs": {
        "group_by_province": {
          "terms": {
            "field": "address.province"
          },
          "aggs": {
            "group_by_city": {
              "terms": {
                "field": "address.city"
              }
            }
          }
        }
      }
    }
  }
}
================================聚合条件不仅包含nested字段===============================
GET /user_index/user/_search
{
  "size": 0,
  "aggs": {
    "group_by_address": {
      "nested": {
        "path": "address"
      },
      "aggs": {
        "group_by_province": {
          "terms": {
            "field": "address.province"
          },
          "aggs": {
            "reverse_aggs": {
              "reverse_nested": {},			//标识子聚合不在nested字段中进行
              "aggs": {
                "avg_by_age": {
                  "avg": {
                    "field": "age"
                  }
                }
              }
            }
          }
        }
      }
    }
  }
}
```

虽然语法变的复杂了，但是在数据的读写操作上都不会有错误发生，是推荐的设计方式。

### 文件系统数据建模

用于实现文件目录搜索

```txt
=======================================创建索引========================================
PUT /code
{
  "settings": {
    "analysis": {
      "analyzer": {
        "path_analyzer": {					//创建路径分析器
          "tokenizer": "path_hierarchy"
        }
      }
    }
  },
  "mappings": {
    "doc": {
      "properties": {
        "path": {
          "type": "text",
          "analyzer": "path_analyzer",		//使用路径分析器
          "fields": {
            "stand": {
              "type": "text",
              "analyzer": "standard"
            }
          }
        },
         "author": {
          "type": "text",
          "analyzer": "standard"
        },
        "content": {
          "type": "text",
          "analyzer": "standard"
        }
      }
    }
  }
}
=======================================查询数据========================================
GET /code/doc/_search
{
  "query": {
    "match": {
      "path": "/com"			//只能搜索出/com/kun路径开始的doc
    }
  }
}

GET /code/doc/_search
{
  "query": {
    "match": {
      "path.stand": "/com/kun"	//能搜索包含/com/kun的路径
    }
  }
}
```

### 父子关系数据建模

父子关系数据建模是模拟关系型数据库的建模方式，用不同的索引保存各自的数据，通过底层提供的父子关系，让ElasticSearch辅助实现类似关系型数据库的多表联合查询。这种建模方式，数据几乎没有冗余，且查询效率高。

```txt
==================================建立父子关系mapping==================================
PUT /ecommerce_products_index
{
  "mappings": {
    "ecommerce": {
      "properties": {
        "category_name": {
          "type": "text",
          "analyzer": "ik_max_word"
        },
        "product_name": {
          "type": "text",
          "analyzer": "ik_max_word"
        },
        "remark": {
          "type": "text",
          "analyzer": "ik_max_word"
        },
        "price": {
          "type": "long"
        },
        "sellpoint": {
          "type": "text",
          "analyzer": "ik_max_word"
        },
        "ecommerce_join_field": {			//描述父子关系
          "type": "join",					//类型为join
          "relations": {					//关系描述，父在前，子在后
            "category": "product"
          }
        }
      }
    }
  }
}
=======================================插入数据========================================
POST /ecommerce_products_index/ecommerce/_bulk
{"index" : {"_id" : "1"}}
{"category_name" : "电视", "ecommerce_join_field" : { "name" : "category" }}
{"index" : {"_id" : "2"}}
{"category_name" : "电脑", "ecommerce_join_field" : { "name" : "category" }}
{"index" : {"_id" : "3"}}
{"category_name" : "手机", "ecommerce_join_field" : { "name" : "category" }}
{"index" : {"_id" : "4", "routing" : "1"}}
{"product_name" : "长虹电视", "remark" : "国产电视", "price" : 199800, "sellpoint" : "便宜，个大", "ecommerce_join_field" : { "name" : "product", "parent" : "1" }}
{"index" : {"_id" : "5", "routing" : "1"}}
{"product_name" : "西门子电视", "remark" : "进口电视", "price" : 599800, "sellpoint" : "就是贵", "ecommerce_join_field" : { "name" : "product", "parent" : "1" }}
{"index" : {"_id" : "6", "routing" : "2"}}
{"product_name" : "Thinkpad T99", "remark" : "不知道啥时候生产", "price" : 2999800, "sellpoint" : "可能会生产吧？", "ecommerce_join_field" : { "name" : "product", "parent" : "2" }}
{"index" : {"_id" : "9", "routing" : "3"}}
{"product_name" : "IPhone X", "remark" : "齐刘海手机", "price" : 899800, "sellpoint" : "卖肾买手机", "ecommerce_join_field" : { "name" : "product", "parent" : "3" }}
```

在父子关系数据模型中，要求有关系的父子数据必须在同一个shard中保存，否则ES无法实现数据的关联管理，所以在保存子数据的时候，必须使用其对应的父数据在存储时使用的routing。默认情况下，ES使用document的id作为routing值，所以子数据在保存的时候，必须使用父数据的id作为routing才可，否则无法建立父子关系。

**父子关系查询数据**

```txt
=================================查询类型指定父类型的数据================================
GET /ecommerce_products_index/ecommerce/_search
{
  "query": {
    "parent_id": {
      "type": "product",
      "id": 1
    }
  }
}
===============================查询满足条件的子类型的父类型===============================
GET /ecommerce_products_index/ecommerce/_search
{
  "query": {
    "has_child": {
      "type": "product",
      "query": {
        "range": {
          "price": {
            "gte": 500000,
            "lte": 1000000
          }
        }
      }
    }
  }
}
===============================查询满足条件的父类型的子类型===============================
GET /ecommerce_products_index/ecommerce/_search
{
  "query": {
    "has_parent": {
      "parent_type": "category",
      "query": {
        "match": {
          "category_name": "电脑"
        }
      }
    }
  }
}
```

**父子关系聚合数据**

```txt
==================================计算品类下产品的均价===================================
GET /ecommerce_products_index/ecommerce/_search
{
  "size": 0,
  "aggs": {
    "group_by_category": {
      "terms": {
        "field": "category_name.keyword"
      },
      "aggs": {
        "products_bucket": {
          "children": {
            "type": "product"
          },
          "aggs": {
            "avg_by_price": {
              "avg": {
                "field": "price"
              }
            }
          }
        }
      }
    }
  }
}
```

###  祖孙三代关系数据模型

原理和父子关系数据模型一致，就是在数据层级关系上更加深入。`type`为`join`用于描述关系的字段mapping继续向下扩展即可。

## 十六、重建索引&零停机

### 重建索引-reindex

​	索引类型一旦建立便不可更改，如果要修改数据类型，只能重建索引：用新的设置创建新的索引并把文档从旧的索引复制到新的索引。

**方法一：用 scroll 从旧的索引检索批量文档 ，然后用 bulk API 把文档推送到新的索引中**

```txt
//准备数据
PUT /old_index/test_type/1
{
  "field" : "2018-08-08"
}
PUT /old_index/test_type/2
{
  "field" : "2018-08-07"
}
PUT /old_index/test_type/3
{
  "field" : "2018-08-06"
}

//新建索引mapping
PUT /new_index
{
  "mappings": {
     "test_type":{
        "properties":{
          "field":{
            "type": "keyword"
          }
        } 
     }
  }
}

//读取信息
GET old_index/test_type/_search?scroll=1m
{
  "query" : {
    "match_all" : {}
  },
  "sort" : [ "_doc" ],
  "size" : 1
}
GET /_search/scroll
{
  "scroll" : "1m",
  "scroll_id" : "根据具体返回结果替换"
}

//bulk批量添加数据
POST /_bulk
{"index" : {"_index" : "new_index", "_type" : "test_type", "_id" : "1"}}
{"field" : "2018-08-08"}
```

**方法二：使用_reindex迁移索引数据**

```txt
//默认情况下一次批处理1000条记录，也可支持远程索引重建
POST _reindex
{
  "source": {
    "index": "old_index"
  }, 
  "dest": {
    "index": "new_index"
  }
}
```

### 零停机

**零停机问题**：reindex索引，操作指向新的index，则必须停止后台服务，更改索引名称，这样会引起停机问题。

**解决方案**：可以使用索引别名来解决reindex中的索引名变更问题。

**别名特性**：在ElasticSearch中一个别名可以指向多个索引，一个索引也可以被多个别名指向。

```txt
//定义索引别名
PUT index_name/_alias/alias_index

//原子操作，移除就别名指向，添加新别名指向
POST _aliases
{
  "actions": [
    {
      "add": {
        "index": "new_index",
        "alias": "current_index"
      }
    },
    {
      "remove": {
        "index": "old_index",
        "alias": "current_index"
      }
    }
  ]
}
```

> 建议：在应用中使用别名而不是索引名。这样可以在任何时候重建索引，且别名的开销很小

其它使用场景：如电商中有手机索引，座机索引，使用别名电话设备作为两个索引的别名。访问电话设备，则可以在两个索引中搜索数据。

## 十七、正排索引

​	ElasticSearch在存储document时，会根据document中的field类型建立对应的索引。通常来说只创建倒排索引，倒排索引是为了搜索而存在的。但是如果对数据进行排序、聚合、过滤等操作的时候，就需要创建正排索引（doc values）。doc values会保存到磁盘中，如果OS的内存足够会被缓存。

​	Doc values只在不分词的字段中存在。倒排索引只在分词的字段中出现。正排索引和倒排索引不是在同一个字段同时存在的。但可以使用fielddata替代doc values正排索引和倒排索引在同一个字段出现。

​	如果字段类型为text。那么默认是需要分词的，则只有倒排索引。

​	如果真的需要同时使用倒排索引和正排索引，都是使用子字段的方式来实现。

### Doc Values与聚合分析内部原理

假设倒排索引的内容如下【文档模型中只有一列body】

```txt
Term        Doc_1       Doc_2       Doc_3
-------------------------------------------------------------------------------------
brown       |   X       |   X       |
dog         |   X       |           |   X
dogs        |           |   X       |   X
fox         |   X       |           |   X
foxes       |           |   X       |
in          |           |   X       |
jumped      |   X       |           |   X
lazy        |   X       |   X       |
leap        |           |   X       |
over        |   X       |   X       |   X
quick       |   X       |   X       |   X
summer      |           |   X       |
the         |   X       |           |   X
-------------------------------------------------------------------------------------
```

```txt
GET /my_index/_search
{
  "aggs" : {
    "popular_terms": {
      "terms" : {
        "field" : "body"
      }
    }
  }
}
```

聚合部分，则无法使用正拍索引【不容易确定原doc内容】。Doc values 通过转置两者间的关系来解决这个问题。倒排索引将词项映射到包含它们的文档，doc values（正排索引） 将文档映射到它们包含的词项

```txt
Doc      Terms
--------------------------------------------------------------------------------------
Doc_1   | brown, dog, fox, jumped, lazy, over, quick, the
Doc_2   | brown, dogs, foxes, in, lazy, leap, over, quick, summer
Doc_3   | dog, dogs, fox, jumped, over, quick, the
--------------------------------------------------------------------------------------
```

当数据被转置之后，收集到 Doc中的token信息会非常容易。获得每个文档行，获取所有的词项即可。

Doc values 不仅可以用于聚合。还包括排序，访问字段值的脚本，父子关系处理。Doc values是在不可分词的类型field中创建的。如：keyword、int、date、long等

### doc values特征总结

- doc values也是索引创建时生成的，简单来说，就是数据录入索引时创建正排索引（index-time）

- doc values也有缓存应用（内存级别， OS cache等），如果内存不足时，doc values 会写入磁盘文件

ElasticSearch大部分操作都是基于系统缓存进行的，而不是JVM。官方建议不要给JVM分配太多的内存空间，这样会导致GC开销太大。通常来说，给JVM分配的内存不超过服务器物理内存的1/4。

### fielddata

如何没有建立doc value可设置fielddata=true，这样可以反转倒排索引并加载到内存中。大量消耗资源不推荐使用。默认fielddata在query time时产生。

如果必须在text类型的field上，执行聚合分析。则由两种实现方案：

- 为text类型的field增加一个子字段，子字段类型为keyword，执行聚合分析的时候，使用自字段（推荐方案）

- 在text类型的field中，设置fielddata=true，辅助完成聚合分析。【以内存为代价】

### fielddata内存

- fielddata对内存的开销极大，可以通过参数来设置内存限制。在配置文件config/elasticsearch.yml中增加

```yaml
indices.fielddata.cache.size : 20%
```

- fielddata内存监控命令

```txt
GET _stats/fielddata?fields=*
GET _nodes/stats/indices/fielddata?fields=*
GET _nodes/stats/indices/fielddata?level=indices&fields=*
```

- circuit breaker短路器

​    如果一次聚合操作时，加载的fielddata超过了总内存容量，则会抛出内存溢出错误（OOM out of memory），这个时候可以增加短路器circuit breaker，circuit breaker会估算本次操作要加载的fielddata大小，如果超出了总内存容量，则请求短路，直接返回错误响应，不会产生OOM错误导致ES所在节点宕机或应用关闭。

```yaml
indices.breaker.fielddata.limit : 60% 	# fielddata的内存限制，默认60%
indices.breaker.request.limit: 40% 		# 执行聚合的一次请求的内存限制，默认40%
indices.breaker.total.limit: 70% 		# 综合上述两个限制，总计内存限制多少。默认无限制。
```

### fielddata的filter过滤

在创建index时，为fielddata增加filter过滤器，实现一个更加细粒度的内存控制。

min：只有聚合分析的数据在大于min的document中出现过，才会加载fielddata到内存中。

max：只有聚合分析的数据在小于max的document中出现过，才会加载fielddata到内存中。

min_segment_size： 只有segment中的document数量大于min_segment_size时，才会加载到内存中。

```txt
PUT /fieddata_filter
{
  "mappings": {
    "doc": {
      "properties": {
        "note": {
          "type": "text",
          "fielddata": true,
          "fielddata_frequency_filter": {
            "min": 0.001,
            "max": 0.1,
            "min_segment_size": 500
          }
        }
      }
    }
  }
}
```

### fielddata的预加载

fielddata是一个`query-time`生成的正排索引，如果某index中必须使用fielddata，又希望可以提升其效率，则可以使用预加载的方式来实现性能的提升。预加载fielddata会提高query过程的效率，但是降低index写入数据的效率。且始终对内存有很高的压力。不建议使用。

```txt
PUT /fieddata_filter
{
  "mappings": {
    "doc": {
      "properties": {
        "note": {
          "type": "text",
          "fielddata": true,
          "fielddata_frequency_filter": {
            "min": 0.001,
            "max": 0.1,
            "min_segment_size": 500
          },
          "eager_global_ordinals": true					//开启预加载，默认false
        }
      }
    }
  }
}
```

## 十八、JAVA API

### 创建连接客户端

```java
 @Test
public void testCreateClient() {
    //设置Cluster名称
    Settings settings =Settings.builder()
        .put("cluster.name","elasticsearch")
        .build();

    //获取transportClient用于操作文档
    TransportClient transportClient = new PreBuiltTransportClient(settings);

    //设置链接主机的ip和端口
    transportClient.addTransportAddress(
        new TransportAddress(new InetSocketAddress("192.168.1.155", 9300)));

    //获取AdminClient用于操作Cluster和indices
    AdminClient adminClient = transportClient.admin();

    //获取indicesClient用于操作索引
    IndicesAdminClient indicesClient = adminClient.indices();

    //获取ClusterClient用于操作集群
    ClusterAdminClient clusterClient = adminClient.cluster();
}
```

### 索引操作

- **创建索引**

```java
 @Test
public void testCreateIndex() throws IOException {
    //==============================创建indicesClient开始==============================
    Settings settings =Settings.builder()
        .put("cluster.name","elasticsearch")
        .build();
    /*
	 * 执行创建索引逻辑，创建创建索引的预编译请求构建器
	 * ES的操作命令是文本，是计算机硬件不可识别的
	 * ES接收到文本命令后，必须先编译成机器码，再执行命令
	 * ES客户端提供了预编译能力。客户端发送命令的时候，就是可运行的机器码
	 */
    TransportClient transportClient = new PreBuiltTransportClient(settings);
    transportClient.addTransportAddress(
        new TransportAddress(new InetSocketAddress("192.168.1.155", 9300)));
    IndicesAdminClient indicesClient = transportClient.admin().indices();
    //==============================创建indicesClient结束==============================

    CreateIndexRequestBuilder createIndexBuilder = indicesClient.prepareCreate("student");
    /*
     * 通过构建器增加settings设置。
     * PUT /student
     * {
     *  "settings" : {
     *  	"number_of_shards" : 2,
     *  	"number_of_replicas" : 1
     *  }
     * }
     */
    createIndexBuilder.setSettings(
        Settings.builder()
        .put("number_of_shards", 2)
        .put("number_of_replicas", 1)
    );
    /*
     *  通过构建器增加mapping设置
     *  PUT /student
     *  {
     *   "mappings" : {
     *       "java" : {
     *           "properties" : {
     *                "fieldName" : {
     *                     "type" : "text", 
     *                     "analyzer" : "english", 
     *                     “fielddata”: true
     *                }
     *           }
     *       }
     *   }
     *  }
     */
    XContentBuilder mapping = XContentFactory.jsonBuilder()
        .startObject()
            .startObject("properties")
                .startObject("fieldName")
                    .field("type","text")
                    .field("analyzer","english")
                    .field("fielddata", true)
                .endObject()
            .endObject()
        .endObject();

    createIndexBuilder.addMapping("java", mapping);

    //返回信息
    CreateIndexResponse response = createIndexBuilder.get();
    System.out.println("index name : " + response.index());
    System.out.println("acknowledged : " + response.isAcknowledged());
    System.out.println("shards acknowledged : " + response.isShardsAcknowledged());
}
```

- **修改索引Settings**

```java
/**
 * 更新索引settings配置
 * 在ES中，索引创建后，不能修改primary shard数量，但是可以修改replica shard 数量。
 */
@Test
public void testUpdateIndexSettings() {
    //==============================创建indicesClient开始==============================
    Settings settings =Settings.builder()
        .put("cluster.name","elasticsearch")
        .build();
    TransportClient transportClient = new PreBuiltTransportClient(settings);
    transportClient.addTransportAddress(
        new TransportAddress(new InetSocketAddress("192.168.1.155", 9300)));
    IndicesAdminClient indicesClient = transportClient.admin().indices();
    //==============================创建indicesClient结束==============================

    UpdateSettingsRequestBuilder updateSettingsBuilder = indicesClient.prepareUpdateSettings("student");

    //设置新的Settings
    updateSettingsBuilder.setSettings(
        Settings.builder()
        .put("number_of_replicas" , 0)
    );
    //发送请求
    AcknowledgedResponse response = updateSettingsBuilder.get();
    //返回信息
    System.out.println("acknowledged : " + response.isAcknowledged());
}
```

- **修改索引Mapping**

```java
/**
 * 更新索引mapping配置。
 * 此操作就是为已有的index增加新的mapping配置。并不是真正的修改。index mapping创建后，不可变。
 */
@Test
public void testUpdateIndexMappings() throws IOException {
    //==============================创建indicesClient开始==============================
    Settings settings =Settings.builder()
        .put("cluster.name","elasticsearch")
        .build();
    TransportClient transportClient = new PreBuiltTransportClient(settings);
    transportClient.addTransportAddress(
        new TransportAddress(new InetSocketAddress("192.168.1.155", 9300)));
    IndicesAdminClient indicesClient = transportClient.admin().indices();
    //==============================创建indicesClient结束==============================

    PutMappingRequestBuilder putMappingBuilder = indicesClient.preparePutMapping("student");
    putMappingBuilder.setType("java");
    /*
	 * PUT /student/_mapping/java
	 * {
	 *  "propeties" : {
	 *      "remark" : {
	 *          "type" : "text" , "analyzer" : "standard"
	 *      }
	 *  }
	 * }
     */
    XContentBuilder mapping = XContentFactory.jsonBuilder()
        .startObject()
            .startObject("properties")
                .startObject("remark")
                    .field("type", "text")
                    .field("analyzer", "standard")
                .endObject()
            .endObject()
        .endObject();
    putMappingBuilder.setSource(mapping);

    //发送请求
    AcknowledgedResponse response = putMappingBuilder.get();
    //返回信息
    System.out.println("acknowledged : " + response.isAcknowledged());
}
```

- **删除索引**

```java
@Test
public void testDeleteIndex() {
    //==============================创建indicesClient开始==============================
    Settings settings =Settings.builder()
        .put("cluster.name","elasticsearch")
        .build();
    TransportClient transportClient = new PreBuiltTransportClient(settings);
    transportClient.addTransportAddress(
        new TransportAddress(new InetSocketAddress("192.168.1.155", 9300)));
    IndicesAdminClient indicesClient = transportClient.admin().indices();
    //==============================创建indicesClient结束==============================
    DeleteIndexRequestBuilder deleteIndexBuilder = indicesClient.prepareDelete("student");
    //发送请求
    AcknowledgedResponse response = deleteIndexBuilder.get();
    //返回信息
    System.out.println("acknowledged : " + response.isAcknowledged());
}
```

### 文档操作

- **创建文档**

```java
/**
 * 新增/覆盖document数据。
 */
@Test
public void testCreateDocument() throws IOException {
    //=============================创建TransportClient开始=============================
    Settings settings = Settings.builder()
        .put("cluster.name", "elasticsearch")
        .build();
    TransportClient transportClient = new PreBuiltTransportClient(settings);
    transportClient.addTransportAddress(
        new TransportAddress(new InetSocketAddress("192.168.1.155", 9300)));
    //=============================创建TransportClient结束=============================

    IndexRequestBuilder indexBuilder = transportClient.prepareIndex("student", "java", "1");

    /*
     * 当作为全量替换使用时，可以使用外部版本实现乐观锁控制。
     * VersionType.INTERNAL - 使用内部版本号进行控制。、
     * VersionType.EXTERNAL - 使用外部版本号控制。
     */
    indexBuilder.setVersionType(VersionType.EXTERNAL);
    indexBuilder.setVersion(5);

    //如果设置create为true则必须为创建操作，文档存在则报错
    //indexBuilder.setCreate(true);
    
    /**
      * PUT /student/java/1?version_type=external&version=5
      * {
      *   "fieldName":"10001",
      *   "remark" : "number one"
      * }
      */
    XContentBuilder document = XContentFactory.jsonBuilder()
        .startObject()
        .field("fieldName","10001")
        .field("remark", "number one")
        .endObject();

    //发送请求并得到返回数据
    IndexResponse response = indexBuilder.setSource(document).get();
    System.out.println("_index : " + response.getIndex());
    System.out.println("_type : " + response.getType());
    System.out.println("_id : " + response.getId());
    System.out.println("_version : " + response.getVersion());
    System.out.println("status : " + response.status());

    //关闭客户端
    transportClient.close();
}
```

- **更新文档**

```java
 /**
   * 修改Document partial update
  */
@Test
public void testPartialUpdateDocument() throws IOException {
    //=============================创建TransportClient开始=============================
    Settings settings = Settings.builder()
        .put("cluster.name", "elasticsearch")
        .build();
    TransportClient transportClient = new PreBuiltTransportClient(settings);
    transportClient.addTransportAddress(
        new TransportAddress(new InetSocketAddress("192.168.1.155", 9300)));
    //=============================创建TransportClient结束=============================

    UpdateRequestBuilder updateBuilder = transportClient.prepareUpdate("student", "java", "1");

    
    /**
      * POST /student/java/1/_update?retry_on_conflict=3
      * {
      *   "doc":{
      *      "remark" : "number two"
      *   }
      * }
      */
    XContentBuilder document = XContentFactory.jsonBuilder()
        .startObject()
        .field("remark", "number two")
        .endObject();
    updateBuilder.setDoc(document);

    updateBuilder.setRetryOnConflict(3);    //设置retry次数
    //发送请求
    UpdateResponse response = updateBuilder.get();
    //返回数据
    System.out.println("_index : " + response.getIndex());
    System.out.println("_type : " + response.getType());
    System.out.println("_id : " + response.getId());
    System.out.println("_version : " + response.getVersion());
    System.out.println("status : " + response.status());
    transportClient.close();
}
```

- **使用脚本实现partial update**

```java
@Test
public void testUpdateWithScript(){
    //=============================创建TransportClient开始=============================
    Settings settings = Settings.builder()
        .put("cluster.name", "elasticsearch")
        .build();
    TransportClient transportClient = new PreBuiltTransportClient(settings);
    transportClient.addTransportAddress(
        new TransportAddress(new InetSocketAddress("192.168.1.155", 9300)));
    //=============================创建TransportClient结束=============================

    UpdateRequestBuilder UpdateBuilder = transportClient.prepareUpdate("student", "java", "1");
    //设置脚本
    UpdateRequestBuilder updateRequest = UpdateBuilder.setScript(new Script("ctx._source.age=\"45\""));

    //发送执行脚本请求
    UpdateResponse response = updateRequest.get();

    //返回数据
    System.out.println("_index : " + response.getIndex());
    System.out.println("_type : " + response.getType());
    System.out.println("_id : " + response.getId());
    System.out.println("_version : " + response.getVersion());
    System.out.println("status : " + response.status());

    transportClient.close();
}
```

- **插入或更新**

```java
 /*
  * upsert:如果对应id的document存在，则实现更新操作。如果对应的document不存在，则创建document。
  * 如果本次的upsert操作是新增，则忽略update操作。
  * 如果本次的upsert操作是更新，则忽略insert操作。
  */
@Test
public void testUpsertDocument() throws IOException {
    //=============================创建TransportClient开始=============================
    Settings settings = Settings.builder()
        .put("cluster.name", "elasticsearch")
        .build();
    TransportClient transportClient = new PreBuiltTransportClient(settings);
    transportClient.addTransportAddress(
        new TransportAddress(new InetSocketAddress("192.168.1.155", 9300)));
    //=============================创建TransportClient结束=============================

    UpdateRequestBuilder updateBuilder = transportClient.prepareUpdate("student", "java", "5");

    //新增文档
    XContentBuilder insertDoc = XContentFactory.jsonBuilder()
        .startObject()
        .field("name", "james gosling")
        .field("age", 45)
        .field("join_date", "2017-01-01")
        .field("remark", "java architect")
        .endObject();

    //更新文档
    XContentBuilder updateDoc = XContentFactory.jsonBuilder()
        .startObject()
        .field("name", "james gosling")
        .field("age", 45)
        .field("join_date", "2017-01-01")
        .field("remark", "java architect")
        .endObject();

    //设置upsert
    updateBuilder.setDoc(updateDoc).setUpsert(insertDoc);

    //发送请求
    UpdateResponse response = updateBuilder.get();

    //返回数据
    System.out.println("_index : " + response.getIndex());
    System.out.println("_type : " + response.getType());
    System.out.println("_id : " + response.getId());
    System.out.println("_version : " + response.getVersion());
    System.out.println("status : " + response.status());

    transportClient.close();
}
```

- **查询文档**

  - 查询单个文档

  ```java
  @Test
  public void testGetDocument(){
      //===========================创建TransportClient开始===========================
      Settings settings = Settings.builder()
          .put("cluster.name", "elasticsearch")
          .build();
      TransportClient transportClient = new PreBuiltTransportClient(settings);
      transportClient.addTransportAddress(
          new TransportAddress(new InetSocketAddress("192.168.1.155", 9300)));
      //===========================创建TransportClient结束===========================
  
      //GET /student/java/5
      GetRequestBuilder getBuilder = transportClient.prepareGet("student", "java", "5");
  
      //发送请求
      GetResponse response = getBuilder.get();
  
      //得到返回_source
      Map<String, Object> sourceAsMap = response.getSourceAsMap();
  
      //遍历_source
      for (Map.Entry<String, Object> entry : sourceAsMap.entrySet()) {
          System.out.println(entry.getKey()+" - "+entry.getValue());
      }
  }
  ```

  - 查询多个文档

  ```java
  @Test
  public void testMGetDocument(){
      //===========================创建TransportClient开始===========================
      Settings settings = Settings.builder()
          .put("cluster.name", "elasticsearch")
          .build();
      TransportClient transportClient = new PreBuiltTransportClient(settings);
      transportClient.addTransportAddress(
          new TransportAddress(new InetSocketAddress("192.168.1.155", 9300)));
      //===========================创建TransportClient结束===========================
  
      /**
        * GET /student/java/_mget
        * {
        *   "ids":["1","5"]
        * }
        */
      MultiGetRequestBuilder multiGetBuilder = transportClient.prepareMultiGet();
      multiGetBuilder.add("student", "java", "1", "5");
  
      //发送请求
      MultiGetResponse response = multiGetBuilder.get();
  
      //得到响应数组数据
      MultiGetItemResponse[] multiGetItemResponses = response.getResponses();
      //遍历响应数组
      for (MultiGetItemResponse itemRespons : multiGetItemResponses) {
          //拿到id
          System.out.println("id:" + itemRespons.getId());
          //遍历_source
          Map<String, Object> sourceAsMap = itemRespons.getResponse().getSourceAsMap();
          for (Map.Entry<String, Object> entry : sourceAsMap.entrySet()) {
              System.out.println(entry.getKey()+" - "+entry.getValue());
          }
      }
      transportClient.close();
  }
  ```

- **删除文档**

```java
@Test
public void testDeleteDocument(){
    //=============================创建TransportClient开始=============================
    Settings settings = Settings.builder()
        .put("cluster.name", "elasticsearch")
        .build();
    TransportClient transportClient = new PreBuiltTransportClient(settings);
    transportClient.addTransportAddress(
        new TransportAddress(new InetSocketAddress("192.168.1.155", 9300)));
    //=============================创建TransportClient结束=============================

    /**
     * DELETE /student/java/1
     */
    DeleteRequestBuilder deleteBuilder = transportClient.prepareDelete("student", "java", "1");

    //发送请求
    DeleteResponse response = deleteBuilder.get();

    //返回信息
    System.out.println("_index : " + response.getIndex());
    System.out.println("_type : " + response.getType());
    System.out.println("_id : " + response.getId());
    System.out.println("_version : " + response.getVersion());
    System.out.println("status : " + response.status());
}
```

- **bulk 批处理**

```java
/**
  * bulk api 批量操作
  * bulk api大多数情况都是循环加入具体的操作规则。
  * 通常一个bulk请求中只有一类批量处理。
  */
@Test
public void testBulk() throws IOException {
    //=============================创建TransportClient开始=============================
    Settings settings = Settings.builder()
        .put("cluster.name", "elasticsearch")
        .build();
    TransportClient transportClient = new PreBuiltTransportClient(settings);
    transportClient.addTransportAddress(
        new TransportAddress(new InetSocketAddress("192.168.1.155", 9300)));
    //=============================创建TransportClient结束=============================

   /**
     * PUT /_bulk
     * {
     *   { "create" : { "_index" : "student" , "_type" : "java", "_id" : "2" } }
     *   { "fieldName":"10002", "remark" : "number two"}
     *   { "delete" : { "_index" : "test_index", "_type" : "my_type", "_id" : "1" } }
     * }
     */
    //创建预编译bulk
    BulkRequestBuilder bulkBuilder = transportClient.prepareBulk();

    //创建bulk中的新增文档请求
    IndexRequestBuilder indexBuilder = transportClient.prepareIndex("student", "java","2");
    XContentBuilder inserDoc = XContentFactory.jsonBuilder()
        .startObject()
            .field("fieldName","10002")
            .field("remark", "number two")
        .endObject();
    indexBuilder.setSource(inserDoc);

    //创建bulk中的删除文档请求
    DeleteRequestBuilder deleteBuilder = transportClient.prepareDelete("student", "java", "1");

    //将请求加入bulk
    bulkBuilder.add(indexBuilder);
    bulkBuilder.add(deleteBuilder);

    BulkResponse bulkResponse = bulkBuilder.get();

    //hasFailures获取本次bulk请求的响应中，是否有错误
    if(bulkResponse.hasFailures()) {
        System.out.println("本次操作有错误情况！！！");
    }

    //遍历bulk的每项返回信息
    for (BulkItemResponse itemResponse : bulkResponse) {
        System.out.println("_index : " + itemResponse.getIndex());
        System.out.println("_type : " + itemResponse.getType());
        System.out.println("_id : " + itemResponse.getId());
        System.out.println("_version : " + itemResponse.getVersion());
        System.out.println("is failed : " + itemResponse.isFailed());
        System.out.println("failure message : " + itemResponse.getFailureMessage());
    }
}
```

### 搜索操作

```java
/**
     * GET /student/java/_search
     * {
     *   "query": {
     *     "bool": {
     *       "must": [
     *         {
     *           "match": {
     *           "remark": "java"
     *           }
     *         }
     *       ],
     *       "filter": {
     *         "range": {
     *           "age": {
     *             "gt": 20,
     *             "lt": 48
     *           }
     *         }
     *       }
     *     }
     *   },
     *   "from": 0,
     *   "size": 20
     * }
     */
@Test
public void testSearch() {
    //=============================创建TransportClient开始=============================
    Settings settings = Settings.builder()
        .put("cluster.name", "elasticsearch")
        .build();
    TransportClient transportClient = new PreBuiltTransportClient(settings);
    transportClient.addTransportAddress(
        new TransportAddress(new InetSocketAddress("192.168.1.155", 9300)));
    //=============================创建TransportClient结束=============================
    SearchRequestBuilder searchBuilder = transportClient.prepareSearch("student");
    searchBuilder.setTypes("java");

    //设置查询
    searchBuilder.setQuery(QueryBuilders.matchQuery("remark", "java"));

    //设置过滤
    searchBuilder.setPostFilter(QueryBuilders.rangeQuery("age").gt(20).lt(48));

    //设置分页
    searchBuilder.setFrom(0);
    searchBuilder.setSize(20);

    // 返回SearchResponse对象。内部的数据结构和rest api访问的结果一致。
    SearchResponse response = searchBuilder.get();

    System.out.println("took : " + response.getTook());
    System.out.println("time_out : " + response.isTimedOut());
    System.out.println("total shards : " + response.getTotalShards());

    System.out.println("hits.total : " + response.getHits().getTotalHits());
    System.out.println("hits.max_score : " + response.getHits().getMaxScore());

    SearchHit[] hits = response.getHits().getHits();

    for(int i = 0; i < hits.length; i++){
        System.out.println(hits[i].getSourceAsString());
    }
}
```

### 聚合操作

```java
/**
  * GET /student/java/_search
  * {
  *   "aggs": {
  *     "group_by_date": {
  *       "date_histogram": {
  *         "field": "join_date",
  *         "interval": "year"
  *       },
  *       "aggs": {
  *         "avg_by_age": {
  *           "avg": {
  *             "field": "age"
  *           }
  *         }
  *       }
  *     }
  *   }
  * }
  */
@Test
public void testAggregationSearch(){
    //=============================创建TransportClient开始=============================
    Settings settings = Settings.builder()
        .put("cluster.name", "elasticsearch")
        .build();
    TransportClient transportClient = new PreBuiltTransportClient(settings);
    transportClient.addTransportAddress(
        new TransportAddress(new InetSocketAddress("192.168.1.155", 9300)));
    //=============================创建TransportClient结束=============================

    SearchRequestBuilder searchBuilder = transportClient.prepareSearch("student");
    searchBuilder.setTypes("java");

    //创建子聚合操作
    AvgAggregationBuilder subAggregationBuilder = AggregationBuilders.avg("avg_by_age").field("age");

    // 增加聚合， ES的聚合可以无限嵌套，建议不超过5层。 嵌套方式为 subAggregation
    DateHistogramAggregationBuilder aggregationBuilder = AggregationBuilders.dateHistogram("group_by_date")
        .field("join_date")
        .dateHistogramInterval(DateHistogramInterval.YEAR)
        .subAggregation(subAggregationBuilder);

    searchBuilder.addAggregation(aggregationBuilder);

    //如果search请求中增加了aggregation聚合分析的话，可以使用execute方法来发起请求。 相当于是get()方法
    SearchResponse response = searchBuilder.execute().actionGet();

    Histogram groupByDateAggr = (Histogram)response.getAggregations().get("group_by_date");

    for (Histogram.Bucket bucket : groupByDateAggr.getBuckets()) {
        // 输出聚合数据的内容， 数据内容包括key, key_as_string, doc_count
        System.out.println(bucket.getKeyAsString() + " - " + bucket.getDocCount());
        // 获取子聚合项，如果聚合内容是计算结果，而不是统计值，则直接获取计算结果 - value。 如果是统计值，获取统计信息docCount
        Avg avgByAge = (Avg) bucket.getAggregations().asMap().get("avg_by_age");
        System.out.println(avgByAge.getName() + " - " + avgByAge.getValueAsString());
    }
}
```

## 十九、Logstash

### 从数据库MySQL中增量导入数据到ES

**第一步：环境准备**

- 数据库如果做增量数据导入，必须提高一个可控边界。可控边界一般使用自增主键或者时间戳

- logstash不能创建ElasticSearch中的索引，所以需要手工创建索引

- 导入mysql驱动jar包。最佳保存在$logstash_home/config目录中

**第二步：定义logstash-mysql增量导入配置文件**

$logstash_home/config创建文件logstash-mysql-es.conf

```conf
input {
  jdbc {
    # mysql相关jdbc配置
    jdbc_connection_string => "jdbc:mysql://192.168.1.117:3306/logstash?useUnicode=true&characterEncoding=utf-8&useSSL=false"
    jdbc_user => "root"
    jdbc_password => "123456"

    # jdbc连接mysql驱动的文件目录
    jdbc_driver_library => "./config/mysql-connector-java-5.1.5.jar"
    jdbc_driver_class => "com.mysql.jdbc.Driver"
    jdbc_paging_enabled => true
    jdbc_page_size => "50000"

    jdbc_default_timezone =>"Asia/Shanghai"

    # 可以直接写SQL语句在此处，使用大于等于避免丢失数据。如下：
    statement => "select * from test_logstash where update_time >= :sql_last_value"
    
    # 也可以将SQL定义在文件中，如下：
    #statement_filepath => "./config/jdbc.sql"

    # 这里类似crontab,可以定制定时操作，比如每分钟执行一次同步(分 时 天 月 年)
    schedule => "* * * * *"
    
    # 在ES6.x版本中，不需要定义type。即使定义，logstash也是自动创建索引type为doc，将此处定义的type作为document的一部分保存
    #type => "test"

    # 是否记录上次执行结果, 如果为真，将会把上次执行到的tracking_column字段的值记录下来，保存到last_run_metadata_path指定的文件中，对磁盘的存储压力太大
    #record_last_run => true

    # 是否需要记录某个column的值，如果record_last_run为真，可以自定义我们需要track的column名称，此时该参数就要为true。否则默认track的是timestamp的值
    use_column_value => true

    # 如果use_column_value为真,需配置此参数.track的数据库column名,该column必须是递增的. 一般是mysql主键
    tracking_column => "update_time"
    
    tracking_column_type => "timestamp"

    last_run_metadata_path => "./logstash_capital_bill_last_id"

    # 是否清除last_run_metadata_path的记录,如果为真那么每次都相当于从头开始查询所有的数据库记录
    clean_run => false

    #是否将字段(column)名称转小写
    lowercase_column_names => false
  }
}

output {
  elasticsearch {
    hosts => "localhost:9200"       #如果是多个ES,使用逗号分隔多个ip和端口
    index => "mysql_datas"          #索引名称 
    document_id => "%{id}"          #数据库中数据与ES中document数据关联的字段，此处代表数据库中的id字段和ES中的document的id关联
    template_overwrite => true      #是否使用模板，开启效率更高
  }

  # 这里输出调试，正式运行时可以注释掉
  stdout {
      codec => json_lines
  } 
}
```

**第三步：启动**

```shell
bin/logstash -f config/logstash-mysql-es.conf
```

