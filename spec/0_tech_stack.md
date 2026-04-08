# Tech Stack

## Runtime

| 技術 | 版本 | 用途 |
|------|------|------|
| Python | 3.13 | 主要語言 |
| FastAPI | 0.115.x (latest) | API 框架，提供 5 個限流節點 |
| Uvicorn | 0.34.x (latest) | ASGI server，執行 FastAPI |
| Redis | 8.x (latest) | 儲存限流狀態（計數器、時間戳、token 數量） |
| redis-py (asyncio) | 5.x (latest) | Python 非同步 Redis 客戶端 |

## Testing

| 技術 | 版本 | 用途 |
|------|------|------|
| pytest | 8.x (latest) | 測試框架 |
| pytest-asyncio | 0.25.x (latest) | 非同步測試支援 |
| httpx | 0.28.x (latest) | FastAPI 測試用 async HTTP client（替代 TestClient） |

## Code Quality

| 技術 | 用途 |
|------|------|
| ruff | Linter + Formatter |
| pyright | 靜態型別檢查 |

---

## 架構風格

採用 **Clean Architecture + Hexagonal Architecture（Ports & Adapters）** 開發。

### 核心原則

- **依賴方向**: 永遠由外層指向內層，Domain 不依賴任何框架或基礎設施
- **Port**: 內層定義的介面（interface），描述「需要什麼能力」
- **Adapter**: 外層的具體實作，實現 Port 定義的介面

### 分層結構（`src/`）

```
src/
├── domain/                  # 核心層（最內層，零依賴）
│   ├── rate_limiter/        # 各演算法的純邏輯與 Port 定義
│   │   ├── i_rate_limiter.py        # Port: 限流器介面
│   │   ├── fixed_window.py          # Domain: Fixed Window 演算法邏輯
│   │   ├── sliding_window_log.py
│   │   ├── token_bucket.py
│   │   ├── leaky_bucket.py
│   │   └── sliding_window_counter.py
│   └── ports/
│       └── i_rate_limit_store.py    # Port: 儲存層介面（不知道 Redis）
│
├── application/             # 應用層（編排 use case，依賴 domain）
│   └── rate_limit_service.py        # 呼叫 domain 邏輯 + port
│
├── adapters/                # 外層（具體實作，依賴 domain ports）
│   ├── redis_store.py               # Adapter: Redis 實作 i_rate_limit_store
│   └── api/
│       ├── main.py                  # FastAPI app
│       └── routers/
│           ├── fixed_window.py
│           ├── sliding_window_log.py
│           ├── token_bucket.py
│           ├── leaky_bucket.py
│           └── sliding_window_counter.py
```

### 請求流程

```
Client
  │
  ▼
FastAPI Router (Adapter)
  │  [1] /api/fixed-window/ping
  │  [2] /api/sliding-window-log/ping
  │  [3] /api/token-bucket/ping
  │  [4] /api/leaky-bucket/ping
  │  [5] /api/sliding-window-counter/ping
  │
  ▼
Application Service
  │
  ▼
Domain (演算法邏輯) ←→ Port: i_rate_limit_store
                              │
                              ▼
                        Redis Adapter（具體實作）
                              │
                              ▼
                            Redis
```

每個 endpoint 打進來後，先通過對應的 Rate Limiter 判斷：
- 允許 → `200 OK`
- 超出限制 → `429 Too Many Requests`
