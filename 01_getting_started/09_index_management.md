# 索引管理
## 创建索引
要手工创建索引，可以创建时从Body传递任何`settings`或`mappings`参数:
```json
PUT /my_index
{
    "settings": { ... any settings ... },
    "mappings": {
        "type_one": { ... any mappings ... },
        "type_two": { ... any mappings ... },
        ...
    }
}
```

如果想要避免自动创建索引，可以在`config/elasticsearch.yml`配置以下选项:
```yaml
action.auto_create_index: false
```

## 删除索引
```
DELETE /my_index

DELETE /index_one,index_two
DELETE /index_*

DELETE /_all
DELETE /*
```

> **NOTE**: 一条命令就干掉数据是非常危险的。如果你想避免因意外操作导致的删除，可以在`elasticsearch.yml`启用以下选项:
> ```yaml
> action.destructive_requires_name: true
> ```
> 
> 这将限制只能通过具体的名字删除，而不允许特殊的`_all`或通配符。

## 索引配置
详细的索引配置文档可以参考官方文档: https://www.elastic.co/guide/en/elasticsearch/reference/current/index-modules.html

这里将介绍两个最重要的参数:
- `number_of_shards`: The number of primary shards that an index should have, which defaults to `5`. This setting cannot be changed after index creation.
- `number_of_replicas`: The number of replica shards (copies) that each primary shard should have, which defaults to `1`. This setting can be changed at any time on a live index.

## Types and Mappings
*Type*在ES中是一组相似的文档的集合。*mapping*像数据库的schema一样，描述了type中的文档的字段或*properties*的属性，如每个字段的数据类型，以及这些字段在Lucence如何索引和存储。

Lucence没有类型概念，所有存储的数据都是二进制(像HBase一样)。数据类型解析在ES这一层完成。所以同名字段不能存入不同类型。

## Type Takeaways
Type并不适合将不同的数据存入。因为这将得到大量的稀疏字段，最终影响性能。

## The Root Object
最顶级的mapping称为*Root Object*，Root Object包含三个属性:
- *properties*
- 各种元数据，如`_type`,`_id`,`_source`
- Settings, which control how the dynamic detection of new fields is handled, such as `analyzer`, `dynamic_date_formats`, and `dynamic_templates`
- Other settings, which can be applied both to the root object and to fields of type `object`, such as `enabled`, `dynamic`, and `include_in_all`

### Properties
文档字段的配置，包含以下三个配置:
- `type`
- `index`
- `analyzer`

### Metadata: _source Field
默认情况下，ES存储在`_source`字段存储一份代表文档内容的JSON string。正如所有存储的字段一样，`_source`同样是写入磁盘之前先压缩的。

这几乎总是期望的功能，因为这有以下优点:
- 完整的文档总是可以直接从搜索结果中得到，无需反复从其他数据源获取。
- 局部的`update`请求在没有`_source`字段时无法工作。
- 当你的mapping修改了，并且你需要reindex数据的时候，你可以直接从ES中实现，无需从其他数据源重新回放数据(通常很慢)。
- 当你在`get`或`search`时不想看到整个文档时，单个字段可以从`_source`字段直接解出，返回给`get`或`search`请求。
- 很容易debug queries，因为你直接能看到每个完整的文档，而不需要从一个列表ID中猜测。

存储`_source`会占用额外的磁盘空间，如果不需要存储`_source`的话，可以使用以下设置:
```json
PUT /my_index
{
    "mappings": {
        "my_type": {
            "_source": {
                "enabled":  false
            }
        }
    }
}
```

在一个search请求中，你可以要求只返回指定字段，通过加入`_source`到请求体中:
```json
GET /_search
{
    "query":   { "match_all": {}},
    "_source": [ "title", "created" ]
}
```

这些字段的值将直接从`_source`字段中返回，而不是返回整个`_source`字段。

## Metadata: _all Field
在Search Lite中已经见识到了`_all`字段的作用，直接追加`?q=string`参数可以返回所有字段匹配的值。

`_all`在探索阶段很有用，尤其是在你仍不确定最后的文档结构的时候。`_all`字段在结构固定之后会用的越来越少，当你不再需要`_all`的时候，可以通过以下配置关闭:
```json
PUT /my_index/_mapping/my_type
{
    "my_type": {
        "_all": { "enabled": false }
    }
}
```

`_all`字段通过每个字段的`include_in_all`属性控制，默认值是`true`。如果只想保留部分字段的值到`_all`:
```json
PUT /my_index/my_type/_mapping
{
    "my_type": {
        "include_in_all": false,
        "properties": {
            "title": {
                "type":           "string",
                "include_in_all": true
            },
            ...
        }
    }
}
```

## Metadata: Document Identity
- `_id`: The string ID of the document
- `_type`: The type name of the document
- `_index`: The index where the document lives
- `_uid`: The `_type` and `_id` concatenated together as `type#id`

By default, the `_uid` field is stored (can be retrieved) and indexed (searchable). The `_type` field is indexed but not stored, and the `_id` and `_index` fields are neither indexed nor stored, meaning they don’t really exist.

## Dynamic Mapping
当ES遇到未知字段时，会根据*Dynamic Mapping*决定字段类型，然后把新字段加入type mapping。

你可以通过`dynamic`参数控制这个行为，这个参数有以下选项:
- `true`: Add new fields dynamically—the default
- `false`: Ignore new fields
- `strict`: Throw an exception if an unknown field is encountered

`dynamic`参数可以加到root object，或其他`object`类型的字段下。
```json
PUT /my_index
{
    "mappings": {
        "my_type": {
            "dynamic":      "strict", 
            "properties": {
                "title":  { "type": "string"},
                "stash":  {
                    "type":     "object",
                    "dynamic":  true 
                }
            }
        }
    }
}
```

## Customizing Dynamic Mapping
### date_detection
有时候字符串看的像日期，如`2014-01-01`。如果是第一个字段是日期样式:
```json
{ "note": "2014-01-01" }
```
但是后面上来的是个普通的字符串:
```json
{ "note": "Logged out" }
```

此时就会报错。

日期探测可以通过在root object关闭`date_detection`设置:
```json
PUT /my_index
{
    "mappings": {
        "my_type": {
            "date_detection": false
        }
    }
}
```

通过这个配置，`string`就总是`string`了。如果你需要`date`类型，只能手工添加了。

> **NOTE**: ES检测日期类型的算法: https://www.elastic.co/guide/en/elasticsearch/reference/current/dynamic-field-mapping.html#date-detection

### dynamic_templates
通过`dynamic_templates`配置，你可以完全控制新字段发现的方式，你甚至可以根据字段名或数值类型建立mapping。

每个template都有一个name，你可以用来描述模板的作用。也有一个`mapping`，指定应用何种映射，至少有一个参数(例如`match`)，定义哪些字段应用这种映射。

Templates按照顺序检测的，第一个匹配到的模板会被应用。

举个例子，我们需要对`string`字段建立两种模板:
- `es`: Field names ending in `_es` should use the `spanish` analyzer.
- `en`: All others should use the `english` analyzer.

我们首先应该把`es`模板放在前面，因为这比`en`匹配到的字段更具体:
```json
PUT /my_index
{
    "mappings": {
        "my_type": {
            "dynamic_templates": [
                { "es": {
                      "match":              "*_es", 
                      "match_mapping_type": "string",
                      "mapping": {
                          "type":           "string",
                          "analyzer":       "spanish"
                      }
                }},
                { "en": {
                      "match":              "*", 
                      "match_mapping_type": "string",
                      "mapping": {
                          "type":           "string",
                          "analyzer":       "english"
                      }
                }}
            ]
}}}
```

`match_mapping_type`参数允许你根据字段值类型应用模板。

`match`参数仅仅是匹配字段名,`path_match`可以匹配Object中完整的路径，如`address.*.name`可以匹配这样的field:
```json
{
    "address": {
        "city": {
            "name": "New York"
        }
    }
}
```

`unmatch`和`path_unmatch`参数可以用来取反匹配。

> 更多动态模板的配置项可以参考官方文档: https://www.elastic.co/guide/en/elasticsearch/reference/current/dynamic-mapping.html

## Default Mapping
通常情况下，同一个index下所有的type都有着非常类似的字段和设置。将这些通用的设置放到`_default_`配置中会更方便，不用每次创建新字段都重复一遍。`_default_`配置作为新type的模板使用，可以被type自己的模板覆盖掉。

例如，我们可以在所有的type中关闭`_all`字段，但是`blog`不用:
```json
PUT /my_index
{
    "mappings": {
        "_default_": {
            "_all": { "enabled":  false }
        },
        "blog": {
            "_all": { "enabled":  true  }
        }
    }
}
```

`_default_` mapping同样也是指定index级别的动态模板的好选择。

## Reindexing Your Data
mapping设置只能应用于新存储的字段，对于已有数据不受影响，这将可能影响到你的查询结果。

一个简单的办法是reindex: 使用新配置创建一个新index，然后从旧index拷贝所有文档到新index。

`_source`字段的一个优势就是，所有的文档已经存储在ES了，你不需要从数据库重建索引，这通常慢的多。

想要从旧索引中为所有文档高效的重建索引，使用[`scroll`](https://www.elastic.co/guide/en/elasticsearch/guide/current/scroll.html)从旧索引中获取一批文档，然后使用[`bulk`](https://www.elastic.co/guide/en/elasticsearch/guide/current/bulk.html) API推送至新索引中。

### Reindexing in Batches
```json
GET /old_index/_search?scroll=1m
{
    "query": {
        "range": {
            "date": {
                "gte":  "2014-01-01",
                "lt":   "2014-02-01"
            }
        }
    },
    "sort": ["_doc"],
    "size":  1000
}
```

## Index Aliases and Zero Downtime
一个index *alias*就像一个快捷方式或软连接，可以指向一个或多个索引，同样也可以用在任何期望index名的API中。Alias提供了巨大的灵活性:
- Switch transparently between one index and another on a running cluster
- Group multiple indices (for example, `last_three_months`)
- Create “views” on a subset of the documents in an index

这里我们利用索引别名从旧索引切换到新索引。

管理alias有两个endpoint: `_alias`操作单个，`_aliases`操作多个。

假设本例中，应用程序访问的索引是`my_index`，实际上，`my_index`是一个alias，指向当前实际的索引。我们为真实的索引加上版本号，`my_index_v1`, `my_index_v2`，以此类推。

首先我们先创建`my_index_v1`，然后创建alias `my_index`指向这个索引:
```json
PUT /my_index_v1 
PUT /my_index_v1/_alias/my_index 
```

你可以检测哪个索引指向了这个别名:
```json
GET /*/_alias/my_index
```

或者哪个alias指向了这个index:
```json
GET /my_index_v1/_alias/*
```

返回结果都是这样:
```json
{
    "my_index_v1" : {
        "aliases" : {
            "my_index" : { }
        }
    }
}
```

然后，我们决定改变fields的mappings，当然，我们无法修改已存在的mappings。现在以新配置创建`my_index_v2`:
```json
PUT /my_index_v2
{
    "mappings": {
        "my_type": {
            "properties": {
                "tags": {
                    "type":   "string",
                    "index":  "not_analyzed"
                }
            }
        }
    }
}
```

然后从`my_index_v1`重建索引到`my_index_v2`，按照上述Reindexing your data的步骤操作。

一个别名可以指向多个indices，所以我们需要指向新索引的同时删除旧索引的指向。这个修改需要原子性(atomic)，我们必须使用`_aliases` endpoint:
```json
POST /_aliases
{
    "actions": [
        { "remove": { "index": "my_index_v1", "alias": "my_index" }},
        { "add":    { "index": "my_index_v2", "alias": "my_index" }}
    ]
}
```

你的应用程序将透明的从`my_index_v1`切换至`my_index_v2`，不宕机。

> **TIP**: 即使你觉得目前的索引设计已经非常完美了，但是产品环境下依旧面临着修改的可能性。可以提前做好准备: 应用程序中使用别名而不是真正的索引名，那么你可以在需要时随时reindex你的数据。Alias是廉价的，应该被经常使用。
