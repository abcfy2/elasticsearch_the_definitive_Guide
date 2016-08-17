# Full-Body Search
在Search DSL中，需要通过GET方法带Body请求:
```json
GET /_search
{} 

GET /index_2014*/type1,type2/_search
{}

GET /_search
{
  "from": 30,
  "size": 10
}
```

> 带Body的GET请求看上去可能很奇怪，某些语言的HTTP类库甚至不支持`GET`请求带Body，如JavaScript。但是这实际是符合[RFC 7231](https://tools.ietf.org/html/rfc7231#page-24)标准的，但是标准并未定义`GET`带Body应该作何种响应。所以基于这种原因，某些HTTP Server支持，某些不支持，尤其是代理服务器。ES开发者认为`GET`相比`POST`更合语义，但是由于某些类库不支持`GET`带Body，因此某些API也支持`POST`请求。
> 如:
> ```json
> POST /_search
> {
>   "from": 30,
>   "size": 10
> }
> ```

## Query DSL
要使用Query DSL，给`query`字段传递一个查询条件即可:
```json
GET /_search
{
    "query": YOUR_QUERY_HERE
}
```

空query(`{}`)等价于使用`match_all`:
```json
GET /_search
{
    "query": {
        "match_all": {}
    }
}
```

## Query结构
一个Query通常结构如下:
```json
{
    QUERY_NAME: {
        ARGUMENT: VALUE,
        ARGUMENT: VALUE,...
    }
}
```

如果引用了一个特定的字段，还应该有如下的结构:
```json
{
    QUERY_NAME: {
        FIELD_NAME: {
            ARGUMENT: VALUE,
            ARGUMENT: VALUE,...
        }
    }
}
```

例如，可以使用`match`查询`tweet`字段是否提到过`elasticsearch`:
```json
{
    "match": {
        "tweet": "elasticsearch"
    }
}
```

完整的查询就像这样:
```json
GET /_search
{
    "query": {
        "match": {
            "tweet": "elasticsearch"
        }
    }
}
```

## 组合多个查询条件
*Query clauses*可以组合简单查询条件成为复杂查询:
- *Leaf clauses* (like the `match` clause) that are used to compare a field (or fields) to a query string.
- *Compound clauses* that are used to combine other query clauses. For instance, a `bool` clause allows you to combine other clauses that either `must` match, `must_not` match, or `should` match if possible. They can also include non-scoring, filters for structured search:
```json
{
    "bool": {
        "must":     { "match": { "tweet": "elasticsearch" }},
        "must_not": { "match": { "name":  "mary" }},
        "should":   { "match": { "tweet": "full text" }},
        "filter":   { "range": { "age" : { "gt" : 30 }} }
    }
}
```

## Query and Filters
ES的Query DSL是一组查询条件的集合。每一组查询都可以分为`filtering` context and `query` context.

`filtering`表示`non-scoring`或`filtering`查询，如"Does this document match?". The answer is always a simple, binary yes|no.

`query`指的是"scoring" query。这将判断文档是否匹配，以及文档如何匹配。

As a general rule, use query clauses for *full-text* search or for any condition that should affect the *relevance score*, and use filters for everything else.

几个常用的查询:
```json
{ "match_all": {}}

{ "match": { "age":    26           }}
{ "match": { "date":   "2014-09-01" }}
{ "match": { "public": true         }}
{ "match": { "tag":    "full_text"  }}

{
    "multi_match": {
        "query":    "full text search",
        "fields":   [ "title", "body" ]
    }
}

{
    "range": {
        "age": {
            "gte":  20,
            "lt":   30
        }
    }
}

{ "term": { "age":    26           }}
{ "term": { "date":   "2014-09-01" }}
{ "term": { "public": true         }}
{ "term": { "tag":    "full_text"  }}

{ "terms": { "tag": [ "search", "full_text", "nosql" ] }}

{
    "exists":   {
        "field":    "title"
    }
}
```

The `term` query is used to search by exact values, be they numbers, dates, Booleans, or `not_analyzed` exact-value string fields.

## 联合查询
使用`bool`查询将多个子查询联合在一起。包含以下几个属性:
- `must`: Clauses that *must* match for the document to be included.
- `must_not`: Clauses that *must not* match for the document to be included.
- `should`: If these clauses match, they increase the `_score`; otherwise, they have no effect. They are simply used to refine the relevance score for each document.
- `filter`: Clauses that *must* match, but are run in non-scoring, filtering mode. These clauses do not contribute to the score, instead they simply include/exclude documents based on their criteria.

每个子查询都会分别为每个文档独立的计算关联性得分，`bool`将每个子查询的得分进行合并汇总。

以下这个查询将查找`title`匹配字符串`how to make millions`，并且mark不是`spam`的文档。如果有文档属于`starred`，或者是`2014`年以后的，排名就比那些不匹配的高。如果同时满足这两种条件排名会更高:
```json
{
    "bool": {
        "must":     { "match": { "title": "how to make millions" }},
        "must_not": { "match": { "tag":   "spam" }},
        "should": [
            { "match": { "tag": "starred" }},
            { "range": { "date": { "gte": "2014-01-01" }}}
        ]
    }
}
```

> **TIP**: 如果没有`must`条件，那么`should`条件至少会匹配一个。但是，如果含有至少一个`must`，那么`should`条件可以无需匹配到。

如果不想让`date`影响到分数，我们可以用`filter`子句预限定文档范围:
```json
{
    "bool": {
        "must":     { "match": { "title": "how to make millions" }},
        "must_not": { "match": { "tag":   "spam" }},
        "should": [
            { "match": { "tag": "starred" }}
        ],
        "filter": {
          "range": { "date": { "gte": "2014-01-01" }} 
        }
    }
}
```

任何查询条件都可以以这种形式，简单的移动到`filter`子句中，自动转换为non-scoring查询。

如果你需要以多种不同形式查询，`bool`本身就可以放入`filter`作为non-scoring查询:
```json
{
    "bool": {
        "must":     { "match": { "title": "how to make millions" }},
        "must_not": { "match": { "tag":   "spam" }},
        "should": [
            { "match": { "tag": "starred" }}
        ],
        "filter": {
          "bool": { 
              "must": [
                  { "range": { "date": { "gte": "2014-01-01" }}},
                  { "range": { "price": { "lte": 29.99 }}}
              ],
              "must_not": [
                  { "term": { "category": "ebooks" }}
              ]
          }
        }
    }
}
```

## 检验查询
`validate-query` API可以验证一个查询是否合法:
```json
GET /gb/tweet/_validate/query
{
   "query": {
      "tweet" : {
         "match" : "really powerful"
      }
   }
}
```

```json
{
  "valid" :         false,
  "_shards" : {
    "total" :       1,
    "successful" :  1,
    "failed" :      0
  }
}
```

想要知道为什么查询不合法，给query string追加`explain`参数即可:
```json
GET /gb/tweet/_validate/query?explain 
{
   "query": {
      "tweet" : {
         "match" : "really powerful"
      }
   }
}
```

```json
{
  "valid" :     false,
  "_shards" :   { ... },
  "explanations" : [ {
    "index" :   "gb",
    "valid" :   false,
    "error" :   "org.elasticsearch.index.query.QueryParsingException:
                 [gb] No query registered for [tweet]"
  } ]
}
```

`explain`有助于理解ES的查询过程:
```json
GET /_validate/query?explain
{
   "query": {
      "match" : {
         "tweet" : "really powerful"
      }
   }
}
```

```json
{
  "valid" :         true,
  "_shards" :       { ... },
  "explanations" : [ {
    "index" :       "us",
    "valid" :       true,
    "explanation" : "tweet:really tweet:powerful"
  }, {
    "index" :       "gb",
    "valid" :       true,
    "explanation" : "tweet:realli tweet:power"
  } ]
}
```