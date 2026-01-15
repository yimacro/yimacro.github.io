---
layout: post

title: ClaudeCode原理分析

categories: [AI]

---

claude-code是一个终端编程的工具，非常强大能独立的完成独立软件的开发，最近提出的skill的概念也很强，让模型熟练掌握一项技能之后再去做事，能打到事半功倍的效果

learn-claude-code是大佬通过反编译claude-code的源码，对claude-code的整体设计有一定见解后写的一个能让大家理解claude-code原理的项目。项目从一个循环调用工具的模型的观点出发，循序渐进，一个版本引入一个claude-code的核心设计，让你一步步深入，了解claude-code的全貌。

## 复杂能力能从简单规则中涌现

v0的版本，作者通过bash工具和模型的结合达到能分析整个项目的需求，核心代码如下：

```python
while True:
    response = model(messages, tools)
    if response.stop_reason != "tool_use":
        return response.text
    results = execute(response.tool_calls)
    messages.append(results)
```

通过不断调用工具来响应模型的分析结果，直到模型得到全部的信息来回答用户的问题。真正调用工具的不是模型是模型输出对应的工具需要的参数，如需要使用工具就在参数中填加“tool_use”，然后在输出命令如“find -name *.py”,最后的执行是终端。终端检测到tool_use的参数后，终端来调用独立的bash来执行这个指令。这个是整个claude-code的主逻辑，后续的版本讲围绕该代码进行优化。

## 给模型工具，让它工作

v1版本新增了一些工具扩展了终端的能力，首先需要在prompt描述你的能力，如下这里新增了读取文件的能力：

```

TOOLS = [
    # Tool 1: Bash - The gateway to everything
    # Can run any command: git, npm, python, curl, etc.
    {
        "name": "bash",
        "description": "Run a shell command. Use for: ls, find, grep, git, npm, python, etc.",
        "input_schema": {
            "type": "object",
            "properties": {
                "command": {
                    "type": "string",
                    "description": "The shell command to execute"
                }
            },
            "required": ["command"],
        },
    },

    # Tool 2: Read File - For understanding existing code
    # Returns file content with optional line limit for large files
    {
        "name": "read_file",
        "description": "Read file contents. Returns UTF-8 text.",
        "input_schema": {
            "type": "object",
            "properties": {
                "path": {
                    "type": "string",
                    "description": "Relative path to the file"
                },
                "limit": {
                    "type": "integer",
                    "description": "Max lines to read (default: all)"
                },
            },
            "required": ["path"],
        },
    },
]

```

新增工具后需要实现该工具的代码，如下：

```
def run_read(path: str, limit: int = None) -> str:
    """
    Read file contents with optional line limit.

    For large files, use limit to read just the first N lines.
    Output truncated to 50KB to prevent context overflow.
    """
    try:
        text = safe_path(path).read_text()
        lines = text.splitlines()

        if limit and limit < len(lines):
            lines = lines[:limit]
            lines.append(f"... ({len(text.splitlines()) - limit} more lines)")

        return "\n".join(lines)[:50000]

    except Exception as e:
        return f"Error: {e}"
```
在这个版本中，由于模型来控制整个循环，当没有工具调用后结束整个流程，工具的使用记录和模型的回答都一次增加到history后面，工具使用的结果给模型反馈推动模型持续处理直到解决问题。

## 适当的约束是赋能

上面的流程可以工作，但是对于复杂的任务，模型会失去方向，如让它“总结代码，重构代码，补充测试”等，模型没有显式的规划，它会忘记步骤失去焦点。

所以在v2版本中，新增了“TODO工具”，来改变Agent的工作方式，以前要做什么在模型的脑子中，现在让它展示出来。
```
输入：
  [x] 重构认证（已完成）
  [>] 添加测试（进行中）
  [ ] 更新文档（待办）

返回：
  "[x] 重构认证
   [>] 添加测试
   [ ] 更新文档
   (1/3 已完成)"
```

模型看到自己的计划，更新它，带着上下文继续。对于列出步骤的约束也很重要，它就像工作列表的护栏，防止异常和崩溃

| 规则 | 原因 |
|------|------|
| 最多 20 条 | 防止无限列表 |
| 只能一个进行中 | 强制聚焦 |
| 必填字段 | 结构化输出 |

好的约束不是限制，而是脚手架。Todo 的约束（最大条目、只能一个进行中）赋能了（可见计划、追踪进度）。

Agent 设计中的模式：
- `max_tokens` 约束 → 赋能可管理的响应
- 工具 Schema 约束 → 赋能结构化调用
- Todo 约束 → 赋能复杂任务完成


## 分而治之，保持聚焦

v3 引入的子代理（subagent / Task）机制，是对 v0–v2 线性历史增长问题的直接回应：当探索、规划和实现需要各自独立的上下文时，单一 Agent 的连续对话会把无关内容堆积进历史，导致忘记、混淆或失焦。

把每个子任务交给一个“干净”的短期子代理处理，子代理有自己的系统提示、受限工具集合与独立的会话历史，完成后只把最终摘要或结果返回给主代理。代码如下

```python
def run_task(description, prompt, agent_type):
    config = AGENT_TYPES[agent_type]
    sub_system = f"You are a {agent_type} subagent.\n{config['prompt']}"
    sub_tools = get_tools_for_agent(agent_type)
    sub_messages = [{"role": "user", "content": prompt}]

    while True:
        response = client.messages.create(model=MODEL, system=sub_system, messages=sub_messages, tools=sub_tools)
        if response.stop_reason != "tool_use":
            break
        # 执行工具调用并把结果追加到 sub_messages

    return extract_final_text(response)
```

在进行任务处理时，采用分而治之的策略，将探索、规划和实现等过程在子代理中独立进行，以避免污染主代理的历史记录，使主代理能够始终保持关注核心逻辑和目标；同时，子代理设计遵循最小权限原则以保障安全与审计的可控性，并通过输出抽象化的方式，仅返回关键结论或补丁摘要，而不是大量原始数据或文件内容，从而提升结果清晰性减少成本。

需要注意的是对子代理应设定运行预算，包括工具调用次数、执行时间和步骤上限，以防止资源过度消耗或无限循环；同时，禁止子代理创建新任务，避免引发任务递归膨胀；此外，子代理输出的结果需具备可验证性，即包含可复现的命令或文件路径，便于主代理或人类进行审查与确认。

## 把知识从参数搬到文件

v3 解决了「如何分工」，而 v4 关注模型「如何知道」：把专业知识从模型参数外化到文件系统是 Skills 机制的核心。将知识以可编辑、可版本化的 `SKILL.md` 存储，按需注入对话历史，既可维护也利于审计。

要点概览：

- 知识外化：把专门化知识从模型参数转移到文件系统，任何人通过编辑 `SKILL.md` 即可改变 Agent 行为。
- 按需加载：Skill 分层（元数据、主体、资源），元数据可常驻，主体仅在触发时注入，资源按需读取，节省上下文预算。
- 注入方式：Skill 内容作为工具返回（tool_result）追加到消息末尾，而非修改 system prompt，从而保持 KV cache 的命中率。

Skill 的结构与使用：

- 目录结构：每个 skill 目录包含 `SKILL.md`，可选 `references/`、`scripts/` 等辅助资源。
- 格式：YAML 前置元数据（name、description） + Markdown 正文（指南、命令示例、检查清单）。
- Skill 工具：提供 `Skill` 工具用于加载指定 skill 的完整内容，返回结果追加到对话历史供模型调用。

实现要点（精要）：

- `SkillLoader`：扫描技能目录、解析 `SKILL.md`、提供简要与完整内容的加载接口。
- 注入策略：将 skill 内容包装为 `<skill-loaded name="...">...</skill-loaded>` 的 `tool_result`，以 user message 形式追加历史，避免修改 system 前缀。
- 缓存与成本：因只追加消息，前缀不变，可命中 KV cache，节省推理成本。

实践建议：

- 将可运行示例与复现命令放入 Skill（方便验证）。
- 限制注入体量：仅在触发点注入必要段落，复杂内容按需分页加载。
- 版本控制：把 Skill 放在 Git 下，通过 PR 审查变更，确保知识来源可信与可追溯。




