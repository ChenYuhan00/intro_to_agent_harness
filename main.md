# Agent Harness 入门

## 这次分享怎么讲

这次分享就按仓库里的材料顺序来讲，一共三段：

1. **先看 `learn-claude-code` 项目实例。** 用一个最小但完整的 agent harness 教会大家：模型怎么和真实世界连接起来，工具怎么接，计划怎么维护，为什么需要子代理、skill loading 和后台任务。
2. **再讲 MCP 和 Skill。** 前一部分解决的是"agent 框架怎么跑起来"，这一部分解决的是"外部能力怎么接进来、知识怎么按需注入"。
3. **最后讲一个对比例子。** 用飞书这个实际案例，对比 MCP Server 路线和 Skill + CLI 路线，让前面的抽象概念落地。

仓库里对应的三组材料：

- `learn_claude_code_sample/`：项目实例拆解
- [mcp_intro.md](./mcp_intro.md)：MCP 讲解
- [skill_intro.md](./skill_intro.md)：Skill 讲解
- [tmp/mcp_vs_skill_with_feishu_example.md](./tmp/mcp_vs_skill_with_feishu_example.md)：MCP vs Skill 对比例子

---

## 第一部分：从 `learn-claude-code` 项目实例理解 Agent Harness

这一部分不从概念空讲，而是直接基于 [shareAI-lab/learn-claude-code](https://github.com/shareAI-lab/learn-claude-code) 这个教学仓库来拆。它的价值在于：代码短、结构清楚，而且把一个编程 agent harness 里最核心的机制都单独做成了可运行示例。

### 先建立三个基本概念

#### LLM

LLM（大语言模型）就是 ChatGPT、Claude 背后的技术。输入一段文字，输出一段文字。

关键限制：**LLM 只能生成文本，做不了别的。**

#### Agent

Agent 是模型本身。它负责理解任务、做判断、决定下一步该调用什么能力。

#### Harness

Harness 是模型外面的那层控制程序。它负责发请求、执行工具、维护状态、管理上下文、回填结果。

### 三者关系

以 Claude Code 这类编程 agent 为例，整个系统可以粗略看成两端：

```text
        ┌─────────────────┐             ┌────────────────────┐
        │  Agent Server   │             │   本地 Harness      │
        │  (模型提供方)    │  HTTPS API  │   (你的电脑)        │
        │                 │◄───────────►│                    │
        │  Claude / GPT   │  请求/响应   │  Agent Loop        │
        │                 │             │  工具执行           │
        │  接收 messages   │             │  文件读写           │
        │  返回 response   │             │  Shell 命令         │
        │                 │             │  上下文管理         │
        └─────────────────┘             └────────────────────┘
              │                                  │
              │                                  │
         只做推理                           负责执行与控制
```

- **模型端**：只负责推理，不直接碰本地文件、shell、网络权限。
- **Harness 端**：负责真实执行，是 agent 真正"能做事"的工程壳。
- **中间 API**：把模型决策和本地执行循环连接起来。

一句话：**模型负责思考，harness 负责动手。**

### 为什么用这个项目来讲

这个项目把一个 agent harness 里最常见的六个机制拆开了：

| 机制 | 解决什么问题 |
|------|-------------|
| Agent Loop | 模型只能生成文本，怎么让它持续操作真实世界 |
| Tool Use | 只给 bash 太粗糙，怎么提供结构化工具 |
| TodoWrite | 长任务会失忆，怎么维护执行计划 |
| Subagent | 主上下文会被污染，怎么隔离子任务 |
| Skill Loading | 领域知识不能全常驻，怎么按需加载 |
| Background Tasks | 慢操作会阻塞，怎么后台执行 |

下面的讲解顺序也是建议的分享顺序。

---

## 1. Agent Loop：一切的地基

模型只能输出文本，不能直接执行命令。Harness 的最小闭环就是：

```text
发请求 → 模型回复 → 执行工具 → 回填结果 → 再发请求 → ...
```

这里最关键的三个词：

- `tool_use`：模型请求某个动作
- `tool_result`：harness 执行后把结果喂回去
- `stop_reason`：决定继续循环还是结束

这一层讲明白后，后面所有机制都只是往这个循环上叠能力。

详细材料：**[Agent Loop 详解 →](./learn_claude_code_sample/agent_loop.md)**

## 2. Tool Use：把"能执行"变成"可控地执行"

如果只有一个 `bash` 工具，会有三个问题：

- 安全边界难控
- 参数语义不清楚
- 错误处理不稳定

所以实际 harness 会把能力拆成更细的专用工具，例如 `read_file`、`write_file`、`edit_file`。本质上还是同一个循环，但从"固定执行 shell"变成"按工具名分发到 handler"。

详细材料：**[Tool Use 详解 →](./learn_claude_code_sample/tool_use.md)**

## 3. TodoWrite：让模型自己维护执行计划

任务一长，模型会忘记已经做到哪一步。这时需要一个计划工具，让模型自己维护 todo 列表，并把最新状态重新注入上下文。

这个机制的重点不是 UI，而是：**计划状态也可以被当作工具结果重新写回消息流。**

详细材料：**[TodoWrite 详解 →](./learn_claude_code_sample/todo_write.md)**

## 4. Subagent：把探索过程隔离出去

如果所有中间探索都堆在主线程的 `messages` 里，上下文很快就会变脏。Subagent 的做法是：把某个子任务交给一个新的模型调用，只把结论带回来，不把整段探索历史带回来。

核心收益：

- 主上下文更干净
- 子任务可独立推理
- 父代理只消费摘要，不消费全过程

详细材料：**[Subagent 详解 →](./learn_claude_code_sample/subagent.md)**

## 5. Skill Loading：知识不是一直常驻，而是按需进入

如果把所有技能全文都塞进 system prompt，会非常浪费上下文。所以更合理的方式是：

- 常驻：skill 名称和一句话描述
- 按需：真正触发时再加载完整 `SKILL.md`

这一步非常重要，因为它正好是后面讲 Skill 的桥。

详细材料：**[Skill Loading 详解 →](./learn_claude_code_sample/skill_loading.md)**

## 6. Background Tasks：把慢操作移到后台

有些任务会跑很久，比如测试、构建、爬取。主循环不应该卡住等待，所以需要后台执行，并在完成后通过通知队列把结果补回上下文。

这一步让大家看到：**agent harness 不只是"一问一答"，而是一个有状态、有并发、有调度的运行时。**

详细材料：**[Background Tasks 详解 →](./learn_claude_code_sample/background_tasks.md)**

### 第一部分讲完，应该让听众得到什么

讲完 `learn-claude-code` 这一段，听众应该先建立一个判断框架：

- agent 不是"一个更聪明的 prompt"，而是一套运行时
- tool、todo、subagent、background task 都是在扩展同一个 loop
- skill loading 不是附属功能，而是上下文管理机制

有了这个框架，再讲 MCP 和 Skill，听众才知道它们分别插在 harness 的哪一层。

---

## 第二部分：MCP 和 Skill

前一部分讲的是 agent harness 的运行方式，这一部分讲两种常见扩展点：

- **MCP**：把外部系统能力标准化接进 agent
- **Skill**：把任务知识和工作流按需注入 agent

### 先讲 MCP

MCP（Model Context Protocol）要解决的是：不同 AI 应用怎么用统一方式接入外部能力。

讲 MCP 时建议按下面这条线讲：

1. MCP 是协议，不是模型
2. 它有 Host / Client / Server 三个角色
3. Server 可以暴露 tools、resources、prompts 三类能力
4. Host 连接 Server 后，会把这些能力以结构化方式带进上下文
5. 所以 MCP 解决的是"连接和标准化接入"

详细材料：**[MCP 讲解 →](./mcp_intro.md)**

### 再讲 Skill

Skill 要解决的是另一类问题：某个任务怎么做、遇到什么条件时该触发、需要读哪些补充资料、需要运行哪些脚本。

讲 Skill 时建议抓住这几个点：

1. Skill 是一个带 `SKILL.md` 的目录，不只是"一段 prompt"
2. Skill 的发现和加载是分层的
3. `description` 决定它什么时候该被触发
4. `SKILL.md` 负责主流程，`references/` 和 `scripts/` 负责按需展开
5. 所以 Skill 解决的是"知识组织和工作流注入"

详细材料：**[Skill 讲解 →](./skill_intro.md)**

### 这一部分最重要的一句话

可以把两者先粗略区分成：

- **MCP 偏能力接入**
- **Skill 偏任务指导**

也就是：

- MCP 主要回答"agent 能调用什么"
- Skill 主要回答"agent 遇到这类任务时应该怎么做"

---

## 第三部分：用飞书例子对比 MCP 和 Skill

前两部分分别讲了 harness、MCP、Skill，但如果停在概念层，大家还是容易混。最好的收束方式就是上一个真实案例。

这个仓库已经准备好了一个非常适合讲的例子：**飞书操作**。

它的优点是：

- 同一件事，既可以做成 MCP Server，也可以做成 Skill + CLI
- 两条路线最终都落到同一套飞书能力实现
- 差异非常集中地体现在"能力如何暴露给模型"

建议把这个例子当成最后的收束部分。

### 这个例子里要重点对比什么

#### 1. 上下文注入方式不同

- **MCP**：注入的是 tool 名称、描述、参数 schema
- **Skill**：注入的是 `SKILL.md` 里的自然语言操作手册，以及后续按需读取的 reference 文档

#### 2. 模型知道怎么调用的方式不同

- **MCP**：模型通过协议自动发现工具
- **Skill**：模型通过阅读文档学会如何拼命令、如何检查前置条件

#### 3. 宿主承担的责任不同

- **MCP**：宿主负责接入 Server，把工具定义暴露给模型
- **Skill**：宿主只需要支持技能加载和 shell/文件等基础工具

#### 4. 适用场景不同

- 外部能力已经做成标准服务、希望多宿主复用：更适合 MCP
- 需要强执行流程、前置检查、文档化工作流：更适合 Skill

详细材料：**[飞书案例：MCP vs Skill →](./tmp/mcp_vs_skill_with_feishu_example.md)**

---

## 整体收束

按这套顺序讲，整场分享的主线会比较顺：

1. **先用 `learn-claude-code` 建立 agent harness 的运行框架。**
2. **再把 MCP 和 Skill 放回这个框架里，解释它们分别负责哪一层。**
3. **最后用飞书案例做对比，让抽象概念落到具体工程实现。**

如果只让听众记住三句话，我建议是：

1. **Agent 不是单个模型，而是"模型 + harness 运行时"。**
2. **MCP 解决的是外部能力的标准化接入，Skill 解决的是知识和工作流的按需注入。**
3. **MCP 和 Skill 不是互斥关系，它们常常会在同一个 agent 系统里一起出现。**

---

## 附录：Codex 上下文构建实例

为了让大家知道一个真实编程 agent 的上下文到底长什么样，这个仓库还附了一个 Codex 请求结构的拆解：

- [codex_context_structure.json](./codex_context_structure.json)

这个例子可以放在分享最后作为补充，帮助大家把前面讲的几件事对应到真实产品里：

- system / developer / user 指令如何分层
- tools 和 MCP tools 如何一起进入上下文
- AGENTS.md / skill / memory 这类长期信息如何参与构建
- 整个运行过程如何表现为 `assistant -> tool_call -> tool_result` 的循环

如果前面的主线是"最小教学实现"，这个附录就是"真实产品实现的上下文切片"。
