# SenseNova 模型对比与任务分配

> 商汤日日新 (SenseNova) 平台三款模型的横向对比，以及面向 AI CLI Agent 的任务分配方案。
> 基于实际使用经验整理，2026-05 版。

---

## 三款模型关键能力定调

| | **6.7 Flash-Lite** | **U1 Fast** | **DeepSeek V4 Flash** |
|---|---|---|---|
| **核心能力** | 多模态理解 | 图像生成 | 深度推理 |
| **输入模态** | 图文 | **纯文本** | 纯文本 |
| **输出模态** | 文本 | **图像** | 文本 |
| **上下文** | 256K | 32K | 256K |
| **思考模式** | ❌ | ❌ | ✅ Think/No-think |
| **工具调用** | 基础 | 基础 | ✅ 强（JSON/Function Calling） |
| **配额 / 5h** | 1500 | 1500 | **150** |
| **速度** | 毫秒级 | ~15s/张图 | 中等 |
| **API 接口** | Chat Completions | **独立 images 接口** | Chat Completions |

### 关键发现

- **U1 Fast** 用的是 `POST /v1/images/generations`，不是 Chat Completions，且**不支持图像输入**，只管"出图"
- **DeepSeek V4 Flash** 能力最强，但配额只有 **150 次 / 5 小时**，是另外两款的 1/10
- **Flash-Lite** 是唯一支持图像输入的模型

---

## 9 个任务分配

### 分给 6.7 Flash-Lite（多模态 · 高频 · 1500 配额）

| # | 任务 | 理由 |
|---|---|---|
| **1** | Vision / Image analysis | **三款中唯一支持图像输入**，截图/图表/OCR 只有它能干。原生多模态架构不走"视觉→文本"转译，理解质量高 |
| **2** | Web Extract / Page summarization | 256K 上下文够吞长页面；网页常含图片/表格，Flash-Lite 的图文理解是刚需；配额充足支撑中等频率 |
| **3** | Compression / Context compaction | 最高频任务——每隔几轮就要压缩一次。必须用配额最足 + 速度最快的模型，Flash-Lite 毫秒级响应 + 1500 配额 + 60% Token 降耗是唯一合理选择 |
| **5** | Skills Hub / Skill search | 轻量匹配任务，query → skill list，不需要深度推理，需要快速 + 低延迟，完美适配 |
| **7** | MCP / MCP tool routing | 工具链自主编排是 Flash-Lite **五大核心能力之一**（官方原话）。且 MCP 路由是高频任务——每次 tool call 都触发，1500 配额才能扛住 |
| **8** | Title Gen / Session titles | 纯文本生成，极简单，用 DeepSeek 是浪费配额 |

### 分给 DeepSeek V4 Flash（推理 · 低频高价值 · 150 配额）

| # | 任务 | 理由 |
|---|---|---|
| **4** | Session Search / Recall queries | 需要在长对话中做语义检索 + 相关性判断，256K 上下文 + Think 模式能更准确地匹配用户意图 |
| **6** | Approval / Smart auto-approve | 自动审批是**安全关键路径**，错误成本极高。必须用 Think 模式做风险评估，不能图快。配额虽少但审批通常不会密集触发 |
| **9** | Curator / Skill-usage review | 需要通读整个会话、识别模式、发现改进点——本质是深度分析任务。低频（定期触发），适合配额的慢消费节奏 |

### U1 Fast 的定位

U1 Fast 是**图像生成模型**，与上述 9 个任务不直接匹配。它适合的场景是：

> Agent 产出可视化报告时，把分析结果渲染成**信息图 / 排版图文**发给用户。

比如 Curator 审查完 skill 使用情况后，用 U1 Fast 生成一张"Skill 使用健康度仪表盘"。这是"锦上添花"的输出层能力，不是 Agent 内部运转必需的环节。

---

## 最终分配总览

```
6.7 Flash-Lite (6个)          DeepSeek V4 Flash (3个)
├─ ① Vision / Image           ├─ ④ Session Search
├─ ② Web Extract              ├─ ⑥ Approval
├─ ③ Compression              └─ ⑨ Curator
├─ ⑤ Skill Search
├─ ⑦ MCP Routing
└─ ⑧ Title Gen

U1 Fast (按需)
└─ 可视化报告输出（输出层，非 Agent 内部任务）
```

### 核心逻辑

**Flash-Lite 扛高频 + 多模态，DeepSeek 守关键决策路径，U1 做锦上添花的可视化输出。**

配额约束决定了 DeepSeek 不能上高频任务（150 次 / 5 小时跑 MCP 路由瞬间打穿），而 Flash-Lite 的 1500 配额 + 毫秒延迟天然适合。

---

## 相关链接

- [cc-switch PR #2559](https://github.com/farion1231/cc-switch/pull/2559) — SenseNova provider preset
- [SenseNova 平台](https://platform.sensenova.cn)
- [本仓库配置指南](../README.md)