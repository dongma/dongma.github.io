# 项目纪律
1. 视觉 1:1：所有颜色/字号/间距/圆角以 resource/dongma_blog_ui_designs/src/*.html 为准，禁止顺手优化设计
2. 技术栈：保持 Jekyll，禁止迁移到其他 SSG
3. 先计划后动手：每阶段输出计划，确认后才写代码
4. 验收：每阶段完成→bundle exec jekyll serve→渲染结果逐格对照 resource/src 自查，过 spec §8 清单
5. resource/ 目录只读参考，不得修改、不得被站点引用

## 文章标签分类法（11 个固定标签）

| 标签 | 适用领域 | 当前文章数 |
|---|---|---|
| 大数据 | Spark / Hadoop / HDFS / MapReduce / Yarn / Hive / Flink | 6 |
| 云原生 | Docker / K8s / Nginx / Service Mesh / 网关 | 4 |
| Java | Java 语言特性 / 工具链 / 生态 | 3 |
| JVM | 字节码 / 内存模型 / GC / 类加载 | 2 |
| Go | Go 语言 / 生态 | 2 |
| Kafka | Kafka / Pulsar / 消息队列 | 2 |
| K8s | Kubernetes / 容器编排 | 2 |
| Nginx | Nginx / 反向代理 / 网关 | 2 |
| Docker | Docker / 容器 | 1 |
| Elasticsearch | ES / OpenSearch / 搜索引擎 | 1 |
| 前端 | 前端 / 小程序 / 可视化 | 1 |
| Scala | Scala 语言 / 函数式编程 / 类型系统 | 4 |

**打标签规则：**
1. 每篇**支持多个标签**（1~3 个都是合理的），按需扩，不要凑数
2. 优先双标签：先归一级主题（Java / Go / 大数据 / 云原生 / 前端），再按内容决定是否补二级（JVM / Kafka / K8s / Elasticsearch / Nginx / Docker）
3. 工具/基础设施类（Docker / Nginx）**必须**双标签：工具名 + "云原生"
4. 标签中文，与正文用语一致（用 "K8s" 不是 "k8s"，用 "Elasticsearch" 不是 "ES"）
5. 不用宽泛词（"教程" / "实战"）—— 由标题承担
6. 新增标签必须先在本表登记，保持标签集收敛在 ~11 个
