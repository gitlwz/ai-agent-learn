---
title: "大模型流式输出：为什么常用 SSE 而不是 WebSocket"
date: 2026-07-13
tags: [AI Agent, SSE, WebSocket, Streaming, 工程实践, 面试]
---

# 大模型流式输出：为什么常用 SSE 而不是 WebSocket

> 一句话导读：大模型流式输出本质是服务器向浏览器单向推 token，SSE 刚好贴合这个场景；WebSocket 更适合双向、高频、长连接协作。

做 AI 应用时，最常见的用户体验就是“打字机效果”：

```text
模型不是等整段回答生成完再返回，
而是一边生成，一边把 token 推给前端。
```

面试里经常会被问：

```text
这个流式输出怎么实现？
为什么很多大模型接口用 SSE，而不是 WebSocket？
```

这个问题表面是在问前端通信方案，本质是在考你有没有理解大模型生成的通信模式。

---

## 一、先抓住根本区别：单向还是双向

SSE 和 WebSocket 最大的区别不是“哪个更高级”，而是方向。

```text
SSE：Server-Sent Events
服务器 -> 客户端
单向推送

WebSocket：
客户端 <-> 服务器
双向通信
```

可以这样记：

```text
SSE 像收音机：
服务器一直播，客户端一直听。

WebSocket 像对讲机：
客户端能说，服务器也能说，双方都能主动发消息。
```

大模型流式输出的核心动作是：

```text
服务器把模型生成的 token 一个个推给浏览器。
```

这天然是单向的。

用户输入 prompt 之后，服务端开始生成，前端只需要不断接收：

```text
用户发起请求
  -> 服务端调用模型
  -> 模型持续吐 token
  -> 服务端持续推给浏览器
  -> 浏览器逐步渲染
```

在这个过程中，客户端通常不需要在同一条连接里高频往回发消息。

所以 SSE 很贴合。

---

## 二、大模型流式输出为什么适合 SSE

### 1. 通信方向刚好匹配

大模型生成是典型的服务端单向推送：

```text
server -> client
```

前端不是和模型实时对话式互相喊话，而是在等待服务器不断推送生成片段。

如果只是为了接收 token，用 WebSocket 就有点重。

不是不能用，而是没有必要。

---

### 2. SSE 基于普通 HTTP

SSE 本质上还是 HTTP。

这意味着它天然兼容很多已有基础设施：

```text
HTTP 网关
反向代理
鉴权中间件
日志系统
负载均衡
浏览器 EventSource API
```

服务端只要返回：

```http
Content-Type: text/event-stream
Cache-Control: no-cache
Connection: keep-alive
```

然后持续写入：

```text
data: 你好

data: ，我是

data: Claude

data: [DONE]
```

前端就可以一边收到一边渲染。

---

### 3. 前端实现简单

浏览器原生支持 `EventSource`。

典型代码是：

```js
const source = new EventSource("/api/chat/stream");

source.onmessage = (event) => {
  if (event.data === "[DONE]") {
    source.close();
    return;
  }

  appendToken(event.data);
};

source.onerror = () => {
  source.close();
};
```

如果用 `fetch` 读取 stream，也可以实现类似效果：

```js
const response = await fetch("/api/chat", {
  method: "POST",
  body: JSON.stringify({ prompt }),
});

const reader = response.body.getReader();
const decoder = new TextDecoder();

while (true) {
  const { value, done } = await reader.read();
  if (done) break;

  const chunk = decoder.decode(value, { stream: true });
  appendToken(chunk);
}
```

真实项目里，SSE 的实现和调试成本通常比 WebSocket 低。

---

### 4. SSE 自带断线重连语义

`EventSource` 有自动重连机制。

服务端还可以通过 `id` 字段配合 `Last-Event-ID` 做续传：

```text
id: 101
data: token chunk

id: 102
data: token chunk
```

客户端断线后重连，可以带上最后收到的事件 ID。

不过要注意：大模型 token 流的“断线续传”不一定总能完美恢复，因为模型生成状态可能在服务端、模型供应商或中间层里。SSE 提供的是协议层重连能力，业务上是否能续上，还要看服务端有没有保存上下文和已发送片段。

---

## 三、那 WebSocket 什么时候更合适

WebSocket 的优势在于双向、低延迟、长连接。

它适合这些场景：

```text
聊天室
在线游戏
协同编辑
实时白板
多人光标
语音/视频信令
需要客户端持续发送控制消息的 Agent 界面
```

比如一个复杂 Agent 工作台，用户可能在 Agent 执行过程中不断干预：

```text
暂停
恢复
取消
修改目标
批准某个工具调用
发送新输入
切换分支
多人协作评论
```

如果这些控制消息需要和服务端保持高频双向通信，WebSocket 会更自然。

所以判断标准不是：

```text
SSE 一定比 WebSocket 好。
```

而是：

```text
单向推流优先 SSE。
双向高频交互优先 WebSocket。
```

---

## 四、SSE 的限制也要知道

SSE 很适合 LLM streaming，但不是没有坑。

### 1. 浏览器连接数限制

在 HTTP/1.1 下，浏览器对同一域名的并发连接数有限制，常见是 6 个左右。

如果一个页面同时打开很多 SSE 连接，可能互相阻塞。

HTTP/2 会缓解这个问题，因为它支持多路复用。

---

### 2. 原生 EventSource 主要是 GET

浏览器原生 `EventSource` 默认发 GET 请求，不方便直接带复杂 JSON body。

实际项目里常见解法有几种：

```text
1. 先 POST 创建任务，返回 streamId，再用 EventSource GET /stream/:id。
2. 使用 fetch + ReadableStream 自己读流。
3. 使用支持 POST SSE 的封装库。
```

很多大模型 API 的流式接口在 SDK 里并不直接暴露为浏览器原生 `EventSource`，而是用 `fetch` stream 或 Node stream 包了一层。

---

### 3. 代理和缓冲要处理

一些网关、Nginx、Serverless 平台可能会缓冲响应，导致 token 不是一段段到达，而是攒一批再吐出来。

这会破坏打字机效果。

工程上要关注：

```text
关闭代理缓冲
及时 flush
设置正确 Content-Type
避免中间层压缩/缓存影响流式输出
```

Nginx 场景下常见配置包括：

```nginx
proxy_buffering off;
proxy_cache off;
```

具体还要看部署平台。

---

### 4. SSE 不是二进制通道

SSE 传的是文本事件。

如果你的场景需要大量二进制数据、复杂双向帧、强实时交互，WebSocket 更合适。

但 LLM token 本来就是文本，所以这个限制通常不是问题。

---

## 五、一个典型 LLM SSE 链路

以聊天接口为例，链路通常是：

```text
浏览器
  -> POST /api/chat
  -> 服务端收到 prompt
  -> 调模型流式接口
  -> 模型返回 delta
  -> 服务端转成 SSE chunk
  -> 浏览器逐步渲染
```

服务端伪代码：

```ts
app.post("/api/chat", async (req, res) => {
  res.setHeader("Content-Type", "text/event-stream");
  res.setHeader("Cache-Control", "no-cache");
  res.setHeader("Connection", "keep-alive");

  const stream = await model.stream({
    messages: req.body.messages,
  });

  for await (const chunk of stream) {
    const text = chunk.delta ?? "";
    res.write(`data: ${JSON.stringify({ text })}\n\n`);
  }

  res.write("data: [DONE]\n\n");
  res.end();
});
```

前端伪代码：

```js
const response = await fetch("/api/chat", {
  method: "POST",
  headers: {
    "Content-Type": "application/json",
  },
  body: JSON.stringify({ messages }),
});

const reader = response.body.getReader();
const decoder = new TextDecoder();

let buffer = "";

while (true) {
  const { value, done } = await reader.read();
  if (done) break;

  buffer += decoder.decode(value, { stream: true });

  const events = buffer.split("\n\n");
  buffer = events.pop() ?? "";

  for (const event of events) {
    if (!event.startsWith("data: ")) continue;

    const data = event.slice("data: ".length);
    if (data === "[DONE]") return;

    const payload = JSON.parse(data);
    appendToken(payload.text);
  }
}
```

这就是 AI 产品里常见的流式渲染骨架。

---

## 六、和 Agent Harness 的关系

在 Agent 系统里，流式输出不只包括“模型 token”。

还可能包括：

```text
assistant 文本增量
tool_use 事件
tool_result 事件
权限请求
任务状态
子 Agent 完成通知
错误事件
最终 done 事件
```

所以一个成熟的 Agent Harness 往往不会只推纯文本，而会推结构化事件：

```text
event: message_delta
data: {"text":"正在分析"}

event: tool_use
data: {"name":"Read","input":{"file_path":"src/index.ts"}}

event: tool_result
data: {"tool_use_id":"xxx","status":"success"}

event: done
data: {}
```

这也是 SSE 很适合 Agent 的原因：它本来就支持事件类型。

前端可以根据不同事件更新不同 UI：

```text
message_delta -> 追加文本
tool_use -> 展示工具卡片
tool_result -> 更新工具状态
permission_request -> 弹审批按钮
done -> 收尾
```

如果是更复杂的 Agent IDE，比如需要用户在执行中频繁插话、多人协作、实时编辑上下文，那就可能混合使用：

```text
SSE：负责服务端向前端推执行事件。
HTTP：负责用户提交消息、批准权限、取消任务。
WebSocket：负责更复杂的实时双向协作。
```

不要把协议选择变成信仰。它只是架构取舍。

---

## 七、面试怎么答

如果面试官问：

```text
大模型流式输出为什么用 SSE，不用 WebSocket？
```

可以这样答：

```text
首先看通信方向。大模型生成时，主要是服务端把 token 单向推给客户端，客户端不需要在同一条连接里高频回传消息，所以 SSE 的单向推送模型刚好匹配。

其次 SSE 基于普通 HTTP，能复用现有网关、鉴权、代理和负载均衡，实现也简单，浏览器有 EventSource，服务端只要 text/event-stream 持续写 chunk。

WebSocket 更适合双向、高频、低延迟场景，比如聊天室、协同编辑、在线游戏。如果 Agent 产品需要执行中频繁控制、多人实时协作，也可以用 WebSocket。

所以不是 WebSocket 不行，而是 LLM streaming 这个场景下 SSE 够用、更轻、更贴合。
```

再补两个加分点：

```text
1. SSE 在 HTTP/1.1 下有同域连接数限制，HTTP/2 能缓解。
2. EventSource 原生偏 GET，如果要 POST body，可以用 fetch stream，或者先 POST 创建任务再 GET 订阅流。
```

这样回答基本就能说明你不是只会背概念，而是真的理解工程场景。

---

## 八、总结

大模型流式输出选择 SSE，核心不是因为它“更高级”，而是因为它刚好匹配：

```text
大模型生成：服务端单向推 token。
SSE：服务端单向推事件。
```

判断协议时抓住这一句：

```text
单向推流用 SSE。
双向高频用 WebSocket。
```

在 AI Agent 产品里更完整的判断是：

```text
普通聊天流式输出：SSE 足够。
结构化 Agent 事件流：SSE 很适合。
用户控制动作：HTTP 请求即可。
复杂实时协作：考虑 WebSocket。
```

技术选型不是选“最强”的协议，而是选“刚好贴合场景”的协议。
