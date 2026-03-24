---
title: "Skill Loading"
tags:
  - skill-loading
  - 上下文管理
  - harness
date: 2026-03-24
---

# Skill Loading

> [!question] 核心问题
> 把所有知识都塞进 system prompt，会越来越贵

在已经具备 [[learn_claude_code_sample/subagent|子代理]]、[[learn_claude_code_sample/todo_write|计划状态]]和基础工具之后，agent 已经可以：

- 调工具操作外部世界
- 用 `todo` 维护自己的执行状态
- 用 `task` 把局部探索隔离到子代理里

但还有一个没有解决的问题：**领域知识应该放在哪里？**

如果把所有技能文档全文都直接塞进 system prompt，会立刻遇到三个问题：

1. system prompt 体积迅速膨胀
2. 大部分技能在当前任务里根本用不上，却长期占用上下文
3. 模型每一轮都要重复读这些无关知识，成本很高

这里的解决方式不是"提前把所有知识注入"，而是把技能拆成两层：

| | Layer 1：技能目录 | Layer 2：技能正文 |
|---|---|---|
| 放在哪 | system prompt | tool_result（按需） |
| 信息量 | 小——只有名字和一句话描述 | 大——完整的技能文档 |
| 什么时候出现 | 每一轮都在 | 模型调用 `load_skill` 后才进入上下文 |

这一章的核心思想可以概括成一句话：==让知识可发现，但不要让知识全文常驻。==

这和前面的上下文隔离机制形成了很自然的对照：

- 前面的上下文隔离机制是把"中间探索过程"隔离到子上下文——**减少**上下文里不必要的内容
- 这里是把"领域知识正文"延迟到真正需要时才注入——**控制**上下文里知识的进入时机

两章都在做同一件事：**管理上下文里长期常驻的内容，只保留真正必要的部分。**

## 技能从哪里来？

所有技能都放在本地 `skills/` 目录下：

```text
skills/
  agent-builder/
    SKILL.md
  code-review/
    SKILL.md
  mcp-builder/
    SKILL.md
  pdf/
    SKILL.md
```

每个 `SKILL.md` 的结构都是 **frontmatter（元信息） + 正文**：

```markdown
---
name: code-review
description: Perform thorough code reviews with security, performance, and maintainability analysis.
---

# Code Review Skill

You now have expertise in conducting comprehensive code reviews.
Follow this structured approach:

## Review Checklist
### 1. Security (Critical)
...
```

`SkillLoader` 在 agent 启动时扫描所有 `SKILL.md`，把每个技能拆成两部分存到内存里：

- **元信息**（name、description）→ 用来生成 Layer 1 的技能目录
- **正文**（frontmatter 之后的全部内容）→ 用来在 `load_skill` 时返回 Layer 2

## 模型一开始能看到什么？

### System Prompt：只放技能目录，不放全文

system prompt 中的技能部分展开后，模型看到的效果是：

```text
You are a coding agent at /Users/chenyuhan/My/learn-claude-code.
Use load_skill to access specialized knowledge before tackling unfamiliar topics.

Skills available:
  - code-review: Perform thorough code reviews with security, performance, and maintainability analysis.
  - mcp-builder: Build MCP servers that give Claude new capabilities.
  - pdf: Process PDF files - extract text, create PDFs, merge documents.
```

模型此时只知道"有哪些技能"和"大概是干什么的"，并没有拿到任何技能的完整正文。

### 工具声明：多了一个 `load_skill`

```json
{
  "name": "load_skill",
  "description": "Load specialized knowledge by name.",
  "input_schema": {
    "type": "object",
    "properties": {
      "name": { "type": "string", "description": "Skill name to load" }
    },
    "required": ["name"]
  }
}
```

模型拥有了一个新能力：当它觉得当前任务需要某种专门知识时，可以主动把那份知识拉进上下文。

## `load_skill` 实际返回什么？

工具分发表只新增一行：

```python
"load_skill": lambda **kw: SKILL_LOADER.get_content(kw["name"]),
```

`get_content()` 的实现很直接：

```python
def get_content(self, name: str) -> str:
    skill = self.skills.get(name)
    if not skill:
        return f"Error: Unknown skill '{name}'. Available: {', '.join(self.skills.keys())}"
    return f"<skill name=\"{name}\">\n{skill['body']}\n</skill>"
```

返回的不是一句摘要，而是用 `<skill>` 标签包裹的完整正文：

```xml
<skill name="code-review">
# Code Review Skill

You now have expertise in conducting comprehensive code reviews.
...完整正文...
</skill>
```

用 XML 标签包裹是为了让模型能清楚区分技能正文和其他 `tool_result` 内容的边界。

这段文本随后作为 `tool_result` 进入 `messages`，从下一轮开始就成为模型的上下文知识。

## 例子：先加载 `code-review`，再回答技能重点

用户输入：

```text
I need to do a code review. Load the relevant skill first, then briefly tell me what that skill focuses on.
```

### Round 1：模型先调用 `load_skill`

模型根据 system prompt 里的技能目录判断当前任务需要 `code-review`，于是主动加载：

```json
{
  "stop_reason": "tool_use",
  "content": [
    {
      "type": "tool_use",
      "id": "toolu_01G9EUDmSRM2492od65KFB1j",
      "name": "load_skill",
      "input": { "name": "code-review" }
    }
  ]
}
```

回填结果是完整技能正文：

```json
{
  "role": "user",
  "content": [
    {
      "type": "tool_result",
      "tool_use_id": "toolu_01G9EUDmSRM2492od65KFB1j",
      "content": "<skill name=\"code-review\">\n# Code Review Skill\n\nYou now have expertise in conducting comprehensive code reviews.\n...\n</skill>"
    }
  ]
}
```

**完整技能正文第一次出现在 `messages` 里。** 在此之前，模型只看过 system prompt 里那一行简短描述。

> [!important] 关键机制
> 模型的总结建立在上一轮刚刚注入的技能正文之上，而不是建立在 system prompt 那几行短描述之上。

### Round 2：模型带着技能正文继续回答

第二轮的上下文形态：

```json
{
  "system": "... 包含技能目录，不包含正文 ...",
  "tools": ["... 包含 load_skill ..."],
  "messages": [
    { "role": "user", "content": "I need to do a code review. Load the relevant skill first, then briefly tell me what that skill focuses on." },
    { "role": "assistant", "content": [{ "type": "tool_use", "name": "load_skill", "input": { "name": "code-review" } }] },
    { "role": "user", "content": [{ "type": "tool_result", "content": "<skill name=\"code-review\"> ... full skill body ... </skill>" }] }
  ]
}
```

模型现在带着完整技能正文推理，返回：

```json
{
  "stop_reason": "end_turn",
  "content": [
    {
      "type": "text",
      "text": "Loaded code-review skill. Focus areas: security, correctness, performance, maintainability, and testing."
    }
  ]
}
```

这里的关键不是"模型会总结"，而是：**模型的总结建立在上一轮刚刚注入的技能正文之上，而不是建立在 system prompt 那几行短描述之上。**

实际场景中，模型加载技能后会在后续轮次中基于这些知识执行具体任务（如逐项 review 代码）。这里为了展示机制，只用了一个简短的例子。

## 全局关系

把整章的机制压缩来看：

```
agent 启动
    → SkillLoader 扫描 skills/*/SKILL.md
        → 拆成元信息（name, description）和正文（body）

每轮请求
    → system prompt 携带技能目录（Layer 1：名字 + 一句话描述）

当模型判断需要某项专门知识
    → 调用 load_skill(name)
        → harness 返回 <skill name="...">正文</skill>（Layer 2）
            → 正文进入 messages
                → 下一轮模型带着新知识继续推理
```

这套机制的核心收益：

- 大部分无关知识不必长期常驻
- 真正需要的知识可以按需拉入
- 知识注入仍然通过标准的 `tool_use → tool_result` 协议完成

## 小结

> [!abstract] 核心要点
> 这里新增了三件关键事情：
>
> - **`SkillLoader`**：扫描 `skills/` 目录，把技能拆成元信息和正文
> - **`load_skill` 工具**：让模型可以主动请求某个技能的完整内容
> - **两层知识注入**：system prompt 里常驻目录，正文通过 `tool_result` 按需进入上下文
>
> 核心转变：从”把探索过程隔离出去”，到==把领域知识延迟注入，只在需要时扩展上下文==。
