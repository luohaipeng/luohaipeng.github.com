---
layout:     post
title:      （二）Elasticsearch安装和配置
author:     luohaipeng
header-img: img/post-bg-ios9-web.jpg
catalog: true
tags:
    - elasticsearch
---
## 安装使用
&#160; &#160; &#160; &#160;elasticsearch是支持Linux和window系统的，而我当前的系统是Linux，发行版为centos7，我们就在centos7下做elasticsearch的安装和测试，并且后续的环境都是在centos7下学习，所以这里默认大家对Linux有一定的了解，并且熟悉常用的命令，如果大家对Linux不了解的话，欢迎来到叩丁狼学习，或者关注我们的技术文章和公开视频。  
**安装步骤：**  
**1、下载elasticsearch：**  
elasticsearch官网上最新的稳定版本是6,2,4，该版本不支持jdk8以下的，所以我们在当前的centos7系统下安装好jdk8  
elasticsearch下载地址：http://www.elastic.co/downloads/elasticsearch  
我们选择下载的是tar包。包的全名为：elasticsearch-6.2.4.tar.gz  

**2、安装elasticsearch**  
把elasticsearch-6.2.4.tar.gz上传到/usr/local目录下并执行解压命令：  
``$ tar -zxvf elasticsearch-6.2.4.tar.gz``  
为了方便后续的操作，我们把 elasticsearch-6.2.4改个名字  
``$ mv elasticsearch-6.2.4 elasticsearch``  

**3、运行elasticsearch**  
从5.0开始，ElasticSearch 安全级别提高了，不允许采用root帐号启动，所以我们要添加一个用户。  
创建用户组：  
``$ groupadd es``  
创建用户，-e代表把es用户分配到es用户组中，-p代表给es用户设置密码为123456：  
``$ useradd es -g es -p 123456``  
修改权限，更改elasticsearch文件夹以及内部文件的所属用户以及组为es，-R表示逐级（N层目录）  
``$ chown -R es:es /usr/local/elasticsearch``  
切换为es用户登录  
``$ su es``  
进入到elasticsearch安装目录的bin目录下  
``$ cd /usr/local/elasticsearch/bin``  
执行运行操作，-d表示后台运行  
``$ ./elasticsearch -d``  
查看是否运行成功：  
``$ curl http://localhost:9200``  
如果打印以下信息，则代表elasticsearch运行成功  
```
{
  "name" : "p2gU_GO",
  "cluster_name" : "elasticsearch",
  "cluster_uuid" : "DoHcTzHrSEqNIQVltUG2XA",
  "version" : {
    "number" : "6.2.4",
    "build_hash" : "ccec39f",
    "build_date" : "2018-04-12T20:37:28.497551Z",
    "build_snapshot" : false,
    "lucene_version" : "7.2.1",
    "minimum_wire_compatibility_version" : "5.6.0",
    "minimum_index_compatibility_version" : "5.0.0"
  },
  "tagline" : "You Know, for Search"
}
```
&#160; &#160; &#160; &#160;至此，我们就把elasticsearch安装到我们主机上了，但是我们现在是在本机上访问elasticsearch服务：localhost:9200，那如果想要外部的主机访问呢？比如我们当前运行这个elasticsearch服务的主机IP是192.168.85.131，然后我用192.168.85.1这台主机去访问192.168.85.131:9200，那是访问不了的，原因是elasticsearch本身的安全机制和系统防火墙等因素限制。那实际上，我们的elasticsearch服务肯定是要给外部主机访问的，那该怎么做呢？接下来我们就来学学elasticsearch的配置。  
## 基础配置
&#160; &#160; &#160; &#160;elasticsearch的主配置文件在config/elasticsearch.yml中，该配置文件包含集群、节点、网络和数据存储等等的重要配置。接下来我们来看看该配置文件中的一些基础的配置属性是代表什么意思，更多的配置我们后续讲到相关的内容再详细介绍。  
```
################################### 集群相关配置 ################################### 
# cluster.name可以确定你的集群名称,当你的elasticsearch集群在同一个网段中elasticsearch会自动的找到具有相同cluster.name的elasticsearch服务. 
# 所以当同一个网段具有多个elasticsearch集群时cluster.name就成为同一个集群的标识.，可以手动指定也可以自动生成
# cluster.name: elasticsearch 

#################################### 节点相关配置 ##################################### 
# 节点名称同理,可自动生成也可手动配置. 
# node.name: node-1

# 允许一个节点是否可以成为一个master节点,es是默认集群中的第一台机器为master,如果这台机器停止就会重新选举master. 
# node.master: true 

# 允许该节点存储数据(默认开启) 
# node.data: true 

# 默认情况下，多个节点可以在同一个安装路径启动，如果你想让你的es只启动一个节点，可以进行如下设置
# node.max_local_storage_nodes: 1 

#################################### 索引相关配置 #################################### 
# 设置索引的分片数,默认为5 
#index.number_of_shards: 5 

# 设置索引的副本数,默认为1: 
#index.number_of_replicas: 1 

# 配置文件中提到的最佳实践是,如果服务器够多,可以将分片提高,尽量将数据平均分布到大集群中去
# 同时,如果增加副本数量可以有效的提高搜索性能 
# 需要注意的是,"number_of_shards" 是索引创建后一次生成的,后续不可更改设置 
# "number_of_replicas" 是可以通过API去实时修改设置的 

#################################### 路径相关配置 #################################### 
# 配置文件存储位置 
# path.conf: /path/to/config 

# 数据存储位置(单个目录设置) 
# path.data: /path/to/data 
# 多个数据存储位置,有利于性能提升 
# path.data: /path/to/data1,/path/to/data2 

# 临时文件的路径 
# path.work: /path/to/work 

# 日志文件的路径 
# path.logs: /path/to/logs 

# 插件安装路径 
# path.plugins: /path/to/plugins 


################################### 内存相关配置 #################################### 
# 当JVM开始写入交换空间时（swapping）ElasticSearch性能会低下,你应该保证它不会写入交换空间 
# 设置这个属性为true来锁定内存,同时也要允许elasticsearch的进程可以锁住内存,linux下可以通过 `ulimit -l unlimited` 命令 
# bootstrap.mlockall: true 
# 确保 ES_MIN_MEM 和 ES_MAX_MEM 环境变量设置为相同的值,以及机器有足够的内存分配给Elasticsearch 
# 注意:内存也不是越大越好,一般64位机器,最大分配内存别才超过32G 

############################## 网络相关配置 ############################### 
# 设置绑定的ip地址,可以是ipv4或ipv6的，默认绑定本机ip
# network.bind_host: 192.168.0.1  

# 设置其它节点和该节点交互的ip地址,如果不设置它会自动设置,值必须是个真实的ip地址 
# network.publish_host: 192.168.0.1 

# 同时设置bind_host和publish_host上面两个参数 
# network.host: 192.168.0.1    #绑定监听IP

# 设置节点间交互的tcp端口,默认是9300 
# transport.tcp.port: 9300 

# 设置是否压缩tcp传输时的数据，默认为false,不压缩
# transport.tcp.compress: true 

# 设置对外服务的http端口,默认为9200 
# http.port: 9200 

# 设置请求内容的最大容量,默认100mb 
# http.max_content_length: 100mb 

# 使用http协议对外提供服务,默认为true,开启 
# http.enabled: false 
```
&#160; &#160; &#160; &#160;以上就是elasticsearch的基础配置，由于elasticsearch每个版本的config/elasticsearch.ym文件内容都不同，所以可能我以上写的这些配置在大家那里看到又是不一样的。如果某个属性在某个elasticsearch的版本中不存在，那么直接添加这个属性上去测试一下效果，他们的属性大多数都是可以通用的。

## 取消主机限制
&#160; &#160; &#160; &#160;接下来我们具体来配置一下，让elasticsearch服务能够给外部IP访问：  
&#160; &#160; &#160; &#160;首先我们通过刚刚的基础配置发现，网络相关配置中有3个配置项是跟IP有关的：network.bind_host、network.publish_host和network.host，他们的用法分别是：  
**network.bind_host：**  
&#160; &#160; &#160; &#160;在默认情况下elasticsearch服务只能本机访问，如果外部主机需要访问该elasticsearch服务，就需要把外部的这台主机的IP配置到该属性中。那如果希望elasticsearch服务不限制所有主机的访问，那么该属性可以设置为0.0.0.0。  

**network.publish_host：**  
&#160; &#160; &#160; &#160;在默认情况下elasticsearch的节点不能跨主机交互，如果需要，则在该属性配置elasticsearch节点服务所在的IP。那如果希望elasticsearch服务不限制所有主机的节点交互，那么该属性可以配置为0.0.0.0。  

**network.host：**  
&#160; &#160; &#160; &#160;同时设置bind_host和publish_host两个参数，那我们只需要把该属性设置为0.0.0.0，那么就不限制主机的访问和节点的交互  

那么我们就来修改elasticsearch的配置文件：  
``$ vi config/elasticsearch.yml``  
找到network.host这行配置，把注释解开，值设置为0.0.0.0  
``$ network.host: 0.0.0.0``  
退出并保存elasticsearch.yml文件，然后重启elasticsearch服务  
关闭elasticsearch服务：  
``使用强制杀进程的方式关闭elasticsearch``  
启动elasticsearch服务：  
``$ ./bin/elasticsearch  #注意不要加上-d参数，因为接下来可能会报错，在当前控制台上查看错误信息比较方便``  
如果启动出现以下错误：  
```
ERROR: bootstrap checks failed
max virtual memory areas vm.max_map_count [65530] is too low, increase to at least [262144]
```
则执行以下操作：  
1、切换到root用户  
``$ su root``  
2、修改配置sysctl.conf：  
``$ vi /etc/sysctl.conf ``  
``添加下面配置：vm.max_map_count=655360``  
3、修改配置limits.conf  
`$ vi /etc/security/limits.conf`  
```
添加以下内容
* soft nofile 65536
* hard nofile 131072
* soft nproc 2048
* hard nproc 4096
```
4、退出并保存，然后执行命令：  
``$ sysctl -p``  
最后，重新启动elasticsearch，即可启动成功。  

&#160; &#160; &#160; &#160;至此，我们已经把elasticsearch的服务取消的主机访问的限制了，但是我们还是不能通过外部的主机访问elasticsearch服务，因为我们系统的防火墙是开启的，我们需要把elasticsearch服务的网络端口放开，这样才能让外界访问。我们通过地址访问elasticsearch服务：http://IP:9200可以知道它的http网络端口为9200，并且我们刚刚学习的配置我们发现，网络相关配置中有个http.port属性，该属性是配置elasticsearch的http端口的，默认端口为9200，所以我们需要把9200端口放开：  
``$ firewall-cmd --zone=public --add-port=9200/tcp --permanent   #--permanent永久生效，没有此参数重启后失效``  
重新加载防火墙：  
``$ firewall-cmd --reload``  
&#160; &#160; &#160; &#160;那么这些准备工作已经完成了，接下来的章节我们就要来具体的进入Elasticsearch的使用了。