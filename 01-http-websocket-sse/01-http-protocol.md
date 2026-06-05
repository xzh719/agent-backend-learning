# 01 — HTTP 协议深入

> Agent 后端的一切通信都建立在 HTTP 之上。理解它的每一个细节，才能在调试时游刃有余。

---

## 1. 概念讲解

### HTTP 是什么？

HTTP (HyperText Transfer Protocol) 是一个 **应用层协议**，基于 TCP，采用 **请求-响应** 模型。它是 Web 的基石，也是 LLM API (OpenAI、Anthropic 等) 的通信基础。

### Agent 后端为什么需要深入 HTTP？

- LLM API 调用全是 HTTP POST → Chat Completions endpoint
- 流式输出依赖 HTTP 的 `Transfer-Encoding: chunked` 或 SSE
- 工具调用结果通过 HTTP 返回
- 调试 Agent 行为第一步就是看 HTTP 请求/响应

---

## 2. 核心原理

### 2.1 HTTP 报文结构

```
┌──────────────────────┐
│   Start Line         │  ← 请求: GET /api/chat HTTP/1.1
│                      │    响应: HTTP/1.1 200 OK
├──────────────────────┤
│   Headers            │  ← Content-Type, Authorization, ...
│   (key: value)       │
├──────────────────────┤
│   (空行 CRLF)        │  ← 分隔线
├──────────────────────┤
│   Body (可选)        │  ← JSON / form-data / binary
└──────────────────────┘
```

**关键点：** Header 和 Body 之间必须有一个空行 (`\r\n\r\n`)，这是协议层面的分隔符。

### 2.2 HTTP 方法语义

| 方法 | 语义 | Agent 场景 |
|------|------|------------|
| GET | 读取资源 | 获取 agent 状态、历史消息 |
| POST | 创建资源/提交数据 | 发送 prompt、调用 LLM |
| PUT | 完整替换资源 | 更新 agent 配置 |
| PATCH | 部分更新 | 修改单条记忆 |
| DELETE | 删除资源 | 清除会话 |
| OPTIONS | 预检请求 | CORS preflight |

### 2.3 状态码体系

```
1xx — 信息 (处理中)
  101 Switching Protocols  → WebSocket 升级
2xx — 成功
  200 OK                    → 标准成功
  201 Created               → 资源创建成功
  204 No Content            → 成功但无返回体 (常用于删除)
3xx — 重定向
  301/302                   → 永久/临时重定向
  304 Not Modified          → 缓存未过期
4xx — 客户端错误
  400 Bad Request           → JSON 格式错误 / Schema 不匹配
  401 Unauthorized          → 缺 API Key
  403 Forbidden             → 权限不足
  404 Not Found             → 模型名不存在
  429 Too Many Requests     → 触发速率限制 ← Agent高频踩坑点
5xx — 服务端错误
  500 Internal Server Error → 服务端 bug
  502 Bad Gateway           → 上游 (OpenAI) 挂了
  503 Service Unavailable   → 过载/维护中
  504 Gateway Timeout       → LLM 推理超时 ← 长文本生成常见
```

### 2.4 关键 Headers

```http
# 请求头
Authorization: Bearer sk-xxxx          # API Key 认证
Content-Type: application/json         # 请求体格式
Accept: text/event-stream              # 接受 SSE 流
User-Agent: MyAgent/1.0                # 标识自己 (API 方可能拒绝无 UA 的请求)

# 响应头
Content-Type: application/json
Content-Length: 1234                   # 响应体长度
Transfer-Encoding: chunked             # 分块传输 (流式)
X-Request-ID: req_abc123               # 请求追踪 ID (重要!)
X-RateLimit-Remaining: 499             # 剩余配额
Retry-After: 30                        # 429 后的重试等待秒数
```

---

## 3. 代码实践

### 3.1 用 Python aiohttp 调 OpenAI API（非流式）

```python
import aiohttp
import asyncio
import json

async def call_llm(prompt: str, api_key: str) -> dict:
    """一个完整的、带错误处理的 LLM 调用"""
    url = "https://api.openai.com/v1/chat/completions"

    headers = {
        "Authorization": f"Bearer {api_key}",
        "Content-Type": "application/json",
        "User-Agent": "AgentTutorial/1.0",
    }

    payload = {
        "model": "gpt-4o-mini",
        "messages": [{"role": "user", "content": prompt}],
        "temperature": 0.7,
    }

    async with aiohttp.ClientSession() as session:
        try:
            async with session.post(url, headers=headers, json=payload, timeout=30) as resp:
                # 1. 先看状态码
                print(f"Status: {resp.status}")
                print(f"X-Request-ID: {resp.headers.get('X-Request-ID', 'N/A')}")

                if resp.status == 429:
                    retry_after = resp.headers.get("Retry-After", "10")
                    print(f"Rate limited! Retry after {retry_after}s")
                    raise Exception(f"429: retry after {retry_after}s")

                if resp.status != 200:
                    body = await resp.text()
                    raise Exception(f"HTTP {resp.status}: {body}")

                # 2. 再读 body
                return await resp.json()

        except asyncio.TimeoutError:
            raise Exception("Request timed out — LLM inference took too long")
        except aiohttp.ClientError as e:
            raise Exception(f"Network error: {e}")

# 运行
# asyncio.run(call_llm("Hello!", "sk-xxx"))
```

### 3.2 构建一个极简 REST API (FastAPI)

```python
# server.py
from fastapi import FastAPI, HTTPException, Header
from pydantic import BaseModel
from typing import Optional
import uvicorn

app = FastAPI(title="Agent Backend Demo")

# --- 数据模型 ---
class ChatRequest(BaseModel):
    message: str
    session_id: Optional[str] = None

class ChatResponse(BaseModel):
    reply: str
    session_id: str

# --- 内存存储 ---
sessions = {}

# --- 路由 ---
@app.post("/chat", response_model=ChatResponse)
async def chat(req: ChatRequest, authorization: str = Header(None)):
    # 认证
    if not authorization or not authorization.startswith("Bearer "):
        raise HTTPException(status_code=401, detail="Missing API Key")

    api_key = authorization.split(" ")[1]
    # 这里应该验证 API Key...

    # 会话管理
    sid = req.session_id or f"sess_{len(sessions)+1}"
    if sid not in sessions:
        sessions[sid] = []

    sessions[sid].append({"role": "user", "content": req.message})

    # 模拟 Agent 响应
    reply = f"Echo: {req.message}"

    sessions[sid].append({"role": "assistant", "content": reply})

    return ChatResponse(reply=reply, session_id=sid)


@app.get("/sessions/{session_id}")
async def get_session(session_id: str):
    if session_id not in sessions:
        raise HTTPException(status_code=404, detail="Session not found")
    return {"session_id": session_id, "messages": sessions[session_id]}


@app.delete("/sessions/{session_id}", status_code=204)
async def delete_session(session_id: str):
    sessions.pop(session_id, None)
    return  # 204 No Content, no body

# if __name__ == "__main__":
#     uvicorn.run(app, host="0.0.0.0", port=8000)
```

运行方式：
```bash
pip install fastapi uvicorn aiohttp
python server.py
# 另一个终端测试:
curl -X POST http://localhost:8000/chat \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer test-key" \
  -d '{"message": "hello"}'
```

### 3.3 HTTP 抓包调试

```bash
# 用 curl -v 看完整请求/响应
curl -v https://api.openai.com/v1/chat/completions \
  -H "Authorization: Bearer $OPENAI_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"model":"gpt-4o-mini","messages":[{"role":"user","content":"hi"}]}'

# 关键输出解读:
# > POST /v1/chat/completions HTTP/1.1    ← 请求行
# > Host: api.openai.com
# > Authorization: Bearer sk-...
# > Content-Type: application/json
# >                                        ← 空行 = Header结束
# < HTTP/1.1 200 OK                       ← 响应状态
# < Content-Type: application/json
# < x-request-id: req_xxxx                 ← 用于报bug给OpenAI
```

---

## 4. Agent 场景应用

### 场景1: API Key 轮换

```python
import random

class APIKeyPool:
    """多 Key 池，自动避开 429 的 Key"""
    def __init__(self, keys: list[str]):
        self.keys = keys
        self.cooldown = {}  # key -> 解封时间

    def get_key(self) -> str:
        now = time.time()
        available = [k for k in self.keys
                     if self.cooldown.get(k, 0) < now]
        if not available:
            raise Exception("All keys are rate-limited!")
        return random.choice(available)

    def mark_rate_limited(self, key: str, retry_after: int):
        self.cooldown[key] = time.time() + retry_after
```

### 场景2: 请求重试中间件

```python
import asyncio

async def call_with_retry(fn, max_retries=3, base_delay=1):
    """指数退避 + Jitter 的重试"""
    for attempt in range(max_retries + 1):
        try:
            return await fn()
        except Exception as e:
            if "429" in str(e) or "5" in str(getattr(e, 'status', '')):
                if attempt == max_retries:
                    raise
                delay = base_delay * (2 ** attempt) + random.uniform(0, 1)
                print(f"Retry {attempt+1}/{max_retries} after {delay:.1f}s")
                await asyncio.sleep(delay)
            else:
                raise  # 非可恢复错误, 不重试
```

---

## 5. 常见问题

| 问题 | 原因 | 解决 |
|------|------|------|
| 429 Too Many Requests | 超过 RPM/TPM 限制 | 查看 `Retry-After` 头, 实现指数退避 |
| 401 Unauthorized | API Key 错误/过期 | 检查 Key, 是否需要 `Bearer` 前缀 |
| 504 Gateway Timeout | LLM 推理太慢 | 增大 timeout, 或切换到更快的模型 |
| 400 Bad Request | JSON 格式或 Schema 不对 | 打印完整请求体, 用 pydantic 校验 |
| `Connection reset by peer` | 服务端主动断开 | 检查是否使用了 HTTP/1.1 keep-alive 过久 |
| `SSL: CERTIFICATE_VERIFY_FAILED` | 证书问题 (公司代理) | 检查 `REQUESTS_CA_BUNDLE` 环境变量 |

---

## 6. 延伸阅读

- [MDN HTTP Overview](https://developer.mozilla.org/en-US/docs/Web/HTTP/Overview)
- [FastAPI 官方教程](https://fastapi.tiangolo.com/tutorial/)
- [HTTP 状态码列表](https://developer.mozilla.org/en-US/docs/Web/HTTP/Status)
- [aiohttp 文档](https://docs.aiohttp.org/en/stable/)
