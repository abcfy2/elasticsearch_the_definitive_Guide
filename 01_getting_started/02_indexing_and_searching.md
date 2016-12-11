# 索引(Indexing)文档
存储数据到ES的操作称为*索引*(indexing)。一个ES集群可以包含多个*索引*(indices)，每个索引包含多个*类型*(types)，每个类型可以存储多个*文档*(documents)，每个文档那个包含多个*字段*(fields)。

## index vs index vs index
ES术语中有多个index出现，这里做一下词义辨析。
- `Index (noun)`: 做名词时，类似与关系性数据库的*数据库*概念，存储着相关的文档。复数形式为*indices*或*indexes*
- `Index (verb)`: ***To index a document*** is to store a document in an ***index (noun)*** so that it can be retrieved and queried. It is much like the **INSERT** keyword in SQL except that, if the document already exists, the new document would replace the old.
- `Inverted index`: 倒排索引，详情参见官方文档: https://www.elastic.co/guide/en/elasticsearch/guide/current/inverted-index.html

下面的范例将在`megacorp`这个index的`employee`类型存储文档:
```json
PUT /megacorp/employee/1
{
    "first_name" : "John",
    "last_name" :  "Smith",
    "age" :        25,
    "about" :      "I love to go rock climbing",
    "interests": [ "sports", "music" ]
}
```

`/megacorp/employee/1`包含了三个信息:
- `megacorp`: The index name
- `employee`: The type name
- `1`: The ID of this particular employee

ES会尝试解析这个json文档，猜测字段类型之后进行索引。

添加更多的文档:
```json
PUT /megacorp/employee/2
{
    "first_name" :  "Jane",
    "last_name" :   "Smith",
    "age" :         32,
    "about" :       "I like to collect rock albums",
    "interests":  [ "music" ]
}

PUT /megacorp/employee/3
{
    "first_name" :  "Douglas",
    "last_name" :   "Fir",
    "age" :         35,
    "about":        "I like to build cabinets",
    "interests":  [ "forestry" ]
}
```

## 取出文档
```json
GET /megacorp/employee/1
{
  "_index" :   "megacorp",
  "_type" :    "employee",
  "_id" :      "1",
  "_version" : 1,
  "found" :    true,
  "_source" :  {
      "first_name" :  "John",
      "last_name" :   "Smith",
      "age" :         25,
      "about" :       "I love to go rock climbing",
      "interests":  [ "sports", "music" ]
  }
}
```

使用`GET`方法请求文档的地址即可，返回的数据中包含一些元数据，同时，原始存储文档将会在`_source`这个字段中。

> **TIP**: 把`PUT`换成`GET`就可以取出对应的数据，还可以用`DELETE`删除指定文档，`HEAD`方法可以判断文档是否存在，想要替换文档，只需要再用`PUT`方法即可

# 搜索文档
`GET`方法可以简单直接得到一个具体的文档，当我们想做简单的搜索时，使用以下请求:
```json
GET /megacorp/employee/_search
{
   "took":      6,
   "timed_out": false,
   "_shards": { ... },
   "hits": {
      "total":      3,
      "max_score":  1,
      "hits": [
         {
            "_index":         "megacorp",
            "_type":          "employee",
            "_id":            "3",
            "_score":         1,
            "_source": {
               "first_name":  "Douglas",
               "last_name":   "Fir",
               "age":         35,
               "about":       "I like to build cabinets",
               "interests": [ "forestry" ]
            }
         },
         {
            "_index":         "megacorp",
            "_type":          "employee",
            "_id":            "1",
            "_score":         1,
            "_source": {
               "first_name":  "John",
               "last_name":   "Smith",
               "age":         25,
               "about":       "I love to go rock climbing",
               "interests": [ "sports", "music" ]
            }
         },
         {
            "_index":         "megacorp",
            "_type":          "employee",
            "_id":            "2",
            "_score":         1,
            "_source": {
               "first_name":  "Jane",
               "last_name":   "Smith",
               "age":         32,
               "about":       "I like to collect rock albums",
               "interests": [ "music" ]
            }
         }
      ]
   }
}
```

和获取一个具体的文档类似，最后不用ID，而是换成了`_search`这个endpoint。搜索的结果包含在`hits`这个数组中。默认情况下，搜索只会返回前10个结果。

通过追加`q=`参数，可以添加简单查询条件:
```json
GET /megacorp/employee/_search?q=last_name:Smith
{
   ...
   "hits": {
      "total":      2,
      "max_score":  0.30685282,
      "hits": [
         {
            ...
            "_source": {
               "first_name":  "John",
               "last_name":   "Smith",
               "age":         25,
               "about":       "I love to go rock climbing",
               "interests": [ "sports", "music" ]
            }
         },
         {
            ...
            "_source": {
               "first_name":  "Jane",
               "last_name":   "Smith",
               "age":         32,
               "about":       "I like to collect rock albums",
               "interests": [ "music" ]
            }
         }
      ]
   }
}
```

> 更多关于*Search Lite*的语法与限制，请参考官方文档: https://www.elastic.co/guide/en/elasticsearch/guide/current/search-lite.html#query-string-query

## Search with Query DSL
Search Lite便于命令行下进行快速查找，但是功能有限。ES提供了一个更强大，更灵活的*Query DSL*。
```json
GET /megacorp/employee/_search
{
    "query" : {
        "match" : {
            "last_name" : "Smith"
        }
    }
}
```

## 更复杂的查询
查询年龄在30岁以上的员工:
```json
GET /megacorp/employee/_search
{
    "query" : {
        "filtered" : {
            "filter" : {
                "range" : {
                    "age" : { "gt" : 30 }
                }
            },
            "query" : {
                "match" : {
                    "last_name" : "smith"
                }
            }
        }
    }
}
```

返回结果:
```json
{
   ...
   "hits": {
      "total":      1,
      "max_score":  0.30685282,
      "hits": [
         {
            ...
            "_source": {
               "first_name":  "Jane",
               "last_name":   "Smith",
               "age":         32,
               "about":       "I like to collect rock albums",
               "interests": [ "music" ]
            }
         }
      ]
   }
}
```

## 全文搜索(Full-Text Search)
```json
GET /megacorp/employee/_search
{
    "query" : {
        "match" : {
            "about" : "rock climbing"
        }
    }
}

{
   ...
   "hits": {
      "total":      2,
      "max_score":  0.16273327,
      "hits": [
         {
            ...
            "_score":         0.16273327,
            "_source": {
               "first_name":  "John",
               "last_name":   "Smith",
               "age":         25,
               "about":       "I love to go rock climbing",
               "interests": [ "sports", "music" ]
            }
         },
         {
            ...
            "_score":         0.016878016,
            "_source": {
               "first_name":  "Jane",
               "last_name":   "Smith",
               "age":         32,
               "about":       "I like to collect rock albums",
               "interests": [ "music" ]
            }
         }
      ]
   }
}
```

> 全文检索会返回一个`_score`字段，返回一个分数。默认按照得分由高到低排序

### 短语检索(Phrase Search)
和全文检索相比，短语检索是精确匹配。
```json
GET /megacorp/employee/_search
{
    "query" : {
        "match_phrase" : {
            "about" : "rock climbing"
        }
    }
}

{
   ...
   "hits": {
      "total":      1,
      "max_score":  0.23013961,
      "hits": [
         {
            ...
            "_score":         0.23013961,
            "_source": {
               "first_name":  "John",
               "last_name":   "Smith",
               "age":         25,
               "about":       "I love to go rock climbing",
               "interests": [ "sports", "music" ]
            }
         }
      ]
   }
}
```

### 高亮搜索结果
查询条件添加一个`highlight`参数即可，会将ES匹配到的结果以`<em></em>`标签高亮显示出来。
```json
GET /megacorp/employee/_search
{
    "query" : {
        "match_phrase" : {
            "about" : "rock climbing"
        }
    },
    "highlight": {
        "fields" : {
            "about" : {}
        }
    }
}

{
   ...
   "hits": {
      "total":      1,
      "max_score":  0.23013961,
      "hits": [
         {
            ...
            "_score":         0.23013961,
            "_source": {
               "first_name":  "John",
               "last_name":   "Smith",
               "age":         25,
               "about":       "I love to go rock climbing",
               "interests": [ "sports", "music" ]
            },
            "highlight": {
               "about": [
                  "I love to go <em>rock</em> <em>climbing</em>"
               ]
            }
         }
      ]
   }
}
```

> 更多详情参考官方文档: https://www.elastic.co/guide/en/elasticsearch/reference/current/search-request-highlighting.html
