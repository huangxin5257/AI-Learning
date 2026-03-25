# 06 · 高级检索模式

> 模块 F — Nice to Know

---

## 本模块解决什么工程问题

有了模块 A-E 的基础，RAG 系统已经能跑起来了。但你会在评估时发现一类顽固的问题：

- 员工问"那个什么认证怎么用"——太模糊，直接检索什么都找不到
- 员工问"我们产品支持 SAML 吗"——文档里没有直接写"支持 SAML"，但确实提到了相关配置
- 员工问"升级到 v3 之后，原来的 v2 接口还能用吗"——需要同时找到 v2 文档和 v3 迁移指南，对比后才能回答

这些都是"提问到检索之间的鸿沟"问题。本模块的三个技术，各自针对一类鸿沟。

---

## 知识点 1：查询改写（Query Rewriting）

### 它在做什么

在把用户问题送去检索之前，先用 LLM 把问题改写成更适合检索的形式。

**在贯穿案例里：**

```
员工原始问题：
"那个接口为什么老是报 500，不知道怎么回事"

直接检索效果：
→ 向量匹配"接口报 500"，可能找到一些相关结果，但"不知道怎么回事"这类口语噪声会干扰向量

改写后：
"API 接口返回 HTTP 500 Internal Server Error 原因及解决方法"

改写后检索效果：
→ 去掉口语噪声，加入标准术语，语义精准指向故障排查文档
```

### 三种改写策略

#### 策略一：问题补全（Query Expansion）

把模糊或简短的问题扩展成完整、清晰的检索查询。

```python
# Prompt 给改写 LLM
"""
你是一个搜索查询优化专家。
将以下用户问题改写成适合技术文档检索的查询语句：
- 去除口语化表达
- 补全缺失的技术术语
- 保持原始意图
- 只输出改写后的查询，不要解释

原始问题：{user_question}
"""
```

#### 策略二：多查询生成（Multi-Query）

一个问题生成多个不同角度的查询，分别检索，合并结果后去重。

```python
# 伪代码
queries = llm.generate(f"""
为以下问题生成3个不同角度的搜索查询，覆盖不同的表达方式：
问题：{user_question}
输出3行，每行一个查询。
""")

all_results = []
for query in queries:
    results = retrieve(query, top_k=5)
    all_results.extend(results)

final_results = deduplicate_and_rerank(all_results)
```

**在贯穿案例里的效果：**
```
原问题：OAuth 怎么配置

生成的三个查询：
1. "OAuth 2.0 配置步骤"
2. "授权认证设置方法"
3. "client_id client_secret 获取配置"

三条查询分别检索，覆盖了文档里不同的表达方式
```

#### 策略三：对话历史感知改写（Contextual Compression）

多轮对话场景：把当前追问结合历史，改写成独立可检索的完整问题。

```
历史：
用户：怎么获取 access_token？
系统：调用 POST /auth/token，传 client_id 和 client_secret...

当前追问：
用户：那 token 过期了呢？

直接用"那 token 过期了呢"检索 → 太短，语义不完整

改写：
"access_token 过期后如何刷新，refresh_token 使用方法"
```

```python
# 伪代码
standalone_query = llm.generate(f"""
根据以下对话历史，将当前追问改写成一个独立的完整问题（不依赖历史也能理解）：

对话历史：{conversation_history}
当前追问：{current_question}

只输出改写后的问题。
""")
results = retrieve(standalone_query)
```

---

## 知识点 2：HyDE（假设文档嵌入）

### 它在做什么

不用问题本身去检索，而是先让 LLM 根据问题生成一个"假设答案"，用这个假设答案去检索。

→ **术语注解：HyDE（Hypothetical Document Embeddings，假设文档嵌入）**，2022年 arXiv 论文提出。核心直觉：好问题的向量 ≠ 好答案的向量；而好答案的向量 ≈ 文档库里真实答案的向量。

**为什么有效——直觉解释：**

```
问题："怎么获取 access token？"
  ↓ 向量化
问题向量（疑问句风格，语义指向"获取方法"）

vs.

文档块："Bearer Token 认证：调用 POST /auth/token，传入 client_id 和 client_secret，返回 access_token..."
  ↓ 向量化
文档向量（陈述句风格，包含具体步骤和参数）

问题向量和文档向量之间有"风格鸿沟"，相似度不够高。

HyDE 的做法：
先让 LLM 生成一个假设答案："获取 access token 需要调用 /auth/token 接口，
传入 client_id 和 client_secret 参数..."
  ↓ 向量化
假设答案向量（和真实文档向量风格一致！）

假设答案向量 ≈ 真实文档向量 → 相似度更高 → 检索更准
```

### 实现流程

```python
# 伪代码
def hyde_retrieve(question, top_k=5):
    # Step 1：生成假设答案
    hypothetical_answer = llm.generate(f"""
    请根据以下技术问题，生成一个可能的技术文档段落作为答案。
    不需要准确，只需要像一篇技术文档的写法：
    
    问题：{question}
    """)
    
    # Step 2：用假设答案（而不是问题）去检索
    hyde_vector = embed(hypothetical_answer)
    results = vector_db.search(hyde_vector, top_k=top_k)
    
    return results
```

### 适用场景和局限

**适合：** 问题比较抽象或口语化，和文档写法差距大。

**不适合：**
- 精确查询（函数名、错误码）：HyDE 生成的假设答案可能包含错误术语，反而干扰检索
- 延迟敏感场景：多了一次 LLM 调用，延迟增加 200-500ms

**在贯穿案例里：** 可以和普通检索并行跑，用 HyDE 的结果补充普通检索的盲区：
```python
results_normal = retrieve(question, top_k=10)
results_hyde = hyde_retrieve(question, top_k=10)
final = rrf_merge(results_normal, results_hyde)[:5]
```

---

## 知识点 3：多跳检索（Multi-hop Retrieval）

### 它在做什么

有些问题需要跨多个文档片段，通过"推理链"才能回答，单次检索找不到完整答案。

**在贯穿案例里的典型场景：**

```
员工问："新版本的 TEMS 产品，OAuth 认证流程和旧版有什么变化？"

单次检索的局限：
→ 找到了"v3 认证流程"的块
→ 或者找到了"v2 认证流程"的块
→ 但两个块不一定同时被召回，LLM 无法比较

多跳检索的做法：
跳1：检索 "TEMS v3 OAuth 认证" → 找到 v3 文档
跳2：基于 v3 文档里提到的变化，再检索 "TEMS v2 OAuth 认证" 补充旧版信息
跳3：汇总两跳结果，回答差异
```

### 实现方式：迭代检索

```python
# 伪代码：简单的两跳检索
def multi_hop_retrieve(question, max_hops=3):
    all_chunks = []
    current_query = question
    
    for hop in range(max_hops):
        # 检索当前查询
        new_chunks = retrieve(current_query, top_k=3)
        all_chunks.extend(new_chunks)
        
        # 让 LLM 判断：已有信息够了吗？如果不够，下一跳应该查什么？
        decision = llm.generate(f"""
        用户问题：{question}
        已检索到的信息：{[c.text for c in all_chunks]}
        
        判断：
        1. 当前信息是否足以回答问题？（是/否）
        2. 如果否，还需要补充什么信息？请给出下一个检索查询。
        
        输出 JSON：{{"sufficient": true/false, "next_query": "..."}}
        """)
        
        if decision["sufficient"]:
            break
        
        current_query = decision["next_query"]
    
    return deduplicate(all_chunks)
```

### 局限性

多跳检索的代价：
- **延迟高**：每跳都需要 LLM 调用 + 向量检索，3 跳可能需要 3-5 秒
- **错误累积**：第一跳检索偏了，后续跳越走越偏
- **实现复杂**：需要仔细设计 LLM 的决策 Prompt

**在贯穿案例里的实用建议：** 先不做多跳检索。多数技术文档问题单次检索可以解决。只在评估发现"需要跨文档综合"的问题比例超过 20% 时，再考虑引入。

---

## 知识点 4：查询路由（Query Routing）

### 它在做什么

不是所有问题都该走同一个检索流程。根据问题类型，自动选择不同的检索路径。

**在贯穿案例里，可以把问题分类路由：**

```
问题类型           路由目标
-----------        -----------
精确查询           BM25 为主的稀疏检索（函数名、错误码）
语义查询           向量检索（口语问题、概念性问题）
跨文档问题         多跳检索
闲聊 / 范围外      直接回答"超出文档范围"，不检索
```

```python
# 伪代码：LLM 做路由分类
def route_query(question):
    route = llm.generate(f"""
    将以下问题分类为：
    - exact：精确查询（包含函数名、错误码、版本号等精确标识符）
    - semantic：语义查询（概念性、描述性问题）
    - multi_doc：需要跨多个文档对比或综合
    - out_of_scope：与技术文档无关的问题
    
    问题：{question}
    只输出分类标签。
    """)
    return route

# 根据路由结果选择检索策略
route = route_query(user_question)
if route == "exact":
    results = bm25_search(user_question)
elif route == "semantic":
    results = hybrid_search(user_question)
elif route == "multi_doc":
    results = multi_hop_retrieve(user_question)
elif route == "out_of_scope":
    return "该问题超出文档范围，建议联系相关团队"
```

**值不值得做：** 在贯穿案例这种规模的系统里，简单的路由（识别 out_of_scope + 精确查询走 BM25）有明显收益，复杂的多路由可以后期再加。

---

## 知识点 5：Self-RAG（自我反思检索）

### 它在做什么

让 LLM 自己决定：这个问题需不需要检索？检索到的内容相关吗？回答的质量够不够？

→ **术语注解：Self-RAG**，2023 年论文提出的框架。LLM 在生成过程中会插入"反思标记"，主动判断何时需要检索、检索结果是否相关、生成内容是否有依据。不是所有问题都需要检索（比如"1+1等于几"），Self-RAG 让 LLM 自主判断。

**标准 RAG vs Self-RAG 的区别：**

```
标准 RAG：
用户问题 → 强制检索 → 送 LLM → 回答
（无论问题是否需要检索，都走完整流程）

Self-RAG：
用户问题 → LLM 判断"需要检索吗？"
  → 不需要：直接回答
  → 需要：检索 → LLM 评估"检索结果相关吗？"
    → 不相关：换查询重新检索
    → 相关：生成回答 → LLM 评估"回答有依据吗？"
      → 有依据：输出
      → 没依据：标记为不确定或重新检索
```

**在贯穿案例里的价值：** 当员工问的是通用技术问题（比如"什么是 REST API"），不需要检索公司内部文档，直接回答更快更好。Self-RAG 能避免无意义的检索。

**实现复杂度：** 较高，需要模型支持或精心设计的 Prompt 链。初期可以用简单规则替代（如问题包含"我们公司"、"产品"等词才触发检索）。

---

## 模块小结

高级检索模式是在基础 RAG 稳定后的进阶优化，适用优先级：

```
查询改写（Query Rewriting）  ← 投入产出比最高，先做
混合检索 + Reranking         ← 已在模块 C 覆盖
HyDE                         ← 语义鸿沟大时有效，按需引入
查询路由                      ← 系统复杂后自然需要
多跳检索                      ← 有明确需求再做
Self-RAG                      ← 追求极致时考虑
```

**和下一个模块（G：系统架构与工程化）的衔接：**

前六个模块讲的都是"RAG 的功能怎么做好"。模块 G 的问题是：**这套功能怎么在生产环境里跑得稳、跑得快、跑得省钱。**

文档每周更新，怎么增量同步到向量库？高峰期并发 100 个请求，怎么不崩？同样的问题被问了 50 次，怎么不用每次都重新检索和调用 LLM？这些是工程化的问题，和功能设计同等重要。
