# [3] Token Bucket Rate Limiter - Spec

## 1. Overview

Token Bucket 以固定速率補充令牌（token），每個請求消耗一個令牌。
桶有容量上限，允許短時間的突發流量（burst），只要桶內有令牌就能通過。

**API 節點**: `POST /api/token-bucket/ping`

---

## 2. User Story

- 作為 API 使用者，我打 `/api/token-bucket/ping`，如果桶內有令牌，我會得到 `200 OK`
- 作為 API 使用者，當令牌耗盡時，我會得到 `429 Too Many Requests`
- 作為開發者，我希望令牌以固定速率自動補充，不需要手動觸發

---

## 3. Business Rules

1. **桶容量**: 桶最多儲存 `CAPACITY` 個令牌（預設 10）
2. **補充速率**: 每秒補充 `REFILL_RATE` 個令牌（預設 1）
3. **初始狀態**: 服務啟動時桶為滿的（`tokens = CAPACITY`）
4. **消耗規則**: 每個請求消耗 1 個令牌
5. **補充上限**: 補充後令牌數不超過 `CAPACITY`
6. **補充時機**: 以 lazy refill 方式，在每次請求時根據距上次請求的時間計算應補充的令牌數
7. **Redis 儲存**: 令牌數量與最後補充時間存於 Redis，key 為 `rate_limit:token_bucket:{client_id}`
8. **原子性**: 令牌的讀取與扣除需為原子操作（使用 Redis Lua script）

---

## 4. Acceptance

### 4.1 正常流量

- [ ] 打 `/api/token-bucket/ping`，桶內有令牌時回傳 `200 OK`
- [ ] Response body 包含 `remaining_tokens` 欄位，顯示剩餘令牌數
- [ ] 連續打 10 次，全部 `200 OK`（桶滿時）

### 4.2 超出限制

- [ ] 令牌耗盡後打 `/api/token-bucket/ping` 回傳 `429 Too Many Requests`
- [ ] `429` response body 包含 `retry_after` 欄位（秒數），告知多久後可重試

### 4.3 令牌補充

- [ ] 令牌耗盡後等待足夠時間，再打可得 `200 OK`
- [ ] 補充後的令牌數不超過 `CAPACITY`

### 4.4 原子性

- [ ] 高併發下（多個請求同時到達），令牌扣除不會超過實際數量
