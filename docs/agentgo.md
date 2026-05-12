# 1、开发概念

## 1、基础

| 概念      | 它主要回答什么问题          | 在我们工程里的角色                  |
| :-------- | :-------------------------- | :---------------------------------- |
| Rule      | 什么事绝对不能乱来          | 基础规矩、红线、底线                |
| Skill     | 这件事具体应该怎么做        | 固定动作的标准操作手册              |
| Sub Agent | 复杂任务由谁分工处理        | 不同阶段的专业角色                  |
| Workflow  | 这些角色按什么顺序接力      | 前进、暂停、打回、重跑规则          |
| Scripts   | 最后谁来判断到底做没做好    | 统一门禁和事后验证                  |
| MCP       | AI 怎么安全接上外部工程系统 | 外接能力与工具链（含 Unity 等宿主） |

skills:

**精髓一：智能触发。**

YAML元数据里的description，会始终保持在AI的系统提示中。

**精髓二：渐进式披露（按需加载）。**

平时绝不占用脑容量，只在需要的时候才占用。平时绝不占用脑容量，只在需要的时候才占用。

**精髓三：行动导向与子代理。**

Skills不只是让AI"读说明书"。在Skill的指导下，Claude Code可以像人类一样敲命令行、搜索文件、运行测试脚本。

更厉害的是，碰到特别复杂的任务，AI可以召唤一个"子代理"（Subagent）——相当于它的"影分身"。

rule：

| 每次代码修改完成后必须编译 | 编译通过后必须跑测试               |
| -------------------------- | ---------------------------------- |
| 测试通过后必须跑事后验证   | 只有三步全部通过，任务才算真的完成 |



## 2、RAG

以下是 RAG 解决的四大问题的简要笔记：

- **幻觉**：避免模型编造事实 — 检索真实内容作为依据。
- **时效**：突破知识截止日期 — 实时检索最新信息。
- **私域**：融入内部/私有数据 — 无需微调，动态检索企业或个人知识库。
- **溯源**：提供答案来源引用 — 可核对原文，增强可信度。

```
流程：
  -> 文本解析
  -> LangChain 文本切分
  -> LangChain Embedding
  -> Qdrant 向量入库
  -> 用户提问
  -> Query Embedding
  -> Qdrant Top-K 检索
  -> 拼接上下文
  -> LangChain ChatOpenAI 生成答案
  -> 返回答案和引用来源
```







## 3、harness


理解：如何让 AI 在你的项目里，持续、稳定、规范、顺畅地做出你真正想要的结果。

业务场景、数据来源、chunk策略、召回链路、rerank、评估指标、失败case和成本/延迟优化。这样面试官追问时才有完整链路可讲。



## 4、向量检索

```
向量检索流程总结：

用户问题 → Embeddings 转换为向量 → 在 Milvus 中做 ANN 近似最近邻搜索
                                            ↓
                                      返回最相关的文档片段
```

```
MVP1
目标：实现一个能持续对话的客服机器人，具备以下能力：
    - 系统提示词定义客服角色
    - 循环对话（用户输入 → AI 回复）
    - 友好的欢迎语
    - 简单的意图识别（关键词匹配）
    - 退出指令
MVP2
- 滑动窗口记忆（记住最近 N 轮对话）
- 多会话管理（不同用户独立对话）
- 会话超时清理
- 会话记录导出

MVP3
```







# 2、项目实现

## 一、需求与场景定义（写什么文档？）

在写代码之前，通常需要产出：

- **需求文档**：明确知识库的文档类型（PDF/Word/扫描件/网页）、用户提问方式、期望的回答形式（纯文本/带引用/表格）、性能指标（延迟、QPS、并发量）。
- **技术方案设计文档**：包括架构图、数据流、模块划分、技术选型理由、风险评估。

**怎么写**：用 Markdown + 流程图（如 Mermaid），放在项目 `docs/` 目录下。

示例结构：

markdown

```
# 知识库问答系统设计文档
## 1. 背景与目标
## 2. 范围 - 支持的文档类型、用户角色
## 3. 整体架构
   - 数据流入：文档上传 → 解析 → 分块 → 向量化 → 存储
   - 查询流出：用户提问 → 检索 → 重排序 → 生成 → 流式返回
## 4. 模块详细设计
   - 文档解析服务
   - 分块策略
   - 向量化与索引
   - 检索与融合
   - LLM 生成与流式
   - 用户会话管理
   - 可观测性
## 5. 非功能性需求
   - 性能：P99 延迟 < 500ms
   - 可用性：99.5%
   - 安全性：数据隔离、防注入
## 6. 部署与运维
## 7. 风险评估（如 OCR 精度、幻觉控制）
```



------

## 二、文档预处理与解析（写代码）

这是第一个工程难点：把五花八门的文档转成可检索的文本。

**涉及方面**：

- 格式识别（PDF、DOCX、PPT、TXT、HTML、图片）
- 内容提取：PDF 用 `pypdf`/`pdfplumber`/`PyMuPDF`；DOCX 用 `python-docx`；扫描件 PDF 用 OCR（`pytesseract` + `pdf2image`）
- 表格提取：保留表格结构（转为 Markdown 表格或 JSON）
- 元信息提取：文件名、章节标题、页数、创建时间等

**怎么写**：

- 定义一个 `DocumentParser` 基类，每种格式一个子类，工厂模式加载。
- 输出统一结构：`{"content": str, "metadata": dict, "page_num": int}`

示例代码结构：

python

```
# parsers/base.py
class BaseParser(ABC):
    @abstractmethod
    def parse(self, file_path: str) -> List[ParsedPage]: pass

# parsers/pdf_parser.py
class PDFParser(BaseParser):
    def parse(self, file_path):
        # 尝试文本提取，若文本长度过小则触发 OCR
        ...

# 使用
parser = get_parser(file_extension)
pages = parser.parse(uploaded_file)
```



**工程注意**：异步处理大文件（放任务队列），避免阻塞 API。

------

## 三、文本分块（Chunking）

**涉及方面**：

- 固定大小分块（按字符数/token数，有重叠）
- 语义分块（基于句子边界、段落或递归分割）
- 针对表格、代码块、列表的特殊处理（保持结构不被打碎）
- 保留元信息（来源文档、章节路径）

**怎么写**：

- 常用库：`langchain.text_splitter`，或自己实现递归分割。
- 关键配置：`chunk_size`（如 512 tokens）、`chunk_overlap`（如 50 tokens）
- 输出：每个 chunk 包含 `text`、`metadata`（含文档 ID、页码、标题路径）、`embedding`（待填充）

python

```
from langchain.text_splitter import RecursiveCharacterTextSplitter

splitter = RecursiveCharacterTextSplitter(
    chunk_size=500,
    chunk_overlap=50,
    separators=["\n\n", "\n", "。", "；", " ", ""]
)
chunks = splitter.split_documents(documents)
```



**优化点**：对于技术手册，可根据标题层级（Markdown `#`）进行**父子块**结构，检索时先粗筛父块再细看子块。

------

## 四、向量化（Embedding）

**涉及方面**：

- 选择 Embedding 模型（BGE、OpenAI `text-embedding-3-small`、M3E、GTE 等）
- 批量向量化提升吞吐
- 缓存相同文本的向量（避免重复计算）
- 维度对齐（模型输出维度需与向量数据库一致）

**怎么写**：

- 封装一个 `EmbeddingService`，统一调用接口。
- 可使用 `langchain.embeddings` 或直接调用模型 API。

python

```
from langchain_openai import OpenAIEmbeddings

embeddings = OpenAIEmbeddings(model="text-embedding-3-small")
vectors = embeddings.embed_documents([chunk.text for chunk in chunks])
```



**工程注意**：向量化通常是耗时操作，建议异步处理（如放入消息队列），并设超时重试。

------

## 五、向量存储与索引

**涉及方面**：

- 选型：Chroma / Qdrant / Milvus / PGVector / FAISS（本地或云端）
- 索引类型：HNSW、IVF 等（影响查询速度与内存）
- 元数据过滤（如按文档 ID、时间、部门过滤）
- 多租户隔离（不同租户使用不同 collection，或通过 metadata 字段区分）
- 增量更新：插入、删除、修改向量的 API

**怎么写**：

- 定义统一的 `VectorStore` 接口（add, delete, search, update）。
- 底层实现针对不同数据库。
- 使用环境变量切换。

python

```
# vector_store.py
class QdrantVectorStore:
    def __init__(self, collection_name, embedding_dim):
        self.client = qdrant_client.QdrantClient(host=..., port=...)
        self.collection = collection_name

    def add(self, vectors, payloads, ids):
        self.client.upsert(...)

    def search(self, query_vector, top_k=5, filter=None):
        return self.client.search(...)
```



**性能调优**：批量写入时设置 `batch_size`，避免单条提交；建索引参数（如 `ef_construct`）根据数据量调整。

------

## 六、检索与重排序

**涉及方面**：

- **混合检索**：向量相似度 + 关键词（BM25）
- **融合方法**：RRF（倒数排名融合）、加权求和
- **重排序（Rerank）**：使用交叉编码器（如 BGE-reranker、Cohere Rerank）对初筛结果精排
- **查询改写**：多轮对话中，将用户问题结合历史改写为独立查询（HyDE 或简单 rewrite）
- **检索后处理**：去重、剥离低分结果、补充元数据

**怎么写**：

- 实现一个 `Retriever` 类，内部调用向量库和 BM25（可借助 `Elasticsearch` 或 `rank_bm25` 内存版）。
- 重排序：`rerank` 函数调用本地或 API 模型。

python

```
def hybrid_search(query, top_k):
    vec_hits = vector_store.search(query_vec, top_k=20)
    bm25_hits = bm25_index.search(query, top_k=20)
    merged = reciprocal_rank_fusion([vec_hits, bm25_hits], k=60)
    if use_reranker:
        merged = rerank(query, [hit.text for hit in merged])
    return merged[:top_k]
```



**注意事项**：重排序会增加延迟，可设计为可配置开关；对于实时性要求高的场景，可只做混合检索不加 rerank。

------

## 七、生成与流式响应（LLM）

**涉及方面**：

- Prompt 工程：包含系统角色、检索到的上下文、用户问题、格式要求（如返回引用）
- 模型选择：GPT-4o-mini、DeepSeek、Claude、本地 Qwen 等
- 流式输出：SSE (Server-Sent Events) 或 WebSocket
- 中断机制：用户停止生成时，取消 LLM 请求
- 引用格式化：将检索结果中 `metadata` 的文档名、页码拼接到回答末尾

**怎么写**：

- 使用 FastAPI 的 `StreamingResponse` + 异步生成器。
- LLM 调用时传递 `stream=True` 参数。

python

```
@app.post("/chat")
async def chat(request: ChatRequest):
    context = retriever.get_context(request.query)
    messages = build_messages(system_prompt, context, request.history)
    async def generate():
        async for chunk in llm.astream(messages):
            yield f"data: {json.dumps({'delta': chunk})}\n\n"
        yield "data: [DONE]\n\n"
    return StreamingResponse(generate(), media_type="text/event-stream")
```



**中断实现**：使用 `asyncio.Task` 和 `cancel()`，并设置 `on_cancel` 处理 LLM 流的提前关闭。

------

## 八、会话与记忆管理

**涉及方面**：

- 会话存储（Redis / 数据库）
- 记忆窗口控制：保留最近 N 轮对话
- 上下文拼接：将历史问答组装成 prompt
- 多会话隔离（每个用户/会话独立）

**怎么写**：

- 设计 `Session` 表：session_id, user_id, created_at, updated_at
- `Message` 表：message_id, session_id, role, content, timestamp
- 查询时动态加载最后 K 条消息，构建消息列表给 LLM。

**优化**：当历史过长超出 token 限制时，采用**摘要式记忆**（每隔 N 轮触发一次摘要生成，把之前的对话压缩成一段文本）。

------

## 九、后端 API 设计

**涉及方面**：

- RESTful 接口：上传文档、检索（非对话）、对话 endpoint、文档删除、会话列表等
- 请求/响应 Schema（Pydantic 模型）
- 错误处理与重试
- 限流、鉴权（JWT）
- 版本控制（`/v1/chat` 等）

**怎么写**：

- 使用 FastAPI（性能高、自动文档）。
- 统一响应格式：`{"code":200, "data":..., "msg":"success"}`。

python

```
from pydantic import BaseModel

class ChatRequest(BaseModel):
    query: str
    session_id: str
    history_limit: int = 5
```



**工程注意**：对文件上传接口需限制大小，使用临时文件，防止恶意上传。

------

## 十、异步与任务队列

**涉及方面**：

- 文档处理（解析、分块、向量化）应异步执行，避免阻塞主请求
- 使用 Celery + Redis/RabbitMQ，或 FastAPI 的 `BackgroundTasks`（轻量）
- 任务状态查询（任务 ID → 进度或结果）

**怎么写**：

python

```
from celery import Celery

celery_app = Celery('tasks', broker='redis://localhost:6379/0')

@celery_app.task
def process_document(file_path, doc_id):
    chunks = parse_and_chunk(file_path)
    vectors = embed(chunks)
    store_vectors(vectors)
    return {"status": "done", "doc_id": doc_id}
```



前端上传后立即返回 `task_id`，之后轮询 `/task/{task_id}` 状态。

------

## 十一、可观测性与日志

**涉及方面**：

- 结构化日志（JSON 格式），包含请求 ID、用户 ID、模型名称、耗时
- 指标采集（Prometheus）：请求总数、延迟分位数、检索耗时、生成耗时
- 链路追踪（可选）：OpenTelemetry
- 检索结果的可视化调试（在开发环境展示检索出的 chunks 和分数）

**怎么写**：

- 使用 Python `logging` + `python-json-logger`。
- 使用 `fastapi-prometheus-metrics` 中间件。

python

```
import logging
logger = logging.getLogger("rag")
logger.info({"event": "search", "latency_ms": 120, "topk": 5})
```



**加分项**：提供一个**调试面板**，输入问题后显示检索到的文档片段及相似度，方便排查检索质量问题。

------

## 十二、测试与评估

**涉及方面**：

- 单元测试：parser、chunker、retriever 各组件
- 集成测试：端到端 API 测试
- 评估数据集：构建 QA 对（问题 + 期望的答案 + 期望的引用文档）
- 评估指标：Hit Rate，MRR，精确匹配，LLM 作为裁判（GPT-Eval）
- 回归测试：每次调整参数时跑评估集

**怎么写**：

- 使用 `pytest` + `httpx` 测试 API。
- 评估脚本：

python

```
def evaluate(retriever, eval_data):
    hits = 0
    for q, expected_doc_id in eval_data:
        retrieved = retriever.search(q, top_k=5)
        if expected_doc_id in [r.metadata["doc_id"] for r in retrieved]:
            hits += 1
    return hits / len(eval_data)
```



------

## 十三、部署与运维

**涉及方面**：

- 容器化（Dockerfile，docker-compose）
- 编排（K8s 或简单用 Docker Swarm）
- 环境变量管理（.env 文件、K8s ConfigMap）
- CI/CD（GitLab CI / GitHub Actions）：自动测试、构建镜像、部署
- 监控告警（Prometheus + Grafana + AlertManager）
- 模型服务分离（将 LLM 和 Embedding 作为远程 API 以降低耦合）

**怎么写**：

- 编写 `Dockerfile`，使用多阶段构建。

dockerfile

```
FROM python:3.11-slim
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt
COPY . .
CMD ["uvicorn", "main:app", "--host", "0.0.0.0", "--port", "8000"]
```



- 编写 `docker-compose.yml`，包含 PostgreSQL（或向量库）、Redis、应用服务。

------



向量库：

> “我们这个项目是给一个中小型客户做的内部知识库问答，客户提供了 2000 多份文档，主要是操作手册和常见问题。当时在选向量数据库时，我排除了 Milvus 集群，因为那个规模用集群完全是过度设计。但为什么没有选 pgvector 呢？因为客户那边并没有统一的 PostgreSQL 环境，他们是多种数据库混用，如果为了向量检索单独再上一个 PG，运维成本也不低。
>
> 所以我最后选了 **Qdrant 的单机版**。理由有三条：
>
> **第一，部署极简。** 一条 Docker 命令就能启动，不需要 Kubernetes 也不依赖外部组件。我给客户交付的是一个 `docker-compose.yml` 文件，加一个环境变量就能跑起来，客户自己的运维看了就能上手。
>
> **第二，性能够用且稳定。** 2000 多份文档分块后大概几十万条向量，Qdrant 单机版在百万级向量下查询延迟能稳定在 50 毫秒以内，完全满足客户日均几百次的查询需求。而且它原生支持 HNSW 索引，建好后检索非常快。
>
> **第三，功能完整且轻量。** Qdrant 单机版已经支持了 payload 过滤、多租户隔离、向量与关键词混合检索（通过 `shard` 内的 `full-text` 索引），这些能力足够支撑客户未来两年的扩展需求。同时它的内存效率比 Milvus 高，单机资源占用只有两三 GB，可以跟业务服务混部，不需要单独申请大规格机器。
>
> 总结下来，我的选型逻辑就是：**在满足性能和功能的前提下，选择运维成本最低、客户最容易接受的方案。** Qdrant 单机版正好符合这个标准。”

------

如果面试官追问：“那为什么不用 Chroma？更轻啊。”

你可以补一句：

> “Chroma 确实更轻，适合本地开发或 POC。但给客户交付的生产环境，我需要考虑数据持久化、备份、多租户隔离这些能力。Chroma 在这些方面还不够成熟，而 Qdrant 单机版已经是一个生产级的组件。所以我在 MVP 阶段会先用 Chroma 快速验证，但真正给客户部署时一定会切换到 Qdrant。”



