---
title: "Tool Use"
tags:
  - tool-use
  - 工具分发
  - harness
date: 2026-03-24
---

# Tool Use

> [!question] 核心问题
> 只有 `bash` 为什么还不够？

在 [[learn_claude_code_sample/agent_loop|Agent Loop]] 的最小版里，模型已经可以通过 `bash` 做任何事。但"能做"和"做得好"是两回事。只靠 `bash` 有三个实际问题：

1. **安全边界难控制**：`bash` 权限太大，模型可以 `rm -rf`、可以读工作区外的文件。前面的最小实现里虽然有一个简陋的黑名单拦截，但防不住所有情况。如果把文件操作拆成专用工具，就可以在工具层统一加路径边界检查。

2. **可靠性差**：让模型拼 `sed 's/old/new/'` 来编辑文件，远不如给它一个 `edit_file(path, old_text, new_text)` 接口可靠。专用工具的参数结构化、语义明确，模型出错的概率更低。

3. **可观测性弱**：当模型调用 `bash` 时，harness 只知道"它跑了一段 shell"，不知道意图是读文件还是写文件还是别的。拆成专用工具后，每次调用的意图一目了然，方便记录和审计。

这里要做的就是：**循环不变，把工具层拆细。**

## 唯一的循环变化：按工具名分发

和最小版 agent loop 相比，循环只改了一行——从"固定调用 `run_bash`"变成"按工具名查表分发"：

```python
# 最小版：固定调用
output = run_bash(block.input["command"])

# 当前实现：按名分发
handler = TOOL_HANDLERS.get(block.name)
output = handler(**block.input) if handler else f"Unknown tool: {block.name}"
```

`TOOL_HANDLERS` 就是一张名字到函数的映射表：

```python
TOOL_HANDLERS = {
    "bash":       lambda **kw: run_bash(kw["command"]),
    "read_file":  lambda **kw: run_read(kw["path"], kw.get("limit")),
    "write_file": lambda **kw: run_write(kw["path"], kw["content"]),
    "edit_file":  lambda **kw: run_edit(kw["path"], kw["old_text"], kw["new_text"]),
}
```

这张表是这一版的核心新增。理解了它，整套机制就理解了大半。

## 全局关系

新增的零件不多，但它们的协作关系值得画一遍：

```
模型看到 TOOLS（工具声明）
    → 决定调用哪个工具，返回 tool_use（含工具名和参数）
        → 循环代码拿到 tool_use
            → 查 TOOL_HANDLERS，找到对应的本地函数
                → 执行函数（内部调 safe_path 做路径边界检查）
                    → 返回结果，包装成 tool_result，回填给模型
```

注意两侧的对称：
- **`TOOLS`**（工具声明）是给模型看的——没有它，模型不知道有哪些工具可用
- **`TOOL_HANDLERS`**（执行映射）是给本地代码用的——没有它，工具调用无法落地

两者必须同时存在，且名字必须对得上。

## 完整代码

### 工具声明：告诉模型它能用什么

```json
{
  "system": "You are a coding agent at /Users/chenyuhan/My/learn-claude-code. Use tools to solve tasks. Act, don't explain.",
  "tools": [
    {
      "name": "bash",
      "description": "Run a shell command.",
      "input_schema": {
        "type": "object",
        "properties": { "command": { "type": "string" } },
        "required": ["command"]
      }
    },
    {
      "name": "read_file",
      "description": "Read file contents.",
      "input_schema": {
        "type": "object",
        "properties": {
          "path": { "type": "string" },
          "limit": { "type": "integer" }
        },
        "required": ["path"]
      }
    },
    {
      "name": "write_file",
      "description": "Write content to file.",
      "input_schema": {
        "type": "object",
        "properties": {
          "path": { "type": "string" },
          "content": { "type": "string" }
        },
        "required": ["path", "content"]
      }
    },
    {
      "name": "edit_file",
      "description": "Replace exact text in file.",
      "input_schema": {
        "type": "object",
        "properties": {
          "path": { "type": "string" },
          "old_text": { "type": "string" },
          "new_text": { "type": "string" }
        },
        "required": ["path", "old_text", "new_text"]
      }
    }
  ]
}
```

和只有 `bash` 的最小版相比，模型不再只有一个模糊的入口，而是能看到四个语义明确的工具。

### 路径边界：`safe_path`

```python
def safe_path(p: str) -> Path:
    path = (WORKDIR / p).resolve()
    if not path.is_relative_to(WORKDIR):
        raise ValueError(f"Path escapes workspace: {p}")
    return path
```

作用很直接：把相对路径拼到工作区上，检查最终路径是否还在工作区内。

- `read_file("requirements.txt")` — 合法
- `write_file("demo/a.py", "...")` — 合法
- `read_file("../secret.txt")` — 拦截，抛异常

这就是前面说的"安全边界"：`bash` 做不到这种细粒度的路径控制，但专用工具可以。

### 工具执行：三个文件操作函数

```python
def run_read(path: str, limit: int = None) -> str:
    try:
        text = safe_path(path).read_text()
        lines = text.splitlines()
        if limit and limit < len(lines):
            lines = lines[:limit] + [f"... ({len(lines) - limit} more lines)"]
        return "\n".join(lines)[:50000]
    except Exception as e:
        return f"Error: {e}"

def run_write(path: str, content: str) -> str:
    try:
        fp = safe_path(path)
        fp.parent.mkdir(parents=True, exist_ok=True)
        fp.write_text(content)
        return f"Wrote {len(content)} bytes to {path}"
    except Exception as e:
        return f"Error: {e}"

def run_edit(path: str, old_text: str, new_text: str) -> str:
    try:
        fp = safe_path(path)
        content = fp.read_text()
        if old_text not in content:
            return f"Error: Text not found in {path}"
        fp.write_text(content.replace(old_text, new_text, 1))
        return f"Edited {path}"
    except Exception as e:
        return f"Error: {e}"
```

三者角色不同：
- `run_read()`：读取文件内容，可按行截断
- `run_write()`：创建父目录并写入新文件
- `run_edit()`：做一次精确字符串替换，而不是让模型重写整个文件

这些函数都先经过 `safe_path(path)`，再执行真正的文件操作。

## 例子：读文件、写文件、编辑文件

用户输入：

```text
Read the file requirements.txt. Then create a file called greet_trace.py with a greet(name) function. Then edit greet_trace.py to add a docstring to the function.
```

这个任务会经历 4 轮循环。前两轮展开看细节，后两轮只看关键信息。

和最小版一样，这 4 轮里固定不变的是 `system` 和 `tools`，逐轮增长的仍然主要是 `messages`。

### Round 1：模型选择 `read_file` 而非 `bash`

模型响应：

```json
{
  "stop_reason": "tool_use",
  "content": [
    {
      "type": "tool_use",
      "id": "toolu_01ATPoXdFbhV1BxKe5nMbcRQ",
      "name": "read_file",
      "input": { "path": "requirements.txt" }
    }
  ]
}
```

这正是这里想展示的效果：模型没有调用 `bash cat requirements.txt`，而是选择了专用的 `read_file`。循环代码通过 `TOOL_HANDLERS["read_file"]` 找到 `run_read`，执行后回填结果：

```json
{
  "role": "user",
  "content": [
    {
      "type": "tool_result",
      "tool_use_id": "toolu_01ATPoXdFbhV1BxKe5nMbcRQ",
      "content": "anthropic>=0.25.0\npython-dotenv>=1.0.0"
    }
  ]
}
```

### Round 2：模型用 `write_file` 创建新文件

```json
{
  "stop_reason": "tool_use",
  "content": [
    {
      "type": "tool_use",
      "id": "toolu_01YbitFPBFhCVNZH29tCRiNZ",
      "name": "write_file",
      "input": {
        "path": "greet_trace.py",
        "content": "def greet(name):\n    return f\"Hello, {name}!\"\n"
      }
    }
  ]
}
```

本地执行 `run_write`，回填 `"Wrote 46 bytes to greet_trace.py"`。

### Round 3：模型用 `edit_file` 做局部修改

这一轮最有意思——模型不是重写整个文件，而是精确指定要替换的片段：

```json
{
  "stop_reason": "tool_use",
  "content": [
    {
      "type": "tool_use",
      "id": "toolu_01CcrNqiVmRcNeZcnpJbdBBz",
      "name": "edit_file",
      "input": {
        "path": "greet_trace.py",
        "old_text": "def greet(name):\n    return f\"Hello, {name}!\"",
        "new_text": "def greet(name):\n    \"\"\"Greet a person by their name.\"\"\"\n    return f\"Hello, {name}!\""
      }
    }
  ]
}
```

本地执行 `run_edit`：找到 `old_text`，替换为 `new_text`，只改动一处。回填 `"Edited greet_trace.py"`。

### Round 4：模型结束循环

经过前三轮 `tool_result` 回填后，模型已经拿到了读文件、创建文件、编辑文件的执行结果，因此判断任务完成，返回文本答案。

此时 `stop_reason` 变为 `"end_turn"`，循环结束。

### 回顾：4 轮调用链

| Round | 工具 | 意图 | 分发到 |
|-------|------|------|--------|
| 1 | `read_file` | 读取现有文件 | `run_read` |
| 2 | `write_file` | 创建新文件 | `run_write` |
| 3 | `edit_file` | 局部编辑 | `run_edit` |
| 4 | — | 任务完成，返回文本 | — |

每一轮的循环机制和最小版完全一样，区别只在于工具名不同、分发的目标函数不同。

## 小结

> [!abstract] 核心要点
> 这一版在原有循环之上新增了三样东西：
>
> - **`TOOLS`（工具声明）**：让模型看到更多、更细的工具接口
> - **`TOOL_HANDLERS`（执行映射）**：让本地代码根据工具名找到对应函数
> - **`safe_path`（路径边界）**：让文件操作被限制在工作区内
>
> 循环本身没有变。这里的主题是：==在同一个 loop 上，如何让模型拥有更细粒度、更安全、更可靠的工具能力。==
