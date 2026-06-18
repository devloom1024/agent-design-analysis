# AI SDK Stream Protocols 中文译文

AI SDK 的 UI 函数，例如 `useChat` 和 `useCompletion`，支持两类流协议：文本流和数据流。Stream protocol 定义的是在 HTTP 之上，后端如何把数据流式传输到前端。

这份文档说明两种协议，以及如何在前后端使用它们。你也可以用这些信息开发自定义后端或前端，例如用 Python 实现兼容 AI SDK 的 API endpoint。

## Text Stream Protocol

文本流由普通文本 chunk 组成。前端收到 chunk 后按顺序追加，最终形成完整文本回复。

文本流被 `useChat`、`useCompletion` 和 `useObject` 支持。使用 `useChat` 或 `useCompletion` 时，需要通过 `streamProtocol: 'text'` 或 `TextStreamChatTransport` 启用文本流。

后端可用 `streamText` 生成文本流；对结果对象调用 `toTextStreamResponse()` 后，会返回一个 streaming HTTP response。

> 文本流只支持基础文本数据。如果需要传输工具调用、reasoning、source、文件或自定义数据，应使用 data stream。

### Text Stream 示例

前端使用 `TextStreamChatTransport`：

```tsx
'use client';

import { useChat } from '@ai-sdk/react';
import { TextStreamChatTransport } from 'ai';
import { useState } from 'react';

export default function Chat() {
  const [input, setInput] = useState('');
  const { messages, sendMessage } = useChat({
    transport: new TextStreamChatTransport({ api: '/api/chat' }),
  });

  return (
    <form
      onSubmit={e => {
        e.preventDefault();
        sendMessage({ text: input });
        setInput('');
      }}
    >
      <input value={input} onChange={e => setInput(e.currentTarget.value)} />
    </form>
  );
}
```

后端返回文本流：

```ts
import { streamText, UIMessage, convertToModelMessages } from 'ai';

export const maxDuration = 30;

export async function POST(req: Request) {
  const { messages }: { messages: UIMessage[] } = await req.json();

  const result = streamText({
    model: __MODEL__,
    messages: await convertToModelMessages(messages),
  });

  return result.toTextStreamResponse();
}
```

## Data Stream Protocol

Data stream 使用 AI SDK 定义的专用协议，把结构化信息发送到前端。该协议采用 Server-Sent Events（SSE）格式，因此具备更标准的事件边界、keep-alive ping、重连能力和更好的缓存处理。

自定义后端返回 data stream 时，需要设置响应头：

```text
x-vercel-ai-ui-message-stream: v1
```

后端可用 `streamText` 生成结果，并调用 `toUIMessageStreamResponse()` 返回 streaming HTTP response。前端 `useChat` 和 `useCompletion` 默认使用 data stream。`useCompletion` 只支持 `text` 和 `data` stream parts。

## Data Stream Part 类型

### Message Start Part

表示新消息开始，并携带元数据。

```text
data: {"type":"start","messageId":"..."}
```

### Text Parts

文本内容使用 start/delta/end 模式。每个 text block 有唯一 ID。

```text
data: {"type":"text-start","id":"msg_..."}
data: {"type":"text-delta","id":"msg_...","delta":"Hello"}
data: {"type":"text-end","id":"msg_..."}
```

### Reasoning Parts

推理内容同样使用 start/delta/end 模式。每个 reasoning block 有唯一 ID。

```text
data: {"type":"reasoning-start","id":"reasoning_123"}
data: {"type":"reasoning-delta","id":"reasoning_123","delta":"This is some reasoning"}
data: {"type":"reasoning-end","id":"reasoning_123"}
```

### Source Parts

Source part 用于提供外部内容引用。

URL source：

```text
data: {"type":"source-url","sourceId":"https://example.com","url":"https://example.com"}
```

Document source：

```text
data: {"type":"source-document","sourceId":"https://example.com","mediaType":"file","title":"Title"}
```

### File Part

File part 保存文件 URL 和媒体类型。

```text
data: {"type":"file","url":"https://example.com/file.png","mediaType":"image/png"}
```

### Data Parts

自定义 data part 允许传输任意结构化数据。`type` 使用 `data-*` 模式，前端可按后缀做类型化处理。

```text
data: {"type":"data-weather","data":{"location":"SF","temperature":100}}
```

### Error Part

错误 part 会按收到的顺序追加到消息中。

```text
data: {"type":"error","errorText":"error message"}
```

### Tool Input Parts

工具输入也使用分阶段协议。

```text
data: {"type":"tool-input-start","toolCallId":"call_...","toolName":"getWeatherInformation"}
data: {"type":"tool-input-delta","toolCallId":"call_...","inputTextDelta":"San Francisco"}
data: {"type":"tool-input-available","toolCallId":"call_...","toolName":"getWeatherInformation","input":{"city":"San Francisco"}}
```

### Tool Output Available Part

工具执行完成后，通过 output part 传回结果。

```text
data: {"type":"tool-output-available","toolCallId":"call_...","output":{"city":"San Francisco","weather":"sunny"}}
```

### Step Parts

Step 表示一次后端 LLM API 调用。工具调用、多步推理或 stitched assistant calls 场景中，step 边界能帮助前端正确处理多段 assistant 输出。

```text
data: {"type":"start-step"}
data: {"type":"finish-step"}
```

### Finish / Abort Parts

消息完成：

```text
data: {"type":"finish"}
```

流被中止：

```text
data: {"type":"abort","reason":"user cancelled"}
```

### Stream Termination

Data stream 以特殊的 `[DONE]` 标记结束。

```text
data: [DONE]
```

## UI Message Stream 示例

前端默认使用 UI message stream：

```tsx
'use client';

import { useChat } from '@ai-sdk/react';
import { useState } from 'react';

export default function Chat() {
  const [input, setInput] = useState('');
  const { messages, sendMessage } = useChat();

  return (
    <form
      onSubmit={e => {
        e.preventDefault();
        sendMessage({ text: input });
        setInput('');
      }}
    >
      <input value={input} onChange={e => setInput(e.currentTarget.value)} />
    </form>
  );
}
```

后端返回 UI message stream：

```ts
import { streamText, UIMessage, convertToModelMessages } from 'ai';

export const maxDuration = 30;

export async function POST(req: Request) {
  const { messages }: { messages: UIMessage[] } = await req.json();

  const result = streamText({
    model: __MODEL__,
    messages: await convertToModelMessages(messages),
  });

  return result.toUIMessageStreamResponse();
}
```

## 与 Message 入库的关系

AI SDK 的 stream protocol 只定义“如何传输实时事件”，不规定应用如何入库。比较合理的入库策略是：

- 用 data stream part 更新前端临时状态。
- 文本、reasoning、tool input/output 等 block 在 `finish` 后折叠为完整 `UIMessage.parts`。
- 数据库存储完成态 `UIMessage`、会话 ID、模型、provider、usage、工具调用结果和错误信息。
- 不把每个 `text-delta` 当成独立 message 入库，除非你需要审计级 token replay。
