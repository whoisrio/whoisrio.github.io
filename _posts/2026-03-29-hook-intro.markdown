---
layout: post
title:  "什么是hook"
date:   2026-03-29 21:52:31 +0800
categories: harness
---

# 什么是Hook

**什么是 Hooks？** 码农们一定很熟悉
简单的说：**Hooks 是一种在应用中“插个队”的技术。**
Hook是Harness Engineering的关键控制手段

以 Claude Code 这类 Agent 为例，一个典型任务的工作流如下：
① 用户启动会话（Session）  
② 用户提交 Prompt  
③ Agent 调用工具（Tools）  
④ Agent 委托子 Agent 执行任务  
⑤ Agent 返回结果给用户

在上述**每一个关键动作的前后**，Claude 都允许开发者植入自定义逻辑。这个“植入的逻辑”，就是 **Hook**。
通过 Hook，你可以实现对 Agent 行为的精细控制——比如在调用工具前做参数校验，在返回结果前做内容过滤，甚至在某些节点完全接管执行逻辑。
除了上述时机，Claude Code 官方还支持更多 Hook 节点，包括：
- Task 创建 / 完成
- 上下文压缩前后
- 会话结束
等等
可以说，**Hook 让开发者从“被动调用”变成了“主动干预”，在 Agent 的自动化流程中，随时拥有话语权。****

![claude_hook](https://cdn.jsdelivr.net/gh/riodong20/rio-image-bed/images/hooks-lifecycle.svg)


# 最常用的 Hook 时机：PreToolUse 与 PostToolUse
在所有 Hook 时机中，PreToolUse（调用工具前） 和 PostToolUse（调用工具后） 是目前最常用的两个。

## PreToolUse 典型应用
阻断危险操作：拦截 rm -rf / 等破坏性命令
保护敏感文件：阻止 Agent 修改 .env、credentials 等关键文件
日志审计：记录每一次工具调用的类型、参数和执行时间
替换默认工具：用自定义脚本替换默认 Bash 执行器，精简输出，减少上下文窗口占用

## PostToolUse 典型应用
输出格式化：在工具返回结果前，美化或精简输出内容
结果脱敏：检测工具返回内容中的敏感信息（如密码、Token），自动脱敏后再交给 Agent
等等

另外，在Agent 返回任务完成前 ，**STOP** 前，要求Agent执行lint验证，也是常用的Hook

Hook 如何与 Claude 交互
配置 Hook 后，Claude 会在对应时机将消息发送给 Hook 处理。例如，在调用工具前发送的消息格式如下：


```json
{
  "session_id": "abc123",
  "transcript_path": "/home/user/.claude/projects/.../transcript.jsonl",
  "cwd": "/home/user/my-project",
  "permission_mode": "default",
  "hook_event_name": "PreToolUse",
  "tool_name": "Bash",
  "tool_input": {
    "command": "npm test"
  }
}
```

Hook 执行完成后，通过退出状态码告知 Claude 如何处理：

|状态码|含义|行为|
|---|---|---|
|0|成功|Claude 继续执行后续流程|
|2|失败|Claude 阻断当前行为，不再继续|

通过这种机制，Hook 可以在关键节点上实现“允许”或“拒绝”的决策。

# Hook的配置


一个完整的 Hook 配置包含三个核心要素：

- 时机：指定在哪个事件节点触发，如 PreToolUse
- 条件（matcher）：满足什么条件时才执行，如工具名称为 Bash
- 动作：要执行的具体脚本

配置示例（claude.json 或 .claude/settings.json）：


```json
{
  "hooks": {
    "PreToolUse": [
      {
        "matcher": "Bash",
        "hooks": [
          {
            "type": "command",
            "command": "\"$CLAUDE_PROJECT_DIR\"/.claude/hooks/block-rm.sh"
          }
        ]
      }
    ]
  }
}
```

关键说明：
matcher 支持工具名称匹配（如 Bash、Write）或通配符
$CLAUDE_PROJECT_DIR 是 Claude 自动注入的环境变量，指向项目根目录
脚本需要具有可执行权限，并通过退出码告知结果
如需更复杂的条件判断（如针对特定命令 rm *），可在脚本内部自行实现。

# 总结

Hook 机制让 Claude Code 从一个“黑盒”自动化工具，变成了一个**可观测、可干预、可定制**的智能助手。
- **PreToolUse** 让你在工具执行前把关
- **PostToolUse** 让你在工具返回后优化
- 其他时机（Task、Stop、Session 等）让你在更宏观的维度掌控流程
    

通过合理配置 Hook，你可以：
- 提升安全性（阻断危险操作）
- 保障代码质量（强制 lint/测试）
- 控制成本（限制高频调用）
- 满足合规要求（日志审计、权限校验）

Hook是Harness Enginering 的关键手段，
赶紧去试试在你的项目中配置第一个 Hook，从阻断 `rm -rf` 开始，防止你被删库跑路吧


# 更多参考
[Claude Code Hook Reference](https://code.claude.com/docs/en/hooks#hooks-reference)
