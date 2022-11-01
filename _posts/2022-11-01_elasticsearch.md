---
layout: post
title: elasticsearch核心技术与实战
---
在极客时间上学习`elasticsearch`课程，主要关注点在`query`的`DSL`语句以及集群的管理，在本地基于`es 7.1`来构建集群服务，启动脚本如下，同时在`conf/elasticsearch.yml`中添加`xpack.ml.enabled: false`、`http.host: 0.0.0.0`的配置(禁用`ml`)：
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
`elasticsearch`中创建新文档，用`post`请求方式，`url`内容为`index/_doc/id`。当未指定{id}时，会自动生成随机的`id`。`put`方式用于更新文档，当`PUT users/_doc/1?op_type=create`或`PUT users/_create/1`指定文档`id`存在时，就会报错。
```bash
POST users/_doc
{
  "user": "mike",
  "post_date": "2019-04-15T14:12:12",
  "message": "trying out kibana"
}
```
`elasticsearch`的分词器`analysis`，分词是指把全文本转换为一些列的单词`(term/token)`的过程，其通常由`Character Filters`、`Tokenizer`、`Token Filters`这三部分组成。具体`url`示例如下，`analyzer`的类型可以有：`standard`、`stop`、`simple`等。
```bash
GET _analyze
{
  "analyzer": "stop",
  "text": "2 running Quick brown-foxes leap over lazy dogs in the summer evening."
}
```
url中query string的语法，指定字段v.s.泛查询，其中`df`为默认字段，当不指定`df`只按`q`查询时，则是泛查询，从`_doc`的所有字段检索：
```bash
GET /movies/_search?q=2012&df=title&sort=year:desc&from=0&size=10&timeout=1s
```
