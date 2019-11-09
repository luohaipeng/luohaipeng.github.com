---
layout:     post
title:      （四）Elasticsearch RESTful API之搜索
author:     luohaipeng
header-img: img/post-bg-ios9-web.jpg
catalog: true
tags:
    - elasticsearch
---
## 前言
&#160; &#160; &#160; &#160;上一节我们已经介绍过了使用RESTful API来操作Elasticsearch了，但是上一节我们只是学到了如何新增文档、删除文档和通过文档id获取文档，那接下来我们将来学习一下使用RESTful API来操作Elasticsearch的文档搜索。  
## 简单搜索
**首先我们来看看不带任何搜索条件的最简单的搜索：**  
`GET  /store/employee/_search`  
```
{
    "took": 203,
    "timed_out": false,
    "_shards": {
        "total": 5,
        "successful": 5,
        "skipped": 0,
        "failed": 0
    },
    "hits": {
        "total": 3,
        "max_score": 1,
        "hits": [
            {
                "_index": "store",
                "_type": "employee",
                "_id": "2",
                "_score": 1,
                "_source": {
                    "name": "李四",
                    "age": 24,
                    "about": "我是一个作家",
                    "interests": [
                        "看书",
                        "写作"
                    ]
                }
            },
            {
                "_index": "store",
                "_type": "employee",
                "_id": "1",
                "_score": 1,
                "_source": {
                    "name": "张三",
                    "age": 25,
                    "about": "我是一个中国人，张姓在中国是大姓，你有神马意见吗？",
                    "interests": [
                        "sports",
                        "music"
                    ]
                }
            },
           省略下面的数据.....
        ]
    }
}
```
通过上面的这个RESTful API我们可以发现，我们查询的依然是store索引库和employee文档类型，但是该API跟我们上一节查询的又有不一样的地方，上一节内容我们给了一个文档id，获取指定的文档内容，而现在我们没有给指定的文档id，取而代之的是用了一个`_search`这个URL后缀。我们来看看这个返回结果中各个字段分别代表什么意思：  
took：是查询花费的时间，毫秒单位  
time_out：标识查询是否超时  
_shards：描述了查询分片的信息，查询了多少个分片、成功的分片数量、失败的分片数量等  
hits：搜索的结果，total是全部的满足的文档数目，hits是返回的实际数目（默认是10）  
_score是文档的分数信息，与排名相关度有关，参考各大搜索引擎的搜索结果，就容易理解。  

**带搜索条件的搜索**  
`GET  /store/employee/_search?q=name:张三`  
```
{
    "took": 292,
    "timed_out": false,
    "_shards": {
        "total": 5,
        "successful": 5,
        "skipped": 0,
        "failed": 0
    },
    "hits": {
        "total": 1,
        "max_score": 0.5753642,
        "hits": [
            {
                "_index": "store",
                "_type": "employee",
                "_id": "1",
                "_score": 0.5753642,
                "_source": {
                    "name": "张三",
                    "age": 25,
                    "about": "我是一个中国人，张姓在中国是大姓，你有神马意见吗？",
                    "interests": [
                        "sports",
                        "music"
                    ]
                }
            }
        ]
    }
}
```
通过以上的RESTful API我们可以发现：我们这次使用的RESTful API还是上面那个URL，但是我们给这个URL路径传入了参数了，这个参数名叫**q**,参数的值为**name:张三**，这就表示我们这次的搜索是需要指定特定内容的文档的，而这个特定内容就是name这个字段，值为张三的文档。  

## 使用DSL语句搜索
我们除了可以使用简单的传参方式搜索外，还可以使用Elasticsearch提供丰富且灵活的查询语言：DSL(Domain Specific Language特定领域语言)查询,它允许你构建更加复杂、强大的查询。DSL以JSON数据格式为载体，用HTTP请求体传输数据内容。  
**match查询：**  
`POST /store/employee/_search`  
`{
    "query" : {
        "match" : {
            "name" : "张"
        }
    }
}`  
```
{
    "took": 8,
    "timed_out": false,
    "_shards": {
        "total": 5,
        "successful": 5,
        "skipped": 0,
        "failed": 0
    },
    "hits": {
        "total": 1,
        "max_score": 0.2876821,
        "hits": [
            {
                "_index": "store",
                "_type": "employee",
                "_id": "1",
                "_score": 0.2876821,
                "_source": {
                    "name": "张三",
                    "age": 25,
                    "about": "我是一个中国人，张姓在中国是大姓，你有神马意见吗？",
                    "interests": [
                        "sports",
                        "music"
                    ]
                }
            }
        ]
    }
}
```
以上的这种搜索方式跟我们之前使用简单搜索效果是一样的，`{
    "query" : {
        "match" : {
            "name" : "张"
        }
    }
}`这个JSON数据就是Elasticsearch的DSL查询语言，它需要放到HTTP请求体中，**query**代表执行搜索操作，在该属性下，可以灵活的使用各种查询类型进行组合，**match**代表DSL语句中的其中一种查询类型，在该属性中就可以匹配我们需要查询文档的什么字段和字段值了，并且可以匹配多个字段。
同时我们根据返回内容上看发现，搜索出的文档score（得分）为0.28..，这是因为我们这次的搜索没有精准的匹配name字段为“张三”的文档，而是只匹配“张”，那么搜索出的文档得分就比较低了。
除此之外，我们还可以结合DSL语句的其他语法来控制搜索结果。如：  
`{
    "query" : {
        "match_all":{}
    },
    "size":2
}`  
*搜索全部员工文档，并返回前面两条数据，如果不给size字段，那么就返回前10条*。  

`{
    "query" : {
         "match" : {
            "name" : "张"
        }
    },
    "from":2,
    "size":2
}
`  
*搜索name字段带有“张”的员工文档，并且返回搜索结果从第二条开始，共两条数据*。  

`{
    "query" : {
        "match_all":{}
    },
    "sort":{
    	"age":{"order":"desc"}
    }
}`  
*搜索全部文档，并且返回结果安照“age”字段做倒序排序*。  

`{
	"query":{
		"match_phrase":{
			"about":"rock climbing"
		}
	}
}`  
*使用match的话我们只能匹配到一个单词，但是如果我们想要同时匹配多个单词的话就不能使用match了，而是使用match_phrase，该属性意思是短语搜索，例如上面这条DSL语句就是搜索包含rock climbing这个短语的文档，这个短语包含两个单词，这个两个单词是需要相邻的*。  

**bool查询：**  
elasticsearch还提供了bool语法查询，该语法可以把多个查询条件组合在一起，看以下的例子：  
`POST /store/employee/_search`  
`{
  "query": {
    "bool": {
      "must": [
        { 
        	"match": { 
        		"name": "张" 
        	}
        },
        { 
        	"range": {
        		"age": {
        			"gte":25
        		}        		
        	}        	
        }
      ]
    }
  }
}`  
查询结果如下：  
```
{
    "took": 18,
    "timed_out": false,
    "_shards": {
        "total": 5,
        "successful": 5,
        "skipped": 0,
        "failed": 0
    },
    "hits": {
        "total": 1,
        "max_score": 1.5753641,
        "hits": [
            {
                "_index": "store",
                "_type": "employee",
                "_id": "1",
                "_score": 1.5753641,
                "_source": {
                    "name": "张三",
                    "age": 25,
                    "about": "我是一个中国人，张姓在中国是大姓，你有神马意见吗？",
                    "interests": [
                        "sports",
                        "music"
                    ]
                }
            }
        ]
    }
}
```
以上的查询涉及到DSL新的语法：**bool**查询，这是一个可以让我们把多个搜索条件组合在一起的语法，结合属性**must**意思为所有组合条件都必须满足。同时这里还涉及到两个新的属性：**range**意思为范围查询和**gte**属性，意思为大于等于（gt为大于）。  
*那么整个查询语句意思为查询名字带有“张”，并且年龄大于等于25岁的员工文档。*  
此外，bool查询语法除了**must**之外，还有**must_not**和**should**。  
*如果我们把上面那条查询语句中的must改成should，那么意思就变成了：查询名字带有“张”，或者年龄大于等于25岁的员工文档。*  
在一条bool查询语句中还可以把**must**，**must_not**和**should**组合成更复杂的查询语句，比如以下的这条语句：  
`{
  "query": {
    "bool": {
      "must": [
        { 
        	"match": { 
        		"name": "张" 
        	}
        },
        { 
        	"range": {
        		"age": {
        			"gte":25
        		} 
        	} 
        }
      ],
      "must_not":{
      	"match":{
      		"age":29
      	}
      }
    }
  }
}
`  
*该语句意思是查询名字包含“张”并且年龄大于等于25岁的，但是不包含年龄为29岁的员工文档。*  
由此可以看出，bool查询可以很灵活的组合各种查询条件，类似于我们SQL语句中的where多个条件组合。  

**过滤查询**  
之前我们说过score字段指定了文档的分数，使用query搜索会计算文档的分数，最后通过分数确定哪些文档更相关，返回哪些文档，并且文档的排序默认是按分数倒序的。但是有的时候我们可能对分数不感兴趣，就可以使用filter进行过滤，它不会去计算分值，因此效率也就更高一些。我们来看看以下这条查询语句：  
`POST /store/employee/_search`  
`
{
  "query": {
    "bool": {
      "must": { "match_all": {} },
      "filter": {
        "range": {
          "age": {
            "gt": 20,
            "lt": 25
          }
        }
      }
    }
  }
}`  
```
{
    "took": 44,
    "timed_out": false,
    "_shards": {
        "total": 5,
        "successful": 5,
        "skipped": 0,
        "failed": 0
    },
    "hits": {
        "total": 2,
        "max_score": 1,
        "hits": [
            {
                "_index": "store",
                "_type": "employee",
                "_id": "2",
                "_score": 1,
                "_source": {
                    "name": "李四",
                    "age": 24,
                    "about": "I love to go rock climbing",
                    "interests": [
                        "sports",
                        "music",
                        "coding"
                    ]
                }
            },
            {
                "_index": "store",
                "_type": "employee",
                "_id": "1",
                "_score": 1,
                "_source": {
                    "name": "张三",
                    "age": 25,
                    "about": "我是一个中国人，张姓在中国是大姓，你有神马意见吗？",
                    "interests": [
                        "sports",
                        "music"
                    ]
                }
            }
        ]
    }
}
```
filter过滤可以嵌套在bool查询内部使用，比如这条语句意思就是过滤查询年龄大于20，小于25之间的员工文档。  

**高亮结果**  
所有带有全文检索的应用在搜索出来的文档结果上都会把搜索词高亮标记，这样用户就可以直观的知道为什么这些文档和查询条件相匹配了。在Elasticsearch中高亮片段是非常容易的，我们可以来看看以下这个例子：  
`GET /store/employee/_search`  
`
{
    "query" : {
        "match" : {
            "name" : "张"
        }
    },
    "highlight": {
        "fields" : {
            "name" : {}
        }
    }
}
`  
```
{
    "took": 8,
    "timed_out": false,
    "_shards": {
        "total": 5,
        "successful": 5,
        "skipped": 0,
        "failed": 0
    },
    "hits": {
        "total": 1,
        "max_score": 0.2876821,
        "hits": [
            {
                "_index": "store",
                "_type": "employee",
                "_id": "1",
                "_score": 0.2876821,
                "_source": {
                    "name": "张三",
                    "age": 25,
                    "about": "我是一个中国人，张姓在中国是大姓，你有神马意见吗？",
                    "interests": [
                        "sports",
                        "music"
                    ]
                },
                "highlight": {
                    "name": [
                        "<em>张</em>三"
                    ]
                }
            }
        ]
    }
}
```
这样，我们就可以把搜索名字带有“张”的员工文档搜索出来，并且搜索的结果多出了一个highlight的属性，在该属性中有name这个文档字段，其中字段值中的“张”是被<em>标签包裹的，代表高亮的意思，因为我们的搜索词就是“张”，所以elasticsearch把结果中的“张”高亮了。  

以上就是常用的搜索语句用法，当然，DSL功能非常强大，语法也不仅仅只有这些，但是目前为止，我们先学会了这些已经足够我们使用了，后续章节如何会涉及到其他的DSL语句我们再逐一的详细介绍其用法。

