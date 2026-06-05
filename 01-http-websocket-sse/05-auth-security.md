# 05 — 认证与安全基础

> Agent 后端的 API 暴露在网络中，需要认证。LLM API Key 的管理更是重中之重。安全不是可选项，是必须项。

---

## 1. 概念讲解

### Agent 后端面临的安全问题

```
                           ┌─────────────────┐
  用户/前端 ────API Key───> │   Agent Backend  │ ────LLM API Key───> OpenAI/Anthropic
                           │                  │
  攻击者 ──未授权访问──>    │   (你的服务)      │ ←──Key 泄露── 黑客
                           └─────────────────┘
```

**三个层面的安全：**
1. **用户认证** — 谁在用你的 Agent？(JWT / OAuth / API Key)
2. **LLM Key 管理** — 上游 API Key 怎么安全存储和使用？
3. **HTTPS/TLS** — 传输过程的加密防护

---

## 2. 核心原理

### 2.1 JWT (JSON Web Token) 详解

```
JWT 结构: header.payload.signature

eyJhbGciOi... .eyJzdWIiOi... .SflKxwRJ...
└── Base64 ──┘ └── Base64 ──┘ └── HMAC ──┘
   Header        Payload        Signature
```

**Header（算法声明）：**
```json
{"alg": "HS256", "typ": "JWT"}
```

**Payload（Claims — 声明）：**
```json
{
  "sub": "user_123",        // 主体 (subject) — 用户ID
  "name": "Alice",
  "iat": 1717000000,        // 签发时间 (issued at)
  "exp": 1717086400,        // 过期时间 (expires) ← 必须设!
  "role": "premium"         // 自定义 claim
}
```

**Signature（签名）：**
```
HMACSHA256(
  base64UrlEncode(header) + "." + base64UrlEncode(payload),
  secret_key
)
```

**验证流程：**
```
1. 收到 JWT → 用 Base64 解码 header + payload
2. 用同样的 secret 重新计算签名
3. 对比签名是否一致 → 一致 = 未被篡改
4. 检查 exp 是否过期
5. 检查其他 claims (iss, aud, ...)
```

**重要：JWT 只签名不加密！Payload 任何人 Base64 解码就能看到。**
用 JWE 做加密，或不在 payload 中放敏感数据。

### 2.2 OAuth 2.0 授权码流程

Agent 后端最常见的场景：**让用户的 Agent 访问用户的 GitHub / Slack / Google 账号。**

```
用户                 Agent 后端              第三方 (GitHub)
 │                      │                        │
 │  1. "连接 GitHub"    │                        │
 │─────────────────────>│                        │
 │                      │  2. 重定向到 GitHub     │
 │<─────────────────────│  /authorize?client_id  │
 │                      │  &redirect_uri=...     │
 │  3. 用户授权          │                        │
 │──────────────────────────────────────────────>│
 │                      │                        │
 │  4. 回调 code        │                        │
 │<─────────────────────│                        │
 │                      │  5. 用 code 换 token   │
 │                      │───────────────────────>│
 │                      │  6. access_token       │
 │                      │<───────────────────────│
 │                      │  7. 加密存储 token     │
 │                      │                        │
 │  8. "连接成功"        │                        │
 │<─────────────────────│                        │
```

**关键概念：**
- **Authorization Code** — 一次性、短期（10分钟）、只用于换 token
- **Access Token** — 实际调用 API 用的凭证，通常 1 小时过期
- **Refresh Token** — 长期凭证，用于刷新 Access Token（30天-永久）
- **PKCE** — 防止授权码拦截攻击（移动/SPA 必须用）

### 2.3 HTTPS/TLS 握手

```
Client                              Server
  │                                    │
  │ 1. ClientHello                    │
  │   (支持的加密套件, 随机数1)         │
  │───────────────────────────────────>│
  │                                    │
  │ 2. ServerHello                    │
  │   (选定加密套件, 随机数2, 证书)     │
  │<───────────────────────────────────│
  │                                    │
  │ 3. 验证证书                        │
  │   (CA 签名? 域名匹配? 未过期?)     │
  │                                    │
  │ 4. 生成 premaster secret          │
  │   用证书公钥加密发过去              │
  │───────────────────────────────────>│
  │                                    │
  │ 5. 双方独立计算 session key       │
  │   = f(随机数1, 随机数2, premaster) │
  │                                    │
  │ 6. Finished (加密)                │
  │<══════════════════════════════════>│
  │  后续全加密通信                     │
```

---

## 3. 代码实践

### 3.1 JWT 实现（FastAPI）

```python
# auth_jwt.py
from datetime import datetime, timedelta, timezone
from fastapi import FastAPI, Depends, HTTPException, Header
from fastapi.security import HTTPBearer, HTTPAuthorizationCredentials
import jwt

app = FastAPI()

# 配置
SECRET_KEY = "change-me-in-production-use-env-var"  # 用环境变量！
ALGORITHM = "HS256"
ACCESS_TOKEN_EXPIRE_MINUTES = 60

security = HTTPBearer()

# --- 工具函数 ---
def create_access_token(user_id: str, role: str = "user") -> str:
    """生成 JWT"""
    payload = {
        "sub": user_id,
        "role": role,
        "iat": datetime.now(timezone.utc),
        "exp": datetime.now(timezone.utc) + timedelta(minutes=ACCESS_TOKEN_EXPIRE_MINUTES),
    }
    return jwt.encode(payload, SECRET_KEY, algorithm=ALGORITHM)

def decode_token(token: str) -> dict:
    """解码并验证 JWT"""
    try:
        payload = jwt.decode(
            token,
            SECRET_KEY,
            algorithms=[ALGORITHM],
            options={"require": ["exp", "sub"]}  # 必须包含 exp 和 sub
        )
        return payload
    except jwt.ExpiredSignatureError:
        raise HTTPException(status_code=401, detail="Token expired")
    except jwt.InvalidTokenError:
        raise HTTPException(status_code=401, detail="Invalid token")

# --- 依赖注入：从请求中提取当前用户 ---
async def get_current_user(
    credentials: HTTPAuthorizationCredentials = Depends(security)
) -> dict:
    token = credentials.credentials  # "Bearer xxx" → "xxx"
    return decode_token(token)

# --- 路由 ---
@app.post("/login")
async def login(username: str, password: str):
    # 实际应该验证用户名密码...
    if username == "admin" and password == "admin123":
        token = create_access_token(user_id="user_1", role="admin")
        return {"access_token": token, "token_type": "bearer"}
    raise HTTPException(status_code=401, detail="Invalid credentials")

@app.get("/me")
async def me(user: dict = Depends(get_current_user)):
    return {"user_id": user["sub"], "role": user["role"]}

@app.get("/admin-only")
async def admin_only(user: dict = Depends(get_current_user)):
    if user.get("role") != "admin":
        raise HTTPException(status_code=403, detail="Admin only")
    return {"secret": "you're admin!"}
```

### 3.2 API Key 管理最佳实践

```python
# api_key_manager.py
import os
import hashlib
import secrets
from typing import Optional

class APIKeyManager:
    """生产级 API Key 管理"""

    def __init__(self):
        # ⚠️ 永远不要硬编码 Key！
        self.llm_keys = self._load_keys()

    def _load_keys(self) -> dict[str, str]:
        """从环境变量加载多个 Provider 的 Key"""
        keys = {}
        for provider in ["openai", "anthropic", "google", "deepseek"]:
            env_var = f"{provider.upper()}_API_KEY"
            key = os.getenv(env_var)
            if key:
                keys[provider] = key
        return keys

    def get_key(self, provider: str) -> Optional[str]:
        key = self.llm_keys.get(provider)
        if not key:
            raise ValueError(f"API Key for {provider} not configured")
        return key

    # --- 用户 API Key 管理 (给你的用户用的) ---
    @staticmethod
    def generate_user_key(prefix: str = "agent") -> str:
        """生成用户 API Key: agent_xxxxxxxxxxxxxxxxxxxxxxxx"""
        random_part = secrets.token_hex(16)
        full_key = f"{prefix}_{random_part}"
        # 只存哈希！不存明文！
        hashed = hashlib.sha256(full_key.encode()).hexdigest()
        return full_key, hashed

    @staticmethod
    def verify_user_key(provided_key: str, stored_hash: str) -> bool:
        """验证用户提供的 Key 是否匹配"""
        return hashlib.sha256(provided_key.encode()).hexdigest() == stored_hash

    # --- 日志中脱敏 ---
    @staticmethod
    def mask_key(key: str) -> str:
        """sk-a1b2c3...8z9 → sk-a1b2***8z9"""
        if len(key) <= 12:
            return "***"
        return key[:6] + "***" + key[-4:]
```

### 3.3 OAuth 2.0 简化实现

```python
# oauth_github.py
from fastapi import FastAPI, Request
from fastapi.responses import RedirectResponse
import httpx
import os

app = FastAPI()

GITHUB_CLIENT_ID = os.getenv("GITHUB_CLIENT_ID")
GITHUB_CLIENT_SECRET = os.getenv("GITHUB_CLIENT_SECRET")
REDIRECT_URI = "https://your-agent.com/oauth/github/callback"

@app.get("/oauth/github/connect")
async def github_connect():
    """Step 1: 重定向用户到 GitHub 授权页"""
    params = {
        "client_id": GITHUB_CLIENT_ID,
        "redirect_uri": REDIRECT_URI,
        "scope": "repo read:user",           # 请求的权限
        "state": secrets.token_urlsafe(16),  # 防 CSRF 的随机字符串
        "allow_signup": "false",
    }
    qs = "&".join(f"{k}={v}" for k, v in params.items())
    return RedirectResponse(f"https://github.com/login/oauth/authorize?{qs}")


@app.get("/oauth/github/callback")
async def github_callback(code: str, state: str):
    """Step 2: 用 code 换 access_token"""
    # ⚠️ 实际应该验证 state（防 CSRF）

    async with httpx.AsyncClient() as client:
        # 用 code 换 token
        resp = await client.post(
            "https://github.com/login/oauth/access_token",
            data={
                "client_id": GITHUB_CLIENT_ID,
                "client_secret": GITHUB_CLIENT_SECRET,
                "code": code,
                "redirect_uri": REDIRECT_URI,
            },
            headers={"Accept": "application/json"}
        )
        token_data = resp.json()

    if "error" in token_data:
        raise HTTPException(400, f"OAuth failed: {token_data['error']}")

    access_token = token_data["access_token"]

    # Step 3: 用 access_token 获取用户信息
    resp = await client.get(
        "https://api.github.com/user",
        headers={"Authorization": f"Bearer {access_token}"}
    )
    user_info = resp.json()

    # Step 4: 存储 token (加密!) 并关联到你的用户
    encrypted_token = encrypt_token(access_token)  # 实现略
    save_user_integration(user_info["id"], "github", encrypted_token)

    return {"message": "GitHub connected!", "user": user_info["login"]}
```

---

## 4. Agent 场景应用

### 多租户 API Key 隔离

```python
class TenantKeyManager:
    """每个租户/用户使用自己的 LLM API Key"""

    def __init__(self):
        self.tenant_keys: dict[str, dict[str, str]] = {}  # tenant_id -> {provider: key}

    def set_tenant_key(self, tenant_id: str, provider: str, key: str):
        if tenant_id not in self.tenant_keys:
            self.tenant_keys[tenant_id] = {}
        self.tenant_keys[tenant_id][provider] = encrypt(key)  # 存储加密

    def get_tenant_key(self, tenant_id: str, provider: str) -> str:
        encrypted = self.tenant_keys.get(tenant_id, {}).get(provider)
        if not encrypted:
            raise ValueError(f"No key configured for {tenant_id}/{provider}")
        return decrypt(encrypted)
```

### Rate Limiting 中间件

```python
from collections import defaultdict
import time

class RateLimiter:
    """简单的滑动窗口速率限制"""
    def __init__(self, max_requests: int, window_seconds: int = 60):
        self.max_requests = max_requests
        self.window = window_seconds
        self.requests: dict[str, list[float]] = defaultdict(list)

    def is_allowed(self, key: str) -> bool:
        now = time.time()
        cutoff = now - self.window

        # 清理过期记录
        self.requests[key] = [t for t in self.requests[key] if t > cutoff]

        if len(self.requests[key]) >= self.max_requests:
            return False

        self.requests[key].append(now)
        return True

# FastAPI 中间件
@app.middleware("http")
async def rate_limit_middleware(request: Request, call_next):
    key = request.headers.get("Authorization", "anonymous")
    if not limiter.is_allowed(key):
        return JSONResponse(
            status_code=429,
            content={"detail": "Too many requests", "retry_after": 60}
        )
    return await call_next(request)
```

---

## 5. 常见问题

| 问题 | 原因 | 解决 |
|------|------|------|
| JWT 泄露 → 账户被盗 | JWT 不加密 | 用 HTTPS、短过期时间、Refresh Token 轮换 |
| API Key 泄露到 Git | 硬编码在代码里 | 用环境变量、`.env` + `.gitignore`、Vault |
| 429 被 LLM API 限流 | RPM/TPM 超限 | 多 Key 轮换、请求队列、用户级别限流 |
| OAuth state 参数被篡改 | CSRF | 生成随机 state、回调时验证 |
| 日志中暴露 Key | 直接 log 了请求 | 脱敏中间件，mask 前 6 + 后 4 位 |

---

## 6. 延伸阅读

- [JWT 官方文档](https://jwt.io/introduction)
- [OAuth 2.0 简化指南](https://aaronparecki.com/oauth-2-simplified/)
- [OWASP API 安全 Top 10](https://owasp.org/www-project-api-security/)
- [FastAPI Security 教程](https://fastapi.tiangolo.com/tutorial/security/)
