# Embedding Check - 向量模型准确率测试平台设计文档

## 概述

一个用于测试和评估向量（Embedding）模型召回准确率的平台。支持上传文档、解析分块、向量化存储、单次/批量召回测试、LLM 多维度打分、测试报告导出。通过 OpenAI 兼容协议统一管理 API，用户自行配置 API Key。

## 技术栈

| 层 | 技术 |
|---|------|
| 前端 | Next.js (App Router) + Ant Design + TypeScript |
| 后端 | Python FastAPI |
| 数据库 | SQLite（默认）/ PostgreSQL（可选） |
| ORM & 迁移 | SQLAlchemy + Alembic |
| 向量数据库 | Chroma（PersistentClient 内嵌模式） |
| 异步任务 | SQLite 任务表 + ThreadPoolExecutor |
| API 协议 | OpenAI 兼容（base_url + api_key + model） |
| 前端 UI 设计 | 使用 ui-ux-pro-max skill 设计 |

## 架构

```
┌─────────────────┐     REST API     ┌──────────────────────────────┐
│   Next.js 前端   │ ◄─────────────► │      FastAPI 后端             │
│   (Ant Design)  │                  │                              │
│   Port: 3000    │                  │  ┌─ API Routes              │
└─────────────────┘                  │  ├─ 文档解析 & Chunk 引擎     │
                                     │  ├─ 向量化 & Chroma 客户端    │
                                     │  ├─ 召回测试引擎              │
                                     │  ├─ LLM 打分引擎             │
                                     │  ├─ 异步任务管理器            │
                                     │  └─ 报告导出引擎              │
                                     │                              │
                                     │  SQLite/PG ── 业务数据+任务队列│
                                     │  Chroma ── 内嵌向量存储        │
                                     │  Port: 8000                  │
                                     └──────────────────────────────┘
```

- 前后端分离，Next.js 纯前端，FastAPI 提供所有 API
- Chroma 以 PersistentClient 模式内嵌在 FastAPI 进程中，数据持久化到本地目录
- SQLite/PostgreSQL 存储所有业务数据（文档、chunk、测试用例、任务、配置、评分结果）
- 异步任务通过数据库任务表 + 后台线程池实现，前端轮询进度
- 数据库通过 SQLAlchemy 支持 SQLite 和 PostgreSQL 切换，Alembic 管理迁移

## 数据模型

### api_configs — API 配置

| 字段 | 类型 | 说明 |
|------|------|------|
| id | Integer PK | |
| name | String | 配置名称 |
| model_type | String | embedding 或 llm |
| base_url | String | OpenAI 兼容 API 地址 |
| api_key | String | 加密存储 |
| model_name | String | 模型名称（如 text-embedding-3-small, gpt-4o） |
| created_at | DateTime | |
| updated_at | DateTime | |

### documents — 文档

| 字段 | 类型 | 说明 |
|------|------|------|
| id | Integer PK | |
| filename | String | 文件名 |
| file_path | String | 存储路径（UUID 命名，存于 uploads/ 目录） |
| file_type | String | txt/md/pdf/docx/xlsx |
| file_size | Integer | 文件大小（字节） |
| status | String | uploaded/parsing/parsed/failed |
| chunk_strategy | String | auto/custom/parent_child |
| chunk_params | JSON | 分块参数配置 |
| created_at | DateTime | |
| updated_at | DateTime | |

### chunks — 分块

| 字段 | 类型 | 说明 |
|------|------|------|
| id | Integer PK | |
| document_id | Integer FK | 关联文档 |
| parent_id | Integer FK (self) | 父 chunk ID（父子模式下子 chunk 指向父 chunk，null 表示顶层） |
| chunk_type | String | parent / child / normal |
| content | Text | chunk 文本内容 |
| chunk_index | Integer | 序号 |
| metadata | JSON | 来源页码、标题等元信息 |
| chroma_id | String | Chroma 中的向量 ID（仅可检索 chunk 有值） |
| embedding_model | String | 使用的 embedding 模型 |
| created_at | DateTime | |

### Chroma Collection 策略

- 每个 `(document_id, embedding_model)` 组合创建一个独立 collection
- Collection 命名规则：`doc_{document_id}_model_{model_config_id}`
- 同一文档用不同 embedding 模型向量化时，创建不同的 collection
- 重新分块时，先删除旧 collection 和旧 chunk 记录，再重新创建
- 父子模式下，仅子 chunk 存入 Chroma 用于检索，父 chunk 仅存在 SQLite 中提供上下文

### test_cases — 测试用例

| 字段 | 类型 | 说明 |
|------|------|------|
| id | Integer PK | |
| group_name | String | 用例分组名称（用于组织和筛选） |
| query | Text | 测试查询 |
| expected_chunks | JSON | 期望召回的内容（可选） |
| created_at | DateTime | |
| updated_at | DateTime | |

### test_tasks — 测试任务

| 字段 | 类型 | 说明 |
|------|------|------|
| id | Integer PK | |
| parent_task_id | Integer FK (self) | 父任务 ID（多模型对比时，子任务指向父任务） |
| name | String | 任务名称 |
| type | String | single / batch |
| status | String | pending/running/completed/failed/cancelled |
| progress | Integer | 0-100 |
| cancel_requested | Boolean | 是否请求取消（默认 false） |
| document_ids | JSON | 目标文档 ID 列表 |
| test_case_ids | JSON | 测试用例 ID 列表（批量测试） |
| embedding_model_id | Integer FK | 使用的 embedding 配置 |
| scoring_model_id | Integer FK | 使用的 LLM 配置 |
| scoring_mode | String | single/multi/custom |
| scoring_dimensions_snapshot | JSON | 打分时使用的维度快照（冻结维度定义） |
| config_params | JSON | Top-K、相似度阈值、并发数等 |
| error_message | Text | 失败时的错误信息 |
| created_at | DateTime | |
| updated_at | DateTime | |
| completed_at | DateTime | |

多模型对比测试时：创建一个父任务（parent_task_id=null），每个 embedding 模型生成一个子任务。父任务进度 = 子任务进度平均值。

### test_results — 测试结果

| 字段 | 类型 | 说明 |
|------|------|------|
| id | Integer PK | |
| task_id | Integer FK | 关联任务 |
| test_case_id | Integer FK | 关联测试用例 |
| query | Text | 查询内容 |
| retrieved_chunks | JSON | 召回的 chunk 列表 |
| similarity_scores | JSON | 各 chunk 相似度分数 |
| llm_scores | JSON | LLM 各维度评分 |
| llm_reasoning | Text | LLM 评分理由 |
| total_score | Float | 加权总分 |
| created_at | DateTime | |

### scoring_dimensions — 评分维度

| 字段 | 类型 | 说明 |
|------|------|------|
| id | Integer PK | |
| name | String | 维度名称 |
| description | Text | 维度描述 |
| max_score | Integer | 最大分值 |
| is_default | Boolean | 是否默认维度 |
| created_at | DateTime | |
| updated_at | DateTime | |

## 文档解析 & Chunk 引擎

### 文件解析（借鉴 Dify）

| 文件类型 | 解析库 | 输出 |
|---------|--------|------|
| TXT | 内置，自动检测编码 | 纯文本 |
| Markdown | 自定义解析器，按标题分割，保留代码块 | (标题, 内容) 元组 |
| PDF | pypdfium2 | 逐页文本 |
| Word (.docx) | python-docx | Markdown 格式（含表格） |
| Excel (.xlsx) | openpyxl | 每行转 `"列名":"值"` 键值对 |

### 三种 Chunk 策略

**1. 自动模式**
- 递归字符分割
- 默认参数：max_tokens=500, overlap=50
- 分隔符优先级：`\n\n` → `。` → `. ` → ` ` → `""`
- 逐级递归，找到能在 max_tokens 内切分的最大分隔符

**2. 自定义模式**
- 用户指定：max_tokens、overlap、固定分隔符
- 两步分割：先按固定分隔符切分，超长 chunk 再递归切分
- 递归使用同样的分隔符优先级

**3. 父子模式**
- 两层分割，各有独立的 max_tokens、overlap、分隔符参数
- 父 chunk 提供上下文，子 chunk 用于实际检索和嵌入
- 支持两种父模式：段落模式（正常分割）、全文模式（整个文档为一个父 chunk）

### 预处理（分块前）

- 替换无效符号、移除控制字符和无效 Unicode
- 可选：合并 3+ 连续空行为 2 行，合并多余空白
- 可选：移除 URL 和邮箱地址

### 处理流程

上传文件 → 类型检测 → 调用对应解析器 → 预处理 → 选择策略分块 → 调用 embedding API 向量化 → 存入 Chroma + SQLite

## 召回测试引擎

### 单次召回测试

- 用户输入一条 query，选择目标文档集和 embedding 模型
- 调用 embedding API 将 query 向量化
- 在 Chroma 中进行相似度检索，返回 Top-K 结果（K 可配置）
- 展示每条召回 chunk 的内容、相似度分数、来源文档
- 可直接触发 LLM 打分

### 批量召回测试

- 支持手动添加或文件导入（CSV/Excel）测试用例
- 用户可选择：
  - 一个或多个 embedding 模型（对比测试）
  - 一个或多个 LLM 打分模型
  - 打分模式（单维度 / 多维度 / 自定义）
  - Top-K、相似度阈值等参数
- 创建为异步任务，后台线程池执行
- 前端轮询进度，展示实时进度条（已完成数/总数）

### 异步任务管理

```
任务生命周期：pending → running → completed / failed

机制：
1. FastAPI 启动时创建后台线程池（ThreadPoolExecutor）
2. 定时扫描 pending 任务，提交到线程池
3. 任务执行中更新 progress 字段（0-100）
4. 前端 GET /api/tasks/{id}/progress 轮询（间隔 2 秒）
5. 支持取消：设置 cancel_requested=true，执行线程在每条 test_case 迭代间检查标志后退出，保留已完成的部分结果
6. 失败任务记录错误信息，支持重试
```

## LLM 打分引擎

### 三种打分模式

**1. 单维度 — 相关性评分**
- 对每个召回 chunk 打 0-5 分
- Prompt：给定 query 和 chunk 内容，评估相关性
- 输出：分数 + 一句话理由

**2. 多维度评分**
- 三个固定维度：相关性、完整性、准确性，各 0-5 分
- Prompt：要求 LLM 对每个维度分别打分并给出理由
- 输出：各维度分数 + 理由 + 加权总分（权重可配置）

**3. 自定义评分**
- 用户自定义维度名称、描述、分值范围
- 动态生成 Prompt，将自定义维度注入模板
- 输出：各自定义维度分数 + 理由 + 总分

### 打分流程

```
取召回结果 → 构造 Prompt（query + chunk + 评分维度）
→ 调用 LLM API（OpenAI 兼容协议，使用 response_format: { type: "json_object" }）
→ 解析结构化 JSON 响应（含 JSON 修复层，处理 markdown 代码围栏等异常格式）
→ 存入 test_results
```

### 批量打分优化

- 并发调用 LLM API（可配置并发数，默认 5）
- 失败自动重试（最多 3 次）
- 每完成一条更新任务进度

## 报告导出引擎

### Excel 报告（.xlsx）

- Sheet 1 — **总览**：任务名称、测试时间、embedding 模型、打分模型、打分模式、总体平均分、Top-K 配置
- Sheet 2 — **详细结果**：每行一条 query，列包含 query、召回 chunk 内容、相似度分数、各维度 LLM 评分、理由
- Sheet 3 — **模型对比**（批量多模型测试时）：不同 embedding 模型的平均分、各维度对比
- 使用 openpyxl 生成

### PDF 报告

- 封面页：项目名、测试概要信息
- 统计图表：平均分分布、各维度雷达图、模型对比柱状图
- 详细结果表格
- 使用 reportlab 生成

### 导出流程

前端点击导出 → 选择格式（Excel/PDF）→ 后端异步生成文件 → 返回下载链接 → 文件存储在临时目录，定时清理

## 前端页面结构

```
├── 首页/仪表盘
│   └── 最近测试任务概览、快速入口
│
├── 配置管理 (/settings)
│   └── API Key 配置（增删改查）
│       ├── 名称、base_url、api_key、模型类型(embedding/llm)
│       └── 连接测试按钮
│
├── 文档管理 (/documents)
│   ├── 文档列表（上传、删除、查看状态）
│   ├── 文档详情 → chunk 列表、chunk 策略选择
│   └── 重新分块（选择不同策略和参数）
│
├── 测试用例 (/test-cases)
│   ├── 用例列表（增删改查）
│   ├── 手动添加 query + 期望结果
│   └── 文件导入（CSV/Excel）
│
├── 召回测试 (/tests)
│   ├── 单次测试页 — 输入 query，选模型，查看结果
│   ├── 批量测试页 — 选用例集、选模型组合、配置参数、启动
│   └── 任务列表 — 进度、状态、查看结果、导出报告
│
├── 测试结果 (/results)
│   ├── 结果详情 — 每条 query 的召回和评分详情
│   ├── 模型对比视图 — 不同模型的评分对比
│   └── 导出报告（Excel/PDF）
│
└── 评分维度管理 (/scoring)
    └── 自定义评分维度的增删改查
```

### 交互要点

- 批量测试创建时，支持多选 embedding 模型和 LLM 模型形成交叉组合
- 任务进度用进度条 + 已完成数/总数展示，轮询间隔 2 秒
- 测试结果页支持筛选、排序、分页
- 使用 ui-ux-pro-max skill 设计前端 UI

## 项目目录结构

```
embedding-check/
├── backend/
│   ├── alembic/                  # 数据库迁移
│   │   ├── versions/
│   │   └── env.py
│   ├── app/
│   │   ├── main.py               # FastAPI 入口
│   │   ├── config.py             # 配置（SQLite/PostgreSQL 切换）
│   │   ├── database.py           # SQLAlchemy 引擎 & Session
│   │   ├── models/               # ORM 模型
│   │   ├── schemas/              # Pydantic 请求/响应模型
│   │   ├── api/                  # 路由
│   │   │   ├── configs.py
│   │   │   ├── documents.py
│   │   │   ├── chunks.py
│   │   │   ├── test_cases.py
│   │   │   ├── tests.py
│   │   │   ├── tasks.py
│   │   │   ├── scoring.py
│   │   │   └── reports.py
│   │   ├── services/             # 业务逻辑
│   │   │   ├── document_parser.py
│   │   │   ├── chunking/
│   │   │   │   ├── auto.py
│   │   │   │   ├── custom.py
│   │   │   │   └── parent_child.py
│   │   │   ├── embedding.py
│   │   │   ├── retrieval.py
│   │   │   ├── scoring.py
│   │   │   ├── task_manager.py
│   │   │   └── report_export.py
│   │   └── utils/
│   │       ├── crypto.py          # API Key 加密
│   │       └── chroma_client.py
│   ├── requirements.txt
│   └── alembic.ini
├── frontend/
│   ├── src/
│   │   ├── app/                   # Next.js App Router
│   │   │   ├── page.tsx           # 仪表盘
│   │   │   ├── settings/
│   │   │   ├── documents/
│   │   │   ├── test-cases/
│   │   │   ├── tests/
│   │   │   ├── results/
│   │   │   └── scoring/
│   │   ├── components/
│   │   ├── lib/
│   │   │   └── api.ts             # 后端 API 客户端
│   │   └── types/
│   ├── package.json
│   └── next.config.js
├── docker-compose.yml             # 可选的容器化部署
├── README.md                      # 部署方法（含 Chroma 部署）
└── .env.example
```

## 关键依赖

### 后端 (Python)

| 包 | 用途 |
|---|------|
| fastapi + uvicorn | Web 框架 |
| sqlalchemy + alembic | ORM + 迁移 |
| chromadb | 向量数据库 |
| openai | OpenAI 兼容 API 客户端 |
| pypdfium2 | PDF 解析 |
| python-docx | Word 解析 |
| openpyxl | Excel 解析 + 报告生成 |
| reportlab | PDF 报告生成 |
| cryptography | API Key 加密 |
| python-multipart | 文件上传 |

### 前端 (TypeScript)

| 包 | 用途 |
|---|------|
| next | 框架 |
| antd + @ant-design/icons | UI 组件库 |
| axios | HTTP 客户端 |
| dayjs | 日期处理 |

## 技术细节

### 数据库并发

- SQLite 模式下强制启用 WAL（Write-Ahead Logging）模式，设置 `check_same_thread=False`
- 非高并发场景 SQLite 够用；高并发建议切换 PostgreSQL
- SQLAlchemy 连接池配置：SQLite 使用 `StaticPool`，PostgreSQL 使用默认连接池

### CORS

- FastAPI 启用 `CORSMiddleware`，开发环境允许 `http://localhost:3000`
- 生产环境通过环境变量 `CORS_ORIGINS` 配置允许的源

### API 分页

- 所有列表接口支持分页：`GET /api/xxx?page=1&page_size=20`
- 返回格式：`{ items: [], total: N, page: N, page_size: N }`

### 文件存储

- 上传文件存储在 `backend/uploads/` 目录，以 UUID 命名避免冲突
- 导出报告存储在 `backend/exports/` 目录，定时清理（启动时清理 24 小时前的文件）
- 最大上传文件大小：50MB（可通过环境变量配置）

### Embedding 向量化

- 使用 OpenAI 兼容客户端调用 embedding API，预计算向量后传入 Chroma
- 查询时必须使用与 collection 相同的 embedding 模型计算 query 向量
- Token 计数使用字符数近似（1 token ≈ 2 中文字符 / 4 英文字符），不依赖特定 tokenizer

### 设计约束

- 单用户本地工具，无认证层
- 不做水平扩展设计
