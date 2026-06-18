# learn-claude-code — Stream Protocol 与 Message 入库设计

调研对象：`codes/learn-claude-code`

## 核心定位

learn-claude-code 是 Claude Code 教程/学习项目，不是独立运行的聊天产品。它的价值在于按主题解释 Claude Code 的 agent loop、tool use、permission、context compact、error recovery 等机制。

## Stream Protocol

教程中强调 Claude Code 的 streaming path：

- 模型/SDK 原始事件会在运行中产生文本、工具、权限、错误等事件。
- streaming 期间部分可恢复错误会被暂存，不立即暴露给 SDK 消费者。
- 流结束后再判断是否需要恢复，例如 413、max_tokens、media error 等。

因此它更像“协议行为说明”，不是项目自有的 stream transport。

## Message 入库保存协议

learn-claude-code 本身不实现 message 入库。它解释的是 Claude Code 的 transcript/resume/compact 概念：

- 用户消息、assistant 完整消息、工具调用/结果组成 transcript。
- stream delta 服务实时显示。
- resume 使用已保存 session/transcript。
- compact 会把历史压缩成新的上下文边界。

## 设计评价

该项目适合作为 Claude Code 协议行为的阅读指南。若要落地自研产品，应参考 Claude Code / codex / acpx 这类真实实现，而不是把教程文本当成协议源代码。
