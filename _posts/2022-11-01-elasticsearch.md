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
