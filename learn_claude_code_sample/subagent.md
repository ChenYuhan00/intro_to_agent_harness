---
title: "Subagent"
tags:
  - subagent
  - 上下文隔离
  - harness
date: 2026-03-24
---

# Subagent

> [!question] 核心问题
> 任务一复杂，单个上下文会越来越乱

在已经引入 [[learn_claude_code_sample/todo_write|计划状态]] 之后，模型已经能一边执行任务，一边维护计划。但还有一个更深的问题没有解决：**所有思考、所有探索、所有中间结果，仍然堆在同一个 `messages` 里。**

当任务只是“读一个文件、改一个函数”时，这还不是大问题。但如果主任务里包含一个独立的探索子任务，比如：

- 先去读几份文件，归纳结论
- 再回来继续主线实现
- 中间过程不一定都值得保留在主上下文里

如果这些探索都发生在父代理自己的 `messages` 里，就会出现两个问题：

1. 主上下文被大量中间试探、读取结果、临时想法污染
2. 主任务只需要一个结论，但却被迫携带整段探索历史

这里的解决方式是：**把子任务交给一个新的 agent，在全新的上下文里做，做完只把最终摘要带回来。**

> [!tip] 关键认知
> 隔离的不是"世界"，而是"上下文"——父子代理共享文件系统，但不共享 `messages`。

## `task` 和之前的工具有什么不同？

在前面的实现里，模型调用的都是“直接作用于当前世界”的工具：

- `read_file` 直接读文件
- `write_file` 直接写文件
- `todo` 直接改当前 agent 的计划状态

这些工具虽然做的事不同，但有一个共同点：**执行结果直接回到当前 agent 自己的上下文里。**

`task` 不一样。它不是让父代理“直接做一件事”，而是让父代理**启动另一个 agent 去做**。

也就是说：

| | `read_file` / `write_file` / `todo` | `task` |
|---|---|---|
| 调用后发生什么 | 当前 agent 直接执行工具 | 当前 agent 启动一个子代理 |
| 工具结果是什么 | 直接返回某个操作结果 | 返回子代理最终摘要 |
| 中间过程留在哪 | 留在当前上下文 | 留在子代理自己的上下文 |
| 对父上下文的影响 | 中间轨迹全部进入父 messages | 只带回最后的总结 |

这就是这里的核心：**不是共享同一个思考过程，而是把思考过程隔离出去。**

## 子代理的本质：新的 `messages=[]`

最关键的实现只在 `run_subagent()` 里：

```python
def run_subagent(prompt: str) -> str:
    sub_messages = [{"role": "user", "content": prompt}]  # fresh context
    for _ in range(30):
        response = client.messages.create(
            model=MODEL, system=SUBAGENT_SYSTEM, messages=sub_messages,
            tools=CHILD_TOOLS, max_tokens=8000,
        )
        sub_messages.append({"role": "assistant", "content": response.content})
        if response.stop_reason != "tool_use":
            break
        results = []
        for block in response.content:
            if block.type == "tool_use":
                handler = TOOL_HANDLERS.get(block.name)
                output = handler(**block.input) if handler else f"Unknown tool: {block.name}"
                results.append({"type": "tool_result", "tool_use_id": block.id, "content": str(output)[:50000]})
        sub_messages.append({"role": "user", "content": results})
    return "".join(b.text for b in response.content if hasattr(b, "text")) or "(no summary)"
```

关键点只有两个：

1. `sub_messages` 是一个全新的列表，父代理原来的历史完全不会传进去
2. 返回给父代理的不是子代理完整轨迹，而只是最后那段文本摘要

所以这里的上下文隔离不是靠“删消息”实现的，而是靠**从一开始就不给子代理父历史**实现的。

## 为什么说是“上下文隔离”，不是“文件系统隔离”？

很多人第一次看到子代理，会误以为它是一个完全隔离的环境。其实不是。

这里父代理和子代理：

- **共享同一个工作目录**
- **共享同一套文件工具实现**
- **共享同一个文件系统**

但它们：

- **不共享 `messages`**
- **不共享对子任务的中间推理轨迹**

这从代码里也能直接看出来：

```python
WORKDIR = Path.cwd()

def safe_path(p: str) -> Path:
    path = (WORKDIR / p).resolve()
    ...

TOOL_HANDLERS = {
    "bash":       lambda **kw: run_bash(kw["command"]),
    "read_file":  lambda **kw: run_read(kw["path"], kw.get("limit")),
    "write_file": lambda **kw: run_write(kw["path"], kw["content"]),
    "edit_file":  lambda **kw: run_edit(kw["path"], kw["old_text"], kw["new_text"]),
}
```

父子代理最终都落到同一个 `WORKDIR` 和同一组工具实现上，所以它们操作的是同一份代码仓库。  
隔离的不是“世界”，而是“上下文”。

## 父代理和子代理分别看到什么？

### 父代理：多了一个 `task` 工具

父代理看到的新增工具声明是：

```json
{
  "name": "task",
  "description": "Spawn a subagent with fresh context. It shares the filesystem but not conversation history.",
  "input_schema": {
    "type": "object",
    "properties": {
      "prompt": {
        "type": "string"
      },
      "description": {
        "type": "string",
        "description": "Short description of the task"
      }
    },
    "required": [
      "prompt"
    ]
  }
}
```

父代理的 system prompt 也明确告诉它可以委托子任务：

```text
You are a coding agent at /Users/chenyuhan/My/learn-claude-code. Use the task tool to delegate exploration or subtasks.
```

### 子代理：只有基础工具，没有 `task`

子代理不是父代理的完整复制。它看到的是：

```json
{
  "system": "You are a coding subagent at /Users/chenyuhan/My/learn-claude-code. Complete the given task, then summarize your findings.",
  "tools": [
    { "name": "bash", "...": "..." },
    { "name": "read_file", "...": "..." },
    { "name": "write_file", "...": "..." },
    { "name": "edit_file", "...": "..." }
  ]
}
```

注意这里**没有 `task`**。这意味着：

- 子代理不能继续递归地产生孙代理
- 委托层级被硬性限制在一层

代码里这件事是通过 `CHILD_TOOLS` 和 `PARENT_TOOLS` 的差异实现的：

```python
CHILD_TOOLS = [
    {"name": "bash", ...},
    {"name": "read_file", ...},
    {"name": "write_file", ...},
    {"name": "edit_file", ...},
]

PARENT_TOOLS = CHILD_TOOLS + [
    {"name": "task", ...},
]
```

## 父代理如何调用子代理？

父代理在主循环里看到 `task` 时，不走普通工具分发，而是单独进入 `run_subagent()`：

```python
for block in response.content:
    if block.type == "tool_use":
        if block.name == "task":
            output = run_subagent(block.input["prompt"])
        else:
            handler = TOOL_HANDLERS.get(block.name)
            output = handler(**block.input) if handler else f"Unknown tool: {block.name}"
        results.append({"type": "tool_result", "tool_use_id": block.id, "content": str(output)})
```

也就是说，`task` 的执行结果不是某个文件内容，也不是某条命令输出，而是：

**子代理跑完整个自己的 loop 后，返回的最终总结文本。**

## 例子：父代理委托“读 requirements.txt”

用户输入：

```text
Use a subtask to read requirements.txt and tell me what Python dependencies this repo declares.
```

这个例子里会同时出现两条上下文线：

- 父代理的上下文
- 子代理的上下文

两条线共享文件系统，但消息历史彼此独立。

### Parent Round 1：父代理决定委托

父代理响应：

```json
{
  "role": "assistant",
  "stop_reason": "tool_use",
  "content": [
    {
      "type": "tool_use",
      "id": "toolu_parent_task_1",
      "name": "task",
      "input": {
        "description": "Read requirements",
        "prompt": "Read requirements.txt and summarize the Python dependencies declared in it."
      }
    }
  ]
}
```

此时父代理不会自己去读 `requirements.txt`。它把任务打包成一个新的 prompt，交给子代理。

### Child Round 1：子代理在全新上下文中启动

子代理收到的第一轮请求，其实只有一条新的 user 消息：

```json
{
  "system": "You are a coding subagent at /Users/chenyuhan/My/learn-claude-code. Complete the given task, then summarize your findings.",
  "tools": [
    { "name": "bash", "...": "..." },
    { "name": "read_file", "...": "..." },
    { "name": "write_file", "...": "..." },
    { "name": "edit_file", "...": "..." }
  ],
  "messages": [
    {
      "role": "user",
      "content": "Read requirements.txt and summarize the Python dependencies declared in it."
    }
  ]
}
```

注意，这里没有父代理之前的用户原话，也没有父代理此前的任何历史。  
子代理只知道自己当前要做什么，不知道父代理之前聊了什么。

### Child Round 1：子代理调用 `read_file`

```json
{
  "role": "assistant",
  "stop_reason": "tool_use",
  "content": [
    {
      "type": "tool_use",
      "id": "toolu_child_read_1",
      "name": "read_file",
      "input": {
        "path": "requirements.txt"
      }
    }
  ]
}
```

回填结果：

```json
{
  "role": "user",
  "content": [
    {
      "type": "tool_result",
      "tool_use_id": "toolu_child_read_1",
      "content": "anthropic>=0.25.0\npython-dotenv>=1.0.0"
    }
  ]
}
```

这段读文件的中间轨迹只存在于子代理自己的 `sub_messages` 里。

### Child Round 2：子代理生成最终摘要

```json
{
  "role": "assistant",
  "stop_reason": "end_turn",
  "content": [
    {
      "type": "text",
      "text": "The repo declares two Python dependencies: anthropic>=0.25.0 and python-dotenv>=1.0.0."
    }
  ]
}
```

`run_subagent()` 最终返回给父代理的，只是这段文本：

```text
The repo declares two Python dependencies: anthropic>=0.25.0 and python-dotenv>=1.0.0.
```

子代理内部的 `tool_use -> tool_result -> end_turn` 全部到此为止，不会原样塞回父上下文。

### Parent Round 1 恢复：父代理收到子代理摘要

父代理收到的 `task` 工具结果是：

```json
{
  "role": "user",
  "content": [
    {
      "type": "tool_result",
      "tool_use_id": "toolu_parent_task_1",
      "content": "The repo declares two Python dependencies: anthropic>=0.25.0 and python-dotenv>=1.0.0."
    }
  ]
}
```

从父代理视角看，子任务就像一个“返回摘要字符串的工具”。

### Parent Round 2：父代理给出最终回答

父代理随后基于这个摘要继续自己的主循环，并返回最终答案：

```json
{
  "role": "assistant",
  "stop_reason": "end_turn",
  "content": [
    {
      "type": "text",
      "text": "The Python dependencies declared are anthropic>=0.25.0 and python-dotenv>=1.0.0."
    }
  ]
}
```

## 全局关系

把整个调用链压缩来看：

```
父代理收到用户任务
    → 调用 task
        → 启动子代理（fresh messages）
            → 子代理自己跑完整个 tool loop
                → 子代理返回最终摘要
        → 父代理把摘要当作 tool_result 接回主上下文
            → 父代理继续主任务并结束
```

这里的关键收益不是“并行”或“更快”，而是：

- 主上下文保持干净
- 探索过程被局部化
- 父代理只消费最需要的结论

## 小结

> [!abstract] 核心要点
> 这里新增了三件关键事情：
>
> - **`task` 工具**：让父代理可以把某段子任务委托出去
> - **`run_subagent()`**：用全新的 `sub_messages` 启动子代理
> - **父子工具集分离**：父代理有 `task`，子代理没有 `task`
>
> 核心转变：从”模型维护自己的执行状态”，到==模型把局部探索过程隔离到另一个上下文里，只带回摘要==。
