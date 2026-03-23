# Context Engineering 知识地图

> **目标读者：** 有 LLM 应用开发经验，Context Engineering 知识碎片化，需要系统性知识框架
> **本文档定位：** 全景概览，不展开细节。每个模块后续独立成文深入讨论

---

## 模块总览与学习路径

```
M1 基础概念 ──→ M2 Context 的组成结构 ──→ M3 设计原则 ──→ M4 工程化模式 ──→ M5 质量评估与调试
  (地基)          (认识原材料)            (组装心法)       (工程落地)        (质量保障)
```

**模块切分标准：**

| 模块 | 回答的核心问题 |
|------|---------------|
| M1 | Context Engineering **是什么**，边界在哪 |
| M2 | Context **由什么构成**（静态分类，认识原材料） |
| M3 | Context **该怎么设计**（原则与约束） |
| M4 | Context **运行时怎么组装**（动态工程实现） |
| M5 | Context **好不好、怎么改**（评估与调试） |

---

## M1 — 基础概念与设计思想

> **核心问题：Context Engineering 是什么，和 Prompt Engineering 什么关系，受什么约束**

| # | 知识点 | 重要度 | 说明 |
|---|--------|--------|------|
| 1.1 | Context 的定义：LLM 推理时能"看到"的一切信息的总和 | **Must Know** | 不只是 prompt，还包括检索结果、工具输出、对话历史等 |
| 1.2 | Context Engineering vs Prompt Engineering — 范畴差异与演进关系 | **Must Know** | CE 包含 PE，PE 是 CE 的子集 |
| 1.3 | Context Window 的物理约束（token 限制、注意力衰减、位置偏差） | **Must Know** | 所有设计决策的硬约束来源 |

**学习目标：** 建立"Context 是一个需要被工程化管理的运行时输入"的认知，而不是"写好 prompt 就行"。

---

## M2 — Context 的组成结构

> **核心问题：Context 由哪几种原材料构成（静态分类）**
> **注意：** 本模块只回答"有哪些"，不回答"怎么做"。每种原材料的工程实现在 M4 展开。

| # | 知识点 | 重要度 | 说明 |
|---|--------|--------|------|
| 2.1 | System Prompt — 角色、规则、行为边界 | **Must Know** | context 中最稳定的部分 |
| 2.2 | Few-shot Examples — 示例驱动行为校准 | **Must Know** | 用示例"教"模型比用指令"说"模型更可靠 |
| 2.3 | 检索增强上下文（RAG 注入的外部知识） | **Must Know** | 解决模型知识截止和私域知识缺失 |
| 2.4 | 对话历史（Conversation History） | **Must Know** | 多轮对话的连贯性基础 |
| 2.5 | 工具定义与工具调用结果（Tool Definitions & Results） | **Must Know** | Agent 场景的核心 context 来源 |
| 2.6 | 结构化元数据（用户画像、环境变量、时间戳等） | Nice to Know | 个性化与条件分支的输入 |
| 2.7 | 多模态上下文（图片、文件、音频的注入方式） | Nice to Know | 非纯文本场景的扩展 |

**学习目标：** 看到任何一个 LLM 应用，能拆解出它的 context 由哪几种原材料组成。

---

## M3 — 设计原则

> **核心问题：设计 context 时应遵循什么原则，什么样的 context 是"好的"**
> **定位：** 这是整份地图的判断力核心。学完这个模块，你就有语言描述"为什么这个 context 设计得不好"。

| # | 知识点 | 重要度 | 说明 |
|---|--------|--------|------|
| 3.1 | 信噪比原则 — 每个 token 都应有贡献 | **Must Know** | 冗余信息不是无害的，它会稀释关键信息的注意力权重 |
| 3.2 | 位置策略 — 信息放置影响注意力分配（首尾效应） | **Must Know** | 关键指令放开头或结尾，中间段容易被"遗忘" |
| 3.3 | 指令与知识分离 — 避免角色污染 | **Must Know** | System Prompt 管行为规则，检索内容管事实知识，不要混在一起 |
| 3.4 | 渐进式披露 — 按需注入而非全量堆砌 | **Must Know** | 不是信息越多越好，而是当前步骤需要的信息刚好够用 |
| 3.5 | 一致性原则 — context 内各部分不能自相矛盾 | **Must Know** | 矛盾的 context 导致模型行为随机化 |
| 3.6 | 安全性原则 — Prompt Injection 防护与信息泄漏防范 | **Must Know** | 生产环境的 P0 级风险，不是事后补丁 |
| 3.7 | 可测试性原则 — context 模板应参数化、边界条件可枚举 | **Must Know** | 事前设计：让 context 在部署前就能被系统性验证 |
| 3.8 | 可调试性原则 — context 应可观测、可复现 | **Must Know** | 事后诊断：出问题时能定位是 context 的哪个部分导致的 |
| 3.9 | 生产级 System Prompt 编写范式 | **Must Know** | M3 原则的直接实操体现，将原则落地为可执行的编写规范 |

**学习目标：** 面对任何 context 设计方案，能用这些原则做出"好/不好/怎么改"的判断。

---

## M4 — 工程化模式

> **核心问题：运行时如何组装和管理 context（动态工程实现）**
> **与 M2 的关系：** M2 告诉你有哪些原材料，M4 告诉你怎么在运行时把它们组装起来。

| # | 知识点 | 重要度 | 说明 |
|---|--------|--------|------|
| 4.1 | 动态 Context 组装（运行时拼接 vs 静态模板） | **Must Know** | 大多数生产系统的 context 不是写死的 |
| 4.2 | Context 压缩与摘要策略 | **Must Know** | 包含长文档处理模式（Map-Reduce / Refine） |
| 4.3 | RAG 管道设计（检索 → 排序 → 注入的完整链路） | **Must Know** | M2.3 的工程实现 |
| 4.4 | Memory 系统设计（短期/长期/工作记忆的分层） | **Must Know** | M2.4 的工程实现 |
| 4.5 | Multi-turn Context 管理（窗口滑动、截断策略） | **Must Know** | M2.4 的多轮场景工程实现 |
| 4.6 | 多步骤 Agent 的 Context 编排 | **Must Know** | Agent 场景下 context 如何跨步骤传递、累积、裁剪 |
| 4.7 | Context Caching（前缀缓存的工程价值与约束） | Nice to Know | 性能优化与成本控制手段 |
| 4.8 | 多 Agent 间的 Context 传递与隔离 | Nice to Know | 复杂 Agent 架构的进阶话题 |
| 4.9 | 多租户场景下的 Context 隔离与复用 | Nice to Know | SaaS 产品的特定需求 |

**学习目标：** 能为一个具体的 LLM 应用设计完整的 context 组装流程。

---

## M5 — 质量评估与调试

> **核心问题：怎么判断 context 的质量，出了问题怎么定位和修复**
> **与 M3 的关系：** M3 的可测试性/可调试性是事前设计原则，M5 是事后评估与诊断的具体方法。

| # | 知识点 | 重要度 | 说明 |
|---|--------|--------|------|
| 5.1 | Context 失效的典型症状识别 | **Must Know** | 幻觉、指令遗忘、格式错乱等背后的 context 归因 |
| 5.2 | Token 预算规划与监控 | **Must Know** | 把 token 当资源做容量规划 |
| 5.3 | Context 对输出质量的因果归因方法 | Nice to Know | 控制变量法：改 context 的一个部分看输出变化 |
| 5.4 | A/B 测试 Context 变体的实验框架 | Nice to Know | 规模化验证 context 改进效果 |
| 5.5 | Context 版本管理与 diff | Nice to Know | 像管理代码一样管理 context 的变更历史 |

**学习目标：** LLM 输出不符合预期时，能系统性地排查是 context 的问题还是模型的问题，并定位到具体环节。

---

## 下一步

各模块将独立成子文档深入讨论，文件命名规则：

```
Context_Engineering_M1_基础概念.md
Context_Engineering_M2_组成结构.md
Context_Engineering_M3_设计原则.md
Context_Engineering_M4_工程化模式.md
Context_Engineering_M5_质量评估与调试.md
```

逐模块讨论确认后再生成对应子文件。
