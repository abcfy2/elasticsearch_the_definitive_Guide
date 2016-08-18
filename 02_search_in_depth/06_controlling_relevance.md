###控制关联性#
在index级别修改权重  
 - 使用的参数为`indices_boost`  

```
GET /docs_2014_*/_search 
{
  "indices_boost": { 
    "docs_2014_10": 3,
    "docs_2014_09": 2
  },
  "query": {
    "match": {
      "text": "quick brown fox"
    }
  }
}
```
docs_2014_10增加到3，docs_2014_9增加到2，其他的保持默认值，即为1。  
NOT Quite Not    
搜索一个词apple，会出现公司，水果，各种食谱，我们想把公司放在前面，而把其他的放在后面，可以使用must加must_not,但这样就会把其他完全过滤了，解决这种问题就可以使用boosting Query  

```
GET /_search
{
  "query": {
    "boosting": {
      "positive": {
        "match": {
          "text": "apple"
        }
      },
      "negative": {
        "match": {
          "text": "pie tart fruit crumble tree"
        }
      },
      "negative_boost": 0.5
    }
  }
}
```
其中negative_boost必须小于1.0，像0.5就是对其得分减半。  
function_score查询  
该function_score查询是采取计分过程的控制的终极工具。它可以让你的函数应用到每个主查询，以改变或完全取代原有查询的_score。
 - weight
- field_value_factor  如 popularity或者votes
- random_score 
- Decay functions---linear,exp,gauss
- script_score  
以popularity为主  
一般blog的post请求  

```
PUT /blogposts/post/1
{
  "title":   "About popularity",
  "content": "In this post we will talk about...",
  "votes":   6
}
```
查询的方式

```
GET /blogposts/post/_search
{
  "query": {
    "function_score": { 
      "query": { 
        "multi_match": {
          "query":    "popularity",
          "fields": [ "title", "content" ]
        }
      },
      "field_value_factor": { 
        "field": "votes" 
      }
    }
  }
}
```
field_value_factor作为每个文档的主要查询，为了function_score可以工作，每个文档必须在votes字段有的一个数字。  
处理随机选择    
如选择酒店，结果有1,2,3,4,5，其_score的值可能都在2到3之间最多,5的很少，怎么处理2,3之间的结果。使同一个用户每次搜索的结果都相同，这就是随机一致性。解决方法。加上一个function,为random_score。

```
GET /_search
{
  "query": {
    "function_score": {
      "filter": {
        "term": { "city": "Barcelona" }
      },
      "functions": [
        {
          "filter": { "term": { "features": "wifi" }},
          "weight": 1
        },
        {
          "filter": { "term": { "features": "garden" }},
          "weight": 1
        },
        {
          "filter": { "term": { "features": "pool" }},
          "weight": 2
        },
        {
          "random_score": { 
            "seed":  "the users session id" 
          }
        }
      ],
      "score_mode": "sum"
    }
  }
}
```
random_score没有filter，因此可以应用于所有的文档中。  
使用脚本来评分  

```
GET /_search
{
  "function_score": {
    "functions": [
      { ...location clause... }, 
      { ...price clause... }, 
      {
        "script_score": {
          "params": { 
            "threshold": 80,
            "discount": 0.1,
            "target": 10
          },
          "script": "price  = doc['price'].value; margin = doc['margin'].value;
          if (price < threshold) { return price * margin / target };
          return price * (1 - discount) * margin / target;" 
        }
      }
    ]
  }
}
```
用户是会员级别，给一个折扣(discount)，得出每晚确切的价格(threshold),折扣之后，需要的押金(margin)。
TIP script_score函数提供了更大的灵活性，在脚本里，可以使用文档的字段，当前的-score，甚至是TF，IDF，字段长度标准。
