---
title: "MCP：让 AI 应用连接外部世界的标准协议"
tags:
  - MCP
  - 协议
  - AI-agent
date: 2026-03-24
---

# MCP：让 AI 应用连接外部世界的标准协议

---

## 第一部分：MCP 是什么？

MCP，全称 Model Context Protocol，是一套开放协议，用来把外部系统的能力标准化地接入 AI 应用。

说白了，AI 模型本身只会处理文字，它不知道怎么登录飞书、怎么查数据库、怎么调 GitHub API。要让它真正能"做事"，就得有人把这些外部能力包装好给它用。

MCP 干的就是这件事——定一个统一的标准，让外部能力都按同样的方式接入 AI 应用。

不管你的 AI 应用是 Claude Desktop、Cursor、Codex 还是自己写的 agent，只要大家都遵守 MCP 协议，外部能力就可以复用，不用每个应用都从头写一遍集成代码。

MCP 不是万能的，它不会让模型变聪明，也不会替代底层 API。它只是==统一了接入方式==。

---

## 第二部分：MCP 的架构

MCP 官方定义了三个核心概念：Host、Client、Server。下面按官方规范逐个说清楚。

### Host

Host 就是你直接使用的那个 AI 应用。官方的定义是：**创建并管理一个或多个 MCP Client 的 AI 应用程序。**

常见的 Host 有：

- Claude Desktop
- Claude Code
- Cursor
- Codex
- 你自己写的 agent 程序

Host 的职责：

- 面向用户：接收输入，展示回答
- 调用 AI 模型
- 创建和管理 MCP Client（决定连哪些 Server、控制权限）
- 把从各个 Server 获取到的能力聚合起来给模型用

Host 是整个系统的==控制中心==。

### Server

Server 是**通过 MCP 协议向 Client 暴露能力的程序**。每个 Server 封装一类具体的外部能力。

比如：

- `feishu-mcp`：暴露飞书相关能力（搜文档、发消息等）
- `github-mcp`：暴露 GitHub 相关能力（创建 issue、查 PR 等）
- `filesystem-mcp`：暴露本地文件操作
- `postgres-mcp`：暴露数据库查询

Server 通过三种标准原语暴露能力：tools、resources、prompts（后面会详细讲）。

注意：Server 不是外部系统本身。`feishu-mcp` 不是飞书，它只是把飞书 API 包装成了 MCP 标准格式，背后还是在调飞书开放平台的接口。

### Client

Client 是最容易搞混的一个概念。官方定义是：**维护与某一个 Server 的连接、为 Host 从 Server 获取上下文的组件。**

关键点：

- Client 不是一个独立的程序，它是 Host 内部创建出来的
- **一个 Client 只对应一个 Server**，这是 1:1 的关系
- Host 要连多个 Server，就创建多个 Client

拿 VS Code 举个例子：VS Code 作为 Host，连接了 Sentry MCP Server 和 Filesystem MCP Server。它内部就会创建两个 Client 实例，一个负责和 Sentry Server 通信，一个负责和 Filesystem Server 通信。两条连接互相隔离。

Client 具体干的事情：

- 和对应的 Server 建立连接，完成协议版本和能力的协商
- 转发 JSON-RPC 请求和响应
- 管理订阅和通知
- 处理断连和超时

所以 Client 的角色就是：==Host 为每个 Server 创建的专属连接管理器。==

这里的 `JSON-RPC` 你可以先把它简单理解成：**一种结构化的“发请求 / 收响应”消息格式**。  
初学时不用纠结它的细节，只要知道 Host 和 Server 之间不是在传自然语言，而是在传这种格式化消息就够了。

### 三者的关系

```text
Host（AI 应用，比如 Claude Desktop）
 │
 ├── Client A ──── 1:1 ──── feishu-mcp Server ──── 飞书开放平台
 │
 ├── Client B ──── 1:1 ──── github-mcp Server ──── GitHub API
 │
 └── Client C ──── 1:1 ──── filesystem Server ──── 本地文件系统
```

- Host 管理所有 Client
- 每个 Client 只和一个 Server 对话
- 每个 Server 背后对接一个外部系统

再加上外部系统，整条链路就是：

```text
用户 → Host → Client → Server → 外部系统
```

用户跟 Host 说话，Host 调模型，模型决定要用某个工具，Host 通过对应的 Client 把请求发给 Server，Server 去调外部系统拿结果，再原路返回。

### 具体一点：到底什么东西跑在哪？

光看定义还是抽象，下面拿一个真实场景来说：你在自己电脑上打开 Claude Desktop，对它说"帮我搜飞书文档"。

这时候各个东西分别跑在哪：

```text
┌─────────────────── 你的电脑（本地） ───────────────────┐
│                                                        │
│  Claude Desktop（Host）                                │
│    └── MCP Client（Host 内部的一段代码）               │
│                                                        │
│  feishu-mcp 进程（Server，本地启动的一个进程）         │
│                                                        │
└────────────────────────────────────────────────────────┘
          │                           │
          │ 调模型                    │ 调飞书 API
          ▼                           ▼
   Anthropic 云端                  飞书云端
  （Claude 模型）              （飞书开放平台）
```

逐个看：

- **Host（Claude Desktop）**：跑在你电脑上，是你能看到、能点的那个应用窗口
- **Client**：跑在 Host 进程内部，就是 Host 里面的一段代码，不是一个单独的程序
- **Server（feishu-mcp）**：也跑在你电脑上，是 Host 拉起来的一个本地进程。Host 通过 stdin/stdout 跟它通信
- **AI 模型（Claude）**：跑在 Anthropic 的云端服务器上，Host 通过网络调用它
- **外部系统（飞书）**：跑在飞书的云端服务器上，Server 通过网络调用飞书的 API

所以在这个场景里，你电脑上跑着两个东西：Host 和 Server。模型和飞书都在远端。

如果换成远程部署的 MCP Server（比如团队共享的一个 GitHub MCP 服务部署在公司服务器上），那图就变成：

```text
┌───── 你的电脑 ────┐
│                   │
│  Claude Desktop   │
│   └── Client A    │ ── stdio ──→ feishu-mcp（本地进程）──→ 飞书云端
│   └── Client B    │ ── HTTP ───→ github-mcp（公司服务器）──→ GitHub
│                   │
└───────────────────┘
         │
         │ 网络请求
         ▼
    Anthropic 云端
   （Claude 模型）
```

总结一下哪些在哪：

| 组件 | 通常跑在哪 | 说明 |
|---|---|---|
| Host | 你的电脑 | 你直接用的那个应用 |
| Client | Host 进程内部 | 不是独立程序，是 Host 的一部分 |
| Server（stdio 模式） | 你的电脑 | Host 拉起的本地进程 |
| Server（HTTP 模式） | 远程服务器 | 团队共享部署，通过网络连接 |
| AI 模型 | 云端（Anthropic / OpenAI 等） | Host 通过 API 调用 |
| 外部系统 | 看具体是什么 | 飞书、GitHub 在云端；文件系统在本地 |

---

## 第三部分：MCP Server 能提供哪三类能力？

MCP Server 可以向 Host 暴露三类能力：Tools、Resources、Prompts。它们进入模型上下文的方式各不相同。

下面用 GitHub MCP Server（官方开源的，很多人在用）做例子，逐个讲清楚每类能力**是什么、怎么注入上下文、模型看到的是什么**。

### 第 1 类：Tools（工具）——模型可以调用的动作

**是什么：** Server 预先定义好的一组可执行操作，每个工具有名字、参数 schema、返回格式。

**怎么注入上下文：**

1. Host 启动时，通过 Client 向 Server 发送 `tools/list` 请求
2. Server 返回所有可用工具的定义（名字 + 参数 schema + 描述）
3. Host 把这批工具定义暴露给模型，常见做法是通过发给模型的 tools 参数

注意：不是所有工具每次都会全给模型看到，具体暴露哪些取决于 Host 的实现。

**模型看到的东西大概长这样：**

```text
可用工具：
- create_issue(owner, repo, title, body)：在指定仓库创建 issue
- search_repositories(query)：按关键词搜索 GitHub 仓库
- create_pull_request(owner, repo, title, head, base)：创建 PR
- list_commits(owner, repo, sha)：查看提交记录
```

**然后怎么用：** 用户说"帮我在 xxx 仓库提一个 bug issue"，模型看到工具列表里有 `create_issue`，就决定调用它，输出一段结构化的调用请求（工具名 + 参数）。Host 收到后通过 Client 转发给 Server 执行，拿到结果后再回填给模型。具体的完整流程在第五部分会详细展开。

### 第 2 类：Resources（资源）——提前塞给模型的背景材料

**是什么：** Server 暴露的一组可读取的数据，每个资源有一个 URI（类似网址），内容可以是文本、代码、JSON 等。

**怎么注入上下文：**

1. Host 通过 Client 向 Server 发送 `resources/list` 请求，拿到可用资源列表
2. Host、用户，或者具体实现里的模型选择逻辑决定需要哪些资源
3. Host 通过 Client 发送 `resources/read` 请求，拿到资源内容
4. Host 把资源内容直接拼进发给模型的消息里，作为上下文的一部分

关键区别：资源的内容是在==调用模型之前==就塞进去的，模型拿到的时候已经能直接看到这些数据了。

**举个例子：** GitHub MCP Server 可以把这些东西作为资源暴露：

- `github://repos/owner/repo/readme`——某个仓库的 README
- `github://repos/owner/repo/pulls/123/diff`——某个 PR 改了哪些代码
- `github://repos/owner/repo/issues/456/comments`——某个 issue 的讨论记录

用户说"帮我 review 一下这个 PR"，Host 可以先通过 `resources/read` 拿到 PR 的 diff 内容，把它塞进模型的上下文，然后模型看到完整的代码改动，再给 review 意见。

### 第 3 类：Prompts（提示模板）——用户可以触发的预设工作流

**是什么：** Server 预先定义好的、带参数的提示词模板。每个模板有名字、参数列表、和一段预写好的 prompt 内容。

**怎么注入上下文：**

1. Host 通过 Client 向 Server 发送 `prompts/list` 请求，拿到可用模板列表
2. Host 把这些模板展示给用户（比如在界面上显示一个可选列表）
3. 用户选择某个模板，填入参数
4. Host 通过 Client 发送 `prompts/get` 请求，Server 返回渲染好的 `messages` 数组
5. Host 把这些消息并入当前会话，再发给模型

**模型看到的是一组已经组织好的消息。** 在简单场景里，它常常看起来像一段 prompt；但从协议上说，返回值不是单个字符串，而是 `messages`。

**举个例子：** GitHub MCP Server 定义了一个叫 "review-pr" 的模板：

- 用户在界面上选这个模板，填入 PR 链接
- Server 返回一组消息，可能包含一条或多条 `user` / `assistant` 消息；简单时可以近似理解成："请 review 以下 PR。先检查代码风格，再检查逻辑错误，最后给出总结。PR diff 如下：{diff 内容}"
- Host 把这些消息并入会话，模型按这个结构去执行

用户不需要自己想怎么写 prompt，选一个模板、填几个参数就行。

**在不同 Host 里长什么样？**

不同的 Host 会用不同的方式把 MCP Prompts 展示给用户：

- **Claude Desktop**：在界面上显示为一个可选列表，用户点击选择、填参数、触发
- **Claude Code**：有些 prompt 会以比较接近 command 的方式暴露出来。比如 GitHub MCP Server 定义了一个叫 `review-pr` 的 prompt，在 Claude Code 里你可能会看到类似 `/mcp__github__review-pr` 这样的触发入口

但这里要注意：**这是某个 Host 的产品呈现方式，不是 MCP 协议强制规定所有 Host 都必须变成 slash command。**
协议只规定”Server 可以提供 prompts”，至于 Host 在界面里把它做成按钮、列表、命令还是别的入口，是 Host 自己的产品设计。

> [!tip] 提示
> 在 Claude Code 中，MCP Prompts 和 [[skill_intro|Skills]] 的呈现方式有相似之处（都可能作为 slash command 出现），但底层机制完全不同。

### 三类能力对比

| 类型 | 是什么 | 怎么进入模型上下文 | 谁主导 |
|---|---|---|---|
| Tools | 可执行的动作 | 工具定义列表随每次调用一起发给模型，模型自己选择调用 | 模型 |
| Resources | 可读取的数据 | Host 提前读取内容，拼进发给模型的消息里 | 应用（Host） |
| Prompts | 预设的提示模板 | 用户选择模板后，`prompts/get` 返回的消息集合并入会话 | 用户 |

三类能力的本质区别就是==谁触发、什么时候进入上下文==：

- Tools：模型在对话过程中主动调用，结果事后回填
- Resources：模型被调用之前，数据已经在上下文里了
- Prompts：用户主动选择触发，模板渲染出的消息进入会话

---

## 第四部分：启动 MCP 后，上下文里多了什么？

第三部分讲了三类能力分别怎么注入上下文。这里换一个角度看：从上下文占用的角度，MCP 到底往里塞了多少东西？

### 固定开销：工具定义列表

Host 在初始化阶段通过 `tools/list` 拿到可用工具定义。很多实现会在每次调用模型时，把当前这轮要暴露给模型的工具列表作为参数发过去。

一个工具的定义占几十到几百个 token。接了多个 Server、几十个工具的话，光工具定义就可能占掉几千甚至上万个 token。

**关键点：这些工具定义不是"注册一次就完事"，而是==每次调用都要带上==。**
像 Claude Code 这样的 Host 支持 `Tool Search`，可以按需延迟暴露工具，但很多 Host 还是全量注入。

### 按需注入（触发后才进入上下文）

- **Resource 内容**：启动时只拿到资源列表，不会自动加载内容。只有通过 `resources/read` 读取后才进入上下文。但一旦加载，一个 PR 的 diff 可能有几千行，直接占掉大量 token。
- **Prompt 模板渲染结果**：用户触发后，渲染出的消息注入会话。
- **Tool 调用结果**：模型每调一次工具，返回的结果就留在上下文里，逐次累积。

### 为什么 MCP 特别占上下文？

原因很直接：

1. **工具定义往往是固定开销。** 在不做按需筛选的实现里，不管用户这轮问的问题需不需要这些工具，整批工具定义都会带上。接的 Server 越多、工具越多，这个固定开销越大。

2. **工具结果会累积。** 模型每调用一次工具，返回的结果就留在上下文里。一轮对话中如果连续调了好几个工具，上下文会迅速膨胀。

3. **Resource 内容往往很大。** 一个文件的内容、一个 PR 的 diff、一个 issue 的讨论记录，随便一个都可能占掉大量 token。

举个直观的例子：

> [!example] 上下文开销估算
> 假设你接了 3 个 MCP Server，总共暴露 40 个工具
>
> - 工具定义列表：约 4000-8000 token（在全量暴露工具的实现里，每次调用都带）
> - 用户问了一个问题，模型调了 2 个工具
>   - 工具结果 1：约 500 token
>   - 工具结果 2：约 2000 token
> - Host 还加载了一个 resource：约 3000 token
>
> ==这一轮对话，MCP 相关内容就占了约 10000-14000 token==

这就是为什么接了很多 MCP Server 之后，你会明显感觉上下文"不够用"——大量空间被工具定义和调用结果吃掉了。

---

## 第五部分：一次完整的 tool 调用流程

前面零散地提到了 tool 怎么用，这里把一次调用从头到尾完整走一遍。

### 场景

用户对 Claude Desktop 说："帮我搜索飞书上关于项目进度的文档"

### 流程

```text
步骤 1｜Claude Desktop（Host）把用户的问题发给 AI 模型

步骤 2｜模型分析问题，发现需要搜索飞书文档
       → 模型说："我想调用 search_feishu_docs 工具，参数是'项目进度'"

步骤 3｜Host 确认这个调用是允许的（可能会弹窗问用户）

步骤 4｜Host 通过 MCP Client 向 feishu-mcp 发送 tools/call 请求
       → 请求内容：调用 search_feishu_docs，参数 { query: "项目进度" }

步骤 5｜feishu-mcp 收到请求，调用飞书开放平台的搜索 API

步骤 6｜飞书返回搜索结果

步骤 7｜feishu-mcp 把结果按 MCP 协议格式返回给 Host

步骤 8｜Host 把 tool 的结果回填给模型

步骤 9｜模型基于搜索结果，生成自然语言回答给用户
```

核心就一句话：模型并不是直接调飞书 API，它调用的是一个 ==MCP 标准化后的工具==，MCP Server 在幕后负责和飞书打交道。

---

## 第六部分：MCP 连接是怎么建立的？

前面看了 tool 怎么调用，但在调用之前，Host 和 MCP Server 之间还有一个建立连接的过程。

### 三个阶段

```text
阶段 1：初始化（Initialization）
  → 双方交换信息、协商能力

阶段 2：正常工作（Operation）
  → 开始真正的 tools/call、resources/read 等操作

阶段 3：关闭（Shutdown）
  → 任务结束，断开连接
```

### 初始化阶段具体做了什么？

1. **Client 发送 `initialize` 请求**，告诉 Server：
   - 我支持哪个版本的 MCP 协议
   - 我有哪些能力

2. **Server 返回响应**，告诉 Client：
   - 我支持哪个版本
   - 我能提供哪些能力（tools？resources？prompts？）

3. **Client 发送 `initialized` 通知**，意思是："协商完毕，可以开始工作了。"

为什么要有这一步？因为不是所有 Server 都提供全部能力，也不是所有 Client 都支持所有特性。先协商清楚，后面才不会乱。

一个关键认知：MCP 连接不是"发一次请求就结束"的，而是==一段有生命周期的会话==。先建立连接、协商能力，然后工作，最后关闭。

---

## 第七部分：消息通过什么方式传输？

MCP 定义了两种主要的传输方式。

### 1. stdio（标准输入输出）——本地场景

最常见的本地方式：

- Host 启动一个本地进程（MCP Server）
- 通过进程的 stdin/stdout 收发 JSON 消息

简单、不用开端口、适合桌面和命令行工具。

这里的 `JSON` 也可以先把它理解成：**机器之间交换的结构化文本消息**。  
对初学者来说，重点不是记格式细节，而是理解“stdio 模式下，Host 是通过本地进程的输入输出和 Server 对话的”。

很多 MCP 配置长这样：

```toml
[mcp_servers.feishu]
command = "feishu-mcp"
args = ["--stdio"]
```

### 2. Streamable HTTP——远程场景

适合 MCP Server 部署在远程服务器上：

- 用 HTTP 传输消息
- 支持会话管理
- 一个 Server 可以服务多个 Client

简单记：
- 本地工具 → 用 `stdio`
- 远程服务 → 用 `Streamable HTTP`

---

## 第八部分：MCP 和直接调 API 是什么关系？

直接给结论：MCP 不是为了消灭 API，而是在 API 之上加了一层标准化包装。

```text
外部系统（飞书、GitHub、数据库……）
  ↓
底层接口（各自的 API / 协议）
  ↓
MCP Server（把底层接口包装成统一格式）
  ↓
AI 应用（通过 MCP 协议统一调用）
```

- API 是能力来源——真正干活的还是飞书 API、GitHub API
- MCP 是接入标准——让 AI 应用不用关心底层 API 长什么样

什么时候该用 MCP？

- 需要接入多个外部系统
- 需要在不同 AI 应用间复用同一套能力
- 需要让模型能自主选择和调用工具

什么时候不需要？

- 只写一次性脚本
- 只接一个简单接口
- 不涉及模型调用工具

---

## 第九部分：常见误区

> [!warning] 常见误区
> | 误区 | 纠正 |
> |---|---|
> | MCP 是一种 AI 模型 | MCP 是协议，不是模型。模型负责思考，MCP 负责连接 |
> | 上了 MCP 模型就变聪明了 | MCP 只解决"连接"问题，模型选错工具、理解错结果，MCP 帮不了 |
> | MCP 替代了 API | MCP 底层仍然调 API，它是 API 之上的标准化层 |
> | MCP 就是 function calling 换个名字 | function calling 只是"模型调工具"这一个点；MCP 还包括连接管理、生命周期、能力协商、资源和模板等完整体系 |
> | 上了 MCP 就不用管安全 | MCP 提供了更好的结构，但权限控制、审批机制仍然需要 Host 来实现 |

---

## 总结

> [!abstract] 核心要点
> 1. MCP 是把外部能力标准化接入 AI 应用的协议。
> 2. 架构是 ==Host → Client → Server → 外部系统==，不只是一个"server"。
> 3. Server 提供三类能力：==Tools==（做事）、==Resources==（给材料）、==Prompts==（给模板）。
> 4. MCP 连接有完整生命周期——先协商再工作，不是发一次请求就完事。

---

## 参考资料

- MCP 官方首页：[What is MCP?](https://modelcontextprotocol.io/)
- MCP 官方架构概览：[Architecture](https://modelcontextprotocol.io/specification/draft/architecture)
- MCP 官方生命周期：[Lifecycle](https://modelcontextprotocol.io/specification/draft/basic/lifecycle)
- MCP 官方传输方式：[Transports](https://modelcontextprotocol.io/specification/2025-11-25/basic/transports)
- MCP 官方 tools 文档：[Tools](https://modelcontextprotocol.io/docs/concepts/tools)
- MCP 官方 resources 文档：[Resources](https://modelcontextprotocol.io/docs/concepts/resources)
- MCP 官方 prompts 文档：[Prompts](https://modelcontextprotocol.io/specification/draft/server/prompts)
