---
layout:     post
title:      （十四）Elasticsearch集群配置
author:     luohaipeng
header-img: img/post-bg-ios9-web.jpg
catalog: true
tags:
    - elasticsearch
---
## 前言
&emsp;&emsp;在我们es系列文章开篇介绍中，已经提到，elasticsearch是天生支持集群的，他不需要依赖其他的服务发现和注册的组件，如zookeeper这些，因为他内置了一个名字叫ZenDiscovery的模块，是elasticsearch自己实现的一套用于节点发现和选主等功能的组件，所以elasticsearch做起集群来非常简单，不需要太多额外的配置和安装额外的第三方组件。
## 数据分片
&emsp;&emsp;我们在前面的篇章中介绍过elasticsearch的一些重要的基本概念了，但是为了循序渐进的学习elasticsearch，我们并没有把所有概念都一次性列出来，而是在学习的过程中不断的补充，那现在我们需要学习elasticsearch的集群了，就需要补充一个基本概念，叫做“分片（shard）”，分片在集群中起到了非常关键的作用，那接下来，我们来看看分片是什么。  
**分片的概念和作用**
&emsp;&emsp;分片是一个底层的工作单元 ，它仅保存了全部数据中的一部分，一个分片是一个 Lucene 的实例，它本身就是一个完整的搜索引擎，我们的文档被存储和索引到分片内。
&emsp;&emsp;每个分片又有一个副本分片，副本分片就是主分片的copy，对于分布式搜索引擎来说, 分片及副本的分配将是高可用及快速搜索响应的设计核心.主分片与副本都能处理查询请求，它们的唯一区别在于只有主分片才能处理更新请求，这就有点像我们常见的“主从”的概念了。  
&emsp;&emsp;Elasticsearch 是利用分片将数据分发到集群内各处的。分片是数据的容器，文档保存在分片内，分片又被分配到集群内的各个节点里。当你的集群规模扩大或者缩小时， Elasticsearch 会自动的在各节点中迁移分片，使得数据仍然均匀分布在集群里。所以集群和分片关系非常密切，学习集群之前，需要了解分片的概念和作用。
## ES集群节点类型
&emsp;&emsp;通过前面对elasticsearch的基本概念了解到，在一个elasticsearch集群（cluster）中，有多个节点（node），并且这些节点有不同的类型：我们部署一个elasticsearch集群常用的节点类型有：主节点，数据节点，候选主节点，客户端节点。  
这些节点的类型是通过在elasticsearch主配置文件path/to/config/elasticsearch.yml文件指定的，配置属性如下：
```
node.master: true/false #该节点有机会成为master节点
node.data: true/false #该节点可以存储数据
```
&emsp;&emsp;通过以上两行配置的组合，我们可以指定出elasticsearch的节点类型。  
每种节点的作用如下：
### 1、主节点：
&emsp;&emsp;主节点负责创建索引、删除索引、分配分片、追踪集群中的节点状态等工作。Elasticsearch中的主节点的工作量相对较轻，用户的请求可以发往集群中任何一个节点，由该节点负责分发和返回结果，而不需要经过主节点转发。而主节点是由候选主节点通过ZenDiscovery机制选举出来的，所以要想成为主节点，首先要先成为候选主节点。
### 2、候选主节点
&emsp;&emsp;在elasticsearch集群初始化或者主节点宕机的情况下，由候选主节点中选举其中一个作为主节点。指定候选主节点的配置为：`node.master: true`。  
&emsp;&emsp;但是在elasticsearch集群中，会有偶尔出现这样的一种现象，就是当主节点负载压力过大，或者集中环境中的网络问题，导致其他节点与主节点通讯的时候，主节点没来的及响应，这样的话，某些节点就认为主节点宕机，重新选择新的主节点，这样的话整个集群的工作就有问题了，比如我们集群中有10个节点，其中7个候选主节点，1个候选主节点成为了主节点，这种情况是正常的情况。但是如果现在出现了我们上面所说的主节点响应不及时，导致其他某些节点认为主节点宕机而重选主节点，那就有问题了，这剩下的6个候选主节点可能有3个候选主节点去重选主节点，最后集群中就出现了两个主节点的情况，这种情况官方成为**“脑裂现象”**。  
如果避免这种问题呢？主要从以下几个方面来避免：  
（1）尽量不要让候选主节点同时作为数据节点，因为数据节点是需要承担存储和搜索的工作的，压力会很大。所以如果该节点同时作为候选主节点和数据节点，那么一旦选上它作为主节点了，这时主节点的工作压力将会非常大，出现脑裂现象的概率就增加了。  
（2）配置主节点的响应时间，在默认情况下，主节点3秒没有响应，其他节点就认为主节点宕机了，那我们可以把该时间设置的长一点，该配置是：`discovery.zen.ping_timeout: 3`  
（3）配置候选主节点最小数量，从脑裂现象出现的原因中，我们看到了，如果集群中有部分候选主节点重新选择主节点，那集群中的候选主节点就会被分成两部分，导致集群的可用节点个数和实际的节点个数不一致，那这样的话，我们就可用通过配置的方式，指定如果候选主节点个数达到某个值时，才能进行主节点选举，而该配置属性如下：`discovery.zen.minimum_master_nodes`，该属性默认值是1，官方的推荐值是(N/2)+1，其中N是候选主节点的数量，那这样的话，比如我们集群有7个候选主节点，那么通过官方推荐值，我们应该设置为4，这样的话，至少有4个候选主节点都认为需要重选主节点的情况下才进行选举。

### 3、数据节点
&emsp;&emsp;数据节点负责数据的存储和相关具体操作，比如CRUD、搜索、聚合。所以，数据节点对机器配置要求比较高，首先需要有足够的磁盘空间来存储数据，其次数据操作对系统CPU、Memory和IO的性能消耗都很大。通常随着集群的扩大，需要增加更多的数据节点来提高可用性。指定数据节点的配置：`node.data: true`。  
elasticsearch是允许一个节点既做候选主节点也做数据节点的，但是数据节点的负载较重，所以需要考虑将二者分离开，设置专用的候选主节点和数据节点，避免因数据节点负载重导致主节点不响应。
### 4、客户端节点
&emsp;&emsp;按照官方的介绍，客户端节点就是既不做候选主节点也不做数据节点的节点，只负责请求的分发、汇总等等，但是这样的工作，其实任何一个节点都可以完成，因为在elasticsearch中一个集群内的节点都可以执行任何请求，其会负责将请求转发给对应的节点进行处理。所以单独增加这样的节点更多是为了负载均衡。指定该节点的配置为：
```
node.master: false 
node.data: false 
```
![image.png](https://upload-images.jianshu.io/upload_images/10574922-04cfe1e362fa59c7.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

## 集群配置
接下来，我们做一个集群测试一下，我这里使用4个节点做测试，分别为：
```
es1   192.168.85.133:9300
es2   192,168.85.133:9500
es3   192.168.85.135:9300
es4   192.168.85.135:9500
```
**1、es1  既是候选主节点也是数据节点，elasticsearch.yml配置如下：**
```
cluster.name: elasticsearch #集群的名称，同一个集群该值必须设置成相同的
node.name: es1 #该节点的名字
node.master: true #该节点有机会成为master节点
node.data: true #该节点可以存储数据
network.bind_host: 0.0.0.0 #设置绑定的IP地址，可以是IPV4或者IPV6
network.publish_host: 192.168.85.133 #设置其他节点与该节点交互的IP地址
network.host: 192.168.85.133 #该参数用于同时设置bind_host和publish_host
transport.tcp.port: 9300 #设置节点之间交互的端口号
transport.tcp.compress: true #设置是否压缩tcp上交互传输的数据
http.port: 9200 #设置对外服务的http端口号
http.max_content_length: 100mb #设置http内容的最大大小
http.enabled: true #是否开启http服务对外提供服务
discovery.zen.minimum_master_nodes: 2 #设置这个参数来保证集群中的节点可以知道其它N个有master资格的节点。官方推荐（N/2）+1
discovery.zen.ping_timeout: 120s #设置集群中自动发现其他节点时ping连接的超时时间
discovery.zen.ping.unicast.hosts: ["192.168.85.133:9300","192.168.85.133:9500","192.168.85.135:9300"] #设置集群中的Master节点的初始列表，可以通过这些节点来自动发现其他新加入集群的节点
http.cors.enabled: true  #跨域连接相关设置
http.cors.allow-origin: "*"  #跨域连接相关设置  
```
**2、es2   既是候选主节点也是数据节点，elasticsearch.yml配置如下：**
```
cluster.name: elasticsearch #集群的名称，同一个集群该值必须设置成相同的
node.name: es2 #该节点的名字
node.master: true #该节点有机会成为master节点
node.data: true #该节点可以存储数据
network.bind_host: 0.0.0.0 #设置绑定的IP地址，可以是IPV4或者IPV6
network.publish_host: 192.168.85.133 #设置其他节点与该节点交互的IP地址
network.host: 192.168.85.133 #该参数用于同时设置bind_host和publish_host
transport.tcp.port: 9500 #设置节点之间交互的端口号
transport.tcp.compress: true #设置是否压缩tcp上交互传输的数据
http.port: 9400 #设置对外服务的http端口号
http.max_content_length: 100mb #设置http内容的最大大小
http.enabled: true #是否开启http服务对外提供服务
discovery.zen.minimum_master_nodes: 2 #设置这个参数来保证集群中的节点可以知道其它N个有master资格的节点。官方推荐（N/2）+1
discovery.zen.ping_timeout: 120s #设置集群中自动发现其他节点时ping连接的超时时间
discovery.zen.ping.unicast.hosts: ["192.168.85.133:9300","192.168.85.133:9500","192.168.85.135:9300"] #设置集群中的Master节点的初始列表，可以通过这些节点来自动发现其他新加入集群的节点
http.cors.enabled: true  #跨域连接相关设置
http.cors.allow-origin: "*"  #跨域连接相关设置  
```
**3、es3   是候选主节点，elasticsearch.yml配置如下：**
```
cluster.name: elasticsearch #集群的名称，同一个集群该值必须设置成相同的
node.name: es3 #该节点的名字
node.master: true #该节点有机会成为master节点
node.data: false #该节点可以存储数据
network.bind_host: 0.0.0.0 #设置绑定的IP地址，可以是IPV4或者IPV6
network.publish_host: 192.168.85.135 #设置其他节点与该节点交互的IP地址
network.host: 192.168.85.135 #该参数用于同时设置bind_host和publish_host
transport.tcp.port: 9300 #设置节点之间交互的端口号
transport.tcp.compress: true #设置是否压缩tcp上交互传输的数据
http.port: 9200 #设置对外服务的http端口号
http.max_content_length: 100mb #设置http内容的最大大小
http.enabled: true #是否开启http服务对外提供服务
discovery.zen.minimum_master_nodes: 2 #设置这个参数来保证集群中的节点可以知道其它N个有master资格的节点。官方推荐（N/2）+1
discovery.zen.ping_timeout: 120s #设置集群中自动发现其他节点时ping连接的超时时间
discovery.zen.ping.unicast.hosts: ["192.168.85.133:9300","192.168.85.133:9500","192.168.85.135:9300"] #设置集群中的Master节点的初始列表，可以通过这些节点来自动发现其他新加入集群的节点
http.cors.enabled: true  #跨域连接相关设置
http.cors.allow-origin: "*"  #跨域连接相关设置  
```
**4、es4   是数据节点，elasticsearch.yml配置如下：**
```
cluster.name: elasticsearch #集群的名称，同一个集群该值必须设置成相同的
node.name: es4 #该节点的名字
node.master: false #该节点有机会成为master节点
node.data: true #该节点可以存储数据
network.bind_host: 0.0.0.0 #设置绑定的IP地址，可以是IPV4或者IPV6
network.publish_host: 192.168.85.135 #设置其他节点与该节点交互的IP地址
network.host: 192.168.85.135 #该参数用于同时设置bind_host和publish_host
transport.tcp.port: 9500 #设置节点之间交互的端口号
transport.tcp.compress: true #设置是否压缩tcp上交互传输的数据
http.port: 9400 #设置对外服务的http端口号
http.max_content_length: 100mb #设置http内容的最大大小
http.enabled: true #是否开启http服务对外提供服务
discovery.zen.minimum_master_nodes: 2 #设置这个参数来保证集群中的节点可以知道其它N个有master资格的节点。官方推荐（N/2）+1
discovery.zen.ping_timeout: 120s #设置集群中自动发现其他节点时ping连接的超时时间
discovery.zen.ping.unicast.hosts: ["192.168.85.133:9300","192.168.85.133:9500","192.168.85.135:9300"] #设置集群中的Master节点的初始列表，可以通过这些节点来自动发现其他新加入集群的节点
http.cors.enabled: true  #跨域连接相关设置
http.cors.allow-origin: "*"  #跨域连接相关设置  
```
&emsp;&emsp;以上配置就是我们集群中4个elasticsearch节点的配置，我们有两个既是候选主节点也是数据节点，一个候选主节点，一个数据节点，没有配置客户端节点，因为客户端节点仅起到负载均衡的作用。

## 验证群状态
**查看集群状态的REST API**  
我们逐一的把集群中的节点启动起来，然后使用REST API查看集群状态，访问集群中任意一个节点即可:  
`GET ip:port/_cluster/health`  
响应的结果：
```
{
  "cluster_name" : "elasticsearch",
  "status" : "green",
  "timed_out" : false,
  "number_of_nodes" : 4,
  "number_of_data_nodes" : 3,
  "active_primary_shards" : 46,
  "active_shards" : 92,
  "relocating_shards" : 0,
  "initializing_shards" : 0,
  "unassigned_shards" : 0,
  "delayed_unassigned_shards" : 0,
  "number_of_pending_tasks" : 0,
  "number_of_in_flight_fetch" : 0,
  "task_max_waiting_in_queue_millis" : 0,
  "active_shards_percent_as_number" : 100.0
}
```
我们来看看返回结果中的几个关键的属性：  
- 集群状态有3种（status）：  
red，表示有主分片没有分配，某些数据不可用。  
yellow，表示主分片都已分配，数据都可用，但是有副本分片没有分配。  
green，表示主分片和副本分片都已分配，一切正常。  
那么通过以上返回的结果中的status属性，我们可以知道，当前的elasticsearch集群是正常的。  
- number_of_nodes属性说明我们集群中有4个节点。  
- number_of_data_nodes说明我们集群中有3个为数据节点，所有数据分片储存在这3个数据节点中。  
- active_primary_shards属性说明主分片有46个（我们有10个索引，其中9个索引有5个主分片，1个索引有1个主分片，合计46个主分片）  
- active_shards属性说明所有分片（主分片 + 副本分片）总共是92个  

**使用head插件管理ES集群**  
&emsp;&emsp;我们在之前安装了es-head插件的时候提到，es-head插件可以管理我们elasticsearch集群的，那现在我们就使用之前安装好的es-head插件，来图形化界面的方式管理elasticsearch集群。  
打开首页我们可以看到以下界面：  
![image.png](https://upload-images.jianshu.io/upload_images/10574922-6df273dae3d9d0f2.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
&emsp;&emsp;连接任意一个节点看到的结果都是一样的，这个就验证了我们上面提到的概念：用户的请求可以发往集群中任何一个节点，由该节点负责请求分发和返回结果。 
- 在该界面中，是以表格的形式展示数据的，纵向的是集群中的索引，横向的是集群中的节点。  
- 横向有4行，分别是4个我们自己命名的节点，其中标记了es1是星号，代表它为主节点。  
- 然后我们可以看到，每个单元格中都有一些绿色的小方块，小方块中标有数字，这些绿色小方块就代表分片，其中边框加粗的是主分片，没加粗的是副本分片。  
- 我们在创建索引时，如果没有指定分片的数量，那么默认是5个，再加上每个分片都有一个副本，所以加起来就有10个绿色小方块了。  
- 每个绿色小方块上都有0-4的数字，分别代表0-4号分片，和对应的0-4号分片的副本分片，我们点击绿色小方块，会弹出一个窗口，列出该分片的一些信息，其中有条信息是“primary”，如果为true，就代表主分片，为false就代表副本分片。  
- 其中es3节点是没有给它分配分片的，那是因为当时我们给该节点设置的是候选主节点，不是数据节点，所以他不承担数据存储的工作。  
- 这10个分片（包含主分片和副本分片）被elasticsearch分配到各个数据节点中，并且根据我们上面对分片概念的了解，主分片和副本分片都可以接收数据查询请求，但是只有主分片才能接收数据修改请求，副本分片的数据是从主分片同步过来的。  
**测试数据**  
&emsp;&emsp;最后，我们从用户的角度上来看看，向集群中各个节点发送一个查询请求，测试数据的一致性。  
我们先往集群中新增一条数据，以store索引作为测试，向任意一个节点发送一个新增文档的请求：
```
PUT /store/employee/3
{
    "name":"王五",
    "age":28,
    "about":"我叫王五，但不是隔壁老王",
    "interests":["code","music"]
}
```
看到以下返回结果提示添加成功：  
![image.png](https://upload-images.jianshu.io/upload_images/10574922-df29c2323e205ff4.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
&emsp;&emsp;但这里有一个问题，就是该文档会被存储到哪一个主分片中呢？首先这肯定不会是随机的，否则将来要获取文档的时候我们就不知道从何处寻找了。实际上，这个过程是根据下面这个公式决定的：
```
shard = hash(routing) % number_of_primary_shards
```
`routing` 是一个可变值，默认是文档的 `_id` ，也可以设置成一个自定义的值。 `routing` 通过 hash 函数生成一个数字，然后这个数字再除以 `number_of_primary_shards` （主分片的数量）后得到 **余数** 。这个分布在 `0` 到 `number_of_primary_shards-1` 之间的余数，就是我们所寻求的文档所在分片的位置。  

这就顺便解释了为什么我们要在创建索引的时候就确定好主分片的数量，并且永远不会改变这个数量，因为如果数量变化了，那么所有之前路由的值都会无效，文档也再也找不到了。  

那接下来我们看看每个节点是否可以查询到该数据:  
![image.png](https://upload-images.jianshu.io/upload_images/10574922-6380194159404cca.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

![image.png](https://upload-images.jianshu.io/upload_images/10574922-76aa659f8a185997.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

![image.png](https://upload-images.jianshu.io/upload_images/10574922-f3700e81e035ecdd.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
![image.png](https://upload-images.jianshu.io/upload_images/10574922-3e4e24bdd6d9a885.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
&emsp;&emsp;这4个节点都是可以查询到刚刚我们新增的文档id为3的数据，集群正常运作。在我们整个集群配置的过程中，需要我们配置的东西很少，所以elasticsearch真的可以很轻松的完成一个规模超大的集群，并且在该集群中存储海量的数据。





