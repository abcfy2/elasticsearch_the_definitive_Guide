# 近似聚合
如果所有的数据都在一台机器上，那么生活会容易许多，CS201 课商教的经典算法就足够应付这些问题。但如果所有的数据都在一台机器上，那么就不需要像 Elasticsearch 这样的分布式软件了。不过一旦我们开始分布式数据存储，算法的选择就需务必小心。

有些算法可以分布执行，到目前为止讨论过的所有聚合都是单次请求获得精确结果的。这些类型的算法通常是高度并行的，因为它们无须任何额外代价，就能在多台机器上并行执行。比如当计算 max 度量时，以下的算法就非常简单：
1. 请求广播到所有分片。
2. 查看每个文档的 `price`字段。如果 `price > current_max`，将 `current_max` 替换成 `price`。
3. 返回所有分片的最大 `price` 并传给协调节点。
4. 找到从所有分片返回的最大 `price` 。这是最终的最大值。

这个算法可以随着机器数的线性增长而横向扩展，无须任何协调操作（机器之间不需要讨论中间结果），而且内存消耗很小（一个整型就能代表最大值）。

不幸的是，不是所有的算法都像获取最大值这样简单。更加复杂的操作则需要在算法的性能和内存使用上做出权衡。对于这个问题，我们有个三角因子模型：大数据、精确性和实时性。

我们需要选择其中两项：
- **精确 + 实时**:数据可以存入单台机器的内存之中，我们可以随心所欲，使用任何想用的算法。结果会 100% 精确，响应会相对快速。
- **大数据 + 精确**:传统的 Hadoop。可以处理 PB 级的数据并且为我们提供精确的答案，但它可能需要几周的时间才能为我们提供这个答案。
- **大数据 + 实时**:近似算法为我们提供准确但不精确的结果。

Elasticsearch 目前支持两种近似算法（`cardinality`（基数） 和 `percentiles`（百分位数））。它们会提供准确但不是 100% 精确的结果。以牺牲一点小小的估算错误为代价，这些算法可以为我们换来高速的执行效率和极小的内存消耗。

对于大多数应用领域，能够实时返回高度准确的结果要比 100% 精确结果重要得多。乍一看这可能是天方夜谭。有人会叫“我们需要精确的答案！”。但仔细考虑 0.5% 错误所带来的影响：

- 99% 的网站延时都在 132ms 以下。
- 0.5% 的误差对以上延时的影响在正负 0.66ms 。
- 近似计算会在毫秒内放回结果，而“完全正确”的结果就可能需要几秒，甚至无法返回。

只要简单的查看网站的延时情况，难道我们会在意近似结果是在 132.66ms 内返回而不是 132ms？当然，不是所有的领域都能容忍这种近似结果，但对于绝大多数来说是没有问题的。接受近似结果更多的是一种文化观念上的壁垒而不是商业或技术上的需要。

## 查找唯一值的数目（Finding Distinct Counts）
Elasticsearch 提供的首个近似聚合是基数度量。它提供一个字段的基数，即该字段的唯一值的数目。可能会对 SQL 形式比较熟悉：
```sql
SELECT COUNT(DISTINCT color)
FROM cars
```
Distinct 计数是一个普通的操作，可以回答很多基本的商业问题：
- 网站的独立访问用户（UVs）是多少？
- 卖了多少种汽车？
- 每月有多少独立用户购买了商品？

我们可以用基数度量确定经销商销售汽车颜色的种类：
```json
GET /cars/transactions/_search
{
    "size" : 0,
    "aggs" : {
        "distinct_colors" : {
            "cardinality" : {
              "field" : "color"
            }
        }
    }
}
```
返回的结果表明已经售卖了三种不同颜色的汽车：
```json
...
"aggregations": {
  "distinct_colors": {
     "value": 3
  }
}
...
```
可以让我们的例子变得更有用：每月有多少颜色的车被售出？为了得到这个度量，我们只需要将一个`cardinality`度量嵌入一个`date_histogram`：
```json
GET /cars/transactions/_search
{
  "size" : 0,
  "aggs" : {
      "months" : {
        "date_histogram": {
          "field": "sold",
          "interval": "month"
        },
        "aggs": {
          "distinct_colors" : {
              "cardinality" : {
                "field" : "color"
              }
          }
        }
      }
  }
}
```

## 学会权衡（Understanding the Trade-offs）
正如我们本章开头提到的，`cardinality`度量是一个近似算法。它是基于`HyperLogLog++`（HLL）算法的，HLL 会先对我们的输入作哈希运算，然后根据哈希运算的结果中的`bits`做概率估算从而得到基数。

我们不需要理解技术细节（如果确实感兴趣，可以阅读这篇论文），但我们最好应该关注一下这个算法的特性：
- 可配置的精度，用来控制内存的使用（更精确 ＝ 更多内存）。
- 对于低基数集能够达到高准确度。
- 固定的内存使用。即使有几千或甚至上百亿的唯一值，内存的使用也只是依赖于配置里的精度要求。

要配置精度，我们必须指定`precision_threshold`参数的值。这个阀值定义了在何种基数水平下我们希望得到一个近乎精确的结果。参考以下示例：
```json
GET /cars/transactions/_search
{
    "size" : 0,
    "aggs" : {
        "distinct_colors" : {
            "cardinality" : {
              "field" : "color",
              "precision_threshold" : 100
            }
        }
    }
}
```
`precision_threshold`接受 0–40,000 之间的数字，更大的值还是会被当作40,000来处理。

示例会确保当字段唯一值在 100 以内时会得到非常准确的结果。尽管算法是无法保证这点的，但如果基数在阀值以下，几乎总是 100% 正确的。高于阀值的基数会开始节省内存而牺牲准确度，同时也会对度量结果带入误差。

对于指定的阀值，`HLL` 的数据结构会大概使用内存`precision_threshold` * 8 字节，所以就必须在牺牲内存和获得额外的准确度间做平衡。

在实际应用中，100 的阀值可以在唯一值为百万的情况下仍然将误差维持 5% 以内。

## 速度优化（Optimizing for Speed）
如果想要获得唯一数目的值，通常需要查询整个数据集合（或几乎所有数据）。所有基于所有数据的操作都必须迅速，原因是显然的。HyperLogLog 的速度已经很快了，它只是简单的对数据做哈希以及一些位操作。

但如果速度对我们至关重要，可以做进一步的优化，因为 HLL 只需要字段内容的哈希值，我们可以在索引时就预先计算好。就能在查询时跳过哈希计算然后将数据信息直接加载出来。

> **NOTE**:
>- 预先计算哈希值只对内容很长或者基数很高的字段有用，计算这些字段的哈希值的消耗在查询时是无法忽略的。
>- 尽管数值字段的哈希计算是非常快速的，存储它们的原始值通常需要同样（或更少）的内存空间。这对低基数的字符串字段同样适用，Elasticsearch 的内部优化能够保证每个唯一值只计算一次哈希。
>- 基本上说，预先计算并不能保证所有的字段都快，它只对那些具有高基数和内容很长的字符串字段有作用。需要记住的是预先计算也只是简单地将代价转到索引时，代价就在那里，不增不减。

要想这么做，我们需要为数据增加一个新的多值字段。我们先删除索引，再增加一个包括哈希值字段的映射，然后重新索引：
```json
DELETE /cars/

PUT /cars/
{
  "mappings": {
    "transactions": {
      "properties": {
        "color": {
          "type": "string",
          "fields": {
            "hash": {
              "type": "murmur3"
            }
          }
        }
      }
    }
  }
}

POST /cars/transactions/_bulk
{ "index": {}}
{ "price" : 10000, "color" : "red", "make" : "honda", "sold" : "2014-10-28" }
{ "index": {}}
{ "price" : 20000, "color" : "red", "make" : "honda", "sold" : "2014-11-05" }
{ "index": {}}
{ "price" : 30000, "color" : "green", "make" : "ford", "sold" : "2014-05-18" }
{ "index": {}}
{ "price" : 15000, "color" : "blue", "make" : "toyota", "sold" : "2014-07-02" }
{ "index": {}}
{ "price" : 12000, "color" : "green", "make" : "toyota", "sold" : "2014-08-19" }
{ "index": {}}
{ "price" : 20000, "color" : "red", "make" : "honda", "sold" : "2014-11-05" }
{ "index": {}}
{ "price" : 80000, "color" : "red", "make" : "bmw", "sold" : "2014-01-01" }
{ "index": {}}
{ "price" : 25000, "color" : "blue", "make" : "ford", "sold" : "2014-02-12" }
```
多值字段的类型是`murmur3`，这是一个哈希函数。

现在当我们执行聚合时，我们使用 `color.hash` 字段而不是 `color` 字段：
```json
GET /cars/transactions/_search
{
    "size" : 0,
    "aggs" : {
        "distinct_colors" : {
            "cardinality" : {
              "field" : "color.hash"
            }
        }
    }
}
```
注意我们指定的是哈希过的多值字段，而不是原始字段。

现在`cardinality`度量会读取`"color.hash"`里的值（预先计算的哈希值），并将它们作为原始值的动态哈希值。

每个文档节省的时间有限，但如果哈希每个字段需要 10 纳秒而我们的聚合需要访问一亿文档，那么每个查询就需要多花 1 秒钟的时间。如果我们发现自己在很多文档都会使用 `cardinality` 基数，可以做些性能分析看是否有必要在我们部署的应用中采用预先计算哈希的方式。

## 百分计算（Calculating Percentiles）
Elasticsearch 提供的另外一个近似度量就是 `percentiles` 百分位数度量。百分位数展现某以具体百分比下观察到的数值。例如，第95个百分位上的数值，是高于 95% 的数据总和。

百分位数通常用来找出异常。在（统计学）的正态分布下，第 0.13 和 第 99.87 的百分位数代表与均值距离三倍标准差的值。任何处于三倍标准差之外的数据通常被认为是不寻常的，因为它与平均值相差太大。

更具体的说，假设我们正运行一个庞大的网站，而我们的任务是保证用户请求能得到快速响应，因此我们就需要监控网站的延时来判断响应是否能达到目标。

在此场景下，一个常用的度量方法就是平均响应延时，但这是一个不好的选择（尽管很常用），因为平均数通常会隐藏那些异常值，中位数有着同样的问题。我们可以尝试最大值，但这个度量会轻而易举的被单个异常值破坏。

在图 Figure 40, “Average request latency over time” 查看问题。如果我们倚靠如平均值或中位数这样的简单度量，就会得到像这样一幅图:
## Figure 40.Average request latency over time
![](https://www.elastic.co/guide/en/elasticsearch/guide/current/images/elas_33in01.png)

一切正常。图上有轻微的波动，但没有什么值得关注的。但如果我们加载 99 百分位数时（这个值代表最慢的 1% 的延时），我们看到了完全不同的一幅画面，如图:
## Figure 41.Average request latency with 99th percentile over time
![](https://www.elastic.co/guide/en/elasticsearch/guide/current/images/elas_33in02.png)
令人吃惊！在上午九点半时，均值只有 75ms。如果作为一个系统管理员，我们都不会看他第二眼。一切正常！但 99 百分位告诉我们有 1% 的用户碰到的延时超过 850ms，这是另外一幅场景。在上午4点48时也有一个小波动，这甚至无法从平均值和中位数曲线上观察到。

这只是百分位的一个应用场景，百分位还可以被用来快速用肉眼观察数据的分布，检查是否有数据倾斜或双峰甚至更多。

## 百分位度量（Percentile Metric）
让我加载一个新的数据集（汽车的数据不太适用于百分位）。我们要索引一系列网站延时数据然后运行一些百分位操作进行查看：
```json
POST /website/logs/_bulk
{ "index": {}}
{ "latency" : 100, "zone" : "US", "timestamp" : "2014-10-28" }
{ "index": {}}
{ "latency" : 80, "zone" : "US", "timestamp" : "2014-10-29" }
{ "index": {}}
{ "latency" : 99, "zone" : "US", "timestamp" : "2014-10-29" }
{ "index": {}}
{ "latency" : 102, "zone" : "US", "timestamp" : "2014-10-28" }
{ "index": {}}
{ "latency" : 75, "zone" : "US", "timestamp" : "2014-10-28" }
{ "index": {}}
{ "latency" : 82, "zone" : "US", "timestamp" : "2014-10-29" }
{ "index": {}}
{ "latency" : 100, "zone" : "EU", "timestamp" : "2014-10-28" }
{ "index": {}}
{ "latency" : 280, "zone" : "EU", "timestamp" : "2014-10-29" }
{ "index": {}}
{ "latency" : 155, "zone" : "EU", "timestamp" : "2014-10-29" }
{ "index": {}}
{ "latency" : 623, "zone" : "EU", "timestamp" : "2014-10-28" }
{ "index": {}}
{ "latency" : 380, "zone" : "EU", "timestamp" : "2014-10-28" }
{ "index": {}}
{ "latency" : 319, "zone" : "EU", "timestamp" : "2014-10-29" }
```
数据有三个值：延时、数据中心的区域以及时间戳。让我们对数据全集进百分位操作以获得数据分布情况的直观感受：
```json
GET /website/logs/_search
{
    "size" : 0,
    "aggs" : {
        "load_times" : {
            "percentiles" : {
                "field" : "latency"
            }
        },
        "avg_load_time" : {
            "avg" : {
                "field" : "latency"
            }
        }
    }
}
```
1. `percentiles` 度量被应用到 `latency` 延时字段。
2. 为了比较，我们对相同字段使用 `avg` 度量。

默认情况下，`percentiles` 度量会返回一组预定义的百分位数值：`[1, 5, 25, 50, 75, 95, 99]`。它们表示了人们感兴趣的常用百分位数值，极端的百分位数在范围的两边，其他的一些处于中部。在返回的响应中，我们可以看到最小延时在 75ms 左右，而最大延时差不多有 600ms。与之形成对比的是，平均延时在 200ms 左右，信息并不是很多：
```json
...
"aggregations": {
  "load_times": {
     "values": {
        "1.0": 75.55,
        "5.0": 77.75,
        "25.0": 94.75,
        "50.0": 101,
        "75.0": 289.75,
        "95.0": 489.34999999999985,
        "99.0": 596.2700000000002
     }
  },
  "avg_load_time": {
     "value": 199.58333333333334
  }
}
```
所以显然延时的分布很广，然我们看看它们是否与数据中心的地理区域有关：
```json
GET /website/logs/_search
{
    "size" : 0,
    "aggs" : {
        "zones" : {
            "terms" : {
                "field" : "zone"
            },
            "aggs" : {
                "load_times" : {
                    "percentiles" : {
                      "field" : "latency",
                      "percents" : [50, 95.0, 99.0]
                    }
                },
                "load_avg" : {
                    "avg" : {
                        "field" : "latency"
                    }
                }
            }
        }
    }
}
```
1. 首先根据区域我们将延时分到不同的桶中。
2. 再计算每个区域的百分位数值。
3. `percents` 参数接受了我们想返回的一组百分位数，因为我们只对长的延时感兴趣。

在响应结果中，我们发现欧洲区域（EU）要比美国区域（US）慢很多，在美国区域（US），50 百分位与 99 百分位十分接近，它们都接近均值。

与之形成对比的是，欧洲区域（EU）在 50 和 99 百分位有较大区分。现在，显然可以发现是欧洲区域（EU）拉低了延时的统计信息，我们知道欧洲区域的 50% 延时都在 300ms+。
```json
...
"aggregations": {
  "zones": {
     "buckets": [
        {
           "key": "eu",
           "doc_count": 6,
           "load_times": {
              "values": {
                 "50.0": 299.5,
                 "95.0": 562.25,
                 "99.0": 610.85
              }
           },
           "load_avg": {
              "value": 309.5
           }
        },
        {
           "key": "us",
           "doc_count": 6,
           "load_times": {
              "values": {
                 "50.0": 90.5,
                 "95.0": 101.5,
                 "99.0": 101.9
              }
           },
           "load_avg": {
              "value": 89.66666666666667
           }
        }
     ]
  }
}
...
```
## 百分位等级（Percentile Ranks）
这里有另外一个紧密相关的度量叫 `percentile_ranks`。百分位度量告诉我们落在某个百分比以下的所有文档的最小值。例如，如果 50 百分位是 119ms，那么有 50% 的文档数值都不超过 119ms。`percentile_ranks` 告诉我们某个具体值属于哪个百分位。119ms 的 `percentile_ranks` 是在 50 百分位。这基本是个双向关系，例如：
- 50 百分位是 119ms。
- 119ms 百分位等级是 50 百分位。

所以假设我们网站必须维持的服务等级协议（SLA）是响应时间低于 210ms。然后，开个玩笑，我们老板警告我们如果响应时间超过 800ms 会把我开除。可以理解的是，我们希望知道有多少百分比的请求可以满足 SLA 的要求（并期望至少在 800ms 以下！）。

为了做到这点，我们可以应用 `percentile_ranks` 度量而不是 `percentiles` 度量：
```json
GET /website/logs/_search
{
    "size" : 0,
    "aggs" : {
        "zones" : {
            "terms" : {
                "field" : "zone"
            },
            "aggs" : {
                "load_times" : {
                    "percentile_ranks" : {
                      "field" : "latency",
                      "values" : [210, 800]
                    }
                }
            }
        }
    }
}
```
`percentile_ranks`度量接受一组我们希望分级的数值。

在聚合运行后，我们能得到两个值：
```json
"aggregations": {
  "zones": {
     "buckets": [
        {
           "key": "eu",
           "doc_count": 6,
           "load_times": {
              "values": {
                 "210.0": 31.944444444444443,
                 "800.0": 100
              }
           }
        },
        {
           "key": "us",
           "doc_count": 6,
           "load_times": {
              "values": {
                 "210.0": 100,
                 "800.0": 100
              }
           }
        }
     ]
  }
}
```
这告诉我们三点重要的信息：
- 在欧洲（EU），210ms 的百分位等级是 31.94% 。
- 在美国（US），210ms 的百分位等级是 100% 。
- 在欧洲（EU）和美国（US），800ms 的百分位等级是 100% 。

通俗的说，在欧洲区域（EU）只有 32% 的响应时间满足服务等级协议（SLA），而美国区域（US）始终满足服务等级协议的。但幸运的是，两个区域所有响应时间都在 800ms 以下，所有我们还不会被炒鱿鱼（至少目前不会）。

`percentile_ranks` 度量提供了与 `percentiles` 相同的信息，但它以不同方式呈现，如果我们对某个具体数值更关心，使用它会更方便。

## 学会权衡（Understanding the Trade-offs）
和基数一样，计算百分位需要一个近似算法。朴素的实现会维护一个所有值的有序列表，但当我们有几十亿数据分布在几十个节点时，这几乎是不可能的。

取而代之的是`percentiles` 使用一个 `TDigest` 算法（由 Ted Dunning 在 [Computing Extremely Accurate Quantiles Using T-Digests](https://github.com/tdunning/t-digest/blob/master/docs/t-digest-paper/histo.pdf) 里面提出的）。有了 HyperLogLog，就不需要理解完整的技术细节，但有必要了解算法的特性：

- 百分位的准确度与百分位的极端程度相关，也就是说 1 或 99 的百分位要比 50 百分位要准确。这只是数据结构内部机制的一种特性，但这是一个好的特性，因为多数人只关心极端的百分位。
- 对于数值集合较小的情况，百分位非常准确。如果数据集足够小，百分位可能 100% 精确。
- 随着桶里数值的增长，算法会开始对百分位进行估算。它能有效在准确度和内存节省之间做出权衡。不准确的程度比较难以总结，因为它依赖于聚合时数据的分布以及数据量的大小。

与基数类似，我们可以通过修改参数 `compression` 来控制内存与准确度之间的比值。

`TDigest` 算法用节点近似计算百分比：节点越多，准确度越高（同时内存消耗也越大），这都与数据量成正比。`compression` 参数限制节点的最大数目为 `20 * compression`。

因此，通过增加压缩比值，我们可以提高准确度同时也消耗更多内存。更大的压缩比值会使算法运行更慢，因为底层的树形数据结构的存储也会增长，也导致操作的代价更高。默认的压缩比值是 100。

一个节点粗略计算使用 32 字节的内存，所以在最坏的情况下（例如，大量数据有序存入），默认设置会生成一个大小约为 64KB 的 `TDigest`。在实际应用中，数据会更随机，所以 `TDigest` 使用的内存会更少。