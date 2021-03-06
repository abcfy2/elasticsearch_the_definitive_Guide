# 对多值桶排序
多值桶（terms、histogram 和 date_histogram）动态生成很多桶，Elasticsearch 是如何决定这些桶展示给用户的顺序呢？

默认的，桶会根据 doc_count 降序排列，这是一个好的默认行为，因为通常我们想要找到文档中与查询条件相关的最大值：售价、人口数量、频率。但有些时候我们希望能修改这个顺序，不同的桶有着不同的处理方式。

## 排序的本质（Intrinsic Sorts）
这些排序模式是桶 固有的 能力：它们操作桶生成的数据，比如 doc_count。它们共享相同的语法，但是根据使用桶的不同会有些细微差别。

让我们做一个 terms 聚合但是按 doc_count 值的升序排序：
```json
GET /cars/transactions/_search
{
    "size" : 0,
    "aggs" : {
        "colors" : {
            "terms" : {
              "field" : "color",
              "order": {
                "_count" : "asc"
              }
            }
        }
    }
}
```
用关键字` _count`，我们可以按`doc_count`值的升序排序。

我们为聚合引入了一个`order`对象，它使我们可以根据以下值进行排序：

- `_count`:按文档数排序。在`terms`、`histogram`、`date_histogram`内使用。

- `_term`:按词项的字符串值的字母顺序排序。只在 `terms` 内使用。

- `_key`:按每个桶的键值数值排序（理论上与` _term` 类似）。只在 `histogram` 和 `date_histogram` 内使用。

## 按度量排序（Sorting by a Metric）
有时，我们会想基于度量计算的结果值进行排序。在我们的汽车销售分析仪表盘中，我们可能想按照汽车颜色创建一个销售条状图表，但按照汽车平均售价的升序进行排序。

我们可以增加一个度量，再指定 `order` 参数引用这个度量即可：
```json

GET /cars/transactions/_search
{
    "size" : 0,
    "aggs" : {
        "colors" : {
            "terms" : {
              "field" : "color",
              "order": {
                "avg_price" : "asc"
              }
            },
            "aggs": {
                "avg_price": {
                    "avg": {"field": "price"}
                }
            }
        }
    }
}
```
1. 计算每个桶的平均售价。
2. 桶按照计算平均值的升序排序。

我们可以采用这种方式用任何度量排序，只需简单的引用度量的名字。不过有些度量会输出多个值。`extended_stats` 度量是一个很好的例子：它输出好几个度量值。

如果我们想使用多值度量进行排序，我们只需以关心的度量为关键词使用点式路径:
```json
GET /cars/transactions/_search
{
    "size" : 0,
    "aggs" : {
        "colors" : {
            "terms" : {
              "field" : "color",
              "order": {
                "stats.variance" : "asc"
              }
            },
            "aggs": {
                "stats": {
                    "extended_stats": {"field": "price"}
                }
            }
        }
    }
}
```
使用点式标记方式，我们可以对感兴趣的度量进行排序。

## 基于“深度”度量排序（Sorting Based on "Deep" Metrics）
在上面这个例子中，我们按每个桶的方差来排序，所以售价方差最小的会出现在方差更多之前。在前面的示例中，度量是桶的直接子节点。平均售价是根据每个 term 来计算的。在一定条件下，我们也有可能对更深的度量进行排序，比如孙子桶或从孙桶。

我们可以定义更深的路径，将度量用尖括号（>）嵌套起来，像这样： `my_bucket>another_bucket>metric`。

需要提醒的是嵌套路径上的每个桶都必须是单值的。`filter` 桶生成一个单值桶：所有与过滤条件匹配的文档都在桶中。多值桶（如：`terms`）动态生成许多桶，无法通过指定一个确定路径来识别。

目前，只有三个单值桶：`filter`、`global` 和 `reverse_nested`。让我们快速用示例说明，创建一个汽车售价的直方图，但是按照红色和绿色（不包括蓝色）车各自的方差来排序：
```json
GET /cars/transactions/_search
{
    "size" : 0,
    "aggs" : {
        "colors" : {
            "histogram" : {
              "field" : "price",
              "interval": 20000,
              "order": {
                "red_green_cars>stats.variance" : "asc"
              }
            },
            "aggs": {
                "red_green_cars": {
                    "filter": { "terms": {"color": ["red", "green"]}},
                    "aggs": {
                        "stats": {"extended_stats": {"field" : "price"}}
                    }
                }
            }
        }
    }
}
```
1. 按照嵌套度量的方差对桶的直方图进行排序。
2. 因为我们使用单值过滤器，我们可以使用嵌套排序。
3. 按照生成的度量对统计结果进行排序。

本例中，可以看到我们如何访问一个嵌套的度量，`stats` 度量是 `red_green_cars` 聚合的子节点，而`red_green_cars` 又是 `colors` 聚合的子节点。为了根据这个度量排序，我们定义了路径 `red_green_cars>stats.variance` 。我们可以这么做，因为 `filter` 桶是个单值桶。
