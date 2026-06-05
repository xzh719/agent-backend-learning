# 02 — WebSocket 实时通信

> Agent 需要实时推送中间状态（thinking 过程、工具调用进度、多步执行中）。HTTP 轮询太浪费，WebSocket 是正确答案。

---

## 1. 概念讲解

### WebSocket 是什么？

WebSocket 是 **全双工、长连接** 的通信协议。它通过 HTTP 升级握手建立连接，之后双方可以在任意时刻发送数据，无需等待对方请求。

```
HTTP:      Client ──req──> Server ──res──> Client (结束)
           Client ──req──> Server ──res──> Client (结束)
           ...每次都要新建连接...

WebSocket: Client ──握手──> Server
           Client <──消息──> Server  ← 一条连接,双向自由收发
           Client <──消息──> Server
           ...连接保持,直到任一方关闭...
```

### 在 Agent 后端的应用场景

- 流式 token 输出（alternative to SSE）
- Agent 多步执行的实时进度推送
- 工具调用的中间结果通知
- 多人协作 Agent 的实时同步
- 前端与 Agent 后端的双向命令通道（用户中断、参数调整）

---

## 2. 核心原理

### 2.1 握手过程

**客户端发起 HTTP 升级请求：**
```http
GET /ws/chat HTTP/1.1
Host: agent.example.com
Upgrade: websocket
Connection: Upgrade
Sec-WebSocket-Key: dGhlIHNhbXBsZSBub25jZQ==   ← 随机base64
Sec-WebSocket-Version: 13
```

**服务端响应：**
```http
HTTP/1.1 101 Switching Protocols
Upgrade: websocket
Connection: Upgrade
Sec-WebSocket-Accept: s3pPLMBiTxaQ9kYGzzhZRbK+xOo=  ← SHA1(Key+固定GUID)
```

握手成功后，协议从 HTTP 切换到 WebSocket，后续数据用 **WebSocket 帧** 传输。

### 2.2 WebSocket 帧结构

```
 0               1               2               3
 0 1 2 3 4 5 6 7 0 1 2 3 4 5 6 7 0 1 2 3 4 5 6 7 0 1 2 3 4 5 6 7
+-+-+-+-+-------+-+-------------+-------------------------------+
|F|R|R|R| opcode|M| Payload len |    Extended payload length    |
|I|S|S|S|  (4)  |A|     (7)     |          (16/64)              |
|N|V|V|V|       |S|             |  (if payload len==126/127)    |
| |1|2|3|       |K|             |                               |
+-+-+-+-+-------+-+-------------+-------------------------------+
|     Masking-key (if MASK set to 1) — 客户端→服务端必须mask   |
+---------------------------------------------------------------+
|     Payload Data                                              |
+---------------------------------------------------------------+
```

关键字段：
- **FIN** (1 bit): 是否是最后一帧（大消息分片用）
- **Opcode** (4 bits): `0x1`=文本, `0x2`=二进制, `0x8`=关闭, `0x9`=Ping, `0xA`=Pong
- **MASK** (1 bit): 客户端→服务端 **必须** mask (防缓存投毒攻击)
- **Payload len**: 7/7+16/7+64 位变长编码

### 2.3 Ping/Pong 心跳

```
Client: Ping frame (opcode=0x9) ──────>
Server:                         <────── Pong frame (opcode=0xA)
```

- 用于保持连接存活（穿透 NAT/代理超时）
- 也可用于测量延迟
- **Pong 必须原样返回 Ping 的 Payload**

### 2.4 关闭握手

```
A: Close frame (opcode=0x8, status=1000) ───>
                                          <─── Close frame (opcode=0x8, status=1000)  B
```

常见关闭状态码：
- `1000` — 正常关闭
- `1001` — 端点离开（浏览器关闭标签页）
- `1006` — 异常关闭（连接丢失，不可发送）
- `1011` — 服务端内部错误
- `1012` — 服务重启

---

## 3. 代码实践

### 3.1 FastAPI WebSocket 服务端

```python
# ws_server.py
from fastapi import FastAPI, WebSocket, WebSocketDisconnect
from typing import dict
import asyncio
import json
import uuid

app = FastAPI()

class ConnectionManager:
    """管理所有 WebSocket 连接"""
    def __init__(self):
        self.connections: dict[str, WebSocket] = {}  # session_id -> ws

    async def connect(self, websocket: WebSocket) -> str:
        await websocket.accept()
        sid = str(uuid.uuid4())[:8]
        self.connections[sid] = websocket
        return sid

    def disconnect(self, sid: str):
        self.connections.pop(sid, None)

    async def send_json(self, sid: str, data: dict):
        ws = self.connections.get(sid)
        if ws:
            await ws.send_json(data)

    async def broadcast(self, data: dict):
        for ws in self.connections.values():
            await ws.send_json(data)

manager = ConnectionManager()


@app.websocket("/ws/agent/{session_id}")
async def agent_websocket(websocket: WebSocket, session_id: str):
    await websocket.accept()
    print(f"Agent connected: {session_id}")

    try:
        while True:
            # 接收客户端消息（可能是中断、参数调整等）
            raw = await websocket.receive_text()
            msg = json.loads(raw)
            action = msg.get("action")

            if action == "run":
                # 模拟 Agent 多步执行，实时推送进度
                task = msg.get("task", "")
                steps = [
                    ("thinking", "正在分析任务..."),
                    ("tool_call", "调用 search(query='...')"),
                    ("tool_result", "搜索结果: 3 条相关文档"),
                    ("thinking", "正在基于结果生成回复..."),
                    ("done", "这是最终回复：你好！..."),
                ]

                for step_type, content in steps:
                    await websocket.send_json({
                        "type": step_type,
                        "content": content,
                        "timestamp": asyncio.get_event_loop().time()
                    })
                    await asyncio.sleep(0.5)  # 模拟延迟

            elif action == "stop":
                await websocket.send_json({"type": "stopped", "content": "已中断"})
                break

            elif action == "ping":
                await websocket.send_json({"type": "pong", "content": "alive"})

    except WebSocketDisconnect:
        print(f"Agent disconnected: {session_id}")
```

### 3.2 Python WebSocket 客户端

```python
# ws_client.py
import asyncio
import websockets
import json

async def agent_client():
    uri = "ws://localhost:8000/ws/agent/my-session"

    async with websockets.connect(uri) as ws:
        # 发送任务
        await ws.send(json.dumps({
            "action": "run",
            "task": "帮我查一下今天的天气"
        }))

        # 接收实时推送
        async for raw in ws:
            msg = json.loads(raw)
            print(f"[{msg['type']}] {msg['content']}")

            if msg["type"] == "done":
                # 发送停止信号
                await ws.send(json.dumps({"action": "stop"}))
                break

asyncio.run(agent_client())
```

输出：
```
[thinking] 正在分析任务...
[tool_call] 调用 search(query='...')
[tool_result] 搜索结果: 3 条相关文档
[thinking] 正在基于结果生成回复...
[done] 这是最终回复：你好！...
```

---

## 4. Agent 场景应用

### 场景: Agent 思考过程可视化

WebSocket 天然适合推送 Agent 的 ReAct 循环：

```python
# Agent 内部的步骤会实时推送给前端
async def agent_react_loop(ws: WebSocket, task: str):
    messages = [{"role": "user", "content": task}]

    for iteration in range(MAX_STEPS):
        # 1. LLM 思考
        await ws.send_json({"type": "thinking_start", "step": iteration})
        thought = await call_llm(messages, tools=TOOLS)

        # 2. 流式推送思考内容
        if thought.content:
            await ws.send_json({"type": "thought", "content": thought.content})

        # 3. 工具调用
        if thought.tool_calls:
            for tc in thought.tool_calls:
                await ws.send_json({
                    "type": "tool_call_start",
                    "tool": tc.function.name,
                    "args": tc.function.arguments
                })
                result = await execute_tool(tc)
                await ws.send_json({
                    "type": "tool_result",
                    "tool": tc.function.name,
                    "result": str(result)[:200]
                })
        else:
            # 最终回复
            await ws.send_json({"type": "final", "content": thought.content})
            break
```

### WebSocket vs SSE 选择指南

| 特性 | WebSocket | SSE |
|------|-----------|-----|
| 通信方向 | 双向 | 单向（Server→Client） |
| 协议层级 | 独立协议 (ws://) | 基于 HTTP |
| 自动重连 | 需手动实现 | 浏览器内置 |
| 二进制数据 | 支持 | 不支持 |
| 穿透代理 | 可能有问题 | 与 HTTP 一致 |
| Agent 适用场景 | 需客户端控制（中断/参数） | 纯流式输出 |

---

## 5. 常见问题

| 问题 | 原因 | 解决 |
|------|------|------|
| 1006 异常断连 | 网络抖动/Nginx 超时 | 实现自动重连 + Ping/Pong 心跳 |
| Nginx 反代 426 错误 | 未配置 Upgrade 头 | 加 `proxy_set_header Upgrade $http_upgrade` |
| 消息丢失 | 网络瞬断时发送 | 应用层 ACK 确认机制 |
| 内存泄漏 | 僵尸连接未清理 | 定期清理 + 心跳超时断开 |
| Connection closed before accept | 客户端提前断开 | try/except WebSocketDisconnect |

---

## 6. 延伸阅读

- [RFC 6455 — WebSocket 协议规范](https://datatracker.ietf.org/doc/html/rfc6455)
- [FastAPI WebSocket 文档](https://fastapi.tiangolo.com/advanced/websockets/)
- [websockets Python 库](https://websockets.readthedocs.io/)
