# 临近匹配
在全文检索中，match query 只是告诉我们在document中是否包含有我们要查询的，但是，这只是这个故事的一部分，其并没有告诉我们词语之间有任何的联系。  
如下面的两个不同的句子  
- Sue ate the alligator.
- The alligator ate Sue.
- Sue never goes anywhere without her alligator-skin purse.  

具体的内容请看[Proximity Matching](https://www.elastic.co/guide/en/elasticsearch/guide/2.x/proximity-matching.html)
# 短语匹配    
使用`match_phrase`     
```json
GET /my_index/my_type/_search
{
    "query": {
        "match_phrase": {
            "title": "quick brown fox"
        }
    }
}
```

也可以用match查询，给其加上type
```json
"match": {
    "title": {
        "query": "quick brown fox",
        "type":  "phrase"
    }
}
```
