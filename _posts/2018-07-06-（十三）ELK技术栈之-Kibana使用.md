---
layout:     post
title:      （十三）ELK技术栈之-Kibana使用
author:     luohaipeng
header-img: img/post-bg-ios9-web.jpg
catalog: true
tags:
    - elasticsearch
---
## 匹配索引
我们在正式使用Kibana之前，需要先匹配我们Elasticsearch中的索引库，因为我们的Elasticsearch有可能会有很多索引库，Kibana为了性能因素，是不会事先把所有的索引库都导进来的，我们需要用那个索引就导哪个索引。  
按照如下步骤操作：**Management >> Index Patterns >> Create Index Patterns** 然后我们可以看到如下界面：  
![image.png](https://upload-images.jianshu.io/upload_images/10574922-582bcfa9c91b293f.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)  
在index pattern输入框中输入索引库，可以使用模糊查询的方式匹配，比如我们准备导入莎士比亚作品数据，输入“shak*”，当匹配成功后，输入框下方会出现一个成功的提示，并且右边会出来一个Next step按钮，点击该按钮进入下一步操作，然后点击Create index pattern 完成匹配：  
![image.png](https://upload-images.jianshu.io/upload_images/10574922-658c9386569ecf3b.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)  
我们按照这种方式再匹配账户数据和日志数据，但是匹配日志数据时需要注意：日志数据是按天来创建索引的，那我们的日志测试数据有3天的日志，需要把这3天的测试数据一起导入经历，所以我们用一下字符来匹配测试日志数据：  
![image.png](https://upload-images.jianshu.io/upload_images/10574922-8972941a79ee7722.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)  
然后Kibana需要我们选择一个时间字段，因为什么这3个索引是按照时间关联的，我们选择的时间字段是@timestamp。  
那现在我们已经把这3个类型的测试数据匹配到Kibana中了，接下来就可以使用Kibana了  
## 探索数据
点击**Discover**菜单，打开Kibana的数据发现功能：  
![image.png](https://upload-images.jianshu.io/upload_images/10574922-109f6c8655d994be.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)  
**索引数据列表**  
可以看到，这个界面右边列表，列出了莎士比亚作品测试数据，如果我们想看其他索引数据，可以左上角的下拉框中选择，比如我们选择了账户测试数据，右边列表就切换到账户测试数据了。  
**搜索数据**  
在页面的正下方，有一个查询框用于搜索你的数据。搜索需要一个特定的查询语法，而这个特定的语法就是Lucene的查询语法，它们能让你创建自己的搜索，点击查询框右边的按钮能保存这些搜索。在查询框的下方，当前的索引匹配模式显示在一个下拉选中，选择下拉选以改变匹配模式。你能用字段名和你感兴趣的值构建一个搜索，数字类型的数据可使用比较操作符比如>、<、=等,你可使用AND、OR、 NOT逻辑符连接元素，必须是大写。  
比如：`account_number:<100 AND balance:>47500`意思是查询账户测试数据索引中的account_number小于100并且balance大于47500的数据，那么查询出来的结果如下：  
![image.png](https://upload-images.jianshu.io/upload_images/10574922-53b3162b63cb3bfa.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)  
**字段选择**  
为了窄化显示某些感兴趣的字段，高亮索引模式匹配下面的列表中的字段，然后点击Add按钮。在这个例子中，注意怎么实现的，添加一个account_number字段后改变界面显示从5条记录的完整文本到一个只有账户号码的简单列表。  
![image.png](https://upload-images.jianshu.io/upload_images/10574922-0425fe04f2da4438.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)  
## 数据可视化
点击**Visualize**菜单，进入可视化图表创建界面，Kibana自带有上10种图表，下面我们来看看这些图表的使用。  

**饼状图使用示例**  
点击create visualize按钮，然后点击Pie图表，在From a New Search, Select Index中选择需要进行图表分析的索引，比如我们使用使用用户账号的索引ban*，点击了之后出现如下界面：  
![image.png](https://upload-images.jianshu.io/upload_images/10574922-751840c125e0a147.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)  
在该界面中，我们看到了一个完整的饼图，那是因为我们什么数据没有没有录入，那接下来我们录入一些数据看看。  
在Select buckets type下拉列表中，选择Split Slices，然后在Aggregation下拉列表中选择Range选项，在字段下拉列表中选择balance字段，点击Add Range按钮4次把区间增加到6个，输入一下区间。  
```
1000-1999
2000-2999
3000-3999
4000-4999
5000-5999
6000-6999
```
点击上方的绿色箭头，出来以下界面：  
![image.png](https://upload-images.jianshu.io/upload_images/10574922-2b2fedab5fdf0529.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)  
该饼状态显示出了各个转户资金范围的比例，除此之外，我们还可以增加一个维度来做数据的分析：点击add sub-buckets 选择Split Slices，然后在Aggregation下拉列表中选择Terms选项，在字段下拉列表中选择age字段，然后点击上方绿色箭头按钮，出来的结果如下：  
![image.png](https://upload-images.jianshu.io/upload_images/10574922-5a3cbec60ad0a4d9.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)  
以上的饼状图在原来的基础上再加了一个外环，表示在某个账户总额范围的年龄统计。  
同理，我们还可以再增加一维度，操作的方式跟上面一样，最后，如果有需要，还可以把这个图表保存起来，点击右上角**save**连接即可，保存好了之后，下次再进入可视化界面的首页，就可以看到之前保存过的图形了。  

**条形图使用示例**  
点击create visualize按钮，然后点击Vertical Bar图表，在From a New Search, Select Index中选择需要进行图表分析的索引，比如我们使用使用莎士比亚作品集测试数据shak*，点击了之后出现如下界面：  
![image.png](https://upload-images.jianshu.io/upload_images/10574922-941ff991f211dac2.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)  
对于Y轴的刻度聚合，在Aggregation下拉列表中选择Unique Count选项，在字段下拉列表中选择speaker字段。对于莎士比亚戏剧，知道那部戏剧需要最少数量的台前幕后人员可能是很有用的，如果你的戏剧公司短缺演员的话。对于X轴的量值，选择Terms聚合和play_name字段。对于排序，选择Ascending，Size保持默认值5。让其他参数保持默认值，然后点击上方绿色箭头，可以看到如下界面：  
![image.png](https://upload-images.jianshu.io/upload_images/10574922-27befb188cf0f685.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)  
 注意一下每部剧名是怎么显示成一个完整的词组而不是被拆分成单独的单词。这是我们在教程的前段部分设置映射的结果，我们把play_name标记为 keyword，不会进行拆词处理。鼠标移到每一个条上以tooltip形式显示每个剧台前幕后的数量。  
既然你有了莎士比亚剧中最小的演员表，你可能感兴趣知道这些剧本中哪一个对单个演员的要求最高，通过显示给定剧情的最大对话量。用Add metrics按钮增加一个Y轴聚合，为speech_number选择Max聚合。可以看到以下图表：  
![image.png](https://upload-images.jianshu.io/upload_images/10574922-12c986cd1ff0e86e.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)  

**地图使用示例**  
点击create visualize按钮，然后点击Coordinate Map，在From a New Search, Select Index中选择需要进行图表分析的索引，比如我们使用日志测试数据Logstash-2015*，点击了之后出现如下界面：  
![](https://upload-images.jianshu.io/upload_images/10574922-a8e538d0e2b73051.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)  
此时，我们只看到一个地图，但是地图上面没有任何数据，因为我们还没有设置任何值，而通过之前录入的测试数据知道，Logstash-2015包含3个索引，分别是Logstash-2015.5.18、Logstash-2015.5.19和Logstash-2015.5.20，所以，我们在该地图表最顶部，设置时间范围，点击Absolute，选择From为2015.5.18，To为：2015.5.20，点击Go按钮：  
![image.png](https://upload-images.jianshu.io/upload_images/10574922-bb6814efe444208a.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)  
然后设置数值，选择Geo Coordinates，然后在Aggregation下拉列表中选择Geohash，Field下拉列表中选择geo coordinates，然后点击上方绿色箭头的按钮，看到如下结果：  
![image.png](https://upload-images.jianshu.io/upload_images/10574922-6d6f26b393878c51.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)  
该图表的结果就是根据每个数据经纬度，定位到该数据的所在地，并且同一个位置会统计个数，圆点越大，说明该位置的数量越大，我们鼠标放上去可以看具体的经纬度和数量。  
*那我们以这3个比较典型的图表做例子，其他图表的使用方式大同小异，大家自行探索。*  
## 仪表盘
点击**Dashboard**菜单，进入仪表盘创建界面。  
一个Kibana仪表盘是许多图表的集合，它允许你整理和分享，点击Create a dashboard按钮，再点击Add按钮，显示出已保存图表的列表：  
![image.png](https://upload-images.jianshu.io/upload_images/10574922-169b44eee42db219.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)  
bar、map和pic这3个图表是我们上面学习图表使用时保存的，图表名是我们保存的时候取的名字，这时你点击该列表的图表名字，下方就会出现该图表，可以把多个图表放到一块，这就形式了我们说的仪表盘，如下图所示：  
![image.png](https://upload-images.jianshu.io/upload_images/10574922-12adec166c982372.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)  
仪表盘中的图表可以拖拽和放大缩小，比如我把上面的仪表盘变成了如下样子，位置和大小改成这样：  
![image.png](https://upload-images.jianshu.io/upload_images/10574922-5ca88acc3d1a25ad.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)  
最后，如果有需要，你可以保存该仪表盘，点击最上方的Save连接，然后为仪表盘命名，我取名为my dashboard。你还可以通过点击Share连接来显示HTML嵌入代码或者是一个定向链接分享一个保存的仪表盘。