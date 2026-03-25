# Agent Harness 入门

这份文档主要包含三部分内容：agent harness 的基本工作方式，MCP 和 Skill 分别解决什么问题，以及同一个真实任务在 MCP 路线和 Skill 路线下的具体展开方式。文中也配了示例和补充材料，方便对照阅读。

`harness` 的原意是"马具"——套在马身上、让人能控制和驱动马的那套装备。在 agent 语境下，它指的是包在模型外面的运行时框架：负责发请求、暴露工具、执行动作、维护状态、管理上下文。模型负责决定下一步做什么，harness 负责把这些决定真正执行起来。我们现在用的 Claude Code、Codex、OpenClaw，都是这套东西。2026 年 2 月 OpenAI 在 [Harness engineering](https://openai.com/index/harness-engineering/) 一文中用了这个词，把围绕它去搭环境、接工具、做反馈闭环的工作称为 `harness engineering`。文中说的 `agent harness`，指的就是这整套让 agent 能持续工作的运行机制。

这篇文档围绕三个问题展开：

1. 一个最小但完整的 agent harness 是怎么工作的。
2. MCP 和 Skill 分别解决什么问题，它们和 harness 的关系是什么。
3. 面对同一个真实任务，MCP 路线和 Skill 路线分别长什么样。

> [!info] 补充材料
> - `learn_claude_code_sample/`：基于 `learn-claude-code` 的 harness 机制拆解
> - [[mcp_intro]]：MCP 介绍
> - [[skill_intro]]：Skill 介绍
> - [[mcp_vs_skill_with_feishu_example]]：飞书场景下的 MCP vs Skill 对比
> - `codex_context_structure.json`：Codex 对话上下文解包后的精简可读版，用占位符替代了大段原始内容
> - `codex_context_full.json`：Codex 对话上下文解包后的完整版，保留更完整的原始请求上下文

---

## 第一部分：从 `learn-claude-code` 理解 Agent Harness

[learn-claude-code](https://github.com/shareAI-lab/learn-claude-code) 是一个很适合入门的教学项目。它把 agent 的运行过程拆成几个清楚的部件，让人能看见模型决策、工具执行、状态维护和上下文管理是怎么配合起来的。

### LLM、Agent、Harness

先区分三个容易混在一起的概念。

**LLM** 是大语言模型，输入和输出本质上都是文本。关键限制：**LLM 只能生成文本，做不了别的。**

**Agent** 是一个系统——模型在循环中自主使用工具来完成任务。它不是单独的模型，也不是单独的框架，而是模型 + 工具 + 循环三者构成的整体。Anthropic 的定义是："LLMs dynamically direct their own processes and tool usage"（[Building Effective Agents](https://www.anthropic.com/research/building-effective-agents)），更简洁的说法是 "LLMs autonomously using tools in a loop"。

**Harness** 是包在模型外面的运行时。它负责向模型发请求、向模型暴露工具、执行模型请求的动作、维护任务状态，并把执行结果重新写回对话上下文。

以 Claude Code 这类编程 agent 为例，整个系统可以粗略看成两端：

```text
        ┌──────────────────────┐             ┌──────────────────────┐
        │ Agent Server         │             │ 本地 Harness         │
        │ （模型提供方）       │  HTTPS API  │ （你的电脑）         │
        │                      │◄───────────►│                      │
        │ Claude / GPT         │  请求 / 响应 │ Agent Loop          │
        │                      │             │ 工具执行             │
        │ 接收 messages        │             │ 文件读写             │
        │ 返回 response        │             │ Shell 命令           │
        │                      │             │ 上下文管理           │
        └──────────────────────┘             └──────────────────────┘
               │                                       │
               │                                       │
            只做推理                              负责执行与控制
```

> [!tip] 核心判断
> 真正能完成任务的系统通常不是"一个模型"，而是"模型 + harness"。模型负责思考，harness 负责动手。

### 一个最基本的闭环

从最小视角看，agent harness 的核心就是一个循环：

```text
发送请求 → 模型回复 → 执行工具 → 写回结果 → 再次发送请求
```

模型本身不会直接执行 shell、不会直接改文件、也不会直接访问外部系统。它能做的是发出"我想调用某个工具"的结构化请求。Harness 收到这个请求后负责真正执行，然后把结果继续交给模型。

这里最关键的三个词：

- `tool_use`：模型请求某个动作
- `tool_result`：harness 执行后把结果喂回去
- `stop_reason`：决定继续循环还是结束

这就是后面所有机制的共同地基。

### 六个基础机制示例

这里挑了六个比较基础的 harness 机制来讲：

#### 1. Agent Loop

Agent loop 解决的是"模型怎么持续工作"。没有 loop，模型只会给出一段回答；有了 loop，模型就可以在每一轮根据新结果决定下一步动作。也正因为有这一层，agent 才不再只是问答系统，而是能逐步完成任务的执行系统。

→ [[learn_claude_code_sample/agent_loop|agent_loop]]

#### 2. Tool Use

只有一个通用 `bash` 工具时，能力边界、安全限制和参数语义都很模糊。更常见的做法是把动作拆成明确的工具，比如 `read_file`、`write_file`、`edit_file`。这样 harness 对模型开放的能力变得更清楚、更可控，也更容易做权限边界和错误处理。

→ [[learn_claude_code_sample/tool_use|tool_use]]

#### 3. TodoWrite

任务一长，模型就容易丢步骤、重复步骤，或者忘记当前进度。`TodoWrite` 这类机制的作用是把计划状态也纳入运行时，让模型自己维护一份任务清单，并根据最新状态继续推进。这说明一件重要的事：进入上下文的不只是"外部工具执行结果"，也可以是"系统内部状态"。

→ [[learn_claude_code_sample/todo_write|todo_write]]

#### 4. Subagent

主线程里的上下文是有限的。如果所有探索、试错和分支推理都堆在同一个消息流里，很快就会变得嘈杂。Subagent 的作用是把某个子问题交给一个新的上下文去处理，主代理只拿回结论或摘要。这是一种典型的上下文管理策略。

→ [[learn_claude_code_sample/subagent|subagent]]

#### 5. Skill Loading

如果把所有领域知识都常驻在 system prompt 里，上下文会迅速膨胀。更合理的做法是只让技能名称和简短描述常驻，真正相关时再加载对应 `SKILL.md` 的正文。知识不是一直都在，而是在需要的时候进入上下文。这部分也是理解后面 Skill 机制的关键桥梁。

→ [[learn_claude_code_sample/skill_loading|skill_loading]]

#### 6. Background Tasks

有些操作很慢（测试、编译、批处理、远程任务）。如果主循环必须阻塞等待，整个交互就会变得很差。后台任务把慢操作放到后台执行，等结果准备好后再注入上下文，让 harness 更像一个有调度能力的运行时。

→ [[learn_claude_code_sample/background_tasks|background_tasks]]

> [!abstract] 小结
> Agent 的关键不只是模型能力，更是 harness 如何组织循环、工具、状态和上下文。理解了这一层，再去看 MCP 和 Skill，就更容易看清它们分别插在系统的哪一部分。

---

## 第二部分：MCP 和 Skill

前一部分讨论的是 harness 的运行机制，这一部分讨论两种常见的扩展方式。

### MCP 解决什么问题

MCP（Model Context Protocol）解决的是"外部能力怎么以统一方式接进 AI 应用"。

如果模型要访问飞书、GitHub、数据库或本地文件系统，底层都需要有人把这些能力包装出来。MCP 做的事情就是为这种包装提供一个统一协议，让不同 Host 都能用一致的方式连接外部能力。

MCP 这一层关注的是：

- Host、Client、Server 之间怎么分工
- 外部能力如何被声明为 tools、resources、prompts
- 这些能力如何通过协议被带入上下文

详细内容见 [[mcp_intro]]。

### Skill 解决什么问题

Skill 解决的是另一个问题：当 agent 遇到某一类任务时，应该按什么流程做、先检查什么、需要读取哪些资料、什么时候运行哪些脚本。

Skill 更像是"任务知识包"和"工作流说明书"的组合。它不是在协议层声明外部能力，而是在上下文层组织任务做法。

Skill 这一层关注的是：

- 一个 skill 如何被发现
- `description` 如何帮助模型判断是否要加载
- `SKILL.md`、`references/`、`scripts/` 如何分工
- 为什么 skill 应该按需加载，而不是全量常驻

详细内容见 [[skill_intro]]。

### 两者的区别

> [!note] 一句粗略的区分
> - **MCP 更偏"能力接入"**——主要回答"系统能调用什么外部能力"
> - **Skill 更偏"任务指导"**——主要回答"面对这类任务应该怎么做"

二者并不冲突。一个完整的 agent 系统里，往往既会有 MCP，也会有 Skill。

---

## 第三部分：飞书场景下的 MCP 与 Skill 对比

理解概念之后，直接看一个真实任务中的差异：同一件事如果分别走 MCP 路线和 Skill 路线，实际会有什么不同。

飞书场景很适合做这个对比，因为同一类能力既可以通过 MCP Server 暴露，也可以通过 Skill + CLI 的方式间接使用。

### MCP 路线

模型拿到的是结构化工具定义（tool 名称、描述、参数 schema）。模型知道"有哪些动作可以调"，并且知道参数应该长什么样。调用动作时，走的是协议层。

### Skill 路线

模型先拿到的是一份操作说明：什么时候该用这套流程、调用前要做哪些检查、参数说明在哪份 reference 文档里、最终应该如何拼出 CLI 命令。模型先学会"这类任务该怎么做"，再借助 shell 或 CLI 去执行动作。

### 重点差异

两条路线最终都可能落到同一套底层能力实现，但模型接触它们的方式不一样：

| | MCP | Skill |
|---|---|---|
| **注入形式** | 结构化工具定义 | 自然语言工作流 + 补充文档 |
| **触发方式** | 协议发现 | 按描述触发、按需加载 |
| **责任边界** | 宿主如何接入外部服务 | 任务流程如何被组织和约束 |

完整例子见 [[mcp_vs_skill_with_feishu_example]]。

---

## 总结

> [!important] 三句核心结论
> 1. **Agent 通常不是单独一个模型，而是模型和 harness 组成的运行系统。**
> 2. **MCP 让外部能力以统一协议接入，Skill 让任务知识按需进入上下文。**
> 3. **MCP 和 Skill 往往不是二选一，而是同一套 agent 系统里的两层机制。**

---

## 附录：一个真实产品中的上下文长什么样

[[codex_context_structure.json]] 展示的是，在真实产品中，system 指令、developer 指令、用户输入、memory、工具定义、MCP tools 和工具调用结果，是怎样一起组成完整上下文的。

如果前面的 `learn-claude-code` 展示的是一个教学型最小实现，这个附录展示的就是一个真实编程 agent 在生产环境中的上下文切片。
