# JetBrains CC GUI — Stream Protocol 与 Message 入库设计

调研对象：`codes/jetbrains-cc-gui`

## 核心定位

JetBrains CC GUI 是 JetBrains 插件侧的 Claude Code 图形界面。它的协议设计偏轻量：Java callback 接收 SDK/daemon 输出，再交由上层 UI 处理。

## Stream Protocol

核心回调接口类似：

```java
void onMessage(String type, String content);
void onError(String error);
void onComplete(SDKResult result);
```

常见 `type`：

- `message_start`
- `content`
- `content_delta`
- `message_end`

Daemon 模式下，插件读取 NDJSON 行，例如：

```json
{"id":"1","line":"[CONTENT_DELTA]..."}
```

Per-process 模式则直接读取 stdout 行，并触发 callback。

项目中还有 `ReplayDeduplicator`，用于去重已经通过 full-message sync 出现过的 streaming delta，避免 UI 重复展示。

## Message 入库保存协议

插件侧的消息结果是轻量 `SDKResult`：

- `success`
- `error`
- `messageCount`
- `messages`
- `rawOutput`
- `finalResult`

它没有复杂数据库 schema，也不把 `content_delta` 逐条入库。真正的历史保存更多依赖 Claude Code 原生 session/transcript 或上层 UI 自己的存储。

## 设计评价

CC GUI 的协议是桥接层协议：低成本、低类型约束、容易接入，但也要求下游负责归一化和持久化。适合作为 IDE 插件 adapter，不适合作为大型 Agent 产品的核心 message schema。
