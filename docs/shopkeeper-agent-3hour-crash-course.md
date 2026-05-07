# shopkeeper-agent 3 小时速成指南

适用对象：已经有一点 Python 后端基础，现在想在 3 小时内尽快把这个项目看明白，至少能讲清楚它在做什么、为什么这么设计、主链路怎么走。

这份文档不是“完整掌握版”，而是“高收益速读版”。

---

## 1. 3 小时内你真正要达到什么目标

3 小时不够你把所有实现细节都吃透，但足够你做到下面几件事：

1. 知道这个项目本质上是什么
2. 知道它为什么不是“直接让 LLM 写 SQL”
3. 知道它的两条主线
4. 知道几个最关键的代码入口
5. 能用 1 分钟到 3 分钟讲清项目

如果你时间非常紧，优先目标只有一句话：

> 看懂“项目架构 + 两条主链路 + 关键设计亮点”。

---

## 2. 先用 3 分钟建立正确心智模型

这个项目本质上是一个企业问数 Agent 后端，不是通用聊天机器人。

它要解决的问题是：

1. 业务人员不会写 SQL
2. 企业库表很多，字段和指标口径复杂
3. 直接让大模型生成 SQL 很容易选错字段、理解错指标、写出不可执行 SQL

所以它采用的不是“自然语言 -> LLM -> SQL”的直连模式，而是：

1. 先检索相关字段、指标、真实取值
2. 再基于检索结果组织上下文
3. 最后生成、校验、修正并执行 SQL

你可以把这个项目理解成：

> 一个“检索增强的 NL2SQL Agent 后端”。

---

## 3. 只记住两条主线

如果你 3 小时只能记两件事，就记这两条主线。

### 3.1 主线一：元数据知识库构建链路

目标：给后续问数提供可检索的“知识底座”。

它大致做这些事：

1. 从配置和教学数仓中拿到表、字段、指标、字段真实取值
2. 把结构化元数据写进 MySQL
3. 把字段和指标做成 Qdrant 向量索引
4. 把字段真实取值做成 Elasticsearch 全文索引

一句话概括：

> 先把“问数时要用到的知识”准备好。

### 3.2 主线二：问数执行链路

目标：把用户问题变成更可靠的 SQL。

它大致做这些事：

1. 抽取关键词
2. 并行召回字段、指标、真实取值
3. 合并并过滤候选信息
4. 补充上下文
5. 生成 SQL
6. 校验 SQL
7. 校验失败则修正
8. 最终执行 SQL

一句话概括：

> 先召回，再约束，再生成，再执行。

---

## 4. 3 小时内最值得看的文件

不要一上来就漫无目的翻代码。按这个顺序看，收益最高。

### 第一组：5 分钟建立全局认知

1. [pyproject.toml](D:\pythonProject\shopkeeper-agent\pyproject.toml)
2. [conf/app_config.yaml](D:\pythonProject\shopkeeper-agent\conf\app_config.yaml)
3. [docker/docker-compose.yaml](D:\pythonProject\shopkeeper-agent\docker\docker-compose.yaml)

你要看出来的是：

1. 这个项目依赖什么技术栈
2. 本地要起哪些服务
3. 每个服务分别扮演什么角色

### 第二组：20 分钟看懂“知识库构建链路”

1. [app/scripts/build_meta_knowledge.py](D:\pythonProject\shopkeeper-agent\app\scripts\build_meta_knowledge.py)
2. [app/services/meta_knowledge_service.py](D:\pythonProject\shopkeeper-agent\app\services\meta_knowledge_service.py)

这是第一条主线的核心。

看这两个文件时，你只需要回答 4 个问题：

1. 表和字段怎么进入 MySQL
2. 字段和指标怎么进入 Qdrant
3. 真实取值怎么进入 Elasticsearch
4. 为什么要把结构化元数据和检索索引分开存

### 第三组：20 分钟看懂“问数执行链路”

1. [app/agent/graph.py](D:\pythonProject\shopkeeper-agent\app\agent\graph.py)
2. [app/agent/state.py](D:\pythonProject\shopkeeper-agent\app\agent\state.py)
3. [app/agent/context.py](D:\pythonProject\shopkeeper-agent\app\agent\context.py)

这是第二条主线的核心。

你重点看：

1. 节点顺序
2. 哪些步骤是召回
3. 哪些步骤是过滤和约束
4. `State` 和 `Context` 分别承担什么职责

### 第四组：20 分钟补齐“工程分层”

可以扫一眼这些目录：

1. `app/clients/*`
2. `app/repositories/*`
3. `app/agent/nodes/*`

你不需要逐个精读，只要明白：

1. `clients` 负责初始化外部依赖
2. `repositories` 负责和存储系统打交道
3. `service` 负责业务编排
4. `graph/nodes` 负责 Agent 执行流

---

## 5. 最推荐的 3 小时时间分配

下面这版是最实用的。

## 第 0 阶段：0 - 15 分钟

目标：先知道项目是干嘛的。

看：

1. [pyproject.toml](D:\pythonProject\shopkeeper-agent\pyproject.toml)
2. [conf/app_config.yaml](D:\pythonProject\shopkeeper-agent\conf\app_config.yaml)
3. [docker/docker-compose.yaml](D:\pythonProject\shopkeeper-agent\docker\docker-compose.yaml)

你要记住：

1. Python `>=3.14`
2. 依赖 `FastAPI`、`LangGraph`、`SQLAlchemy`、`Qdrant`、`Elasticsearch`
3. 本地基础服务有 MySQL、Qdrant、Elasticsearch、Kibana、Embedding

产出：

1. 写一句话总结项目

建议你直接记这句：

> 这是一个面向企业问数场景的检索增强 NL2SQL Agent 后端。

## 第 1 阶段：15 - 55 分钟

目标：吃透第一条主线，知识库构建链路。

看：

1. [app/scripts/build_meta_knowledge.py](D:\pythonProject\shopkeeper-agent\app\scripts\build_meta_knowledge.py)
2. [app/services/meta_knowledge_service.py](D:\pythonProject\shopkeeper-agent\app\services\meta_knowledge_service.py)

你要记住：

1. `build_meta_knowledge.py` 是执行入口
2. `MetaKnowledgeService` 是核心业务编排
3. 元数据写 MySQL
4. 字段和指标建 Qdrant 向量索引
5. 真实取值建 ES 全文索引

你要会说：

> 这个项目先把表、字段、指标和真实取值整理成可检索知识库，后续问数不是盲生成，而是建立在元数据底座上的。

## 第 2 阶段：55 - 105 分钟

目标：吃透第二条主线，LangGraph 问数执行链路。

看：

1. [app/agent/graph.py](D:\pythonProject\shopkeeper-agent\app\agent\graph.py)
2. [app/agent/state.py](D:\pythonProject\shopkeeper-agent\app\agent\state.py)
3. [app/agent/context.py](D:\pythonProject\shopkeeper-agent\app\agent\context.py)

你要记住这条顺序：

1. `extract_keywords`
2. `recall_column`
3. `recall_value`
4. `recall_metric`
5. `merge_retrieved_info`
6. `filter_table`
7. `filter_metric`
8. `add_extra_context`
9. `generate_sql`
10. `validate_sql`
11. `correct_sql`
12. `run_sql`

你要理解：

1. 为什么要三路召回
2. 为什么召回后还要过滤
3. 为什么生成 SQL 后还要校验和修正

你要会说：

> 它不是大 Prompt 一把梭，而是把关键词提取、召回、过滤、生成、校验、执行拆成了一个可编排的图。

## 第 3 阶段：105 - 140 分钟

目标：看懂工程分层，不陷入细节。

扫读：

1. `app/clients/*`
2. `app/repositories/*`
3. `app/agent/nodes/*`

你只需要建立这层认知：

1. `client manager` 管生命周期
2. `repository` 管存取
3. `service` 管业务编排
4. `graph/node` 管 Agent 执行流

这一步的意义是：让你能从“能看懂流程”升级到“能讲出工程结构”。

## 第 4 阶段：140 - 180 分钟

目标：把理解转成表达。

这 40 分钟不要继续读新代码，直接做输出。

你至少完成 3 件事：

1. 写出 1 分钟项目介绍
2. 写出 3 个亮点
3. 写出 2 个边界或局限

---

## 6. 这 3 小时里你可以暂时跳过什么

为了效率，下面这些内容先别深挖：

1. 每个 node 的细节实现
2. 所有 mapper 和 ORM model
3. 复杂配置细枝末节
4. 部署优化
5. 日志细节
6. SSE 和流式返回细节

3 小时速成的关键不是“全看完”，而是“抓主干”。

---

## 7. 你必须记住的几个亮点

如果后面你要汇报、面试、写简历，最值得讲的是这几个点。

### 7.1 检索增强 NL2SQL

不是直接用大模型写 SQL，而是先召回字段、指标、真实取值，再组织上下文生成 SQL。

### 7.2 混合检索设计

1. MySQL：权威结构化元数据
2. Qdrant：字段和指标的语义召回
3. Elasticsearch：字段真实取值的全文检索

### 7.3 LangGraph 多阶段编排

把问数过程显式拆成多个节点，便于扩展、调试和观察。

### 7.4 工程分层清晰

`client manager -> repository -> service -> graph/node`

这个结构很适合讲工程化能力。

---

## 8. 你必须知道的边界

只讲优点不够，速成时也要知道边界。

### 8.1 效果依赖元数据质量

如果表说明、字段描述、指标口径本身很差，系统效果会被明显拖累。

### 8.2 这不是通用知识问答项目

它的核心场景是企业结构化问数，不是文档问答或聊天陪伴。

### 8.3 当前仓库里有课程式骨架

从代码能看出来，部分节点更像教学阶段的占位实现，不是一个所有环节都完全产品化的成熟系统。

---

## 9. 速成时最值得背下来的 5 句话

如果你真的时间非常紧，这 5 句话能帮你快速进入状态。

1. 这是一个企业问数 Agent 后端，不是通用聊天机器人。
2. 它的核心目标是提升自然语言转 SQL 的稳定性和可控性。
3. 它不是直接让大模型写 SQL，而是先检索字段、指标和真实取值，再生成 SQL。
4. 它把结构化元数据放在 MySQL，把语义召回放在 Qdrant，把真实取值检索放在 Elasticsearch。
5. 它用 LangGraph 把问数链路拆成关键词提取、召回、过滤、生成、校验、修正和执行多个阶段。

---

## 10. 1 分钟项目介绍模板

可以直接用这版：

这个项目是一个面向企业问数场景的 Agent 后端，核心目标是解决业务人员不会写 SQL、而企业数仓字段和指标又很复杂的问题。它没有直接把自然语言问题丢给大模型生成 SQL，而是先基于 MySQL、Qdrant 和 Elasticsearch 构建元数据知识库，在查询时先召回相关字段、业务指标和真实取值，再通过 LangGraph 编排关键词提取、过滤、SQL 生成、校验和执行等阶段，从而提升 NL2SQL 的稳定性和可控性。

---

## 11. 3 小时结束后，至少自测这 8 个问题

如果下面这 8 个问题你能回答出来，就说明这次速成已经够用了。

1. 这个项目是干什么的？
2. 为什么它不是直接让 LLM 生成 SQL？
3. 两条主线分别是什么？
4. 为什么 MySQL、Qdrant、Elasticsearch 要同时存在？
5. `build_meta_knowledge.py` 在整套系统里扮演什么角色？
6. `MetaKnowledgeService` 为什么重要？
7. `graph.py` 里的主链路顺序是什么？
8. 这个项目最值得写进简历的亮点是什么？

---

## 12. 如果你只剩最后 10 分钟

那就只做这几件事：

1. 看 [app/services/meta_knowledge_service.py](D:\pythonProject\shopkeeper-agent\app\services\meta_knowledge_service.py)
2. 看 [app/agent/graph.py](D:\pythonProject\shopkeeper-agent\app\agent\graph.py)
3. 背熟第 9 节的 5 句话
4. 练一遍第 10 节的 1 分钟介绍

这样至少你不会只停留在“看过代码”，而是能真正讲出来。

---

## 13. 配套文档

如果你后面不止 3 小时，可以继续看这两份：

1. [shopkeeper-agent-one-week-study-guide.md](D:\pythonProject\shopkeeper-agent\docs\shopkeeper-agent-one-week-study-guide.md)
2. [shopkeeper-agent-daily-checklist.md](D:\pythonProject\shopkeeper-agent\docs\shopkeeper-agent-daily-checklist.md)

如果你现在目标只是“今晚速成，明天能讲”，那这份 3 小时版就够你先冲一轮了。
