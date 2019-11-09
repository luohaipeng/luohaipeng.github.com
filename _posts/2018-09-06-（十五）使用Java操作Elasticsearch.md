---
layout:     post
title:      （十五）使用Java操作Elasticsearch
author:     luohaipeng
header-img: img/post-bg-ios9-web.jpg
catalog: true
tags:
    - elasticsearch
---
## 前言
&emsp;&emsp;到目前为止，我们一直都是使用RESTful风格的 API操作elasticsearch服务，但是通过我们之前的学习知道，elasticsearch提供了很多语言的客户端用于操作elasticsearch服务，例如：java、python、.net、JavaScript、PHP等。而我们此次就学习如何使用java语言来操作elasticsearch服务。在elasticsearch的官网上提供了两种java语言的API，一种是Java Transport Client，一种是Java REST Client。  
&emsp;&emsp;而Java REST Client又分为Java Low Level REST Client和Java High Level REST Client，Java High Level REST Client是在Java Low Level REST Client的基础上做了封装，使其以更加面向对象和操作更加便利的方式调用elasticsearch服务。  
&emsp;&emsp;官方推荐使用Java High Level REST Client，因为在实际使用中，Java Transport Client在大并发的情况下会出现连接不稳定的情况。  
&emsp;&emsp;那接下来我们就来看看elasticsearch提供的Java High Level REST Client（以下简称高级REST客户端）的一些基础的操作，跟多的操作大家自行阅读elasticsearch的官方文档：`https://www.elastic.co/guide/en/elasticsearch/client/java-rest/current/java-rest-high.html`在官网上已经对高级REST客户端的各种API做了很详细的使用说明，我们这篇文章主要还是翻译官网上的内容，先让大家以更友好的中文文档方式入门，等大家熟悉了这些API之后在查阅官网。  
## 测试项目
&emsp;&emsp;在这里我也做了一个高级REST客户端的使用测试，该测试项目使用springboot开发，并且使用反射和泛型做了简易的封装，加强通用性。该项目的GitHub地址在以下链接：
> https://github.com/luohaipeng/es-java-api
## 高级REST客户端使用
### 导入依赖
我们这里以maven为例，使用高级REST客户端需要两个依赖，分别是：
```
      <dependency>
            <groupId>org.elasticsearch</groupId>
            <artifactId>elasticsearch</artifactId>
            <version>6.2.4</version>
        </dependency>

        <dependency>
            <groupId>org.elasticsearch.client</groupId>
            <artifactId>elasticsearch-rest-high-level-client</artifactId>
            <version>6.2.4</version>
        </dependency>
```
这两个依赖的版本是跟elasticsearch服务版本同步更新的，所以选择的依赖版本无需按照我这里的版本，而是以自己的elasticsearch服务的版本为主。
### 初始化
首先，我们想要操作elasticsearch，那必须先创建出连接的客户端对象，创建客户端对象的API如下：
```
        String[] ips = {"192.168.85.133:9200","192.168.85.133:9400"}
        HttpHost[] httpHosts = new HttpHost[ips.length];
        for(int i=0;i<ips.length;i++){
            httpHosts[i] = HttpHost.create(ips[i]);
        }
        RestClientBuilder builder = RestClient.builder(httpHosts);
        RestHighLevelClient client = new RestHighLevelClient(builder);
```
得到的RestHighLevelClient对象就是我们接下来操作elasticsearch所需要的了。
### 插入和更新文档操作
插入和更新文档需要我们构建一个IndexRequest对象。   
- 构造方法：`IndexRequest(String index, String type, String id)`第一个参数是该文档插入到哪个索引中，第二个参数是该文档插入到哪个文档类型中，第三个参数是指定文档的id。
- 设置文档内容方法：`source()`该方法有多个重载方法，我们可以把文档内容以json字符串的方式传递，也可以以xml方式传递，还可以用Map方式传递。  
- 更新文档操作：调用高级REST客户端的index方法，并传入IndexRequest对象。
具体代码如下：
```
public void insertOrUpdate(Object o) throws Exception {
        Map map = BeanUtil.bean2Map(o);
        IndexRequest request = new IndexRequest(baseIndex, baseType, map.get("id")+"");
        request.source(map);
        client.index(request);
    }
```
### 通过文档id删除文档
删除文档操作需要创建DeleteRequest对象。
- 构造方法：常用的构造方法：`DeleteRequest(String index, String type, String id)`第一个参数代表将要删除的该文档所在的索引，第二个参数代表将要删除的索引所在的文档类型，第三个参数代表要删除的文档对应的id
- 删除文档：调用高级REST客户端的delete方法，并传入DeleteRequest对象。  
具体代码如下：
```
public void delete(Long id) throws Exception {
        DeleteRequest request = new DeleteRequest(baseIndex, baseType, id + "");
        client.delete(request);
    }
````
### 通过文档id获取文档
通过文档id获取文档需要创建GetRequest对象。  
- 构造方法：常用构造方法：`GetRequest(String index, String type, String id)`第一个参数代表要获取的文档所在的索引，第二个参数代表要获取的索引所在的文档类型，第三个参数代表要获取的文档对应的id
- 获取文档：调用高级REST客户端的get方法，并传入GetRequest对象。  
- 返回结果：返回GetResponse对象，可以使用该对象的getSource()方法，获得文档数据，该数据封装成Map对象。  
具体代码如下：
```
public T get(Long id) throws Exception {
        GetRequest request = new GetRequest(baseIndex, baseType, id+"");
        GetResponse response = client.get(request);
        Map<String, Object> source = response.getSource();
        T t = BeanUtil.map2Bean(source, clazz);
        return t;
    }
```
### 搜索文档
搜索文档需要创建SearchRequest对象。
- 设置搜索的索引：`indices(String... indices)`，elasticsearch允许对多个索引一起搜索，所以SearchRequest对象中的indices方法可以设置多个索引。
- 设置搜索的文档类型：`types(String... types)`，elasticsearch允许对多个文档类型一起搜索，所以SearchRequest对象中的types方法可以设置多个文档类型。
- 设置搜索条件：搜索条件需要封装在SearchSourceBuilder对象中，我们只需要new SearchSourceBuilder()出该对象出来，然后为该对象设置搜索条件和数据范围相关参数即可。数据范围由from(int from)方法和 size(int size)方法指定。搜索条件由query(QueryBuilder query)方法设置。QueryBuilder对象就是最终封装搜索条件的对象，一个搜索条件就需要创建出一个该对象，该对象不需要我们手动创建，可以从QueryBuilders获取，QueryBuilders定义了各种搜索匹配的方式，我们只需要传入搜索的文档字段即可。
- 设置搜索关键字高亮：当我们输入关键字搜索，如果能搜索出相应的文档，那一般我们都会在该文档上，把匹配到的关键字高亮显示，设置高亮显示的对象为HighlightBuilder，该对象有两个方法，preTags()高亮前缀，postTags()高亮后缀，通过这两个前缀和后缀，把搜索匹配到的文档中，出现的搜索关键字的地方包裹起来，实现高亮的效果。创建出来的HighlightBuilder设置到SearchSourceBuilder中。然后SearchSourceBuilder又设置到SearchRequest对象中。
- 搜索：调用高级REST客户端的search方法，并传入SearchRequest对象。
- 返回结果：返回SearchResponse对象，调用该对象的getHits()方法，获取返回结果，并最终转为我们自己的业务Page对象。  
具体代码如下：
```
public PageResult search(QueryObject qo) throws Exception {
        SearchRequest request = new SearchRequest();
        request.indices(baseIndex);
        request.types(baseType);
        SearchSourceBuilder sourceBuilder = qo.createSearchSourceBuilder();
        HighlightBuilder highlightBuilder = qo.createHighlightBuilder();
        sourceBuilder.highlighter(highlightBuilder);
        request.source(sourceBuilder);
        SearchResponse response = client.search(request);
        SearchHits searchHits = response.getHits();
        long total = searchHits.getTotalHits();
        SearchHit[] searchHitArray = searchHits.getHits();
        List<T> data = new ArrayList<>();
        for(SearchHit hit : searchHitArray){
            Map<String, Object> source = hit.getSourceAsMap();
            T t = BeanUtil.map2Bean(source, clazz);
            qo.setHighlightFields(t,hit);
            data.add(t);
        }
        return new PageResult(data,Integer.parseInt(total+""),qo.getCurrentPage(),qo.getPageSize());

    }
```
