# SenseNova CLI 实战手册：Claude Code 配置 + Hermes 多模型调度方案

> SenseNova（日日新）平台接入 AI CLI 工具（Claude Code、Hermes Agent）的完整指南。
> 涵盖 API 配置、cc-switch 管理、以及面向 Hermes Agent 的模型分配方案。

---

## 目录

- [背景](#背景)
- [第一部分：Claude Code 接入 SenseNova](#第一部分claude-code-接入-sensenova)
  - [API 基础信息](#api-基础信息)
  - [⚠️ 关键陷阱：/v1 路径问题](#️-关键陷阱v1-路径问题)
  - [直接配置 Claude Code](#直接配置-claude-code)
  - [使用 cc-switch 管理配置（推荐）](#使用-cc-switch-管理配置推荐)
- [第二部分：Hermes 的 SenseNova 模型分配方案](#第二部分hermes-的-sensenova-模型分配方案)
  - [三款模型优劣势分析](#三款模型优劣势分析)
  - [任务分配方案](#任务分配方案)
  - [最终分配总览](#最终分配总览)
  - [核心分配逻辑](#核心分配逻辑)
- [常见问题](#常见问题)
- [致谢](#致谢)

---

## 背景

SenseNova（日日新）平台提供多款 AI 模型，通过统一的 OpenAI-compatible API 对外服务。本指南解决两个核心问题：

1. **Claude Code 如何接入 SenseNova** —— 包括直接配置和通过 cc-switch 管理
2. **Hermes Agent 如何在多个 SenseNova 模型之间分配任务** —— 每个模型各有优劣，如何根据能力和配额合理分配 9 个核心 Agent 任务

> Claude Code 部分基于实际使用验证，Hermes 分配方案基于模型特性分析 + 实际使用经验。

---

## 第一部分：Claude Code 接入 SenseNova

### API 基础信息

| 项目 | 值 |
|------|-----|
| Base URL（OpenAI 格式） | `https://token.sensenova.cn/v1` |
| Base URL（Claude Code 专用） | `https://token.sensenova.cn`（不带 `/v1`，原因见下文） |
| 认证方式 | Bearer Token (`Authorization: Bearer ***`) |
| 模型列表端点 | `https://token.sensenova.cn/v1/models` |

#### 可用模型

| 模型 ID | 说明 | 推荐用途 |
|---------|------|---------|
| `deepseek-v4-flash` | DeepSeek V4 Flash | Claude Code 主模型（推理能力强） |
| `sensenova-6.7-flash-lite` | SenseNova 6.7 Flash Lite | 轻量任务、降耗备选 |

### ⚠️ 关键陷阱：`/v1` 路径问题

**Claude Code 内部会自动在 `ANTHROPIC_BASE_URL` 后追加 `/v1/messages`。**

这意味着：

```
// ❌ 错误：实际请求变成 https://token.sensenova.cn/v1/v1/messages → 404
"ANTHROPIC_BASE_URL": "https://token.sensenova.cn/v1"

// ✅ 正确：Claude Code 自动补全为 https://token.sensenova.cn/v1/messages
"ANTHROPIC_BASE_URL": "https://token.sensenova.cn"
```

**其他工具（Hermes、Codex、OpenCode）** 使用 OpenAI-compatible API 直接调 `/v1/chat/completions`，不会自动追加路径段，因此它们的 base URL 需要包含 `/v1`。

### 直接配置 Claude Code

编辑 `~/.claude/settings.json`：

```json
{
  "providers": [
    {
      "name": "sensenova",
      "apiKey": "sk-你的API密钥",
      "baseUrl": "https://token.sensenova.cn"
    }
  ],
  "model": "deepseek-v4-flash"
}
```

或使用环境变量方式：

```json
{
  "env": {
    "ANTHROPIC_BASE_URL": "https://token.sensenova.cn",
    "ANTHROPIC_AUTH_TOKEN": "sk-你的API密钥",
    "ANTHROPIC_MODEL": "deepseek-v4-flash",
    "ANTHROPIC_DEFAULT_HAIKU_MODEL": "sensenova-6.7-flash-lite",
    "ANTHROPIC_DEFAULT_SONNET_MODEL": "deepseek-v4-flash",
    "ANTHROPIC_DEFAULT_OPUS_MODEL": "deepseek-v4-flash"
  }
}
```

验证：

```bash
claude -p "Say hi"
```

### 使用 cc-switch 管理配置（推荐）

[cc-switch](https://github.com/farion1231/cc-switch) 是一个跨平台桌面应用，可一站式管理所有 AI CLI 工具的 provider 配置。

#### 内置 Preset（PR #2559）

cc-switch 已内置 SenseNova provider preset（由 [@DreamEnding](https://github.com/DreamEnding) 贡献），支持 Claude Code、Codex、Hermes、OpenCode、OpenClaw 的一键配置。

![cc-switch SenseNova 配置界面](docs/sensenova-cc-switch-config.png)

> **注意**：Claude Code 的 `ANTHROPIC_BASE_URL` 在 preset 中已修正为 `https://token.sensenova.cn`（无 `/v1`），详见 [PR #2723](https://github.com/farion1231/cc-switch/pull/2723)。

#### 手动添加

1. 打开 cc-switch → 供应商管理
2. 点击"添加供应商"
3. 选择或手动配置 SenseNova
4. 填入 API Key
5. 保存并同步到各工具

---

## 第二部分：Hermes 的 SenseNova 模型分配方案

Hermes Agent 在运行时涉及多个不同类型的任务，包括多模态理解、文本推理、安全审批、会话检索等。SenseNova 平台提供了三款模型，每款能力、速度、配额各不相同——**需要根据模型特性进行合理的任务分配**，而不是让所有任务共用同一个模型。

### 三款模型优劣势分析

#### 模型对比总表

| | **6.7 Flash-Lite** | **U1 Fast** | **DeepSeek V4 Flash** |
|---|---|---|---|
| **核心能力** | 多模态理解 | 图像生成 | 深度推理 |
| **输入模态** | 图文 | **纯文本** | 纯文本 |
| **输出模态** | 文本 | **图像** | 文本 |
| **上下文** | 256K | 32K | 256K |
| **思考模式** | ❌ 不支持 | ❌ 不支持 | ✅ Think / No-think 可选 |
| **工具调用** | 基础 | 基础 | ✅ 强（JSON / Function Calling） |
| **配额 / 5h** | **1500** | 1500 | **150** |
| **速度** | 毫秒级 | ~15s/张图 | 中等 |
| **API 接口** | Chat Completions | **独立 `/v1/images/generations`** | Chat Completions |

#### 三款模型核心差异

**6.7 Flash-Lite** —— 多模态高频模型

| 优势 | 劣势 |
|------|------|
| 三款中**唯一支持图像输入** | 不支持思考/推理模式 |
| 256K 超长上下文 | 工具调用能力基础（非强 Function Calling） |
| **1500 次 / 5h** 高配额 | 不适合复杂推理任务 |
| **毫秒级**响应速度 | |

**DeepSeek V4 Flash** —— 推理决策模型

| 优势 | 劣势 |
|------|------|
| 三款中**推理能力最强** | **配额仅 150 次 / 5h**（Flash-Lite 的 1/10） |
| 支持 Think / No-think 模式 | 速度中等 |
| 强 JSON / Function Calling | 不支持图像输入 |
| 256K 上下文 | |

**U1 Fast** —— 图像生成模型

| 优势 | 劣势 |
|------|------|
| 专业图像生成 | **不支持图像输入**（不能做视觉理解） |
| 1500 次 / 5h 高配额 | 纯文本输入，文本输出弱 |
| 独立 images 接口 | 响应慢（~15s/张） |

#### 关键发现

1. **U1 Fast 不是 Chat 模型** —— 它用的是 `POST /v1/images/generations`，不是 Chat Completions，**不支持图像输入**，只管"出图"
2. **DeepSeek V4 Flash 能力最强但配额最稀缺** —— 150 次 / 5h 是 Flash-Lite 的 1/10，**不能上高频任务**
3. **Flash-Lite 是唯一多模态模型** —— 截图分析、OCR、图文理解只有它能干
4. **配额决定了分配上限** —— 高频任务必须给 Flash-Lite，低频高价值任务给 DeepSeek

### 任务分配方案

Hermes Agent 有 9 个核心任务需要分配模型。以下是基于**能力匹配 + 配额约束**的分配方案。

#### 分配给 6.7 Flash-Lite（多模态 · 高频 · 1500 配额）

Flash-Lite 承担 **6 个任务**，全部是高频或需要多模态能力的。

**① Vision / Image Analysis**

| 维度 | 分析 |
|------|------|
| 为什么是 Flash-Lite？ | **三款中唯一支持图像输入的模型**。截图分析、图表解读、OCR 识别只有它能完成 |
| 为什么不给 DeepSeek？ | DeepSeek 不支持图像输入，无法处理视觉任务 |
| 配额匹配 | 视觉分析使用频率中等，1500 配额绰绰有余 |

**② Web Extract / Page Summarization**

| 维度 | 分析 |
|------|------|
| 为什么是 Flash-Lite？ | 256K 上下文足够吞下长页面；网页常含图片/表格，Flash-Lite 的图文理解是刚需 |
| 为什么不给 DeepSeek？ | DeepSeek 也有 256K 上下文，但配额稀缺（150/5h），应该留给更高价值的推理任务 |
| 配额匹配 | 页面提取使用频率较高，1500 配额合理 |

**③ Compression / Context Compaction**

| 维度 | 分析 |
|------|------|
| 为什么是 Flash-Lite？ | **这是最高频的任务**——Agent 每隔几轮就要压缩一次上下文。需要**最快速度 + 最高配额**的模型 |
| 为什么不给 DeepSeek？ | 如果压缩任务用 DeepSeek，150 次配额在几轮对话内就打穿了，后续所有任务都无法运行 |
| 配额匹配 | 这是最关键的配额决策——压缩是频次最高的任务，**必须**用 1500 配额的模型 |

**⑤ Skills Hub / Skill Search**

| 维度 | 分析 |
|------|------|
| 为什么是 Flash-Lite？ | 轻量匹配任务：query → skill list，不需要深度推理，需要快速 + 低延迟 |
| 为什么不给 DeepSeek？ | 浪费——skill 搜索本质是向量匹配 + 简单排序，用推理模型是大材小用 |
| 配额匹配 | 每次 tool call 都会触发 skill 搜索，频次极高，必须用高配额模型 |

**⑦ MCP / MCP Tool Routing**

| 维度 | 分析 |
|------|------|
| 为什么是 Flash-Lite？ | 工具链自主编排是 Flash-Lite 的**五大核心能力之一**（官方文档明确列出）。MCP 路由是每次 tool call 都会触发的任务 |
| 为什么不给 DeepSeek？ | MCP 路由是**最高频的内部任务之一**，用 DeepSeek 的 150 配额瞬间打穿 |
| 配额匹配 | 工具路由是 Agent 的"心跳"任务，1500 配额是底线要求 |

**⑧ Title Gen / Session Titles**

| 维度 | 分析 |
|------|------|
| 为什么是 Flash-Lite？ | 纯文本生成，逻辑极简单——"给这段对话起个标题" |
| 为什么不给 DeepSeek？ | 用推理模型做标题生成是配额浪费，好比用超级计算机做加减法 |
| 配额匹配 | 每次新会话触发一次，中等频率，Flash-Lite 完全胜任 |

#### 分配给 DeepSeek V4 Flash（推理 · 低频高价值 · 150 配额）

DeepSeek 承担 **3 个任务**，全部是**安全关键或需要深度推理的**。

**④ Session Search / Recall Queries**

| 维度 | 分析 |
|------|------|
| 为什么是 DeepSeek？ | 需要在长对话中做语义检索 + 相关性判断。256K 上下文 + Think 模式能更准确地匹配用户意图 |
| 为什么不给 Flash-Lite？ | 语义相关性判断需要推理能力，Flash-Lite 没有 Think 模式，匹配精度不够 |
| 配额匹配 | Session Search 不是每次对话都触发，低频使用，150 配额可以支撑 |

**⑥ Approval / Smart Auto-Approve**

| 维度 | 分析 |
|------|------|
| 为什么是 DeepSeek？ | **自动审批是安全关键路径**，错误成本极高——错误批准可能导致文件删除、配置修改、数据泄露 |
| 为什么不给 Flash-Lite？ | Flash-Lite 没有 Think 模式，无法做深度的风险评估和安全判断 |
| 配额匹配 | 审批触发不密集，150 配额虽然少但够用。**安全不能图快** |

**⑨ Curator / Skill-Usage Review**

| 维度 | 分析 |
|------|------|
| 为什么是 DeepSeek？ | 需要通读整个会话、识别使用模式、发现改进点——本质是**深度分析任务** |
| 为什么不给 Flash-Lite？ | Curator 需要跨会话的模式识别和洞察提取，Flash-Lite 的推理能力不足以胜任 |
| 配额匹配 | 定期触发（每小时/每天），低频任务，适合 DeepSeek 的慢消费节奏 |

#### U1 Fast 的定位

U1 Fast 是**图像生成模型**，与上述 9 个 Agent 内部任务不直接匹配。它属于**输出层能力**——锦上添花，不是 Agent 运转必需。

适合的场景：

> Agent 产出可视化报告时，把分析结果渲染成**信息图 / 排版图文**发给用户。

例如 Curator 审查完 skill 使用情况后，用 U1 Fast 生成一张"Skill 使用健康度仪表盘"发到群聊。这是增强用户体验的能力，不是 Agent 内部必需的。

### 最终分配总览

```
6.7 Flash-Lite（6 个任务）          DeepSeek V4 Flash（3 个任务）
├─ ① Vision / Image Analysis        ├─ ④ Session Search
├─ ② Web Extract / Summarization    ├─ ⑥ Approval / Auto-Approve
├─ ③ Compression / Context           └─ ⑨ Curator / Skill Review
├─ ⑤ Skills Hub / Skill Search
├─ ⑦ MCP / Tool Routing
└─ ⑧ Title Gen / Session Titles

U1 Fast（按需）
└─ 可视化报告输出（输出层，非 Agent 内部任务）
```

#### 按频次分布

```
高频任务 ──────→ 中频任务 ──────→ 低频任务
  ③ Compression    ② Web Extract    ④ Session Search
  ⑤ Skill Search   ① Vision         ⑥ Approval
  ⑦ MCP Routing    ⑧ Title Gen      ⑨ Curator
  ────── Flash-Lite ──────            ── DeepSeek ──
```

高频任务全部由 Flash-Lite 承载，低频高价值任务给 DeepSeek——这是配额约束下的最优分配。

### 核心分配逻辑

**Flash-Lite 扛高频 + 多模态，DeepSeek 守关键决策路径，U1 做锦上添花的可视化输出。**

三条决策原则：

1. **能力匹配优先** —— 多模态任务（Vision）只能给 Flash-Lite，图像生成只能给 U1 Fast。这是硬约束
2. **配额约束决定上限** —— DeepSeek 的 150 次 / 5h 配额决定了它不能承担任何高频任务。MCP 路由如果给 DeepSeek，在一轮对话中就能耗尽全部配额
3. **安全关键任务用强推理模型** —— 自动审批的误判成本远高于速度成本，必须用支持 Think 模式的 DeepSeek，不能用 Flash-Lite

---

## 常见问题

### Q: 为什么 Claude Code 的 base URL 不能带 `/v1`？

Claude Code SDK 内部自动在 `ANTHROPIC_BASE_URL` 后追加 `/v1/messages`。如果 base URL 已经是 `.../v1`，最终请求路径变成 `.../v1/v1/messages`，导致 404。

### Q: Hermes 为什么可以用带 `/v1` 的 base URL？

Hermes 使用 OpenAI-compatible API，直接调用 `/v1/chat/completions`，不会自动追加路径段。所以 base URL 需要包含 `/v1`。

### Q: 为什么 DeepSeek V4 Flash 不能承担高频任务？

配额限制：DeepSeek 的配额是 **150 次 / 5 小时**。一个高频任务（如 Compression）在一轮对话中可能触发 10-20 次，如果用 DeepSeek，几轮对话就耗尽了全部配额，导致后续所有任务无法运行。

### Q: U1 Fast 能替代 Flash-Lite 做视觉理解吗？

**不能。** U1 Fast 是图像**生成**模型，不支持图像输入（只接受文本 prompt），不能做截图分析、OCR、图表解读。它是 `POST /v1/images/generations` 接口，不是 Chat Completions。

### Q: 如何查看 API 余量？

SenseNova 的余量查询需要 OAuth2 token（从浏览器 LevelDB 提取），API Key 本身不可查余量。

### Q: settings.json 被 cc-switch 还原了怎么办？

cc-switch 有 settings.json 文件监听器，外部直接修改 `~/.claude/settings.json` 会被覆盖。**必须从 cc-switch UI 内修改**。

---

## 致谢

- [@DreamEnding](https://github.com/DreamEnding) — 提交 PR #2559 为 cc-switch 添加 SenseNova 内置支持
- [@farion1231](https://github.com/farion1231) — 维护 cc-switch 开源项目
- [SenseNova（日日新）](https://platform.sensenova.cn) — AI API 服务

## 许可证

MIT