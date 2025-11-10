# RAGAnything 后端集成指南

> 本指南面向需要在全新业务后端中复用 **RAGAnything** 流水线的工程团队。文档覆盖架构概览、安装部署、配置说明、核心 API、二次开发扩展点以及常见实践，旨在帮助团队快速完成二次封装与业务交付。

---

## 目录

- [1. 框架定位与整体架构](#1-框架定位与整体架构)
- [2. 安装与运行环境](#2-安装与运行环境)
- [3. 配置体系](#3-配置体系)
- [4. 快速上手示例](#4-快速上手示例)
- [5. 核心工作流详解](#5-核心工作流详解)
- [6. 查询能力](#6-查询能力)
- [7. 模型与处理器扩展](#7-模型与处理器扩展)
- [8. 存储与资源管理](#8-存储与资源管理)
- [9. 常见集成模式](#9-常见集成模式)
- [10. 调试与监控](#10-调试与监控)
- [11. 常见问题排查](#11-常见问题排查)
- [12. 下一步定制方向](#12-下一步定制方向)
- [13. 参考资源](#13-参考资源)

---

## 1. 框架定位与整体架构

RAGAnything 基于 [LightRAG](https://github.com/HKUDS/LightRAG) 构建，将「文档解析 → 多模态处理 → 检索问答」整合为统一的异步管线。核心数据结构位于 [`raganything/raganything.py`](../raganything/raganything.py)，`RAGAnything` 数据类通过 mixin 组合如下功能模块：

- **`ProcessorMixin`**（[`raganything/processor.py`](../raganything/processor.py)）：负责解析文档、拆分文本与多模态内容、缓存处理结果并调度多模态处理器。
- **`QueryMixin`**（[`raganything/query.py`](../raganything/query.py)）：封装纯文本与多模态检索接口，并提供 VLM 增强查询策略。
- **`BatchMixin`**（[`raganything/batch.py`](../raganything/batch.py)）：实现单文档与批量目录处理，内置并发控制、状态上报以及失败重试机制。

在运行期，`RAGAnything` 会：

1. 加载 `.env` 或系统变量，初始化 [`RAGAnythingConfig`](../raganything/config.py)；
2. 根据配置选择 MinerU 或 Docling 解析器，并准备工作目录、日志、缓存；
3. 创建或接管一个 `LightRAG` 实例，负责文本/实体/关系向量库、KV 缓存以及知识图谱；
4. 结合多模态处理器（图片/表格/公式/自定义）完成内容入库，并在查询时提供上下文增强能力。

---

## 2. 安装与运行环境

### 2.1 基础依赖安装

```bash
# 1. 克隆项目并进入目录
git clone https://github.com/HKUDS/RAG-Anything.git
cd RAG-Anything

# 2. 创建虚拟环境（推荐）
python -m venv .venv
source .venv/bin/activate  # Windows 使用 .venv\Scripts\activate

# 3. 安装依赖
pip install -U pip wheel
pip install -e .  # 或者 pip install -r requirements.txt
```

> **提示**：MinerU 解析器依赖 LibreOffice、Poppler 等原生组件，详见 `docs/offline_setup.md`；Docling 解析器对纯 Python 依赖更友好，可在配置中切换。

### 2.2 环境变量与工作目录

- 默认工作目录：`./rag_storage`（可通过 `WORKING_DIR` 覆盖）；
- 解析输出目录：`./output`（`OUTPUT_DIR`）；
- `.env` 文件会被自动加载，系统环境变量拥有更高优先级；
- 对离线环境，建议配置 `TIKTOKEN_CACHE_DIR`、`HF_HOME` 等缓存路径以避免网络访问。

---

## 3. 配置体系

所有配置均通过 [`RAGAnythingConfig`](../raganything/config.py) 管理，可由 `.env`、系统变量或直接实例化时传参设置。关键选项如下：

| 类别 | 字段 | 说明 | 默认值 |
| --- | --- | --- | --- |
| 目录 | `WORKING_DIR` | LightRAG 存储与缓存目录 | `./rag_storage` |
| 目录 | `OUTPUT_DIR` | 解析输出目录 | `./output` |
| 解析 | `PARSE_METHOD` | MinerU 解析策略（`auto`/`ocr`/`txt`） | `auto` |
| 解析 | `PARSER` | 解析器选择（`mineru` 或 `docling`） | `mineru` |
| 解析 | `DISPLAY_CONTENT_STATS` | 是否输出解析统计 | `True` |
| 多模态 | `ENABLE_IMAGE_PROCESSING` | 是否处理图片内容 | `True` |
| 多模态 | `ENABLE_TABLE_PROCESSING` | 是否处理表格内容 | `True` |
| 多模态 | `ENABLE_EQUATION_PROCESSING` | 是否处理公式内容 | `True` |
| 批处理 | `MAX_CONCURRENT_FILES` | 同时处理的文件数 | `1` |
| 批处理 | `SUPPORTED_FILE_EXTENSIONS` | 支持的扩展名 | 多格式列表 |
| 批处理 | `RECURSIVE_FOLDER_PROCESSING` | 目录处理时是否递归 | `True` |
| 上下文 | `CONTEXT_WINDOW` | 上下文窗口（按页/块） | `1` |
| 上下文 | `CONTEXT_MODE` | 上下文模式（`page`/`chunk`） | `page` |
| 上下文 | `MAX_CONTEXT_TOKENS` | 上下文最大 Token 数 | `2000` |
| 上下文 | `INCLUDE_HEADERS`/`INCLUDE_CAPTIONS` | 是否包含标题/脚注 | `True` |
| 上下文 | `CONTEXT_FILTER_CONTENT_TYPES` | 上下文可见内容类型 | `text` |

> 兼容性：旧的 `MINERU_PARSE_METHOD` 会被自动映射到 `PARSE_METHOD` 并提示弃用。

### 3.1 运行时动态调整

- 配置实例可直接传入 `RAGAnything` 构造函数；
- 在长生命周期服务中，可根据请求参数临时修改 `rag.config` 字段，但需注意线程/协程安全；
- 多模态开关支持热切换（例如根据模型可用性动态关闭表格处理）。

---

## 4. 快速上手示例

```python
import asyncio
from raganything.raganything import RAGAnything

async def main():
    rag = RAGAnything(
        llm_model_func=my_llm_callable,
        vision_model_func=my_vlm_callable,
        embedding_func=my_embedding_callable,
    )

    # 1. 处理单个文档（解析 + 文本入库 + 多模态描述）
    await rag.process_document_complete("/path/to/report.pdf")

    # 2. 批量处理目录
    await rag.process_folder_complete("/data/reports")

    # 3. 执行检索问答
    answer = await rag.aquery("项目的核心指标是什么？")
    print(answer)

    # 4. 多模态检索
    answer = await rag.aquery_with_multimodal(
        "结合图表分析收入趋势",
        multimodal_content=[{"type": "image", "img_path": "/path/to/chart.png"}],
    )
    print(answer)

    await rag.finalize_storages()  # 可选：主动释放 LightRAG 资源

asyncio.run(main())
```

> **协程环境**：所有核心 API 均为异步方法，需在事件循环中调用。对于同步框架，可使用 `asyncio.run` 包装或引入异步执行器。

---

## 5. 核心工作流详解

### 5.1 LightRAG 初始化

- 当未手动传入 `LightRAG` 实例时，`RAGAnything` 会在首次调用解析或查询前自动创建实例，并接管向量库、KV 缓存、知识图谱等存储句柄；
- 自定义 `LightRAG` 初始化参数可通过 `lightrag_kwargs` 传递，例如更换向量数据库或调节 Token 限制；
- 若业务需复用既有 `LightRAG`（例如多服务共享库），可直接将实例传入构造函数。

### 5.2 文档解析与缓存

1. `process_document_complete` 会调用 `parse_document`（`ProcessorMixin`）完成 MinerU/Docling 解析；
2. 解析结果通过 `LightRAG` 的 KV 存储按「文件路径 + 修改时间 + 配置签名」缓存，避免重复解析；
3. 文本内容使用 `insert_text_content`（封装自 LightRAG API）写入向量库，可通过 `split_by_character` 控制切块方式；
4. 多模态条目根据类型分发至对应处理器，由模型生成描述后再入库，并在文档状态表中记录进度。

### 5.3 批量处理策略

- `process_folder_complete`：简化目录遍历，支持扩展名过滤、递归与并发度控制；
- `process_documents_batch`：接受混合的文件/目录列表，适合调度平台或自定义任务队列；
- 每次处理会输出成功与失败列表，可在业务层记录日志或触发告警。

---

## 6. 查询能力

### 6.1 纯文本检索 `aquery`

- 参数 `mode` 透传给 LightRAG，支持 `local`、`global`、`hybrid`、`naive`、`mix`、`bypass`；
- 当实例存在 `vision_model_func` 且未显式关闭 `vlm_enhanced` 时，会自动启用 **VLM 增强查询**：对检索结果中的图片路径进行 base64 编码，并调用视觉模型生成补充描述，提高回答质量；
- 其他 LightRAG 查询参数（如 `top_k`、`temperature` 等）可通过 `**kwargs` 传入。

### 6.2 多模态检索 `aquery_with_multimodal`

- `multimodal_content` 列表可包含图片、表格、公式等条目；
- 内部调用 `get_processor_for_type`、`encode_image_to_base64` 等工具完成预处理，并拼装 `QueryParam`；
- 自动生成查询缓存键，避免重复编码；
- 适合实现「图片问答」、「表格 QA」等上层业务能力。

---

## 7. 模型与处理器扩展

### 7.1 注入自定义模型

- `llm_model_func`：文本分析/总结模型，要求签名为 `Callable[[str], Awaitable[str]]` 或兼容的同步包装；
- `vision_model_func`：视觉模型，用于图片/多模态描述与 VLM 查询增强；
- `embedding_func`：文本嵌入函数，用于向量入库；
- 业务可通过装饰器、异步客户端或 SDK 连接自有模型服务（如 OpenAI、DashScope、内网推理服务等）。

### 7.2 多模态处理器

- 默认处理器包含 `ImageModalProcessor`、`TableModalProcessor`、`EquationModalProcessor`、`GenericModalProcessor`（[`raganything/modalprocessors`](../raganything/modalprocessors)）；
- 处理器共享 `ContextExtractor`（[`raganything/modalprocessors/base.py`](../raganything/modalprocessors/base.py)）提供的上下文窗口，可根据 `CONTEXT_MODE`/`CONTEXT_WINDOW`/`MAX_CONTEXT_TOKENS` 动态截断；
- 扩展自定义处理器：
  1. 继承 `BaseModalProcessor`，实现 `process_batch` / `supports` 等方法；
  2. 在 `_initialize_processors` 中注册新类型，或通过 `get_processor_supports` 查询当前能力；
  3. 在解析结果中标记对应类型，即可被新处理器消费。

---

## 8. 存储与资源管理

- `RAGAnything.finalize_storages()` 会并发关闭 LightRAG 的向量库、KV 存储、知识图谱等资源，适合在服务停止或长时间闲置时调用；
- 对于 Web 服务，可在应用生命周期钩子（如 FastAPI 的 `startup`/`shutdown`）中创建与释放实例；
- `close()` 方法已在 `__post_init__` 中通过 `atexit` 注册，确保进程退出时也能安全清理。

---

## 9. 常见集成模式

### 9.1 同步 Web 框架（Flask/Django）

- 建议在启动阶段创建全局事件循环或使用 `asyncio.run` 包裹异步任务；
- 可通过工作队列（如 Celery、RQ）将文档解析放入异步任务中执行；
- 查询接口可使用 `asyncio.run`，或迁移至原生异步框架（FastAPI、Quart）。

### 9.2 FastAPI 异步服务

- 在 `startup` 中初始化 `RAGAnything` 与模型客户端；
- 在路由中直接 `await` 异步方法；
- 在 `shutdown` 钩子中调用 `await rag.finalize_storages()`。

### 9.3 调度/批处理系统

- 使用 `BatchMixin.process_documents_batch` 接收一组任务，结合配置控制并发；
- `doc_status` 存储可以用于实现任务追踪、状态可视化；
- 对于失败文件可重试或触发人工介入。

---

## 10. 调试与监控

- 默认复用 LightRAG 的 `logger`，可通过 `logging` 配置输出到文件或监控系统；
- 解析与多模态处理流程会详细记录每个阶段，建议在测试环境开启 DEBUG 级别；
- 文档状态表（`lightrag.doc_status`）保存了文本/多模态处理是否完成、错误信息等，可用于健康检查。

---

## 11. 常见问题排查

| 问题 | 原因 | 排查建议 |
| --- | --- | --- |
| 解析耗时长 | MinerU 依赖 OCR/版面分析 | 检查 `PARSE_METHOD`、硬件加速、是否可切换 Docling |
| 多模态描述失败 | 模型未配置或服务不可达 | 确认 `vision_model_func`/`llm_model_func` 正常返回、网络连通性 |
| 查询报错 `LightRAG not initialized` | 未调用解析流程或未提前初始化 LightRAG | 在服务启动时调用 `await rag._ensure_lightrag_initialized()` 或先处理文档 |
| 图片丢失 | 解析结果未包含图片路径 | 确认 MinerU 输出目录、源文件路径是否可访问；必要时将图片复制到工作目录 |
| Token 超限 | 上下文窗口过大 | 调整 `MAX_CONTEXT_TOKENS` 或 `CONTEXT_WINDOW`，使用字符级截断 |

---

## 12. 下一步定制方向

- **知识图谱扩展**：结合 LightRAG `max_graph_nodes`、`addon_params` 实现实体筛选或权重调整；
- **自定义上下文策略**：通过继承 `ContextExtractor` 调整截断、排序逻辑，或在业务层注入固定上下文；
- **多语言支持**：解析阶段传入 `lang` 参数，或在模型调用中自动检测语言；
- **安全审查**：在模型调用前后增加敏感词过滤、权限校验；
- **增量更新**：通过比对解析缓存或文档哈希，仅对变更文件重新处理。

---

## 13. 参考资源

- LightRAG 官方文档与示例
- 项目目录下的 [`examples/`](../examples) 与 [`scripts/`](../scripts) 用于演示不同场景的流水线调用
- `docs/` 中其他指南（离线部署、上下文感知处理等）可作为拓展阅读

---

借助本指南，团队可以以 RAGAnything 为核心快速构建面向文本、多模态的检索增强型应用。建议在正式接入前先完成小规模验证，逐步调整解析策略、模型配置与上下文窗口，以满足具体业务需求。
