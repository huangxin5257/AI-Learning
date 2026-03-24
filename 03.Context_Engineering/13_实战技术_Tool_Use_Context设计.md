# Tool Use Context 设计

**一句话：** Tool Use（工具调用）的本质是让模型"动手做事"而不只是"动嘴说话"，而你写的工具描述和结果格式就是在做 Context Engineering——你在设计模型理解和使用工具的全部信息环境。

## 是什么

Tool Use（也叫 Function Calling）是一种让 LLM 调用外部函数的机制。流程如下：

```
1. 你在 context 中告诉模型：有哪些工具可用，每个工具做什么，需要什么参数
2. 用户提问
3. 模型分析问题，决定是否需要调用工具。如果需要，输出一个结构化的"调用请求"
4. 你的代码执行这个调用，拿到结果
5. 你把结果注入回 context
6. 模型基于工具结果生成最终回答
```

用「鲜果优选」客服助手的例子来走一遍完整流程：

```
用户："我上周下的订单到哪了？订单号是 ORD-20240308-042"

→ 模型看到可用工具列表，决定调用 query_order(order_id="ORD-20240308-042")
→ 你的代码执行查询，从数据库拿到订单数据
→ 你把订单数据注入回 context
→ 模型基于订单数据回答："您的订单目前已从北京仓库发出，正在配送中，
   预计明天下午送达。快递单号是 SF1234567890，您可以在顺丰官网查询。"
```

**为什么这是 Context Engineering？**

因为整个过程中，有两个关键的"context 设计"环节：

1. **工具描述**（Tool Description）：你告诉模型有什么工具、怎么用——这段文本决定了模型能否正确决策"该不该用"和"怎么用"
2. **工具结果注入**（Tool Result Injection）：你把工具执行结果放回 context——这段文本的格式决定了模型能否正确理解和使用结果

这两个环节的质量直接决定了工具调用的成功率。

---

### 工具定义：告诉模型"你有什么工具"

以「鲜果优选」客服助手为例，定义三个核心工具：

```python
tools = [
    {
        "type": "function",
        "function": {
            "name": "query_order",
            "description": "查询用户的订单信息，包括订单状态、配送进度、商品明细。当用户询问订单相关问题时使用此工具。需要用户提供订单号。",
            "parameters": {
                "type": "object",
                "properties": {
                    "order_id": {
                        "type": "string",
                        "description": "订单编号，格式为 ORD-YYYYMMDD-NNN，例如 ORD-20240315-001"
                    }
                },
                "required": ["order_id"]
            }
        }
    },
    {
        "type": "function",
        "function": {
            "name": "search_products",
            "description": "在商品目录中搜索水果产品。返回匹配的产品列表，包含名称、价格、规格、库存状态。当用户询问有什么产品、某类水果的信息、或进行比价时使用。",
            "parameters": {
                "type": "object",
                "properties": {
                    "keyword": {
                        "type": "string",
                        "description": "搜索关键词，如水果名称、品类。例如：'芒果'、'草莓'、'热带水果'"
                    },
                    "category": {
                        "type": "string",
                        "enum": ["tropical", "berry", "citrus", "melon", "stone_fruit", "other"],
                        "description": "产品分类筛选（可选）"
                    },
                    "max_price": {
                        "type": "number",
                        "description": "价格上限（元），用于筛选价格范围内的产品（可选）"
                    }
                },
                "required": ["keyword"]
            }
        }
    },
    {
        "type": "function",
        "function": {
            "name": "check_inventory",
            "description": "查询指定产品的实时库存状态和预计补货时间。当用户想购买某产品、或询问是否有货时使用。注意：此工具只返回库存信息，不返回价格，价格请用 search_products 查询。",
            "parameters": {
                "type": "object",
                "properties": {
                    "product_id": {
                        "type": "string",
                        "description": "产品ID，格式为字母+数字，例如 MG001、SB002。可以从 search_products 的返回结果中获取。"
                    }
                },
                "required": ["product_id"]
            }
        }
    }
]
```

**工具描述中每一个字都很重要。** 拆解一下好的工具描述应该包含什么：

| 要素 | 说明 | 示例 |
|------|------|------|
| **做什么** | 工具的核心功能 | "查询用户的订单信息" |
| **返回什么** | 调用后能拿到什么数据 | "包括订单状态、配送进度、商品明细" |
| **何时使用** | 模型判断是否调用的依据 | "当用户询问订单相关问题时使用" |
| **前置条件** | 调用前需要什么 | "需要用户提供订单号" |
| **参数格式** | 怎么传参数 | "格式为 ORD-YYYYMMDD-NNN" |
| **边界说明** | 这个工具不做什么 | "此工具只返回库存信息，不返回价格" |

对比好的和差的工具描述：

```
# 差的描述
"name": "query_order",
"description": "查订单"

# 好的描述
"name": "query_order",
"description": "查询用户的订单信息，包括订单状态、配送进度、商品明细。
当用户询问订单相关问题时使用此工具。需要用户提供订单号。"
```

差的描述带来的问题：
- 模型不知道能查到什么信息（状态？物流？价格？）
- 模型不知道什么时候该调用（用户随口提到"订单"就调？还是只有明确查询时才调？）
- 模型不知道需要什么参数（用户没给订单号时模型可能传一个空字符串）

---

### 工具结果注入：告诉模型"工具返回了什么"

当模型决定调用工具后，你的代码执行调用并拿到原始结果。接下来关键问题是：**怎么把结果放回 context？**

#### 原始 JSON vs 格式化结果

**原始 JSON（不推荐直接使用）：**

```json
{
  "status": "success",
  "data": {
    "order_id": "ORD-20240308-042",
    "user_id": "USR-10042",
    "status_code": 3,
    "status_text": "in_transit",
    "created_at": "2024-03-08T14:23:11.000Z",
    "updated_at": "2024-03-14T09:15:33.000Z",
    "shipping": {
      "carrier": "SF",
      "tracking_number": "SF1234567890",
      "estimated_delivery": "2024-03-15T18:00:00.000Z",
      "current_location": {
        "city": "北京",
        "facility": "北京转运中心",
        "lat": 39.9042,
        "lng": 116.4074
      },
      "events": [
        {"time": "2024-03-14T09:15:33.000Z", "description": "包裹已到达北京转运中心", "code": "TRANSIT"},
        {"time": "2024-03-13T16:42:00.000Z", "description": "包裹已从广州仓库发出", "code": "SHIPPED"},
        {"time": "2024-03-13T10:30:00.000Z", "description": "商品已打包", "code": "PACKED"},
        {"time": "2024-03-08T14:23:11.000Z", "description": "订单已创建", "code": "CREATED"}
      ]
    },
    "items": [
      {
        "product_id": "MG001",
        "sku": "MG2024031500",
        "name": "越南金枕芒果",
        "spec": "500g/盒",
        "quantity": 2,
        "unit_price_cents": 3990,
        "subtotal_cents": 7980
      }
    ],
    "payment": {
      "method": "wechat_pay",
      "total_cents": 7980,
      "discount_cents": 0,
      "paid_cents": 7980,
      "paid_at": "2024-03-08T14:25:00.000Z"
    },
    "internal_notes": "优先配送客户，VIP等级3",
    "warehouse_id": "WH-GZ-001"
  }
}
```

这个原始 JSON 有很多问题：
- 包含内部字段（`user_id`、`warehouse_id`、`internal_notes`、`sku`）——模型可能会把这些暴露给用户
- 价格用"分"而不是"元"（`unit_price_cents: 3990`）——模型可能直接说"3990元"
- 经纬度信息完全无用——浪费 token
- `status_code: 3` 没有人类可读的含义

**格式化后的结果（推荐）：**

```python
def format_order_result(raw_data: dict) -> str:
    """将订单查询的原始结果格式化为模型可读的 context"""
    data = raw_data["data"]
    shipping = data["shipping"]
    items = data["items"]

    # 格式化商品列表
    items_text = "\n".join(
        f"  - {item['name']}（{item['spec']}）× {item['quantity']}，"
        f"单价 {item['unit_price_cents']/100:.1f} 元"
        for item in items
    )

    # 格式化物流事件（只保留最近 3 条）
    events_text = "\n".join(
        f"  - {event['description']}（{format_time(event['time'])}）"
        for event in shipping["events"][:3]
    )

    total = data["payment"]["paid_cents"] / 100

    return f"""### 订单查询结果
订单号：{data['order_id']}
订单状态：配送中
下单时间：{format_time(data['created_at'])}

商品明细：
{items_text}

实付金额：{total:.1f} 元

物流信息：
  快递公司：顺丰速运
  快递单号：{shipping['tracking_number']}
  预计送达：{format_time(shipping['estimated_delivery'])}
  当前位置：{shipping['current_location']['city']}

最新物流动态：
{events_text}"""
```

格式化后注入 context 的内容：

```
### 订单查询结果
订单号：ORD-20240308-042
订单状态：配送中
下单时间：2024年3月8日 14:23

商品明细：
  - 越南金枕芒果（500g/盒）× 2，单价 39.9 元

实付金额：79.8 元

物流信息：
  快递公司：顺丰速运
  快递单号：SF1234567890
  预计送达：2024年3月15日 18:00
  当前位置：北京

最新物流动态：
  - 包裹已到达北京转运中心（3月14日 09:15）
  - 包裹已从广州仓库发出（3月13日 16:42）
  - 商品已打包（3月13日 10:30）
```

**对比效果：**

| 维度 | 原始 JSON | 格式化结果 |
|------|----------|-----------|
| Token 数 | ~350 tokens | ~150 tokens |
| 内部信息泄露风险 | 高（含 VIP 等级、仓库 ID） | 无 |
| 模型理解难度 | 需要解析嵌套 JSON | 直接可读 |
| 数据格式 | 分为单位、ISO 时间戳 | 元为单位、人类可读时间 |

---

### 完整的对话示例

展示一个包含工具调用的完整对话流程：

```python
# 完整的 messages 构建过程

messages = [
    # 1. System Prompt
    {
        "role": "system",
        "content": """你是鲜果优选的客服助手。
规则：
- 使用工具查询真实数据后再回答，不要猜测
- 如果用户没有提供必要的参数（如订单号），先询问用户
- 回答要简洁友好，突出用户最关心的信息"""
    },

    # 2. 用户消息
    {
        "role": "user",
        "content": "我上周下的订单到哪了？订单号是 ORD-20240308-042"
    },

    # 3. 模型的工具调用请求（由 API 返回）
    {
        "role": "assistant",
        "content": None,
        "tool_calls": [
            {
                "id": "call_abc123",
                "type": "function",
                "function": {
                    "name": "query_order",
                    "arguments": "{\"order_id\": \"ORD-20240308-042\"}"
                }
            }
        ]
    },

    # 4. 工具执行结果（由你的代码注入）
    {
        "role": "tool",
        "tool_call_id": "call_abc123",
        "content": """### 订单查询结果
订单号：ORD-20240308-042
订单状态：配送中
下单时间：2024年3月8日 14:23

商品明细：
  - 越南金枕芒果（500g/盒）× 2，单价 39.9 元

实付金额：79.8 元

物流信息：
  快递公司：顺丰速运
  快递单号：SF1234567890
  预计送达：2024年3月15日 18:00
  当前位置：北京

最新物流动态：
  - 包裹已到达北京转运中心（3月14日 09:15）
  - 包裹已从广州仓库发出（3月13日 16:42）
  - 商品已打包（3月13日 10:30）"""
    },

    # 5. 模型基于工具结果生成最终回答
    # （这一步是 API 自动完成的，下面是模型可能生成的回答）
]

# 模型的最终回答：
# "您的订单目前正在配送中！包裹今天上午已到达北京转运中心，
#  预计明天（3月15日）下午6点前送达。
#  快递单号是 SF1234567890，您可以在顺丰官网或APP实时查看物流动态。
#  如果有其他问题随时告诉我～"
```

---

### 错误处理：工具调用失败时怎么办

工具不是万能的，调用可能失败。错误信息也需要精心设计格式：

```python
def format_tool_error(tool_name: str, error: Exception) -> str:
    """格式化工具调用错误，注入回 context"""

    error_map = {
        "OrderNotFound": {
            "message": "未找到该订单",
            "suggestion": "请确认订单号是否正确。订单号格式为 ORD-YYYYMMDD-NNN。"
        },
        "ServiceTimeout": {
            "message": "查询服务暂时无法响应",
            "suggestion": "系统繁忙，建议稍后重试。如果急需帮助，可以转接人工客服。"
        },
        "InvalidParameter": {
            "message": "参数格式不正确",
            "suggestion": "请检查输入参数。"
        },
    }

    error_type = type(error).__name__
    error_info = error_map.get(error_type, {
        "message": "工具调用失败",
        "suggestion": "请尝试其他方式帮助用户。"
    })

    return f"""### 工具调用失败
工具：{tool_name}
错误：{error_info['message']}
建议操作：{error_info['suggestion']}"""
```

关键原则：**不要把原始异常信息（堆栈跟踪、数据库错误码）注入到 context 中。** 模型看到 `ConnectionRefusedError: [Errno 111] Connection refused` 不知道该怎么办，而且可能把这些技术细节暴露给用户。

应该注入的是：(1) 发生了什么（人类可理解的错误描述）(2) 模型应该怎么做（明确的下一步建议）。

```
# 差的错误注入
"Error: psycopg2.OperationalError: connection to server at '10.0.1.5:5432' failed"

# 好的错误注入
"### 工具调用失败
工具：query_order
错误：未找到该订单
建议操作：请确认订单号是否正确。订单号格式为 ORD-YYYYMMDD-NNN。"
```

---

### 多步工具调用：一个工具的结果驱动下一个工具

很多真实场景需要连续调用多个工具：

```
用户："我想买点草莓，有货吗？"

第一步：模型调用 search_products(keyword="草莓")
→ 返回产品列表，其中丹东草莓的 product_id 是 SB002

第二步：模型调用 check_inventory(product_id="SB002")
→ 返回库存信息
```

这个"链式调用"在 context 中是这样呈现的：

```python
messages = [
    {"role": "system", "content": system_prompt},
    {"role": "user", "content": "我想买点草莓，有货吗？"},

    # 模型第一次响应：调用 search_products
    {
        "role": "assistant",
        "content": None,
        "tool_calls": [{
            "id": "call_001",
            "type": "function",
            "function": {
                "name": "search_products",
                "arguments": "{\"keyword\": \"草莓\"}"
            }
        }]
    },

    # search_products 的结果
    {
        "role": "tool",
        "tool_call_id": "call_001",
        "content": """### 搜索结果：草莓
找到 2 个相关产品：

1. 丹东99草莓
   - 产品ID：SB002
   - 规格：300g/盒
   - 售价：29.9 元
   - 产地：辽宁丹东

2. 进口白草莓
   - 产品ID：SB005
   - 规格：200g/盒
   - 售价：59.9 元
   - 产地：日本"""
    },

    # 模型第二次响应：调用 check_inventory（使用了上一步的 product_id）
    {
        "role": "assistant",
        "content": None,
        "tool_calls": [
            {
                "id": "call_002",
                "type": "function",
                "function": {
                    "name": "check_inventory",
                    "arguments": "{\"product_id\": \"SB002\"}"
                }
            },
            {
                "id": "call_003",
                "type": "function",
                "function": {
                    "name": "check_inventory",
                    "arguments": "{\"product_id\": \"SB005\"}"
                }
            }
        ]
    },

    # 两个 check_inventory 的结果
    {
        "role": "tool",
        "tool_call_id": "call_002",
        "content": """### 库存查询结果
产品：丹东99草莓（SB002）
库存状态：有货
可用库存：86 盒
发货仓库：北京仓（可次日达）"""
    },
    {
        "role": "tool",
        "tool_call_id": "call_003",
        "content": """### 库存查询结果
产品：进口白草莓（SB005）
库存状态：缺货
预计补货时间：2024年3月20日
备注：可预订，到货后优先发出"""
    },

    # 模型基于全部工具结果生成最终回答
]

# 模型的最终回答：
# "我们有两款草莓：
#  1. 丹东99草莓 300g/盒 29.9元 — 现在有货，北京仓发出可以次日达
#  2. 进口白草莓 200g/盒 59.9元 — 目前缺货，预计3月20日到货，可以先预订
#  您想要哪款呢？"
```

注意模型在第二步同时调用了两个 `check_inventory`（并行调用）。好的工具描述让模型能做出这种高效决策——如果描述中没有说明 `check_inventory` 只能查单个产品，模型可能会困惑是否应该一次传两个 ID。

---

### 引导工具选择：在 System Prompt 中给出工具使用策略

工具描述告诉模型"有什么工具"，但模型还需要知道"什么情况下用哪个工具"。这部分指引应该写在 system prompt 中：

```
## 可用工具使用指南

你有以下工具可以使用：

### query_order — 查订单
- 使用场景：用户问订单状态、物流进度、订单详情
- 前提条件：需要订单号。如果用户没有提供，请先询问
- 注意：一次只能查一个订单

### search_products — 搜产品
- 使用场景：用户想了解产品信息、比较价格、浏览商品
- 可以通过关键词、品类、价格范围搜索
- 搜索结果中包含 product_id，后续查库存时需要用到

### check_inventory — 查库存
- 使用场景：用户要下单或问"有没有货"
- 前提条件：需要 product_id（从 search_products 结果中获取）
- 如果用户问的是库存，通常需要先 search_products 再 check_inventory

### 工具使用原则
1. 能用工具查到真实数据就不要猜测
2. 如果不确定该用哪个工具，优先用 search_products（覆盖面最广）
3. 用户问价格 → search_products；问库存 → check_inventory
4. 工具调用失败时，向用户解释情况，不要重复调用同一个工具
```

---

## 为什么重要

### 场景一：差的工具描述导致错误调用

工具描述写得不清楚时，模型会犯各种错误：

```
# 描述太简略
"name": "check_inventory",
"description": "查库存"

用户问："你们有草莓吗？"

模型可能的错误行为：
→ 直接调用 check_inventory(product_id="草莓")
   错误：product_id 应该是 SB002 这样的 ID，不是产品名称
   但模型不知道，因为参数描述没说清楚格式

→ 或者模型根本不调用工具，直接回答"我们有草莓"
   因为 "查库存" 这个描述太模糊，模型不确定这是否是正确的工具
```

```
# 描述清晰详细
"name": "check_inventory",
"description": "查询指定产品的实时库存状态和预计补货时间。
当用户想购买某产品、或询问是否有货时使用。
注意：此工具只返回库存信息，不返回价格，价格请用 search_products 查询。"
"parameters": {
    "product_id": {
        "description": "产品ID，格式为字母+数字，例如 MG001、SB002。
        可以从 search_products 的返回结果中获取。"
    }
}

模型的正确行为：
→ 先调用 search_products(keyword="草莓") 获取产品列表和 product_id
→ 再调用 check_inventory(product_id="SB002") 查询库存
```

### 场景二：原始 JSON 导致信息泄露

```
用户："我的订单什么情况？"

如果直接把原始 JSON 注入 context，其中包含：
{
  "internal_notes": "优先配送客户，VIP等级3",
  "warehouse_id": "WH-GZ-001",
  "user_id": "USR-10042",
  "cost_price_cents": 2500
}

模型可能回答：
"您的订单正在从广州仓库（WH-GZ-001）发出，
作为VIP3级客户，您享受优先配送服务。
这批芒果的成本价是25元..."

→ 暴露了仓库ID、VIP等级、成本价——这些都不应该告诉用户
```

格式化后的结果只包含用户应该看到的信息，从根源上避免泄露。

### 场景三：多步调用中的失败传播

```
用户："帮我查一下 SB002 还有没有货，有的话顺便看看多少钱"

步骤1：check_inventory(product_id="SB002") → 成功，返回库存信息
步骤2：search_products(keyword="SB002") → 失败，服务超时

如果错误处理不好，模型可能：
"丹东草莓目前有货，价格是...呃...系统出了点问题，我帮你查到的。"
→ 回答不完整，体验很差

如果错误格式化得当：
步骤2 注入的 context：
"### 工具调用失败
工具：search_products
错误：查询服务暂时无法响应
建议操作：可以告知用户库存情况，价格信息稍后提供。"

模型回答：
"丹东草莓目前有货（库存86盒），可以次日达。
不过价格查询系统暂时有点繁忙，我稍后为您确认价格，
或者您可以在APP上直接查看。请问需要先预留一盒吗？"
→ 坦诚告知部分信息不可用，同时推进对话
```

---

## 设计启示

### 1. 工具描述是最容易被忽视的 Context Engineering

很多开发者花大量时间优化 system prompt，但对工具描述敷衍了事。实际上，工具描述的质量对模型行为的影响可能更大，因为它直接决定了：

- 模型会不会调用这个工具（触发条件）
- 模型传的参数对不对（参数理解）
- 模型会不会在不该调用时调用（误触发）

工具描述的投入产出比极高——花 10 分钟把描述写清楚，能避免大量线上错误。

### 2. 工具结果的格式化是一个"翻译层"

把工具结果格式化看作一个翻译过程：从"机器语言"（API JSON）翻译成"模型可用语言"（结构化文本）。

翻译原则：
- **去掉内部 ID 和编码**：模型不需要、用户不应该看到
- **转换单位和格式**：分 → 元，ISO时间 → 人类可读时间
- **只保留相关字段**：一个订单 API 可能返回 50 个字段，模型通常只需要其中 5-8 个
- **添加语义标签**：`status_code: 3` → `订单状态：配送中`

### 3. 为每个工具设计"失败路径"

不要只设计"调用成功"的 context，也要为"调用失败"的场景准备格式化方案。一个健壮的工具 context 设计应该覆盖：

```
成功情况 → 格式化的结果数据
查无结果 → "未找到相关信息" + 建议下一步操作
参数错误 → 提示正确的参数格式
服务超时 → 建议重试或替代方案
权限不足 → 解释原因，引导正确路径
```

### 4. 工具数量不是越多越好

每增加一个工具，就增加了 context 中的工具描述长度，也增加了模型选择错误工具的概率。经验法则：

- **3-5 个工具**：模型能可靠地选择正确工具
- **6-10 个工具**：需要在 system prompt 中添加工具选择指南
- **10+ 个工具**：考虑分组（比如用一个 "router" 工具先确定类别，再暴露该类别的具体工具）

对于「鲜果优选」客服助手，3 个核心工具（query_order、search_products、check_inventory）覆盖了 80% 的场景。

### 5. 并行调用 vs 顺序调用的 context 设计

如果多个工具可以并行调用（互相不依赖），在工具描述中暗示这种可能性：

```
# search_products 的描述中可以添加：
"注意：搜索结果中会包含 product_id，如果后续需要查库存，
可以使用 check_inventory 工具。如果需要查多个产品的库存，
可以同时发起多个 check_inventory 调用。"
```

这个提示让模型知道它可以并行调用多个 `check_inventory`，而不是一个一个地调。

---

## 常见误区

### 误区一："工具描述不太重要，模型自己能搞清楚"

**错误理解：** 只要工具名字起得好（比如 `query_order`），模型就知道什么时候用、怎么用。

**实际情况：** 模型对工具的理解 100% 来自你写的描述。名字只是一个提示，描述才是决策依据。

一个反例：工具名叫 `search_products`，但实际上它只搜索水果类产品，不搜索其他品类。如果描述中没有说明这个限制，用户问"你们卖坚果吗"时，模型会调用 `search_products(keyword="坚果")`，返回空结果，然后可能错误地告诉用户"我们没有坚果"——实际上只是这个工具搜不到而已。

### 误区二："直接返回 API 原始 JSON 就行"

**错误理解：** 模型能理解 JSON，不需要格式化。

**实际情况：** 模型确实能解析 JSON，但问题不在于"能不能读懂"，而在于：

1. **信息安全**：原始 JSON 几乎必然包含不应该暴露的内部字段
2. **Token 效率**：一个嵌套 5 层的 JSON 用了大量的花括号、引号、键名，实际信息密度很低
3. **注意力分配**：50 个字段中只有 5 个有用，模型需要在"噪音"中找到关键信息
4. **格式不匹配**：`price_cents: 3990` 直接被模型使用可能导致回答"3990元"

格式化不是"锦上添花"，是工程必须。

### 误区三："模型总是知道该用哪个工具"

**错误理解：** 给了模型工具列表，它就能在任何情况下正确选择。

**实际情况：** 模型在以下情况下容易选错工具或不选工具：

- **工具功能重叠**：如果 `search_products` 和 `check_inventory` 的描述都提到"查产品信息"，模型会困惑该用哪个
- **用户意图模糊**："芒果怎么样"——是问价格（search_products）、问库存（check_inventory）、还是问评价（可能没有对应工具）？
- **工具太多**：当有 15 个工具时，模型做选择的准确率会明显下降
- **缺少使用条件**：描述只说"查询订单"，没说"需要用户提供订单号"，模型可能在用户没给订单号时强行调用

解决方案：
- 在工具描述中明确区分各工具的适用场景（"查价格用A，查库存用B"）
- 在 system prompt 中给出工具选择的决策逻辑
- 用参数的 `required` 字段明确标注必填项
- 控制工具总数，必要时做工具分层

### 误区四："工具调用失败了重试就行"

**错误理解：** 工具失败时让模型再调一次。

**实际情况：** 盲目重试会导致：
- 重复调用一个已经宕机的服务，浪费时间和资源
- 用户等待时间加倍
- 如果是参数错误，用同样的参数重试必然再次失败

正确的做法是在错误 context 中告诉模型"不要重试"以及"应该怎么做"：

```
### 工具调用失败
工具：query_order
错误：订单系统暂时无法访问
建议操作：
- 不要重试此工具调用
- 告知用户系统暂时繁忙
- 建议用户稍后再试，或提供人工客服电话 400-XXX-XXXX
```

这样模型会优雅地处理失败，而不是陷入重试循环。

---

[⬆ 回到目录](00_知识地图.md) | [➡ 下一篇：输出格式约束](14_实战技术_输出格式约束.md)
