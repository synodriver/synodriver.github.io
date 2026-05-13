+++
date = '2026-05-12T22:43:15+08:00'
draft = false
title = 'Agno AI Agent框架架构与设计思路'
author = 'synodriver'
+++

# Agno AI Agent 框架架构与设计思路

## 概述

**Agno** 是一个功能完备的 AI Agent 框架（由 phidata 演进而来），用于构建具有记忆、知识、工具调用、推理、团队协作和工作流编排能力的智能体系统。它内置 50+ 模型提供商、130+ 工具、17+ 向量数据库，覆盖了从单 Agent 到多 Agent 团队再到复杂工作流编排的完整链路。本文基于 Agno 源码，深入剖析其架构设计与核心实现。

---

## 整体架构

Agno 的架构围绕三个核心执行单元构建，辅以可插拔的组件系统：

```
┌─────────────────────────────────────────────────────────────────────┐
│                         三大执行单元                                  │
│  ┌───────────┐   ┌───────────┐   ┌───────────────┐                  │
│  │   Agent    │   │   Team    │   │   Workflow    │                  │
│  │ (单智能体) │   │ (多智能体)│   │ (工作流编排)  │                  │
│  └─────┬─────┘   └─────┬─────┘   └──────┬────────┘                  │
│        │               │                │                            │
│        └───────────────┼────────────────┘                            │
│                        │                                             │
├────────────────────────┼─────────────────────────────────────────────┤
│                   可插拔组件层                                        │
│  ┌─────────┐ ┌─────────┐ ┌──────────┐ ┌─────────┐ ┌───────────┐     │
│  │  Model  │ │  Tools  │ │ Knowledge│ │ Memory  │ │ Reasoning │     │
│  │ (50+种) │ │ (130+种)│ │  (RAG)   │ │ (记忆)  │ │  (推理)   │     │
│  └─────────┘ └─────────┘ └──────────┘ └─────────┘ └───────────┘     │
│  ┌─────────┐ ┌─────────┐ ┌──────────┐ ┌─────────┐ ┌───────────┐     │
│  │   DB    │ │VectorDB │ │ Guardrail│ │  Eval   │ │  Hooks    │     │
│  │ (12+种) │ │ (17+种) │ │  (护栏)  │ │ (评估)  │ │  (钩子)   │     │
│  └─────────┘ └─────────┘ └──────────┘ └─────────┘ └───────────┘     │
└─────────────────────────────────────────────────────────────────────┘
```

---

## 一、顶层目录结构

Agno 包含约 40 个子模块，按职责划分如下：

| 分类 | 模块 | 说明 |
|------|------|------|
| **核心抽象** | `agent/`, `team/`, `workflow/` | 三大执行单元 |
| **模型层** | `models/` | 50+ 模型提供商适配 |
| **工具层** | `tools/` | 130+ 内置工具 |
| **知识层** | `knowledge/`, `vectordb/` | RAG 检索增强生成 |
| **数据持久化** | `db/` | 12+ 数据库实现 |
| **记忆系统** | `memory/` | 会话记忆管理 |
| **推理系统** | `reasoning/` | Chain-of-Thought 推理 |
| **会话管理** | `session/`, `run/` | 运行时状态 |
| **安全/护栏** | `guardrails/`, `approval/` | 输入输出安全 |
| **评估** | `eval/` | Agent 评估框架 |
| **基础设施** | `factory/`, `registry/`, `tracing/`, `utils/` | 工厂、注册、追踪 |
| **辅助** | `compression/`, `culture/`, `context/`, `hooks/`, `learn/`, `skills/` | 压缩、文化、学习等 |

---

## 二、Agent — 核心执行单元

### 2.1 类结构

`Agent` 是框架的核心类，使用 `@dataclass(init=False)` 装饰，拥有约 100+ 个配置字段：

```python
@dataclass(init=False)
class Agent:
    # 模型配置
    model: Optional[Model]
    fallback_models: Optional[List[Model]]
    
    # 会话管理
    session_id: Optional[str]
    session_state: Optional[Dict[str, Any]]
    
    # 知识系统
    knowledge: Optional[KnowledgeProtocol]
    knowledge_filters: Optional[Dict[str, Any]]
    
    # 工具系统
    tools: Optional[List[Union[Toolkit, Callable, Function]]]
    tool_call_limit: int
    
    # 记忆/推理/学习/文化
    memory_manager: Optional[MemoryManager]
    reasoning: Optional[Reasoning]
    learning: Optional[LearningMachine]
    culture_manager: Optional[CultureManager]
    
    # 输出控制
    output_schema: Optional[Union[Type[BaseModel], str]]
    structured_outputs: bool
    # ... 更多字段
```

### 2.2 Facade + Module Delegation 模式

Agent 类最关键的设计是 **Facade + Module Delegation**（门面 + 模块委托）模式。Agent 本身作为统一入口，将大量逻辑委托给私有模块：

```
Agent (Facade)
  ├── _init      → 初始化与模型获取
  ├── _run       → 运行逻辑 (run/arun/continue_run)
  ├── _tools     → 工具获取与管理
  ├── _messages  → 消息构建（系统消息、知识检索）
  ├── _session   → 会话管理
  ├── _storage   → 序列化/反序列化、持久化
  ├── _managers  → 记忆与文化管理
  ├── _hooks     → 钩子处理
  ├── _response  → 响应处理
  ├── _cli       → CLI 显示与交互
  └── _telemetry → 遥测数据
```

这种模式将一个可能超过万行的巨型类拆分为功能内聚的模块，每个模块独立维护，Agent 类本身仅保留字段定义和方法转发，是经典的企业级复杂度管理手段。

### 2.3 运行流程

Agent 的核心运行方法：

| 方法 | 说明 |
|------|------|
| `run()` / `arun()` | 同步/异步运行，支持流式/非流式 |
| `continue_run()` | 暂停后继续运行（HITL 场景） |
| `print_response()` | 运行并打印结果 |
| `cli_app()` | 交互式 CLI 应用 |

一次 `run()` 的简化流程：

```
run(message)
  │
  ├── 1. 初始化 (_init)
  │     └── 获取模型、创建会话、加载工具
  │
  ├── 2. 预处理钩子 (_hooks)
  │     └── pre_hooks 依次执行
  │
  ├── 3. 构建消息 (_messages)
  │     ├── 系统消息 (instructions + description)
  │     ├── 知识检索 (knowledge.build_context)
  │     └── 用户消息
  │
  ├── 4. Agent 循环 (_run)
  │     │
  │     ├── 4a. 模型调用 (model.response)
  │     ├── 4b. 推理 (reasoning_manager)
  │     ├── 4c. 工具调用 (function_call.execute)
  │     ├── 4d. 记忆更新 (memory_manager)
  │     └── 4e. 重复 4a-4d 直到无更多工具调用
  │
  ├── 5. 后处理钩子 (_hooks)
  │     └── post_hooks 依次执行
  │
  └── 6. 返回 RunResponse
```

---

## 三、Model — 模型抽象层

### 3.1 基类设计

```python
@dataclass
class Model(ABC):
    id: str
    name: Optional[str]
    provider: Optional[str]
    model_type: ModelType  # MODEL, OUTPUT_MODEL, PARSER_MODEL 等
    
    # 能力声明
    supports_native_structured_outputs: bool
    supports_json_schema_outputs: bool
    
    # 重试配置
    retries: int
    delay_between_retries: float
    exponential_backoff: bool
    
    # 缓存配置
    cache_response: bool
    cache_ttl: timedelta
    cache_dir: Optional[str]
```

Model 使用 **Template Method** 模式，基类定义了 `invoke()`, `ainvoke()`, `response()`, `aresponse()` 等抽象方法，子类只需实现与特定 API 的交互逻辑。

### 3.2 支持的模型提供商

Agno 支持 50+ 种模型提供商，按类型分布：

| 类别 | 提供商 |
|------|--------|
| **主流** | OpenAI, Anthropic(Claude), Google(Gemini), Groq, Mistral, Cohere |
| **云服务** | AWS Bedrock, Azure OpenAI, Azure AI Foundry, Google VertexAI |
| **开源/本地** | Ollama, LMStudio, LlamaCPP, vLLM, HuggingFace |
| **聚合** | LiteLLM, OpenRouter, DeepInfra, Fireworks, Together |
| **其他** | DeepSeek, Cerebras, SambaNova, IBM WatsonX, Nvidia, Perplexity |

每个提供商实现为一个独立的 Python 模块，遵循统一的 `Model` 接口：

```
models/
  ├── base.py          # Model ABC
  ├── openai/          # OpenAI 实现
  ├── anthropic/       # Anthropic 实现
  ├── google/          # Google Gemini 实现
  ├── ollama/          # Ollama 实现
  └── ...              # 50+ 提供商
```

### 3.3 模型分层

Agent 中可以使用不同类型的模型处理不同阶段：

- **`model`** — 主模型，处理对话
- **`reasoning_model`** — 推理专用模型（如 DeepSeek-R1）
- **`output_model`** — 输出格式化模型
- **`parser_model`** — 解析模型响应的模型
- **`fallback_models`** — 主模型失败后的备选列表

---

## 四、Team — 多智能体团队

### 4.1 类结构

```python
@dataclass(init=False)
class Team:
    members: Union[List[Union[Agent, Team]], Callable[..., List]]
    model: Optional[Model]
    # 与 Agent 类似的配置结构（共享 Module Delegation 模式）
```

Team 与 Agent 结构高度相似，但增加了 `_task_tools`（任务管理工具）和团队专用 `_default_tools`。关键区别在于 `members` 字段——一个 Agent 或 Team 的列表，支持递归嵌套。

### 4.2 团队模式 (TeamMode)

Agno 定义了四种团队协作模式：

```
┌──────────────────────────────────────────────────────────────┐
│                    coordinate (监督模式)                       │
│  Leader → 挑选成员 → 分配任务 → 收集结果 → 综合响应            │
│  适用：需要综合多方意见的复杂任务                                │
├──────────────────────────────────────────────────────────────┤
│                    route (路由模式)                            │
│  Leader → 分析意图 → 路由到专家 → 直接返回专家响应             │
│  适用：问题可明确分类，各专家独立解答                            │
├──────────────────────────────────────────────────────────────┤
│                    broadcast (广播模式)                        │
│  Leader → 将同一任务分配给所有成员 → 收集所有响应              │
│  适用：需要多视角答案或投票场景                                  │
├──────────────────────────────────────────────────────────────┤
│                    tasks (任务模式)                            │
│  Leader → 将目标分解为共享任务列表 → 成员认领执行 → 循环至完成  │
│  适用：大型项目需要任务分解与并行执行                            │
└──────────────────────────────────────────────────────────────┘
```

### 4.3 Team 运行流程

```
Team.run(message)
  │
  ├── 1. 选择成员 (coordinate/route 模式)
  │     └── 使用 Leader 模型决定分配给谁
  │
  ├── 2. 分配任务
  │     ├── coordinate: 选择 1-N 个成员，分配子任务
  │     ├── route: 路由到 1 个成员
  │     ├── broadcast: 分配给所有成员
  │     └── tasks: 分解为任务列表
  │
  ├── 3. 成员执行
  │     └── 各 Agent.run(sub_task)
  │
  └── 4. 综合响应 (coordinate/tasks 模式)
        └── Leader 模型综合所有成员结果
```

---

## 五、Workflow — 工作流编排

### 5.1 核心构建块

Workflow 是三大执行单元中最复杂的，支持多步骤编排、条件分支、循环和并行：

```
Workflow
  │
  ├── Step ──────────── 单步执行（Agent/Team/自定义函数/嵌套 Workflow）
  ├── Steps ─────────── 顺序步骤集合
  ├── Condition ──────── 条件分支 (if/else)
  ├── Router ─────────── 动态路由到不同步骤
  ├── Loop ──────────── 循环执行
  └── Parallel ──────── 并行执行
```

### 5.2 Step 设计

`Step` 是 Workflow 的基本执行单元：

```python
@dataclass
class Step:
    name: str
    # 四种执行器，任选其一
    agent: Optional[Agent]
    team: Optional[Team]
    executor: Optional[StepExecutor]  # Callable
    workflow: Optional[Workflow]     # 嵌套，最大深度10
    
    # 条件与控制
    condition: Optional[Callable]
    
    # Human-in-the-Loop
    reviewer: Optional[Callable]
    requires_confirmation: bool
    timeout: Optional[float]
```

Step 执行器类型签名：

```python
StepExecutor = Callable[[StepInput], Union[StepOutput, Iterator, Awaitable, AsyncIterator]]
```

### 5.3 工作流执行模式

```
┌─────────────────────────────────────────────────────────────────┐
│  顺序执行 (Steps)                                                │
│  Step1 → Step2 → Step3 → ... → Result                          │
├─────────────────────────────────────────────────────────────────┤
│  条件分支 (Condition)                                            │
│  Step → [Condition] → StepA (if true) / StepB (if false)       │
├─────────────────────────────────────────────────────────────────┤
│  循环 (Loop)                                                     │
│  Step → [Loop Condition] → Step (repeat) / Exit                 │
├─────────────────────────────────────────────────────────────────┤
│  并行 (Parallel)                                                 │
│  ┌─ StepA ─┐                                                    │
│  ├─ StepB ─┤ → Merge → Result                                   │
│  └─ StepC ─┘                                                    │
├─────────────────────────────────────────────────────────────────┤
│  路由 (Router)                                                   │
│  Step → [Router] → RouteA / RouteB / RouteC → ...               │
└─────────────────────────────────────────────────────────────────┘
```

### 5.4 Human-in-the-Loop (HITL)

Workflow 深度支持人工介入：

- **审核 (Review)**：Step 可指定 `reviewer`，执行后暂停等待人工审核
- **确认 (Confirmation)**：`requires_confirmation=True`，执行前需人工确认
- **超时 (Timeout)**：等待人工操作的超时处理
- **暂停/继续**：`pause()` / `continue_run()` 支持工作流中断与恢复

---

## 六、Tools — 工具系统

### 6.1 三层工具抽象

```
@tool 装饰器 → Function 对象 → 注册到 Toolkit
```

#### Function (Pydantic BaseModel)

```python
class Function(BaseModel):
    name: str
    description: Optional[str]
    parameters: Dict[str, Any]        # JSON Schema
    entrypoint: Optional[Callable]    # 实际执行函数
    
    # 钩子系统
    pre_hook: Optional[Callable]
    post_hook: Optional[Callable]
    tool_hooks: Optional[List[ToolHook]]
    
    # 控制流
    stop_after_tool_call: bool
    requires_confirmation: bool
    requires_user_input: bool
    external_execution: bool
    approval_type: Optional[ApprovalType]
    
    # 缓存
    cache_results: bool
    cache_dir: Optional[str]
    cache_ttl: Optional[int]
```

#### Toolkit

```python
class Toolkit:
    name: str
    tools: Sequence[Union[Callable, Function]]
    functions: Dict[str, Function]         # 同步函数
    async_functions: Dict[str, Function]   # 异步函数
```

Toolkit 自动扫描标记了 `@tool` 装饰器的方法，注册到 `functions` / `async_functions` 字典中。

### 6.2 FunctionCall — 运行时执行

```python
class FunctionCall(BaseModel):
    function: Function
    arguments: Optional[Dict[str, Any]]
    result: Optional[Any]
    
    def execute(self) -> FunctionExecutionResult
    async def aexecute(self) -> FunctionExecutionResult
```

### 6.3 嵌套钩子执行链 (Middleware Pattern)

`FunctionCall._build_nested_execution_chain()` 使用 `functools.reduce` 构建嵌套的钩子执行链：

```python
# 伪代码：每个 hook 包装下一个，entrypoint 在最内层
chain = reduce(
    lambda next_fn, hook: hook.wrap(next_fn),
    tool_hooks,           # 外层
    entrypoint             # 内层（核心执行）
)
```

这是经典的 **Middleware/Chain of Responsibility** 模式，类似于 Django 中间件或 Express.js 中间件的洋葱模型：

```
Hook1.before → Hook2.before → ... → entrypoint → ... → Hook2.after → Hook1.after
```

### 6.4 自动参数注入

工具函数签名中若包含特殊参数名，框架会自动注入对应实例，并从 JSON Schema 中移除：

| 参数名 | 注入对象 |
|--------|----------|
| `agent` | 当前 Agent 实例 |
| `team` | 当前 Team 实例 |
| `run_context` | 运行时上下文 (RunContext) |
| `images` / `videos` / `audios` / `files` | 多模态输入 |

### 6.5 内置工具概览

约 130+ 种内置工具，覆盖：

| 类别 | 工具 |
|------|------|
| **搜索引擎** | Tavily, Serper, Brave, Exa, SearXNG |
| **代码执行** | E2B Sandbox, Daytona, Docker |
| **数据库** | Postgres, MySQL, DuckDB, Neo4j |
| **文件操作** | CSV, JSON, YAML, Excel, DOCX, PDF |
| **开发工具** | GitHub, GitLab, File, Shell |
| **通信** | Slack, Discord, Telegram, Email |
| **云服务** | AWS, Azure, GCP |
| **协议** | MCP (Model Context Protocol) |

---

## 七、Knowledge — 知识库系统 (RAG)

### 7.1 Protocol-driven 设计

Knowledge 使用 `typing.Protocol` 定义接口：

```python
@runtime_checkable
class KnowledgeProtocol(Protocol):
    def build_context(self, **kwargs) -> str: ...
    def get_tools(self, **kwargs) -> List[Callable]: ...
    async def aget_tools(self, **kwargs) -> List[Callable]: ...
    # 可选: retrieve(), aretrieve()
```

使用 `Protocol` 而非 `ABC`，允许任何实现了这些方法的对象作为 Knowledge 注入，无需继承关系——这是 **Structural Subtyping（结构化子类型）** 的体现。

### 7.2 Knowledge 类结构

```python
@dataclass
class Knowledge(RemoteKnowledge):
    vector_db: Optional[VectorDb]        # 向量存储
    contents_db: Optional[BaseDb]        # 原文存储
    max_results: int = 10
    
    # 子系统
    readers: Optional[Dict[str, Reader]]     # 文档读取器
    embedder: Optional[Embedder]             # 嵌入模型
    chunking_strategy: ChunkingStrategy      # 分块策略
    reranker: Optional[Reranker]             # 重排序器
```

### 7.3 知识库子系统

```
Knowledge (RAG Pipeline)
  │
  ├── Reader (20+ 种) ──────── 文档加载
  │   ├── PDF, DOCX, CSV, Excel, ArXiv
  │   ├── Web, YouTube
  │   └── S3, Azure Blob, GCS, SharePoint
  │
  ├── ChunkingStrategy (8 种) ── 文档分块
  │   ├── Agentic, Code, Document, Fixed
  │   ├── Markdown, Recursive, Row, Semantic
  │   └── (通过 ChunkingStrategyFactory 创建)
  │
  ├── Embedder (15+ 种) ────── 文本向量化
  │   ├── OpenAI, Google, Cohere, AWS Bedrock
  │   ├── Ollama, HuggingFace, Jina, Mistral
  │   └── VoyageAI
  │
  ├── VectorDb (17+ 种) ────── 向量存储与检索
  │   ├── PgVector, ChromaDB, Qdrant, Pinecone
  │   ├── Milvus, MongoDB, LanceDB, Weaviate
  │   └── Redis, ClickHouse, Cassandra, Upstash
  │
  └── Reranker (4 种) ──────── 结果重排序
      ├── AWS Bedrock, Cohere, Infinity
      └── Sentence Transformer
```

### 7.4 知识检索流程

```
用户查询
  │
  ├── 1. embedder.embed(query)          → 查询向量化
  ├── 2. vector_db.search(embedding)     → 向量相似性搜索
  ├── 3. reranker.rerank(results)       → 重排序（可选）
  ├── 4. contents_db.get(contents)       → 获取原文
  └── 5. 组装为上下文字符串 → 注入系统消息
```

---

## 八、VectorDB — 向量数据库抽象

### 8.1 基类接口

```python
class VectorDb(ABC):
    # 生命周期
    def create(self) -> None: ...
    async def async_create(self) -> None: ...
    def drop(self) -> None: ...
    def exists(self) -> bool: ...
    
    # 数据操作
    def insert(self, content_hash, documents, filters) -> None: ...
    def upsert(self, content_hash, documents, filters) -> None: ...
    def search(self, query, limit, filters) -> List[Document]: ...
    async def async_search(self, query, limit, filters) -> List[Document]: ...
    
    # 删除操作
    def delete(self) -> None: ...
    def delete_by_id(self, id) -> None: ...
    def delete_by_name(self, name) -> None: ...
    def delete_by_metadata(self, metadata) -> None: ...
    def delete_by_content_id(self, content_id) -> None: ...
    
    # 辅助
    def name_exists(self, name) -> bool: ...
    def optimize(self) -> None: ...
    def update_metadata(self, id, metadata) -> None: ...
```

### 8.2 Document 模型

所有 VectorDB 实现统一操作 `Document` 对象：

```python
@dataclass
class Document:
    id: Optional[str]
    name: Optional[str]
    content: Optional[str]          # 原文
    embed_text: Optional[str]       # 用于嵌入的文本
    embedding: Optional[List[float]]
    meta_data: Optional[Dict]
    usage: Optional[int]            # token 使用量
```

---

## 九、DB — 持久化层

### 9.1 基类设计

`BaseDb` 定义了 15+ 张表的完整 CRUD 操作：

```python
class BaseDb(ABC):
    # 会话管理
    def get_session(self, session_id) -> Optional[SessionRow]: ...
    def save_session(self, session) -> None: ...
    def delete_session(self, session_id) -> None: ...
    
    # 记忆管理
    def get_memories(self, user_id) -> List[Memory]: ...
    def add_memory(self, memory) -> None: ...
    def update_memory(self, memory) -> None: ...
    def delete_memory(self, memory_id) -> None: ...
    
    # 知识内容
    def get_knowledge_content(self, id) -> Optional[KnowledgeRow]: ...
    def save_knowledge_content(self, content) -> None: ...
    
    # 追踪 (OpenTelemetry 风格)
    def get_trace(self, trace_id) -> Optional[TraceRow]: ...
    def save_trace(self, trace) -> None: ...
    def get_span(self, span_id) -> Optional[SpanRow]: ...
    def save_span(self, span) -> None: ...
    
    # 组件版本管理 (Agent/Team/Workflow 配置版本化)
    def get_component(self, id) -> Optional[ComponentRow]: ...
    def save_component(self, component) -> None: ...
    def publish_component(self, id) -> None: ...   # draft → published
    
    # 学习记录、调度器、审批流 ...
```

`AsyncBaseDb` 提供完全对称的异步接口。

### 9.2 支持的数据库实现

共 12 种：

PostgreSQL, MySQL, SQLite, MongoDB, DynamoDB, Firestore, Redis, SingleStore, SurrealDB, GCS JSON, In-Memory, JSON 文件

### 9.3 组件版本管理

BaseDb 内置了 Agent/Team/Workflow 的配置版本化系统，支持 `draft` / `published` 生命周期：

```
创建组件 (draft) → 编辑配置 (draft) → 发布 (published)
                      ↑                    │
                      └──── 回滚 ←──────────┘
```

---

## 十、Memory — 记忆管理

### 10.1 MemoryManager 设计

```python
@dataclass
class MemoryManager:
    model: Optional[Model]
    db: Optional[Union[BaseDb, AsyncBaseDb]]
    # 能力开关
    add_memories: bool
    update_memories: bool
    delete_memories: bool
    clear_memories: bool
```

### 10.2 LLM 驱动的记忆管理

MemoryManager 本质上是一个 **"微型 Agent"**——它使用 LLM 来决定是否添加/更新/删除记忆：

```
对话内容
  │
  ├── 构建记忆管理提示词
  │   └── "根据以下对话，决定是否需要添加/更新/删除记忆..."
  │
  ├── 提供记忆管理工具
  │   ├── add_memory(content, topics)
  │   ├── update_memory(memory_id, content)
  │   ├── delete_memory(memory_id)
  │   └── clear_memories()
  │
  ├── LLM 决策
  │   └── 调用适当的记忆管理工具
  │
  └── 执行工具 → 写入数据库
```

### 10.3 搜索模式

| 模式 | 说明 |
|------|------|
| `last_n` / `first_n` | 按时间排序返回最近/最早的 N 条记忆 |
| `agentic` | 使用 LLM 做语义搜索，理解查询意图后筛选最相关的记忆 |

### 10.4 优化策略

```python
class MemoryOptimizationStrategy:
    SUMMARIZE  # 使用 LLM 总结冗余记忆，减少存储量
```

通过 `MemoryOptimizationStrategyFactory` 创建具体策略实例。

---

## 十一、Reasoning — 推理系统

### 11.1 两种推理模式

```
┌─────────────────────────────────────────────────────────────┐
│  原生推理模型 (Native Reasoning)                             │
│  DeepSeek-R1, Claude (extended thinking), OpenAI o1/o3      │
│  → 直接使用模型的 reasoning_content 字段                      │
├─────────────────────────────────────────────────────────────┤
│  默认 Chain-of-Thought (CoT)                                │
│  → 使用 ReasoningManager 驱动多步推理                        │
│  → 配置: min_steps, max_steps, reasoning_tools               │
└─────────────────────────────────────────────────────────────┘
```

### 11.2 ReasoningManager

```python
@dataclass
class ReasoningManager:
    model: Optional[Model]
    # ReasoningConfig: min_steps, max_steps, tools 等
    # ReasoningResult: message, steps
```

推理过程产生的事件：

```
started → content_delta → step → step → ... → completed / error
```

---

## 十二、其他核心模块

### 12.1 Guardrails（安全护栏）

```python
class BaseGuardrail(ABC):
    def check(self, run_input) -> None: ...      # 同步检查
    async def async_check(self, run_input) -> None: ...  # 异步检查
```

内置实现：
- `OpenAIModerationGuardrail` — OpenAI 内容审核
- `PIIDetectionGuardrail` — PII 个人信息检测
- `PromptInjectionGuardrail` — 提示注入检测

### 12.2 RunContext（运行上下文）

```python
@dataclass
class RunContext:
    run_id: str
    session_id: str
    user_id: Optional[str]
    dependencies: Optional[Dict[str, Any]]      # 外部依赖
    session_state: Optional[Dict[str, Any]]      # 会话状态
    messages: Optional[List[Message]]            # 实时消息引用
    # 运行时工厂解析结果
    tools: Optional[List]
    knowledge: Optional[Any]
    members: Optional[List]
```

`RunContext` 贯穿整个运行过程，工具函数可通过参数注入获取。

### 12.3 RunStatus（运行状态）

```
PENDING → RUNNING → COMPLETED
                  → PAUSED     (HITL 暂停)
                  → CANCELLED  (取消)
                  → ERROR      (异常)
```

---

## 十三、核心设计模式总结

### 13.1 架构模式一览

| 模式 | 应用位置 | 说明 |
|------|---------|------|
| **Facade + Module Delegation** | `Agent`, `Team` | 类本身是门面，逻辑委托给私有模块 |
| **Strategy Pattern** | Chunking, Embedder, Reranker, MemoryOptimization | 可插拔策略接口 |
| **Protocol (Structural Subtyping)** | `KnowledgeProtocol` | Python `typing.Protocol` 鸭子类型 |
| **Factory Pattern** | `BaseFactory`, `ChunkingStrategyFactory`, `MemoryOptimizationStrategyFactory` | 创建策略实例 |
| **Template Method** | `Model`, `VectorDb`, `BaseDb`, `Embedder` | ABC 定义模板，子类实现 |
| **Middleware/Chain of Responsibility** | `FunctionCall._build_nested_execution_chain()` | Hook 链式执行 |
| **Observer/Event** | `RunEvent`, `WorkflowRunEvent`, `ReasoningEvent` | 事件驱动流式输出 |
| **Repository** | `BaseDb`, `AsyncBaseDb` | 数据访问抽象层 |
| **Composite** | `Workflow` 嵌套 (Step 可包含嵌套 Workflow) | 递归组合 |

### 13.2 依赖注入模式

Agno 使用多种依赖注入方式：

| 注入方式 | 示例 | 说明 |
|---------|------|------|
| **构造器注入** | `Agent(model=..., db=..., knowledge=...)` | 所有组件通过构造函数参数注入 |
| **自动参数注入** | `def my_tool(agent: Agent, run_context: RunContext)` | 工具函数中的特殊参数名自动注入 |
| **Callable Factory** | `Agent(tools=get_tools, knowledge=get_knowledge)` | 运行时解析 Callable 获取组件 |
| **延迟初始化** | `LearningMachine`, `background_executor` | 首次访问时创建 |

### 13.3 异步双轨设计

整个框架严格保持同步/异步对偶：

```
Agent.run()           ←→  Agent.arun()
BaseDb                ←→  AsyncBaseDb
FunctionCall.execute() ←→  FunctionCall.aexecute()
Toolkit.functions     ←→  Toolkit.async_functions
VectorDb.search()     ←→  VectorDb.async_search()
```

### 13.4 事件系统

Agent 运行时的事件体系：

```
RunEvent:
  run_started
    → run_content (流式内容)
    → tool_call_started → tool_call_completed
    → reasoning_started → reasoning_step → reasoning_completed
    → memory_update_started → memory_update_completed
    → followups_started → followups_completed
  run_completed / run_paused / run_cancelled / run_error
```

---

## 十四、组件交互关系总图

```
┌─────────────────────────────────────────────────────────────────┐
│                          Agent                                   │
│  ┌──────────┐ ┌──────────┐ ┌────────────┐ ┌──────────────────┐  │
│  │  Model   │ │  Tools   │ │ Knowledge  │ │  MemoryManager   │  │
│  │ (LLM)   │ │(Function)│ │(RAG)       │ │ (LLM+DB)         │  │
│  └──────────┘ └──────────┘ └─────┬──────┘ └────────┬─────────┘  │
│                                  │                  │            │
│  ┌──────────┐ ┌──────────┐       │                  │            │
│  │Reasoning │ │Guardrails│       │                  │            │
│  │ (CoT)    │ │ (安全)   │       │                  │            │
│  └──────────┘ └──────────┘       │                  │            │
│                                  ▼                  ▼            │
│  ┌──────────┐ ┌──────────┐ ┌──────────┐     ┌──────────┐       │
│  │Culture   │ │Learning  │ │ VectorDb │     │  BaseDb  │       │
│  │Manager   │ │Machine  │ │(17+种)   │     │ (12+种)  │       │
│  └──────────┘ └──────────┘ └──────────┘     └──────────┘       │
└─────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────┐
│                           Team                                    │
│  ┌────────┐ ┌────────┐ ┌────────┐                                │
│  │ Agent1 │ │ Agent2 │ │ Agent3 │  ← members (可嵌套 Team)      │
│  └────────┘ └────────┘ └────────┘                                │
│  ┌────────────┐ ┌───────────────┐                                │
│  │ Leader     │ │  TaskTools    │  ← 任务分配与管理              │
│  │ Model      │ │  (共享任务列表)│                                │
│  └────────────┘ └───────────────┘                                │
└─────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────┐
│                         Workflow                                  │
│  ┌───────┐  ┌───────────┐  ┌───────────┐  ┌────────────┐        │
│  │ Step1 │→│ Condition │→│  Step2    │→│   Loop     │→ ...    │
│  │(Agent)│  │ (if/else) │  │ (Team)   │  │(Steps × N) │        │
│  └───────┘  └───────────┘  └───────────┘  └────────────┘        │
│  ┌─────────────────────────────────────┐                        │
│  │ Parallel: [StepA, StepB, StepC]     │  ← 并行执行            │
│  └─────────────────────────────────────┘                        │
│  ┌─────────────────────────────────────┐                        │
│  │ HITL: Reviewer / Confirmation / Timeout│ ← 人工介入          │
│  └─────────────────────────────────────┘                        │
└─────────────────────────────────────────────────────────────────┘
```

---

## 十五、总结

Agno 是一个**高度模块化、功能极其丰富**的 AI Agent 框架，其核心设计特点：

1. **Facade + Module Delegation**：通过将 Agent/Team 的逻辑拆分到独立模块（`_run`, `_tools` 等），在保持单一入口的同时管理巨大类的复杂度
2. **Protocol-driven 设计**：`KnowledgeProtocol` 等使用 `typing.Protocol`，允许鸭子类型扩展，无需继承约束
3. **全栈异步支持**：每个同步方法都有对应的异步版本，严格的双轨设计
4. **Middleware Pattern 工具链**：`FunctionCall` 的嵌套钩子执行链，灵活的工具调用前后拦截
5. **LLM 驱动的元操作**：MemoryManager 使用 LLM 决定记忆的增删改，ReasoningManager 使用 LLM 驱动推理
6. **内置生态极广**：50+ 模型提供商、130+ 工具、17+ 向量数据库、12+ 关系数据库
7. **企业级特性完备**：组件版本管理、审批流、调度器、评估框架、追踪系统、安全护栏
8. **深度 HITL 支持**：人工审核、确认、用户输入、外部执行，覆盖人机协作的关键场景
