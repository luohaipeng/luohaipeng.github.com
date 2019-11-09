---
layout:     post
title:      （五）安装Elasticsearch-head插件
author:     luohaipeng
header-img: img/post-bg-ios9-web.jpg
catalog: true
tags:
    - elasticsearch
---
## 前言
通过前面的学习我们已经可以往elasticsearch中存数据了，我们知道elasticsearch天生就是为海量数据和大规模集群而存在的，所以如果我们想要管理这些数据和集群，那么肯定得借助其他的一些工具进行管理，不然大家可以想象一下，我想要知道我的索引有哪些，数据量有多大，还得发送一个一个REST去查。我想管理我elasticsearch集群中所有的主机，看看它们的运行状态，集群数量，还得一台一台机器远程连接进去。这个工作量我想任何一个运维人员都是受不了的，所有这个时候，如果有一个图形管理界面，可以让我们集中并很直观管理我们的数据和集群，那这个工具没有理由不使用把！而这个图形界面就是elasticsearch-head插件。  
## elasticsearch-head插件介绍
elasticsearch-head是一个界面化的集群操作和管理工具，可以对集群进行傻瓜式操作。你可以通过插件把它集成到elasticsearch（5.0版本后不支持此方式）,也可以安装成一个独立webapp。elasticsearch-head插件是使用JavaScript开发的，依赖Node.js库，使用Grunt工具构建，所以等会我们要安装elasticsearch-head，还需要先安装Node.js和Grunt。  
*elasticsearch-head主要的作用有以下这些方面：*  
- 显示集群的拓扑,并且能够执行索引和节点级别操作  
- 搜索接口能够查询集群中原始json或表格格式的检索数据  
- 能够快速访问并显示集群的状态  
- 有一个输入窗口,允许任意调用RESTful API。这个接口包含几个选项,可以组合在一起以产生不同的结果;   
- 请求方法(get、put、post、delete),查询json数据,节点和路径  
- 支持JSON验证器  
- 支持重复请求计时器  
- 支持使用javascript表达式变换结果  
## 安装步骤

#### 安装node.js
1、下载node.js  
下载地址：`https://nodejs.org/dist/v6.9.2/node-v6.9.2-linux-x64.tar.xz`，把node.js的安装包下载下来后上传Linux主机上  

2、解压node.js安装包  
`xz -d node-v6.9.2-linux-x64.tar.xz   #使用xz命令解压xz包，解压出来的结果是tar包`  
`tar –xvf node-v6.9.2-linux-x64.tar  #解压tar包`  
`mv node-v6.9.2-linux-x64 /usr/local/node`  

3、配置环境变量  
`vi /etc/profile  #打开配置文件`   
`NODE_HOME=/usr/local/node  #添加node_home变量`  
`PATH=$PATH:$NODE_HOME/bin  #把node_home变量添加到环境变量中`  
`export PATH NOTH_HOME`  
`source /etc/profile  #重新加载环境变量配置文件`  
`node -v  #验证node.js是否安装成功，如果能查看到版本号说明安装成功`  
`npm -v  #验证npm工具是否安装成功，如果能查看到版本号说明安装成功`  

#### 安装elasticsearch-head插件
1、下载elasticsearch-head  
下载地址：`https://github.com/mobz/elasticsearch-head`把源码下载下来，下载的包名为：`elasticsearch-head-master.zip`，然后上传到Linux主机上。  

2、解压elasticsearch-head源码包  
`unzip elasticsearch-head-master.zip`   
`mv  elasticsearch-head-master /usr/local/es-head`  

3、安装grunt工具  
`cd /usr/local/es-head  #进入elasticsearch-head插件的源码目录`  
`npm install -g grunt --registry=https://registry.npm.taobao.org  #安装grunt工具，由于在国内连接国外的镜像速度奇慢无比，所以我们使用了国内taobao的镜像。`  

4、编译elasticsearch-head源码  
`npm install  #使用npm编译es-head源码，但是建议大家不要用这个这种方式安装，因为使用npm编译源码会下载很多依赖，这些依赖默认到国外镜像下载，所以速度会很慢，看下面的命令：`  
`npm install -g cnpm --registry=https://registry.npm.taobao.org  #安装cnpm，这是链接中国的镜像`  
`cnpm install  #使用cnpm代替npm编译es-head源码`  
编译好后es-head根目录下会出现一个叫node_modules的目录，该目录就是存放源码编译后的可执行文件。  
那到现在我们已经把elasticsearch-head插件和它的依赖都安装好了，那接下来我们就要对他的配置进行修改，准备运行。  

## 插件配置修改
1、设置插件管理界面跨主机访问  
插件默认是只有本机的IP才能访问的，也就是127.0.0.1，这样我们就无法跨主机访问head 插件的管理界面，所以需要把它改成所有IP地址都能访问。该配置在head插件安装目录根目录下，文件名为Gruntfile.js。  
`cd /usr/local/es-head`  
`vi Gruntfile.js`  
在该配置文件中connect-server-options下添加`hostname: '0.0.0.0',`这个配置，这样就不限制IP地址的访问了，具体看以下配置文件截图：  
![配置.png](https://upload-images.jianshu.io/upload_images/10574922-e6688a6c8a51f9f3.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)  
*注意：要把主机防火墙9100的端口打开，这里就不写出打开防火墙端口的命令了，之前也说过很多了。*  

2、设置连接elasticsearch的地址  
head插件默认是连接本机的elasticsearch的，如果你的elasticsearch和head插件是安装在同一台主机上，那么就不需要修改配置，如果不是安装在同一台主机的，就必须修改配置了，配置文件在head插件安装目录的_site目录下，文件名为app.js。  
`vi app.js  #编辑app.js文件`  
把`this.base_uri = this.config.base_uri || this.prefs.get("app-base_uri") || "http://localhost:9200";`这行配置中的`localhost`改成你elasticsearch服务所在IP地址（如果安装在同一台主机就不需要修改），具体看以下配置文件截图：  
![配置3.png](https://upload-images.jianshu.io/upload_images/10574922-6165e4c049b7740a.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)  

3、elasticsearch配置允许跨域访问  
`cd /usr/local/elasticsearch/config  #进入elasticsearch存放配置文件的目录`  
`vi elasticsearch.ym  #编辑elasticsearch配置文件`  
在该配置文件中最末尾添加两个属性：`http.cors.enabled: true`和`http.cors.allow-origin: "*"` 使head插件可以访问elasticsearch，具体看以下配置文件截图：  
![配置2.png](https://upload-images.jianshu.io/upload_images/10574922-86991661e9fa5448.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)  

## 运行和使用head插件
1、因为我们刚刚修改了elasticsearch的配置，所以首先需要重启elasticsearch  

2、运行head插件  
`cd /usr/local/es-head  #先进入到head插件的安装目录根目录`  
`grunt server  #使用刚刚安装的grunt工具运行head插件`  
如果能打印以下的信息，那就说明head插件安装并运行成功，并且我们通过打印信息看到，它告诉我们可以通过localhost:9100访问web服务，该web服务就是head插件的管理界面，但是这个localhost要改成head插件所在的主机的具体IP地址。接下来我们就可以通过浏览器来访问head插件的管理界面了，在这管理界面中管理我们的elasticsearch服务。  
```
Running "connect:server" (connect) task
Waiting forever...
Started connect web server on http://localhost:9100
```
3、访问head插件管理界面  
我的head插件所在的IP地址为192.168.85.133，所以我在另一台主机的浏览器地址栏输入http://192.168.85.133:9100。看到的界面如下：  
![head管理界面.png](https://upload-images.jianshu.io/upload_images/10574922-9517bd3a01ded24a.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)  

## head插件功能
通过刚刚我们用浏览器访问head插件看到的管理界面，大概可以知道它有几块功能块：概述、索引、数据浏览、基本查询、复合查询这几块，我们一个一个来看看：  
1、概述：  
这个功能块主要是用来管理elasticsearch集群状况的，我们可以用图形化界面来管理集群，查看集群中各个elasticsearch节点运行状态和节点的各种信息。后面我们学习elasticsearch集群再来详细看看它的功能。  

2、索引：  
这部分功能块是让我们管理整个elasticsearch的索引的，它展示了索引的列表，并且我们可以在上面以图形化的方式创建索引：  
![索引功能块.png](https://upload-images.jianshu.io/upload_images/10574922-40dc95a15027738f.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)  

3、数据浏览：  
这部分功能块顾名思义就是浏览elasticsearch中的数据的，我们可以查看整个elasticsearch所有数据，也可以根据索引、类型、字段组合查询出我们想要看的数据，左边蓝色背景的界面就是用于查询的，其中字段查询是可以精确查询和模糊查询，右边就是展示出具体的数据了。展示的数据列表中，有索引、类型和各个字段的数据，在列表最上方还有显示数据查询所消耗的时间：  
![数据浏览.png](https://upload-images.jianshu.io/upload_images/10574922-71c1162e4667fddd.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)  

4、基本查询：  
基本查询其实就相当于我们基础班学的高级查询了，不过它的功能要强大很多，可以做各种组合查询，其实就是以图形化的界面来查询数据，不需要我们手动写RESTful API去查询，对于想要简单快速查询数据的情景下还是挺好用的，但是它只能做查询搜索，不能创建、修改和删除数据，如果想要执行这些操作，那就只能用下一个功能块“复合查询”了。  
![简单查询.png](https://upload-images.jianshu.io/upload_images/10574922-6c955b34ff488b1a.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)  

5、复合查询：  
复合查询就是发送RESTful API到elasticsearch服务执行，然后它返回执行结果给我们，所以，我们使用head管理界面中的复合查询功能块，就可以不需要postman了。  
![复合查询.png](https://upload-images.jianshu.io/upload_images/10574922-8cae35ea87d8f2b5.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)  





