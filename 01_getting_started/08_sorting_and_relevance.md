# 排序和相关性
默认情况下，返回结果按照相关性排序，即按照`_score`降序排列。`_score`是一个`float`类型的值。

## sorting
有时候你不会得到一个有意义的分数，例如以下这个查询:
```json
GET /_search
{
    "query" : {
        "bool" : {
            "filter" : {
                "term" : {
                    "user_id" : 1
                }
            }
        }
    }
}
```

由于使用了`filter`，因此不会加分，最终导致每个文档的得分都为0。此时匹配到的文档将以随机顺序排列。

如果相关性得分0分会影响你的逻辑的话，可以使用`constant_score`代替`bool`:
```json
GET /_search
{
    "query" : {
        "constant_score" : {
            "filter" : {
                "term" : {
                    "user_id" : 1
                }
            }
        }
    }
}
```

`constant_score`和`bool`的参数及用法几乎一样，唯一不同的是`constant_score`正如字面意思一样，固定分数，不会因为`match`而加分，性能和`bool`完全一样，可以使查询更清晰明了。`constant_score`默认固定分数为`1`。

## 按字段值排序
```json
GET /_search
{
    "query" : {
        "bool" : {
            "filter" : { "term" : { "user_id" : 1 }}
        }
    },
    "sort": { "date": { "order": "desc" }}
}

"hits" : {
    "total" :           6,
    "max_score" :       null, 
    "hits" : [ {
        "_index" :      "us",
        "_type" :       "tweet",
        "_id" :         "14",
        "_score" :      null, 
        "_source" :     {
             "date":    "2014-09-24",
             ...
        },
        "sort" :        [ 1411516800000 ] 
    },
    ...
}
```

由于`_score`不参与排序，因此不计算该值，如果你非要计算`_score`，可以设置`track_scores`为`true`。

排序的最简形式:
```json
    "sort": "number_of_children"
```

字段值将按照升序排序，`_score`值将按照降序排序。

## 多级排序
```json
GET /_search
{
    "query" : {
        "bool" : {
            "must":   { "match": { "tweet": "manage text search" }},
            "filter" : { "term" : { "user_id" : 2 }}
        }
    },
    "sort": [
        { "date":   { "order": "desc" }},
        { "_score": { "order": "desc" }}
    ]
}
```

数组顺序很重要，这将决定排序优先级。

> **NOTE**: Query-string search也支持自定义排序，使用`sort`参数即可:
> ```json
> GET /_search?sort=date:desc&sort=_score&q=search
>```

## 多值字段排序
多指字段本质上是无序的，选用哪个值进行排序？

对于数字和`date`类型的字段，可以通过使用`max`,`min`,`avg`或`sum` *sort mode*缩减为单一值。
```json
"sort": {
    "dates": {
        "order": "asc",
        "mode":  "min"
    }
}
```

## 字符串排序
涉及到分词问题，参见文档: https://www.elastic.co/guide/en/elasticsearch/guide/current/multi-fields.html

## 相关性
影响相关性的因素:
- `Term frequency`: How often does the term appear in the field? The more often, the more relevant. A field containing five mentions of the same term is more likely to be relevant than a field containing just one mention.
- `Inverse document frequency`: How often does each term appear in the index? The more often, the less relevant. Terms that appear in many documents have a lower weight than more-uncommon terms.
- `Field-length norm`: How long is the field? The longer it is, the less likely it is that words in the field will be relevant. A term appearing in a short title field carries more weight than the same term appearing in a long content field.

通过`explain`参数可以看到这个`_score`是如何计算的。

要想知道一个文档为什么被匹配到，可以给具体的文档使用`explain` API:
```json
GET /us/tweet/12/_explain
{
   "query" : {
      "bool" : {
         "filter" : { "term" :  { "user_id" : 2           }},
         "must" :  { "match" : { "tweet" :   "honeymoon" }}
      }
   }
}
```

和上面的一样，但是多了`description`元素，描述为何没有匹配到:

    "failure to match filter: cache(user_id:[2 TO 2])"

也就是说，`user_id` filter阻止了文档被匹配到。

