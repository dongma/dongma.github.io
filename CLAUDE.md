# 项目纪律
1. 视觉 1:1：所有颜色/字号/间距/圆角以 resource/dongma_blog_ui_designs/src/*.html 为准，禁止顺手优化设计
2. 技术栈：保持 Jekyll，禁止迁移到其他 SSG
3. 先计划后动手：每阶段输出计划，确认后才写代码
4. 验收：每阶段完成→bundle exec jekyll serve→渲染结果逐格对照 resource/src 自查，过 spec §8 清单
5. resource/ 目录只读参考，不得修改、不得被站点引用

## 文章标签分类法（12 个固定标签，按文章数降序）

| 标签 | 适用领域 | 当前文章数 |
|---|---|---|
| AI | AI / LLM / Agent | 8 |
| Go | Go 语言 / 生态 | 8 |
| Spark | Spark / 分布式计算 | 8 |
| MySQL | MySQL / 数据库 / SQL | 5 |
| Scala | Scala 语言 / 函数式编程 / 类型系统 | 4 |
| 大数据 | Hadoop / HDFS / MapReduce / Yarn / Hive / Flink | 4 |
| JVM | 字节码 / 内存模型 / GC / 类加载 | 2 |
| 前端 | 前端 / 小程序 / 可视化 | 2 |
| K8s | K8s / 容器编排 | 2 |
| Kafka | Kafka / Pulsar / 消息队列 | 2 |
| Docker | Docker / 容器 | 1 |
| Elasticsearch | ES / OpenSearch / 搜索引擎 | 1 |

**打标签规则：**
1. 每篇**只打一个标签** —— 选最具体的那一个（JVM 比 Java 具体，K8s 比"云原生"具体，Spark 比"大数据"具体，工具名比"云原生"具体）
2. 一级主题优先（AI / Go / 大数据 / 前端），但**只在有且仅有一个清晰归属时**用一级；否则选二级（JVM / Kafka / K8s / Elasticsearch / Nginx / Docker / Spark / MySQL / Scala）
3. 工具/基础设施类（Docker / K8s / Nginx）**只保留工具名**，不再叠"云原生"
4. 标签中文，与正文用语一致（用 "K8s" 不是 "k8s"，用 "Elasticsearch" 不是 "ES"）
5. 不用宽泛词（"教程" / "实战"）—— 由标题承担
6. 新增标签必须先在本表登记，保持标签集收敛在 ~12 个
