# 02 · 文档处理与分块

> 模块 B — Must Know

---

## 本模块解决什么工程问题

贯穿案例里有数百份文档：最短的是一页 FAQ，最长的是 200 页的产品手册。这些文档不能整篇塞进向量库——一篇 200 页的手册向量化后只有一个向量，检索时这一个向量要代表手册里所有内容，会严重失真。

也不能逐句拆开——"请参见下一节"单独成一个块，完全没有意义。

**文档处理与分块，就是解决"原始文档 → 适合检索的最小单元"这个转换问题**。它是离线流水线的核心工作，决定了向量库里存的"原料质量"，直接影响后续所有检索的天花板。

---

## 知识点 1：文档加载（Document Loading）

### 它在做什么

把各种格式的文件转换成统一的纯文本（或结构化文本），才能进行后续处理。

**在贯穿案例里：** 文档库里可能有 PDF 版的产品手册、Word 版的故障排查记录、HTML 版的在线 API 文档、Markdown 版的内部 Wiki。每种格式的解析方式不同，必须统一转换。

**缺少它会怎样：** 系统只能处理纯文本，PDF 和 Word 文档直接被排除在外，大量知识无法入库。

### 各格式的坑

| 格式 | 常见问题 | 处理建议 |
|------|---------|---------|
| PDF | 扫描件没有文字层；多栏排版被解析成乱序文本 | 扫描件需要 OCR；复杂排版需要专用解析器 |
| Word(.docx) | 表格和图片容易丢失 | 用 python-docx 解析，表格转 Markdown 格式 |
| HTML | 导航栏、页脚、广告等噪声混入正文 | 需要提取正文区域，过滤 boilerplate |
| Markdown | 相对干净，基本无损 | 直接处理，保留标题层级 |

→ **术语注解：OCR（Optical Character Recognition，光学字符识别）**，把扫描图片里的文字识别成可编辑文本的技术。扫描版 PDF 没有文字层，必须经过 OCR 才能提取内容。

→ **术语注解：Boilerplate**，页面上重复出现的固定内容，比如导航菜单、版权声明、页眉页脚。这些内容对问答没有价值，混进文档块会污染检索结果。

### 关键实现思路

```python
# 伪代码：统一加载入口
def load_document(file_path):
    ext = get_extension(file_path)
    
    if ext == ".pdf":
        return load_pdf(file_path)       # 用 PyMuPDF 或 pdfplumber
    elif ext == ".docx":
        return load_docx(file_path)      # 用 python-docx
    elif ext == ".html":
        return load_html(file_path)      # 用 BeautifulSoup + 正文提取
    elif ext in [".md", ".txt"]:
        return load_text(file_path)      # 直接读取
    
    # 输出：统一的 Document 对象，包含 text + metadata
    return Document(text=..., metadata={"source": file_path, "format": ext})
```

**工程建议：** LangChain 和 LlamaIndex 都内置了各种格式的 Document Loader，原型阶段直接用，不需要自己写解析器。

---

## 知识点 2：文档分块（Chunking）

### 它在做什么

把长文档切分成若干小块（chunks），每个块独立向量化存入向量库，成为检索的最小单元。

**在贯穿案例里：** 一份 200 页的产品手册切成几百个块，每个块包含一个相对完整的知识单元（比如"一个 API 端点的完整说明"或"一个故障现象的排查步骤"）。

**缺少合理分块策略会怎样：** 两种极端情况：
- 块太大（整章作为一块）：向量代表不了块内所有内容，检索相关性下降；塞进 context 后占用大量 token
- 块太小（逐句切）：单句往往没有完整语义，LLM 收到"步骤 3：点击确认按钮"但不知道这是哪个流程的步骤 3

### 四种分块策略

#### 策略一：固定长度分块（Fixed-size Chunking）

按字符数或 token 数硬切，加上一定的重叠。

```
文档内容：AAAAAAAA BBBBBBBB CCCCCCCC DDDDDDDD
chunk_size=100, overlap=20

块1：AAAAAAAA BBB（100字符）
块2：     BBBBBB CCCC（从上一块结尾往前20字符开始）
块3：         CCCCCC DDD（同理）
```

→ **术语注解：Overlap（重叠）**，相邻两个块之间共享的内容。设置重叠是为了防止一个完整的知识单元被切断后两半分别在不同块里，LLM 看任何一块都只看到一半。

**优点：** 实现简单，适合快速验证系统流程。
**缺点：** 完全不考虑语义边界，可能把一个句子或一个步骤从中间切断。

**在贯穿案例里：** 一条故障排查记录"原因：DNS 解析超时。解决方案：检查……"可能被切成"……原因：DNS 解析超"和"时。解决方案：检查……"两块，两块单独都不完整。

#### 策略二：语义分块（Semantic Chunking）

按自然语言边界切：标点句子、段落、章节标题处。

```python
# 伪代码
def semantic_chunk(text):
    # 按段落切（空行分隔）
    paragraphs = text.split("\n\n")
    
    chunks = []
    current_chunk = ""
    
    for para in paragraphs:
        if len(current_chunk) + len(para) < MAX_CHUNK_SIZE:
            current_chunk += para
        else:
            chunks.append(current_chunk)
            current_chunk = para
    
    return chunks
```

**优点：** 块内语义更完整。
**缺点：** 段落长短不一，有些段落本身就很长，块大小不均匀。

#### 策略三：递归分块（Recursive Chunking）

先按大分隔符切（章节），切出来的块如果还太大，再按小分隔符切（段落），还太大继续按句子切。LangChain 的 `RecursiveCharacterTextSplitter` 就是这种策略。

```
分隔符优先级：["\n\n\n", "\n\n", "\n", "。", " "]
先用最大的分隔符切，切出来的块如果超过 max_size，
再用下一级分隔符切这个块，直到满足大小限制。
```

**优点：** 兼顾语义完整性和大小可控性，是目前最常用的通用策略。
**在贯穿案例里：** 产品手册先按章节切，每章再按段落切，段落还太长则按句子切——保留了最自然的语义边界。

#### 策略四：结构感知分块（Structure-aware Chunking）

利用文档的结构信息切分：Markdown 按标题层级、代码按函数边界、HTML 按标签层级。

**在贯穿案例里尤其重要：** API 文档往往有固定结构——"端点名称 → 请求参数 → 响应格式 → 示例"。按这个结构切，每个块就是一个完整的 API 说明，非常适合检索。

```markdown
# 原始 API 文档结构

## POST /auth/token
**描述**：获取访问令牌
**请求参数**：
  - client_id: string
  - client_secret: string
**响应**：
  - access_token: string
  - expires_in: int

## GET /users/{id}
...
```

按 `## ` 二级标题切分，每个 API 端点成为一个块，完整且自包含。

→ **术语注解：自包含（Self-contained）**，一个块不依赖其他块就能被理解。这是好的分块的核心标准：LLM 单独看这一块，能获得完整的知识，不需要上下文补充。

---

## 知识点 3：Chunk 大小的工程决策

### 怎么选 Chunk Size

没有通用最优值。影响决策的因素：

**文档类型：**
- 叙述性文档（故障排查记录、说明文字）：500-800 tokens，保留足够上下文
- 结构化文档（API 文档、参数说明）：按结构边界切，不强制固定大小
- FAQ 类文档：一问一答作为一个块，往往 100-300 tokens

**下游使用方式：**
- 如果要直接把块塞进 Prompt：考虑 LLM 的 context window 上限，5 个块不能超过可用空间
- 如果要先经过 Reranker：可以先多召回再精排，块可以稍大

**在贯穿案例里的建议起点：**
- 产品手册正文：chunk_size=512 tokens，overlap=64 tokens
- API 文档：按端点结构切，通常 200-400 tokens
- 故障排查记录：chunk_size=256 tokens，overlap=32 tokens（问题和解决方案通常比较短）

### 为什么 Overlap 很重要

```
原文：...步骤2：进入设置页面。步骤3：点击"高级"选项卡。
      步骤4：在"超时"字段填入60。步骤5：保存配置...

无 Overlap 切法：
  块1：...步骤2：进入设置页面。步骤3：点击"高级"选项卡。
  块2：步骤4：在"超时"字段填入60。步骤5：保存配置...

员工问："怎么设置超时时间？" → 检索到块2
LLM看到步骤4、5，但不知道步骤1、2、3 → 回答不完整

有 Overlap 切法（overlap包含步骤3）：
  块2：步骤3：点击"高级"选项卡。步骤4：在"超时"字段填入60。步骤5：保存配置...
LLM 至少看到了连贯的操作步骤 → 回答更完整
```

---

## 知识点 4：元数据管理（Metadata）

### 它在做什么

给每个 chunk 附加描述性的标签信息，和向量一起存入向量库。检索时可以用元数据做过滤，检索结果可以附上来源说明。

**在贯穿案例里：** 每个 chunk 除了文本内容和向量，还存：

```json
{
  "text": "Bearer Token 认证方式：在请求头中携带 Authorization: Bearer <token>...",
  "metadata": {
    "source_file": "API_文档_v2.3.pdf",
    "chapter": "第3章 认证与授权",
    "page": 42,
    "doc_type": "api_doc",
    "last_updated": "2024-11-01",
    "product": "TEMS"
  }
}
```

**缺少元数据会怎样：**
- 无法引用来源：回答"根据文档……"但不知道是哪份文档
- 无法过滤检索：不能实现"只查 API 文档"或"只查最近半年更新的内容"
- 文档更新时无法精准删除：不知道向量库里哪些块来自某份文档，只能全量重建

### 元数据过滤的工程价值

```python
# 伪代码：带元数据过滤的检索
results = vector_db.search(
    query_vector=embed("OAuth token 怎么用"),
    top_k=5,
    filter={
        "doc_type": "api_doc",           # 只查 API 文档类型
        "last_updated": {"gte": "2024-01-01"}  # 只查今年更新的
    }
)
```

这个过滤是在向量相似度搜索的基础上加的约束，不影响性能，但大幅提升结果精准度。

→ **术语注解：Pre-filtering vs Post-filtering**，Pre-filtering 是先用元数据筛掉不符合条件的向量再做相似度搜索（效率高），Post-filtering 是搜完再筛（可能导致返回数量不足）。Qdrant、Weaviate 等主流向量库支持高效的 Pre-filtering。

---

## 知识点 5：父子块策略（Parent-Child Chunking）

### 它在做什么

这是一个进阶但非常实用的模式：**用小块检索，用大块生成**。

把文档切两层：
- **子块（Child Chunk）**：小块，100-200 tokens，用于向量化和检索。小块向量更精准，检索相关性更高。
- **父块（Parent Chunk）**：大块，500-1000 tokens，存储更完整的上下文，用于最终送给 LLM。

检索时找到子块，但送给 LLM 的是子块所属的父块。

**在贯穿案例里：**

```
父块：[API 认证章节完整内容，800 tokens]
  ├── 子块1：[认证方式概述，150 tokens]  ← 向量化，用于检索
  ├── 子块2：[Bearer Token 说明，180 tokens]  ← 向量化，用于检索
  └── 子块3：[OAuth 2.0 流程，200 tokens]  ← 向量化，用于检索

员工问："access token 怎么获取？"
→ 子块3 被检索到（相似度最高）
→ 但送给 LLM 的是父块（完整的认证章节）
→ LLM 有完整上下文，回答更准确
```

**解决了什么问题：** 小块检索精准，大块生成完整——两全其美。

→ **术语注解：LlamaIndex 的 ParentDocumentRetriever**，LlamaIndex 框架内置了这种模式的实现，LangChain 里叫 `ParentDocumentRetriever`。

---

## 模块小结

分块策略是 RAG 系统里"投入产出比"最高的优化点之一：不需要更好的模型，不需要更复杂的架构，只是换一种切法，检索质量可以有明显提升。

**核实一个好的分块的标准：**
1. 自包含——单独看这一块能理解
2. 大小合适——不超出 context 预算，又不过于碎片化
3. 有元数据——知道来自哪里，可以被过滤和引用

**和下一个模块（C：检索策略）的衔接：**

块已经存进向量库了。现在的问题是：员工提问时，怎么从库里找到最相关的块？

向量相似度检索是基础，但有两个限制：
1. 它只能做语义匹配，精确术语（比如函数名 `getAccessToken()`）可能找不准
2. 相似度高不等于真正相关，返回的 Top-K 里可能有"看起来像但其实没用"的块

模块 C 就是解决这两个问题——用混合检索和重排序，让召回结果更准。
