# 阶段1 完成清单

| # | 主题 | 文件 | 状态 |
|---|------|------|------|
| 1 | HTTP 协议深入 | `01-http-protocol.md` | ✅ |
| 2 | WebSocket 实时通信 | `02-websocket.md` | ✅ |
| 3 | SSE 服务端推送 | `03-sse.md` | ✅ |
| 4 | 异步编程模型 | `04-async-programming.md` | ✅ |
| 5 | 认证与安全基础 | `05-auth-security.md` | ✅ |

## 实践项目建议

做一个 **最小 Agent 后端**，整合本阶段所有知识：

```
项目结构:
mini-agent/
├── server.py          # FastAPI + WebSocket + SSE
├── auth.py            # JWT 认证
├── llm_client.py      # aiohttp 异步调用 OpenAI
├── config.py          # 环境变量加载
└── .env.example       # 配置模板
```

功能:
- POST /chat — 非流式聊天
- GET /chat/stream — SSE 流式输出
- WS /ws/agent/{id} — WebSocket 实时双向通信
- JWT 认证保护所有接口
