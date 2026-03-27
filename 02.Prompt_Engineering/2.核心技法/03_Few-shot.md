# Few-shot 提示（少样本示例）

> **一句话总结**：在 prompt 中嵌入 2～8 个"输入-输出"示例对，利用模型的 in-context learning 能力，无需微调即可对齐格式、风格与推理模式。

---

## 模块一：Few-shot 的本质——示例在教什么

### ① 一句话定义

Few-shot 示例不是在"教知识"，而是在向模型演示任务的接口契约：输入长什么样、输出长什么样、推理链用什么粒度展示。

### ② 底层原理

**In-context learning（情境学习）** 是 LLM 在预训练阶段习得的元能力：看到一批输入→输出对后，推断出背后的隐式映射规则，并将其应用于新输入。

模型从示例中实际学到三层内容：

```text
层次 1  格式契约（Format Contract）
  示例输入是 JSON → 期望输出也是 JSON
  示例用中文 → 输出用中文

层次 2  风格对齐（Style Alignment）
  示例语气正式 → 输出语气正式
  示例简洁 → 输出不冗长

层次 3  推理模式（Reasoning Pattern）
  示例先列条件再推结论 → 模型复现该结构
```

关键洞察：**示例不补充知识，模型已知道"怎么做"；示例告诉模型"在这个 context 下以哪种形式做"。**

反直觉现象：示例标签（如"答："、`Answer:`）换成随机词也能部分奏效——模型关注整体分布模式而非单词绝对含义。

### ③ 最小可运行示例

**Zero-shot（无示例）**

```text
System: 你是代码仓库 issue 分类助手。

User: crash: NullPointerException in PaymentService when cart is empty

Assistant: 这是一个 bug，支付服务在购物车为空时抛出空指针异常。
```

**Few-shot（两个示例）**

```text
System: 你是代码仓库 issue 分类助手。

User:
示例 1：
Input: fix: JWT token not refreshing after expiry, causes 401 on long sessions
Output: {"label": "bug", "component": "auth", "severity": "high"}

示例 2：
Input: docs: update API reference for /v2/users endpoint
Output: {"label": "docs", "component": "api", "severity": "low"}

现在请分析：
Input: crash: NullPointerException in PaymentService when cart is empty
Output:

Assistant:
{"label": "bug", "component": "payment", "severity": "high"}
```

**输出对比**：Zero-shot 输出是非结构化自然语言，难以程序解析；Few-shot 输出是可直接 `json.loads()` 的结构化数据。

### ④ 适用 vs 不适用场景

**适用**：
- API 响应 pipeline：下游服务需要可解析的 JSON，示例是最直接的格式锁定手段
- 领域特定风格：法律文书摘要必须遵循特定格式，示例比指令更精确

**不适用**：
- 开放式创意写作：无固定输出格式要求，Zero-shot 即可
- 需要注入知识的任务：示例无法补充模型不知道的事实，应使用 RAG

### ⑤ 常见误用和排查

| 误用 | 症状 | 排查方法 |
|------|------|----------|
| 认为示例在补充知识 | 模型仍然编造不知道的事实 | Few-shot 只改变形式不补充知识，用 RAG 注入知识 |
| 示例与指令冲突 | System 说只返回 JSON 但示例 output 含解释文字 | 示例优先级通常高于指令，保持一致 |
| 示例格式不统一 | 输出格式随机混用 | 模板化生成示例，统一所有字段名和结构 |

> **开发者视角**：Few-shot 是格式/风格对齐的最快路径，不是知识注入手段——明确这个边界才能用对它。

---

## 模块二：示例质量 vs 示例数量——哪个更重要

### ① 一句话定义

**质量比数量更重要**：1 个高质量示例通常优于 5 个平庸示例；示例数量超过一定阈值后边际收益趋近于零。

### ② 底层原理

Brown et al. 2020（GPT-3）的研究显示，增加示例数量的收益曲线呈对数增长。以下数字来自 GPT-3 级模型，**现代强模型的 Zero-shot 基线更高，实际收益曲线已经平移**（0 个示例的起点更好，收益拐点可能更早出现）：

- 0→1 个示例：收益最大（激活 in-context learning 模式）——在现代模型上，这一步有时收益较小，因为 Zero-shot 本身已很强
- 1→4 个示例：稳定提升（覆盖更多分布变体）
- 4～8 个示例：边际收益递减
- 8+ 个示例：几乎无收益，且挤占有效 context

```text
示例数   准确率提升（示意，基于 GPT-3 研究）
0→1     ++++ （最显著）
1→2     +++
2→4     ++
4→8     +
8→16    ≈0  （边际收益消失）
```

关键结论不随模型版本失效：**示例质量比数量更重要**。质量定义：代表性强、格式一致、推理链正确、覆盖边界情况。

**关于错误示范（negative examples）**：研究发现，示例中标签随机打错（random labels）不会显著降低模型性能——这说明模型更依赖示例的格式和结构，而非具体标签内容。但有意构造的错误推理链会误导模型，因为推理模式会被复现。

结论：纯格式示范中，标签准确性要求没那么高；推理链示范中，任何错误都会被放大复现。

### ③ 最小可运行示例

```text
# 5 个低质量示例（全部是同类型的简单成功状态）
示例 1: Input: tests passed    Output: success
示例 2: Input: build ok         Output: success
示例 3: Input: all green        Output: success
示例 4: Input: pipeline done    Output: success
示例 5: Input: deploy complete  Output: success

新输入: "unit tests: 143 passed. Deploy to prod: FAILED – OOM in container"
→ 模型输出: success  ← 错误！被示例分布拉偏，忽略了 FAILED 关键词

# 2 个高质量示例（覆盖 edge case：混合结果）
示例 1: Input: "Unit tests: 142 passed, 0 failed. Deploy: FAILED – DB connection timeout"
        Output: {"stage": "deploy", "root_cause": "infrastructure", "action": "check DB"}
示例 2: Input: "Build OK. Tests: 89 passed, 3 FAILED (TestPaymentFlow). Deploy: skipped"
        Output: {"stage": "test", "root_cause": "code", "action": "fix failing tests"}

新输入: "unit tests: 143 passed. Deploy to prod: FAILED – OOM in container"
→ 模型输出: {"stage": "deploy", "root_cause": "resource", "action": "increase memory limit"}
← 正确，理解了混合结果并定位到真正的失败点
```

### ④ 适用 vs 不适用场景

**适用（关注质量胜于数量）**：
- 生产环境批量处理任务：宁可花时间精心构造 3 个好示例而非堆 10 个
- 任务有明确 edge case：用 1-2 个专门针对高频失效点的示例

**不适用（需要更多示例）**：
- 输出分类标签很多（>10 类）：每类至少要有 1 个示例覆盖
- 输入格式极其多样化：需要足够示例覆盖不同输入变体

### ⑤ 常见误用和排查

| 误用 | 症状 | 排查方法 |
|------|------|----------|
| 无脑堆示例 | 超过 30% token 被示例占用，有效信息被挤掉 | 设置示例 token 上限，超出时切换动态选择 |
| 只用正面示例 | 遇到边界情况输出格式崩坏 | 主动构造 edge case 示例，至少 20-30% 覆盖困难样本 |
| 示例中含错误推理链 | 模型系统性复现错误推理 | 每个示例的推理链必须人工验证，这是最高优先级的质量检查 |

> **开发者视角**：从 0 到 1 个示例的收益最大；4 个示例后边际收益趋近于零；始终优先提升示例质量而非增加数量。

---

## 模块三：示例顺序的影响——靠近问题的示例权重更高吗

### ① 一句话定义

是的——LLM 存在 recency bias（近因偏差），距离新问题越近的示例对模型输出的影响权重越大。

### ② 底层原理

LLM 的 attention（注意力机制）对序列中不同位置的 token 权重分配不均匀。实验（Zhao et al. 2021）证明：
- 最后一个示例（紧邻新问题之前）的影响最大
- 第一个示例的影响次之（primacy effect，首因效应）
- 中间的示例影响最小（lost in the middle 现象）

```text
示例 1 [首位]   影响权重: ★★★★
示例 2 [中间]   影响权重: ★★
示例 3 [中间]   影响权重: ★★
示例 4 [末位]   影响权重: ★★★★★  ← 最高
新问题           ↑ 基于以上所有示例作答
```

实践含义：
- 如果你有一个最能代表当前问题的示例，把它放在最后
- 如果示例质量参差不齐，把最差的放中间
- 不要把 edge case 示例放在最后，除非新问题本身就是 edge case

### ③ 最小可运行示例

任务：从文本中提取 JSON 格式的产品信息。

**版本 A：最相关示例放最后（正确示范）**

```text
System: 从文本中提取产品信息，输出 JSON。

User:
示例 1（一般案例）：
Input: 苹果手机，价格 5999 元。
Output: {"name": "苹果手机", "price": 5999, "currency": "CNY"}

示例 2（与新问题最相似——含折扣价）：
Input: 耐克跑鞋原价 899 元，折后 699 元。
Output: {"name": "耐克跑鞋", "original_price": 899, "sale_price": 699, "currency": "CNY"}

现在请提取：
Input: 索尼耳机定价 2199 元，双十一特价 1599 元。

Assistant:
{"name": "索尼耳机", "original_price": 2199, "sale_price": 1599, "currency": "CNY"}  ✅
```

**版本 B：无关示例放最后（错误示范）**

```text
System: 从文本中提取产品信息，输出 JSON。

User:
示例 1（含折扣价，相关性高但放前面）：
Input: 耐克跑鞋原价 899 元，折后 699 元。
Output: {"name": "耐克跑鞋", "original_price": 899, "sale_price": 699, "currency": "CNY"}

示例 2（简单案例放最后，格式更简单）：
Input: 苹果手机，价格 5999 元。
Output: {"name": "苹果手机", "price": 5999}

现在请提取：
Input: 索尼耳机定价 2199 元，双十一特价 1599 元。

Assistant:
{"name": "索尼耳机", "price": 1599}  ❌ 丢失了 original_price 字段，跟随了最后的简单示例格式
```

### ④ 适用 vs 不适用场景

**适用（顺序影响显著）**：
- 静态示例库且示例质量有差异：最优先示例放末位
- 动态检索示例：按相似度排序后，最相似的放最后

**不适用（顺序影响较小）**：
- 示例数量少（1-2 个）且全部高质量：顺序差异可忽略
- 格式极其简单的任务（纯关键词提取）：模型不依赖顺序

### ⑤ 常见误用和排查

| 误用 | 症状 | 排查方法 |
|------|------|----------|
| 随机排列示例 | 输出质量不稳定，难以复现 | 固定顺序策略：最相关示例置于末位 |
| 把最差示例放最后 | 模型在最关键的格式细节上跟随最差示例 | 在 prompt 测试中轮换顺序，评估稳定性 |
| 忽略 lost-in-the-middle 效应 | 中间示例中的关键规则被忽视 | 把最重要的规则性示例放在首位或末位，不放中间 |

> **开发者视角**：示例顺序不是细节是变量——最相关示例放最后，最重要规则示例放最前，中间留给普通示例。

---

## 模块四：示例选取策略——随机、固定、动态检索的区别

### ① 一句话定义

示例选取策略决定了"用哪几个示例"，从简单到复杂有三种：随机抽样、固定精选、基于 RAG 的动态检索，适用场景依次递进。

### ② 底层原理

```text
策略        选取方式                           优点              缺点
随机选取     每次从示例库随机抽 k 个             零配置成本        输出不稳定，难以复现
固定精选     人工挑选 k 个最代表性示例           稳定可控          无法适应输入分布变化
动态检索     用户输入→向量化→检索最相似 k 个    自适应能力强      需要嵌入模型+向量数据库
```

动态检索的工作流：

```text
用户输入 → embedding 模型向量化 → 向量数据库检索 top-k 相似示例 → 按相似度排序 → 注入 prompt
```

RAG-based 示例选取的优势：当输入是"退款问题"时，自动检索与退款相关的示例；当输入是"物流问题"时，检索物流相关示例——同一套示例库，自适应覆盖不同场景。

### ③ 最小可运行示例

```python
import anthropic

# 示例库（生产环境中存储在向量数据库）
EXAMPLE_BANK = [
    {
        "input": "部署到 production 后 API 返回 502，staging 正常",
        "output": '{"category": "部署问题", "urgency": "high", "action": "回滚版本 + 检查 nginx 配置"}',
        "tags": ["部署", "502", "生产环境"]
    },
    {
        "input": "Jenkins pipeline 卡住 30 分钟了，不动了",
        "output": '{"category": "CI/CD", "urgency": "high", "action": "检查 agent 状态，终止卡死进程"}',
        "tags": ["Jenkins", "CI", "卡死", "pipeline"]
    },
    {
        "input": "怎么给新同事申请 GitHub 仓库的 write 权限",
        "output": '{"category": "权限申请", "urgency": "low", "action": "提交权限申请单，等 owner 审批"}',
        "tags": ["权限", "GitHub", "申请"]
    },
    {
        "input": "线上服务 CPU 飙到 95%，告警已触发",
        "output": '{"category": "线上告警", "urgency": "critical", "action": "立即扩容 + 查日志 + 通知 on-call"}',
        "tags": ["CPU", "告警", "线上", "紧急"]
    },
]


def select_examples(user_query: str, k: int = 2) -> list:
    """简化版相似度计算（生产环境用向量嵌入）"""
    query_words = set(user_query)
    scored = []
    for example in EXAMPLE_BANK:
        example_words = set(example["input"] + "".join(example["tags"]))
        overlap = len(query_words & example_words)
        scored.append((overlap, example))
    scored.sort(key=lambda x: x[0], reverse=True)
    # 最相似的放最后（recency bias 策略）
    selected = [ex for _, ex in scored[:k]]
    return list(reversed(selected))  # 最相关放末位


def build_prompt_with_examples(user_query: str) -> str:
    """构建带动态示例的 prompt"""
    examples = select_examples(user_query, k=2)
    example_text = ""
    for i, ex in enumerate(examples, 1):
        example_text += f"示例 {i}:\n"
        example_text += f"Input: {ex['input']}\n"
        example_text += f"Output: {ex['output']}\n\n"
    return f"""{example_text}现在请分析：
Input: {user_query}
Output:"""


def classify_customer_query(user_query: str) -> str:
    client = anthropic.Anthropic()
    prompt = build_prompt_with_examples(user_query)

    message = client.messages.create(
        model="claude-sonnet-4-6",
        max_tokens=256,
        messages=[
            {
                "role": "user",
                "content": prompt
            }
        ],
        system=(
            "你是开发者工单分类助手。"
            "根据用户输入，输出 JSON 格式的分类结果，包含 category、urgency、action 三个字段。"
            "只返回 JSON，不要其他文字。"
        )
    )
    return message.content[0].text


if __name__ == "__main__":
    query = "我的服务今天下午 3 点开始内存一直在涨，已经重启了两次了"
    result = classify_customer_query(query)
    print(f"输入: {query}")
    print(f"输出: {result}")
```

### ④ 适用 vs 不适用场景

**随机选取适用**：快速实验原型，示例库小（<10 条）且质量均匀

**固定精选适用**：生产环境，任务分布稳定，示例库小；要求输出完全可复现

**动态检索适用**：任务类型多样（如全能型客服）；示例库大（>20 条）；不同输入需要不同示范

### ⑤ 常见误用和排查

| 误用 | 症状 | 排查方法 |
|------|------|----------|
| 在生产环境用随机选取 | 相同输入不同运行输出格式不同 | 生产环境必须固定示例或使用确定性检索 |
| 动态检索只看语义相似度 | 检索到 k 个几乎相同的示例（多样性差） | 加入 MMR（最大边际相关性）去重，确保示例多样 |
| 示例库从不更新 | 新类型问题无示例覆盖，输出质量下降 | 建立示例评估流程，定期用失败案例补充示例库 |

> **开发者视角**：项目初期用固定精选快速验证，示例库超过 20 条且输入多样后切换动态检索——这是示例系统的自然演化路径。

---

## 模块五：Few-shot 失效场景——示例和问题分布不匹配时的表现

### ① 一句话定义

当示例覆盖的分布与实际输入分布不匹配时，Few-shot 不只无效，还会主动误导模型——产生比 Zero-shot 更差的结果。

### ② 底层原理

Few-shot 的 in-context learning 基于一个隐性假设：**新输入属于示例所示范的同一分布**。当这个假设被违反时：

```text
类型 1：输入复杂度失配
  示例：简单问句（"这个产品好吗？"）
  实际输入：复杂长文分析（技术规格对比 500 字）
  后果：模型用简单问句的回答粒度处理复杂问题，输出过于简短

类型 2：任务类型失配
  示例：二分类（positive/negative）
  实际输入：需要多标签输出
  后果：模型强行输出二分类，丢失细粒度信息

类型 3：领域失配
  示例：通用电商评论
  实际输入：专业医疗报告
  后果：模型套用电商语义，误解专业术语

类型 4：语言/格式失配
  示例：中文短句
  实际输入：中英混合技术文档
  后果：输出格式混乱，中英切换不一致
```

### ③ 最小可运行示例

```text
# Few-shot 示例（全部是单行简单 issue 标题）
示例 1: Input: 变量命名不规范      Output: style
示例 2: Input: 缺少注释            Output: docs
示例 3: Input: 有个拼写错误        Output: style

# 新输入：复杂多维 code review 意见
Input: 这段代码在单测里看起来没问题，但在高并发场景，两个线程同时走到
       if(cache.get(key) == null) 这里，会各自触发数据库查询，造成缓存击穿。
       同时 catch (Exception e) {} 会吞掉所有异常，上游完全感知不到失败，
       这在支付流程里是严重安全隐患。

# Zero-shot 输出（无示例）
分析：存在两个 critical 问题：1) 并发缓存击穿——应使用分布式锁或 singleflight；
2) 异常吞噬——catch 块必须记录日志并向上抛出，尤其在支付流程中。严重程度：critical。

# Few-shot 输出（有示例但分布不匹配）
style  ← 错误！被简单单行示例锁定，将严重并发 bug 误分类为风格问题
```

### ④ 适用 vs 不适用场景

**适用（分布匹配）**：
- 示例和新输入来自同一数据集或同一业务场景
- 已用评估集验证示例覆盖了真实输入的主要类型

**不适用（分布不匹配，应降级为 Zero-shot）**：
- 通用场景：用户输入高度多样，无法事先确定分布
- 跨领域迁移：用电商示例处理医疗/法律文本

### ⑤ 常见误用和排查

| 误用 | 症状 | 排查方法 |
|------|------|----------|
| 示例全是简单正例 | 遇到复杂输入模型回退到简单输出模式 | 构造与真实输入复杂度相当的示例，用评估集检验覆盖度 |
| 轻信"有示例就比无示例好" | Few-shot 在分布外输入上表现比 Zero-shot 差 | 建立基准评估：分别测 Zero-shot 和 Few-shot 在真实数据上的准确率，不假设 Few-shot 必然更好 |
| 示例库长期不更新 | 业务场景演变后 Few-shot 效果持续下滑 | 定期（如每月）用最新真实失败案例补充示例库，淘汰过时示例 |

> **开发者视角**：Few-shot 不是银弹——分布匹配时是利器，分布偏移时是干扰源；建评估基准，让数据告诉你该不该用。

---

## 综合对比：示例选取策略决策树

```text
任务输入分布稳定且示例库 < 20 条?
├── 是 → 固定精选（3-6 个覆盖典型 + edge case）
│         └── 任务需要展示推理过程?
│             ├── 是 → Few-shot CoT（示例含推理链）
│             └── 否 → 标准 Few-shot（仅 input/output）
└── 否 → 输入分布宽泛，示例库 > 20 条?
          ├── 是 → 动态示例检索（RAG-based）
          └── 否 → 先快速验证: Zero-shot 够用吗?
                    ├── 够 → 不用 Few-shot，减少复杂度
                    └── 不够 → 构建示例库，从固定精选开始
```

## 特别关注点

**收益曲线**：Few-shot 的收益随示例数量增加呈对数递减。

```text
效果
  ↑
  │  ●
  │    ●
  │      ●
  │        ●  ●  ●  ●
  │
  └──────────────────────→ 示例数量
     1  2  3  4  5  6  8

0→1 个示例：效果提升最显著（激活 in-context learning）
1→4 个示例：稳定提升，每个示例仍有可观收益
4→8 个示例：边际收益明显递减
8+ 个示例：收益趋近于零，token 成本持续增加
```

**错误示范的价值**：Stanford/CMU 等机构的研究发现，将示例标签随机打乱（random label scrambling）对模型性能的影响出乎意料地小。这揭示了两个重要推论：

1. **格式比标签内容更重要**：模型从示例中主要学习的是输入输出的结构模式，而非具体标签的语义含义。这意味着在纯格式示范场景下，标签的准确性要求没有我们直觉上认为的那么高。

2. **推理链中的错误会被放大**：虽然标签错误影响有限，但刻意构造的错误推理链（错误的 Chain-of-Thought）会显著误导模型——因为模型会直接复现推理模式。这意味着：
   - 格式示范（format-only demo）中，标签准确性是次要的
   - 推理示范（CoT demo）中，推理链的每一步都必须人工验证，任何错误都会被系统性地放大复现

实践结论：把质量检查资源集中在推理链的正确性上，而非在简单分类标签的准确性上过度投入。