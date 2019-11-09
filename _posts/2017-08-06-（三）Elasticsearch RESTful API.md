---
layout:     post
title:      （三）Elasticsearch RESTful API
author:     luohaipeng
header-img: img/post-bg-ios9-web.jpg
catalog: true
tags:
    - elasticsearch
---
## 前言
&#160; &#160; &#160; &#160;Elasticsearch的一个很大的优势是支持多种语言，比如有Java API，.Net API等等，最重要的是它还支持使用RESTful API。RESTful是统一规范的http接口，任何语言都可以使用，如果大家不了解什么是RESTful，那么可以关注叩丁狼的技术文章和公开视频。我们要想学会Elasticsearch的使用，那么以RESTful作为突破口将是非常适合的，因为我们可以直接使用web客户端，如：postman来测试，甚至你还可以使用Linux上的curl工具测试，不需要自己写程序来调用Elasticsearch服务，轻松的了解Elasticsearch的各种用法。  
## 新建文档
&#160; &#160; &#160; &#160;前面我们已经了解了Elasticsearch的相关概念，我们知道Elasticsearch的数据都是以文档（Document）为载体储存的，相当于MySQL的一行数据，而要想插入文档，首先要先确定索引（index）和文档类型（type），Elasticsearch的索引相当于MySQL的数据库，文档类型相当于MySQL的表，所以我们接下来就把文档插入到对应的索引和文档类型中：  
```
PUT /store/employee/1
{
	"name":"张三",
	"age":25,
	"about":"我是一个中国人，张姓在中国是大姓，你有神马意见吗？",
	"interests":["sports","music"]
}
```
以上的RESTful API就是等会我们要插入的文档，我们来看看这里包含的意思：  
使用http PUT方法请求/store/employee/1资源路径  

路径 /store/employee/1 包含了三部分的信息：  
store：索引名  
employee：文档类型名  
1：员工的id  

json数据格式的请求体内容包含的信息：  
名字叫张三，年龄25岁，个人简介为我是一个中国人，张姓在中国是大姓，你有神马意见吗？，兴趣是运动和唱歌  

*我们使用postman来测试以上的接口：*  
![postman调用新增文档接口.png](https://upload-images.jianshu.io/upload_images/10574922-01b7aafea49ef6a1.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)  
&#160; &#160; &#160; &#160;很简单的就完成了一个文档的新增操作了，这其中的执行过程是：先查找名字叫store的索引，如果没有，则创建该索引。然后查找名字叫employee的文档类型，如果没有，则创建该文档类型。最后查找id为1的员工，如果没有则相当于新建一个员工文档，并设置员工id为1，如果有，则相当于更新员工id为1的文档数据。这里的文档字段类型由Elasticsearch帮我们完成自动匹配，当然我们也可以自定义文档的各个字段内容的数据类型和其他的一些约束，这些我们后面再一步一步来讲。  
## 删除文档
&#160; &#160; &#160; &#160;有新增操作那么肯定少不了删除操作了，那接下来我们来看看elasticsearch使用RESTful API来操作删除文档。  
**删除指定id的文档：**  
``DELETE /store/employee/1``  
该RESTful API的意思是使用http DELETE方法请求/store/employee/1资源路径  
![postman调用删除指定id文档接口.png](https://upload-images.jianshu.io/upload_images/10574922-2641bf77e9b06865.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)  
&#160; &#160; &#160; &#160;我们可以看到postman响应窗口中的响应题Body中，有个result的属性，值为deleted，说明删除成功。那怎么验证员工id为1的文档真的已经删除成了呢？我们再来学习个RESTful API，就是获取文档总数，我们通过查看文档总数来验证删除是否成功：  

**查看文档总数：**  
``GET /store/employee/_count``  
![postman调用获取文档总数接口.png](https://upload-images.jianshu.io/upload_images/10574922-d0e899a4403b3caa.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)  
&#160; &#160; &#160; &#160;我们可以看到postman响应窗口中的响应题Body中，有个count的属性，值为0，说明我们的文档已经被删除掉了。（shards属性中有个total为5，代表有5个碎片，因为我们之前新增了5个文档，然后都删除了，新增过的文档会留下碎片）  

&#160; &#160; &#160; &#160;我们这个章节主要讲新增和删除文档，还有额外补充了一个查看文档总数这几个RESTful API，那么下一个章节我们就要开始学习Elasticsearch的搜索了，这个是最重要，组合最多的操作，就好比我们sql查询语句那样。