# 04 — 异步编程模型

> Agent 后端天然是高并发的：同时处理多个会话，每个会话可能调用多个 LLM API、执行多个工具。同步阻塞 = 不可接受。异步是必修课。

---

## 1. 概念讲解

### 为什么 Agent 必须异步？

```
同步模型 (一个请求处理完才能处理下一个)：
  Request A: [=====等待LLM API(3s)=====][=====等待工具(1s)=====]
  Request B:                                                      [=====等待LLM(3s)=====]
  总耗时: 8s

异步模型 (等待时处理其他请求)：
  Request A: [=====等待LLM API(3s)=====][=====等待工具(1s)=====]
  Request B:    [=====等待LLM(3s)=====]
  总耗时: ~4s  ← 并发度越高,节省越明显
```

**Agent 中的典型 IO 密集型操作：**
- 调用 LLM API (网络 IO, 100ms-30s)
- 向量数据库查询 (网络 IO, 10-100ms)
- 工具执行 (文件 IO、外部 API、子进程)
- 数据库读写 (网络/磁盘 IO)

所有这些都适合异步。

---

## 2. 核心原理

### 2.1 Python asyncio 事件循环

```
┌─────────────────────────────────────────────────┐
│                 Event Loop                       │
│                                                  │
│  ┌──────┐   ┌──────┐   ┌──────┐                │
│  │Task A │   │Task B │   │Task C │    ...       │
│  └──┬───┘   └──┬───┘   └──┬───┘                │
│     │ await    │ await    │ await               │
│     ▼          ▼          ▼                      │
│  ┌──────────────────────────────────┐            │
│  │      IO Waiting (epoll)          │            │
│  │   network / file / subprocess    │            │
│  └──────────────────────────────────┘            │
└─────────────────────────────────────────────────┘
```

- **单线程** 运行所有协程（不是多线程！）
- `await` = "我先挂起，你处理别人，我好了叫我"
- IO 操作注册到 OS 的 `epoll`/`kqueue`，由 OS 通知事件循环

### 2.2 协程 (Coroutine)、任务 (Task)、Future

```python
# 协程函数 (async def) — 返回协程对象
async def my_coro():
    await asyncio.sleep(1)
    return 42

coro = my_coro()  # 只是创建，还没运行，返回 coroutine 对象

# Task — 把协程提交到事件循环去调度
task = asyncio.create_task(my_coro())  # 立即调度！
result = await task  # 等它完成，得到 42

# Future — Task 的基类，低层 API
# 几乎总是用 Task 就够了
```

**关键区别：**
```python
# 错误写法：串行执行
for i in range(10):
    await call_llm(...)  # 一次等一个，10个 = 10x 时间

# 正确写法：并发执行
tasks = [asyncio.create_task(call_llm(...)) for i in range(10)]
results = await asyncio.gather(*tasks)  # 10个同时跑！
```

### 2.3 GIL 的影响

```
CPU 密集型    → GIL 限制 → 用多进程(multiprocessing) 或 run_in_executor
IO 密集型     → GIL 不影响 → 用 asyncio 即可

Agent 后端99%是 IO 密集型 → asyncio 是正确答案
只在训练/预处理等场景才需要多进程
```

### 2.4 不要在异步中调用同步阻塞函数

```python
# ❌ 灾难写法 — 阻塞整个事件循环
async def bad():
    time.sleep(5)        # 所有协程都被卡住 5 秒！
    requests.get(url)    # 同步 HTTP 库也阻塞

# ✅ 正确写法
async def good():
    await asyncio.sleep(5)
    async with aiohttp.ClientSession() as s:
        async with s.get(url) as resp:
            ...

# ✅ 万不得已(旧同步库) — 丢到线程池
def sync_blocking():
    time.sleep(5)
    return "done"

async def wrapper():
    loop = asyncio.get_event_loop()
    return await loop.run_in_executor(None, sync_blocking)
```

---

## 3. 代码实践

### 3.1 并发 LLM 调用模式

```python
import asyncio
import aiohttp

async def call_llm(session: aiohttp.ClientSession, prompt: str) -> str:
    """单个 LLM 调用"""
    async with session.post(
        "https://api.openai.com/v1/chat/completions",
        headers={"Authorization": f"Bearer {API_KEY}"},
        json={"model": "gpt-4o-mini", "messages": [{"role": "user", "content": prompt}]}
    ) as resp:
        data = await resp.json()
        return data["choices"][0]["message"]["content"]


async def batch_llm_calls(prompts: list[str]) -> list[str]:
    """并发调用多个 LLM 请求"""
    async with aiohttp.ClientSession() as session:
        tasks = [call_llm(session, p) for p in prompts]
        results = await asyncio.gather(*tasks, return_exceptions=True)

    # 处理异常
    outputs = []
    for i, r in enumerate(results):
        if isinstance(r, Exception):
            outputs.append(f"Error: {r}")
        else:
            outputs.append(r)
    return outputs

# 使用
# results = asyncio.run(batch_llm_calls(["prompt1", "prompt2", "prompt3"]))
```

### 3.2 Producer-Consumer 模式（适合 Agent Pipeline）

```python
import asyncio
from asyncio import Queue

async def llm_producer(queue: Queue, prompts: list[str]):
    """生产者：发送请求，结果放入队列"""
    async with aiohttp.ClientSession() as session:
        for prompt in prompts:
            result = await call_llm(session, prompt)
            await queue.put(result)
        # 发送结束信号
        for _ in range(NUM_CONSUMERS):
            await queue.put(None)

async def result_consumer(queue: Queue, consumer_id: int):
    """消费者：从队列取结果，做后续处理"""
    while True:
        result = await queue.get()
        if result is None:
            break
        # 处理结果：存DB、分词、embedding...
        print(f"Consumer {consumer_id}: processing {result[:50]}...")
        await asyncio.sleep(0.1)  # 模拟处理时间

async def pipeline(prompts: list[str]):
    queue = Queue(maxsize=10)  # 背压控制
    producer = asyncio.create_task(llm_producer(queue, prompts))
    consumers = [
        asyncio.create_task(result_consumer(queue, i))
        for i in range(NUM_CONSUMERS)
    ]
    await asyncio.gather(producer, *consumers)
```

### 3.3 超时与取消

```python
async def call_with_timeout(prompt: str, timeout: float = 30):
    """带超时的 LLM 调用"""
    try:
        async with asyncio.timeout(timeout):
            return await call_llm(session, prompt)
    except asyncio.TimeoutError:
        print(f"LLM call timed out after {timeout}s")
        return None

async def call_with_cancellation(prompt: str):
    """可被取消的 LLM 调用"""
    task = asyncio.create_task(call_llm(session, prompt))
    try:
        return await task
    except asyncio.CancelledError:
        print("LLM call was cancelled")
        # 做清理工作...
        raise  # 一定要重新 raise!
```

---

## 4. Agent 场景应用

### Agent 并发执行工具

```python
async def execute_tools_parallel(tool_calls: list) -> list:
    """并行执行多个工具调用（当工具之间无依赖时）"""
    tasks = []
    for tc in tool_calls:
        tool_fn = TOOL_REGISTRY[tc.function.name]
        args = json.loads(tc.function.arguments)
        tasks.append(asyncio.create_task(tool_fn(**args)))

    results = await asyncio.gather(*tasks, return_exceptions=True)
    return results
```

### 会话隔离与并发控制

```python
import asyncio
from collections import defaultdict

class AgentSessionManager:
    def __init__(self, max_concurrent_sessions=50):
        self.semaphore = asyncio.Semaphore(max_concurrent_sessions)
        self.sessions = {}

    async def process_message(self, session_id: str, message: str):
        """每个会话的消息串行处理，但不同会话并发"""
        if session_id not in self.sessions:
            self.sessions[session_id] = asyncio.Lock()

        async with self.sessions[session_id]:  # 同一会话串行
            async with self.semaphore:          # 全局并发限制
                result = await self._agent_loop(session_id, message)
                return result

    async def _agent_loop(self, session_id: str, message: str):
        # ... ReAct 循环 ...
        pass
```

---

## 5. 常见问题

| 问题 | 原因 | 解决 |
|------|------|------|
| `RuntimeWarning: coroutine was never awaited` | 创建了协程但没 await | 用 `asyncio.create_task()` 或 `await` |
| 事件循环卡死 | 同步阻塞代码 (time.sleep, requests) | 改用异步版本 或 `run_in_executor` |
| `Task was destroyed but it is pending!` | 任务没完成就销毁了 | 确保 `await` 或 `.cancel()` |
| 并发数爆炸 | 无限制创建 Task | 用 `asyncio.Semaphore` 限流 |
| 内存泄漏 | 大量未完成 Task 堆积 | 限制队列大小，定期清理 |

---

## 6. Node.js 的异步模型对比

```javascript
// Node.js 的异步也很重要（很多 Agent 工具用 Node）

// Promise + async/await (和 Python 非常像)
async function callLLM(prompt) {
  const resp = await fetch("https://api.openai.com/v1/chat/completions", { ... });
  return resp.json();
}

// 并发
const results = await Promise.all([
  callLLM("prompt1"),
  callLLM("prompt2"),
]);

// 事件循环: libuv (类似 epoll)
// 单线程 + worker threads (类似 run_in_executor)
```

---

## 7. 延伸阅读

- [Python asyncio 官方文档](https://docs.python.org/3/library/asyncio.html)
- [Real Python: Async IO in Python](https://realpython.com/async-io-python/)
- [aiohttp 文档](https://docs.aiohttp.org/)
- [Node.js Event Loop 官方指南](https://nodejs.org/en/learn/asynchronous-work/event-loop-timers-and-nexttick)
