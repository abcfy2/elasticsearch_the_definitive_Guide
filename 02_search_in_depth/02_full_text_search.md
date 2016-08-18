###全文检索#
匹配查询  
  匹配查询是`go-to `查询，查询首先要知道你要查询那个字段，然后再做其他处理，这就是说match可以处理全文查询和精确值查询。   
索引
首先指定了主分片数,然后用_bulk插入。

```
PUT /my_index
{ "settings": { "number_of_shards": 1 }}
```

```
POST /my_index/my_type/_bulk
{ "index": { "_id": 1 }}
{ "title": "The quick brown fox" }
{ "index": { "_id": 2 }}
{ "title": "The quick brown fox jumps over the lazy dog" }
{ "index": { "_id": 3 }}
{ "title": "The quick brown fox jumps over the quick dog" }
{ "index": { "_id": 4 }}
{ "title": "Brown fox brown dog" }
```
查询条件

```
GET /my_index/my_type/_search
{
  "query": {
    "match": {
      "title": "QUICK!"
    }
  }
}
```
结果

```
  "hits": {
    "total": 3,
    "max_score": 0.5,
    "hits": [
      {
        "_index": "my_index",
        "_type": "my_type",
        "_id": "1",
        "_score": 0.5,
        "_source": {
          "title": "The quick brown fox"
        }
      },
      {
        "_index": "my_index",
        "_type": "my_type",
        "_id": "3",
        "_score": 0.44194174,
        "_source": {
          "title": "The quick brown fox jumps over the quick dog"
        }
      },
      {
        "_index": "my_index",
        "_type": "my_type",
        "_id": "2",
        "_score": 0.3125,
        "_source": {
          "title": "The quick brown fox jumps over the lazy dog"
        }
      }
    ]
  }
}
```
匹配查询的步骤  
 - 检查要查字段的type   
title字段是全文(analyzed) string字段，即意味着query string也应该被分词。
 - 分析 query string
用的是标准分词器，结果是一个单一条件 quick
 - 找到匹配的文档
 - 计算每个文档的得分  
  TF  IDF 每个字段的长度(越短的字段相关性越高)具体看[What is Relevance?](https://www.elastic.co/guide/en/elasticsearch/guide/2.x/relevance-intro.html]) 


提高精确性  
原来的查询方式

```
GET /my_index/my_type/_search
{
  "query": {
    "match": {
      "title": "BROWN DOG!"
    }
  }
}
```
新的方式

```
GET /my_index/my_type/_search
{
  "query": {
    "match": {
      "title": {
        "query": "BROWN DOG!",
        "operator": "and"
      }
    }
  }
}
```
匹配查询接受operator参数，默认的operator参数是or，可以改成and的形式  
你也可以自己指定精确度，使用`minimum_should_match`参数，表示最少匹配多少百分比。
如

```
GET /my_index/my_type/_search
{
  "query": {
    "match": {
      "title": {
        "query": "quick brown dog",
        "minimum_should_match": "75%"
      }
    }
  }
}
```
这个参数是很灵活的，用户输入的条件的项数不同则应用不同的规则，详细请看[Minimum Should Match](https://www.elastic.co/guide/en/elasticsearch/reference/current/query-dsl-minimum-should-match.html#query-dsl-minimum-should-match)   
联合查询  

```
GET /my_index/my_type/_search
{
  "query": {
    "bool": {
      "must":     { "match": { "title": "quick" }},
      "must_not": { "match": { "title": "lazy"  }},
      "should": [
                  { "match": { "title": "brown" }},
                  { "match": { "title": "dog"   }}
      ]
    }
  }
}
```
结果

```
{
  "hits": [
     {
        "_id":      "3",
        "_score":   0.70134366, 
        "_source": {
           "title": "The quick brown fox jumps over the quick dog"
        }
     },
     {
        "_id":      "1",
        "_score":   0.3312608,
        "_source": {
           "title": "The quick brown fox"
        }
     }
  ]
}
```
在使用should时，查询的结果可以没有should的结果，只是得分比较低。
也是可以控制精确度的，也是使用`minimum_should_match`

```
GET /my_index/my_type/_search
{
  "query": {
    "bool": {
      "should": [
        { "match": { "title": "brown" }},
        { "match": { "title": "fox"   }},
        { "match": { "title": "dog"   }}
      ],
      "minimum_should_match": 2 
    }
  }
}
```
怎样使用bool查询  
下面的两个查询的结果的一样的
```markdown
{
    "match": { "title": "brown fox"}
}
```

```markdown
{
  "bool": {
    "should": [
      { "term": { "title": "brown" }},
      { "term": { "title": "fox"   }}
    ]
  }
}
```

```markdown
{
    "match": {
        "title": {
            "query":    "brown fox",
            "operator": "and"
        }
    }
}
```

```markdown
{
  "bool": {
    "must": [
      { "term": { "title": "brown" }},
      { "term": { "title": "fox"   }}
    ]
  }
}
```
主要看一下多字段查询

```
{
    "match": {
        "title": {
            "query":                "quick brown fox",
            "minimum_should_match": "75%"
        }
    }
}
```

```
{
  "bool": {
    "should": [
      { "term": { "title": "brown" }},
      { "term": { "title": "fox"   }},
      { "term": { "title": "quick" }}
    ],
    "minimum_should_match": 2 
  }
}
```
增强查询子句条款(Boosting Query Clauses)  
 - 其目的就是提高某个查询条件的权重，以更多的影响到查询结果的排名。 使用`boost`参数

```
GET /_search
{
    "query": {
        "bool": {
            "must": {
                "match": {  
                    "content": {
                        "query":    "full text search",
                        "operator": "and"
                    }
                }
            },
            "should": [
                { "match": {
                    "content": {
                        "query": "Elasticsearch",
                        "boost": 3 
                    }
                }},
                { "match": {
                    "content": {
                        "query": "Lucene",
                        "boost": 2 
                    }
                }}
            ]
        }
    }
}
``` 
结果

```
  "hits": {
    "total": 4,
    "max_score": 0.783625,
    "hits": [
      {
        "_index": "my_index",
        "_type": "my_type",
        "_id": "4",
        "_score": 0.783625,
        "_source": {
          "content": "Full text search with Lucene and Elasticsearch"
        }
      },
      {
        "_index": "my_index",
        "_type": "my_type",
        "_id": "3",
        "_score": 0.40938914,
        "_source": {
          "content": "Full text search with Elasticsearch"
        }
      },
      {
        "_index": "my_index",
        "_type": "my_type",
        "_id": "2",
        "_score": 0.30934063,
        "_source": {
          "content": "Full text search with Lucene"
        }
      },
      {
        "_index": "my_index",
        "_type": "my_type",
        "_id": "1",
        "_score": 0.05462181,
        "_source": {
          "content": "Full text search is great"
        }
      }
    ]
  }
}
```
NOTE boost可以大于1也可以小于1，但其效果并不是线性的，只能说boost越大，_score就越高。具体算法就需要专门的书籍了。  
控制分词   
具体在不同环境中其有不同的默认分词器，详细请看[Controlling Analysis](https://www.elastic.co/guide/en/elasticsearch/guide/2.x/_controlling_analysis.html)  
相关性是碎了！(Relevance Is Broken!)  
在多字段查询中，我们为什么只给我们创建的索引仅仅一个主分片  
多主出现的问题是在我们在一小部分文档中做一个简单的查询，结果，相关性低出现在了相关性高的结果的前面。
 -  相关性 TF/IDF term frequency/inverse document frequency  
 - `TF` 在当前文档中，查询条件出现的次数，`出现次数越多，这个文档的相关性就越强。`
 - `IDF` 在所有索引文档中，查询条件在文档中的出现的总和在索引文档中占的一个百分比。`出现的越频繁，权重就越轻。`
 -  但是，出于性能的原因，es不在所有文档的索引中计算IDF，换句话说，每个分片计算包含在该分片内的本地IDF。导致的问题是同样的条件在不同的分片上其出现的频率是不同的，那么ID/IDF也就不一样了。而我们在进行得分排名时，是一个总的得分。因此就会出现错误的结果。
挽救的方法是在查询时加一个 '?search_type=dfs_query_then_fetch`条件。  

NOTE 建议不要在产品环境下使用。

