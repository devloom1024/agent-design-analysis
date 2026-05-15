# CC GUI — 消息格式

## 概述

CC GUI 的消息格式非常原始，采用**字符串 type + 字符串 content**的简单回调模式。没有统一的消息类型枚举，归一化责任交由上层处理。

## 核心结构

### SDKResult (Java)

```java
public class SDKResult {
    public boolean success;          // 操作是否成功
    public String error;             // 错误信息
    public int messageCount;         // 消息数量
    public List<Object> messages;    // 消息列表
    public String rawOutput;         // 原始输出
    public String finalResult;       // 最终结果文本
}
```

### MessageCallback 接口

```java
public interface MessageCallback {
    void onMessage(String type, String content);  // type: "content"/"content_delta"/"message_start"/"message_end"
    void onError(String error);
    void onComplete(SDKResult result);
}
```

## 消息类型

通过 `MessageCallback.onMessage(type, content)` 的字符串 type 区分：

| type 值 | 含义 |
|---------|------|
| `"content"` | 完整文本内容块 |
| `"content_delta"` | 流式增量文本 |
| `"message_start"` | 消息开始 |
| `"message_end"` | 消息结束 |

## 流式处理

- Daemon 模式：通过 NDJSON (`{"id":"1","line":"[CONTENT_DELTA]..."}`) 流式传输
- Per-Process 模式：stdout 行读取 → 回调 onMessage

## 特点

- 最简洁的消息模型，类型用字符串区分，无编译时类型检查
- 不包含 role、sessionId、tool_call 等高级字段
- 归一化工作交由下游（Claude Code UI 等）完成
