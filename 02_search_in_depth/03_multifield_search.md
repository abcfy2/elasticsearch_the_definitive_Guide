###多字段检索
最好字段

```
PUT /my_index/my_type/1
{
    "title": "Quick brown rabbits",
    "body":  "Brown rabbits are commonly seen."
}

PUT /my_index/my_type/2
{
    "title": "Keeping pets healthy",
    "body":  "My quick brown fox eats rabbits on a regular basis."
}
```
查询条件
```
{
    "query": {
        "bool": {
            "should": [
                { "match": { "title": "Brown fox" }},
                { "match": { "body":  "Brown fox" }}
            ]
        }
    }
}
```
结果
```
  "hits": {
    "total": 2,
    "max_score": 0.14326191,
    "hits": [
      {
        "_index": "my_index",
        "_type": "my_type",
        "_id": "1",
        "_score": 0.14326191,
        "_source": {
          "title": "Quick brown rabbits",
          "body": "Brown rabbits are commonly seen."
        }
      },
      {
        "_index": "my_index",
        "_type": "my_type",
        "_id": "2",
        "_score": 0.09256032,
        "_source": {
          "title": "Keeping pets healthy",
          "body": "My quick brown fox eats rabbits on a regular basis."
        }
      }
    ]
  }
}
```
bool查询中should实际上是对两个得分求个总和，然后再平均分为两份，平均的得分即为_score的值。而`dis_max`则是以最匹配查询的得分作为_score的值。

```
{
    "query": {
        "dis_max": {
            "queries": [
                { "match": { "title": "Brown fox" }},
                { "match": { "body":  "Brown fox" }}
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
        "_id":      "2",
        "_score":   0.21509302,
        "_source": {
           "title": "Keeping pets healthy",
           "body":  "My quick brown fox eats rabbits on a regular basis."
        }
     },
     {
        "_id":      "1",
        "_score":   0.12713557,
        "_source": {
           "title": "Quick brown rabbits",
           "body":  "Brown rabbits are commonly seen."
        }
     }
  ]
}
```
看一下下面这个查询条件

```markdown
{
    "query": {
        "dis_max": {
            "queries": [
                { "match": { "title": "Quick pets" }},
                { "match": { "body":  "Quick pets" }}
            ]
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
        "_score": 0.12713557, 
        "_source": {
           "title": "Quick brown rabbits",
           "body": "Brown rabbits are commonly seen."
        }
     },
     {
        "_id": "2",
        "_score": 0.12713557, 
        "_source": {
           "title": "Keeping pets healthy",
           "body": "My quick brown fox eats rabbits on a regular basis."
        }
     }
   ]
}
```
在dis_max 查询条件下两个文档都不完全匹配，所以，其得分结果相同，这时，可以使用tie_breaker 参数来修改这种情况，找到最匹配的。这个参数是对dis_max 下的match再进行评分，像是在dis_max和bool中间的方法。最终_score的值:   最匹配的_score+其他所有匹配_score的总和*tie_breaker

```
{
    "query": {
        "dis_max": {
            "queries": [
                { "match": { "title": "Quick pets" }},
                { "match": { "body":  "Quick pets" }}
            ],
            "tie_breaker": 0.3
        }
    }
}
```
结果

```
{
  "hits": [
     {
        "_id": "2",
        "_score": 0.14757764, 
        "_source": {
           "title": "Keeping pets healthy",
           "body": "My quick brown fox eats rabbits on a regular basis."
        }
     },
     {
        "_id": "1",
        "_score": 0.124275915, 
        "_source": {
           "title": "Quick brown rabbits",
           "body": "Brown rabbits are commonly seen."
        }
     }
   ]
}
```
NOTE 0表示使用最匹配的条件，1表示所有的匹配条件权值相同，比较好的取值范围是0.1--0.4，这样不会压倒最匹配的结果。  
