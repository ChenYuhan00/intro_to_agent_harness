---
title: "Agent Loop"
tags:
  - agent-loop
  - harness
  - tool-use
date: 2026-03-24
---

# Agent Loop

> [!question] 核心问题
> LLM 只能生成文本，怎么让它操作真实世界？

普通的 LLM 调用是一问一答：你发一段文字，模型回一段文字。但如果你想让模型帮你执行 shell 命令、读写文件、操作数据库——它做不到，因为它只能输出文本。

解决办法是加一个循环：模型说"我想执行 `pwd`"，你的代码替它执行，把结果喂回去，模型再决定下一步。这个循环就是 ==agent loop==。

## 最小 Agent Loop

整个循环只有四步：

```
1. 把上下文发给模型
2. 模型回复——检查它是想调用工具，还是直接给出最终答案
3. 如果要调用工具：本地执行，把结果塞回上下文
4. 回到第 1 步
```

写成代码：

```python
messages = [{"role": "user", "content": query}]

while True:
    response = client.messages.create(
        model=MODEL, system=SYSTEM, messages=messages,
        tools=TOOLS, max_tokens=8000,
    )
    messages.append({"role": "assistant", "content": response.content})

    if response.stop_reason != "tool_use":   # 模型不再需要工具，结束
        break

    results = []
    for block in response.content:
        if block.type == "tool_use":
            output = run_bash(block.input["command"])
            results.append({"type": "tool_result", "tool_use_id": block.id,
                            "content": output})
    messages.append({"role": "user", "content": results})
```

这里有两个关键词需要先认识：

> [!info] 关键概念
> - **`tool_use`**：模型的一种回复方式。它不直接给你文本答案，而是说"我要调用某个工具"。
> - **`stop_reason`**：模型回复里的一个字段，告诉你它为什么停下来。如果是 `"tool_use"`，说明它在等你执行工具；如果是 `"end_turn"`，说明它已经给出最终答案。

## 完整代码

理解了循环的骨架，现在看这份最小实现。代码分三部分：告诉模型它是谁、定义它能用什么工具、工具的执行逻辑。

### System Prompt 与工具定义

```python
SYSTEM = f"You are a coding agent at {os.getcwd()}. Use bash to solve tasks. Act, don't explain."

TOOLS = [{
    "name": "bash",
    "description": "Run a shell command.",
    "input_schema": {
        "type": "object",
        "properties": {"command": {"type": "string"}},
        "required": ["command"],
    },
}]
```

`SYSTEM` 是系统提示词，告诉模型它的角色。`TOOLS` 是工具声明，告诉模型它可以调用一个叫 `bash` 的工具，参数是一个 `command` 字符串。

### 工具执行：`run_bash`

模型只能"请求"执行命令，真正执行的是你的代码：

```python
def run_bash(command: str) -> str:
    dangerous = ["rm -rf /", "sudo", "shutdown", "reboot", "> /dev/"]
    if any(d in command for d in dangerous):
        return "Error: Dangerous command blocked"
    try:
        r = subprocess.run(command, shell=True, cwd=os.getcwd(),
                           capture_output=True, text=True, timeout=120)
        out = (r.stdout + r.stderr).strip()
        return out[:50000] if out else "(no output)"
    except subprocess.TimeoutExpired:
        return "Error: Timeout (120s)"
```

三件事：拦截危险命令、在当前目录执行、返回输出（限制长度）。

## 例子 A：单工具调用

用理论不如跑一遍。用户问：

```text
What is the current working directory?
```

### Round 1：发送请求

这是第一轮发给模型的完整请求。注意结构——`system`、`tools`、`messages` 三部分组成：

```json
{
  "model": "$MODEL_ID",
  "max_tokens": 8000,
  "system": "You are a coding agent at /Users/chenyuhan/My/learn-claude-code. Use bash to solve tasks. Act, don't explain.",
  "tools": [
    {
      "name": "bash",
      "description": "Run a shell command.",
      "input_schema": {
        "type": "object",
        "properties": { "command": { "type": "string" } },
        "required": ["command"]
      }
    }
  ],
  "messages": [
    {
      "role": "user",
      "content": "What is the current working directory?"
    }
  ]
}
```

其中 `system` 和 `tools` 每一轮都不变，变的只有 `messages`——每一轮往里追加新的对话记录。

### Round 1：模型响应

```json
{
  "role": "assistant",
  "stop_reason": "tool_use",
  "content": [
    {
      "type": "tool_use",
      "id": "toolu_015W1D4Tqne52hVNVzktcPcX",
      "name": "bash",
      "input": { "command": "pwd" }
    }
  ]
}
```

模型没有直接回答，而是返回了一个 `tool_use` 块，说"我要执行 `pwd`"。`stop_reason` 是 `"tool_use"`，意味着循环不能结束——你需要执行这个命令，把结果告诉它。

### Round 1：本地执行与回填

本地执行 `pwd`，得到 `/Users/chenyuhan/My/learn-claude-code`。

把结果包装成 `tool_result`，塞进 `messages`：

```json
{
  "role": "user",
  "content": [
    {
      "type": "tool_result",
      "tool_use_id": "toolu_015W1D4Tqne52hVNVzktcPcX",
      "content": "/Users/chenyuhan/My/learn-claude-code"
    }
  ]
}
```

注意 `tool_use_id` 必须和上面 `tool_use` 的 `id` 匹配，这样模型才知道这是哪次调用的结果。

### Round 2：再次发送请求

第二轮请求的 `system` 和 `tools` 不变，`messages` 追加了上一轮的 assistant 回复和 tool_result：

```json
{
  "system": "...",
  "tools": ["..."],
  "messages": [
    { "role": "user", "content": "What is the current working directory?" },
    {
      "role": "assistant",
      "content": [
        { "type": "tool_use", "id": "toolu_015W1D4Tqne52hVNVzktcPcX",
          "name": "bash", "input": { "command": "pwd" } }
      ]
    },
    {
      "role": "user",
      "content": [
        { "type": "tool_result", "tool_use_id": "toolu_015W1D4Tqne52hVNVzktcPcX",
          "content": "/Users/chenyuhan/My/learn-claude-code" }
      ]
    }
  ]
}
```

### Round 2：模型响应——循环结束

```json
{
  "role": "assistant",
  "stop_reason": "end_turn",
  "content": [
    {
      "type": "text",
      "text": "The current working directory is `/Users/chenyuhan/My/learn-claude-code`."
    }
  ]
}
```

这次 `stop_reason` 是 `"end_turn"`——模型认为任务完成，返回了文本答案。循环结束。

## 例子 B：一轮多个工具调用

模型可以在一次响应中请求多次工具调用。用户问：

```text
Create a directory called demo_multi_trace and write two text files in it, then list the directory.
```

### Round 1：模型响应

```json
{
  "role": "assistant",
  "stop_reason": "tool_use",
  "content": [
    {
      "type": "tool_use",
      "id": "toolu_011DxqSiAg2tiUNdhLcQaskt",
      "name": "bash",
      "input": { "command": "mkdir demo_multi_trace" }
    },
    {
      "type": "tool_use",
      "id": "toolu_01HnoBmFZJHNiNuCz5yAfbNk",
      "name": "bash",
      "input": { "command": "echo \"This is the first text file\" > demo_multi_trace/file1.txt" }
    },
    {
      "type": "tool_use",
      "id": "toolu_018dsc6aTRuJ1QbtxX8XaEoA",
      "name": "bash",
      "input": { "command": "echo \"This is the second text file\" > demo_multi_trace/file2.txt" }
    },
    {
      "type": "tool_use",
      "id": "toolu_01FNPsSmaEFMEE46wJTXJ7nm",
      "name": "bash",
      "input": { "command": "ls -la demo_multi_trace" }
    }
  ]
}
```

一次返回 4 个 `tool_use` 块。你的代码需要逐个执行，然后逐个回填结果。

### Round 1：回填结果

```json
{
  "role": "user",
  "content": [
    { "type": "tool_result", "tool_use_id": "toolu_011DxqSiAg2tiUNdhLcQaskt", "content": "(no output)" },
    { "type": "tool_result", "tool_use_id": "toolu_01HnoBmFZJHNiNuCz5yAfbNk", "content": "(no output)" },
    { "type": "tool_result", "tool_use_id": "toolu_018dsc6aTRuJ1QbtxX8XaEoA", "content": "(no output)" },
    { "type": "tool_result", "tool_use_id": "toolu_01FNPsSmaEFMEE46wJTXJ7nm", "content": "total 16\n...\nfile1.txt\nfile2.txt" }
  ]
}
```

### Round 2：循环结束

```json
{
  "role": "assistant",
  "stop_reason": "end_turn",
  "content": [
    {
      "type": "text",
      "text": "Done! I've created the directory `demo_multi_trace` with two text files inside it."
    }
  ]
}
```

所有 `tool_result` 都已回填，模型确认任务完成，`stop_reason` 为 `"end_turn"`，循环结束。

## 小结

> [!abstract] 核心要点
> 1. Agent 的核心是一个循环：==发请求 → 执行工具 → 回填结果 → 再发请求==。
> 2. 每一轮请求的 `system` 和 `tools` 不变，`messages` 不断增长——这就是模型的"记忆"。
> 3. `tool_use` 是模型发出的动作指令，`tool_result` 是你的代码回填的执行结果。
> 4. `stop_reason` 决定循环何时结束。
> 5. 这个最小闭环构成后续所有章节的基础。
