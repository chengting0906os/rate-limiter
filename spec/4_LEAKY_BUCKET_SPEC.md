# [4] Leaky Bucket Rate Limiter - Spec

## 1. Overview

Leaky Bucket 以固定速率「漏水」（處理請求），進來的請求先進入佇列（bucket），
超出佇列容量則直接丟棄。特色是輸出流量平滑，不允許突發。

**API 節點**: `POST /api/leaky-bucket/ping`

---

## 2. User Story

- 作為 API 使用者，我打 `/api/leaky-bucket/ping`，如果佇列未滿，我會得到 `200 OK`
- 作為 API 使用者，當佇列已滿時，我會得到 `429 Too Many Requests`
- 作為開發者，我希望請求以固定速率被處理，輸出流量平穩不暴衝

---

## 3. Business Rules

1. **桶容量**: 佇列最多容納 `CAPACITY` 個請求（預設 10）
2. **漏出速率**: 每秒漏出 `LEAK_RATE` 個請求（預設 1）
3. **初始狀態**: 服務啟動時佇列為空（`queue_size = 0`）
4. **進入規則**: 請求到達時，先根據時間差計算漏出數量，更新佇列大小，再判斷是否能加入
5. **丟棄規則**: 佇列已滿（`queue_size >= CAPACITY`）時，請求直接回傳 `429`
6. **Redis 儲存**: 佇列大小與最後漏出時間存於 Redis，key 為 `rate_limit:leaky_bucket:{client_id}`
7. **原子性**: 佇列大小的讀取與更新需為原子操作（使用 Redis Lua script）

---

## 4. Acceptance

### 4.1 正常流量

- [ ] 打 `/api/leaky-bucket/ping`，佇列未滿時回傳 `200 OK`
- [ ] Response body 包含 `queue_size` 欄位，顯示目前佇列使用量
- [ ] 連續打 10 次，全部 `200 OK`（初始佇列為空時）

### 4.2 超出限制

- [ ] 佇列滿後打 `/api/leaky-bucket/ping` 回傳 `429 Too Many Requests`
- [ ] `429` response body 包含 `retry_after` 欄位

### 4.3 漏出行為

- [ ] 佇列滿後等待足夠時間，再打可得 `200 OK`（佇列已漏出）
- [ ] 漏出後佇列大小正確減少，不低於 0

### 4.4 原子性

- [ ] 高併發下，佇列大小不會超過 `CAPACITY`
