# shopkeeper-agent 一周速成每日打卡清单

配套主文档： [shopkeeper-agent-one-week-study-guide.md](D:\pythonProject\shopkeeper-agent\docs\shopkeeper-agent-one-week-study-guide.md)

这份清单的目标不是让你“看过就算”，而是帮你每天都留下可检验的学习产出。

建议使用方式：

- 每天开始前先看当天目标。
- 学完后把勾选项过一遍。
- 最后一定要完成“今日输出”，不要只停留在阅读代码。

---

## Day 1 打卡：建立全局认知

### 今日目标

- 知道项目解决什么问题。
- 知道项目为什么不是直接让 LLM 写 SQL。
- 知道系统里有哪些核心角色。

### 今日必读

- `pyproject.toml`
- `conf/app_config.yaml`
- `docker/docker-compose.yaml`
- `app` 目录结构

### 打卡清单

- [ ] 我能说出这个项目的核心目标是“企业问数 Agent 后端”。
- [ ] 我能说出它不是通用聊天机器人。
- [ ] 我能说出它的两条主线：知识库构建链路、问数执行链路。
- [ ] 我能列出核心依赖：MySQL、Qdrant、Elasticsearch、Embedding、LLM。
- [ ] 我知道根目录 `main.py` 不是正式业务主入口。
- [ ] 我知道当前学习重点在 `app` 目录。

### 今日输出

- [ ] 画一张一页纸架构概览。
- [ ] 用 3 句话写下“这个项目解决什么问题”。
- [ ] 用 3 句话写下“为什么不能直接让大模型写 SQL”。

### 自测问题

- [ ] 用户输入是什么？
- [ ] 系统输出是什么？
- [ ] 为什么要同时引入 MySQL、Qdrant、ES？

### 达标标准

- [ ] 我能在 3 分钟内口头讲清项目全貌。

---

## Day 2 打卡：环境与配置

### 今日目标

- 搞清本地运行依赖。
- 搞清配置和服务的映射关系。
- 知道环境层哪些内容值得面试时提。

### 今日必读

- `pyproject.toml`
- `.env.example`
- `conf/app_config.yaml`
- `docker/docker-compose.yaml`
- `app/conf/*`

### 打卡清单

- [ ] 我知道项目要求 `Python >= 3.14`。
- [ ] 我知道项目用 `uv` 管理依赖。
- [ ] 我知道 `.env` 里当前关键变量是 `LLM_API_KEY`。
- [ ] 我知道 `db_meta` 和 `db_dw` 对应两套 MySQL。
- [ ] 我知道 `qdrant.embedding_size` 会影响向量集合创建。
- [ ] 我知道本地默认端口：3306、9200、5601、6333、8081。

### 今日输出

- [ ] 整理一张环境依赖表：服务名、作用、端口、对应配置。
- [ ] 写一段 100 字以内的“本项目运行环境说明”。

### 自测问题

- [ ] 为什么 MySQL 要拆成 `meta` 和 `dw`？
- [ ] 为什么 Embedding 服务是单独起的？
- [ ] 为什么 LLM API Key 不直接写进 YAML？

### 达标标准

- [ ] 我能解释每个服务为什么存在，而不只是背端口。

---

## Day 3 打卡：知识库构建链路

### 今日目标

- 搞清楚元数据是怎么被组织和建索引的。
- 能解释为什么项目要把结构化存储和检索索引分开。

### 今日必读

- [app/scripts/build_meta_knowledge.py](D:\pythonProject\shopkeeper-agent\app\scripts\build_meta_knowledge.py)
- [app/services/meta_knowledge_service.py](D:\pythonProject\shopkeeper-agent\app\services\meta_knowledge_service.py)
- `app/repositories/mysql/*`
- `app/repositories/qdrant/*`
- `app/repositories/es/*`

### 打卡清单

- [ ] 我知道 `build_meta_knowledge.py` 是知识库构建入口。
- [ ] 我知道脚本层负责初始化依赖并调用 Service，而不是承载复杂业务逻辑。
- [ ] 我知道 `MetaKnowledgeService` 是这一主线的核心。
- [ ] 我知道表和字段会先落到 Meta MySQL。
- [ ] 我知道字段和指标都会写入 Qdrant。
- [ ] 我知道字段真实取值会写入 Elasticsearch。
- [ ] 我知道字段和指标都会拆成多个 embedding point。

### 今日输出

- [ ] 画一张“表/字段/指标/取值”落库与建索引流程图。
- [ ] 用自己的话写出“为什么结构化元数据和检索索引要分开存”。
- [ ] 用自己的话写出“为什么字段真实取值更适合放 ES 而不是 Qdrant”。

### 自测问题

- [ ] 为什么 MySQL 是权威元数据存储？
- [ ] 为什么 Qdrant 适合字段和指标语义召回？
- [ ] 为什么不是所有字段都同步真实取值？

### 达标标准

- [ ] 我能完整讲清知识库构建链路。

---

## Day 4 打卡：问数主图与状态流

### 今日目标

- 吃透 LangGraph 主图。
- 能手画执行顺序。
- 能区分召回步骤和约束生成步骤。

### 今日必读

- [app/agent/graph.py](D:\pythonProject\shopkeeper-agent\app\agent\graph.py)
- `app/agent/state.py`
- `app/agent/context.py`

### 打卡清单

- [ ] 我知道 `graph.py` 是问数执行主入口。
- [ ] 我知道流程从 `extract_keywords` 开始。
- [ ] 我知道关键词后会并行进入三路召回。
- [ ] 我知道三路结果会先合并，再继续过滤。
- [ ] 我知道生成 SQL 前还有补充上下文步骤。
- [ ] 我知道 SQL 校验失败后会进入 `correct_sql`。
- [ ] 我知道最终节点是 `run_sql`。

### 今日输出

- [ ] 手画一张主链路图。
- [ ] 整理一份节点顺序笔记。
- [ ] 按顺序写出每个节点的 1 句职责说明。

### 自测问题

- [ ] 为什么关键词提取后要并行召回？
- [ ] 为什么合并结果后还要过滤？
- [ ] 为什么校验失败不是直接结束？

### 达标标准

- [ ] 我能不看代码复述整条图执行顺序。

---

## Day 5 打卡：节点与仓储层协作

### 今日目标

- 看懂节点如何通过 `state` 和 `context` 协作。
- 能识别哪些节点是已成型逻辑，哪些是骨架占位。

### 今日必读

- `app/agent/nodes/recall_*`
- `app/agent/nodes/filter_*`
- `app/agent/nodes/generate_sql.py`
- `app/agent/nodes/validate_sql.py`
- `app/clients/*`

### 打卡清单

- [ ] 我知道 `State` 保存共享业务状态。
- [ ] 我知道 `Context` 保存运行时外部依赖。
- [ ] 我知道为什么 Repository 不直接塞进 State。
- [ ] 我知道 `recall_column` 依赖 embedding + Qdrant。
- [ ] 我知道 `recall_value` 依赖 Elasticsearch。
- [ ] 我知道 `recall_metric` 依赖 embedding + Qdrant。
- [ ] 我知道 `error` 字段会影响后续分支。
- [ ] 我注意到当前部分节点仍是章节式占位骨架。

### 今日输出

- [ ] 整理一张状态字段变化表。
- [ ] 至少整理 4 个节点的“读什么 / 用什么 / 写什么”。
- [ ] 写一段 150 字以内说明 `State` 和 `Context` 的区别。

### 推荐表格模板

| 节点             | 读取哪些 state | 使用哪些 context                           | 写回哪些 state         |
| ---------------- | -------------- | ------------------------------------------ | ---------------------- |
| extract_keywords | query          | -                                          | keywords               |
| recall_column    | keywords       | embedding_client, column_qdrant_repository | retrieved_column_infos |
| recall_value     | keywords       | value_es_repository                        | retrieved_value_infos  |
| recall_metric    | keywords       | embedding_client, metric_qdrant_repository | retrieved_metric_infos |

### 自测问题

- [ ] 为什么图执行更适合把共享结果放在 State？
- [ ] 为什么外部依赖更适合放在 Context？
- [ ] 当前哪些节点还不是完整业务实现？

### 达标标准

- [ ] 我能讲清节点、状态和仓储层是怎么串起来的。

---

## Day 6 打卡：亮点、设计取舍、简历表达

### 今日目标

- 把技术理解转换成项目表达。
- 提炼亮点，也能诚实说明边界。

### 今日复盘素材

- Day 1 到 Day 5 的笔记
- 主链路图
- 状态变化表

### 打卡清单

- [ ] 我能概括这个项目的核心亮点。
- [ ] 我能说出混合检索的设计价值。
- [ ] 我能说出 LangGraph 图编排的价值。
- [ ] 我能说出工程分层的价值。
- [ ] 我能说出项目的适用场景。
- [ ] 我能说出项目的边界和风险。

### 今日输出

- [ ] 写 1 分钟项目介绍。
- [ ] 写 5 分钟项目介绍。
- [ ] 写 1 段简历项目描述。
- [ ] 写 3 条项目亮点。
- [ ] 写 3 条项目边界。

### 自测问题

- [ ] 为什么它比“直接 LLM 生成 SQL”更稳？
- [ ] 为什么它适合企业问数，不适合通用知识问答？
- [ ] 为什么这个项目值得写在简历上？

### 达标标准

- [ ] 我能用面试口吻自然讲出项目亮点和取舍。

---

## Day 7 打卡：复盘与最终输出

### 今日目标

- 把这周内容真正变成自己的表达体系。
- 做一次完整项目讲解演练。

### 打卡清单

- [ ] 我重新整理了这周所有笔记。
- [ ] 我用自己的话重写了项目架构说明。
- [ ] 我用自己的话重写了知识库构建链路。
- [ ] 我用自己的话重写了问数执行链路。
- [ ] 我整理了至少 10 个可能的面试问题。
- [ ] 我写下了这个项目后续可以深挖的方向。

### 今日输出

- [ ] 最终学习笔记一份。
- [ ] 面试问答清单一份。
- [ ] 后续深挖方向列表一份。
- [ ] 一次 5 分钟模拟讲解录音或讲稿。

### 自测问题

- [ ] 看到某个文件时，我能快速判断它属于哪一层吗？
- [ ] 我能把这个项目讲给没看过代码的人听懂吗？
- [ ] 我能说清它的亮点和局限，而不是只会背流程吗？

### 达标标准

- [ ] 我已经具备“能讲清项目、能写进简历、能应对基础面试追问”的水平。

---

## 最终通关清单

如果这一周结束时，下面这些你都能勾上，基本就说明你真的学进去了：

- [ ] 我能讲清项目为什么不是直接 LLM 写 SQL。
- [ ] 我能讲清 MySQL、Qdrant、Elasticsearch 的职责分工。
- [ ] 我能讲清知识库构建链路。
- [ ] 我能讲清 LangGraph 问数执行链路。
- [ ] 我能讲清 `State` 和 `Context` 的区别。
- [ ] 我能指出当前仓库哪些部分已落地，哪些更像课程骨架。
- [ ] 我能用 1 分钟介绍项目。
- [ ] 我能用 5 分钟介绍项目。
- [ ] 我能写出一段像样的简历项目描述。
- [ ] 我能说出适用场景和边界，而不是只讲优点。

如果你把这份清单和主学习指南一起用，这一周的学习会非常扎实，也更容易真正沉淀成你的项目表达能力。
