---
layout: post
title: Elasticsearch核心技术与实战
---
在极客时间上学习`elasticsearch`课程，主要关注点在`query`的`DSL`语句以及集群的管理，在本地基于`es 7.1`来构建集群服务，启动脚本如下，同时在`conf/elasticsearch.yml`中添加`xpack.ml.enabled: false`、`http.host: 0.0.0.0`的配置(禁用`ml`及启用`host`)：
```bash
bash> bin/elasticsearch -E node.name=node0 -E cluster.name=geektime -E path.data=node0_data -d
bash> bin/elasticsearch -E node.name=node1 -E cluster.name=geektime -E path.data=node1_data -d
bash> bin/elasticsearch -E node.name=node2 -E cluster.name=geektime -E path.data=node2_data -d
bash> bin/elasticsearch -E node.name=node3 -E cluster.name=geektime -E path.data=node3_data -d
```
在`docker`容器中启动`cerebro`服务，用于监控`elasticsearch`集群的状态，`docker`启动命令如下：
```bash
bash> docker run -d --name cerebro -p 9100:9000 lmenezes/cerebro:latest
```

## 文档index基础操作
1) `elasticsearch`中创建新文档，用`post`请求方式，`url`内容为`index/_doc/id`。当未指定{id}时，会自动生成随机的`id`。`put`方式用于更新文档，当`PUT users/_doc/1?op_type=create`或`PUT users/_create/1`指定文档`id`存在时，就会报错。
```bash
POST users/_doc
{
  "user": "mike",
  "post_date": "2019-04-15T14:12:12",
  "message": "trying out kibana"
}
```
<!-- more -->
2) `elasticsearch`的分词器`analysis`，分词是指把全文本转换为一些列的单词`(term/token)`的过程，其通常由`Character Filters`、`Tokenizer`、`Token Filters`这三部分组成。具体`url`示例如下，`analyzer`的类型可以有：`standard`、`stop`、`simple`等。
```bash
GET _analyze
{
  "analyzer": "stop",
  "text": "2 running Quick brown-foxes leap over lazy dogs in the summer evening."
}
```
3) `url`中`query string`的语法，指定字段v.s.泛查询，其中`df`为默认字段，当不指定`df`只按`q`查询时，则是泛查询，从`_doc`的所有字段检索：
```bash
GET /movies/_search?q=2012&df=title&sort=year:desc&from=0&size=10&timeout=1s
```
## URl Search、Request Body查询及文档Mapping
1）在`elasticsearch`中查询可以分为`url search`和`request body`查询，其中`url search`用`GET`方式，相关参数放在`url`中。`df`指定默认查询字段，`q`为查询字符串。当未指定`df`时，称为泛查询，会拿数值与`doc`中所有字段进行匹配：
```bash
# es中查询的dsl，df指定默认字段，q为查询数值,TermQuery
GET kibana_sample_data_ecommerce/_search?q=Eddie&df=customer_first_name
{
  "profile": "true"
}
# 若不用df的话，可以用q=field:value来进行替换
GET kibana_sample_data_ecommerce/_search?q=customer_first_name:Eddie
{
  "profile": "true"
}
```
2）`Phrase query`与`Term query`的区别，`PhraseQuery`会按整个字符串进行匹配，而`TermQuery`则会对字符串进行分词。对于`term`来说，只要`Field value`中包含任意一个单词就可以。
```bash
# phrase query，相当于不会做分词，匹配完整字符串(1条)
GET kibana_sample_data_ecommerce/_search?q=customer_full_name:"Eddie Underwood"
{
  "profile": "true"
}
# term query，对字符串进行了分词,好像也有keyword概念，任意匹配Eddie或Underwood就可以
GET kibana_sample_data_ecommerce/_search?q=customer_full_name:Eddie Underwood
{
  "profile": "true"
}
```
此外，在`url query`中还支持分组的概念，也就是`Bool Query`。当查询条件为`customer_full_name:(Eddie Underwood)`时，会分别按`Eddie`和`Underwood`进行匹配，其是任意的满足关系。若想在字段中同时满足要求，则可在分组中添加`AND`操作符。此外，`url query`还支持`range`查询及通配符查询。
```bash
# bool query，full_name中包括Eddie或Underwood才可以，实现同时包含，则需添加AND关键字
GET kibana_sample_data_ecommerce/_search?q=customer_full_name:(Eddie AND Underwood)
{
  "profile": "true"
}
# 数值范围查询，(订单总额)taxful_total_price大于50
GET kibana_sample_data_ecommerce/_search?q=taxful_total_price:>=50
{
  "profile": "true"
}
# 通配符查询，只要email字段中含"gwen"就会被匹配
GET kibana_sample_data_ecommerce/_search?q=email:gwen*
{
  "profile": "true"
}
```
3）`Request body`查询的详细解释，这其实是一种更通用的写法，使用`POST`请求方式。在`body`中使用`_source`指定要获取的字段列表，同时`sort`可指定按哪个字段进行排序。`query`部分指定了具体的查询条件，`operator`为`and`最终效果类似于`phrase query`。`elasticsearch`的`painless`脚本用于特定计算，返回计算后的新字段（如金额转换等）。
```bash
# es request body的写法，按订单总金额排序desc,_source过滤doc中的字段
POST kibana_sample_data_ecommerce/_search
{
  "_source": ["taxful_total_price", "total_quantity", "customer_full_name", "manufacturer"],
  "sort": [{"taxful_total_price": "desc"}],
  "query": {
    "match": {
      "customer_full_name": {
        "query": "Eddie Lambert",
        "operator": "and"
      }
    }
  },
  "script_fields": {
    "addtional_field": {
      "script": {
        "lang": "painless",
        "source": "doc['taxful_total_price'].value + '_hello'"
      }
    }
  }
}
```
此外，对于`match_phrase`则不会进行分词，对`_doc`会直接进行查询。`body`中的`slop`参数可用于近似度查询，提升数据检索的容错性。
```bash
# match_phrase查询，不会进行分词，直接匹配total字符串,slop指定term结果
POST kibana_sample_data_ecommerce/_search
{
  "query": {
    "match_phrase": {
      "customer_full_name": {
        "query": "Eddie Lambert",
        "slop": 1
      }
    }
  }
}
```
4）`query_string`与`simple_query_string`的区别，`query_string`与`url query`类似，也需指定`default_field`。同时，其也支持多字段`fields`及多分组`query`的查询，`simple_query_string#query`也需指定查询条件。
```bash
# query_string和url query比较类似，也支持分组，如下的query_string#fields
POST /users/_search
{
  "query": {
    "query_string": {
      "default_field": "name",
      "query": "Ruan AND YiMing"
    }
  }
}
POST /users/_search
{
  "query": {
    "query_string": {
      "fields": ["name", "about"],
      "query": "(Ruan And YiMing) OR (Java AND Elasticsearch)"
    }
  }
}
POST /users/_search
{
  "query": {
    "simple_query_string": {
      "query": "Ruan AND YiMing",
      "fields": ["name"]
    }
  }
}
```
5）对于文档`mapping`这一部分，类似比喻的话，相当于是数据表的`schema`，规定了字段的约束信息。对于`dynamic mapping`，`elasticsearch`支持三种模式：`true`、`false`和`strict`。其默认值为`true`，当设置`mapping`为`false`时，新添加的字段不能检索，但会在`_source`部分展示，当为`strict`时，索引文档新增字段时，会进行报错。
```bash
GET mapping_test/_mapping
# 修改dynamic为false，新加的字段不能被索引
PUT dynamic_mapping_test/_mapping
{
  "dynamic": false
}
PUT dynamic_mapping_test/_doc/10
{
  "anotherField": "otherValue"
}
# dynamic为false时，新增的字段无法被检索，strict模式下，新添加字段会报错
POST dynamic_mapping_test/_search
{
  "query": {
    "match": {
      "anotherField": "otherValue"
    }
  }
}
```
## 深入ElasticSearch搜索机制
1）类似于`mysql`的聚合函数，`elasticsearch`也提供了文档聚合，聚合采用`aggs`运算符号，示例如下，对`kibana`航班数据按`DestCountry`字段进行分组，并计算票价的平均值、最大值、最小值：
  ```bash
  # 按照目的地进行分桶的统计，并按平均票价进行统计
GET kibana_sample_data_flights/_search
{
  "size": 0,
  "aggs": {
    "flight_dest": {
      "terms": {"field": "DestCountry"},
      "aggs": {
        "avg_price": {
          "avg": {
            "field": "AvgTicketPrice"
          }
        },
        "max_price": {
          "max": {
            "field": "AvgTicketPrice"
          }
        },
        "min_price": {
          "min": {
            "field": "AvgTicketPrice"
          }
        }
      }
    }
  }
}
```
除此之外，`elasticsearch`还支持`stats`操作，按`flight_dest`字段进行分组，同时使用`stats`操作，并分别统计平均票价、目的地的天气情况。
```bash
GET kibana_sample_data_flights/_search
{
  "size": 0,
  "aggs": {
    "flight_dest": {
      "terms": {
        "field": "DestCountry"
      },
      "aggs": {
        "stats_price": {
          "stats": {
            "field": "AvgTicketPrice"
          }
        },
        "weather": {
          "terms": {
            "field": "DestWeather",
            "size": 5
          }
        }
      }
    }
  }
}
```
2）深入理解分词的逻辑，在使用`_bulk api`批量写入一批文档后，查询文档时，通过原有的字段是检索不到的，必须将其转换为小些。向`products`索引写入`3`条数据，分别为`Apple`的产品。
```bash
# _bulk api批量写入数据，一次写入3条数据
POST /products/_bulk
{"index": {"_id": 1}}
{"productID": "XHDK-1902-#fj3", "desc": "iPhone", "price": 30}
{"index": {"_id": 2}}
{"productID": "XHDK-1003-#446", "desc": "iPad", "price": 35}
{"index": {"_id": 3}}
{"productID": "XHDK-6902-#521", "desc": "MBP", "price": 40}
```
通过`term query`按`iPhone`进行检索时，是查不到数据的。原因是在存储文档时，`elasticsearch`对字段值进行了分词，数据字段按小写形式进行存储，当用`iphone`检索时是可以的。此外，`elasticsearch`中每个字段都有`keyword`属性，在用`field.keyword`查询时则可以进行完整的匹配。
```bash
# 直接用iPhone在desc#value查询，搜不到记录。但用desc.keyword可以，因为在保存文档时，iPhone在索引中已进行了小写
POST /products/_search
{
  "query": {
    "term": {
      "desc.keyword": {
        "value": "iPhone"
      }
    }
  }
}
# 将query改为filter的方式，忽略TF-IDF算分问题，避免相关性算分的开销，提升查询性能
POST /products/_search
{
  "explain": true,
  "query": {
    "constant_score": {
      "filter": {
        "term": {
          "productID.keyword": "XHDK-1902-#fj3"
        }
      }
    }
  }
}
```
为了提升查询效率，可以用`constant_score#filter`来替换`term query`，因为其不进行算分，所以效率能高一些。同时，其也支持`range query`和`exists`操作符。
```bash
# 用range方式进行范围查询，通过doc.price进行过滤
GET /products/_search
{
  "query": {
    "constant_score": {
      "filter": {
        "range": {
          "price": {
            "gte": 20, "lte": 30
          }
        }
      }
    }
  }
}
# 用exists来查找一些field值非空的文档，并将其进行返回
POST /products/_search
{
  "query": {
    "constant_score": {
      "filter": {
        "exists": {
          "field": "desc"
        }
      }
    }
  }
}
```
3）`query context`与`filter context`影响算分的问题，默认情况下`elasticsearch`会按照匹配度问题给文档进行打分，在文档每部分可使用`boost`来影响其分数，当文档中两个字段都含关键词时，可通过`boost`设置权重，进而影响文档的排名。
```bash
# query context与filter context影响算分问题
POST /blogs/_bulk
{"index": {"_id": 1}}
{"title": "Apple iPad", "content": "Apple iPad,Apple iPad"}
{"index": {"_id": 2}}
{"title": "Apple iPad,Apple iPad", "content": "Apple iPad"}
# 通过boost指定每部分字段的权重，进而影响文档的算分排序
POST blogs/_search
{
  "query": {
    "bool": {
      "should": [
        {"match": {
          "title": {
            "query": "apple,ipad",
            "boost": 1
          }
          }
        },
        {"match": {
          "content": {
            "query": "apple,ipad",
            "boost": 2
          }
        }}
      ]
    }
  }
}
```
在`bool`查询中，`must`和`should`是算分的，而`must_not`则不计入算分，在检索示例中可通过`must`及`must_not`来过滤文档。默认情况下，用`term query`查询时，只要`doc`中包含关键字的频率高，则其相应的算分也会高。在具有相同数量关键词的字段中，`doc`长度越小的文档相关性越高。
```bash
# 批量写入关于apple的新闻数据，批量写入文档记录
POST news/_bulk
{"index": {"_id": 1}}
{"content": "Apple Mac"}
{"index": {"_id": 2}}
{"content": "Apple iPad"}
{"index": {"_id": 3}}
{"content": "Apple employee like Apple Pie and Apple Juice"}
# 然而并不是所期望的，返回了apple食品记录
POST news/_search
{
  "query": {
    "bool": {
      "must": {
        "match": {"content": "apple"}
      }
    }
  }
}
```
可通过`must_not`对不符合条件的文档进行剔除，若只是想将不相关的文档分数减小，则可以通过`boosting#positive`或`boosting#negative`使得对文档进行重新的计分，这样不相关的文档也会进行展示，但其排名比较靠后。
```bash
# 用must_not排除pie字符串，只剩余电子产品
POST news/_search
{
  "query": {
    "bool": {
      "must": {"match": {"content": "apple"}},
      "must_not": {"match": {"content": "pie"}}
    }
  }
}
# 当不想删除时，可使用boosting#positive、negative方式排序
POST news/_search
{
  "query": {
    "boosting": {
      "positive": {
        "match": {"content": "apple"}
      },
      "negative": {
        "match": {"content": "pie"}
      },
      "negative_boost": 0.5
    }
  }
}
```
4）`disjunction query`也是关于文档相关性的，若文档中有两部分都匹配，若想按文档匹配度高的那一部分排序的话（不按累加求和），则应使用此查询。同时，还可按`tie_breaker`对文档分数进行扰乱，进而影响文档的排名。
```bash
PUT /blogs/_bulk
{"index": {"_id": 1}}
{"title": "Quick brown rabbits", "body": "Brown rabbits are commonly seen"}
{"index": {"_id": 2}}
{"title": "Keeping pets happy", "body": "My quick brown fox eats rabbits on a regular basis."}
# 用dis_max#queries找两部分，各自评分最高的内容，此外还可通过tie_breaker进行调整
POST /blogs/_search
{
  "query": {
    "dis_max": {
      "queries": [
        {"match": {"title": "Brown fox"}},
        {"match": {"body": "Brown fox"}}
      ],
      "tie_breaker": 0.2
    }
  }
}
```
多字段查询的搜索语法，`most_fields`会累计多个字段的分数之和，`cross_fields`也就是当`query`在多个字段中存在时，就会返回结果，也就是所谓的跨字段查询。
```bash
PUT address/_doc/1
{
  "street": "5 Poland Street",
  "city": "London",
  "country": "United Kingdom",
  "postcode": "W1V 3DG"
}
# 使用most_fields是可以的，但增加operator:and就不可以了。可将type改为cross_fields，表示将query string在多个字段中进行检索
POST address/_search
{
  "query": {
    "multi_match": {
      "query": "Poland Street W1V",
      "fields": ["street", "city", "country", "postcode"],
      "type": "cross_fields",
      "operator": "and"
    }
  }
}
```
可以使用`alias`语法对索引进行重命名，应用场景多为`elasticsearch`索引数据备份，为避免应用服务端开发时修改配置，可做到无感数据源切换。
```bash
# index的alias操作，用于对address进行重命名
POST _aliases
{
  "actions": [
    {
      "add": {
        "index": "address",
        "alias": "address_latest"
      }
    }
  ]
}
```
