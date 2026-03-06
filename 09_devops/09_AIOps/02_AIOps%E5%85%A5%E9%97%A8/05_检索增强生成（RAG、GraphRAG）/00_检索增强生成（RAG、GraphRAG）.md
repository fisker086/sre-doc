# 检索增强生成（RAG、GraphRAG）实战详解

## 一、什么是检索增强生成（RAG）

**检索增强生成（Retrieval-Augmented Generation，简称 RAG）** 是一种将大语言模型（LLM）与外部知识库结合的技术。它允许模型在回答问题时，动态地从私有或特定领域的知识库中检索相关信息，并基于这些信息生成更准确的回答。

### 核心思想图解：

```
用户提问
    ↓
[问题向量化] → 向量数据库匹配 → [最相似的知识片段]
    ↓
组合成 System Prompt
    ↓
大语言模型生成答案
```

> **说明：** 传统的大模型只能依赖训练数据中的通识知识，而无法访问企业内部的私有文档。RAG 技术通过“先检索 + 再生成”的方式，让模型能够“看到”并使用最新的、专属的知识内容。

## 二、为什么需要 RAG？——解决上下文长度限制

我们知道，大语言模型如 GPT 系列都有一个 **Token 上下文长度限制**（例如 GPT-4 最多支持 32,768 个 Token）。这意味着我们不能把整个公司庞大的知识库一次性塞进提示词（Prompt）里。

### 1、错误做法：直接放入全部知识

```text
System Prompt:
你是一个运维助手。
以下是我们的全部运维手册：
[长达数万字的内容……]
请根据以上内容回答问题。
```

→ 结果：超出 Token 限制，无法运行！

### 2、正确做法：按需检索相关片段

```text
System Prompt:
你是一个运维助手。
以下是与当前问题相关的知识片段：
[仅包含匹配到的几段关键内容]
如果不知道答案，请说“我不知道”。
```

→ 结果：精准、高效、不超限！

## 三、RAG 的两大核心阶段

### 1、阶段一：索引构建（Indexing）

这是“预处理”阶段，目的是为后续检索做准备。

#### 1.步骤1：加载文档（Document Loading）

将各种格式的文件（PDF、Word、Markdown、Excel等）统一转换为纯文本。

| 文件类型 | 加载工具示例                 |
| -------- | ---------------------------- |
| `.pdf`   | PyPDFLoader                  |
| `.docx`  | Docx2txtLoader               |
| `.md`    | TextLoader                   |
| `.pptx`  | UnstructuredPowerPointLoader |

> 使用 `LangChain` 提供的文档加载器可轻松实现多格式支持。

#### 2.步骤2：文本分片（Text Splitting）

由于单个文档可能很长，必须将其拆分为小块（chunks），以便后续向量化和存储。

##### 分片策略对比：

| 策略                          | 优点               | 缺点               |
| ----------------------------- | ------------------ | ------------------ |
| 固定长度切分                  | 实现简单           | 可能切断语义完整性 |
| 按标题分割（Markdown Header） | 保持逻辑结构完整   | 仅适用于结构化文档 |
| 语义感知分块                  | 更智能，保留上下文 | 计算成本高         |

> 推荐：对于 Markdown 文档，使用 `MarkdownHeaderTextSplitter` 按 `#`, `##`, `###` 标题进行分片。

#### 3.步骤3：向量化与存储（Embedding & Storage）

将每个文本块转化为高维向量，并存入**向量数据库**。

##### 向量化流程图解：

```
原始文本块
    ↓
调用 Embedding 模型（如 text-embedding-3-small）
    ↓
输出一个 1536 维的向量（浮点数组）
    ↓
存入向量数据库（如 ChromaDB、Pinecone）
```

> 示例：OpenAI 的 `text-embedding-3-small` 输出维度为 1536；`large` 版本则更高。

### 2、阶段二：检索与生成（Retrieval & Generation）

这是“实时响应”阶段，发生在用户提问之后。

#### 1.步骤1：问题向量化

用户的输入问题也被送入相同的 Embedding 模型，得到其对应的向量表示。

#### 2.步骤2：向量相似度匹配

使用算法计算问题向量与所有知识片段向量之间的“距离”，找出最相近的 Top-K 个片段。

##### 常见相似度算法：

| 算法                               | 公式                                      | 特点                                   |
| ---------------------------------- | ----------------------------------------- | -------------------------------------- |
| 余弦相似度（Cosine Similarity）    | $ \cos(\theta) = \frac{A·B}{\|A\|\|B\|} $ | 忽略长度影响，关注方向一致性，推荐使用 |
| 欧几里得距离（Euclidean Distance） | $ d = \sqrt{\sum (A_i - B_i)^2} $         | 关注绝对差异，适合数值敏感场景         |

> 实践建议：一般使用 **余弦相似度**，值越接近 1 表示越相似。

#### 3.步骤3：拼接 Prompt 并生成答案

将检索到的相关文本片段插入 System Prompt 中，交由大模型生成最终回答。

```python
system_prompt = """
你是一个问答助手。请根据以下上下文回答问题。
如果你不知道答案，请直接说“我不知道”。

上下文：
{retrieved_context}

问题：{user_question}
"""
```

> 注意事项：
>
> - **Embedding 模型必须一致**：索引阶段和检索阶段使用的 Embedding 模型必须相同，否则维度不同会导致匹配失败。
> - **设置拒答机制**：避免模型“幻觉”，明确告知“不知道”也是一种有效回答。

## 四、代码实战：基于 LangChain 构建 RAG 系统

### 1、环境准备

```bash
pip install langchain langchain-openai chromadb python-dotenv
```

### 2、初始化配置

```python
import os
from dotenv import load_dotenv
load_dotenv()

os.environ["OPENAI_API_KEY"] = "your-api-key"
os.environ["OPENAI_API_BASE"] = "https://api.openai.com/v1"  # 或代理地址
```

### 3、加载与分片文档

```python
from langchain.document_loaders import TextLoader
from langchain.text_splitter import MarkdownHeaderTextSplitter

# 加载 Markdown 文件
loader = TextLoader("data/ops_knowledge.md", encoding="utf-8")
docs = loader.load()

# 按标题分片
headers_to_split_on = [
    ("#", "Header 1"),
    ("##", "Header 2"),
    ("###", "Header 3"),
]
splitter = MarkdownHeaderTextSplitter(headers_to_split_on=headers_to_split_on)
splits = splitter.split_text(docs[0].page_content)

print(f"共分出 {len(splits)} 个片段")
```

### 4、向量化并存入数据库

```python
from langchain_openai import OpenAIEmbeddings
from langchain_community.vectorstores import Chroma

embeddings = OpenAIEmbeddings(model="text-embedding-3-small")

# 创建向量数据库
vectorstore = Chroma.from_documents(
    documents=splits,
    embedding=embeddings,
    persist_directory="./lanchain_db"
)

vectorstore.persist()  # 持久化保存
```

### 5、执行检索与问答

```python
from langchain.chains import RetrievalQA
from langchain_openai import ChatOpenAI

# 创建检索器
retriever = vectorstore.as_retriever(search_kwargs={"k": 3})

# 定义 LLM
llm = ChatOpenAI(model="gpt-4o-mini")

# 构建 QA 链
qa_chain = RetrievalQA.from_chain_type(
    llm=llm,
    chain_type="stuff",
    retriever=retriever,
    return_source_documents=True
)

# 提问
question = "payment服务是谁维护的？"
result = qa_chain.invoke({"query": question})

print("答案:", result["result"])
print("来源文档:", result["source_documents"])
```

## 五、RAG 的局限性分析

尽管 RAG 强大，但它存在两个主要缺陷：

### 1、缺陷一：分片破坏语义连贯性

当文档被强制切割时，原本具有逻辑关联的内容可能被分散到不同片段中，导致模型无法理解整体含义。

> 示例：  
> 原文：“小张负责 payment-service 和 auth-service。”  
> 若被分成两块，则单独任一块都无法回答“谁维护的服务最多”。

### 2、缺陷二：无法处理总结性/推理性问题

RAG 本质是“关键词近似匹配”，不具备深层语义理解和归纳能力。

>  无法回答的问题：
>
> - “谁维护的服务最多？”
> - “哪些服务存在安全风险？”
> - “系统宕机的常见原因有哪些？”

## 六、GraphRAG：下一代解决方案

为克服上述问题，微软提出 **GraphRAG** —— 利用知识图谱技术提取实体及其关系，实现全局理解。

### 1、GraphRAG 工作原理图解：

```
原始文档
    ↓
大模型提取实体与关系（如：人-维护->服务）
    ↓
构建知识图谱（Knowledge Graph）
    ↓
存储至图数据库（Neo4j）
    ↓
支持复杂查询（如：统计某人维护的服务数量）
```

### 2、实战步骤（使用 graphrag 工具）：

```bash
# 安装
pip install graphrag

# 初始化项目
python -m graphrag.index init --root ./data

# 修改 .env 和 settings.yaml 中的 API 密钥和模型配置

# 创建 input 目录并放入文本
mkdir data/input && cp data/kb.txt data/input/

# 构建图谱索引
python -m graphrag.index --root ./data
```

### 3、执行全局查询：

```bash
python -m graphrag.query \
  --root ./data \
  --method global \
  "谁维护的服务最多？"
```

> 输出结果：  
> “小张维护的服务最多，因为他同时负责支付系统的前后端基础服务。”

## 七、可视化知识图谱：使用 Neo4j

GraphRAG 生成的知识图谱可通过 Neo4j 可视化查看。

### 1、启动 Neo4j 容器：

```bash
docker run -d \
  --name neo4j \
  -p 7474:7474 -p 7687:7687 \
  -e NEO4J_AUTH=neo4j/my_password \
  neo4j:5
```

### 2、导入数据并查看关系：

```python
from graphrag.neo4j_loader import Neo4jLoader
loader = Neo4jLoader(uri="bolt://localhost:7687", username="neo4j", password="my_password")
loader.load_from_output("./data/output")
```

访问 `http://localhost:7474` 登录后即可查看节点与关系图谱。

## 八、无需编码的 RAG 工具推荐

### 1、 **RAGFlow**

- 支持几乎所有文档类型（含 OCR）
- 提供 Web UI 界面
- 支持混合检索（关键词 + 向量）
- 开源免费

### 2、 **QAnything（网易出品）**

- 支持本地部署
- 检索精度高（ReRank 设计）
- 支持中文优化
- 文档类型支持略少于 RAGFlow

> 用户可根据需求选择是否开发自定义系统，或直接使用成熟工具。

## 总结

| 技术         | 优势                     | 适用场景                         |
| ------------ | ------------------------ | -------------------------------- |
| **RAG**      | 实现简单、响应快         | 明确事实类问答（如“XX由谁负责”） |
| **GraphRAG** | 支持推理、总结、全局分析 | 复杂决策、数据分析、知识发现     |

>  **最佳实践建议**：
>
> - 日常运维问答可用 RAG；
> - 需要智能分析的企业级应用应采用 GraphRAG；
> - 非技术人员可选用 RAGFlow 或 QAnything 快速搭建知识库系统。

通过本篇深度解析，您已掌握从理论到实践的完整 RAG 技术体系，可用于构建真正智能化的企业级 AI 助手。