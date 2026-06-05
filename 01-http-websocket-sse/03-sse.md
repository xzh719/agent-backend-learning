# 03 — SSE (Server-Sent Events) 服务端推送

> LLM 的流式输出是最典型的 SSE 场景。OpenAI 的 `stream=True` 返回的就是 SSE 流。理解它，才能正确处理 token-by-token 的输出。

---

## 1. 概念讲解

### SSE 是什么？

SSE 是一种基于 HTTP 的 **单向推送** 技术。服务端向客户端持续发送事件，客户端通过 `EventSource` API 接收。

```
客户端: GET /stream HTTP/1.1
         Accept: text/event-stream

服务端: HTTP/1.1 200 OK
         Content-Type: text/event-stream
         Cache-Control: no-cache
         Connection: keep-alive

         data: {"token": "你好"}
         (空行)

         data: {"token": "，今天"}
         (空行)

         data: {"token": "天气不错"}
         (空行)

         data: [DONE]
         (空行)
```

### Agent 为什么用 SSE？

- **LLM 流式输出** — 用户不用等完整回复，边生成边显示
- **单工够用** — Agent 输出通常只需要 Server→Client
- **实现简单** — 就是普通 HTTP 响应，不需要协议升级
- **CDN/代理友好** — 走 HTTP 基础设施，不像 WebSocket 需要特殊配置

---

## 2. 核心原理

### 2.1 协议格式

SSE 流由连续的 **消息块** 组成，每个块有以下几个字段：

```
field: value\n

data: 这是数据的第一行\n
data: 这是数据的第二行\n
\n        ← 空行表示一条消息结束
```

关键字段：
- `data:` — 消息数据（允许多行）
- `id:` — 消息 ID（用于断线重连）
- `event:` — 事件类型（默认 `message`）
- `retry:` — 重连间隔（毫秒）

### 2.2 自动重连机制

```
客户端断连 → 等待 retry 毫秒 → 发送 Last-Event-ID 头 → 服务端从该 ID 增量推送
```

这是 SSE 最强大的特性之一，WebSocket 需要手动实现，SSE 浏览器内置支持。

### 2.3 OpenAI 的 SSE 格式

OpenAI Chat Completions API 在 `stream=True` 时返回的格式：

```
data: {"id":"chatcmpl-xxx","object":"chat.completion.chunk","choices":[{"delta":{"content":"你"},"index":0}]}

data: {"id":"chatcmpl-xxx","object":"chat.completion.chunk","choices":[{"delta":{"content":"好"},"index":0}]}

data: {"id":"chatcmpl-xxx","object":"chat.completion.chunk","choices":[{"delta":{"content":"！"},"index":0}]}

data: {"id":"chatcmpl-xxx","object":"chat.completion.chunk","choices":[{"delta":{},"finish_reason":"stop"}]}

data: [DONE]
```

**注意陷阱：**
- **不是标准 SSE**！没有 `event:` 和 `id:` 字段，只有 `data:`
- `[DONE]` 是特殊标记，不是合法 JSON
- `delta.content` 可能为 `null`（tool_calls 返回时）
- `delta.tool_calls` 的 arguments 是**增量拼接**的，不完整

---

## 3. 代码实践

### 3.1 FastAPI SSE 服务端（模拟 LLM 流式）

```python
# sse_server.py
from fastapi import FastAPI
from fastapi.responses import StreamingResponse
import asyncio
import json
import time

app = FastAPI()

async def fake_llm_stream(prompt: str):
    """模拟 LLM token-by-token 输出"""
    full_response = f"好的，我来回答关于「{prompt}」的问题。这是一个详细的回复，包含多个要点..."

    # 模拟逐字输出
    for i, char in enumerate(full_response):
        chunk = {
            "id": f"chatcmpl-{int(time.time())}",
            "object": "chat.completion.chunk",
            "created": int(time.time()),
            "model": "gpt-4o-mini",
            "choices": [{
                "index": 0,
                "delta": {"content": char},
                "finish_reason": None
            }]
        }
        yield f"data: {json.dumps(chunk, ensure_ascii=False)}\n\n"
        await asyncio.sleep(0.02)  # 模拟 ~50 tokens/s

    # 结束标志
    end_chunk = {
        "id": f"chatcmpl-{int(time.time())}",
        "object": "chat.completion.chunk",
        "choices": [{"index": 0, "delta": {}, "finish_reason": "stop"}]
    }
    yield f"data: {json.dumps(end_chunk)}\n\n"
    yield "data: [DONE]\n\n"


@app.get("/chat/stream")
async def chat_stream(prompt: str = ""):
    return StreamingResponse(
        fake_llm_stream(prompt),
        media_type="text/event-stream",
        headers={
            "Cache-Control": "no-cache",
            "Connection": "keep-alive",
            "X-Accel-Buffering": "no",  # 禁用 Nginx 缓冲
        }
    )
```

### 3.2 Python SSE 客户端（解析 OpenAI 格式）

```python
# sse_client.py
import aiohttp
import asyncio
import json

async def consume_sse(url: str, api_key: str):
    headers = {
        "Authorization": f"Bearer {api_key}",
        "Content-Type": "application/json",
    }
    payload = {
        "model": "gpt-4o-mini",
        "messages": [{"role": "user", "content": "介绍一下Python"}],
        "stream": True,
    }

    collected_content = []
    collected_tool_calls = {}

    async with aiohttp.ClientSession() as session:
        async with session.post(url, headers=headers, json=payload) as resp:
            if resp.status != 200:
                print(f"Error: {resp.status} {await resp.text()}")
                return

            # ★ 逐行读取 SSE 流
            async for line in resp.content:
                line = line.decode("utf-8").strip()

                # 跳过空行、注释行
                if not line or line.startswith(":"):
                    continue

                # 去掉 "data: " 前缀
                if line.startswith("data: "):
                    data_str = line[6:]  # len("data: ") = 6

                    # 检查 [DONE]
                    if data_str == "[DONE]":
                        print("\n=== 流结束 ===")
                        break

                    # 解析 JSON
                    try:
                        chunk = json.loads(data_str)
                    except json.JSONDecodeError:
                        print(f"Skip malformed: {data_str[:50]}")
                        continue

                    delta = chunk["choices"][0]["delta"]

                    # 处理 content
                    if "content" in delta and delta["content"]:
                        token = delta["content"]
                        collected_content.append(token)
                        print(token, end="", flush=True)

                    # 处理 tool_calls（增量拼接）
                    if "tool_calls" in delta:
                        for tc in delta["tool_calls"]:
                            idx = tc["index"]
                            if idx not in collected_tool_calls:
                                collected_tool_calls[idx] = {
                                    "id": tc.get("id", ""),
                                    "function": {"name": "", "arguments": ""}
                                }
                            if "function" in tc:
                                if "name" in tc["function"]:
                                    collected_tool_calls[idx]["function"]["name"] += tc["function"]["name"]
                                if "arguments" in tc["function"]:
                                    collected_tool_calls[idx]["function"]["arguments"] += tc["function"]["arguments"]

    print(f"\n\n完整回复: {''.join(collected_content)}")
    if collected_tool_calls:
        print(f"工具调用: {collected_tool_calls}")
```

### 3.3 带事件类型的标准 SSE 服务端

```python
async def agent_progress_stream(session_id: str):
    """推送 Agent 执行进度 —— 标准 SSE 格式"""
    events = [
        ("thinking", "正在分析用户意图..."),
        ("planning", "制定执行计划: 3个步骤"),
        ("tool_exec", json.dumps({"tool": "search", "status": "running"})),
        ("tool_result", json.dumps({"tool": "search", "results": 5})),
        ("generating", "正在生成最终回复..."),
        ("done", json.dumps({"tokens_used": 1234, "cost": 0.003})),
    ]

    for event_type, data in events:
        yield f"id: {hash(data)}\n"
        yield f"event: {event_type}\n"
        yield f"data: {data}\n\n"
        await asyncio.sleep(1)
```

前端接收：
```javascript
const es = new EventSource("/agent/progress/session-123");

es.addEventListener("thinking", (e) => {
  console.log("Agent 正在思考:", e.data);
});

es.addEventListener("tool_exec", (e) => {
  const info = JSON.parse(e.data);
  console.log(`工具 ${info.tool} 执行中...`);
});

es.addEventListener("done", (e) => {
  const info = JSON.parse(e.data);
  console.log(`完成! 用了 ${info.tokens_used} tokens`);
  es.close();
});
```

---

## 4. Agent 场景应用

### 多路 SSE 流设计

```
Client ──GET /agent/stream?session=abc──> Server
         <── event: token      data: {"content":"你"}
         <── event: token      data: {"content":"好"}
         <── event: tool_start data: {"tool":"search"}
         <── event: tool_end   data: {"tool":"search","result":"..."}
         <── event: token      data: {"content":"根据"}
         <── event: token      data: {"content":"搜索结果"}
         <── event: done       data: {"tokens":500}
```

### 中断生成

SSE 本身不是双向的，但可以结合 HTTP 来中断：
```python
# 客户端关闭 EventSource 连接即可
# es.close() → 服务端检测到连接断开 → 停止生成

@app.get("/agent/stream")
async def agent_stream(session_id: str, request: Request):
    async def generate():
        async for chunk in agent_loop(session_id):
            if await request.is_disconnected():  # ★ 检测客户端断开
                agent_stop(session_id)  # 通知 Agent 停止
                break
            yield f"data: {json.dumps(chunk)}\n\n"

    return StreamingResponse(generate(), media_type="text/event-stream")
```

---

## 5. 常见问题

| 问题 | 原因 | 解决 |
|------|------|------|
| 前端收不到第一个事件 | Nginx 缓冲了响应 | `X-Accel-Buffering: no` 或 `proxy_buffering off` |
| data: [DONE] JSON parse error | 这不是 JSON,是特殊标记 | 显式判断 `if data_str == "[DONE]"` |
| tool_calls arguments 不完整 | 流式片段,每次只发一小段 | 按 index 累积拼接 arguments |
| 中文乱码 | 编码问题 | 确保 `Content-Type` 带 `charset=utf-8` |
| 连接被 CDN 缓存 | CDN 缓存了 text/event-stream | 加 `Cache-Control: no-cache` + `Surrogate-Control: no-store` |

---

## 6. 延伸阅读

- [MDN Server-Sent Events](https://developer.mozilla.org/en-US/docs/Web/API/Server-sent_events)
- [WHATWG SSE 规范](https://html.spec.whatwg.org/multipage/server-sent-events.html)
- [OpenAI Streaming 文档](https://platform.openai.com/docs/api-reference/streaming)
