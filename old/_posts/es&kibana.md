---
title: es&kibana
date: 2019-10-29 19:39:05
comments: true #是否可评论
toc: true #是否显示文章目录
categories: "检索" #分类
archives: "近期"
tags:   #标签
	- es
	- kibana
	- 全文搜索
---

1. es概述
===
>- ElasticSearch简称Elastic,开源
>- 目前全文搜索引擎的首选
>- 快速地储存、搜索和分析海量数据
>- Elastic的底层是开源库Lucene，Lucene需要代码调用，es是Lucene的封装，提供RestAPI接口，开箱即用；
>- 维基百科、Stack Overflow、Github均使用es
>- es: 索引相当于数据库，type相当于表，document相当于一行数据，field相当于列
>
```
Relational DB -> Databases -> Tables -> Rows -> Columns
Elasticsearch -> Indices   -> Types  -> Documents -> Fields
索引名称必须小写：
curl -X GET 'g1-sys-es-v01.dns.guazi.com:9200/_cat/indices?v'
```
>- Document的分组为type，它是虚拟的逻辑分组，用来过滤 Document。
不同的 Type 应该有相似的结构（schema），举例来说，id字段不能在这个组是字符串，在另一个组是数值。这是与关系型数据库的表的一个区别。性质完全不同的数据（比如products和logs）应该存成两个 Index，而不是一个 Index 里面的两个 Type（虽然可以做到）。
>
```
curl 'g1-sys-es-v01.dns.guazi.com:9200/fact_imkf_agent_statistics/_mapping?pretty=true'
```
>- 根据规划，Elastic 6.x 版只允许每个 Index 包含一个 Type，7.x 版将会彻底移除 Type。

>- 分布式数据库，每台服务器可以运行多个Elastic 实例，单个elastic实例为单个节点，多个elastic示例组成集群；
>- es数据分片，并存在副本replica,默认5个分片两个副本，不同分片存储在不通的实例上，同一分片的不同副本存储在不通的实例上；某一实例down掉，相关分片会迁移；
>- 针对一个索引，Elasticsearch 中其实有专门的衡量索引健康状况的标志，分为三个等级：
>
```
green，绿色。这代表所有的主分片和副本分片都已分配。你的集群是 100% 可用的。
yellow，黄色。所有的主分片已经分片了，但至少还有一个副本是缺失的。不会有数据丢失，所以搜索结果依然是完整的。不过，你的高可用性在某种程度上被弱化。如果更多的分片消失，你就会丢数据了。所以可把 yellow 想象成一个需要及时调查的警告。
red，红色。至少一个主分片以及它的全部副本都在缺失中。这意味着你在缺少数据：搜索只能返回部分数据，而分配到这个分片上的写入请求会返回一个异常。
如果你只有一台主机的话，其实索引的健康状况也是 yellow，因为一台主机，集群没有其他的主机可以防止副本，所以说，这就是一个不健康的状态，因此集群也是十分有必要的。
```
>- es节点类型：
>
```
主节点：即 Master 节点。主节点的主要职责是和集群操作相关的内容，如创建或删除索引，跟踪哪些节点是群集的一部分，并决定哪些分片分配给相关的节点。稳定的主节点对集群的健康是非常重要的。默认情况下任何一个集群中的节点都有可能被选为主节点。索引数据和搜索查询等操作会占用大量的cpu，内存，io资源，为了确保一个集群的稳定，分离主节点和数据节点是一个比较好的选择。虽然主节点也可以协调节点，路由搜索和从客户端新增数据到数据节点，但最好不要使用这些专用的主节点。一个重要的原则是，尽可能做尽量少的工作。  
数据节点：即 Data 节点。数据节点主要是存储索引数据的节点，主要对文档进行增删改查操作，聚合操作等。数据节点对 CPU、内存、IO 要求较高，在优化的时候需要监控数据节点的状态，当资源不够的时候，需要在集群中添加新的节点。
负载均衡节点：也称作 Client 节点，也称作客户端节点。当一个节点既不配置为主节点，也不配置为数据节点时，该节点只能处理路由请求，处理搜索，分发索引操作等，从本质上来说该客户节点表现为智能负载平衡器。独立的客户端节点在一个比较大的集群中是非常有用的，他协调主节点和数据节点，客户端节点加入集群可以得到集群的状态，根据集群的状态可以直接路由请求。
预处理节点：也称作 Ingest 节点，在索引数据之前可以先对数据做预处理操作，所有节点其实默认都是支持 Ingest 操作的，也可以专门将某个节点配置为 Ingest 节点。
```

[参考博客](https://www.ruanyifeng.com/blog/2017/08/elasticsearch.html)
[安装elastic server](https://www.jianshu.com/p/f283d876b1cb)
[es手册](https://www.elastic.co/guide/cn/elasticsearch/guide/current/_sorting_by_a_metric.html)
[es权威指南](https://es.xiaoleilu.com/300_Aggregations/21_add_metric.html)
[es简介](https://www.ruanyifeng.com/blog/2017/08/elasticsearch.html)
[Es GitHub cli包golang](https://github.com/olivere/elastic)
[Es GitHub cli包golang示例](https://olivere.github.io/elastic/)
[es kibana集群搭建](https://cloud.tencent.com/developer/article/1189282)
[官方教程query](https://www.elastic.co/guide/en/elasticsearch/reference/6.4/query-dsl-query-string-query.html#query-string-syntax)
[es mapping数据类型](https://www.elastic.co/guide/en/elasticsearch/reference/6.4/mapping.html)
2. es检索之search
===

2.1 learf query
---
Leaf query clauses特定field中的特定值，eg: match, term or range queries.   
These queries can be used by themselves.
>- ***query string***:The query_string query parses the input and splits text around operators
>
```
curl 'g1-sys-es-v01.dns.guazi.com:9200/fact_imkf_agent_statistics/_search' -d '{ 
"query":
	{"query_string":
		{
			"default_field":"customer_id",
			"query":"600335867 OR fe598152-aee5-4e17-8e0d-0ed4b04e0485"
		}
	}
}'
```
>- ***Term***,搜索包含特性term的document,boost增加特定term的重要性
>
```
curl 'g1-sys-es-v01.dns.guazi.com:9200/fact_imkf_agent_statistics/_search' -d '{
  "query": {
    "bool": {
      "should": [
        {
          "term": {
            "customer_id": {
              "value": "600335867",
              "boost": 2.0 
            }
          }
        },
        {
          "term": {
            "customer_id": "fe598152-aee5-4e17-8e0d-0ed4b04e0485" 
          }
        }
      ]
    }
  }
}'
// 评价
curl 'g1-sys-es-v01.dns.guazi.com:9200/fact_imkf_rate_record/_search' -d '{
  "query": {
    "bool": {
      "should": [
        {
          "match": {
            "source_id":"43138"
            }
        },
        {
        	"match":{
        		"agent_id":"235210"
        	}
        }
      ]
    }
  },
  "sort": [
    {   
     "timestamp": { "order": "desc"   }
    }
  ]
}'
```
>>- 对类型为text的最好用match, 类型为text的field会由分词器，分词获取倒排索引，term匹配不一定能匹配到（term需要完全一致）
>
>>- 	type text会被分词，type keyword不会被分词；full_text的倒排索引包含一下terms: [quick, foxes];exact_value的倒排索引仅包含the exact term: [Quick Foxes!]
>>
```
PUT my_index
{
  "mappings": {
    "_doc": {
      "properties": {
        "full_text": {
          "type":  "text" 
        },
        "exact_value": {
          "type":  "keyword" 
        }
      }
    }
  }
}
PUT my_index/_doc/1
{
  "full_text":   "Quick Foxes!", 
  "exact_value": "Quick Foxes!"  
}
1：This query matches because the exact_value field contains the exact term Quick Foxes!.
GET my_index/_search
{
  "query": {
    "term": {
      "exact_value": "Quick Foxes!" 
    }
  }
}
2：This query does not match, because the full_text field only contains the terms quick and foxes. It does not contain the exact term Quick Foxes!
GET my_index/_search
{
  "query": {
    "term": {
      "full_text": "Quick Foxes!" 
    }
  }
}
3：A term query for the term foxes matches the full_text field.
GET my_index/_search
{
  "query": {
    "term": {
      "full_text": "foxes" 
    }
  }
}
4：This match query on the full_text field first analyzes the query string, then looks for documents containing quick or foxes or both.
GET my_index/_search
{
  "query": {
    "match": {
      "full_text": "Quick Foxes!" 
    }
  }
}
```
[term query](https://www.elastic.co/guide/en/elasticsearch/reference/6.4/query-dsl-term-query.html)
>
>- ***Range query***:获取term在一定范围内的document,query类型与fiedl type的类型有关， 对string为termRangeQuery,对number&date为NumericRangeQuery
>
```
返回age在[10.20]的document
GET _search
{
    "query": {
        "range" : {
            "age" : {
                "gte" : 10,
                "lte" : 20,
                "boost" : 2.0
            }
        }
    }
}
对日期的range可以用date math
GET _search
{
    "query": {
        "range" : {
            "date" : {
                "gte" : "now-1d/d",
                "lt" :  "now/d"
            }
        }
    }
}
gt
Greater than the date rounded up: 2014-11-18||/M becomes 2014-11-30T23:59:59.999, ie excluding the entire month.
gte
Greater than or equal to the date rounded down: 2014-11-18||/M becomes 2014-11-01, ie including the entire month.
lt
Less than the date rounded down: 2014-11-18||/M becomes 2014-11-01, ie excluding the entire month.
lte
Less than or equal to the date rounded up: 2014-11-18||/M becomes 2014-11-30T23:59:59.999, ie including the entire month.
```
[range类型filed的range](https://www.elastic.co/guide/en/elasticsearch/reference/6.4/range.html)  
[range query](https://www.elastic.co/guide/en/elasticsearch/reference/6.4/query-dsl-range-query.html)

2.2 复合query
---
Compound query clauses
Compound query clauses wrap other leaf or compound queries and are used to combine multiple queries in a logical fashion (such as the bool or dis_max query), or to alter their behaviour (such as the constant_score query).
>- [bool query](https://www.elastic.co/guide/en/elasticsearch/reference/6.4/query-dsl-bool-query.html)
>- 


