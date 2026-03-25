# 04 · Context 构建与注入

> 模块 D — Must Know（你的已有基础直接切入点）

---

## 本模块解决什么工程问题

检索到的 Top-5 文档块是"原材料"。把它们送给 LLM 之前，还需要做一件事：**把这些原材料组装成 LLM 能有效利用的 Prompt**。

这一步在整个 RAG 流水线里常被低估——很多人以为"把检索结果拼进去就行"。但实际上：同样的检索结果，不同的组装方式，LLM 的回答质量可以差很多。

本模块是你的已有 Context Engineering 能力在 RAG 里的直接延伸，理解成本最低，但工程收益很高。

---

## 知识点 1：RAG Prompt 的基本结构

一个完整的 RAG Prompt 由四个部分组成：

```
[1] System Prompt（角色设定 + 行为约束）
[2] 检索到的文档块（context，动态注入部分）
[3] 对话历史（如果是多轮对话）
[4] 用户当前问题
```

这个结构和你已经在做的 Prompt 设计完全一致，唯一的区别是 **[2] 是由检索系统自动填充的**，不是手写的。

**在贯穿案例里的完整 Prompt 示例：**

```
[System Prompt]
你是一名技术文档助手，专门回答关于公司产品的技术问题。
请严格基于下方提供的文档内容进行回答。
如果文档中没有相关信息，请明确说明"文档中未找到相关内容"，不要凭空推测。
回答时请注明信息来源（文件名和章节）。

[检索到的文档块]
--- 来源：API_文档_v2.3.pdf，第3章 ---
Bearer Token 认证方式：所有 API 请求需在请求头中携带
Authorization: Bearer <access_token>
access_token 通过 POST /auth/token 接口获取，有效期 3600 秒。

--- 来源：FAQ.md，第2节 ---
Q：token 过期后怎么处理？
A：使用 refresh_token 调用 POST /auth/refresh 接口获取新 token。

--- 来源：故障排查记录_2024-08.md ---
问题：API 持续返回 401 Unauthorized
原因：access_token 未正确设置或已过期
解决：检查请求头格式，确认 token 有效期

[用户问题]
我调用 API 时一直报 401，怎么排查？
```

**缺少 System Prompt 约束会怎样：** LLM 可能无视检索内容，用训练数据里的通用知识回答，或者在文档没有明确答案时开始"发挥"，产生幻觉。

→ **术语注解：幻觉（Hallucination）**，LLM 生成的内容在检索文档中找不到依据，是凭空编造的。RAG 能大幅降低幻觉，但不能彻底消除——System Prompt 的"忠于文档"约束是重要防线。

---

## 知识点 2：文档块的排列策略

Top-5 块按什么顺序放进 Prompt，对 LLM 的回答质量有影响吗？有，而且影响不小。

### 迷失在中间（Lost in the Middle）现象

研究发现，LLM 对 context 里**开头和结尾**的内容记忆和利用率最高，**中间部分**容易被"忽视"。

→ **术语注解：Lost in the Middle**，2023 年 Stanford 的研究发现，当 context 很长时，LLM 对中间位置的内容利用率明显低于首尾。这对 RAG 的排列策略有直接工程影响。

**在贯穿案例里的排列建议：**

```
最相关的块 → 放最后（紧挨着用户问题）
次相关的块 → 放最前（System Prompt 之后）
相关性一般的块 → 放中间

原因：LLM 最后看到的内容在生成时影响最大（近因效应）；
最前面的内容是"背景设定"，也有较高权重。
```

### 按相关性降序 vs 升序

| 排列方式 | 效果 |
|---------|------|
| 相关性降序（最相关在前） | 直观但 Lost-in-Middle 问题严重 |
| 相关性升序（最相关在后） | 利用近因效应，实测通常更好 |
| 最相关在首尾，次相关在中间 | 理论上最优，实现稍复杂 |

**工程建议：** 优先试升序排列（最相关块放在用户问题上方），简单有效。

---

## 知识点 3：Context Window 管理

### 问题的来源

检索到的块总 token 数 + System Prompt + 对话历史 + 用户问题 + 预留的回答空间 = 必须 ≤ 模型的 context window 上限。

```
Claude Sonnet：200K tokens（基本不用担心）
GPT-4o：128K tokens
GPT-3.5-turbo：16K tokens（老模型，需要严格管理）

实际可用于文档块的空间：
context_window - system_prompt_tokens - history_tokens - question_tokens - output_reserve
```

**在贯穿案例里：** 如果用 16K 的模型，System Prompt 占 200 tokens，对话历史占 2000 tokens，问题占 100 tokens，预留回答 1000 tokens，那么文档块的可用空间只剩约 12700 tokens——大约能放 25 个 512-token 的块。

### 超限时的处理策略

**策略一：按相关性截断**
超出预算的块直接丢弃，从相关性最低的开始删。简单粗暴，但会损失信息。

**策略二：块内容压缩**
用一个更小的 LLM 或摘要模型，先把每个块压缩成更短的摘要，再拼入 Prompt。

```python
# 伪代码
compressed_chunks = []
for chunk in retrieved_chunks:
    if total_tokens + chunk.tokens > budget:
        summary = summarize(chunk.text, max_tokens=100)  # 压缩到100 tokens
        compressed_chunks.append(summary)
    else:
        compressed_chunks.append(chunk.text)
```

**策略三：动态调整 Top-K**
根据当前对话历史长度动态计算可用空间，反过来决定召回多少块。

```python
available_tokens = context_window - system_tokens - history_tokens - question_tokens - output_reserve
max_chunks = available_tokens // avg_chunk_tokens
results = retrieve(query, top_k=max_chunks)
```

→ **术语注解：Token 预算（Token Budget）**，为 Prompt 各部分分配 token 数量的规划。这是 Context Engineering 里的核心概念，你已经熟悉——在 RAG 场景里，文档块是最大的动态变量。

---

## 知识点 4：引用与溯源

### 它在做什么

让 LLM 在回答时标注每句话来自哪份文档，员工可以去原文验证。

**在贯穿案例里为什么关键：** 技术文档问答的受众是工程师，他们不会盲信机器人。有了来源标注，他们可以去原始文档确认细节；更重要的是，当 LLM 给出了一个错误回答，有来源才能快速定位是检索到了错误内容（检索问题）还是 LLM 自己编的（生成问题）。

### 实现方式一：Prompt 约束引用

最简单的方式——在 System Prompt 里要求 LLM 引用来源：

```
System Prompt 加入：
"回答中每个关键信息点后，用[来源：文件名，章节]格式标注出处。
只引用下方文档中明确包含的信息。"
```

LLM 会自然地在回答里插入引用标注：
```
调用 API 时需要在请求头中携带 Bearer Token [来源：API文档v2.3，第3章]。
Token 过期后可以使用 refresh_token 刷新 [来源：FAQ.md，第2节]。
```

**局限：** LLM 可能引用错误的来源（比如引用了块2但内容来自块1），需要后处理验证。

### 实现方式二：结构化输出引用

要求 LLM 输出结构化 JSON，把引用和内容分开：

```python
# Prompt 里指定输出格式
"""
请用以下 JSON 格式回答：
{
  "answer": "回答正文",
  "citations": [
    {"claim": "回答中的某个具体主张", "source": "来源文件名", "chunk_id": "块ID"}
  ]
}
"""
```

```json
// LLM 输出
{
  "answer": "API 调用需要 Bearer Token 认证，Token 过期后需要使用 refresh_token 刷新",
  "citations": [
    {"claim": "Bearer Token 认证", "source": "API_文档_v2.3.pdf", "chunk_id": "42"},
    {"claim": "refresh_token 刷新", "source": "FAQ.md", "chunk_id": "18"}
  ]
}
```

**优点：** 引用结构清晰，可以程序化处理和展示；方便后续验证 faithfulness（回答里每个主张是否真的来自引用的块）。

→ **术语注解：Faithfulness（忠实度）**，RAG 评估指标之一。检验 LLM 的回答是否忠于检索到的文档内容，回答里每个陈述都能在检索块里找到依据。

### 实现方式三：后处理高亮

前端展示时，把 LLM 回答里的句子和检索块做匹配，高亮显示可溯源的部分，未匹配的部分标记为"待验证"。适合对幻觉容忍度低的场景。

---

## 知识点 5：多轮对话的 Context 管理

### 问题来源

员工不只问一个问题，往往是追问：
```
问：API 鉴权怎么做？
答：用 Bearer Token...
追问：Token 失效了怎么办？
追追问：refresh_token 也失效了呢？
```

多轮对话要把历史对话也放进 context，但历史越来越长，会挤占文档块的空间。

### 两个核心问题

**问题一：检索用什么问题？**

直接用"refresh_token 也失效了呢"去检索，效果差——这句话缺乏上下文，向量化后语义不完整。

解决方案：**查询改写（模块 F 的内容，这里先知道有这个问题）**——用历史对话把当前问题补全为独立可检索的句子：
"OAuth access_token 和 refresh_token 都过期失效后如何重新获取认证"

**问题二：历史对话放多少？**

```python
# 伪代码：动态截断历史
def build_context(history, retrieved_chunks, question, budget):
    chunks_tokens = sum(c.tokens for c in retrieved_chunks)
    question_tokens = count_tokens(question)
    remaining = budget - system_tokens - chunks_tokens - question_tokens - output_reserve
    
    # 从最近的历史开始保留，超出 remaining 就截断
    trimmed_history = trim_history_to_budget(history, remaining)
    
    return assemble_prompt(system, retrieved_chunks, trimmed_history, question)
```

**在贯穿案例里的建议：** 保留最近 3-5 轮对话，更早的历史价值有限，且占用宝贵的文档块空间。技术文档问答每轮相对独立，历史不需要保留太多。

---

## 知识点 6：Prompt 模板工程

### 标准 RAG Prompt 模板

```python
RAG_PROMPT_TEMPLATE = """
你是{company_name}的技术文档助手。

## 行为准则
- 只基于下方【参考文档】中的内容回答
- 如果参考文档中没有相关信息，直接说"文档中未找到该问题的答案"
- 不要推测或补充文档中没有的内容
- 每个关键信息点后标注来源

## 参考文档
{context_blocks}

## 问题
{question}

## 回答
"""

# 动态填充
def build_prompt(chunks, question):
    context = "\n\n".join([
        f"--- 来源：{c.metadata['source_file']}，{c.metadata.get('chapter','')} ---\n{c.text}"
        for c in chunks
    ])
    return RAG_PROMPT_TEMPLATE.format(
        company_name="ACME Corp",
        context_blocks=context,
        question=question
    )
```

### 模板的关键设计点

**"文档中没有就说没有"** 这句约束非常重要。没有这个约束，LLM 在文档覆盖不到的问题上会用通用知识填充，员工误以为是公司文档说的。

**来源标注格式** 要和前端展示协商好——是纯文字标注、还是 JSON 结构、还是链接跳转，需要根据产品需求决定。

**多语言支持**：贯穿案例里文档可能是中文，也可能有英文 API 文档，Prompt 模板本身建议用中文，LLM 通常能处理混合语言检索结果。

---

## 模块小结

Context 构建是 RAG 流水线里连接"检索"和"生成"的桥梁，也是你已有能力最直接复用的地方。

几个最高价值的实践总结：
1. System Prompt 里明确"忠于文档，不知道就说不知道"
2. 文档块按相关性升序排列（最相关紧贴用户问题）
3. 动态管理 token 预算，文档块优先，历史可以截断
4. 引用溯源不是可选项，是技术文档场景的基本要求

**和下一个模块（E：评估与调优）的衔接：**

现在系统已经可以端到端运行了：输入问题，输出带来源的回答。

但怎么知道系统表现好不好？靠人工感受是不够的，你改了分块策略，不知道是变好了还是变坏了；换了 Reranker 模型，不知道提升了多少。

模块 E 解决的是：**如何系统性地测量 RAG 系统的质量，以及如何定位问题出在哪个环节**。
