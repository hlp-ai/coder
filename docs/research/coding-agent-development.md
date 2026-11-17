# 编码智能体开发研究报告

> **范围与方法**：本文讨论“编码智能体”（coding agent）——由大语言模型（Large Language Model, LLM）驱动、能在真实代码仓库中读取文件、修改代码、执行命令并根据环境反馈迭代的软件系统。目标是给出与具体语言、框架无关的概念模型与工程决策依据，而非某个产品的使用教程。文中结论尽量引用一手资料（论文、官方文档、官方工程博客、开源项目仓库）；无法从一手资料核实的说法以“（未验证）”标注。具体产品仅作为证据或示例出现，不构成推荐。

---

## 1. 整体架构总览

### 1.1 什么是编码智能体

业界对“智能体”没有统一定义，但一个被广泛采用的工程区分是：**工作流（workflow）** 由预定义代码路径编排 LLM 与工具，**智能体（agent）** 则由 LLM 动态决定流程与工具使用方式（[Anthropic, Building effective agents](https://www.anthropic.com/engineering/building-effective-agents)）。编码智能体属于后者，其最简形式可以概括为“LLM 在循环中自主使用工具”（[Anthropic, Effective context engineering for AI agents](https://www.anthropic.com/engineering/effective-context-engineering-for-ai-agents)）。

Anthropic 在同一篇文章中指出，智能体的实现往往“只是一个在循环中基于环境反馈使用工具的 LLM”，因此**工具集（toolset）及其文档的设计质量**是成败关键，而非框架复杂度。论文层面的早期代表是 ReAct，它把“推理轨迹”和“动作”交错生成，让模型在每一步同时输出思考与可执行动作（[ReAct, arXiv:2210.03629](https://arxiv.org/abs/2210.03629)）。

### 1.2 参考架构

下图给出一个与语言无关的参考架构。它把系统拆成“入口 / 编排 / 循环 / 工具运行时 / 环境与安全边界”五层，外加横切能力。

```text
+------------------------------------------------------------------+
|                        入口层 (Entry)                            |
|   CLI · IDE 插件 · HTTP/Web API · SDK · 无人值守任务队列         |
+-------------------------------+----------------------------------+
                                |
+-------------------------------v----------------------------------+
|                    编排器 (Orchestrator)                         |
|   会话/任务管理 · 规划 · 停止条件 · 预算控制 · 子智能体调度      |
+-------------------------------+----------------------------------+
                                |
+-------------------------------v----------------------------------+
|                   智能体循环 (Agent Loop)                        |
|                                                                  |
|   ┌──────────────┐    ┌──────────────┐    ┌──────────────┐       |
|   │ 组装上下文   │───>│  LLM 推理    │───>│  解析动作    │       |
|   │ (context)    │    │ (inference)  │    │ (tool call)  │       |
|   └──────┬───────┘    └──────────────┘    └──────┬───────┘       |
|          ^                                       │               |
|          │            ┌──────────────┐           │               |
|          └────────────│  观察结果    │<──────────┘               |
|                       │ (observation)│                           |
|                       └──────────────┘                           |
+-------------------------------+----------------------------------+
                                |
+-------------------------------v----------------------------------+
|                   工具运行时 (Tool Runtime)                      |
|   文件读写 · Shell/进程 · 代码搜索 · 测试/构建 · MCP 客户端      |
|   LSP/静态分析 · 网络请求 · 任务/待办 · 记忆存储                 |
+-------------------------------+----------------------------------+
                                |
+-------------------------------v----------------------------------+
|              环境与安全边界 (Environment & Guard)                |
|   代码仓库/工作区 · 沙箱/容器 · 权限策略 · 密钥隔离 · 审计日志    |
+------------------------------------------------------------------+

横切能力 (cross-cutting)：上下文管理 · 提示缓存 · 追踪(tracing) · 评测(eval) · 可观测性
```

这一分层并非唯一答案，但几乎每个真实系统（开源或商业）都能映射到其中。例如 SWE-agent、OpenHands、Aider、opencode 都包含“循环 + 工具 + 仓库环境”三要素，差异主要在工具的抽象层次与安全边界。

### 1.3 与通用聊天机器人的关键差异

| 维度 | 聊天机器人 | 编码智能体 |
| --- | --- | --- |
| 输入 | 单条或多轮对话 | 任务描述 + 可访问的代码仓库与环境 |
| 输出 | 文本 | 对仓库的改动 + 命令执行 + 验证结果 |
| 反馈来源 | 用户回复 | 文件系统、编译器、测试、命令退出码 |
| 成功标准 | 主观满意度 | 可验证：测试通过、构建成功、缺陷修复 |
| 主要风险 | 不当内容 | 破坏性命令、数据外泄、供应链攻击 |
| 上下文 | 短、以对话为主 | 长、以仓库与工具输出为主，易膨胀 |

“可验证”是编码智能体的巨大优势：Anthropic 指出编程任务之所以适合智能体，是因为代码正确性可由自动化测试验证、模型能借助测试结果迭代、问题空间结构化（[Building effective agents](https://www.anthropic.com/engineering/building-effective-agents)）。这一点直接决定了评测与安全设计（见第 4 节）。

### 1.4 最小组件清单

一个能工作的原型至少需要：

1. **模型接入层**：统一的 LLM 调用接口，支持工具调用（tool calling）与流式输出。
2. **循环控制器**：负责“组装上下文 → 推理 → 执行动作 → 回填观察”的迭代，以及停止条件与预算。
3. **工具集**：读文件、写/改文件、列目录、搜索、执行命令（通常是最危险也最重要的一个）。
4. **上下文装配器**：系统提示、工具定义、项目说明、历史消息、检索到的代码。
5. **环境边界**：至少是工作目录限定；生产环境需要沙箱与权限策略。
6. **日志/追踪**：记录每一步的输入、动作与结果，否则无法调试与评测。

---

## 2. Agent 循环与工具调用

### 2.1 循环的三种典型形态

- **ReAct 式交错**：模型在每一步生成“思考 + 动作”，系统执行动作并把观察结果拼回上下文（[ReAct](https://arxiv.org/abs/2210.03629)）。优点是推理与行动紧密结合、可解释性较好；缺点是动作空间通常是离散的文本命令，表达力受限。
- **代码即动作（CodeAct）**：用可执行的代码（通常是 Python）作为统一动作空间，把工具调用、数据处理、控制流都写进一段代码里执行（[CodeAct, arXiv:2402.01030](https://arxiv.org/abs/2402.01030)）。优点是表达力强、可组合；缺点是执行环境更复杂、沙箱要求更高。
- **结构化工具调用（function/tool calling）**：模型输出符合 JSON Schema 的工具名与参数，由宿主执行（见 2.2）。这是当前主流产品最常用的形态，也是 ReAct 思想在工程上的标准化落地。

三者并不互斥：现代编码智能体常常是“结构化工具调用 + 交错推理（可选的思维链/计划）+ 代码执行工具”。

### 2.2 工具调用协议与模式

**历史起点**：OpenAI 于 2023 年 6 月发布函数调用（function calling），模型可根据用户输入决定调用哪个函数并输出符合签名的 JSON 参数，使“模型选择工具”成为标准能力（[OpenAI, Function calling and other API updates](https://openai.com/index/function-calling-and-other-api-updates/)）。更早的 Toolformer 已经展示了让模型自监督地学会何时调用哪个 API 的思路（[Toolformer, arXiv:2302.04761](https://arxiv.org/abs/2302.04761)），但函数调用把它变成了可广泛使用的产品接口。

**标准化**：模型上下文协议（Model Context Protocol, MCP）把“工具发现与调用”抽象成与模型厂商无关的协议，基于 JSON-RPC 2.0。其工具规范定义了：

- `tools/list` 发现工具，`tools/call` 调用工具；
- 每个工具包含 `name`、`description`、`inputSchema`（JSON Schema），可选 `outputSchema`；
- 结果分为非结构化 `content` 与结构化 `structuredContent`，执行错误通过 `isError: true` 表达；
- 工具是**模型控制（model-controlled）** 的，但规范明确要求“应当（SHOULD）始终有一个人工介入点，能够拒绝工具调用”，并建议客户端展示工具、提示确认、校验结果（[MCP Specification 2025-06-18, Tools](https://modelcontextprotocol.io/specification/2025-06-18/server/tools)）。

MCP 的价值在于**解耦**：工具由独立服务器实现，宿主（agent）只需实现一次客户端。这直接影响了架构设计——工具运行时可以是进程内的，也可以是远程 MCP 服务器。

**参数与返回值的工程约束**：

- 输入必须能被模型稳定生成。JSON Schema 应严格、明确，避免需要模型做复杂转义或计数。
- 输出要**token 高效**：对超大文件、长日志、命令输出做截断、摘要或分页（[Effective context engineering](https://www.anthropic.com/engineering/effective-context-engineering-for-ai-agents) 强调工具应返回“token 高效”的信息）。
- 错误要**结构化且可恢复**：区分协议错误（参数非法、未知工具）与执行错误（命令失败、权限拒绝），让模型能据此调整策略。

### 2.3 智能体-计算机接口（ACI）

Anthropic 把智能体与工具之间的契约称为“智能体-计算机接口（Agent-Computer Interface, ACI）”，并主张像对待人机界面（HCI）一样投入设计（[Building effective agents, Appendix 2](https://www.anthropic.com/engineering/building-effective-agents)）。其核心建议可直接落到编码智能体：

- **给模型留出“思考”的 token**，不要在工具格式上强迫它做高难度的格式操作。
- **格式贴近自然文本**：例如写完整文件通常比写精确的 diff 更容易；在 JSON 里写代码需要额外转义，是额外负担。
- **避免格式开销**：不要让模型维护精确的行号计数或做大规模字符串转义。
- **参数名与描述要显式**：把它们当成给新同事写的 docstring，尤其在工具很多且相似时。
- **Poka-yoke（防呆）**：通过参数设计让错误难以发生。Anthropic 在构建 SWE-bench 智能体时发现模型在离开根目录后会搞错相对路径，于是把工具改为**只接受绝对路径**，问题随即消失（同上）。
- **实际测试工具的用法**：用大量样例输入观察模型如何误用，再迭代。

这些结论对“编辑工具”尤其重要。文件编辑至少有三种实现：整文件重写、搜索/替换补丁、统一 diff。它们对模型的难度差异很大，Aider 项目就明确支持多种编辑格式（edit format），并针对不同模型选择不同格式（[Aider 仓库](https://github.com/Aider-AI/aider)）。工程上应把“编辑格式”当作一等设计对象，而不是实现细节。

### 2.4 循环控制与容错

一个健壮的循环需要显式的控制面：

- **迭代上限**：防止无限循环与成本失控。Anthropic 明确提到智能体常需要“最大迭代次数”等停止条件（[Building effective agents](https://www.anthropic.com/engineering/building-effective-agents)）。
- **预算**：token、时间、工具调用次数、费用。超预算时应优雅停止并汇报进度，而非硬截断。
- **终止条件**：任务完成（模型声明 + 验证通过）、无法继续（需要人工输入）、达到上限。
- **错误恢复**：命令失败、测试失败、权限拒绝都应作为“观察”回填，让模型自我修正。Reflexion 表明“语言化的自我反思 + 记忆”能提升后续尝试的表现（[Reflexion, arXiv:2303.11366](https://arxiv.org/abs/2303.11366)）。
- **验证环节**：编码任务的优势是可用测试验证。一个成熟循环通常在改动后运行相关测试/构建，把结果作为下一步输入。
- **幂等与副作用**：写文件、执行命令可能有副作用。重试要有边界，危险的不可逆操作应先确认（见第 4 节）。

### 2.5 工具集的最小可用集合

对编码智能体而言，经验上最小且有表达力的集合是：

| 工具 | 作用 | 关键设计点 |
| --- | --- | --- |
| 读文件 | 获取内容 | 支持行范围、大文件截断提示 |
| 列目录/glob | 探索结构 | 输出应简短、可排序 |
| 文本搜索（grep） | 定位符号/字符串 | 返回文件+行号，限制结果数 |
| 写文件 | 新建/覆盖 | 明确路径语义（绝对/相对） |
| 编辑文件 | 局部修改 | 选择模型易用的补丁格式 |
| 执行命令 | 测试/构建/git | 最危险，必须限权、限时、限输出 |
| 任务/待办 | 长任务进度 | 可显著改善长程一致性 |

Aider 的“repo map（仓库地图）”与自动提交、lint/test 集成说明了另一类能力：**项目级感知与自动化验证**（[Aider 仓库](https://github.com/Aider-AI/aider)、[Aider repo map 文档](https://aider.chat/docs/repomap.html)）。这些能力在下一节的上下文管理中展开。

---

## 3. 上下文与 Harness 设计

### 3.1 为什么“上下文工程”是核心

编码智能体的上下文（context）由系统提示、工具定义、项目说明、代码片段、历史消息和工具输出共同构成。Anthropic 指出，随着上下文 token 数增加，模型对其中信息的准确召回会下降，即所谓“上下文腐化（context rot）”；上下文因此应被视为**有限且边际收益递减的资源**，工程目标是“找到能最大化目标结果的最小高信号 token 集合”（[Effective context engineering](https://www.anthropic.com/engineering/effective-context-engineering-for-ai-agents)）。

由此引出本文对 **Harness** 的定义：**模型之外、围绕模型构建的整套运行时脚手架**，包括上下文装配、工具定义、检索策略、缓存、压缩、循环控制与安全边界。同一个模型，换一套 harness，表现可能差异巨大——这也是评测必须绑定 harness 的原因（见 4.1）。

### 3.2 上下文的组成与装配策略

| 组成 | 内容 | 装配策略 |
| --- | --- | --- |
| 系统提示 | 角色、约束、工作方式、输出约定 | 分段（背景/指令/工具指引/输出格式），用 Markdown 或 XML 标签分隔；保持“正确高度”，既不硬编码脆弱逻辑，也不空泛（[同上](https://www.anthropic.com/engineering/effective-context-engineering-for-ai-agents)） |
| 工具定义 | 名称、描述、schema | 精简、无重叠、示例清晰；工具集膨胀会显著增加歧义 |
| 项目说明 | 构建/测试命令、约定、架构 | 启动时直接注入。Claude Code 采用 `CLAUDE.md` 前置注入；本项目可用 `AGENTS.md` |
| 代码上下文 | 相关文件、符号、仓库结构 | 预索引（repo map）或即时检索（grep/glob/read） |
| 历史消息 | 之前的推理与工具往返 | 压缩、清理旧工具结果、必要时摘要 |
| 工具输出 | 命令/测试结果 | 截断、摘要、结构化 |

一个重要的工程取舍是**预索引 vs 即时检索（just-in-time）**：

- **预索引**：启动或后台构建仓库地图/向量索引，把最相关片段放进上下文。Aider 的 repo map 用语法树与图排序在 token 预算内挑选关键定义与签名，是这一路线的代表（[Aider repo map 文档](https://aider.chat/docs/repomap.html)）。
- **即时检索**：只维护轻量引用（路径、查询），让模型用 grep/glob/read 在运行时按需加载。Anthropic 认为这种方式更贴近人类使用文件系统的方式，能避免索引陈旧，并支持“渐进式披露（progressive disclosure）”（[Effective context engineering](https://www.anthropic.com/engineering/effective-context-engineering-for-ai-agents)）。
- **混合**：Claude Code 采用混合策略——`CLAUDE.md` 预先注入，文件内容则通过 glob/grep 即时获取（同上）。

对多数编码智能体，推荐**以即时检索为默认、以仓库地图为加速手段**：先给模型“地图”和搜索工具，让它自己决定读什么，而不是一次性塞入大量代码。

### 3.3 长任务的上下文管理

当任务跨越长时间、token 超过窗口时，需要专门技术（均见 [Effective context engineering](https://www.anthropic.com/engineering/effective-context-engineering-for-ai-agents)）：

- **压缩（compaction）**：接近上限时把历史摘要化，用摘要 + 最近访问的文件重开窗口。关键是保留架构决策、未解决的缺陷与实现细节，丢弃冗余工具输出。
- **工具结果清理**：较早的工具原始输出通常可以删除，这是“最轻量”的压缩形式。
- **结构化笔记（structured note-taking）**：让智能体把进度写入外部文件（如待办、`NOTES.md`），在后续轮次读回，形成跨窗口的持久记忆。
- **子智能体（sub-agent）**：把探索性工作交给拥有干净上下文的子智能体，只回传精炼摘要（详见第 5 节）。

### 3.4 提示缓存与成本

长系统提示与稳定工具定义会在每一轮重复发送。提示缓存（prompt caching）通过复用相同前缀的计算来降低延迟与成本（[Anthropic, Prompt caching](https://www.anthropic.com/news/prompt-caching)）。对编码智能体的实践含义是：

- 把**稳定内容放在前缀**（系统提示、工具定义、项目说明），把**易变内容放在后面**（新消息、工具输出）。
- 避免无谓地改写系统提示或工具定义，否则破坏缓存命中。
- 缓存与上下文压缩要协同：摘要会改变前缀，需权衡缓存收益与上下文长度。

### 3.5 Harness 的可调试性

Anthropic 的核心原则之一是“透明性”：显式展示智能体的规划步骤（[Building effective agents](https://www.anthropic.com/engineering/building-effective-agents)）。落到 harness：

- 每一步都记录完整的模型输入、输出、工具调用与结果（tracing）。
- 提供可回放（replay）能力，便于复现失败。
- 把“模型决策”和“宿主逻辑”分离，避免把业务规则硬编码进提示而难以定位问题。

OpenAI Agents SDK 把追踪（tracing）作为内建能力，并区分工具、护栏（guardrails）、交接（handoffs）、会话（sessions）等概念（[OpenAI Agents SDK 文档](https://openai.github.io/openai-agents-python/)）；这类抽象可作为设计参考。

---

## 4. 评测与安全

### 4.1 评测：基准、指标与陷阱

**代码生成基准**：

- **HumanEval** 提供 164 个手写编程问题，以函数级单元测试判定正确性，常用 `pass@k` 指标（[HumanEval, arXiv:2107.03374](https://arxiv.org/abs/2107.03374)）。它衡量的是“从零写函数”，与真实仓库修改差距较大。
- **SWE-bench** 从真实 GitHub issue 构造任务，要求智能体在真实仓库中生成补丁并让测试通过，包含 2,294 个任务实例（[SWE-bench, arXiv:2310.06770](https://arxiv.org/abs/2310.06770)）。它更接近编码智能体的真实工作负载。
- **SWE-bench Verified** 是 OpenAI 与人工标注者合作筛出的 500 个“人类确认可解”样本子集，用于降低基准噪声（[OpenAI, Introducing SWE-bench Verified](https://openai.com/index/introducing-swe-bench-verified/)）。

**基准的“harness 敏感性”**：

- **Agentless** 表明，一个不含自主循环的固定流水线（定位 → 修复 → 验证）也能在 SWE-bench 上取得有竞争力的结果（[Agentless, arXiv:2407.01489](https://arxiv.org/abs/2407.01489)）。这说明分数并不只反映“智能体能力”，也反映工程流水线。
- **SWE-agent** 强调“智能体-计算机接口”本身对结果有显著影响（[SWE-agent, arXiv:2405.15793](https://arxiv.org/abs/2405.15793)）。
- **mini-SWE-agent** 用约 100 行代码实现极简智能体，其项目 README 声称在 SWE-bench Verified 上超过 74%（[mini-SWE-agent 仓库](https://github.com/SWE-agent/mini-swe-agent)）；SWE-agent 的 README 亦声称其成绩为 65%（[SWE-agent 仓库](https://github.com/SWE-agent/SWE-agent)）。这些为项目自述数据，横向比较时需注意 harness 与评测配置差异。

**结论**：评测必须**绑定你自己的 harness**，不能直接引用他人“模型分数”推断你的系统表现。应自建与目标场景一致的任务集，并把成本、延迟、步数一并记录。

**评测设计要点**（参考 [Anthropic, Demystifying evals for AI agents](https://www.anthropic.com/engineering/demystifying-evals-for-ai-agents)）：

- 用**可执行的判定**（测试、构建、静态检查）作为主信号，LLM 打分只作补充且需校准。
- 覆盖**过程指标**：工具调用次数、无效循环、是否读取了无关文件、是否触发权限拒绝。
- 建立**回归集**：每次修改提示、工具或循环逻辑都跑同一套任务。
- 警惕**数据污染**与过拟合：任务与仓库应定期更新，避免模型“记住”答案。
- 区分**能力评测**与**产品评测**：前者看上限，后者看稳定性、成本与安全性。

### 4.2 安全：权限、沙箱与人机协同

编码智能体拥有“读写文件 + 执行任意命令”的能力，风险面远大于聊天机器人。安全设计应遵循**最小权限**与**纵深防御**。

**权限模型**：Claude Code 采用基于权限的架构，并区分手动（Manual）与自动（Auto）等权限模式；自动模式用分类器判断是否放行（[Claude Code 安全文档](https://code.claude.com/docs/en/security)）。工程上应至少支持：

- 每次危险操作前确认（写文件、执行命令、网络访问）；
- 命令白名单/黑名单（如禁止 `git push`、`rm -rf`）；
- 按工具、按路径、按参数粒度的规则；
- 计划模式（plan）：先只读分析、再请求执行。

**操作系统级沙箱**：Claude Code 的 Bash 沙箱由操作系统强制执行，限制文件与网络访问；默认关闭，通过 `/sandbox` 或设置开启；覆盖 Shell 命令及其子进程，但**文件工具、MCP 服务器、钩子（hooks）不在其中**；支持 macOS、Linux、WSL2，**原生 Windows 不沙箱**（[Claude Code 沙箱文档](https://code.claude.com/docs/en/sandboxing)）。这段描述揭示了一个关键事实：**沙箱通常只包住“命令执行”，其它工具需要单独的权限控制**，否则会形成绕过路径。

对编码智能体的建议：

- 在容器/虚拟机中运行不可信任务，隔离整个进程而非仅命令（OpenHands 即以沙箱运行时执行智能体动作，[OpenHands 仓库](https://github.com/All-Hands-AI/OpenHands)）。
- 网络默认拒绝，按域名白名单放行，防止数据外泄。
- 凭据隔离：不把密钥放进智能体可读环境；必要时用“哨兵值 + 代理替换”的方式，让命令看到占位符而非真实密钥（Claude Code 沙箱文档描述了该机制）。
- 保护配置文件与钩子，防止智能体通过修改配置提升自身权限（同上）。

**人机协同**：MCP 规范要求工具调用应始终保留人工拒绝的能力，并建议客户端展示工具、给出视觉提示、对敏感操作弹确认（[MCP Tools](https://modelcontextprotocol.io/specification/2025-06-18/server/tools)）。这应被视为底线而非可选项。

**输入校验与供应链**：

- 服务端**必须（MUST）** 校验所有工具输入、实施访问控制、限流、净化输出（同上）。
- 客户端**不应信任**来自不可信服务器的工具注解（annotations），规范明确要求将其视为不可信（同上）。
- 仓库内容、issue 文本、依赖包、网页都可能携带**提示注入（prompt injection）**；工具输出进入上下文前应视为不可信数据。

**护栏（guardrails）**：OpenAI Agents SDK 把护栏作为独立概念，可在输入/输出阶段做校验与拦截（[OpenAI Agents SDK 文档](https://openai.github.io/openai-agents-python/)）。编码智能体可借鉴：对“危险命令”“疑似外泄”“越界路径”设置独立于主模型的检查层。

**审计**：记录每一次工具调用的输入、输出、批准者与结果，既用于安全审计，也用于问题复盘（MCP 亦建议记录工具使用，[MCP Tools](https://modelcontextprotocol.io/specification/2025-06-18/server/tools)）。

---

## 5. 多智能体编排

### 5.1 何时需要多智能体

多智能体不是默认选项。Anthropic 的编排者-工作者（orchestrator-workers）模式适用于“子任务无法预先确定”的复杂任务，编码恰是典型：需要改哪些文件、每个文件改什么，往往取决于任务本身（[Building effective agents](https://www.anthropic.com/engineering/building-effective-agents)）。其代价是成本、延迟与调试复杂度的上升，因此应“仅在能明确改善结果时才增加复杂度”。

多智能体在编码场景的另一个主要价值是**上下文隔离**：主智能体维持高层计划，子智能体在干净上下文中深挖某个子问题，只回传精炼摘要（通常 1,000–2,000 token），从而避免主上下文被大量探索细节污染（[Effective context engineering](https://www.anthropic.com/engineering/effective-context-engineering-for-ai-agents)）。

### 5.2 常见编排模式

| 模式 | 结构 | 适用场景 | 代表 |
| --- | --- | --- | --- |
| 编排者-工作者 | 中央 LLM 动态拆解并分派子任务 | 多文件改动、开放式调研 | [Anthropic](https://www.anthropic.com/engineering/building-effective-agents) |
| 交接（handoffs） | 一个智能体把控制权交给另一个专长者 | 客服、分领域路由 | [OpenAI Agents SDK](https://openai.github.io/openai-agents-python/) |
| 角色化对话 | 多个角色（产品/架构/开发/测试）按 SOP 协作 | 结构化软件开发流程 | [MetaGPT, arXiv:2308.00352](https://arxiv.org/abs/2308.00352) |
| 可对话多智能体 | 多个智能体通过消息协作、可含人类 | 研究与通用任务编排 | [AutoGen, arXiv:2308.08155](https://arxiv.org/abs/2308.08155) |
| 技能库 + 自主课程 | 智能体积累可复用技能并自我设定目标 | 长程开放环境 | [Voyager, arXiv:2305.16291](https://arxiv.org/abs/2305.16291) |
| 子智能体 + 权限隔离 | 主智能体派生子智能体，各自限定工具/权限 | 代码库探索、并行调查 | [opencode Agents 文档](https://opencode.ai/docs/agents/) |

Anthropic 的多智能体研究系统报告称，在复杂研究任务上多智能体相较单智能体有显著提升（[How we built our multi-agent research system](https://www.anthropic.com/engineering/multi-agent-research-system)）。但需注意其适用条件：任务可并行、子任务相对独立、汇总成本可控。

### 5.3 工程要点

- **接口清晰**：子智能体的输入（任务、上下文、可用工具）与输出（结论、证据、改动）必须结构化，否则错误会在智能体间放大。
- **上下文不重复**：子智能体应返回摘要而非原始轨迹；原始轨迹留在子智能体的日志里供追溯。
- **权限最小化**：不同子智能体授予不同工具集。例如“只读调查”子智能体不应有写权限。opencode 的 agent 配置支持按工具设置权限（如对 `edit`、`shell` 设置 `deny`），并通过 Markdown 文件定义主/子智能体（[opencode Agents 文档](https://opencode.ai/docs/agents/)）。
- **通信协议**：工具层可用 MCP 统一（[MCP 规范](https://modelcontextprotocol.io/specification/2025-06-18)）；智能体间协作可参考 OpenHands 的 Agent Client Protocol（ACP）等方案（[OpenHands 仓库](https://github.com/All-Hands-AI/OpenHands)）。
- **终止与预算**：编排层必须为整个任务树设置全局预算，避免子智能体各自膨胀。
- **可观测性**：跨智能体的追踪是调试多智能体的前提，否则难以定位“哪个环节把任务带偏”。

**常见反模式**：

- 为简单任务强行上多智能体，导致成本翻倍而收益有限；
- 子智能体之间传递大段原始上下文，失去隔离优势；
- 缺乏全局预算与终止条件，任务树无限扩张；
- 子智能体权限过宽，扩大攻击面。

---

## 设计决策清单

下表把前文要点收敛为可逐项拍板的工程决策。每项给出常见选项与建议。

| # | 决策 | 选项 | 建议 | 依据 |
| --- | --- | --- | --- | --- |
| 1 | 智能体形态 | 工作流 / 自主循环 / 混合 | 先工作流，确需动态决策再上自主循环 | [Anthropic](https://www.anthropic.com/engineering/building-effective-agents) |
| 2 | 动作空间 | 文本命令 / 结构化工具调用 / 代码执行 | 以结构化工具调用为主，代码执行为补充 | [MCP](https://modelcontextprotocol.io/specification/2025-06-18/server/tools)、[CodeAct](https://arxiv.org/abs/2402.01030) |
| 3 | 工具数量 | 大而全 / 最小集 | 最小、无重叠、描述清晰 | [Anthropic ACI](https://www.anthropic.com/engineering/building-effective-agents) |
| 4 | 路径语义 | 相对 / 绝对 | 编辑类工具优先绝对路径 | [Anthropic](https://www.anthropic.com/engineering/building-effective-agents) |
| 5 | 编辑格式 | 整文件 / 搜索替换 / diff | 选择模型稳定、开销低的格式并实测 | [Aider](https://github.com/Aider-AI/aider) |
| 6 | 代码上下文 | 预索引 / 即时检索 / 混合 | 即时检索为默认，仓库地图加速 | [Aider repomap](https://aider.chat/docs/repomap.html)、[Anthropic](https://www.anthropic.com/engineering/effective-context-engineering-for-ai-agents) |
| 7 | 项目说明注入 | 无 / 前置注入 | 前置注入 `AGENTS.md` 一类文件 | [Anthropic](https://www.anthropic.com/engineering/effective-context-engineering-for-ai-agents) |
| 8 | 长上下文策略 | 不处理 / 压缩 / 笔记 / 子智能体 | 按任务长度组合使用 | 同上 |
| 9 | 缓存 | 无 / 前缀缓存 | 稳定前缀 + 易变后缀 | [Anthropic Prompt caching](https://www.anthropic.com/news/prompt-caching) |
| 10 | 验证 | 无 / 运行测试与构建 | 改动后自动运行相关验证 | [Anthropic](https://www.anthropic.com/engineering/building-effective-agents) |
| 11 | 停止条件 | 无 / 迭代上限 / 预算 | 迭代、token、时间、费用多维预算 | 同上 |
| 12 | 权限 | 全放开 / 逐次确认 / 规则引擎 | 默认确认 + 细粒度规则 | [Claude Code 安全](https://code.claude.com/docs/en/security) |
| 13 | 命令执行 | 本机 / 沙箱 / 容器 | 生产环境用容器或 OS 沙箱 | [Claude Code 沙箱](https://code.claude.com/docs/en/sandboxing)、[OpenHands](https://github.com/All-Hands-AI/OpenHands) |
| 14 | 网络 | 全开 / 白名单 / 关闭 | 默认拒绝 + 域名白名单 | [Claude Code 沙箱](https://code.claude.com/docs/en/sandboxing) |
| 15 | 凭据 | 直接暴露 / 隔离/掩码 | 环境变量清理或掩码代理 | 同上 |
| 16 | 人工介入 | 无 / 关键点确认 | 危险操作强制确认 | [MCP](https://modelcontextprotocol.io/specification/2025-06-18/server/tools) |
| 17 | 评测 | 只看基准分 / 自建 harness 评测 | 绑定自身 harness 的任务集 + 回归 | [Agentless](https://arxiv.org/abs/2407.01489)、[mini-SWE-agent](https://github.com/SWE-agent/mini-swe-agent) |
| 18 | 可观测性 | 仅日志 / 结构化追踪 | 全链路追踪 + 可回放 | [OpenAI Agents SDK](https://openai.github.io/openai-agents-python/) |
| 19 | 多智能体 | 单智能体 / 编排/子智能体 | 有明确并行或隔离收益时再引入 | [Anthropic](https://www.anthropic.com/engineering/multi-agent-research-system) |
| 20 | 工具协议 | 私有 / MCP | 需要生态与解耦时用 MCP | [MCP](https://modelcontextprotocol.io/specification/2025-06-18) |

---

## 落地路线图

以下路线图假定从零开始，目标是可控地演进到生产可用。每一阶段都应有对应的评测与安全基线。

### 阶段 0：最小可用（Minimal Viable）

目标：让模型能在限定工作目录内完成“读—改—跑测试”的闭环。

- CLI 入口与单轮任务输入；
- 模型接入 + 结构化工具调用；
- 工具：读文件、写文件、列目录、grep、执行命令；
- 循环：组装上下文 → 推理 → 执行 → 回填，含迭代上限与超时；
- 上下文：系统提示 + 项目说明（`AGENTS.md`）+ 近期消息；
- 安全：工作目录限定、危险命令黑名单、执行前确认；
- 日志：完整记录每一步。

验收：能在示例仓库完成小型缺陷修复，且测试通过；失败时能从日志定位原因。

### 阶段 1：可用（v1）

目标：提升稳定性、上下文效率与可评测性。

- 编辑工具升级为模型友好的补丁格式（实测选择）；
- 引入仓库地图或代码检索加速，控制上下文膨胀；
- 提示缓存：稳定前缀 + 易变后缀；
- 长任务：工具结果清理、压缩、待办/笔记；
- 验证闭环：自动运行相关测试/构建并回填；
- 追踪与回放；建立内部任务集与回归评测；
- 权限规则细化（按工具/路径/参数）。

验收：在一组内部任务上稳定通过率与成本可量化；回归集可防止提示/工具改动导致的退化。

### 阶段 2：进阶（Advanced）

目标：面向真实、不可信与长程场景。

- 容器/OS 级沙箱，网络默认拒绝 + 白名单，凭据隔离/掩码；
- 权限模式（手动/自动/计划）与独立护栏层；
- 子智能体编排：只读调查、并行修复、权限隔离；
- MCP 客户端，接入外部工具生态；
- 多智能体追踪与全局预算；
- 持续评测：能力集 + 产品集，覆盖安全与成本指标；
- 审计日志与合规能力。

验收：能在不可信仓库中安全运行；关键指标（通过率、成本、延迟、安全事件）有监控与告警。

> 注：路线图是能力递进建议，不是硬性顺序。若一开始就面向不可信代码，应把沙箱与权限提前到阶段 0。

---

## 参考资料

### 论文

- ReAct: Synergizing Reasoning and Acting in Language Models — https://arxiv.org/abs/2210.03629
- Toolformer: Language Models Can Teach Themselves to Use Tools — https://arxiv.org/abs/2302.04761
- Reflexion: Language Agents with Verbal Reinforcement Learning — https://arxiv.org/abs/2303.11366
- Voyager: An Open-Ended Embodied Agent with Large Language Models — https://arxiv.org/abs/2305.16291
- AutoGen: Enabling Next-Gen LLM Applications via Multi-Agent Conversation — https://arxiv.org/abs/2308.08155
- MetaGPT: Meta Programming for A Multi-Agent Collaborative Framework — https://arxiv.org/abs/2308.00352
- SWE-bench: Can Language Models Resolve Real-World GitHub Issues? — https://arxiv.org/abs/2310.06770
- SWE-agent: Agent-Computer Interfaces Enable Automated Software Engineering — https://arxiv.org/abs/2405.15793
- CodeAct: Executable Code Actions Elicit Better LLM Agents — https://arxiv.org/abs/2402.01030
- Agentless: Demystifying LLM-based Software Engineering Agents — https://arxiv.org/abs/2407.01489
- Evaluating Large Language Models Trained on Code (HumanEval) — https://arxiv.org/abs/2107.03374

### 官方文档与工程博客

- Anthropic, Building effective agents — https://www.anthropic.com/engineering/building-effective-agents
- Anthropic, Effective context engineering for AI agents — https://www.anthropic.com/engineering/effective-context-engineering-for-ai-agents
- Anthropic, How we built our multi-agent research system — https://www.anthropic.com/engineering/multi-agent-research-system
- Anthropic, Writing effective tools for agents — https://www.anthropic.com/engineering/writing-tools-for-agents
- Anthropic, Demystifying evals for AI agents — https://www.anthropic.com/engineering/demystifying-evals-for-ai-agents
- Anthropic, Prompt caching — https://www.anthropic.com/news/prompt-caching
- OpenAI, Function calling and other API updates — https://openai.com/index/function-calling-and-other-api-updates/
- OpenAI, Introducing SWE-bench Verified — https://openai.com/index/introducing-swe-bench-verified/
- OpenAI Agents SDK 文档 — https://openai.github.io/openai-agents-python/
- Claude Code 安全文档 — https://code.claude.com/docs/en/security
- Claude Code 沙箱文档 — https://code.claude.com/docs/en/sandboxing
- Model Context Protocol 规范 2025-06-18 — https://modelcontextprotocol.io/specification/2025-06-18
- MCP Tools 规范 — https://modelcontextprotocol.io/specification/2025-06-18/server/tools
- opencode Agents 文档 — https://opencode.ai/docs/agents/

### 开源项目

- Aider — https://github.com/Aider-AI/aider
- Aider repo map 文档 — https://aider.chat/docs/repomap.html
- SWE-agent — https://github.com/SWE-agent/SWE-agent
- mini-SWE-agent — https://github.com/SWE-agent/mini-swe-agent
- OpenHands — https://github.com/All-Hands-AI/OpenHands
- opencode — https://github.com/anomalyco/opencode

---

### 未验证与局限

- Google Gemini 的函数调用官方文档在本次调研中多次无法访问，故未纳入函数调用章节的厂商对比；相关结论仅基于 OpenAI 公告与 MCP 规范。
- OpenAI 官方函数调用指南页面（platform.openai.com / developers.openai.com）访问受限（HTTP 403），本文改用其 2023 年发布公告作为一手来源。
- Anthropic 工具调用 API 参考页因区域限制不可访问，工具协议细节以 MCP 规范与 Anthropic 工程博客为准。
- mini-SWE-agent 的 “>74%” 与 SWE-agent 的 “65%” 均来自各自仓库 README 的自述，未在统一配置下复核（相对基线与 harness 差异会影响可比性）。
