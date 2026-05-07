# Shopkeeper Agent 代码走读指南

> 面试速成文档，按执行顺序从配置到构建到查询完整走读

---

## 一、配置层

### `conf/app_config.yaml` → `app/conf/app_config.py`

YAML 定义所有基础设施地址，Python 用 OmegaConf 加载：

```
app_config.yaml → OmegaConf.load() → OmegaConf.merge(schema, context) → AppConfig dataclass
```

- `.env` 里的 `LLM_API_KEY` 通过 `${oc.env:LLM_API_KEY}` 注入 YAML
- 所有模块只需 `from app.conf.app_config import app_config` 就能拿到配置
- 支持的配置项：Logging、MySQL(meta/dw)、Qdrant、Embedding、Elasticsearch、LLM

配置结构：

```python
@dataclass
class AppConfig:
    logging: LoggingConfig      # 日志配置
    db_meta: DBConfig           # 元数据库连接 (meta库)
    db_dw: DBConfig             # 数据仓库连接 (dw库)
    qdrant: QdrantConfig        # Qdrant向量库 (embedding_size=1024)
    embedding: EmbeddingConfig  # Embedding服务 (BAAI/bge-large-zh-v1.5)
    es: ESConfig                # Elasticsearch
    llm: LLMConfig              # 大模型 (SiliconFlow GLM-5.1)
```

### `conf/meta_config.yaml` → `app/conf/meta_config.py`

定义**哪些表、字段、指标要纳入知识库**：

```yaml
tables:
  - name: dim_region          # 地区维度表
    columns:
      - name: region_name
        alias: [地区, 区域, 大区]  # 别名会被向量化存入Qdrant
        sync: true              # true = 把真实值同步到ES
  - name: fact_order           # 订单事实表
    columns:
      - name: order_amount
        role: measure
        alias: [销售额, 订单金额, 收入]

metrics:
  - name: GMV                  # 成交总额
    relevant_columns: [fact_order.order_amount]
    alias: [成交总额, 订单总额]
```

**面试考点**：`sync: true` 的字段（如省份、地区、品牌）才会把真实数据值写入 ES，供取值召回用。主键/外键设 `sync: false`。

配置结构：

```python
@dataclass
class ColumnConfig:
    name: str           # 字段名
    role: str           # 角色: primary_key / foreign_key / dimension / measure
    description: str    # 业务描述
    alias: list[str]    # 别名列表
    sync: bool          # 是否同步真实取值到ES

@dataclass
class TableConfig:
    name: str
    role: str               # dim / fact
    description: str
    columns: list[ColumnConfig]

@dataclass
class MetricConfig:
    name: str
    description: str
    relevant_columns: list[str]  # 依赖的底层字段 e.g. ["fact_order.order_amount"]
    alias: list[str]

@dataclass
class MetaConfig:
    tables: Optional[list[TableConfig]] = None
    metrics: Optional[list[MetricConfig]] = None
```

---

## 二、知识构建管线（离线，跑一次）

执行命令：

```bash
uv run python -m app.scripts.build_meta_knowledge.py -c conf/meta_config.yaml
```

### 整体流程

```
meta_config.yaml
       │
       ▼
build_meta_knowledge.py (入口，初始化所有客户端)
       │
       ▼
MetaKnowledgeService.build()
       │
       ├── ① _save_tables_to_meta_db     配置 → TableInfo/ColumnInfo → Meta MySQL
       ├── ② _save_column_info_to_qdrant ColumnInfo → 拆成多向量点 → Qdrant
       ├── ③ _save_value_info_to_es      sync=true字段 → 拉全量真实值 → ES
       ├── ④ _save_metrics_to_meta_db    配置 → MetricInfo/ColumnMetric → Meta MySQL
       └── ⑤ _save_metrics_to_qdrant     MetricInfo → 拆成多向量点 → Qdrant
```

### 入口：`app/scripts/build_meta_knowledge.py`

纯调度器，干三件事：

1. 初始化所有客户端（MySQL×2, Qdrant, ES, Embedding）
2. 创建 Repository 对象并注入 Service
3. 调用 `MetaKnowledgeService.build(config_path)`

```python
async def build(config_path: Path):
    # 1. 初始化客户端
    meta_mysql_client_manager.init()
    dw_mysql_client_manager.init()
    qdrant_client_manager.init()
    embedding_client_manager.init()
    es_client_manager.init()

    # 2. 创建 Repository
    meta_mysql_repository = MetaMySQLRepository(meta_session)
    dw_mysql_repository = DWMySQLRepository(dw_session)
    column_qdrant_repository = ColumnQdrantRepository(qdrant_client_manager.client)
    value_es_repository = ValueESRepository(es_client_manager.client)
    metric_qdrant_repository = MetricQdrantRepository(qdrant_client_manager.client)

    # 3. 注入 Service 并执行
    meta_knowledge_service = MetaKnowledgeService(...)
    await meta_knowledge_service.build(config_path)
```

### 核心：`app/services/meta_knowledge_service.py`

`build()` 方法按顺序执行五步：

**① _save_tables_to_meta_db — 表字段入库**

```python
for table in meta_config.tables:
    table_info = TableInfo(id=table.name, name=table.name, role=table.role, ...)
    # 从数仓查真实字段类型
    column_types = await self.dw_mysql_repository.get_column_types(table.name)
    for column in table.columns:
        # 从数仓查少量示例值
        column_values = await self.dw_mysql_repository.get_column_values(table.name, column.name)
        column_info = ColumnInfo(
            id=f"{table.name}.{column.name}",  # 如 "fact_order.order_amount"
            type=column_types[column.name],     # 从数仓补齐真实类型
            examples=column_values,             # 少量示例值
            ...
        )
# 写入 Meta MySQL
self.meta_mysql_repository.save_table_infos(table_infos)
self.meta_mysql_repository.save_column_infos(column_infos)
```

**② _save_column_info_to_qdrant — 字段向量索引**

**关键设计：一个字段拆成多个向量点**

```python
for column_info in column_infos:
    points.append({"embedding_text": column_info.name, "payload": asdict(column_info)})
    points.append({"embedding_text": column_info.description, "payload": asdict(column_info)})
    for alias in column_info.alias:
        points.append({"embedding_text": alias, "payload": asdict(column_info)})
```

这样用户说"销售额"、"订单金额"、"收入"都能命中同一个字段。向量化后分批写入 Qdrant。

**③ _save_value_info_to_es — 字段取值全文索引**

```python
for column_info in column_infos:
    if sync:  # 只有 sync=true 的字段才同步
        values = await self.dw_mysql_repository.get_column_values(
            column_info.table_id, column_info.name, 100000  # 拉全量
        )
        value_infos = [ValueInfo(id=f"{column_info.id}.{v}", value=v, column_id=column_info.id)
                       for v in values]
await self.value_es_repository.index(value_infos)
```

**④⑤ 指标入库 + 向量索引**，逻辑和字段类似。

---

## 三、客户端层（4 个 Manager）

| 文件 | 类 | 连接目标 | 驱动 |
|------|-----|----------|------|
| `qdrant_client_manager.py` | QdrantClientManager | Qdrant :6333 | qdrant-client (async) |
| `es_client_manager.py` | ESClientManager | ES :9200 | elasticsearch-py (async) |
| `mysql_client_manager.py` | MySQLClientManager | MySQL :3306 | sqlalchemy + asyncmy |
| `embedding_client_manager.py` | EmbeddingClientManager | TEI :8081 | HuggingFaceEndpointEmbeddings |

每个 Manager 都是**单例模式**：模块级创建实例 → `.init()` 延迟建连 → `.close()` 释放资源。

MySQL 有两个实例：`meta_mysql_client_manager`（元数据库）和 `dw_mysql_client_manager`（数仓）。

```python
# 模块级单例
qdrant_client_manager = QdrantClientManager(app_config.qdrant)

class QdrantClientManager:
    def init(self):
        self.client = AsyncQdrantClient(url=self._get_url())
    async def close(self):
        await self.client.close()
```

---

## 四、Repository 层

### `column_qdrant_repository.py` / `metric_qdrant_repository.py`

结构几乎一样：

- `ensure_collection()` — 建集合（维度=1024，余弦距离）
- `upsert(ids, embeddings, payloads)` — 分批写入向量点
- `search(embedding, score_threshold=0.6, limit=20)` — 向量检索，返回实体列表

```python
async def search(self, embedding, score_threshold=0.6, limit=20) -> list[ColumnInfo]:
    result = await self.client.query_points(
        collection_name=self.collection_name,
        query=embedding,
        limit=limit,
        score_threshold=score_threshold,
    )
    return [ColumnInfo(**point.payload) for point in result.points]
```

### `value_es_repository.py`

- `ensure_index()` — 建索引（IK 分词器，`ik_max_word`）
- `index(value_infos)` — 批量写入
- `search(keyword, score_threshold=0.6)` — match 全文检索

```python
index_mappings = {
    "dynamic": False,
    "properties": {
        "id": {"type": "keyword"},
        "value": {"type": "text", "analyzer": "ik_max_word", "search_analyzer": "ik_max_word"},
        "column_id": {"type": "keyword"},
    },
}
```

**面试考点**：ES 用 IK 分词器处理中文，Qdrant 用 bge-large-zh 做向量。两者解决不同类型的问题——语义匹配 vs 精确文本匹配。

---

## 五、Entity 层（5 个 dataclass）

```python
@dataclass
class ColumnInfo:           # 字段元数据
    id: str                 # "fact_order.order_amount"
    name: str               # "order_amount"
    type: str               # "decimal" (从数仓补齐)
    role: str               # "measure" / "dimension" / "primary_key" / "foreign_key"
    examples: list[Any]     # ["199.00", "50.00"] 少量示例值
    description: str        # "订单金额"
    alias: list[str]        # ["销售额", "订单金额", "收入"]
    table_id: str           # "fact_order"

@dataclass
class MetricInfo:           # 业务指标
    id: str                 # "GMV"
    name: str               # "GMV"
    description: str        # "所有订单的成交金额总和"
    relevant_columns: list[str]  # ["fact_order.order_amount"]
    alias: list[str]        # ["成交总额", "订单总额"]

@dataclass
class ValueInfo:            # 字段取值
    id: str                 # "dim_region.region_name.华北"
    value: str              # "华北"
    column_id: str          # "dim_region.region_name"

@dataclass
class TableInfo:            # 表元数据
    id: str                 # "fact_order"
    name: str
    role: str               # "dim" / "fact"
    description: str

@dataclass
class ColumnMetric:         # 字段-指标关联
    column_id: str          # "fact_order.order_amount"
    metric_id: str          # "GMV"
```

Repository 把 Qdrant/ES 返回的 payload 直接 `ColumnInfo(**point.payload)` 还原成实体。

---

## 六、Agent 查询管线（在线）

### State vs Context 分离

```python
class DataAgentState(TypedDict):        # 节点间传递的业务数据
    query: str                          # "统计华北地区的销售总额"
    keywords: list[str]                 # ["华北", "销售总额", "统计"]
    retrieved_column_infos: list[ColumnInfo]
    retrieved_metric_infos: list[MetricInfo]
    retrieved_value_infos: list[ValueInfo]
    error: str                          # SQL校验错误

class DataAgentContext(TypedDict):      # 运行时外部依赖（不参与状态合并）
    column_qdrant_repository: ColumnQdrantRepository
    embedding_client: HuggingFaceEndpointEmbeddings
    metric_qdrant_repository: MetricQdrantRepository
    value_es_repository: ValueESRepository
```

**面试考点**：State 是 LangGraph 节点间共享的数据，Context 是运行时工具依赖。节点通过 `runtime.context["key"]` 访问外部工具。

### LangGraph 图编排（`app/agent/graph.py`）

```
START
  │
  ▼
extract_keywords              ← jieba TF-IDF 抽取关键词 + 原始 query 作为兜底
  │
  ├──▶ recall_column          ← LLM 扩展关键词 → Embedding → Qdrant 向量检索
  ├──▶ recall_value           ← LLM 扩展关键词 → ES 全文检索
  └──▶ recall_metric          ← LLM 扩展关键词 → Embedding → Qdrant 向量检索
  │    （三路并行 fan-out）
  ▼
merge_retrieved_info          ← 汇合点（fan-in）
  │
  ├──▶ filter_table           ← 筛选候选表
  └──▶ filter_metric          ← 筛选候选指标
  │    （并行）
  ▼
add_extra_context             ← 组装 SQL 生成上下文
  │
  ▼
generate_sql                  ← LLM 生成 SQL
  │
  ▼
validate_sql                  ← 校验 SQL
  │
  ├── error is None ──▶ run_sql ──▶ END
  └── error ≠ None  ──▶ correct_sql ──▶ run_sql ──▶ END
```

图编译代码：

```python
graph_builder = StateGraph(state_schema=DataAgentState, context_schema=DataAgentContext)

# 注册节点
graph_builder.add_node("extract_keywords", extract_keywords)
graph_builder.add_node("recall_column", recall_column)
# ... 其他节点

# 定义边
graph_builder.add_edge(START, "extract_keywords")
graph_builder.add_edge("extract_keywords", "recall_column")   # fan-out
graph_builder.add_edge("extract_keywords", "recall_value")
graph_builder.add_edge("extract_keywords", "recall_metric")
graph_builder.add_edge("recall_column", "merge_retrieved_info")  # fan-in
graph_builder.add_edge("recall_value", "merge_retrieved_info")
graph_builder.add_edge("recall_metric", "merge_retrieved_info")

# 条件分支
graph_builder.add_conditional_edges(
    source="validate_sql",
    path=lambda state: "run_sql" if state["error"] is None else "correct_sql",
    path_map={"run_sql": "run_sql", "correct_sql": "correct_sql"},
)

graph = graph_builder.compile()
```

### 各节点详解

**extract_keywords — 关键词抽取**

```python
async def extract_keywords(state, runtime):
    query = state["query"]
    # jieba TF-IDF 抽取，只保留有意义的词性
    allow_pos = ("n", "nr", "ns", "nt", "nz", "v", "vn", "a", "an", "eng", "i", "l")
    keywords = jieba.analyse.extract_tags(query, allowPOS=allow_pos)
    # 原始问题也加入，作为兜底检索入口
    keywords = list(set(keywords + [query]))
    return {"keywords": keywords}
```

**recall_column — 字段召回**

```python
async def recall_column(state, runtime):
    keywords = state["keywords"]
    query = state["query"]
    # 1. LLM 扩展关键词
    prompt = PromptTemplate(template=load_prompt("extend_keywords_for_column_recall"), ...)
    chain = prompt | llm | JsonOutputParser()
    result = await chain.ainvoke({"query": query})  # ["销售金额", "成交额"]
    # 2. 合并关键词
    keywords = set(keywords + result)
    # 3. 逐个 Embedding + Qdrant 检索
    column_info_map: dict[str, ColumnInfo] = {}
    for keyword in keywords:
        embedding = await embedding_client.aembed_query(keyword)
        current = await column_qdrant_repository.search(embedding)
        for info in current:
            if info.id not in column_info_map:
                column_info_map[info.id] = info
    # 4. 去重后写回 state
    return {"retrieved_column_infos": list(column_info_map.values())}
```

**recall_value / recall_metric** 结构相同，只是检索目标不同（ES vs Qdrant）。

### LLM 初始化（`app/agent/llm.py`）

```python
llm = init_chat_model(
    model=app_config.llm.model_name,     # "Pro/zai-org/GLM-5.1"
    model_provider="openai",              # OpenAI 兼容协议
    base_url=app_config.llm.base_url,     # "https://api.siliconflow.cn/v1"
    api_key=app_config.llm.api_key,
    temperature=0,                        # 确定性输出
)
```

---

## 七、Prompt 模板（`prompts/` 目录）

| 文件 | 用途 |
|------|------|
| `extend_keywords_for_column_recall.prompt` | 让 LLM 扩展出字段层面的检索词 |
| `extend_keywords_for_value_recall.prompt` | 让 LLM 扩展出字段值层面的检索词 |
| `extend_keywords_for_metric_recall.prompt` | 让 LLM 扩展出指标层面的检索词 |
| `filter_table_info.prompt` | 表过滤提示词 |
| `filter_metric_info.prompt` | 指标过滤提示词 |
| `generate_sql.prompt` | SQL 生成提示词 |
| `correct_sql.prompt` | SQL 修正提示词 |

通过 `load_prompt("name")` 加载：

```python
from app.prompt.prompt_loader import load_prompt
prompt_text = load_prompt("extend_keywords_for_column_recall")
```

---

## 八、数据模型（ORM，`app/models/`）

| 文件 | 对应表 | 说明 |
|------|--------|------|
| `table_info.py` | table_info | 表元数据 |
| `column_info.py` | column_info | 字段元数据 |
| `metric_info.py` | metric_info | 指标元数据 |
| `column_metric.py` | column_metric | 字段-指标关联 |

---

## 九、当前代码完成度

| 节点 | 状态 | 说明 |
|------|------|------|
| extract_keywords | **已实现** | jieba TF-IDF + 原始 query 兜底 |
| recall_column | **已实现** | LLM 扩展 + Qdrant 向量检索 |
| recall_metric | **已实现** | LLM 扩展 + Qdrant 向量检索 |
| recall_value | **已实现** | LLM 扩展 + ES 全文检索 |
| merge_retrieved_info | 占位 | `asyncio.sleep(0.5)` |
| filter_table | 占位 | 同上 |
| filter_metric | 占位 | 同上 |
| add_extra_context | 占位 | 同上 |
| generate_sql | 占位 | 同上 |
| validate_sql | 占位 | 同上 |
| correct_sql | 占位 | 同上 |
| run_sql | 占位 | 同上 |

知识构建管线**完整可用**，Agent 查询管线**关键词 + 三路召回已实现**，后续节点待补。

---

## 十、核心设计亮点（面试必讲）

### 1. 混合检索，不是纯 LLM 生成 SQL

- 直接让 LLM 生成 SQL → 表名字段名瞎编
- 这个项目：先从**元数据库**检索真实的表、字段、指标信息，再喂给 LLM 生成 SQL
- 核心差异：**用检索增强替代纯 prompt 工程**

### 2. 三路召回并行

- **字段**（Qdrant 向量）：用户说"销售额" → 映射到数据库字段 `sale_amount`
- **指标**（Qdrant 向量）：用户说"销售总额" → 映射到定义好的指标 `GMV`
- **取值**（ES 全文）：用户说"华北" → 匹配到数据库里真实存储的值 `华北区`

### 3. 每路召回都有 LLM 关键词扩展

- 用户说"销售总额"，LLM 扩展出 ["GMV", "成交额", "销售金额"]
- 提高召回率，解决同义词问题

### 4. State vs Context 分离

- `State`：LangGraph 节点间传递的业务数据（问题、关键词、召回结果）
- `Context`：运行时依赖（Qdrant/ES/Embedding 客户端），不参与状态合并

### 5. 一个字段多个向量入口

- 字段名、描述、每个别名分别建向量点
- 用户用任何一种说法都能命中同一字段

### 6. 向量检索 vs 全文检索的选择

- 字段名/指标名是**语义匹配**（"销售额"匹配"sale_amount"）→ 向量
- 字段取值是**精确文本匹配**（"华北"匹配"华北区"）→ ES 全文

---

## 十一、面试高频问题

**Q: 为什么不直接用 LLM 生成 SQL？**
> 表名、字段名、枚举值 LLM 容易编造。先检索元数据（哪些表、哪些字段、真实取值），再让 LLM 基于真实 schema 生成 SQL，准确率大幅提升。

**Q: 向量检索和全文检索为什么分开用？**
> 字段名/指标名是语义匹配（"销售额"匹配"sale_amount"），适合向量。字段取值是精确文本匹配（"华北"匹配"华北区"），适合 ES 全文检索。

**Q: 三路召回为什么并行？**
> LangGraph 的 fan-out：`extract_keywords` 后同时启动三路，互不依赖，利用 async 并行加速。

**Q: SQL 出错怎么办？**
> `validate_sql` → 有错进 `correct_sql`（把错误信息喂给 LLM 重新生成）→ 再执行。

**Q: 为什么用 jieba 做关键词抽取而不是直接用 LLM？**
> jieba TF-IDF 速度快、成本低，适合初步抽取。LLM 用在后续的关键词扩展环节，做更精准的同义词扩展。两者配合，各取所长。

**Q: 向量相似度阈值怎么设的？**
> `score_threshold=0.6`，低于这个分数的结果被过滤掉。太低会引入噪声，太高会漏召回。
