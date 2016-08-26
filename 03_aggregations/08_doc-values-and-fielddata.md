# Doc Values and Fielddata
## Doc Values
聚合使用一个叫*doc values* 的数据结构（在 [Doc Values Intro](https://www.elastic.co/guide/en/elasticsearch/guide/current/docvalues-intro.html) 里简单介绍）。Fielddata 通常是 Elasticsearch 集群中内存消耗最大的一部分，所以理解它的工作方式十分重要。

Doc values 的存在是因为倒排索引只对某些操作是高效的。倒排索引的优势在于查找包含某个项的文档，而对于从另外一个方向的相反操作并不高效，即：确定只出现在单个文档里的所有项。聚合需要这种次级的辅助访问模式。

对于以下倒排索引：
```
Term      Doc_1   Doc_2   Doc_3
------------------------------------
brown   |   X   |   X   |
dog     |   X   |       |   X
dogs    |       |   X   |   X
fox     |   X   |       |   X
foxes   |       |   X   |
in      |       |   X   |
jumped  |   X   |       |   X
lazy    |   X   |   X   |
leap    |       |   X   |
over    |   X   |   X   |   X
quick   |   X   |   X   |   X
summer  |       |   X   |
the     |   X   |       |   X
------------------------------------
```

如果我们想要获得任何包含 `brown` 这个词的完整列表，我们会创建如下查询：
```json
GET /my_index/_search
{
  "query" : {
    "match" : {
      "body" : "brown"
    }
  },
  "aggs" : {
    "popular_terms": {
      "terms" : {
        "field" : "body"
      }
    }
  }
}
```
查询部分简单又高效。倒排索引是根据项来排序的，所以我们首先在词项列表中找到 `brown`，然后扫描所有列，找到包含 `brown` 的文档，我们可以快速看到 `Doc_1` 和 `Doc_2` 包含 `brown` 这个 token。

然后，对于聚合部分，我们需要找到 `Doc_1` 和 `Doc_2` 里所有唯一的词项，用倒排索引做这件事情代价很高：我们会迭代索引里的每个词项并收集 `Doc_1` 和 `Doc_2` 列里面 token。这很慢而且难以扩展：随着词项和文档的数量增加，执行时间也会增加。

Fielddata 通过转置两者间的关系来解决这个问题。倒排索引将词项映射到包含它们的文档，而 fielddata 将文档映射到它们包含的词项：
```
Doc      Terms
-----------------------------------------------------------------
Doc_1 | brown, dog, fox, jumped, lazy, over, quick, the
Doc_2 | brown, dogs, foxes, in, lazy, leap, over, quick, summer
Doc_3 | dog, dogs, fox, jumped, over, quick, the
-----------------------------------------------------------------
```
当数据被转置之后，想要收集到 `Doc_1` 和 `Doc_2` 的唯一 token 会非常容易。获得每个文档行，获取所有的词项，然后求两个集合的并集。

因此，搜索和聚合是相互紧密缠绕的。搜索使用倒排索引查找文档，聚合收集用聚合 fielddata 里的数据，而它本身也是通过倒排索引生成的。
>**NOTE**:`Doc values`不仅可以用于聚合。任何需要查找某个文档包含的值的操作都必须使用它。除了聚合，还包括排序，访问字段值的脚本，父子关系处理（参见[Parent-Child Relationship](https://www.elastic.co/guide/en/elasticsearch/guide/current/parent-child.html)）。

## Deep Dive on Doc Values
(详细内容参考这里！)[https://www.elastic.co/guide/en/elasticsearch/guide/current/_deep_dive_on_doc_values.html]
理想情况下我们希望fielddata是全部常驻内存的，但是内存毕竟是有限的，因此需要增加节点来尽量多的保存fielddata数据。但是这样也造成了资源的浪费，在内存被充分利用的情况下，cpu等其他资源则可能是闲置的。如果采用了上述的配置，内存是限制住了，但是缺陷也是显而易见的。有没有一种访问速度快还不占用内存的fielddata的方案呢？有，doc values。
目前doc values的访问速度还是略慢于fielddata，大约10%-25%左右的样子。但是优势也是显而易见的：首先，磁盘存储而不是内存，这就不再多说了。其次，doc value是在index的时候创建的，fielddata需要在查询的时候加载（反转倒排索引），而doc value由于是index的时候创建完成了，因此在初始化过程要明显快于fielddata。
当然没有什么事情是完美的，doc values的优势自然也带来了不足，比如索引会变大，访问速度略慢于fielddata。但是我们真的会在乎访问速度的细微差别么？不尽然。doc values已经足够的高效，所以应用也许并不关注这点细微的差距。况且由此来会带来较快的垃圾回收等优势。
doc values目前还不能应用与analyzed string field。
doc value的启用也非常简单，只需要在mapping中对应的doc_values属性设置为true即可。
在可以遇见的将来doc value format将会成为es默认设置。

## 聚合与分析（Aggregations and Analysis）
有些聚合，比如 `terms` 桶，操作字符串字段。字符串字段可能是 `analyzed` 或 `not_analyzed`，那么问题来了，分析是怎么影响聚合的呢？

答案是影响“很多”，但可以通过一个示例来更好说明这点。首先索引一些代表美国各个州的文档：
```json
POST /agg_analysis/data/_bulk
{ "index": {}}
{ "state" : "New York" }
{ "index": {}}
{ "state" : "New Jersey" }
{ "index": {}}
{ "state" : "New Mexico" }
{ "index": {}}
{ "state" : "New York" }
{ "index": {}}
{ "state" : "New York" }
```
我们希望创建一个数据集里各个州的唯一列表，并且计数。简单，让我们使用 `terms` 桶：
```
GET /agg_analysis/data/_search
{
    "size" : 0,
    "aggs" : {
        "states" : {
            "terms" : {
                "field" : "state"
            }
        }
    }
}
```
得到结果：
```json
{
...
   "aggregations": {
      "states": {
         "buckets": [
            {
               "key": "new",
               "doc_count": 5
            },
            {
               "key": "york",
               "doc_count": 3
            },
            {
               "key": "jersey",
               "doc_count": 1
            },
            {
               "key": "mexico",
               "doc_count": 1
            }
         ]
      }
   }
}
```
这完全不是我们想要的！没有对州名计数，聚合计算了每个词的数目。背后的原因很简单：聚合是基于倒排索引创建的，倒排索引是 后置分析（post-analysis） 的。

当我们把这些文档加入到 Elasticsearch 中时，字符串 `“New York”` 被分析/分词成 `["new", "york"]` 。每个 token 都用来提取 fielddata 里的内容，所以我们最终看到 `new` 的数量而不是 `New York`。

这显然不是我们想要的行为，但幸运的是很容易修正它。

我们需要为 `state` 定义 multifield 并且设置成 `not_analyzed`。这样可以防止 `New York` 被分析，也意味着在聚合过程中它会以单个 token 的形式存在。让我们尝试完整的过程，但这次指定一个 `raw multifield`：
```json
DELETE /agg_analysis/
PUT /agg_analysis
{
  "mappings": {
    "data": {
      "properties": {
        "state" : {
          "type": "string",
          "fields": {
            "raw" : {
              "type": "string",
              "index": "not_analyzed"
            }
          }
        }
      }
    }
  }
}

POST /agg_analysis/data/_bulk
{ "index": {}}
{ "state" : "New York" }
{ "index": {}}
{ "state" : "New Jersey" }
{ "index": {}}
{ "state" : "New Mexico" }
{ "index": {}}
{ "state" : "New York" }
{ "index": {}}
{ "state" : "New York" }

GET /agg_analysis/data/_search
{
  "size" : 0,
  "aggs" : {
    "states" : {
        "terms" : {
            "field" : "state.raw"
        }
    }
  }
}
```
1. 这次我们显式映射 `state` 字段并包括一个 `not_analyzed` 辅字段。
2. 聚合针对 `state.raw` 字段而不是 `state`。

现在运行聚合，我们得到了合理的结果：
```json
{
...
   "aggregations": {
      "states": {
         "buckets": [
            {
               "key": "New York",
               "doc_count": 3
            },
            {
               "key": "New Jersey",
               "doc_count": 1
            },
            {
               "key": "New Mexico",
               "doc_count": 1
            }
         ]
      }
   }
}
```
在实际中，这样的问题很容易被察觉，我们的聚合会返回一些奇怪的桶，我们会记住分析的问题。总之，很少有在聚合中使用分析字段的实例。当我们疑惑时，只要增加一个 `multifield` 就能有两种选择。

## Analyzed strings and Fielddata
>**NOTE**:Historically, fielddata was the default for all fields, but Elasticsearch has been migrating towards doc values to reduce the chance of OOM. Analyzed strings are the last holdout where fielddata is still used. The goal is to eventually build a serialized data structure similar to doc values which can handle highly dimensional analyzed strings, obsoleting fielddata once and for all.

### 高基数内存的影响（High-Cardinality Memory Implications）
避免分析字段的另外一个原因就是：高基数字段在加载到 fielddata 时会消耗大量内存，分析的过程会经常（尽管不总是这样）生成大量的 token，这些 token 大多都是唯一的。这会增加字段的整体基数并且带来更大的内存压力。

有些类型的分析对于内存来说 极度 不友好，想想 n-gram 的分析过程，`New York` 会被 n-gram 成以下 token：
- `ne`
- `ew`
- `w`
- `y`
- `yo`
- `or`
- `rk`

可以想象 n-gramming 的过程是如何生成大量唯一 token 的，特别是在分析成段文本的时候。当这些数据加载到内存中，会轻而易举的将我们堆空间消耗殆尽。

所以，在进行跨字段聚合之前，花点时间验证一下字段是 not_analyzed，如果我们想聚合 analyzed 的字段，确保分析过程不会生成任何不必要的 token。

最后，一个字段是 analyzed 还是 not_analyzed 并不重要，字段里的唯一值越多，字段基数越高，就需要更多的内存。字符串字段特别是这样，因为每个唯一的字符串都必须保存在内存中，字符串越长，需要的内存越多。

## 限制内存使用（Limiting Memory Usage）
为了让聚合（或任何需要访问字段值的操作）更快，访问 fielddata 必须快速，这就是为什么将它载入内存的原因。但加载过多的数据到内存会导致垃圾回收变慢，因为 JVM 会尝试在堆中找到额外的空间，或甚至有可能导致 OutOfMemory 异常。

Elasticsearch 不仅仅将与查询匹配的文档载入到 fielddata 中，这可能会令我们感到吃惊，它还会将 索引内所有文档 的值加载，甚至是那些不同类型 `_type` 的文档！

逻辑是这样：如果查询会访问文档 X、Y 和 Z，那很有可能会在下一个查询中访问其他文档。将所有的信息一次加载，再将其维持在内存中的方式要比每次请求都扫描倒排索引的代价要低。

JVM 堆是有限资源，应该被合理利用。限制 fielddata 对堆使用的影响有多套机制，这些限制方式非常重要，因为堆栈的乱用会导致节点不稳定（感谢缓慢的垃圾回收机制），甚至导致节点宕机（通常伴随 OutOfMemory 异常）。
>## 选择堆大小（Choosing a Heap Size）
>在设置 Elasticsearch 堆大小时需要通过 `$ES_HEAP_SIZE` 环境变量应用两个规则：
- **不要超过可用 RAM 的 50%**:Lucene 能很好利用文件系统的缓存，它是通过系统内核管理的。如果没有足够的文件系统缓存空间，性能会收到影响。
- **不要超过 32GB**：如果堆大小小于 32 GB，JVM 可以利用指针压缩，这可以大大降低内存的使用：每个指针 4 字节而不是 8 字节。  

### Fielddata的大小（Fielddata Size）
`indices.fielddata.cache.size` 控制为 fielddata 分配的堆空间大小。当查询需要访问新字段值时，它会先将值加载到内存中，然后尝试把它们加入到 fielddata 。如果结果中 fielddata 大小超过了指定大小，其他的值会被剔除从而获得空间。

默认情况下，设置都是 *unbounded* ，Elasticsearch 永远都不会从 fielddata 中剔除数据。

这个默认设置是刻意选择的：fielddata 不是临时缓存。它是驻留内存里的数据结构，必须可以快速执行访问，而且构建它的代价十分高昂。如果每个请求都重载数据，性能会十分糟糕。

一个有界的大小会强制数据结构剔除数据。我们会看合适应该设置这个值，但请首先阅读以下警告：
>**警告**:
>- 这个设置是一个安全卫士，而非内存不足的解决方案。
>- 如果没有足够空间可以将 fielddata 保留在内存中，Elasticsearch 就会时刻从磁盘重载数据，并剔除其他数据以获得更多空间。内存的剔除机制会导致重度磁盘I/O，并且在内存中生成很多垃圾，这些垃圾必须在晚些时候被回收掉。

设想我们正在对日志进行索引，每天使用一个新的索引。通常我们只对过去一两天的数据感兴趣，尽管我们会保留老的索引，但我们很少需要查询它们。不过如果采用默认设置，旧索引的 fielddata 永远不会从缓存中剔除！fieldata 会保持增长直到 fielddata 发生断熔（参见断路器（[Circuit Breaker](https://www.elastic.co/guide/en/elasticsearch/guide/current/_limiting_memory_usage.html#circuit-breaker)）），这样我们就无法载入更多的 fielddata。

这个时候，我们被困在了死胡同。但我们仍然可以访问旧索引中的 fielddata，也无法加载任何新的值。相反，我们应该剔除旧的数据，并为新值获得更多空间。

为了防止发生这样的事情，可以通过在 `config/elasticsearch.yml` 文件中增加配置为 fielddata 设置一个上限：
```
indices.fielddata.cache.size:  20%
```
可以设置堆大小的百分比，也可以是某个值 5gb。

有了这个设置，最久未使用（LRU）的 fielddata 会被剔除为新数据腾出空间。

>**警告**:
>- 可能发现在线文档有另外一个设置：indices.fielddata.cache.expire 。
>- 这个设置永远都不会被使用！它很有可能在不久的将来被弃用。
>- 这个设置要求 Elasticsearch 剔除那些过期的 fielddata，不管这些值有没有被用到。
>- 这对性能是件很糟糕的事情。剔除会有消耗性能，它刻意的安排剔除方式，而没能获得任何回报。
>- 没有理由使用这个设置：我们不能从理论上假设一个有用的情形。目前，它的存在只是为了向前兼容。我们只在很有以前提到过这个设置，但不幸的是网上各种文章都将其作为一种性能调优的小窍门来推荐。
>- 它不是。永远不要使用！

## 监控 fielddata（Monitoring fielddata）
无论是仔细监控 fielddata 的内存使用情况，还是看有无数据被剔除都十分重要。高的剔除数可以预示严重的资源问题以及性能不佳的原因。

Fielddata 的使用可以被监控：
- per-index using the indices-stats API:
```
GET /_stats/fielddata?fields=*
```

- per-node using the nodes-stats API:
```
GET /_nodes/stats/indices/fielddata?fields=*
```

- Or even per-index per-node:
```
GET /_nodes/stats/indices/fielddata?level=indices&fields=*
```
使用设置 `?fields=*`，可以将内存使用分配到每个字段。

### 断路器（Circuit Breaker）
机敏的读者可能已经发现 fielddata 大小设置的一个问题。fielddata 大小是在数据加载之后检查的。如果一个查询试图加载比可用内存更多的信息到 fielddata 中会发生什么？答案很丑陋：我们会碰到 OutOfMemoryException 。

Elasticsearch 包括一个 fielddata 断熔器，这个设计就是为了处理上述情况。断熔器通过内部检查（字段的类型、基数、大小等等）来估算一个查询需要的内存。它然后检查要求加载的 fielddata 是否会导致 fielddata 的总量超过堆的配置比例。

如果估算查询的大小超出限制，就会触发断路器，查询会被中止并返回异常。这都发生在数据加载之前，也就意味着不会引起 OutOfMemoryException 。

>### 可用的断路器（Available Circuit Breakers）
>Elasticsearch 有一系列的断路器，它们都能保证内存不会超出限制：
>- indices.breaker.fielddata.limit: fielddata 断路器默认设置堆的 60% 作为 fielddata 大小的上限。
>- indices.breaker.request.limit: request 断路器估算需要完成其他请求部分的结构大小，例如创建一个聚合桶，默认限制是堆内存的 40%。
>- indices.breaker.total.limit: total 揉合 request 和 fielddata 断路器保证两者组合起来不会使用超过堆内存的 70%。

断路器的限制可以在文件 config/elasticsearch.yml 中指定，可以动态更新一个正在运行的集群：
```json
PUT /_cluster/settings
{
  "persistent" : {
    "indices.breaker.fielddata.limit" : "40%"
  }
}
```
这个限制是按对内存的百分比设置的。

最好为断路器设置一个相对保守点的值。记住 fielddata 需要与 request 断路器共享堆内存、索引缓冲内存和过滤器缓存。Lucene 的数据被用来构造索引，以及各种其他临时的数据结构。正因如此，它默认值非常保守，只有 60% 。过于乐观的设置可能会引起潜在的堆栈溢出（OOM）异常，这会使整个节点宕掉。

另一方面，过度保守的值只会返回查询异常，应用程序可以对异常做相应处理。异常比服务器崩溃要好。这些异常应该也能促进我们对查询进行重新评估：为什么单个查询需要超过堆内存的 60% 之多？
>**小贴士**:在 `Fielddata Size` 中，我们提过关于给 `fielddata` 的大小加一个限制，从而确保旧的无用 `fielddata` 被剔除的方法。`indices.fielddata.cache.size` 和 `indices.breaker.fielddata.limit` 之间的关系非常重要。如果断路器的限制低于缓存大小，没有数据会被剔除。为了能正常工作，断路器的限制要比缓存大小要高。

值得注意的是：断路器是根据总堆内存大小估算查询大小的，而非根据实际堆内存的使用情况。这是由于各种技术原因造成的（例如，堆可能看上去是满的但实际上可能只是在等待垃圾回收，这使我们难以进行合理的估算）。但作为终端用户，这意味着设置需要保守，因为它是根据总堆内存必要的，而不是可用堆内存。

## Fielddata 的过滤（Fielddata Filtering）
设想我们正在运行一个网站运行用户收听他们喜欢的歌曲。为了让他们可以更容易的管理自己的音乐库，用户可以为歌曲设置任何他们喜欢的标签，这样我们就会有很多歌曲被附上 rock（摇滚）、hiphop（嘻哈） 和 electronica（电音）这样的标签，但也会有些歌曲被附上 my_16th_birthday_favorite_anthem 这样的标签。

现在设想我们想要为用户展示每首歌曲最受欢迎的三个标签，很有可能 rock 这样的标签会排在三个中的最前面，而 my_16th_birthday_favorite_anthem 则不太可能得到评级。尽管如此，为了计算最受欢迎的标签，我们必须强制将这些一次性使用的项加载到内存中。

感谢 fielddata 过滤，我们可以控制这种状况。我们知道自己只对最流行的项感兴趣，所以我们可以简单地避免加载那些不太有意思的长尾项：
```json
PUT /music/_mapping/song
{
  "properties": {
    "tag": {
      "type": "string",
      "fielddata": {
        "filter": {
          "frequency": {
            "min":              0.01,
            "min_segment_size": 500  
          }
        }
      }
    }
  }
}
```
1. fielddata 关键字允许我们配置 fielddata 处理该字段的方式。
1. frequency 过滤器允许我们基于项频率过滤加载 fielddata。
1. 只加载那些至少在本段文档中出现 1% 的项。
1. 忽略任何少于 500 文档的段。

## 提前加载 fielddata（Preloading Fielddata）
Elasticsearch 加载内存 fielddata 的默认行为是 延迟加载 。当 Elasticsearch 接收一个需要使用某个特定字段 fielddata 的查询时，它会将索引中每个分段的完整字段内容加载到内存中。

对于小分段来说，这个过程的需要的时间可以忽略。但如果我们有些 5 GB 的分段，并希望加载 10 GB 的 fielddata 到内存中，这个过程可能会要数十秒。已经习惯亚秒响应的用户会突然遇到一个打击，网站明显无法响应。

有三种方式可以解决这个延时高峰：
- 预加载 fielddata
- 预加载全局序号
- 预热内存

所有的变化都基于同一概念：预加载 fielddata 这样在用户进行搜索时就不会碰到延迟高峰

### 预加载 fielddata（Eagerly Loading Fielddata）
第一个工具称为预加载（与默认的延迟加载相对）。随着新分段的创建（通过刷新、写入或合并等方式），启动字段预加载可以使那些对搜索不可见的分段里的 fielddata 提前加载。

这就意味着首次命中分段的查询不需要促发 fielddata 的加载，因为缓存内容已经被载入到内存。这也能避免用户碰到冷缓存加载时的延时高峰。

预加载是按字段启用的，所以我们可以控制具体哪个字段可以预先加载：
```json
PUT /music/_mapping/_song
{
  "tags": {
    "type": "string",
    "fielddata": {
      "loading" : "eager"
    }
  }
}
```
设置 `fielddata.loading: eager` 可以告诉 Elasticsearch 预先将此字段的内容载入内存中。

Fielddata 的载入可以使用 update-mapping API 对已有字段设置 lazy 或 eager 两种模式。
>**警告**：
>- 预加载只是简单的将载入 fielddata 的代价转移到索引刷新的时候，而不是查询时。
>- 内容多的分段会比内容少的分段需要更长的刷新时间。通常，内容多的分段是由那些已经对查询可见的小分段合并而成的，所以较慢的刷新时间也不是很重要。

### 全局序号（Global Ordinals）
有种可以用来降低字符串 fielddata 内存使用的技术叫做 序号。

设想我们有十亿文档，每个文档都有自己的 status 状态字段，状态总共有三种：status_pending、status_published、 status_deleted。如果我们为每个文档都保留其状态的完整字符串形式，那么每个文档就需要使用 14 到 16 字节，或总共 15 GB。

取而代之的是我们可以指定三个不同的字符串，对其排序、编号：0，1，2。
```
Ordinal | Term
-------------------
0       | status_deleted
1       | status_pending
2       | status_published
```
序号字符串在序号列表中只存储一次，每个文档只要使用数值编号的序号来替代它原始的值。
```
Doc     | Ordinal
-------------------------
0       | 1  # pending
1       | 1  # pending
2       | 2  # published
3       | 0  # deleted
```
这样可以将内存使用从 15 GB 降到 1 GB 以下！

但这里有个问题，记得 fielddata 是按分段来缓存的。如果一个分段只包含两个状态（status_deleted 和 status_published）那么结果中的序号（0 和 1）就会与包含所有三个状态的分段不一样。

如果我们尝试对 status 字段运行 terms 聚合，我们需要对实际字符串的值进行聚合，也就是说我们需要识别所有分段中相同的值。一个简单粗暴的方式就是对每个分段执行聚合操作，返回每个分段的字符串值，再将它们归纳得出完整的结果。尽管这样做可行，但会很慢而且对大量消耗 CPU。

取而代之的是使用一个被称为 全局序号 的结构。全局序号是一个构建在 fielddata 之上的数据结构，它只占用少量内存。唯一值是跨所有分段识别的，然后将它们存入一个序号列表中，正如我们描述过的那样。

现在，terms 聚合可以对全局序号进行聚合操作，将序号转换成真实字符串值的过程只会在聚合结束时发生一次。这会将聚合（和排序）的性能提高三到四倍。

#### 构建全局序号（Building global ordinals）
当然，天下没有免费的晚餐。全局序号分布在索引的所有段中，所以如果新增或删除一个分段时，需要对全局序号进行重建。重建需要读取每个分段的每个唯一项，基数越高（即存在更多的唯一项）这个过程会越长。

全局序号是构建在内存 fielddata 和 doc values 之上的。实际上，它们正是 doc values 性能表现不错的一个主要原因。

和 fielddata 加载一样，全局序号默认也是延迟构建的。首个需要访问索引内 fielddata 的请求会促发全局序号的构建。由于字段的基数不同，这会导致给用户带来显著延迟这一糟糕结果。一旦全局序号发生重建，仍会使用旧的全局序号，直到索引中的分段产生变化：在刷新、写入或合并之后。

#### 预建全局序号（Eager global ordinals）
单个字符串字段可以通过配置预先构建全局序号：
```json
PUT /music/_mapping/_song
{
  "song_title": {
    "type": "string",
    "fielddata": {
      "loading" : "eager_global_ordinals"
    }
  }
}
```
设置 eager_global_ordinals 也暗示着 fielddata 是预加载的。

正如 fielddata 的预加载一样，预构建全局序号发生在新分段对于搜索可见之前。
>**注意**:序号的构建只被应用于字符串。数值信息（integers（整数）、geopoints（地理经纬度）、dates（日期）等等）不需要使用序号映射，因为这些值自己本质上就是序号映射。

### 索引预热器（Index Warmers）
最后我们谈谈 索引预热器。预热器早于 fielddata 预加载和全局序号预加载之前出现，它们仍然尤其存在的理由。一个索引预热器允许我们指定一个查询和聚合须要在新分片对于搜索可见之前执行。这个想法是通过预先填充或预热缓存让用户永远无法遇到延迟的波峰。

原来，预热器最重要的用法是确保 fielddata 被预先加载，因为这通常是最耗时的一步。现在可以通过前面讨论的那些技术来更好的控制它，但是预热器还是可以用来预建过滤器缓存，当然我们也还是能选择用它来预加载 fielddata。

让我们注册一个预热器然后解释发生了什么：
```json
PUT /music/_warmer/warmer_1
{
  "query" : {
    "bool" : {
      "filter" : {
        "bool": {
          "should": [
            { "term": { "tag": "rock"        }},
            { "term": { "tag": "hiphop"      }},
            { "term": { "tag": "electronics" }}
          ]
        }
      }
    }
  },
  "aggs" : {
    "price" : {
      "histogram" : {
        "field" : "price",
        "interval" : 10
      }
    }
  }
}
```

## 避免组合爆炸(Preventing Combinatorial Explosions)
(详情看这里)[https://www.elastic.co/guide/en/elasticsearch/guide/current/_preventing_combinatorial_explosions.html]
默认是depth_first，大多情况下都能很好的工作，但在一些特殊场景下breadth_first则更适合。比如
我们有一份电影的数据，记录了每一部电影的参演演员信息。我们想得到参演最多的10个演员，并且得到与这10个演员合作做多的演员的前5个助演。
显然，只需要对actors进行terms aggr，size=10，并且内嵌一个同样类型的terms aggr，size=5。
但是，我们只是想得到top10 和 top5，总计50个对象，而默认的执行过程是depth_first，假设一部电影平均n个演员，那这个数量级显然是平方级别的，其实大部分对我们并没有用处。在这种应用场景下，我们只需要第一层聚合的10个对象，然后再进一步拓展，因此需要把第一层其余bucket去除，也就是breadth_first要做的事情。根据名称也可以知道，类似于多叉树的深度优先和广度优先。

## Closing Thoughts
本小节涵盖了许多基本理论以及很多深入的技术问题。聚合给 Elasticsearch 带来了难以言喻的强大能力和灵活性。桶与度量的嵌套能力，基数与百分位数的快速估算能力，定位信息中统计异常的能力，所有的这些都在近乎实时的情况下操作的，而且全文搜索是并行的，它们改变了很多组织和企业的游戏规则。

事情通常是一旦我们开始使用它，我们就能找到很多其他的可用场景。实时报表与分析对于很多组织来说都是核心功能（它远不止商业智能或服务器日志那么简单）。

能力越大责任也越大，对于 Elasticsearch 就是意味着内存适当的管理。在 Elasticsearch 中 内存通常是个限制因素，特别是那些高度使用聚合的节点。因为聚合数据都被加载到 fielddata 中，这是一个内存数据结构，所以对内存使用的高效管理至关重要。

内存的管理形式可以有多种形式，这取决于我们特定的应用场景：
- 在数据层，确保合理的 analyze（或 not_analyze）分析数据从而友好的利用内存。
- 在索引时，对于内容很多的字段使用磁盘存储文档而不是内存里的 fielddata。
- 在搜索时，合理利用近似聚合和数据过滤。
- 在节点层，设置硬内存大小以及动态的断熔限制。
- 在操作层，监控内存使用情况并控制缓慢的内存回收周期，可以给集群增加更多节点。

大多数实施会应用到以上一种或几种方法。确切的组合方式与我们特定的系统环境高度相关。有些组织需要强劲的响应能力所以只是简单地选择增加更多节点。有的组织受限于预算，会选择使用文档值和近似聚合。

无论采取何种方式，对于现有的选择进行评估十分重要，并同时创建短期和长期计划。先决定当前内存的使用情况和需要做的事情（如果有），再决定未来六个月到一年数据会如何增长，使用何种方式来扩展？

最好在建立集群之前就计划好这些内容，而不是在我们集群堆内存使用 90% 的时候再临时抱佛脚。
