---
title: "Background Tasks"
tags:
  - background-tasks
  - 异步
  - harness
date: 2026-03-24
---

# Background Tasks

> [!question] 核心问题
> 有些工作很慢，但主循环不应该一直等

在已经具备 [[learn_claude_code_sample/skill_loading|技能加载]]、[[learn_claude_code_sample/subagent|子代理]]和 [[learn_claude_code_sample/todo_write|计划状态]] 之后，agent 已经能按需加载知识、委托子代理、维护计划状态。但还有一种情况没有被很好处理：

**某个步骤本身非常耗时，但在它运行时，主代理其实还可以继续做别的事。**

典型例子包括：

- 跑一个需要几十秒甚至几分钟的 shell 命令
- 等待某个生成过程完成
- 启动一个异步任务后，先去做前台整理和验证工作

如果仍然只用 `bash` 或 `task` 来做，这两种方式都会阻塞：

- `bash` 会卡在当前这次工具调用里，直到命令执行完成
- `task` 虽然把工作交给了子代理，但父代理仍然要等待子代理跑完整个 loop 才能拿到结果

这里要解决的就是这个问题：
**让一部分命令在后台继续跑，而主代理可以先返回主循环，继续做别的前台工作。**

## `background_run` 和 `task` 有什么根本区别？

第一次看后台任务，容易和前面的 `task` 混在一起。但两者解决的问题完全不同：

| | `task` | `background_run` |
|---|---|---|
| 并发对象 | 一个新的子代理 | 一个后台线程中的命令 |
| 父代理是否等待 | 会，直到子代理返回摘要 | 不会，拿到 task_id 就继续 |
| 隔离的是什么 | 子代理上下文 | 执行时长 |
| 返回给父代理的第一次结果 | 子代理最终摘要 | "后台任务已启动" |
| 真正结果何时进入主上下文 | 子代理结束当下 | 下一轮模型调用前注入 |

一句话总结：`task` 解决的是=="上下文隔离"==，`background_run` 解决的是=="长耗时执行不阻塞主循环"==。

## 后台任务的本质：主线程继续走，后台线程慢慢跑

这里最核心的对象是 `BackgroundManager`，和前面用于管理计划状态的 `TodoManager` 一样是一个常驻内存的 class 实例：

```python
class BackgroundManager:
    def __init__(self):
        self.tasks = {}                # task_id -> {status, result, command}
        self._notification_queue = []  # 已完成但尚未通知模型的结果
        self._lock = threading.Lock()  # 保护通知队列的并发读写

    def run(self, command: str) -> str:
        task_id = str(uuid.uuid4())[:8]
        self.tasks[task_id] = {"status": "running", "result": None, "command": command}
        thread = threading.Thread(
            target=self._execute, args=(task_id, command), daemon=True
        )
        thread.start()
        return f"Background task {task_id} started: {command[:80]}"
```

`run()` 做的事情非常克制：生成 task_id → 记录状态 → 启动后台线程 → **立刻返回"任务已启动"**。

模型第一次拿到的不是命令输出，而只是一个启动确认。

后台线程真正执行命令后，做两件事：

1. 更新 `tasks[task_id]` 的最终状态和结果
2. 往 `_notification_queue` 里追加一条"待通知模型"的记录

> [!important] 后台结果注入时机
> 后台线程不会直接改 `messages`。它只负责把结果放进队列，真正送进上下文要等主循环下次开始时统一处理。

## 模型看到的新工具

```json
[
  {
    "name": "background_run",
    "description": "Run command in background thread. Returns task_id immediately.",
    "input_schema": {
      "type": "object",
      "properties": { "command": { "type": "string" } },
      "required": ["command"]
    }
  },
  {
    "name": "check_background",
    "description": "Check background task status. Omit task_id to list all.",
    "input_schema": {
      "type": "object",
      "properties": { "task_id": { "type": "string" } }
    }
  }
]
```

两者分工清楚：

- `background_run(command)`：启动后台任务，立刻返回 task_id
- `check_background(task_id)`：主动查询后台任务状态；不传 task_id 则列出所有任务

## 后台结果是怎么进入上下文的？

这是这里最重要的机制。后台线程完成后，结果不是立刻送进上下文，而是缓存在通知队列里。主循环在**每轮调用模型之前**，先检查队列：

```python
def agent_loop(messages: list):
    while True:
        notifs = BG.drain_notifications()   # 取出所有待通知，清空队列
        if notifs and messages:
            notif_text = "\n".join(
                f"[bg:{n['task_id']}] {n['status']}: {n['result']}" for n in notifs
            )
            messages.append({"role": "user", "content": f"<background-results>\n{notif_text}\n</background-results>"})
            messages.append({"role": "assistant", "content": "Noted background results."})

        response = client.messages.create(
            model=MODEL, system=SYSTEM, messages=messages,
            tools=TOOLS, max_tokens=8000,
        )
        ...
```

`drain_notifications()` 会一次性取出队列里所有已完成的通知并清空队列。注入到 messages 里的是：

- 一条 `role: "user"` 的 `<background-results>` 消息
- 一条 `role: "assistant"` 的确认消息

所以后台通知是通过普通消息序列进入上下文的，不是改 system prompt。

这意味着模型有两种方式得知后台任务完成：

1. **被动方式**：下一轮前自动注入 `<background-results>`
2. **主动方式**：模型自己调用 `check_background`

## 例子：后台慢任务 + 前台立即工作

用户目标：

```text
后台运行：sleep 2; echo BG_DONE > bg_done.txt; echo background-finished
前台并行：创建 foreground.txt
等待后台结果注入后，读取 bg_done.txt 并汇总
```

这个例子最重要的是看清楚：==后台任务完成和模型得知后台任务完成，不是同一个时刻。==

### Round 1：先把慢命令扔到后台

模型响应：

```json
{
  "stop_reason": "tool_use",
  "content": [
    {
      "type": "tool_use",
      "name": "background_run",
      "id": "toolu_01BQiWjfjX4p8Bu6KkB1uhzw",
      "input": { "command": "sleep 2; echo BG_DONE > bg_done.txt; echo background-finished" }
    }
  ]
}
```

回填：

```json
{
  "role": "user",
  "content": [
    {
      "type": "tool_result",
      "tool_use_id": "toolu_01BQiWjfjX4p8Bu6KkB1uhzw",
      "content": "Background task 8b94c3e3 started: sleep 2; echo BG_DONE > bg_done.txt; echo background-finished"
    }
  ]
}
```

此时后台状态：`8b94c3e3` → running。模型只知道任务启动了，不知道最终结果。

### Round 2：主代理继续做前台工作

后台命令还在跑，但主代理不需要干等——它继续写文件：

```json
{
  "stop_reason": "tool_use",
  "content": [
    {
      "type": "tool_use",
      "name": "write_file",
      "input": { "path": "foreground.txt", "content": "foreground work done" }
    }
  ]
}
```

回填 `"Wrote 19 bytes to foreground.txt"`。

在这一轮结束附近，后台线程可能已经实际完成（`8b94c3e3` → completed），但这个信息还没有进入模型的上下文。

### Round 3 开始前：后台通知被注入

下一次模型调用之前，主循环排空通知队列，在 `messages` 末尾追加：

```json
[
  {
    "role": "user",
    "content": "<background-results>\n[bg:8b94c3e3] completed: background-finished\n</background-results>"
  },
  {
    "role": "assistant",
    "content": "Noted background results."
  }
]
```

从这一刻开始，模型才第一次"正式知道"后台任务已经完成。

### Round 3：模型基于通知继续验证

模型同时读取后台产物和前台产物：

```json
{
  "stop_reason": "tool_use",
  "content": [
    { "type": "tool_use", "name": "read_file", "input": { "path": "bg_done.txt" } },
    { "type": "tool_use", "name": "read_file", "input": { "path": "foreground.txt" } }
  ]
}
```

回填结果：

```json
{
  "role": "user",
  "content": [
    { "type": "tool_result", "content": "BG_DONE" },
    { "type": "tool_result", "content": "foreground work done" }
  ]
}
```

### Round 4：模型结束主任务

拿到前台和后台两侧的验证结果后，模型返回 `end_turn`，循环结束。

### 回顾：时间线

```
Round 1:  background_run 启动后台线程 → 立刻返回 task_id
Round 2:  主代理做前台工作（后台线程此时在跑）
          ── 后台线程完成，结果进入 notification_queue ──
Round 3:  主循环注入 <background-results> → 模型验证后台产物
Round 4:  end_turn
```

## 机制边界

这里展示了"非阻塞执行 + 延迟通知注入"的核心机制，但它仍然是一个教学版实现：

- 后台线程是 `daemon=True`——主进程退出时后台线程直接终止
- 任务状态和通知队列都在进程内存里——不能跨进程恢复，通知没来得及注入就退出时会丢失

所以这里的重点是"运行时行为"，不是"持久任务基础设施"。

## 全局关系

```
模型调用 background_run(command)
    → BackgroundManager 启动后台线程
        → 立刻返回 started + task_id
            → 主代理继续前台工作

后台线程完成
    → 更新 tasks
    → 把结果放进 notification_queue

下一轮模型调用前
    → 主循环 drain_notifications()
        → 把 <background-results> 注入 messages
            → 模型基于后台完成通知继续决策
```

这里真正新增的，不只是两个工具，而是一个新的==时间结构==：

- 工具调用发生在现在
- 后台完成发生在稍后
- 结果注入发生在下一轮调用前

## 小结

> [!abstract] 核心要点
> 这里新增了三件关键事情：
>
> - **`background_run`**：让长耗时命令可以脱离主循环等待
> - **`BackgroundManager` + notification queue**：在后台保存状态并缓存完成通知
> - **下一轮前注入 `<background-results>`**：把后台完成结果延迟送回主上下文
>
> 核心转变：前面的工具调用都是同步的——调用、等待、拿结果。这里引入了==异步时间线：调用和结果分离，中间主代理可以做别的事==。
