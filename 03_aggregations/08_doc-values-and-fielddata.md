# Doc Values and Fielddata
## Doc Values
聚合使用一个叫 fielddata 的数据结构（在 [Doc Values Intro](https://www.elastic.co/guide/en/elasticsearch/guide/current/docvalues-intro.html) 里简单介绍）。Fielddata 通常是 Elasticsearch 集群中内存消耗最大的一部分，所以理解它的工作方式十分重要。

Fielddata 的存在是因为倒排索引只对某些操作是高效的。倒排索引的优势在于查找包含某个项的文档，而对于从另外一个方向的相反操作并不高效，即：确定只出现在单个文档里的所有项。聚合需要这种次级的辅助访问模式。

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

因此，搜索和聚合是相互紧密缠绕的。搜索使用倒排索引查找文档，聚合收集和聚合 fielddata 里的数据，而它本身也是通过倒排索引生成的。
>**NOTE**:`Doc values`不仅可以用于聚合。任何需要查找某个文档包含的值的操作都必须使用它。除了聚合，还包括排序，访问字段值的脚本，父子关系处理（参见[Parent-Child Relationship](https://www.elastic.co/guide/en/elasticsearch/guide/current/parent-child.html)）。

## Deep Dive on Doc Valuesedit
The last section opened by saying doc values are "fast, efficient and memory-friendly". Those are some nice marketing buzzwords, but how do doc values actually work?

Doc values are generated at index-time, alongside the creation of the inverted index. That means doc values are generated on a per-segment basis and are immutable, just like the inverted index used for search. And, like the inverted index, doc values are serialized to disk. This is important to performance and scalability.

By serializing a persistent data structure to disk, we can rely on the OS’s file system cache to manage memory instead of retaining structures on the JVM heap. In situations where the "working set" of data is smaller than the available memory, the OS will naturally keep the doc values resident in memory. This gives the same performance profile as on-heap data structures.

But when your working set is much larger than available memory, the OS will begin paging the doc values on/off disk as required. This will obviously be slower than an entirely memory-resident data structure, but it has the advantage of scaling well beyond the server’s memory capacity. If these data structures were purely on-heap, the only option is to crash with an OutOfMemory exception (or implement a paging scheme just like the OS).

Note
Because doc values are not managed by the JVM, Elasticsearch servers can be configured with a much smaller heap. This gives more memory to the OS for caching. It also has the benefit of letting the JVM’s garbage collector work with a smaller heap, which will result in faster and more efficient collection cycles.

Traditionally, the recommendation has been to dedicate 50% of the machine’s memory to the JVM heap. With the introduction of doc values, this recommendation is starting to slide. Consider giving far less to the heap, perhaps 4-16gb on a 64gb machine, instead of the full 32gb previously recommended.

For a more detailed discussion, see Heap: Sizing and Swapping.

Column-store compressionedit
At a high level, doc values are essentially a serialized column-store. As we discussed in the last section, column-stores excel at certain operations because the data is naturally laid out in a fashion that is amenable to those queries.

But they also excel at compressing data, particularly numbers. This is important for both saving space on disk and for faster access. Modern CPU’s are many orders of magnitude faster than disk drives (although the gap is narrowing quickly with upcoming NVMe drives). That means it is often advantageous to minimize the amount of data that must be read from disk, even if it requires extra CPU cycles to decompress.

To see how it can help compression, take this set of doc values for a numeric field:

Doc      Terms
-----------------------------------------------------------------
Doc_1 | 100
Doc_2 | 1000
Doc_3 | 1500
Doc_4 | 1200
Doc_5 | 300
Doc_6 | 1900
Doc_7 | 4200
-----------------------------------------------------------------
The column-stride layout means we have a contiguous block of numbers: [100,1000,1500,1200,300,1900,4200]. Because we know they are all numbers (instead of a heterogeneous collection like you’d see in a document or row) values can be packed tightly together with uniform offsets.

Further, there are a variety of compression tricks we can apply to these numbers. You’ll notice that each of the above numbers are a multiple of 100. Doc values detect when all the values in a segment share a greatest common divisor and use that to compress the values further.

If we save 100 as the divisor for this segment, we can divide each number by 100 to get: [1,10,15,12,3,19,42]. Now that the numbers are smaller, they require fewer bits to store and we’ve reduced the size on-disk.

Doc values use several tricks like this. In order, the following compression schemes are checked:

If all values are identical (or missing), set a flag and record the value
If there are fewer than 256 values, a simple table encoding is used
If there are > 256 values, check to see if there is a common divisor
If there is no common divisor, encode everything as an offset from the smallest value
You’ll note that these compression schemes are not "traditional" general purpose compression like DEFLATE or LZ4. Because the structure of column-stores are rigid and well-defined, we can achieve higher compression by using specialized schemes rather than the more general compression algorithms like LZ4.

Note
You may be thinking "Well that’s great for numbers, but what about strings?" Strings are encoded similarly, with the help of an ordinal table. The strings are de-duplicated and sorted into a table, assigned an ID, and then those ID’s are used as numeric doc values. Which means strings enjoy many of the same compression benefits that numerics do.

The ordinal table itself has some compression tricks, such as using fixed, variable or prefix-encoded strings.

Disabling Doc Valuesedit
Doc values are enabled by default for all fields except analyzed strings. That means all numerics, geo_points, dates, IPs and not_analyzed strings.

Analyzed strings are not able to use doc values at this time; the analysis process generates many tokens and does not work efficiently with doc values. We’ll discuss using analyzed strings for aggregations in Aggregations and Analysis.

Because doc values are on by default, you have the option to aggregate and sort on most fields in your dataset. But what if you know you will never aggregate, sort or script on a certain field?

While rare, these circumstances do arise and you may wish to disable doc values on that particular field. This will save you some disk space (since the doc values are not being serialized to disk anymore) and may increase indexing speed slightly (since the doc values don’t need to be generated).

To disable doc values, set doc_values: false in the field’s mapping. For example, here we create a new index where doc values are disabled for the "session_id" field:

PUT my_index
{
  "mappings": {
    "my_type": {
      "properties": {
        "session_id": {
          "type":       "string",
          "index":      "not_analyzed",
          "doc_values": false
        }
      }
    }
  }
}


By setting doc_values: false, this field will not be usable in aggregations, sorts or scripts

It is possible to configure the inverse relationship too: make a field available for aggregations via doc values, but make it unavailable for normal search by disabling the inverted index. For example:

PUT my_index
{
  "mappings": {
    "my_type": {
      "properties": {
        "customer_token": {
          "type":       "string",
          "index":      "not_analyzed",
          "doc_values": true,
          "index": "no"
        }
      }
    }
  }
}


Doc values are enabled to allow aggregations



Indexing is disabled, which makes the field unavailable to queries/searches

By setting doc_values: true and index: no, we generate a field which can only be used in aggregations/sorts/scripts. This is admittedly a very rare requirement, but sometimes useful.

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

## 高基数内存的影响（High-Cardinality Memory Implications）
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

So, before aggregating string fields, assess the situation:
- Is it a `not_analyzed` field? If yes, the field will use doc values and be memory-friendly
- Otherwise, this is an `analyzed` field. It will use fielddata and live in-memory. Does this field have a very large cardinality caused by ngrams, shingles, etc? If yes, it may be very memory unfriendly.

## 限制内存使用（Limiting Memory Usage）
为了让聚合（或任何需要访问字段值的操作）更快，访问 fielddata 必须快速，这就是为什么将它载入内存的原因。但加载过多的数据到内存会导致垃圾回收变慢，因为 JVM 会尝试在堆中找到额外的空间，或甚至有可能导致 OutOfMemory 异常。

Elasticsearch 不仅仅将与查询匹配的文档载入到 fielddata 中，这可能会令我们感到吃惊，它还会将 索引内所有文档 的值加载，甚至是那些不同类型 `_type` 的文档！

逻辑是这样：如果查询会访问文档 X、Y 和 Z，那很有可能会在下一个查询中访问其他文档。将所有的信息一次加载，再将其维持在内存中的方式要比每次请求都扫描倒排索引的代价要低。

JVM 堆是有限资源，应该被合理利用。限制 fielddata 对堆使用的影响有多套机制，这些限制方式非常重要，因为堆栈的乱用会导致节点不稳定（感谢缓慢的垃圾回收机制），甚至导致节点宕机（通常伴随 OutOfMemory 异常）。
```
选择堆大小（Choosing a Heap Size）
在设置 Elasticsearch 堆大小时需要通过 `$ES_HEAP_SIZE` 环境变量应用两个规则：
- **不要超过可用 RAM 的 50%**:Lucene 能很好利用文件系统的缓存，它是通过系统内核管理的。如果没有足够的文件系统缓存空间，性能会收到影响。
- **不要超过 32GB**：如果堆大小小于 32 GB，JVM 可以利用指针压缩，这可以大大降低内存的使用：每个指针 4 字节而不是 8 字节。  
```

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
