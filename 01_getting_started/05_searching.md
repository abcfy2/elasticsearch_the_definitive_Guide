# 搜索
ES不止可以搜索存储的文档的数据，也可以搜索文档中的内容(全文检索)。

*search*可以做以下操作:
- A structured query on concrete fields like `gender` or `age`, sorted by a field like `join_date`, similar to the type of query that you could construct in SQL
- A full-text query, which finds all documents matching the search keywords, and returns them sorted by *relevance*(相关性)
- A combination of the two

在搜索前，需要了解以下三个概念:
- `Mapping`: How the data in each field is interpreted
- `Analysis`: How full text is processed to make it searchable
- `Query DSL`: The flexible, powerful query language used by Elasticsearch

> 准备测试数据: https://gist.github.com/clintongormley/8579281

## Empty Search
最基本的语法是跟空参，将返回所有结果:
```json
GET /_search

{
   "hits" : {
      "total" :       14,
      "hits" : [
        {
          "_index":   "us",
          "_type":    "tweet",
          "_id":      "7",
          "_score":   1,
          "_source": {
             "date":    "2014-09-17",
             "name":    "John Smith",
             "tweet":   "The Query DSL is really powerful and flexible",
             "user_id": 2
          }
       },
        ... 9 RESULTS REMOVED ...
      ],
      "max_score" :   1
   },
   "took" :           4,
   "_shards" : {
      "failed" :      0,
      "successful" :  10,
      "total" :       10
   },
   "timed_out" :      false
}
```

### hits
`hits`段是最重要的返回区段，包含了`total`属性，包含了多少个文档被匹配到。

`hits`数组包含了前10个匹配的文档。每个文档包含了`_index`,`_type`,`_id`,以及`_source`。

每个元素还有个`_score`，指的是相关性得分。默认情况将按照降序排列。这里我们没有指明查询条件，因此所有查询都是相关的，`_score`的值为`1`。

### took
执行的时间

### shards
告诉我们有多少个shard参与查询，多少成功多少失败。

### timeout
`timed_out`参数告诉我们查询是否超时。默认情况下，查询操作无超时设置。可以手动指定超时时间:

    GET /_search?timeout=10ms

> **Warning**: `timeout`参数并不中断查询的执行，只是告诉协调节点返回到目前为止的结果，然后关闭连接。使用`timeout`对于SLA(服务等级协议)很重要，但并不能用于中断长时间运行的查询。

## 多索引，多类型查询
可以指定index和type查询:
```
/_search
    Search all types in all indices
/gb/_search
    Search all types in the gb index
/gb,us/_search
    Search all types in the gb and us indices
/g*,u*/_search
    Search all types in any indices beginning with g or beginning with u
/gb/user/_search
    Search type user in the gb index
/gb,us/user,tweet/_search
    Search types user and tweet in the gb and us indices
/_all/user,tweet/_search
    Search types user and tweet in all indices
```

当你在单个index搜索时，ES会转发搜索请求到一个主或其他复制分片上，然后从每个分片上收集结果。从多个索引搜索时几乎和单个index搜索方式相同，仅仅是涉及到更多分片而已。

> **TIP**: 搜索有5个分片的一个索引和搜索5个每个都只有1个分片的索引，几乎时间上是等效的。

## 分页
类似于`SQL`中的`LIMIT`指令。ES也有`from`和`size`参数控制分页:
```json
GET /_search?size=5
GET /_search?size=5&from=5
GET /_search?size=5&from=10
```
- `size`: Indicates the number of results that should be returned, defaults to `10`
- `from`: Indicates the number of initial results that should be skipped, defaults to `0`

> **NOTE**: 避免分页太深或一次请求太多结果: https://www.elastic.co/guide/en/elasticsearch/guide/current/pagination.html

## Search Lite
Search Lite指的是使用query string的方式传递查询查询条件，而不是像Search DSL那样使用JSON body。

this query finds all documents of type `tweet` that contain the word `elasticsearch` in the `tweet` field:
```
GET /_all/tweet/_search?q=tweet:elasticsearch
```

The next query looks for john in the name field and mary in the tweet field. The actual query is just
    
    +name:john +tweet:mary

但是特殊字符需要编码:

    GET /_search?q=%2Bname%3Ajohn+%2Btweet%3Amary

`+`表示必须满足查询条件，类似的`-`表示取反。`+`和`-`可选，匹配到的越多，相关性越高。

### `_all`字段
简单返回所有包含关键字`mary`的文档:

    GET /_search?q=mary

当索引一个文档时，ES会将整个文档的值存储为一个长字符串，叫做`_all`字段。例如下面的文档:
```json
{
    "tweet":    "However did I manage before Elasticsearch?",
    "date":     "2014-09-14",
    "name":     "Mary Jones",
    "user_id":  1
}
```

相当于我们添加了一个`_all`字段，值为:

    "However did I manage before Elasticsearch? 2014-09-14 Mary Jones 1"

> 不需要`_all`字段时，可以关闭。相关内容: https://www.elastic.co/guide/en/elasticsearch/guide/current/root-object.html#all-field

### 更复杂的查询
这样的查询条件:
- `name`字段包含`mary` or `john`
- `date`字段大于`2014-09-10`
- `_all`字段包含`aggregations` or `geo`

对应的Search Lite表达式为:

    +name:(mary john) +date:>2014-09-10 +(aggregations geo)

经过http安全编码后:

    ?q=%2Bname%3A(mary+john)+%2Bdate%3A%3E2014-09-10+%2B(aggregations+geo)

Search Lite使用较为简单，但是遇到`-`, `:`, `/`, or `"`等字符可能会有误解。