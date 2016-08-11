# 安装与运行
ES依赖`JRE7`及以上版本，使用前需要确保已经安装`JRE7`以上版本。

从[下载页面](http://www.elastic.co/downloads/elasticsearch)上下载需要的版本，解压zip包，运行`bin/elasticsearch`即可。

使用`curl`命令请求`localhost:9200`，将得到下列的输出:
```json
curl 'http://localhost:9200/?pretty'
{
  "name" : "Tom Foster",
  "cluster_name" : "elasticsearch",
  "version" : {
    "number" : "2.1.0",
    "build_hash" : "72cd1f1a3eee09505e036106146dc1949dc5dc87",
    "build_timestamp" : "2015-11-18T22:40:03Z",
    "build_snapshot" : false,
    "lucene_version" : "5.3.1"
  },
  "tagline" : "You Know, for Search"
}
```

> 更多安装文档，参见官方文档: https://www.elastic.co/guide/en/elasticsearch/reference/current/_installation.html

## 安装Sense
[`Sense`](https://www.elastic.co/guide/en/sense/current/installing.html)是一个kibana插件，可以方便的将elasticsearch的查询可视化。

安装`Sense`可以在kibana的目录下运行:
```
./bin/kibana plugin --install elastic/sense
```

重启`kibana`，然后在`http://localhost:5601/app/sense`即可访问。

> 此插件仅在开发环境调试时有用，无需部署至产品环境

## 如何连接到ES
如果你使用的是JAVA，那么可以以以下两种节点形式连接到ES集群中:
- `Node client`: 以*非数据节点*(non data node)形式连接到集群中，即本身不保存任何数据，但是`Node client`知道数据节点在哪，并且可以做数据转发
- `Transport client`: 更加轻量级的节点，和上述节点相比，不加入集群，仅仅将请求转发入集群节点中

> **NOTE**: 这两种节点都连接的是ES的`9300`端口，这个端口用于ES的[*transport*](https://www.elastic.co/guide/en/elasticsearch/reference/current/modules-transport.html)协议，集群节点通过这个端口通信。如果这个端口无法访问，将无法组成集群。

## RESTful API with JSON
其他编程语言可以连接ES的`9200`端口，这个端口是RESTful API，接受请求和发送响应都是`JSON`格式。

> **NOTE**: ES官方提供Groovy, JavaScript, .NET, PHP, Perl, Python, and Ruby这几种语言的类库，相关文档可以在这里找到: https://www.elastic.co/guide/en/elasticsearch/client/index.html

ES的请求格式如下:

    curl -X<VERB> '<PROTOCOL>://<HOST>:<PORT>/<PATH>?<QUERY_STRING>' -d '<BODY>'

`< >`标记之间的内容如下:
- `VERB`: HTTP Method: `GET`, `POST`, `PUT`, `HEAD`, or `DELETE`.
- `PROTOCOL`: Either http or https (if you have an https proxy in front of Elasticsearch.)
- `HOST`: The hostname of any node in your Elasticsearch cluster, or localhost for a node on your local machine.
- `PORT`: The port running the Elasticsearch HTTP service, which defaults to `9200`.
- `PATH`: API Endpoint (for example `_count` will return the number of documents in the cluster). Path may contain multiple components, such as `_cluster/stats` or `_nodes/stats/jvm`
- `QUERY_STRING`: Any optional query-string parameters (for example `?pretty` will *pretty-print* the JSON response to make it easier to read.)
- `BODY`: A JSON-encoded request body (if the request needs one.)

例如，计算集群中文档的数量:
```json
curl -XGET 'http://localhost:9200/_count?pretty' -d '
{
    "query": {
        "match_all": {}
    }
}
'
```

服务器会返回`200`响应码，并带上响应主体:
```json
{
    "count" : 0,
    "_shards" : {
        "total" : 5,
        "successful" : 5,
        "failed" : 0
    }
}
```

简写请求格式:
```json
GET /_count
{
    "query": {
        "match_all": {}
    }
}
```

同时，这也是`Sense console`显示的格式。
