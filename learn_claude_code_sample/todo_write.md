---
title: "TodoWrite"
tags:
  - todo
  - 计划状态
  - harness
date: 2026-03-24
---

# TodoWrite

> [!question] 核心问题
> 任务变长之后，模型会"失忆"

在前面的例子里，任务都很短——2 到 4 轮就结束了。但如果任务有 10 个步骤呢？

没有显式计划时，模型只能靠 messages 里不断增长的历史来"记住"自己做过什么。当上下文变长，它可能：

- 做到第 5 步时重复执行第 3 步
- 跳过某个步骤，直接进入后续环节
- 做完一步后不知道下一步是什么，开始发散

对 harness 来说也有问题：你只能看到一连串工具调用，但看不出模型**认为自己在执行哪一步**、整体进度到了哪里。

这里的解决方式不是把规划逻辑写死在代码里，而是给模型一个 `todo` 工具，让它自己维护计划状态。这和前面的文件工具有本质区别：前面的工具（read_file、write_file、edit_file）都是**操作外部世界**的，而 `todo` 是==操作模型自身的执行状态==。

## `todo` 和之前的工具有什么不同？

在前面的实现里，所有文件工具背后都是一个**纯函数**：`run_read(path)` 读完文件就结束了，`run_bash(command)` 执行完命令就结束了。函数本身不记住任何东西——每次调用都是独立的。

`todo` 不一样。它背后是一个**常驻内存的 Python class 实例**：

```python
TODO = TodoManager()    # 在 agent 启动时创建，整个生命周期只有这一个实例

class TodoManager:
    def __init__(self):
        self.items = []   # 状态在这里，跨轮次持久存在

    def update(self, items: list) -> str:
        # 校验 + 覆盖 self.items + 渲染结果
        ...
```

区别一目了然：

| | 前面的工具（bash, read_file 等） | `todo` |
|---|---|---|
| 背后实现 | 纯函数，无状态 | class 实例，有状态 |
| 每次调用 | 独立执行，互不影响 | 读写同一个 `self.items`，后一次覆盖前一次 |
| 状态存在哪 | 不存在——结果直接回填就完了 | 在 `TodoManager` 实例的内存里 |

这意味着模型第 1 轮写的计划、第 3 轮的更新、第 5 轮的收尾，操作的是**同一个对象**。`TodoManager` 在整个 agent 生命周期中持续存在，记住模型上一次写了什么。

### 工具声明

模型看到的工具声明：

```json
{
  "name": "todo",
  "description": "Update task list. Track progress on multi-step tasks.",
  "input_schema": {
    "type": "object",
    "properties": {
      "items": {
        "type": "array",
        "items": {
          "type": "object",
          "properties": {
            "id": { "type": "string" },
            "text": { "type": "string" },
            "status": { "type": "string", "enum": ["pending", "in_progress", "completed"] }
          },
          "required": ["id", "text", "status"]
        }
      }
    },
    "required": ["items"]
  }
}
```

翻译成人话：模型每次调用 `todo`，传入一个任务列表，每个任务有 id、描述文本、状态（pending / in_progress / completed）。

同时 system prompt 加了一句引导：

```
Use the todo tool to plan multi-step tasks. Mark in_progress before starting, completed when done.
```

工具分发表只新增一行：

```python
"todo": lambda **kw: TODO.update(kw["items"]),
```

注意这里调用的是 `TODO.update()`——一个实例方法，不是独立函数。

### `TodoManager` 做了什么？

`TodoManager.update()` 每次被调用时做两件事：

**第一，校验。** 它不接受任意输入：
- 最多 20 个任务
- 每个任务必须有 `id`、`text`、`status`
- `status` 只能是 `pending`、`in_progress`、`completed`
- 同时最多只能有一个任务处于 `in_progress`

不满足约束就报错——模型会在 tool_result 里看到错误信息，下一轮可以修正。

**第二，渲染。** 校验通过后，把任务列表渲染成可读文本，作为 `tool_result` 回填给模型：

```text
[>] #1: Create demo_todo_small/hello.py with hello() function
[ ] #2: Verify the file by reading it

(0/2 completed)
```

这段文本进入 messages 后，计划就不再是模型脑中的隐含意图，而是上下文里显式可见的状态。下一轮模型可以直接看着这个计划决定下一步做什么。

## 例子：先规划，再执行，再推进

用户输入：

```text
Create demo_todo_small/hello.py with a hello() function returning "hi". Use todos and verify the file by reading it.
```

### Round 1：模型先写计划

模型没有直接动手写文件，而是先调用 `todo` 建立计划：

```json
{
  "stop_reason": "tool_use",
  "content": [
    {
      "type": "tool_use",
      "id": "toolu_01DvrrRm9qMRsqds3ZQrozG7",
      "name": "todo",
      "input": {
        "items": [
          { "id": "1", "status": "in_progress", "text": "Create demo_todo_small/hello.py with hello() function" },
          { "id": "2", "status": "pending", "text": "Verify the file by reading it" }
        ]
      }
    }
  ]
}
```

回填结果是渲染后的计划文本：

```json
{
  "role": "user",
  "content": [
    {
      "type": "tool_result",
      "tool_use_id": "toolu_01DvrrRm9qMRsqds3ZQrozG7",
      "content": "[>] #1: Create demo_todo_small/hello.py with hello() function\n[ ] #2: Verify the file by reading it\n\n(0/2 completed)"
    }
  ]
}
```

从这一刻起，计划成为上下文的一部分。

### Round 2：执行当前任务

模型开始执行 `in_progress` 的任务 #1：

```json
{
  "stop_reason": "tool_use",
  "content": [
    {
      "type": "tool_use",
      "name": "write_file",
      "input": { "path": "demo_todo_small/hello.py", "content": "def hello():\n    return \"hi\"\n" }
    }
  ]
}
```

回填 `"Wrote 30 bytes"`。

### Round 3：推进计划状态

这一轮最能体现这里的核心——模型不只是执行动作，还主动把计划往前推：

```json
{
  "stop_reason": "tool_use",
  "content": [
    {
      "type": "tool_use",
      "name": "todo",
      "input": {
        "items": [
          { "id": "1", "status": "completed", "text": "Create demo_todo_small/hello.py with hello() function" },
          { "id": "2", "status": "in_progress", "text": "Verify the file by reading it" }
        ]
      }
    }
  ]
}
```

回填结果变成：

```text
[x] #1: Create demo_todo_small/hello.py with hello() function
[>] #2: Verify the file by reading it

(1/2 completed)
```

### Round 4-5：验证并结束

模型调用 `read_file` 验证文件内容，再次调用 `todo` 把任务 #2 标为 completed，最后返回 `end_turn`。

### 回顾：5 轮调用链

| Round | 工具 | 意图 |
|-------|------|------|
| 1 | `todo` | 建立计划，标记任务 #1 为 in_progress |
| 2 | `write_file` | 执行任务 #1：创建文件 |
| 3 | `todo` | 推进计划：#1 completed，#2 in_progress |
| 4 | `read_file` | 执行任务 #2：验证文件 |
| 5 | `todo` + end_turn | 标记 #2 completed，任务结束 |

注意 Round 1、3、5 都是 `todo` 调用——模型在**执行动作**和**维护计划**之间交替进行。

## 补充机制：当模型忘记更新计划

上面的例子里模型很自觉地维护了计划。但如果它连续好几轮只顾着执行工具、忘了更新 todo 怎么办？

这里在循环里加了一个计数器：如果连续 3 轮没有调用 `todo`，就在回填结果前插入一条提醒：

```python
rounds_since_todo = 0 if used_todo else rounds_since_todo + 1
if rounds_since_todo >= 3:
    results.insert(0, {"type": "text", "text": "<reminder>Update your todos.</reminder>"})
```

注入后，那一轮的 `user` 消息会变成：

```json
{
  "role": "user",
  "content": [
    { "type": "text", "text": "<reminder>Update your todos.</reminder>" },
    { "type": "tool_result", "tool_use_id": "...", "content": "..." }
  ]
}
```

这是一种很轻的外部控制：
- 提醒不替代 `tool_result`，而是和它一起进入上下文
- 提醒不改变工具协议，只是额外提供一条信号
- harness 不替模型规划路径，只在它太久没更新时提醒一次

## 全局关系

所有零件的协作方式：

```
模型看到 TOOLS（包括 todo）
    → 调用 todo，写入当前任务列表
        → TodoManager 校验并渲染
            → 渲染结果以 tool_result 回填进 messages
                → 下一轮模型在"可见计划"上继续决策

若连续 3 轮未调用 todo：
    → harness 插入 <reminder>
        → 提醒进入 messages
            → 促使模型重新更新计划
```

## 小结

> [!abstract] 核心要点
> 这一版新增了三样东西：
>
> - **`todo` 工具**：让模型把执行计划显式写入上下文
> - **`TodoManager`**：对计划状态做结构化约束和可读渲染
> - **Reminder 注入**：连续 3 轮不更新计划时，harness 插入轻量提醒
>
> 核心转变：从”模型只操作外部世界”，到==模型同时操作外部世界和自身执行状态==。
