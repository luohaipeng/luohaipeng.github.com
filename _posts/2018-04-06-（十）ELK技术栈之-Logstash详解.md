---
layout:     post
title:      （十）ELK技术栈之-Logstash详解
author:     luohaipeng
header-img: img/post-bg-ios9-web.jpg
catalog: true
tags:
    - elasticsearch
---
##  前言
在第九章节中，我们已经安装好Logstash组件了，并且启动实例测试它的数据输入和输出，但是用的是最简单的控制台标准输入和标准输出，那这节我们就来深入的学习Logstash的详细使用。
##  常用启动参数
我们在上一节中演示了启动Logstash的实例，其中我们启动的时候给Logstash脚本传入了-e的参数，但实际上，Logstash的启动参数有很多，我们来看一下各个启动参数的作用：  
- -e    #立即启动实例，例如：`./logstash -e "input {stdin {}} output {stdout {}}"`
- -f    #指定启动实例的配置文件，例如：`./logstash -f config/test.conf`
- -t    #测试配置文件的正确性，例如：`./logstash -f config/test.conf -t`
- -l    #指定日志文件名称，例如：`./logstash -f config/test.conf -l logs/test.log`
- -w    #指定filter线程数量，不指定默认是5，例如：`./logstash-f config/test.conf -w 8`

## 配置文件语法
#### 文件结构
我们刚刚知道，启动参数可以指定一个配置文件，那么接下来就有必要来了解一下配置文件的结构：  
*Logstash通过{}来定义区域，区域内可以定义插件，一个区域内可以定义多个插件，如下：*  
```
input {
    #标准输入源插件
    stdin {
    }
    #普通文件源插件
    file {
        path => ["/var/log/*.log", "/var/log/message"]
       ....
    }
   ......
}
filter {
      #grok过滤插件
      grok {
            match => ["message", "%{HTTPDATE:logdate}"]
            .....
      }
     #date过滤插件
     date {
            match => ["logdate", "dd/MMM/yyyy:HH:mm:ss Z"]
           .....
    }
   .....
}
output {
     stdout {
     }
     elasticsearch {
        hosts => ["127.0.0.1:9200"]
        ....
    }
    .....
}
```
我们先大概了解一下配置文件的结构，接下来我们再详细看这些插件的配置。  
#### 数据类型
Logstash配置文件支持的数据类型有：  
1、Boolean，例如：ssl_enable => true  
2、Number，例如：port => 33  
3、String，例如：name => “Hello world”  
4、hash，例如：options => {key1 => "value1", key2 => "value2"}  
5、array，例如：match => ["datetime", "UNIX", "ISO8601"]  

#### 字段引用
Logstash数据流中的数据被称之为Event对象，Event以JSON结构构成，Event的属性被称之为字段，如果你想在配置文件中引用这些字段，只需要把字段的名字写在中括号`[]`里就行了，如`[type]`，对于嵌套字段每层字段名称都写在`[]`里就可以了，比如：`[tags][type]`；除此之外，对于Logstash的arrag类型支持下标与倒序下表，如：`[tags][type][0]`和`[tags][type][-1]`  
*以下的内容就是一个Event对象：*  
```
{
      "message" => "hello logstash",
      "@version" => "1",
      "@timestamp" => 2018-08-13T17:32:01.122Z,
      "host" => "localhost.localdomain"
}
```
#### 条件判断
Logstash支持下面的操作符：  
1、==(等于), !=(不等于), <(小于), >(大于), <=(小于等于), >=(大于等于)  
2、=~(匹配正则), !~（不匹配正则）  
3、in(包含), not in(不包含)  
4、and(与), or(或), nand(非与), xor(非或)  
5、()(复合表达式), !()(对复合表达式结果取反)  
*例如以下的条件判断：*  
```
if "_grokparsefailure" not in [tags] {
} 
else if [status] !~ /^2\d\d/ or ( [url] == "/noc.gif" nand [geoip][city] != "beijing" ) {
} 
else {
}
```
#### 环境变量引用
Logstash支持引用系统环境变量，环境变量不存在时可以设置默认值，例如：  
```
export TCP_PORT=12345
input {
  tcp {
    port => "${TCP_PORT:54321}"
  }
}
```
## 常用输入插件
在第九章中，我们已经使用是标准输入，以键盘的输入数据作为Logstash数据源，但实际上我们也知道，Logstash的数据源有很多，每种数据源都有相应的配置，在Logstash中，这些数据源的相应配置称为插件，我们常用的输入插件有：file、jdbc、redis、tcp、syslog，这些输入插件会监听数据源的数据，如果新增数据，将数据封装成Event进程处理或者传递，更多的输入插件大家可以看Logstash官网，接下来我们以file和jdbc两个输入插件作为例子，来学习输入插件的使用，其他输入插件使用起来大同小异，大家自行扩展。
#### file输入插件
读取文件插件主要用来抓取文件的数据变化信息，以此作为Logstash的数据源。  
- 配置示例：
```
input{
  file {
    path => ["/var/log/*.log", "/var/log/message"]
    type => "system"
    start_position => "beginning"
  }
}
output{
  stdout{}
}
```
- 常用参数  

|参数名称|数据类型|默认值|描述|
|:--:|:--:|:---:|:---:|
|path|array|无|用于匹配被监控的文件，如"/var/logs\/*.log"或者 "/var/log/message"，必须使用绝对路径|
|type|string|无|Event的type字段，如果采用elasticsearch做store，在默认情况下将作为elasticsearch的type|
|sincedb_path|string|“$HOME/.sincedb*”|文件读取记录，必须指定一个文件而不是目录，文件中保存没个被监控的文件等当前inode和byteoffset|
|sincedb_write_interval|number|15|间隔多少秒写一次sincedb文件|
|start_position|string|"end"|值为“beginning”和“end”，从文件等开头还是结尾读取文件内容，默认是结尾，如果需要导入文件中的老数据，可以设置为“beginning”，该选项只在第一次启动logstash时有效，如果文件已经存在于sincedb的记录内，则此配置无效|
|stat_interval|number|1|间隔多少秒检查一下文件是否被修改，加大此参数将降低系统负载，但是增加了发现新日志的间隔时间|
|close_older|number|3600|设置文件多少秒内没有更新就关掉对文件的监听|
|codec|string|“plain”|输入数据之后对数据进行解码|
|add_field|hash|{}|用于向Event中添加字段|
|delimiter|string|“\n”|文件内容的行分隔符，默认按照换行进行Event对象封装，也就是说一行数据就一个Event对象|
|discover_interval|number|15|间隔多少秒查看一下path匹配的路径下是否有新文件产生|
|exclude|array|无|path匹配的文件中指定例外，如：path => “/var/log/“；exclude =>”\*.gz”|
|id|string|无|区分两个相同类型的插件，比如两个filter|
|ignore_older|number|无|忽略历史修改，如果设置3600秒，logstash只会发现一小时内被修改过的文件，一小时之前修改的文件的变化不会被读取，如果再次修改该文件，所有的变化都会被读取，默认被禁用|
|tags|array|无|可以在Event中增加标签，以便于在后续的处理流程中使用|
#### jdbc输入插件
该插件可以使用jdbc把关系型数据库的数据作为Logstash的数据源  
- 配置示例：
```
input {
  jdbc {
    jdbc_driver_library => "/opt/logstash/mysql-connector-java-5.1.36-bin.jar"
    jdbc_driver_class => "com.mysql.jdbc.Driver"
    jdbc_connection_string => "jdbc:mysql://localhost:3306/mydb"
    jdbc_user => "mysql"
    jdbc_password => "123456"
    parameters => { "favorite_artist" => "Beethoven" }
    schedule => "* * * * *"
    statement => "SELECT * from songs where artist = :favorite_artist"
  }
}
output{
  stdout{}
}
```
- 常用参数(空 = 同上)  

|参数名称|数据类型|默认值|描述|
|:--:|:--:|:---:|:---:|
| jdbc_driver_library|string|无|jdbc连接驱动的jar包位置|
| jdbc_driver_class |string|无|jdbc驱动类|
| jdbc_connection_string |string|无|数据库的连接地址|
| jdbc_user |string|无|数据库的用户名|
| jdbc_password |string|无|数据库的密码|
| jdbc_paging_enabled |boolean|false|开启分页|
| jdbc_page_size |number|100000|每页查询多少条数据|
| statement|string|无|执行的SQL查询语句|
| parameters|hash|{}|设置SQL查询语句的参数|
| use_column_value|boolean|false|是否需要在程序中使用查询出来的列|
| tracking_column  |string|无|需要在程序中使用哪个查询出来的列|
|clean_run|boolean|false|是否应该保留先前的运行状态|
|colums_charset|hsah|{}|特定列的字符编码。此选项将覆盖指定列的原来charset配置|
| schedule |string|无|定时执行数据读取的任务，使用cornexpression表达式：分、时、天、月、年，全部为*默认含义为每分钟执行任务，该cornexpression表达式于Linux系统的crontab使用方式一样，如果不会使用cornexpression表达式，可以查看crontab相关文档。不设置的话，只执行一次数据的读取|
| lowercase_column_names|boolean|true|是否需要把查询出来的列名转换成小写，也就是说即使列名是驼峰的，也会转成全部都是小写的，默认为转小写，如果不需要，则设置为false|
| record_last_run|boolean|true|是否记录数据库中最后一条数据的位置|
| last_run_metadata_path|string|无|记录数据库中最后一条数据的位置信息存放路径|
|add_field||||
|codec||||
|id||||
|tags||||
|type||||
## 常用过滤插件
丰富的过滤器插件的是 logstash威力如此强大的重要因素，过滤器插件主要处理流经当前Logstash的事件信息，可以添加字段、移除字段、转换字段类型，通过正则表达式切分数据等，也可以根据条件判断来进行不同的数据处理方式。我们常用的过滤插件有：grok、date、geoip、mutate、json、Split、ruby，更多的过滤插件大家可以看Logstash官网，接下来我们以grok、date和geoip这3个过滤插件作为例子，来学习过滤插件的使用，其他过滤插件使用起来大同小异，大家自行扩展。
#### grok正则插件
grok正则捕获是Logstash中将非结构化数据解析成结构化数据以便于查询的最好工具，非常适合解析system log，web log， database log等任意的 log文件。  
- 内置正则表达式调用  
grok提供100多个常用正则表达式可供使用，这100多个正则表达式定义在`logstash/vendor/bundle/jruby/x.x/gems/logstash-patterns-core-xxx/patterns/grok-patterns`文件中，想要灵活的匹配各种数据，那么必须查看该文件，大概了解grok提供了什么内置的正则表达式。调用它们的语法如下：%{SYNTAX:SEMANTIC}  
SYNTAX：表示内置的正则表达式的名称  
SEMANTIC：表示在Event中创建该字段名来存储匹配到的值  
例如：输入的数据内容为“[debug] 127.0.0.1 - test log content”，我们想提取127.0.0.1这个IP地址，那么可以使用以下语法来匹配：`%{IP:client}`，将获得“client: 127.0.0.1”的结果，该结果将成为Event的一个新的字段和字段值；如果你在捕获数据时想进行数据类型转换可以使用%{NUMBER:num:int}这种语法，默认情况下，所有的返回结果都是string类型，当前Logstash所支持的转换类型仅有“int”和“float”；  

- 自定义表达式调用  
与预定义表达式相同，你也可以将自定义的表达式配置到Logstash中，然后就可以像于定义的表达式一样使用；以下是操作步骤说明：  
1、在Logstash根目录下创建文件夹“patterns”，在“patterns”文件夹中创建文件“extra”（文件名称无所谓，可自己选择有意义的文件名称）；  
2、在文件“extra”中添加表达式，格式：patternName regexp，名称与表达式之间用空格隔开即可，例如：`POSTFIX_QUEUEID [0-9A-F]{10,11}`  
3、使用自定义的表达式时需要在grok插件配置“patterns_dir”属性，属性值为extra文件所在的目录。  

- 配置示例  
日志文件http.log每行内容为：`55.3.244.1 GET /index.html 15824 0.043 message-id:BEF25A72965`  
grok表达式：`表达式：%{IP:client} %{WORD:method} %{URIPATHPARAM:request} %{NUMBER:bytes} %{NUMBER:duration}`  
配置文件内容：  
```
input {
  file {
    path => "/var/log/http.log"
  }
}
filter {
  grok {
    patterns_dir => ["./patterns"]
    match => {"message" => "%{IP:client} %{WORD:method} %{URIPATHPARAM:request} %{NUMBER:bytes} %{NUMBER:duration} message\-id:%{POSTFIX_QUEUEID:queueid}"}
  }
}
output{
    stdout{}
}
```
输出结果：
```
client: 55.3.244.1
method: GET
request: /index.html
bytes: 15824
duration: 0.043
queueid: BEF25A72965
```
- 示例解析  
1、`/var/log/http.log`文件每一行的格式为`55.3.244.1 GET /index.html 15824 0.043 message-id:BEF25A72965`，到时候会把每一行数据封装成一个Event。  
2、使用grok过滤插件处理该文件，match为匹配Event中的message，message就是该文件的一行数据，比如`55.3.244.1 GET /index.html 15824 0.043 message-id:BEF25A72965`。  
3、匹配message的内容使用`%{IP:client} %{WORD:method} %{URIPATHPARAM:request} %{NUMBER:bytes} %{NUMBER:duration} message\-id:%{POSTFIX_QUEUEID:queueid}`表达式，把一条message拆成client、method、request、bytes、duration、queueid这6个字段，并添加到Event中。  
4、其中`%{POSTFIX_QUEUEID:queueid}`为自定义表达式的调用，其他5个是grok内置的表达式调用，如果需要使用自定义表达式，则需要在grok插件配置`patterns_dir`属性，属性值数据类型为array，就是自定义表达式定义的文件所在的目录。  
5、在表达式中，特殊字符需要使用`\`来转义，比如`-`，`""`，`[]`这些特殊字符，所以我们上面实例中的`message-id`数据在用表达式匹配的时候是使用了`message\-id`去匹配了。  
- 常用参数(空 = 同上)  

|参数名称|数据类型|默认值|描述|
|:--:|:--:|:---:|:---:|
| match |array|{}|设置pattern数组|
| patterns_dir |array|[]|指定自定义的pattern文件存放目录，Logstash在启动时会读取文件夹内所有文件，但前提是这些文件的内容是按照grok语法语法来定义的表达式变量|
| patterns_files_glob |string|"*"|用于匹配patterns_dir中的文件|
| add_field||||
| add_tag ||||
| id||||
| break_on_match |boolean|true|match字段存在多个pattern时，当第一个匹配成功后结束后面的匹配，如果想匹配所有的pattern，将此参数设置为false|
| keep_empty_captures|boolean|false|如果为true，捕获失败的字段奖设置为空值|
|clean_run|boolean|false|是否应该保留先前的运行状态|
|colums_charset|hsah|{}|特定列的字符编码。此选项将覆盖指定列的原来charset配置|
|overwrite|array|[]|覆盖字段内容: match=> { “message” => “%{SYSLOGBASE} %{DATA:message}” } overwrite=> [ “message” ]|
|periodic_flush|boolean|false|定期调用filter的flush方法|
|remove_field|array|[]|从Event中删除指定字段: remove_field=> [ “fieldname” ]|
|remove_tag|array|[]|删除“tags”中的值: remove_tag=> [ “tagname” ]|
|tag_on_failure|array|[“_grokparsefailure”]|当没有匹配成功时，将此array添加到“tags”字段内|
|tag_on_timeout|string|“_groktimeout”|当匹配超时时，将此字符串内容添加到“tags”字段内|
|timeout_millis|number|30000|设置单个match到超时时间，单位：毫秒，如果设置为0，则不启用超时设置|
#### date时间处理插件
在讲date插件的使用前，我们需要先讲解一下Logstash的时间记录方式。在Logstash产生了一个Event对象的时候，会给该Event设置一个时间，字段为“@timestamp”，同时，我们的日志内容一般也会有时间，但是这两个时间是不一样的，因为日志内容的时间是该日志打印出来的时间，而“@timestamp”字段的时间是input插件接收到了一条数据并创建Event的时间，所有一般来说的话“@timestamp”的时间要比日志内容的时间晚一点，因为Logstash监控数据变化，数据输入，创建Event导致的时间延迟。这两个时间都可以使用，具体要根据自己的需求来定。  
但是不管“@timestamp”字段的时间还是日志内容中的时间，其时间格式一般都不是我们想要的，所以我们就需要使用date插件，把时间格式转成我们想要的格式，比如：比如将`Apr 17 09:32:01（MMM dd HH:mm:ss）转换为04-17 09:32:01 （MM-dd HH:mm:ss）`。  
- 配置示例：
```
filter {
    date {
        match => ["timestamp","dd/MMM/YYYY:HH:mm:ss Z"]
    } 
}
```
- 示例解析  
`timestamp`是自定义的Event字段，用于存放通过grok解析到的日志内容中的时间，`dd/MMM/YYYY:HH:mm:ss Z`是日志内容中的时间格式，比如：16/Sep/2018:00:42:38 +0800，匹配到了之后，date插件会默认把该时间转成本地格式的时间，并且覆盖掉Event为我们创建的`@timestamp`字段中的时间。  
- 常用参数(空 = 同上)  

|参数名称|数据类型|默认值|描述|
|:--:|:--:|:---:|:---:|
| add_field||||
| add_tag ||||
| periodic_flush||||
| id||||
| remove_field||||
| remove_tag||||
| remove_tag||||
| tag_on_failure||||
| match |array|[]|时间字段匹配，可自定多种格式，直到匹配到或者匹配结束，格式：[ field,formats… ]，如：match=> [ “logdate”, “MMM dd yyyy HH:mm:ss”, “MMM d yyyy HH:mm:ss”, “ISO8601” ]|
| target|string|“@timestamp”|指定match匹配并且转换为date类型的存储位置（字段），默认覆盖到“@timestamp”|
| timezone|string|无|指定时间格式化的时区|
#### geoip插件
geoip插件是用于根据IP地址来确定该IP的归属地，默认的数据来源于Maxmind公司GeoLite2（https://dev.maxmind.com/geoip/geoip2/geolite2/）数据库，该数据库内嵌在geoip插件中，存储位置为：`logstash/vendor/bundle/jruby/x.x/gems/logstash-filter-geoip-x.x-java/vendor`目录中，数据库文件名分别为`GeoLite2-City.mmdb`和`GeoLite2-ASN.mmdb`。  
从Maxmind的描述 ----“GeoLite2数据库是免费的IP地理位置数据库，可与MaxMind收费的GeoIP2数据库相媲美，但没有GeoIP2准确”。 有关更多详细信息，请参阅GeoIP Lite2许可证。  
Maxmind的商业数据库GeoIP2（https://www.maxmind.com/en/geoip2-databases）也支持geoip插件。简单来说就是两个数据库一个免费的一个商业的，免费的不如商业的地理位置精准。  
如果您需要使用内嵌的GeoLite2以外的数据库，则可以直接从Maxmind网站下载数据库，并使用数据库选项指定其位置（下面会讲到如果使用数据选项）。  
- 配置示例：
```
filter {
     if [remote_ip] !~ "^127\.|^192\.168\.|^172\.1[6-9]\.|^172\.2[0-9]\.|^172\.3[01]\.|^10\." {
        geoip {
             source => "remote_ip"
             database => "/usr/share/GeoIP/GeoLite2-Country.mmdb"
        }
    }
}
```
- 示例解析  
由于geoip使用的ip数据库不能匹配私网地址，所以在使用geoip插件前，先判断一下ip地址是否为私网ip，如果不是，这使用geoip插件。source为指定一个ip地址，为其查询归属地等信息，remote_ip为Event中存储ip地址的自定义字段，database为IP地址数据库所在的位置，不指定的话，使用geoip插件内置的GeoLite2-City默认数据库。  
- 常用参数(空 = 同上)

|参数名称|数据类型|默认值|描述|
|:--:|:--:|:---:|:---:|
| add_field||||
| add_tag ||||
| periodic_flush||||
| id||||
| remove_field||||
| remove_tag||||
| remove_tag||||
| source|string|无|要通过geoip插件查询的IP地址所在的字段|
| tag_on_failure|array|[“_geoip_lookup_failure”]|如果ip地址查询不到地理位置信息，这在标签中添加该值|
| cache_size|number|1000|由于geoip查询很耗时间，所以默认情况下，查询过一次的ip地址会缓存起来，目前geoip没有内存数据驱逐策略，设置的缓存用完了就不能在缓存新的数据了，所以缓存需要设置一个合理值，太小查询性能增加不明显，太大就很占用服务器的内存|
| target|string|“geoip”|把IP地址匹配到的信息放到该字段中|
| database|string|无|使用的Maxmind数据库文件的路径。 如果不指定，则默认数据库是GeoLite2-City。 GeoLite2-City，GeoLite2-Country，GeoLite2-ASN是Maxmind支持的免费数据库。 GeoIP2-City，GeoIP2-ISP，GeoIP2-Country是Maxmind支持的商业数据库。|
| default_database_type|string|"City"|该字段的值只能接受"City"和"ASN"，GeoLite2数据库下细分为GeoLite2-City，GeoLite2-ASN和GeoLite2-Country，geoip插件内嵌了GeoLite2-City和GeoLite2-ASN，默认使用GeoLite2-City，如果需要使用GeoLite2-ASN，则需设置该字段并设置为ASN|
| fields|array|无|要包含在Event中的geoip字段的信息（每个数据库的信息都不一样）。 默认情况下，所有信息都会列出。对于内置的GeoLite2-City数据库，可以使用以下信息：city_name, continent_code, country_code2, country_code3, country_name, dma_code, ip, latitude, longitude, postal_code, region_name 和 timezone|
## 常用输出插件
经过以上的学习，我们已经学习了Logstash三大主件中的其中两个了，分别是input和filter，那现在我们就来学习最后一个组件：output。  
每个数据流经过input和filter后，最终要由output输出，以上的内容我们都是使用最简单的标准输出：stdout，把数据输出到显示器，但实际上stdout只是Logstash的其中一个输出插件而已，它的常用输出插件有：Elasticsearch，Redis，File，TCP等等，更多的输出插件大家可以看Logstash官网，接下来我们以Elasticsearch和Redis输出插件作为例子，来学习输出插件的使用，其他过滤插件使用起来大同小异，大家自行扩展。
#### elasticsearch输出插件
用于将Event信息写入到Elasticsearch中，官方推荐插件，ELK技术栈必备插件。  
- 配置示例
```
output {
    elasticsearch {
        hosts => ["127.0.0.1:9200"]
        index => "logstash-%{type}-%{+YYYY.MM.dd}"
        document_type => "%{type}"
    }
}
```
- 常用参数(空 = 同上)  

|参数名称|数据类型|默认值|描述|
|:--:|:--:|:---:|:---:|
| add_field||||
| add_tag ||||
| periodic_flush||||
| flush_size||||
| id||||
| remove_field||||
| remove_tag||||
| codec||||
| enable_metric||||
| hosts |array|无|elasticsearch服务器集群所在的地址|
| index|string|[无]|文档索引库名称|
| document_type |string|[无]|文档类型，不指定的话，类型名为“doc”|
| document_id|string|[无]|文档id，不指定的话，由elasticsearch自动生成|
| user|string|[无]|elasticsearch用户名|
| password|string|[无]|elasticsearch集群访问密码|
| parent|string|“nil”|为文档子节点指定父节点的id|
| pool_max|number|1000|elasticsearch最大连接数|
| pool_max_per_route|number|100|每个“endpoint”的最大连接数|
| proxy|string|无|代理URL|
| template|string|无|设置自定义的文档映射模版存放路径|
| template_name|string|无|设置使用的默版名称|
| template_overwrite|boolean|false|是否始终覆盖现有模版|
| manage_template|boolean|true|是否启用elasticsearch模版，Logstash自带一个模版，但是只有名称匹配“logstash-*”的索引才会应用该默版|
| parameters|hash|无|添加到elasticsearch URL后面的参数键值对|
| timeout|number|60|网络超时时间|
#### redis输出插件
用于将Event写入Redis中进行缓存，由于Redis数据库是先把数据存在内存的，所以效率会非常高，是一个常用的logstash输出插件  
- 配置示例  
```
output {
    redis {
        host => ["127.0.0.1"]
        port => 6379
        data_type => "list"
        key => "logstash-list"
    }
}
```
- 常用参数(空 = 同上)  

|参数名称|数据类型|默认值|描述|
|:--:|:--:|:---:|:---:|
| add_field||||
| add_tag ||||
| periodic_flush||||
| flush_size||||
| id||||
| remove_field||||
| remove_tag||||
| codec||||
| enable_metric||||
| hosts |array|["127.0.0.1"]|redis服务列表，如果配置多个，将随机选择一个，如果当前的redis服务不可用，将选择下一个|
| port |number|6379|文档索引库名称|
| db |number|0|使用的redis数据库编号，默认使用0号数据库|
| password|string|无|redis的密码|
| data_type |string|无|存储在redis中的数据类型，只能使用“list”和“channel”这两个值。如果使用“list”，将采用“RPUSH”操作，如果是“channel”，将采用“PUBLISH”操作|
| key|string|无|给redis设置一个key，来存储收集到的数据库|
| batch|boolean|false|是否启用redis的batch模式，仅在data_type=”list”时有效|
| batch_events|number|50|batch大小，batch达到此大小时执行“RPUSH”|
## 后记
那到这里，我们已经介绍了Logstash配置文件的语法和常用插件的配置方式了，这些常用的插件都是使用频率非常高的，所有我们后面需要来做一个Logstash的实战案例，综合运用我们这章所学的内容。