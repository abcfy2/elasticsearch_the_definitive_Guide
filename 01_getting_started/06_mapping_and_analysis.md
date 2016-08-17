# 映射和分词(Mapping and Analysis)
ES会自动猜测文档中每个字段的类型，然后生成一个映射(Mapping)。

```json
GET /gb/_mapping/tweet

{
   "gb": {
      "mappings": {
         "tweet": {
            "properties": {
               "date": {
                  "type": "date",
                  "format": "strict_date_optional_time||epoch_millis"
               },
               "name": {
                  "type": "string"
               },
               "tweet": {
                  "type": "string"
               },
               "user_id": {
                  "type": "long"
               }
            }
         }
      }
   }
}
```

没有`_all`字段，因为这是默认包含的，而且类型是`string`。

## Exact Values Versus Full Text
*Full Text*通常被称为非结构化数据，以人类可读的形式存储，但是计算机难以解析。

ES针对*Full Text*会首先*分析*文本，然后根据结构构建一个*inverted index*。

### Inverted Index
ES针对全文检索使用了一种称为*inverted index*的结构。一个倒排索引包含了一个列表，显示所有文档包含的不同的字词，以及这些字词都出现在哪些文档中。

一个示例:
```
Term      Doc_1  Doc_2
-------------------------
Quick   |       |  X
The     |   X   |
brown   |   X   |  X
dog     |   X   |
dogs    |       |  X
fox     |   X   |
foxes   |       |  X
in      |       |  X
jumped  |   X   |
lazy    |   X   |  X
leap    |       |  X
over    |   X   |  X
quick   |   X   |
summer  |       |  X
the     |   X   |
------------------------
```

比如我们想搜索`quick brown`，我们只需要找到每个term出现在哪个文档:
```
Term      Doc_1  Doc_2
-------------------------
brown   |   X   |  X
quick   |   X   |
------------------------
Total   |   2   |  1
```

关键词前面跟上`+`，`+Quick +fox`代表字词必须出现，而且必须按照给定顺序出现在文档中。

### 分词和分词器
倒排索引对于分词(Analysis)十分依赖，ES内置了分词器，也可以调用外部。

> https://www.elastic.co/guide/en/elasticsearch/guide/current/analysis-intro.html

## Mapping
ES支持以下几种简单类型:
- String: `string`
- Whole number: `byte`, `short`, `integer`, `long`
- Floating-point: `float`, `double`
- Boolean: `boolean`
- Date: `date`

当索引的新文档包含新字段时，ES按照[*动态模板*](https://www.elastic.co/guide/en/elasticsearch/guide/current/dynamic-mapping.html)规则猜测，按照以下规则:

|JSON type | Field type|
|----------|-----------|
|Boolean: `true` or `false` | `boolean` |
|Whole number: `123` | `long` |
|Floating point: `123.45` | `double` |
|String, valid date: `2014-09-15` | `date` |
|String: `foo` `bar` | `string` |

查看Mapping:

    GET /gb/_mapping/tweet

### 自定义Mapping
```json
{
    "number_of_clicks": {
        "type": "integer"
    }
}
```

对于`string`类型的字段最重要的属性有`index`和`analyzer`。

`index`有三个合法值:
- `analyzed`: First analyze the string and then index it. In other words, index this field as full text.
- `not_analyzed`: Index this field, so it is searchable, but index the value exactly as specified. Do not analyze it.
- `no`: Don’t index this field at all. This field will not be searchable.

默认`string`类型的字段的`index`属性是`analyzed`。如果你想作为exact value，需要设置为`not_analyzed`:
```json
{
    "tag": {
        "type":     "string",
        "index":    "not_analyzed"
    }
}
```

> **NOTE**: 其他简单类型的字段同样有`index`属性，但是只有两个合法值`no`和`not_analyzed`，不会用到分词

### 更新Mapping
当你第一次create index的时候可以指定mapping，也可以使用`/_mapping` endpoint增加或修改mapping。

> **NOTE**: 尽管Mapping可以修改，但不会影响到已存储的文档。

删除`gb`索引，然后在新建的时候指定mapping:
```json
DELETE /gb

PUT /gb 
{
  "mappings": {
    "tweet" : {
      "properties" : {
        "tweet" : {
          "type" :    "string",
          "analyzer": "english"
        },
        "date" : {
          "type" :   "date"
        },
        "name" : {
          "type" :   "string"
        },
        "user_id" : {
          "type" :   "long"
        }
      }
    }
  }
}
```

修改`tweet`的mapping:
```json
PUT /gb/_mapping/tweet
{
  "properties" : {
    "tag" : {
      "type" :    "string",
      "index":    "not_analyzed"
    }
  }
}
```

## 复杂类型
JSON是对象，可以包含一些嵌套类型，或者`null`等。

### Multivalue Fields
比如`tags`是一个数组:
```json
{ "tag": [ "search", "nosql" ]}
```

对于数组没有什么特别的mapping设置，数组类型字段可以包含任意多的值。每个值采取相同的分析规则(相同的`index`与`analyzer`等)。所以不能将不同的值类型放入一个数组中。

> **NOTE**: 数组是有索引的，可以被查询的多值字段。你不可以指定“第一个元素”或“最后一个元素”，只是把数组当成一袋子值即可。

### 空值
没办法在Lucene中存储一个`null`值，所以，包含`null`值的字段会被认为是一个空字段。

下面三种字段都是空，也都不会被索引:
```json
"null_value":               null,
"empty_array":              [],
"array_with_null_value":    [ null ]
```

### 多层级对象
对于多层级对象:
```json
{
    "tweet":            "Elasticsearch is very flexible",
    "user": {
        "id":           "@johnsmith",
        "gender":       "male",
        "age":          26,
        "name": {
            "full":     "John Smith",
            "first":    "John",
            "last":     "Smith"
        }
    }
}
```

ES会生成这样的Mapping:
```json
{
  "gb": {
    "tweet": { 
      "properties": {
        "tweet":            { "type": "string" },
        "user": { 
          "type":             "object",
          "properties": {
            "id":           { "type": "string" },
            "gender":       { "type": "string" },
            "age":          { "type": "long"   },
            "name":   { 
              "type":         "object",
              "properties": {
                "full":     { "type": "string" },
                "first":    { "type": "string" },
                "last":     { "type": "string" }
              }
            }
          }
        }
      }
    }
  }
}
```

多级Object表现就和根对象(*root object*)，只是缺少了某些元数据，如`_source`,`_all`等。

Lucence并不能理解这种嵌套对象，所以ES会转成这种对象:
```json
{
    "tweet":            [elasticsearch, flexible, very],
    "user.id":          [@johnsmith],
    "user.gender":      [male],
    "user.age":         [26],
    "user.name.full":   [john, smith],
    "user.name.first":  [john],
    "user.name.last":   [smith]
}
```

对于数组类型的对象:
```json
{
    "followers": [
        { "age": 35, "name": "Mary White"},
        { "age": 26, "name": "Alex Jones"},
        { "age": 19, "name": "Lisa Smith"}
    ]
}
```

会解析成这样:
```json
{
    "followers.age":    [19, 26, 35],
    "followers.name":   [alex, jones, lisa, smith, mary, white]
}
```

需要注意的是，数组在ES中只是*一袋值*，而不是*有序值*。所以你只能查出*有没有26岁的followers*，但是你不能得到这样准确的问题*有没有一个叫Alex Jones的26岁的followers?*

> 更多细节参考官方文档[Nested Objects](https://www.elastic.co/guide/en/elasticsearch/guide/current/nested-objects.html)
