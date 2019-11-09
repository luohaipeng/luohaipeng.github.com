---
layout:     post
title:      （十一）ELK技术栈之-Logstash实战案例
author:     luohaipeng
header-img: img/post-bg-ios9-web.jpg
catalog: true
tags:
    - elasticsearch
---
## 案例一：收集Nginx访问日志数据

Nginx是企业中常用的http服务器，也有人称它为反向代理服务器和负债均衡服务器，凭借着性能强大和功能完善在整个互联网行业中大量使用，所以收集Nginx服务器的访问日志数据，并且转存到elasticsearch中，就是一个很常见的需求了，然后再根据elasticsearch强大的搜索和聚合能力统计出我们的应用的访问量、访问用户所在地、访问了什么功能、访问时间和使用了什么客户端访问的等等信息。
#### 创建配置文件
首先，我们创建一个Logstash的配置文件，文件名随意，我们起名为：logstash.conf，在该文件中编辑input、filter和output这3个组件的配置。  
然后，在启动Logstash实例的时候，使用`-f`参数的方式启动，参数值为该配置文件的存放路径，比如：`./logstash -f /soft/logstash_conf/logstash.conf`

#### input配置
```
input {
  file {
    path => "/usr/local/nginx/logs/access.log"
    type => "nginx-access"
    start_position => "beginning"
  }
}
```
我们使用了一个file的输入插件，读取的文件是nginx的访问日志`access.log`，存放在：`/usr/local/nginx/logs/`目录中。nginx访问日志默认格式为：
```
 log_format  main  '$remote_addr - $remote_user [$time_local] "$request" '
                   '$status $body_bytes_sent "$http_referer" '
                   '"$http_user_agent" "$http_x_forwarded_for"';
```
如果需要自定义访问日志的格式，只需要在nginx.conf配置文件中修改以上内容。  
通过以上格式输出的日志会写到`/usr/local/nginx/logs/access.log`文件中，内容为：  
`
192.168.85.1 - - [16/Sep/2018:00:42:38 +0800] "GET /catalog/getChildCatalog?catalogId=1 HTTP/1.1" 200 3249 "http://mgrsite.shop.com/" "Mozilla/5.0 (Windows NT 10.0; WOW64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/68.0.3440.106 Safari/537.36
`  
一旦nginx的访问日志有新的内容更新，会被Logstash监控到，并读取到Logstash中，然后Logstash为该数据流创建一个Event对象，我们可以在output组件中设置一个标准输出：stdout，在显示器打印Event对象，打印结果为：
```
{
      "message" => "192.168.85.1 - - [16/Sep/2018:00:42:38 +0800] "GET /catalog/getChildCatalog?catalogId=1 HTTP/1.1" 200 3249 "http://mgrsite.shop.com/" "Mozilla/5.0 (Windows NT 10.0; WOW64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/68.0.3440.106 Safari/537.36",
      "@version" => "1",
      "@timestamp" => 2018-08-13T17:32:01.122Z,
      "host" => "localhost.localdomain"
}
```
上面的打印结果就是一个Event对象，在该Event对象中的message字段封装了一条数据流的数据（注意：在上一节中，我们提到了file输入插件默认是以换行符作为一条数据流的，可以通过配置来设置数据流的分隔符）
#### filter配置
```
filter {
    grok {
        match => {
            "message" => "%{IP:remote_ip} \- \- \[%{HTTPDATE:timestamp}\] \"%{WORD:method} %{URIPATHPARAM:request} HTTP/%{NUMBER:http_version}\" %{NUMBER:status} %{NUMBER:bytes} \"%{URI:domain}\" \"%{GREEDYDATA:user_agent}"
        }
        remove_field => ["message","@timestamp","@version"]
    }
    date {
        match => ["timestamp","dd/MMM/YYYY:HH:mm:ss Z"]
        target => "timestamp"
    }
    if [remote_ip] !~ "^127\.|^192\.168\.|^172\.1[6-9]\.|^172\.2[0-9]\.|^172\.3[01]\.|^10\." {
        geoip {
             source => "remote_ip"
        }
    }
}
```
在Logstash过滤的组件中，我们使用了3个过滤插件，分别是grok、date和geoip，该3个插件从上外下进行3层的数据过滤和转化。  
我们按顺序一个一个来看看这些过滤插件的配置：  
**（1）、grok插件**  
在grok插件中，我们拿到了Event对象中的message字段，该message字段封装了nginx访问日志文件最新追加的一条数据，并且使用match匹配这条日志数据，该匹配的正则为（grok正则匹配语法在上一章提到，如果忘了可以回顾上一章）：  
`
%{IP:remote_ip} \- \- \[%{HTTPDATE:timestamp}\] \"%{WORD:method} %{URIPATHPARAM:request} HTTP/%{NUMBER:http_version}\" %{NUMBER:status} %{NUMBER:bytes} \"%{URI:domain}\" \"%{GREEDYDATA:user_agent}
`  
那通过了该grok插件过滤后，就会在原来的Event对象中新增remote_ip、timestamp、method、request、http_version、status、bytes、domain和user_agent这几个字段，字段的值是由message的值匹配过来的。与此同时，我们还删除了一些不需要的字段，比如："message","@timestamp","@version"。  
**（2）、date插件**  
我们在grok过滤插件匹配到message中的时间后，使用一个新的字段存储，该字段为timestamp，但是该时间的格式是不太直观，是`dd/MMM/YYYY:HH:mm:ss Z`这样的格式，打印出来的数据就是`16/Sep/2018:00:42:38 +0800`。我们需要把这种格式的时间转成比较直观的，比如：`YYYY-MM-dd HH:mm:ss`,所以，我们在date插件中配置了`match => ["timestamp","dd/MMM/YYYY:HH:mm:ss Z"]`，意思是匹配到timestamp字段中的时间格式为dd/MMM/YYYY:HH:mm:ss Z，到时候，date插件会把该匹配到的时间转成YYYY-MM-dd HH:mm:ss格式，并把转换好的时机重新覆盖到timestamp字段中，也就是我们`target => "timestamp"`这行配置的体现，如果没有这个配置，那么转换后的时间默认覆盖到"@timestamp"，但是该字段已经被我们移除了。  
**（3）、geoip插件**  
我们在grok过滤插件匹配到message中的访问IP后，使用一个新的字段存储，该字段为remote_ip，此时，我们如果有需要通过IP地址获取归属地的话，就需要使用到geoip插件了，但是我们在使用该插件前用了一个逻辑判断，排除了127、192、168和172这些内网IP，如果是以这些内网IP访问的话，就不使用geoip插件了（该判断的语法在上一章有提，如果忘了可以回顾上一章），因为内网IP是么有归属地的。如果获取到了归属地等信息后，geoip会在Event对象中新增一个字段，字段名叫：geoip，然后该字段对应的值又是一个对象，那么在默认情况下，geoip对象包含十多个字段，具体可回顾上一章geoip插件的详解。  
综合上述的3个过滤插件后，最终得到的Event为：  
```
{
         "request" => "/property/get/1",
          "method" => "GET",
            "type" => "nginx-access",
          "status" => "200",
    "http_version" => "1.1",
      "user_agent" => "Mozilla/5.0 (Windows NT 10.0; WOW64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/68.0.3440.106 Safari/537.36\" \"-\"",
            "path" => "/usr/local/nginx/logs/access.log",
       "timestamp" => 2018-09-20T14:17:48.000Z,
            "host" => "bogon",
           "bytes" => "503",
       "remote_ip" => "120.77.150.83",
          "domain" => "http://mgrsite.shop.com/",
           "geoip" => {
                    "ip" => "120.77.150.83",
        "continent_code" => "AS",
             "city_name" => "Hangzhou",
           "region_name" => "Zhejiang",
             "longitude" => 120.1614,
           "region_code" => "33",
         "country_code2" => "CN",
         "country_code3" => "CN",
              "latitude" => 30.2936,
              "timezone" => "Asia/Shanghai",
          "country_name" => "China",
              "location" => {
                         "lat" => 30.2936,
                         "lon" => 120.1614
              }
         }
}
```
#### output配置
```
output {
    elasticsearch {
        hosts => ["127.0.0.1:9200"]
        index => "logstash-%{type}"
        document_type => "%{type}"
    }
}
```
以上配置是在output组件加上一个elasticsearch 插件，把一个Event对象写入到elasticsearch中，完成nginx访问日志的收集。  
最后，我们把上述的配置放到一起查看，这样更直观的看到整个`access_log.conf`配置文件的完整内容：
```
input {
  file {
    path => "/usr/local/nginx/logs/access.log"
    type => "nginx-access"
    start_position => "beginning"
  }
}
filter {
    grok {
        match => {
            "message" => "%{IP:remote_ip} \- \- \[%{HTTPDATE:timestamp}\] \"%{WORD:method} %{URIPATHPARAM:request} HTTP/%{NUMBER:http_version}\" %{NUMBER:status} %{NUMBER:bytes} \"%{URI:domain}\" \"%{GREEDYDATA:user_agent}"
        }
        remove_field => ["message","@timestamp","@version"]
    }
    date {
        match => ["timestamp","dd/MMM/YYYY:HH:mm:ss Z"]
        target => "timestamp"
    }
    if [remote_ip] !~ "^127\.|^192\.168\.|^172\.1[6-9]\.|^172\.2[0-9]\.|^172\.3[01]\.|^10\." {
        geoip {
             source => "remote_ip"
        }
    }
}
output {
    elasticsearch {
        hosts => ["127.0.0.1:9200"]
        index => "logstash-%{type}"
        document_type => "%{type}"
    }
    stdout {
    }
}
```
#### 验证数据
我们可以先借助head界面来查看elasticsearch转存过来的nginx访问日志数据（后面需要把head升级为kibana），能看到以下截图的数据，说明配置没有问题，否则再检查一下配置文件。  
![日志数据](https://upload-images.jianshu.io/upload_images/10574922-77b903ded3ef9fb1.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

## 案例二：MySQL数据全量&增量导入到ES
在企业中往往会有这样的情况，原本某张表的数据不需要做全文搜索，现在需要做了，但是我们知道关系型数据库不适合做全文搜索，那么我们就需要想办法把关系数据库中的数据转移到elasticsearch中做全文搜索了。  
还有一种情况，某张关系型数据库的表，原本只需要做简单的like模糊查询就可以了，但是随着该表的数据量日益增加，那么在大数据量下，like查询效率非常低，影响用户使用体验，更严重的是，由于处理一个请求效率低，那可能导致大量的请求阻塞堆积，服务器压力过大，最终导致应用宕机。那为了避免这样的问题发生，我们就需要想办法把大数据表转移到elasticsearch，根据elasticsearch特点，它的搜索速度极快，并且不会因为数据量的增加而导致搜索效率的下降。
#### 创建配置文件
上面的案例我们已经创建了一个Logstash的配置文件了：logstash.conf，所以我们就直接使用logstash.conf文件，不再创建一个新的配置文件来做这个案例，这样就方便我们学习和测试阶段。但是大家需要知道，如果这样的话，但时候我们选择logstash.conf来启动Logstash实例时，该实例就会做两件事情，收集nginx访问日志和导入mysql数据，这样对该Logstash的压力是比较大的，所以，在实际应用中，更建议一个Logstash实例就做一件事情，比如创建两个配置文件：nginx_access.conf和mysql.conf，分别代表两个Logstash实例。
#### input配置
```
input {
    file {
        path => "/usr/local/nginx/logs/access.log"
        type => "nginx-access"
        start_position => "beginning"
    }
    jdbc{
        type => "product"
        jdbc_driver_library => "/soft/logstash_config/mysql-connector-java-5.1.46.jar"
        jdbc_driver_class => "com.mysql.jdbc.Driver"
        jdbc_connection_string => "jdbc:mysql://192.168.85.1:3306/wolfcode_shop_goods"
        jdbc_user => "root"
        jdbc_password => "root"
        statement => "select * from product where id > :sql_last_value"
        jdbc_paging_enabled => true
        jdbc_page_size => 2
        use_column_value => true
        tracking_column => "id"
        last_run_metadata_path => "/soft/logstash_config/mysql_record_info"
        schedule => "* * * * *"
    }
}
```
我们在原来的input组件上，在新增了一个jdbc插件，该插件使用jdbc接口读取所有实现了jdbc标准的关系型数据库的数据，而我们这次的案例使用的关系型数据库是mysql。  
以上配置就是jdbc输入插件的配置，使用该配置和elasticsearch输出插件配合使用，能完成mysql全量和增量的数据导入。其中jdbc_paging_enabled、use_column_value、tracking_column和last_run_metadata_path这4个属性配置是实现增量导入的关键，如果没有配置这4个属性，则只能实现全量导入。  
**mysql数据全量&增量导入的过程**  
我们来看看logstash是如果实现mysql数据全量&增量导入的：  
- 1、首先，因为我们配置了`use_column_value => true`和`tracking_column => "id"`，意思是logstash在程序内存中可以使用查询出来的`product`表中`id`这个列。
- 2、然后，我们配置`schedule => "* * * * *"`代表每分钟执行一次读取数据的任务，而`last_run_metadata_path => "/soft/logstash_config/mysql_record_info"`这行配置意思是：把上一次读取product表数据时最后一条数据的值，记录在`mysql_record_info`文件中，因为我们`tracking_column`配置的是id这一列，所以该`mysql_record_info`文件记录的就是上一次读取数据时，最后一条数据的id值。
- 3、这时，我们启动Logstash实例，会把`mysql_record_info`文件记录的值赋值给程序的`sql_last_value`变量，那所以，我们每次读取数据发送的SQL查询语句：`statement => "select * from product where id > :sql_last_value"`，意思就是只查询上一次记录的id值之后的数据。
- 4、如果logstash在读取`mysql_record_info`文件的时候，发现该文件没有记录内容，那说明之前没有读取过数据，这时logstash就把`sql_last_value`变量设置为0。
- 5、到达定时任务的时间，从`sql_last_value`变量记录的值开始读取数据。logstash会发送一条count语句统计product表共有多少条数据，然后在根据我们设置的`jdbc_page_size => 2（默认是10万条）`每页查询2条数据，计算出需要发送多少条select语句才能把product表剩余的数据读取出来（因为`sql_last_value`变量的值为0，所以这时相当于全量查询），并且把最后一条数据的id重新赋值给`sql_last_value`变量，同时，把该值重新写在`mysql_record_info`文件中，供以后重启Logstash实例时使用，重启实例又进行第3个步骤。以下语句是logstash执行全量查询时发送的SQL语句：
```
 SELECT count(*) AS `count` FROM (select * from product where id > 0) AS `t1` LIMIT 1
 SELECT * FROM (select * from product where id > 0) AS `t1` LIMIT 2 OFFSET 0
 SELECT * FROM (select * from product where id > 0) AS `t1` LIMIT 2 OFFSET 2
 SELECT * FROM (select * from product where id > 0) AS `t1` LIMIT 2 OFFSET 4
 SELECT * FROM (select * from product where id > 0) AS `t1` LIMIT 2 OFFSET 6
 SELECT * FROM (select * from product where id > 0) AS `t1` LIMIT 2 OFFSET 8
 SELECT * FROM (select * from product where id > 0) AS `t1` LIMIT 2 OFFSET 10
 SELECT * FROM (select * from product where id > 0) AS `t1` LIMIT 2 OFFSET 12
 SELECT * FROM (select * from product where id > 0) AS `t1` LIMIT 2 OFFSET 14
 SELECT * FROM (select * from product where id > 0) AS `t1` LIMIT 2 OFFSET 16
 SELECT * FROM (select * from product where id > 0) AS `t1` LIMIT 2 OFFSET 18
 SELECT * FROM (select * from product where id > 0) AS `t1` LIMIT 2 OFFSET 20
```
- 6、再次到达定时任务的时间，又执行第5步骤的操作（但不同的是`sql_last_value`变量已经不为0了，所以此时相当于增量查询）。我们新增3条数据做测试，以下语句是logstash执行增量查询时发送的SQL语句：
```
SELECT count(*) AS `count` FROM (select * from product where id > 44) AS `t1` LIMIT 1
SELECT * FROM (select * from product where id > 44) AS `t1` LIMIT 2 OFFSET 0
SELECT * FROM (select * from product where id > 44) AS `t1` LIMIT 2 OFFSET 2
```
- 7、Logstash获取到jdbc输入进来的数据后，配合后面需要配置的elasticsearch输出插件，把数据写到elasticsearch中。

Logstash周而复始的进行5,6,7的步骤，实现mysql数据全量&增量导入。在这个过程中，我们讲解了几个关键的jdbc插件属性配置的作用，如果还有其他属性是不知道它们的作用的，可以回顾上一章输入插件：jdbc的详细介绍。
#### filter配置
```
filter {
    if [type] == "nginx-access" {
        grok {
            match => {
                "message" => "%{IP:remote_ip} \- \- \[%{HTTPDATE:timestamp}\] \"%{WORD:method} %{URIPATHPARAM:request} HTTP/%{NUMBER:http_version}\" %{NUMBER:status} %{NUMBER:bytes} \"%{URI:domain}\" \"%{GREEDYDATA:user_agent}"
            }
            remove_field => ["message","@timestamp","@version"]
        }
        date {
            match => ["timestamp","dd/MMM/YYYY:HH:mm:ss Z"]
            target => "timestamp"
        }
        if [remote_ip] !~ "^127\.|^192\.168\.|^172\.1[6-9]\.|^172\.2[0-9]\.|^172\.3[01]\.|^10\." {
            geoip {
                source => "remote_ip"
            }
        }
    }
}
```
由于Logstash通过jdbc输入插件，拿到的数据暂时不需要做什么解析，所以不需要配置过滤插件，这里就直接使用逻辑判断数据来源，如果type是`nginx-access`，才进入grok和date过滤插件的处理。
#### output配置
```
output {
    if [type] == "nginx-access" {
        elasticsearch {
            hosts => ["127.0.0.1:9200"]
            index => "logstash-%{type}"
            document_type => "%{type}"
        }
    }
    if [type] == "product" {
        elasticsearch {
            hosts => ["127.0.0.1:9200"]
            index => "mysql-%{type}"
            document_type => "%{type}"
            document_id => "%{id}"
        }
    }
    stdout {
    }
}
```
最后，我们在原来的output组件中再增肌一个elasticsearch输出插件，并且使用逻辑判断数据来源，分别储存到同一个elasticsearch搜索服务器中的不同索引库中，并且type=product的插件中，我们配置了`document_id => "%{id}"`，意思是使用mysql中查询出来的id，作为文档id
#### 验证数据
最后，我们看看elasticsearch中是新增了mysql_porduct索引库、product文档类型和文档数据：  
![image.png](https://upload-images.jianshu.io/upload_images/10574922-47bf0aa810202896.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
