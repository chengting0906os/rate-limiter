# Development Guide

## Architecture

Clean Architecture + Hexagonal Architecture (Ports & Adapters).
詳細結構見 [spec/0_tech_stack.md](spec/0_tech_stack.md)。

## TDD Workflow

每個功能依照以下順序開發：

1. **基本型別** — `domain/` 下定義 dataclass / model
2. **Port 介面** — `domain/ports/` 下定義抽象介面，方法 body 一律 `raise NotImplementedError`
3. **Domain 邏輯 skeleton** — `domain/rate_limiter/` 下建立 class，方法 body 一律 `raise NotImplementedError`
4. **寫測試** — `tests/unit/` 下針對 domain 邏輯寫測試，此時測試應全數 **FAIL**
5. **實作 Domain 邏輯** — 讓測試通過
6. **實作 Adapter** — `adapters/` 下實作 Redis store、FastAPI router

## Project Structure

```
src/
├── domain/
│   ├── models.py                    # 共用型別（RateLimitResult 等）
│   ├── ports/
│   │   └── i_rate_limit_store.py   # Port: 儲存層介面
│   └── rate_limiter/
│       ├── fixed_window.py          # [1]
│       ├── sliding_window_log.py    # [2]
│       ├── token_bucket.py          # [3]
│       ├── leaky_bucket.py          # [4]
│       └── sliding_window_counter.py # [5]
├── application/
│   └── rate_limit_service.py
└── adapters/
    ├── redis_store.py
    └── api/
        ├── main.py
        └── routers/
            ├── fixed_window.py
            ├── sliding_window_log.py
            ├── token_bucket.py
            ├── leaky_bucket.py
            └── sliding_window_counter.py

tests/
└── unit/
    ├── fixed_window_test.py
    ├── sliding_window_log_test.py
    ├── token_bucket_test.py
    ├── leaky_bucket_test.py
    └── sliding_window_counter_test.py
```

## Commands

```bash
# 啟動 Redis
docker compose up -d

# 執行測試
uv run pytest

# 執行特定測試
uv run pytest tests/unit/fixed_window_test.py

# Lint
uv run ruff check .

# 型別檢查
uv run pyright
```
