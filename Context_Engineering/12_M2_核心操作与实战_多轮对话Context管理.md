# 多轮对话 Context 管理

**一句话：** 多轮对话的核心挑战不是"记住所有历史"，而是在有限的 context window 内，决定保留什么、压缩什么、丢弃什么——这些决策直接决定了助手的"记忆质量"。

## 是什么

当用户和 LLM 进行多轮对话时，每一轮对话都会被追加到 context 中。LLM 本身没有记忆——它每次处理请求时，都是把完整的对话历史作为 context 输入，重新"阅读"一遍，然后生成回答。

这意味着：

```
第 1 轮：context = system_prompt + user_msg_1
第 2 轮：context = system_prompt + user_msg_1 + assistant_msg_1 + user_msg_2
第 3 轮：context = system_prompt + user_msg_1 + assistant_msg_1 + user_msg_2 + assistant_msg_2 + user_msg_3
...
第 N 轮：context = system_prompt + 所有历史消息 + user_msg_N
```

问题显而易见：**对话越长，context 越大**。而 context window 有上限（例如 128K tokens）。即使 window 足够大，过长的 context 也会导致：

1. **成本急剧上升**：每轮对话都重新发送完整历史，token 消耗按轮次指数增长
2. **注意力稀释**：模型在海量历史信息中找到关键内容的能力下降
3. **延迟增加**：更多 token 意味着更长的处理时间

所以多轮对话 context 管理要解决的核心问题是：**在保持对话连贯性的前提下，控制 context 的大小**。

---

### 三大主流策略

#### 策略一：滑动窗口（Sliding Window）

最简单的方案——只保留最近的 N 轮对话，丢弃更早的。

```python
def sliding_window(messages: list[dict], system_prompt: str,
                   max_turns: int = 10) -> list[dict]:
    """只保留最近 N 轮对话"""
    # 分离 system prompt 和对话消息
    conversation = [m for m in messages if m["role"] != "system"]

    # 一轮 = 一组 user + assistant
    # 保留最近 max_turns 轮
    kept = conversation[-(max_turns * 2):]

    return [{"role": "system", "content": system_prompt}] + kept
```

**用「鲜果优选」客服助手演示：**

假设用户连续问了 15 个问题，滑动窗口 N=5（只保留最近 5 轮）：

```
轮次 1-5（已被丢弃）：
  用户问了芒果价格 → 客服回答了 39.9 元
  用户说"帮我下个单" → 客服确认了订单
  用户问"送到要几天" → 客服说 2-3 天
  用户给了收货地址 → 客服确认了地址
  用户说"对了，我对芒果不过敏吧" → 客服给了过敏提示

轮次 6-10（已被丢弃）：
  用户问"你们有草莓吗" → 客服介绍了草莓产品
  用户问"草莓和芒果能一起买打折吗" → 客服说有满减活动
  用户说"那草莓也来一盒" → 客服更新了订单
  用户问"能开发票吗" → 客服说明了发票流程
  用户说"算了发票不用了" → 客服确认

轮次 11-15（保留在 context 中）：
  用户问"我的订单什么时候发货" → 客服查询了物流
  用户说"能改成明天送吗" → 客服说需要看配送排期
  用户问"改地址还来得及吗" → 客服说订单已进入拣货
  用户说"那算了不改了" → 客服确认维持原订单
  用户现在问"对了，我一共买了什么来着？"
```

问题来了：用户问"我一共买了什么"，但下单的对话在轮次 1-2（芒果）和轮次 8（草莓），都已经被丢弃了。模型只能看到轮次 11-15 的内容，无法回答这个问题。

**滑动窗口的优缺点：**

| 优点 | 缺点 |
|------|------|
| 实现极其简单 | 早期重要信息会丢失 |
| 性能可预测（固定大小） | 无法处理"回顾性"问题 |
| 无额外 API 调用 | 对话连贯性在窗口边界断裂 |

---

#### 策略二：摘要压缩（Summarization）

把较早的对话压缩成一段摘要，保留关键信息，减少 token 数量。

```python
async def summarize_old_context(
    client,
    old_messages: list[dict],
    existing_summary: str | None = None
) -> str:
    """将旧的对话历史压缩为摘要"""

    summary_prompt = """请将以下客服对话历史压缩为简洁的摘要。
必须保留以下关键信息：
- 用户已下的订单（产品、数量、价格）
- 用户提供的个人信息（地址、联系方式）
- 未解决的问题或待跟进的事项
- 用户明确表达的偏好或要求

可以省略的信息：
- 寒暄、礼貌用语
- 已经解决且不影响后续对话的问题
- 重复确认的信息"""

    messages_text = format_messages_as_text(old_messages)

    if existing_summary:
        content = f"已有摘要：\n{existing_summary}\n\n新增对话：\n{messages_text}"
    else:
        content = f"对话历史：\n{messages_text}"

    response = await client.chat.completions.create(
        model="gpt-4o-mini",  # 用小模型做摘要，节省成本
        messages=[
            {"role": "system", "content": summary_prompt},
            {"role": "user", "content": content}
        ],
        max_tokens=500
    )

    return response.choices[0].message.content
```

**同样的 15 轮对话场景，摘要策略的处理方式：**

当对话进行到第 10 轮时，触发摘要（比如每 8 轮压缩一次）：

```
生成的摘要：
"用户已下单越南金枕芒果（500g/盒，39.9元）和丹东草莓（1盒），
收货地址已确认。配送预计2-3天。用户不需要开发票。
用户此前询问过芒果过敏问题，已告知相关提示。
订单享受了满减活动优惠。"
```

Context 结构变成：

```
┌─────────────────────────────────────────┐
│ System Prompt                           │
├─────────────────────────────────────────┤
│ 对话摘要（替代轮次 1-10）                 │
│ "用户已下单越南金枕芒果和丹东草莓..."      │
├─────────────────────────────────────────┤
│ 完整保留的近期对话（轮次 11-15）           │
│ User: 我的订单什么时候发货？              │
│ Assistant: 您的订单已进入拣货环节...       │
│ ...                                     │
├─────────────────────────────────────────┤
│ User: 对了，我一共买了什么来着？           │
└─────────────────────────────────────────┘
```

现在用户问"我一共买了什么"，模型可以从摘要中找到答案。

**摘要的优缺点：**

| 优点 | 缺点 |
|------|------|
| 保留了关键信息 | 需要额外的 API 调用来生成摘要 |
| Context 大小可控 | 摘要本身可能丢失或扭曲信息 |
| 对话连贯性较好 | 增加系统复杂度 |
| 可以处理回顾性问题 | 摘要质量依赖于 prompt 设计 |

---

#### 策略三：选择性保留（Selective Retention）

最精细的方案——评估每一轮对话的"重要性"，只保留重要的，丢弃不重要的。

```python
def classify_message_importance(message: dict, context: dict) -> str:
    """评估单条消息的重要性等级"""

    # 规则1：包含订单操作的消息 → 关键
    if any(kw in message["content"] for kw in ["下单", "购买", "订单", "付款"]):
        return "critical"

    # 规则2：包含用户个人信息的消息 → 关键
    if any(kw in message["content"] for kw in ["地址", "电话", "姓名"]):
        return "critical"

    # 规则3：包含投诉或负面情绪的消息 → 重要
    if any(kw in message["content"] for kw in ["投诉", "不满", "差评", "退款"]):
        return "important"

    # 规则4：确认性回复 → 低重要性
    if message["content"].strip() in ["好的", "嗯", "OK", "知道了", "行"]:
        return "low"

    # 规则5：闲聊 → 低重要性
    if context.get("is_chitchat", False):
        return "low"

    return "normal"


def selective_retain(messages: list[dict], max_tokens: int = 4000) -> list[dict]:
    """根据重要性选择性保留消息"""
    scored_messages = []
    for msg in messages:
        importance = classify_message_importance(msg, {})
        scored_messages.append((msg, importance))

    # 始终保留 critical 和 important
    retained = [m for m, imp in scored_messages if imp in ("critical", "important")]

    # 用 normal 消息填充剩余空间
    remaining_budget = max_tokens - estimate_tokens(retained)
    normal_messages = [m for m, imp in scored_messages if imp == "normal"]

    for msg in reversed(normal_messages):  # 从最近的开始保留
        msg_tokens = estimate_tokens([msg])
        if remaining_budget >= msg_tokens:
            retained.append(msg)
            remaining_budget -= msg_tokens

    # 按原始顺序排列
    return sort_by_original_order(retained, messages)
```

**同样的 15 轮场景：**

```
保留的消息（标注了重要性）：
[critical] 用户: 帮我下个越南金枕芒果，500g的
[critical] 助手: 好的，已为您添加越南金枕芒果500g/盒，39.9元
[critical] 用户: 收货地址是朝阳区XX路XX号
[critical] 助手: 地址已确认：朝阳区XX路XX号
[important] 用户: 对了，我对芒果不过敏吧
[important] 助手: 芒果含漆酚类物质...建议少量尝试
[critical] 用户: 那草莓也来一盒
[critical] 助手: 已添加丹东草莓1盒，您的订单现在包含...
[normal]   用户: 我的订单什么时候发货
[normal]   助手: 订单已进入拣货环节，预计明天发出

丢弃的消息：
[low] 用户: 好的 / 嗯 / 知道了（多条确认性回复）
[low] 关于发票的讨论（用户最终说不需要了，结论已在保留消息中体现）
[normal] 部分草莓产品介绍（用户已经决定购买了）
```

**选择性保留的优缺点：**

| 优点 | 缺点 |
|------|------|
| 信息保留质量最高 | 实现复杂度最高 |
| 灵活控制保留策略 | 重要性判断可能出错 |
| 不需要额外 API 调用（如果用规则） | 需要领域知识来定义规则 |
| 保留了原始对话细节 | 对话可能出现"跳跃"（中间轮次被删了） |

---

### 混合方案：实践中的最佳选择

真实生产系统几乎都采用混合方案。以「鲜果优选」客服助手为例：

```python
class ConversationManager:
    """混合对话管理器：摘要 + 选择性保留 + 滑动窗口"""

    def __init__(self, max_context_tokens: int = 8000):
        self.max_context_tokens = max_context_tokens
        self.summary = None
        self.pinned_messages = []   # 关键消息，永不丢弃
        self.recent_messages = []   # 近期完整消息

    def add_turn(self, user_msg: str, assistant_msg: str):
        """添加一轮新对话"""
        user_message = {"role": "user", "content": user_msg}
        assistant_message = {"role": "assistant", "content": assistant_msg}

        # 判断是否为关键消息
        if self._is_critical(user_msg) or self._is_critical(assistant_msg):
            self.pinned_messages.append(user_message)
            self.pinned_messages.append(assistant_message)
        else:
            self.recent_messages.append(user_message)
            self.recent_messages.append(assistant_message)

        # 检查是否需要压缩
        if self._total_tokens() > self.max_context_tokens:
            self._compress()

    def _is_critical(self, content: str) -> bool:
        """判断消息是否为关键信息"""
        critical_patterns = [
            "下单", "购买", "订单号", "退款", "投诉",
            "地址", "电话", "发票",
        ]
        return any(p in content for p in critical_patterns)

    def _compress(self):
        """压缩对话历史"""
        # 1. 先用滑动窗口裁剪 recent_messages
        if len(self.recent_messages) > 10:
            old_messages = self.recent_messages[:-10]
            self.recent_messages = self.recent_messages[-10:]

            # 2. 把被裁剪的消息总结进 summary
            self.summary = self._update_summary(old_messages)

    def _update_summary(self, old_messages: list[dict]) -> str:
        """更新摘要（合并旧摘要和新的被淘汰消息）"""
        # 实际实现中会调用 LLM 生成摘要
        # 这里用伪代码表示
        return summarize(
            existing_summary=self.summary,
            new_messages=old_messages
        )

    def build_context(self, system_prompt: str) -> list[dict]:
        """构建完整的 context"""
        messages = [{"role": "system", "content": system_prompt}]

        # 层级1：对话摘要（最早的信息，已压缩）
        if self.summary:
            messages.append({
                "role": "system",
                "content": f"## 对话摘要\n{self.summary}"
            })

        # 层级2：固定的关键消息（不压缩，不丢弃）
        if self.pinned_messages:
            messages.append({
                "role": "system",
                "content": "## 重要对话记录（以下为本次会话中的关键信息）"
            })
            messages.extend(self.pinned_messages)

        # 层级3：近期完整对话（最新的几轮）
        messages.extend(self.recent_messages)

        return messages
```

这个混合方案产生的 context 结构：

```
┌──────────────────────────────────────────┐
│ System Prompt                            │
│ "你是鲜果优选的客服助手..."               │
├──────────────────────────────────────────┤
│ 对话摘要（第 1-6 轮的压缩版）              │
│ "用户咨询了芒果和草莓产品，对比了价格..."    │
├──────────────────────────────────────────┤
│ 关键消息（固定，不随时间淘汰）              │
│ [U] 帮我下个单，芒果和草莓各一盒           │
│ [A] 已为您创建订单 ORD-20240315-001...    │
│ [U] 地址是朝阳区XX路XX号                  │
│ [A] 收货地址已确认                        │
├──────────────────────────────────────────┤
│ 近期对话（完整保留最近 5 轮）               │
│ [U] 我的订单什么时候发货？                 │
│ [A] 订单已进入拣货环节...                  │
│ ...                                      │
├──────────────────────────────────────────┤
│ [U] 对了，我一共买了什么来着？              │
└──────────────────────────────────────────┘
```

---

### Session State：超越消息的状态追踪

对话管理不仅仅是管理消息列表。还需要追踪对话中产生的"状态"：

```python
class SessionState:
    """对话会话状态"""

    def __init__(self):
        # 用户意图追踪
        self.current_intent = None       # 当前主要意图
        self.pending_intents = []        # 未完成的意图队列

        # 话题追踪
        self.current_topic = None        # 当前讨论话题
        self.topic_history = []          # 话题切换历史

        # 业务状态
        self.cart_items = []             # 购物车
        self.active_order_id = None      # 当前讨论的订单
        self.user_info = {}              # 已收集的用户信息

        # 对话质量信号
        self.unresolved_questions = []   # 未回答的问题
        self.sentiment_trend = []        # 情绪变化趋势

    def inject_as_context(self) -> str:
        """将会话状态转化为 context 文本"""
        parts = []

        if self.cart_items:
            items = "\n".join(f"- {item}" for item in self.cart_items)
            parts.append(f"### 当前购物车\n{items}")

        if self.active_order_id:
            parts.append(f"### 当前讨论的订单\n订单号：{self.active_order_id}")

        if self.unresolved_questions:
            questions = "\n".join(f"- {q}" for q in self.unresolved_questions)
            parts.append(f"### 用户尚未得到回答的问题\n{questions}")

        if self.current_topic:
            parts.append(f"### 当前话题\n{self.current_topic}")

        return "\n\n".join(parts)
```

将 session state 注入 context 的好处：即使对话历史被压缩或裁剪，关键业务状态仍然清晰可见。

例如，用户在第 3 轮提到了"我之前有个订单出问题了"，到了第 12 轮突然说"对了那个订单退款了吗"。即使第 3 轮的对话已经被摘要或丢弃，`session_state.unresolved_questions` 中仍然记录着"订单问题待跟进"，模型可以据此继续处理。

---

### 何时重置 Context vs 继续沿用

不是所有对话都应该无限延续。判断何时"断开"也是设计决策：

```python
def should_reset_context(session_state: SessionState,
                         last_message_time: datetime,
                         current_time: datetime) -> bool:
    """判断是否应该重置对话 context"""

    # 场景1：超时（用户离开超过 30 分钟）
    if (current_time - last_message_time).total_seconds() > 1800:
        return True

    # 场景2：用户明确表示新话题
    # "我想问个别的问题"、"换个话题"
    if session_state.explicit_topic_switch:
        return True

    # 场景3：当前会话的所有意图都已完成
    if (not session_state.pending_intents and
        not session_state.unresolved_questions):
        return True

    return False


def reset_context(session_state: SessionState) -> str:
    """重置 context，但保留必要的跨会话信息"""
    carry_forward = []

    # 保留用户身份信息
    if session_state.user_info:
        carry_forward.append(
            f"回访用户信息：{session_state.user_info}"
        )

    # 保留未解决的问题
    if session_state.unresolved_questions:
        carry_forward.append(
            f"上次会话遗留问题：{session_state.unresolved_questions}"
        )

    return "\n".join(carry_forward) if carry_forward else ""
```

**重置 vs 继续的决策框架：**

| 信号 | 建议操作 | 原因 |
|------|---------|------|
| 用户超时 30 分钟后回来 | 重置，但保留用户身份和未解决问题 | 用户可能已经忘记之前的上下文 |
| 用户说"换个话题" | 部分重置（清除当前话题，保留订单状态） | 尊重用户意图，但不丢失业务状态 |
| 所有问题已解决，用户说"谢谢" | 完全重置 | 干净地结束会话 |
| 对话超过 50 轮但仍在同一个问题上 | 不重置，但需要更积极地压缩 | 用户可能在处理复杂问题 |
| 用户切换了设备或渠道 | 重置对话格式，但恢复 session state | 保持业务连续性 |

---

## 为什么重要

### 场景一：不做管理的后果

一个真实的电商客服对话可能这样发展：

```
第 1 轮：用户问"有什么水果推荐"     →  助手推荐了 5 种水果（~300 tokens）
第 2 轮：用户问每种水果的详情        →  助手详细介绍了 5 种（~800 tokens）
第 3 轮：用户问"芒果和草莓怎么选"    →  助手做了对比（~400 tokens）
第 4 轮：用户下单                   →  助手确认订单（~200 tokens）
第 5 轮：用户问配送                  →  助手查了物流（~300 tokens）
...
第 20 轮：累计对话 ~10,000 tokens
```

如果不做任何管理：
- **第 20 轮的 API 调用**需要发送 10,000+ tokens 的历史，其中 80% 对当前问题毫无帮助
- **成本**：假设输入 $3/M tokens，20 轮的总输入 token 约为 1+2+3+...+20 ≈ 100K tokens，仅输入就花费 $0.30。而使用滑动窗口（保留 5 轮），总输入约为 20K tokens，节省 80%
- **质量下降**：模型要在 10,000 tokens 的"大海"里捞到当前问题相关的"针"

### 场景二：摘要的信息损失问题

用户第 3 轮说："我不要太甜的，上次买的荔枝甜到腻了。"

原始消息明确传达了两个信息：(1) 偏好不太甜 (2) 有过荔枝的不好体验。

摘要后可能变成："用户偏好不太甜的水果。"

当第 15 轮用户问"你觉得我会喜欢荔枝吗"，如果只有摘要，模型可能回答"荔枝甜度较高，根据您偏好不太甜的水果，可能不太适合"——这是一个正确但平庸的回答。

如果有原始消息，模型可以回答"您之前提到买过荔枝觉得太甜腻了，所以可能还是不太适合您。要不试试我们的青提？甜度适中，口感也很好。"——这个回答明显更好，因为它引用了用户的具体体验。

这就是为什么选择性保留（保留关键原始消息）在某些场景下比纯摘要效果更好。

---

## 设计启示

### 1. 对话管理策略应该匹配业务场景

不同场景需要不同的策略组合：

```
简单问答场景（用户问一两个问题就走）：
  → 不需要复杂管理，滑动窗口足够

电商客服（跨话题、有订单状态）：
  → 混合方案：session state + 摘要 + 选择性保留关键消息

技术调试（需要完整的错误上下文）：
  → 保守压缩，尽量保留原始信息

情感陪伴（用户期望"被记住"）：
  → 重度使用选择性保留，用户的情感表达是关键信息
```

### 2. 把"状态"和"消息"分开管理

对话消息可以被压缩或丢弃，但业务状态应该独立维护：

```
对话消息（可压缩、可丢弃）：
  "我想买芒果" → "好的，给您推荐..." → "来一盒" → "已加入购物车"

业务状态（持久化，不随对话压缩丢失）：
  cart = ["越南金枕芒果 500g × 1"]
  user_preference = {"sweetness": "低"}
  active_order = None
```

这样即使整段对话被摘要成一句话，购物车的状态仍然精确。

### 3. 压缩的时机很重要

两种常见策略：

- **按轮次触发**：每 N 轮压缩一次。简单可预测，但可能在关键对话中途触发压缩。
- **按 token 数触发**：当 context 接近上限时压缩。更灵活，但需要实时计算 token 数。

推荐做法：**按 token 数触发，但避免在关键操作中途压缩**。比如用户正在确认订单的过程中（"选了芒果" → "加草莓" → "确认收货地址" → "提交订单"），这几轮应该作为一个原子单元，不在中间压缩。

```python
def should_compress(context_tokens: int, max_tokens: int,
                    session_state: SessionState) -> bool:
    """判断是否应该触发压缩"""
    # 达到阈值（通常设为 max 的 75%）
    if context_tokens < max_tokens * 0.75:
        return False

    # 但如果正在进行关键操作，延迟压缩
    if session_state.current_intent in ("placing_order", "processing_refund"):
        return False

    return True
```

### 4. 为压缩后的 context 设置"锚点"

摘要容易变成一堆模糊的描述。给摘要加上结构化的锚点：

```
## 对话摘要

### 已确认事实
- 用户姓名：张先生
- 收货地址：朝阳区XX路XX号
- 订单号：ORD-20240315-001

### 用户偏好
- 不喜欢太甜的水果
- 偏好小包装（1-2人份）

### 对话进展
- 用户已下单芒果和草莓
- 配送预计明天到达
- 用户不需要发票

### 待跟进
- 无
```

比起"用户是张先生，住朝阳区，下了芒果和草莓的订单，不要发票，预计明天到"这样的流水账摘要，结构化锚点让模型更容易定位和使用信息。

---

## 常见误区

### 误区一："Context window 够大，直接保留全部历史就行"

**错误理解：** 现在 128K 甚至更大的 context window，不需要做对话管理了。

**实际情况：**
- **成本问题**：128K tokens 的输入，按 GPT-4 级别模型计算，单次请求成本不低。一个繁忙的客服系统如果每个对话都不压缩，成本会非常可观。
- **注意力问题**："Lost in the Middle"现象——模型对超长 context 的中间部分注意力下降。100 轮对话中，第 30-70 轮的内容最容易被忽略。
- **延迟问题**：更多 token = 更长的首 token 延迟（TTFT）。客服场景对响应速度敏感。

正确的做法：即使 context window 足够大，仍然应该做对话管理，这是一个成本-质量-速度的三角权衡。

### 误区二："摘要不会丢失重要信息"

**错误理解：** 只要摘要 prompt 写得好，就能完美保留所有重要信息。

**实际情况：** 摘要必然是有损的。你无法事先知道哪些信息在未来会变得"重要"。

例如：用户第 2 轮随口提了一句"我家猫喜欢吃草莓"。在摘要时，这句话大概率会被当作闲聊丢弃。但到了第 20 轮，用户问"你们有适合猫吃的零食吗"，如果原始消息还在，模型能做出更好的关联回答。

应对策略：
- 对"可能有用但不确定"的信息，宁可保留到下一轮压缩，不急于丢弃
- 用 session state 单独存储明确的事实（用户偏好、提到的实体等），不完全依赖摘要

### 误区三："每一轮对话都是独立的"

**错误理解：** 每轮只需要 system prompt + 当前 user message，不需要历史。

**实际情况：** 大多数有价值的对话都是多轮的，前后文紧密关联。

```
User: 芒果多少钱？
Assistant: 39.9元/盒。
User: 草莓呢？            ← "呢"指代的是"多少钱"，没有历史就无法理解
Assistant: 草莓29.9元/盒。
User: 那就两个都来一盒。    ← "两个"指代芒果和草莓，没有历史就不知道是什么
```

如果只发当前消息"那就两个都来一盒"，模型完全不知道用户在说什么。对话历史（或至少对话状态）是必须的。

正确的认知：每一轮都不是独立的，但也不是每一轮都同等重要。Context 管理的核心就是在这两个极端之间找到最优解。
