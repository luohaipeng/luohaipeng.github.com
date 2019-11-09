---
layout:     post
title:      （九）ELK技术栈之-了解Logstash
author:     luohaipeng
header-img: img/post-bg-ios9-web.jpg
catalog: true
tags:
    - elasticsearch
---
## 前言
最近这几年，大数据一直都是很热门的话题，很多人会顾名思义的觉得，大数据就是你现在手上有大量的数据。从某个层面上来看，这确实是这样的，但是所谓的大数据不仅仅只是有一堆的数据在这，更重要的是我们要如何解析这堆数据，让它变成有意义或者说有参考价值的数据。而我们知道，在我们应用运行的过程中会产生大量的日志文件，做开发的人都知道，这些日志文件有系统日志，有错误日志，有警告日志还有正常的消息日志，这些日志长年累日的堆积，就成了大数据了。然而，做运维的人都知道，这些日志是杂乱无章的存放着的，一个日志文件又密密麻麻的信息，就算是天天闲着没事干也不想打开这些日志文件来看，甚至是恨不得把这些日志文件删掉。除非等应用出现了问题后，才会去看看日志到底是什么问题。那这样的话，这些日志就成了一堆没用的大数据了。其实，这些日志如果利用的好的话，是很有价值的数据，因为我们可以对这些日志进行各种分析和各个维度的归类，然后以类似报表或图形化的方式展现这些数据，从而快速和直观的了解我们应用的各个指标信息，达到性能监控、问题预警、快速响应和处理及时的目的。
## 数据的转存
然后理想很丰满，现实很骨感，我们希望分析这些日志文件，但如何分析才是重点！不过这个问题很快就有解决方案了。我们先来思考一下，我们为什么不想看这些日志文件？那不就是因为这些日志文件存放数据非常不直观，看起来很费劲，并且在这茫茫字海中也很难找到我们想要关注的数据。那既然这样的话，我们是否可以把这些存在文件中的日志数据转存到一个可以按各个维度结构化归类的数据库中。同时该数据库必须具备高性能的全文检索能力，因为这些日志通常来说都是文档类型的数据，并且搜索我们想要关注的某些数据，这个操作是很关键的。最后，该数据库还必须支持存储海量数据，并且不会随着数据量增加而导致搜索速度下降。那么有什么样的数据库是有我以上提到的这些能力的呢？答案就是我们现在研究的elasticsearch这个搜索服务器了，elasticsearch具备的这些能力，在我们之前章节已经有了解了。所以要对这些数据进行分析，解决方案很明显，就是把这些数据转储到elasticsearch中。
## Logstash介绍
那现在我们希望把日志文件转存到elasticsearch中，首先得收集这些日志文件，然后在解析这些日志文件中的数据，把它们变成elasticsearch支持的数据格式，最后再导入到elasticsearch中，那这些操作肯定得写个自动化的程序来完成的，而市面上已经有第三方的工具专门做这些数据的收集、解析和转存了，这个工具就是我们接下来要讲的Logstash。
#### Logstash是什么
Logstash 是一个开源的数据收集引擎，它具有实时的数据传输能力，可以按照我们定制的规范来做数据的收集、解析和存储。  
所以，Logstash有3个核心组成部分，就是数据收集、数据解析和数据转存。这3个核心组成部分组成了一个类似管道的数据流，由输入端进行数据的采集，管道本身做数据的过滤和解析，输出端把过滤和解析后的数据输出到目标数据库中：  
![Logstash数据管道](https://upload-images.jianshu.io/upload_images/10574922-a5ce15932599fa66.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
####  数据收集
上面提到，我们可以使用Logstash收集这些杂乱无章的日志文件，但实际上，日志的这种数据来源只是Logstash的其中一种，如下图所示：  
![数据收集示意图](https://upload-images.jianshu.io/upload_images/10574922-8bd6852be2266a0b.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)  
通过以上示意图我们可以发现：Logstash的数据源可以有很多的来源，除了可以收集日志之外，还可以收集redis、kafka等热门分布式技术的数据，并且还可以收集实现了java的JMS规范的消息中心的数据，如：ActiveMQ等，和使用JDBC收集关系型数据库的数据，甚至是可以用shell编辑脚本把标准输出作为Logstash的数据输入。
#### 数据过滤和解析
我们通过数据输入端从各种数据源收集到的数据可能会有很多不是我们想要的，这时我们可以给Logstash定义过滤器，过滤器可以定义多个，它们依次执行，最终把我们想要的数据过滤出来，然后把这些数据解析成目标数据库，如elasticsearch能支持的数据格式。  
![数据输入输出和过滤](https://upload-images.jianshu.io/upload_images/10574922-e8bd9d75dbe6107c.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)  

#### 数据转存
在经过管道的过滤后，最终得到我们想要的数据了，此时需要把这些数据转存到我们的目标数据库上，而这个目标数据库就是我们现在学的elasticsearch了，当然，Logstash开源的数据处理引擎，肯定不止于elasticsearch，我们来看看Logstash能支持的转存数据的目标数据库有哪些：  
![数据转存示意图](https://upload-images.jianshu.io/upload_images/10574922-318d5e7d1758260e.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)  
通过上图我们可以发现，Logstash支持的数据转存数据库除了有elasticsearch外，还有solr、Redis、Kafka、RabbitMQ、MongoDB，甚至是可以存为一个普通的磁盘文件或者CSV格式的文件。
## Logstash安装
那接下来我们把Logstash安装使用起来，安装步骤如下：
#### 准备安装包
1、下载安装包  
>https://www.elastic.co/cn/downloads/logstash  

以上地址为Logstash官网下载的地址，目前最新版为6.3.2，Logstash版本和Elasticsearch版本同步更新，所以我们最好使用一致的版本，而我们Elasticsearch使用的版本是6.2.4，所以我们Logstash也需要下载6.2.4。确定好版本后，我们下载tar格式的压缩包，包的全名为：logstash-6.2.4.tar.gz。注意：Logstash需要Java 8.不支持Java 9  

2、解压安装  
`$ tar -zxvf logstash-6.2.4.tar.gz 解压安装包，得到新的目录logstash-6.2.4`  
`$ mv logstash-6.2.4 /usr/local/logstash 移动到到/usr/local目录，并重命名为logstash`
#### 测试Logstash
`$ cd /usr/local/logstash/bin 进入logstash的bin目录`  
`$ ./logstash -e 'input { stdin {} } output { stdout {} }' 执行该命令`  
*命令解析：我们在进入bin目录后，该目录有一个脚本文件叫logstash，运行该文件并传入-e参数，表示立即启动实例，并使用命令行里的配置参数启动，给的配置是数据输入端使用标准输入，数据输出端使用标准输出（标准输入和标准输出是shell编程的概念），没有配置数据过滤器。*  
看到如下打印信息说明启动成功：
```
Sending Logstash's logs to /usr/local/logstash/logs which is now configured via log4j2.properties
[2018-08-14T01:22:45,465][INFO ][logstash.modules.scaffold] Initializing module {:module_name=>"fb_apache", :directory=>"/usr/local/logstash/modules/fb_apache/configuration"}
[2018-08-14T01:22:45,518][INFO ][logstash.modules.scaffold] Initializing module {:module_name=>"netflow", :directory=>"/usr/local/logstash/modules/netflow/configuration"}
[2018-08-14T01:22:46,532][WARN ][logstash.config.source.multilocal] Ignoring the 'pipelines.yml' file because modules or command line options are specified
[2018-08-14T01:22:47,861][INFO ][logstash.runner          ] Starting Logstash {"logstash.version"=>"6.2.4"}
[2018-08-14T01:22:48,900][INFO ][logstash.agent           ] Successfully started Logstash API endpoint {:port=>9600}
[2018-08-14T01:22:53,071][INFO ][logstash.pipeline        ] Starting pipeline {:pipeline_id=>"main", "pipeline.workers"=>2, "pipeline.batch.size"=>125, "pipeline.batch.delay"=>50}
[2018-08-14T01:22:53,318][INFO ][logstash.pipeline        ] Pipeline started successfully {:pipeline_id=>"main", :thread=>"#<Thread:0x3d51afc2 run>"}
The stdin plugin is now waiting for input:
[2018-08-14T01:22:53,494][INFO ][logstash.agent           ] Pipelines running {:count=>1, :pipelines=>["main"]}
```
此时，命令窗口停留在等待输入状态，键盘键入任意字符：
`hello logstash`
下方是Logstash在接受到输入数据后，再把数据输出的效果：
```
{
      "message" => "hello logstash",
      "@version" => "1",
      "@timestamp" => 2018-08-13T17:32:01.122Z,
      "host" => "localhost.localdomain"
}
```
完成了以上步骤说明logstash已经安装上去了，只不过我们现在运行的测试例子比较简单，只是做了个标准输入和标准输出，但这已经够了，我们接着往下深入学习Logstash，肯定是可以做更复杂的数据输入和解析，并最终转存到elasticsearch中。