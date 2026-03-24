---
title: "Skills 资料整理"
tags:
  - skill
  - Claude-Code
  - AI-agent
date: 2026-03-24
---

# Skills 资料整理

这份文档基于 Anthropic/Claude 官方资料与 Agent Skills 开放规范，回答 5 个问题：

1. `skill` 到底是什么
2. 一个 `skill` 目录里通常有哪些东西
3. `skill` 是怎么被发现、加载、使用的
4. 怎么把 `skill` 设计得更好
5. 怎么安装和测试一个 `skill`

本文统一使用 `SKILL.md`（全大写）。Claude Code 官方文档和 Agent Skills 开放规范均使用这个写法。

## 1. 什么是 Skill

> [!info] 定义
> **Skill 就是一个带有 `SKILL.md` 的文件夹，框架能自动发现它、按需加载它。**

它和"一个普通 markdown 文件夹"的区别在于：

- `SKILL.md` 通常会带 YAML frontmatter；`name` 和 `description` 是最常见、也最推荐填写的字段
- 框架（如 Claude Code）会扫描这些文件，把可用于发现 skill 的元信息放进上下文
- 模型根据这些元信息判断要不要在当前任务中加载完整内容

所以 skill 不是"一段 prompt"，而是一个**可被框架发现、可按需加载、可携带脚本与资料的目录**。它的目标是把某个可重复工作流打包成模块，让 agent 在需要时再把相关内容拿进上下文。

官方把它概括成 3 个点：

- skill 是模块化能力（modular capabilities）
- 每个 skill 打包了元信息、说明、以及可选资源
- 它们是按需加载的，不是一直常驻在上下文里

常见用途：

- 某类文件处理流程，比如 PDF、Excel、Word
- 团队内部规范，比如品牌规范、文档模板、提交流程
- 某类固定操作，比如 code review、数据库迁移、报表生成
- 某个领域知识包，比如财务指标口径、内部 API 使用约束

## 2. 一个 Skill 目录下有哪些东西

一个 skill 至少是一个目录，里面至少有一个 `SKILL.md`：

```text
skill-name/
├── SKILL.md          # 必需：metadata + instructions
├── scripts/          # 可选：可执行脚本
├── references/       # 可选：补充说明/参考资料
├── assets/           # 可选：模板、图片、静态资源
└── ...               # 其他附加文件
```

### 2.1 `SKILL.md` 是核心

`SKILL.md` 通常由两部分组成：

1. YAML frontmatter（`---` 包裹的元信息；很多 skill 会写，但字段可以按需省略）
2. Markdown 正文（加载后进入上下文的具体指令）

最小可运行示例：

```markdown
---
name: hello-world
description: A minimal skill for testing. Use when the user says "test skill loading".
---

# Hello World Skill

When this skill is loaded, respond with: "Skill loaded successfully!"
```

把这个文件保存到 `.claude/skills/hello-world/SKILL.md`，然后在 Claude Code 里说 "test skill loading"，就能验证 skill 是否被正确发现和加载。

### 2.2 frontmatter 字段

最常用的两个字段：

- **`name`**
  - skill 名称；在 Claude Code 里，通常也会出现在对应的 slash 命令入口中
  - Agent Skills 规范要求和目录名一致
  - 推荐只用小写字母、数字、连字符
- **`description`**
  - 描述它"做什么"以及"什么时候该用"
  - **这是 skill 能不能被正确触发的==关键字段==**——模型就是根据这个字段决定是否加载

Claude Code 还支持的字段：

- `user-invocable`：设为 `false` 会把它从 `/` 菜单里隐藏；这主要影响菜单可见性，不等于彻底禁止通过 Skill tool 调用
- `disable-model-invocation`：设为 `true` 则只有用户能通过 `/` 命令触发，模型不会自动加载

Agent Skills 开放规范的可选字段：

- `license`
- `compatibility`——适合写清楚运行要求（只适用于 Claude Code、需要 `git`/`docker`、需要联网等）
- `metadata`
- `allowed-tools`（实验性）

### 2.3 `scripts/`：放"确定性执行"逻辑

如果某个步骤很脆弱、必须按固定参数执行、或者让模型临时现写代码风险太高，就应该放脚本。

适合脚本化的内容：

- 数据提取、校验、格式转换
- 迁移、批量处理
- 任何"让模型现场写很容易飘"的步骤

关键点：脚本通过 bash 直接执行，真正进入上下文的是"脚本输出"，而不是整个脚本源码。这比让模型现写一遍代码更省 token，也更稳定。

### 2.4 `references/`：放按需读取的长文档

如果说明很长，不适合全塞在 `SKILL.md` 里，就拆到 `references/` 里。适合放：

- API 细节、业务口径、领域术语
- 边界情况、大量示例

关键点不是"能不能放"，而是"只在需要时再读"。这就是 progressive disclosure。

### 2.5 `assets/`：放静态资源

`assets/` 里放不需要模型全文理解、但任务可能会用到的静态内容：模板、配置样例、图片、数据表、schema。

## 3. Skill 是怎么工作的

Skill 的加载是一个"分层按需"的过程。**重要区分：这个过程在不同实现中由不同角色完成。**

### 谁来做触发判断？

这取决于你用的框架：

| 框架 | 触发方式 |
|---|---|
| **Claude Code** | 框架会先把用于发现 skill 的元信息放进上下文。**模型自己决定**是否需要加载某个 skill；用户也可以通过 `/skill-name` 手动触发。skill 被触发后，Claude 再加载完整正文；如果 skill 还引用了其他文件，Claude 会继续按需读取。 |
| **自建 harness** | 你需要自己实现发现和加载机制。常见做法是提供一个 `load_skill` 工具，由模型调用。本仓库的 `skill_loading.md` 就是这个方案。 |
| **API + Agent SDK** | 通过 SDK 的 skill 加载 API，在代码层面控制哪些 skill 的内容被注入到对话中。 |

不管哪种实现，核心模式都是一样的三层：

### Level 1：metadata 进入发现上下文

在常规会话里，agent 会先拿到每个 skill 的发现信息，用于判断"有哪些 skill 可以用"。

通常这里至少包括 `description`；如果显式写了 `name`，也会一起参与发现。  
不过要注意：如果 skill 设了 `disable-model-invocation: true`，Claude Code 会把它从自动发现上下文里拿掉，等用户手动调用时再加载完整内容。

### Level 2：`SKILL.md` 正文按需加载

当用户任务和某个 skill 的 `description` 匹配时，才把 `SKILL.md` 正文加载进上下文。

- 平时不把全部 skill 正文塞进去
- 真的相关时才把具体说明读进来

### Level 3：脚本和资源进一步按需读取/执行

如果 `SKILL.md` 里引用了 `references/*.md`、`scripts/*.py`、`assets/*`，agent 只会在任务需要时继续去读这些文件或执行脚本。

这就是官方反复强调的 ==progressive disclosure==：

> 先只暴露足够让 agent 知道"要不要用"的元信息；真正细节在需要时才进入上下文。

好处：

- 降低 system prompt 体积
- 避免无关技能长期占用上下文
- 让一个 agent 可以安装很多 skill，但不至于上下文爆炸

## 4. 怎样把 Skill 设计得更好

### 4.1 核心设计决策：给 agent 多大的自由度

这是 skill 设计中最重要的一条判断标准，应该在写任何内容之前先想清楚。

| 任务特征 | 应该给的形式 | 自由度 |
|---|---|---|
| 有很多合理做法，需要 agent 根据上下文判断 | 文字步骤 | 高 |
| 有推荐套路，允许少量参数变化 | 伪代码或参数化脚本 | 中 |
| 非常脆弱，步骤必须固定，一改就错 | 明确脚本 + "不要改参数" | 低 |

很多 skill 写不好，本质上不是信息不够，而是**该给 agent 多大决策空间没有设计清楚**。

### 4.2 聚焦单一工作流，不要做"万能 skill"

官方明确建议：

- 一个 skill 解决一个特定、可重复的问题
- 多个小 skill 的组合，通常比一个超大 skill 更好

反例：`helper`、`utils`、`documents`——太泛，agent 不知道什么时候该触发。

更好的：`pdf-processing`、`writing-documentation`、`testing-code`、`analyzing-spreadsheets`。

### 4.3 `description` 要同时写"做什么"和"什么时候用"

这是 skill 能否被正确触发的核心。好的 `description` 需要同时包含能力范围、触发条件、相关关键词。

好例子：

> [!success] 好的 description
> ```yaml
> description: Extracts text and tables from PDF files, fills PDF forms, and merges multiple PDFs. Use when working with PDF documents or when the user mentions PDFs, forms, or document extraction.
> ```

差例子：

> [!failure] 差的 description
> ```yaml
> description: Helps with PDFs.
> ```

后者太短、没有触发条件、没有关键词，agent 很难判断是否该用。

### 4.4 `SKILL.md` 要短、清楚、像索引页

官方的建议不是"越详细越好"，而是：

- 要简洁、结构化，像总览页/导航页
- skill 一旦触发，`SKILL.md` 正文会整份进入上下文，每个 token 都在竞争上下文空间
- `SKILL.md` 里只放主流程、关键规则、导航链接
- 细节拆到 `references/`，执行逻辑拆到 `scripts/`
- 官方建议控制在大约 500 行以内

### 4.5 避免深层嵌套引用

不推荐：`SKILL.md -> advanced.md -> details.md`

推荐：`SKILL.md` 直接链接到所有补充文件

引用链越深，agent 越容易只看到局部或读不全。

### 4.6 长 reference 文件最好有目录

如果一个 reference 文件很长，在顶部给 table of contents，这样即使 agent 局部预览，也能先看到文件整体覆盖了哪些内容。

### 4.7 用脚本做确定性步骤

只要某一步经常重复、输出格式固定、对参数和顺序敏感、容易被模型"脑补"、或有明确输入输出——就更适合脚本化。收益：更稳定 + 更省上下文。

## 5. 安装和测试

### 5.1 Skill 放在哪里

在 Claude Code 中，skill 可以来自多个位置：

| 位置 | 作用域 |
|---|---|
| enterprise skills | 组织级，管理员下发 |
| `~/.claude/skills/` | 个人全局——所有项目共享 |
| `.claude/skills/` | 项目级别——仅当前项目可用 |
| `<plugin>/skills/` | 插件级，仅该插件启用时可用 |

同名 skill 在多个层级存在时，Claude Code 当前文档给出的优先级是：`enterprise > personal > project`。插件 skill 使用 `plugin-name:skill-name` 命名空间，不和其他层级直接冲突。

如果你在自建 harness 中实现 skill，需要自己决定扫描路径（本仓库的 `skill_loading.md` 使用 `skills/` 目录）。

### 5.2 一个完整的可测试示例

下面是一个最小但完整的 skill，可以直接用来验证 skill 机制是否正常工作：

**目录结构：**

```text
.claude/skills/
  greet/
    SKILL.md
```

**`.claude/skills/greet/SKILL.md`：**

```markdown
---
name: greet
description: Generate a greeting message in a specified language. Use when the user asks for a greeting, hello message, or welcome text in any language.
---

# Greet Skill

Generate a greeting for the user.

## Steps

1. Ask which language the user wants (if not specified, default to English)
2. Respond with a culturally appropriate greeting in that language
3. Include a brief phonetic guide if the language uses non-Latin script

## Examples

- "Say hello in Japanese" → こんにちは (Konnichiwa)
- "Greet me in French" → Bonjour !
```

**测试方式：**

1. 把上面的文件放到项目的 `.claude/skills/greet/SKILL.md`
2. 启动 Claude Code
3. 输入 "say hello in Japanese"——看 skill 是否被自动加载
4. 输入 `/greet`——看是否能通过 slash command 手动触发
5. 输入 "帮我写段代码"——确认不相关的请求不会误触发

### 5.3 触发测试清单

> [!todo] 触发测试清单
> skill 不是"写完即结束"，而是一个需要迭代的路由配置。

**安装前检查：**

- [ ] `SKILL.md` 是否清晰
- [ ] `description` 是否真的反映触发场景
- [ ] 所有被引用文件是否存在

**安装后测试：**

- [ ] 用几种不同说法测试触发（"处理 PDF" / "提取表格" / "读取这个文档"）
- [ ] 检查 Claude 是否真的加载了 skill（观察行为是否符合 skill 中的流程）
- [ ] 如果触发不稳定，优先改 `description`
- [ ] 用不相关的请求测试是否会误触发

## 6. 一个更完整的 Skill 目录示例

如果你要做一个 `pdf-processing` skill：

```text
pdf-processing/
├── SKILL.md
├── references/
│   ├── forms.md
│   ├── reference.md
│   └── examples.md
├── scripts/
│   ├── extract_text.py
│   ├── fill_form.py
│   └── validate_pdf.py
└── assets/
    └── output-template.md
```

各部分职责：

- **`SKILL.md`**——只写任务范围、默认流程、什么时候去看哪个文件
- **`references/forms.md`**——专门放表单填写规则
- **`references/reference.md`**——放 API/命令/约束
- **`references/examples.md`**——放高质量案例
- **`scripts/*.py`**——放提取、校验、转换这类确定性步骤
- **`assets/`**——放模板或静态资源

## 7. 讲解时最值得强调的结论

> [!abstract] 核心结论
> 1. Skill 就是一个带 `SKILL.md` 的文件夹——和普通文件夹的区别是 frontmatter 让框架能自动发现和加载它。
> 2. `description` 决定"何时触发"，正文决定"具体怎么做"。
> 3. 设计 skill 的第一个问题不是"写什么内容"，而是=="给 agent 多大自由度"==。
> 4. 长文档拆到 `references/`，脆弱步骤进 `scripts/`，`SKILL.md` 本身要短。
> 5. Skill 需要测试——不只测"能不能触发"，还要测"不该触发时会不会误触发"。
> 6. 好的 skill 重点不是"内容越多越好"，而是=="触发准确、结构清晰、按需加载"==。

## 8. 参考资料

官方资料：

- Claude Code Docs: [Extend Claude with skills](https://code.claude.com/docs/en/skills)
- Claude Help Center: [Use Skills in Claude](https://support.claude.com/en/articles/12512180-use-skills-in-claude)
- Claude API Docs: [Agent Skills Overview](https://platform.claude.com/docs/en/agents-and-tools/agent-skills/overview)
- Claude API Docs: [Skill authoring best practices](https://platform.claude.com/docs/en/agents-and-tools/agent-skills/best-practices)

开放规范：

- Agent Skills: [Specification](https://agentskills.io/specification)
- GitHub: [agentskills/agentskills](https://github.com/agentskills/agentskills)
- GitHub: [anthropics/skills](https://github.com/anthropics/skills)（Anthropic 官方 skill 示例）

---

## 附录：在本仓库中的对应关系

本仓库的 [[learn_claude_code_sample/skill_loading|skill_loading.md]] 用自建 harness 实现了同样的分层加载机制：

| 官方 Skill 机制 | 本仓库实现 |
|---|---|
| 框架启动时扫描 skill 目录 | `SkillLoader` 扫描 `skills/*/SKILL.md` |
| `name` + `description` 进 system prompt | 技能目录写入 system prompt |
| 模型用文件系统工具读取正文 | 模型调用 `load_skill` 工具 |
| 正文进入上下文 | 正文作为 `tool_result` 进入 `messages` |

两者本质一致：启动时只暴露元信息 → 触发时拉入正文 → 细节资源继续按需读取。
