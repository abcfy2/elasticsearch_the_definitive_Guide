###部分匹配
邮政编码和结构化数据  

```
PUT /my_index
{
  "mappings": {
    "address": {
      "properties": {
        "postcode": {
          "type": "string",
          "index": "not_analyzed"
        }
      }
    }
  }
}
```
插入数据

```
PUT /my_index/address/_bulk
{"index":{"_id":1}}
{"postcode":"W1V 3DG"}
{"index":{"_id":2}}
{"postcode":"W2F 8HW"}
{"index":{"_id":3}}
{"postcode":"W1F 7HW"}
{"index":{"_id":4}}
{"postcode":"WC1N 1LZ"}
{"index":{"_id":5}}
{"postcode":"SW5 0BE"}
```
查询条件

```
GET /my_index/address/_search
{
  "query": {
    "prefix": {
      "postcode": "W1"
    }
  }
}
```
查询的结果

```
  "hits": {
    "total": 2,
    "max_score": 1,
    "hits": [
      {
        "_index": "my_index",
        "_type": "address",
        "_id": "1",
        "_score": 1,
        "_source": {
          "postcode": "W1V 3DG"
        }
      },
      {
        "_index": "my_index",
        "_type": "address",
        "_id": "3",
        "_score": 1,
        "_source": {
          "postcode": "W1F 7HW"
        }
      }
    ]
  }
}
```
prefix 查询是低级别的查询，它在查询之前不会对query string分词。
Tip prefix查询和filter很相似，默认情况下不会计分，即给所有返回的结果_score 都是1，filter和prefix的唯一区别是prefix不会缓存。
为了支持动态前缀匹配，查询将执行以下操作： 
- 略过条件列表找到w1开始的位置
- 收集有关联文档的ID
- 移到下一个条件
- 如果条件仍然是从w1开始，查询重复第二步直到结束。  
`注意` prefix查询在查询条件少时能够自由的使用，但是在大规模使用时会对集群造成很大的压力。尝试使用长的前缀，以减少它们对集群的影响。这会减少对集群访问的项数。  
通配和正则查询  

```
GET /my_index/address/_search
{
  "query": {
    "wildcard": {
      "postcode": "W?F*HW"
    }
  }
}
```
结果为  

```
  "hits": {
    "total": 2,
    "max_score": 1,
    "hits": [
      {
        "_index": "my_index",
        "_type": "address",
        "_id": "2",
        "_score": 1,
        "_source": {
          "postcode": "W2F 8HW"
        }
      },
      {
        "_index": "my_index",
        "_type": "address",
        "_id": "3",
        "_score": 1,
        "_source": {
          "postcode": "W1F 7HW"
        }
      }
    ]
  }
}
```
正则表达式查询

```
GET /my_index/address/_search
{
  "query": {
    "regexp": {
      "postcode": "W[0-9].+"
    }
  }
}
```
查询的结果

```
  "hits": {
    "total": 3,
    "max_score": 1,
    "hits": [
      {
        "_index": "my_index",
        "_type": "address",
        "_id": "2",
        "_score": 1,
        "_source": {
          "postcode": "W2F 8HW"
        }
      },
      {
        "_index": "my_index",
        "_type": "address",
        "_id": "1",
        "_score": 1,
        "_source": {
          "postcode": "W1V 3DG"
        }
      },
      {
        "_index": "my_index",
        "_type": "address",
        "_id": "3",
        "_score": 1,
        "_source": {
          "postcode": "W1F 7HW"
        }
      }
    ]
  }
}
```
`注意` prefix，wildcard，regexp是条件操作，如果使用它们来查询已分词的字段，它们将检查该字段的每个条件，而不是把它们看做一个整体。
Ngrams 部分匹配  
如查询单词quick    
- Length 1 (unigram): [ q, u, i, c, k ]
- Length 2 (bigram): [ qu, ui, ic, ck ]
- Length 3 (trigram): [ qui, uic, ick ]
- Length 4 (four-gram): [ quic, uick ]
- Length 5 (five-gram): [ quick ]
例如

```
PUT /my_index
{
  "settings": {
    "number_of_shards": 1,
    "analysis": {
      "filter": {
        "autocomplete_filter": {
          "type": "edge_ngram",
          "min_gram": 1,
          "max_gram": 20
        }
      },
      "analyzer": {
        "autocomplete": {
          "type": "custom",
          "tokenizer": "standard",
          "filter": [
            "lowercase",
            "autocomplete_filter"
          ]
        }
      }
    }
  }
}
```
edge_ngram是一种type，我们自己定义了一个filter方法`autocomplete_filter`定义了最长和最短的模式。其本身是有分词的。

```
GET /my_index/_analyze
{
  "analyzer": "autocomplete",
  "text": "quick brown"
}
```
结果

```markdown
{
  "tokens": [
    {
      "token": "q",
      "start_offset": 0,
      "end_offset": 5,
      "type": "word",
      "position": 0
    },
    {
      "token": "qu",
      "start_offset": 0,
      "end_offset": 5,
      "type": "word",
      "position": 0
    },
    {
      "token": "qui",
      "start_offset": 0,
      "end_offset": 5,
      "type": "word",
      "position": 0
    },
    {
      "token": "quic",
      "start_offset": 0,
      "end_offset": 5,
      "type": "word",
      "position": 0
    },
    {
      "token": "quick",
      "start_offset": 0,
      "end_offset": 5,
      "type": "word",
      "position": 0
    },
    {
      "token": "b",
      "start_offset": 6,
      "end_offset": 11,
      "type": "word",
      "position": 1
    },
    {
      "token": "br",
      "start_offset": 6,
      "end_offset": 11,
      "type": "word",
      "position": 1
    },
    {
      "token": "bro",
      "start_offset": 6,
      "end_offset": 11,
      "type": "word",
      "position": 1
    },
    {
      "token": "brow",
      "start_offset": 6,
      "end_offset": 11,
      "type": "word",
      "position": 1
    },
    {
      "token": "brown",
      "start_offset": 6,
      "end_offset": 11,
      "type": "word",
      "position": 1
    }
  ]
}
```
这是一种比正则更高效的查询方式。
看下面的例子

```
PUT /my_index/_mapping/my_type
{
    "my_type": {
        "properties": {
            "name": {
                "type":     "string",
                "analyzer": "autocomplete"
            }
        }
    }
}
```

```
POST /my_index/my_type/_bulk
{ "index": { "_id": 1            }}
{ "name": "Brown foxes"    }
{ "index": { "_id": 2            }}
{ "name": "Yellow furballs" }
```
查询字段

```
GET /my_index/my_type/_search
{
    "query": {
        "match": {
            "name": "brown fo"
        }
    }
}
```
结果

```markdown
{

  "hits": [
     {
        "_id": "1",
        "_score": 1.5753809,
        "_source": {
           "name": "Brown foxes"
        }
     },
     {
        "_id": "2",
        "_score": 0.012520773,
        "_source": {
           "name": "Yellow furballs"
        }
     }
  ]
}
```
具体请看[Index-Time Search-as-You-Type](https://www.elastic.co/guide/en/elasticsearch/guide/2.x/_index_time_search_as_you_type.html)

