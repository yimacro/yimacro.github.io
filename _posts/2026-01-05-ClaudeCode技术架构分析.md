---
layout: post

title: Claude Code 技术架构全景分析报告

categories: [AI]

---

基于 claude code 源码与  cli_source.js逆向分析的联合解读。

## 1 总体架构设计

Claude Code 采用 “Host–Plugin” 双层架构，将“执行运行时”与“业务逻辑”分离：

- **Host (宿主/CLI)**：闭源的核心运行时（Node.js），负责底层资源管理、AI 通信、上下文控制和工具执行。
- **Plugins (插件层)**：开源的业务逻辑层，采用 Prompt-as-Code 和多语言 Hooks 模式，定义具体的 Agent 行为、命令和安全策略。

## 2 宿主 (Host) 核心技术解析

（基于 `cli_source.js` 的逆向分析）

宿主程序是一个复杂的事件驱动型 REPL（Read–Eval–Print Loop），核心包含若干子系统：

### 2.1 主循环（The Main Loop）

整个 CLI 的心脏，负责协调用户输入与 AI 响应的流式交互：

- **输入获取**：从终端读取用户指令。
- **消息构建**：将输入封装为 Claude API 消息格式。
- **Token 管理**：检查上下文窗口（Context Window），触发压缩策略。
- **API 调用**：发送请求并流式接收 SSE（Server-Sent Events）。
- **响应解析**：实时解析 `text_delta`（显示文本）和 `tool_use`（调用工具）。
- **状态更新**：维护会话状态、Token 计数和计费信息。

### 2.2 Token 管理与上下文压缩（Context Engine）

为在有限上下文窗口（如 200k/1m/2m）中维持长对话，Host 实现了智能压缩流水线：

**决策逻辑**：

- 限制检测：根据模型 ID 返回限制（例如 `claude-3-opus-20240229` 对应 200k）。
- 环境变量：检查 `CLAUDE_CODE_MAX_OUTPUT_TOKENS` 等配置。

**压缩策略（优先级自高到低）**：

- `Truncate Chunk`：从头部截断旧消息（FIFO）。
- `Summarize`：生成历史对话摘要，替换原始消息。
- `Image Compression`：降低图片分辨率（`maxPixels: 1568x1568`）和质量（`JPEG q=0.8`）。
- `Tool Output Cleaning`：丢弃过大的工具执行结果（非关键时）。

### 2.3 工具执行系统（Tooling System）

Host 充当工具调用的“总线”，包含：

- **验证层**：解析参数 → 权限检查 → 沙箱检查 → 超时熔断。
- **分发层**：将调用分发给具体执行器，例如：
	- `Bash`：执行 Shell 命令（受 `BASH_MAX_OUTPUT_LENGTH` 限制，默认 30000 字符）。
	- `Edit/Read/Write`：文件系统操作。
	- `MCP`：对接外部 MCP 服务器（通过 stdio/http 协议）。
	- `Glob/Grep`：内置高效文件搜索工具。

### 2.4 数据流与协议（Data Flow）

- **输入流**：异步迭代器处理标准输入，支持 4096 bytes 分块缓冲。
- **输出流**：统一 Base64 编解码，通过 `writeStdout` / `writeStderr` 输出。
- **MCP 协议**：实现完整的 MCP 客户端，支持 `call`, `resources`, `tools` 等原语。

## 3 插件（Plugin）架构解析

（基于 `plugins/` 目录源码分析）

Claude Code 的插件系统展示了 Agent-Native 应用的未来形态：逻辑由 Prompts 与 Hooks 主导，代码作为执行器。

### 3.1 Prompt-as-Code（Markdown 驱动）

业务逻辑被“软化”为 Markdown 文件，使 LLM 直接理解意图：

- `Commands`（`commands/*.md`）：Slash Command 的定义即 System Prompt。
- `Agents`（`agents/*.md`）：定义智能体的人设、工具集（Tools）和工作流。
- 示例：`code-architect` 指定了 `color: green` 和工具集如 `[Glob, Grep, Read...]`。
- `Skills`（`skills/**/SKILL.md`）：可复用的能力模块，通过 Prompt 注入给 Agent。

### 3.2 Polyglot Hooks（多语言钩子）

为弥补 Markdown 在精确控制逻辑上的不足，引入 Python/JS 脚本作为拦截器：

- **通信机制**：Host 通过 STDIN 注入 JSON 上下文（含 Transcript、ToolInput），脚本通过 STDOUT 返回控制指令。
- **控制原语**：
	- `permissionDecision: "deny"` → 拦截（例如禁止 `rm -rf`）。
	- `systemMessage: "..."` → 注入警告或提示到上下文中。
	- `hookEvent`：支持 `PreToolUse`、`PostToolUse`、`UserPromptSubmit` 等生命周期。

## 4 总结：核心技术壁垒

Claude Code 的优势主要体现在：

- **极致的 Context 工程**：Host 端的透明压缩、缓存优化（Cache Creation/Read）和 Token 精算，保证长对话稳定性。
- **Agent 编排原语**：通过 Markdown 定义 Agent，通过钩子实现硬约束，构成“软硬结合”的开发范式。
- **MCP 生态集成**：默认集成 Model Context Protocol，使其易于扩展（连接数据库、API 等）。

> 说明：本文基于逆向工程证据与源码事实还原，勾勒出 Claude Code 作为下一代 AI 编程终端的技术底座。
