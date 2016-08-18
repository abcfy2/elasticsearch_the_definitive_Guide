##结构化检索#
  - 结构化检索是使用具有内在结构的数据去搜索。比如说日期，时间，数字都是结构化的，这种数据都   有固定的模式。这就不需要去考虑文档的相关性或者得分的问题了。我们得到的只是简单的有和没有这两种结果。     

查找精确值  
  例如

```
POST /my_store/products/_bulk
{ "index": { "_id": 1 }}
{ "price" : 10, "productID" : "XHDK-A-1293-#fJ3" }
{ "index": { "_id": 2 }}
{ "price" : 20, "productID" : "KDKE-B-9947-#kL5" }
{ "index": { "_id": 3 }}
{ "price" : 30, "productID" : "JODL-X-1937-#pV7" }
{ "index": { "_id": 4 }}
{ "price" : 30, "productID" : "QQPX-R-3956-#aD8" }
```
我们想得到所有产品的一个确切的价格，用sql的方法是

```
SELECT document
FROM   products
WHERE  price = 20
```
在es中term可以完成这个查询

``` 
{
    "term" : {
        "price" : 20
     }
 }
```
 我们只是想要一个具体的结果，文档里有没有包含我们想要查询的值，我们不需要用得分来排名，只想得到在这个index中的type中有没有我们想要的结果。es中使用`constant_score` 查询方法来取消计分。

```
GET /my_store/products/_search
{
    "query" : {
        "constant_score" : { ￼
            "filter" : {
                "term" : { ￼
                    "price" : 20
                }
            }
        }
    }
}
 ```
这是返回的结果。

```markdown
"hits" : [
    {
        "_index" : "my_store",
        "_type" :  "products",
        "_id" :    "2",
        "_score" : 1.0, 
        "_source" : {
          "price" :     20,
          "productID" : "KDKE-B-9947-#kL5"
        }
    }
]
 ```
  term是包含的关系，并不是等于的意思  

```
{ "term" : { "tags" : "search" } }
```

```
{ "tags" : ["search"] }
{ "tags" : ["search", "open_source"] } 
```
NOTE  如果你只想找到唯一的结果，那么，最好是指定一个ID,原因就是es的分词和inverted index 引起的。
看看下面的这个例子

```
GET /my_store/products/_search
{
    "query" : {
        "constant_score" : {
            "filter" : {
                "term" : {
                    "productID" : "XHDK-A-1293-#fJ3"
                }
            }
        }
    }
}
 ```
结果

 ```
  "hits": {
    "total": 0,
    "max_score": null,
    "hits": []
  }
}
```
这是因为被es分词了，es默认是分词的，下面是查询是否分词。

```
GET /my_store/_analyze
{
  "field": "productID",
  "text": "QQPX-R-3956-#aD8"
}
```

```
{
  "tokens": [
    {
      "token": "qqpx",
      "start_offset": 0,
      "end_offset": 4,
      "type": "<ALPHANUM>",
      "position": 0
    },
    {
      "token": "r",
      "start_offset": 5,
      "end_offset": 6,
      "type": "<ALPHANUM>",
      "position": 1
    },
    {
      "token": "3956",
      "start_offset": 7,
      "end_offset": 11,
      "type": "<NUM>",
      "position": 2
    },
    {
      "token": "ad8",
      "start_offset": 13,
      "end_offset": 16,
      "type": "<ALPHANUM>",
      "position": 3
    }
  ]
}
```
这有一些重要的地方  
- 一个token被4个token取代了  
- 所有的字母变成了小写  
- 没有了- 和#   
所以，我们查不到我们所需要的。如果需要避免这种事情的发生，那就告诉es，给这个词设置为not_analyzed  

操作的步骤  

```
DELETE /my_store 

PUT /my_store 
{
    "mappings" : {
        "products" : {
            "properties" : {
                "productID" : {
                    "type" : "string",
                    "index" : "not_analyzed" 
                }
            }
        }
    }

}
```

```
POST /my_store/products/_bulk
{ "index": { "_id": 1 }}
{ "price" : 10, "productID" : "XHDK-A-1293-#fJ3" }
{ "index": { "_id": 2 }}
{ "price" : 20, "productID" : "KDKE-B-9947-#kL5" }
{ "index": { "_id": 3 }}
{ "price" : 30, "productID" : "JODL-X-1937-#pV7" }
{ "index": { "_id": 4 }}
{ "price" : 30, "productID" : "QQPX-R-3956-#aD8" }
```

```
GET /my_store/products/_search
{
    "query" : {
        "constant_score" : {
            "filter" : {
                "term" : {
                    "productID" : "XHDK-A-1293-#fJ3"
                }
            }
        }
    }
}
```
filter操作的本质  
 - es是如何执行不计分操作的
  - 找到匹配的文档  
  如: XHDK-A-1293-#fJ3 如果在一个文档中包含了，这个文档设置为1，否则，设置为0。
  - 构建一个位集合   
    如这个例子就会形成[1,0,0,0]。
  - 对构成的所有位集合进行迭代  
其执行的顺序是试探性的，但一般来说是最稀疏位集合被首先迭代(由于其不包括最大的文件数)
  - 使用次数计数器  

es会缓存不计分的查询，但这个是很少使用的，因为es本身是倒排索引，查询起来已经比较快了，因此，在使用缓存时，我们应该要缓存今后我们会再次使用的查询结果，避免浪费资源。  

  
es的自动策略是，在最近的256次查询中一个查询条件用了几次，那么，es就会把它作为缓存的候选者。并不是所有的segment都保证缓存bitset，只有segments持有超过10k文档(或者总文档其中较大的3%,)。因为小的字段是很快速搜索，快速合并的，它的缓存位集合就没有意义了。一旦缓存了，不计分的位集合就会一直保留到它被驱逐出缓存，驱逐这个操作是基于LRU(least-recently used) filter。一旦缓存满了，这个filter就会去清掉一些缓存。es在使用位集合时，must 和must_not其使用的是相同的位集合。  
segment es在处理实时更新数据的可用和可靠上，在Lucene的处理办法：新收到的的数据写到新的索引文件里，Lucene把每次生成的倒排索引，叫做一个段(segment),然后另外使用一个commit文件，记录索引内所有的segment。在buffer生成一个segment，刷新到文件系统缓存中，Lucene就可以检索到一个新的segment，在文件系统缓存真正同步到磁盘上时，会把小的segment整合成一个大的segment。
 

官方建议是，使用不计分查询优先的思想。这样会编写出高效快捷的搜索请求。
多条件过滤
实现多种条件下的查询方法，下面是sql实现的方法。

```markdown
SELECT product
FROM   products
WHERE  (price = 20 OR productID = "XHDK-A-1293-#fJ3")
  AND  (price != 30)
```
在es中，我们使用`Bool Filter` 的方法实现bool查询
```markdown
{
   "bool" : {
      "must" :     [],
      "should" :   [],
      "must_not" : [],
      "filter":    []
   }
}
```
`should` 相当于逻辑上的OR  
NOTE  
`filter` 只是匹配，不会有记分。相当于 `must`  
所有的条件都是可以并列形成一个查询数组，也可以只有一条，这是一个完整的实现，如  
```markdown
GET /my_store/products/_search
{
   "query" : {
      "constant_score" : { 
         "filter" : {
            "bool" : {
              "should" : [
                 { "term" : {"price" : 20}}, 
                 { "term" : {"productID" : "XHDK-A-1293-#fJ3"}} 
              ],
              "must_not" : {
                 "term" : {"price" : 30} 
              }
           }
         }
      }
   }
}
```
结果  
```markdown
"hits" : [
    {
        "_id" :     "1",
        "_score" :  1.0,
        "_source" : {
          "price" :     10,
          "productID" : "XHDK-A-1293-#fJ3" 
        }
    },
    {
        "_id" :     "2",
        "_score" :  1.0,
        "_source" : {
          "price" :     20, 
          "productID" : "KDKE-B-9947-#kL5"
        }
    }
]
```
####查找多个精确的值  
`term`关键字只能查询单个的条件精确的值，不能查询多个条件的值，es的方法是给`term`加个复数的方法，即`terms`，可以用一个数组来表示具体的值  
```markdown
{
    "terms" : {
        "price" : [20, 30]
    }
}
```
例如
```markdown
GET /my_store/products/_search
{
  "query": {
    "constant_score": {
      "filter": {
        "terms": {
          "price": [ 20, 30 ]
        }
      }
    }
  }
}
```
结果
```markdown
  "hits": {
    "total": 3,
    "max_score": 1,
    "hits": [
      {
        "_index": "my_store",
        "_type": "products",
        "_id": "2",
        "_score": 1,
        "_source": {
          "price": 20,
          "productID": "KDKE-B-9947-#kL5"
        }
      },
      {
        "_index": "my_store",
        "_type": "products",
        "_id": "4",
        "_score": 1,
        "_source": {
          "price": 30,
          "productID": "QQPX-R-3956-#aD8"
        }
      },
      {
        "_index": "my_store",
        "_type": "products",
        "_id": "3",
        "_score": 1,
        "_source": {
          "price": 30,
          "productID": "JODL-X-1937-#pV7"
        }
      }
    ]
  }
}
```
NOTE  由于ES是倒排索引，因此，指定的term越详细，查询的结果就越接近你预想的结果。
####查找一个范围的值#
 - 关键字 range   
例如

```
"range" : {
    "price" : {
        "gt" : 20,
        "lt" : 40
    }
}
```
 - gt: >
 - lt: <
 - gte: >=
 - lte: <=
完整的例子

```
GET /my_store/products/_search
{
    "query" : {
        "constant_score" : {
            "filter" : {
                "range" : {
                    "price" : {
                        "gte" : 20,
                        "lt"  : 40
                    }
                }
            }
        }
    }
}
```
```
  "hits": {
    "total": 3,
    "max_score": 1,
    "hits": [
      {
        "_index": "my_store",
        "_type": "products",
        "_id": "2",
        "_score": 1,
        "_source": {
          "price": 20,
          "productID": "KDKE-B-9947-#kL5"
        }
      },
      {
        "_index": "my_store",
        "_type": "products",
        "_id": "4",
        "_score": 1,
        "_source": {
          "price": 30,
          "productID": "QQPX-R-3956-#aD8"
        }
      },
      {
        "_index": "my_store",
        "_type": "products",
        "_id": "3",
        "_score": 1,
        "_source": {
          "price": 30,
          "productID": "JODL-X-1937-#pV7"
        }
      }
    ]
  }
}
```
可以指定一段时间来查询

```
"range" : {
    "timestamp" : {
        "gt" : "2014-01-01 00:00:00",
        "lt" : "2014-01-07 00:00:00"
    }
}
```
还支持日期的数学运算

```
"range" : {
    "timestamp" : {
        "gt" : "now-1h"
    }
}
```
```
"range" : {
    "timestamp" : {
        "gt" : "2014-01-01 00:00:00",
        "lt" : "2014-01-01 00:00:00||+1M" 
    }
}
```
NOTE 由于是倒排索引，es是按照字典顺序来来排的，因此，也可以这么用。

```
"range" : {
    "title" : {
        "gte" : "a",
        "lt" :  "b"
    }
}
```
NOTE 在使用range时，要注意查询基数的大小，给的条件越多，查的越慢。
####处理空值#
看看下面这个例子

```
POST /my_index/posts/_bulk
{ "index": { "_id": "1"              }}
{ "tags" : ["search"]                }
{ "index": { "_id": "2"              }}
{ "tags" : ["search", "open_source"] }
{ "index": { "_id": "3"              }}
{ "other_field" : "some data"        }
{ "index": { "_id": "4"              }}
{ "tags" : null                      }
{ "index": { "_id": "5"              }}
{ "tags" : ["search", null]          }
```
exits条件
```
GET /my_index/posts/_search
{
  "query": {
    "constant_score": {
      "filter": {
        "exists": {
          "field": "tags"
        }
      }
    }
  }
}

```
结果

```
"hits" : [
    {
      "_id" :     "1",
      "_score" :  1.0,
      "_source" : { "tags" : ["search"] }
    },
    {
      "_id" :     "5",
      "_score" :  1.0,
      "_source" : { "tags" : ["search", null] } 
    },
    {
      "_id" :     "2",
      "_score" :  1.0,
      "_source" : { "tags" : ["search", "open source"] }
    }
]
```
missing条件

```
  "hits": {
    "total": 2,
    "max_score": 1,
    "hits": [
      {
        "_index": "my_index",
        "_type": "posts",
        "_id": "4",
        "_score": 1,
        "_source": {
          "tags": null
        }
      },
      {
        "_index": "my_index",
        "_type": "posts",
        "_id": "3",
        "_score": 1,
        "_source": {
          "other_field": "some data"
        }
      }
    ]
  }
}
```
结果

```
  "hits": {
    "total": 2,
    "max_score": 1,
    "hits": [
      {
        "_index": "my_index",
        "_type": "posts",
        "_id": "4",
        "_score": 1,
        "_source": {
          "tags": null
        }
      },
      {
        "_index": "my_index",
        "_type": "posts",
        "_id": "3",
        "_score": 1,
        "_source": {
          "other_field": "some data"
        }
      }
    ]
  }
}
```
exits/missing 条件也可以有mapping一样的工作方式。若是需要具体指到哪一个的null，使用的方法就是[Types and Mapping](https://www.elastic.co/guide/en/elasticsearch/guide/2.x/mapping.html)。  
如文档为

```
{
   "name" : {
      "first" : "John",
      "last" :  "Smith"
   }
}
```
你可以查询name.first,name.last,还有name.   
如下面的filter也是可以使用的  

```
{
    "exists" : { "field" : "name" }
}
```
真正的执行方式是这样的

```markdown
{
    "bool": {
        "should": [
            { "exists": { "field": "name.first" }},
            { "exists": { "field": "name.last" }}
        ]
    }
}
```

