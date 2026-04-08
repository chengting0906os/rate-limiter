# [2] Sliding Window Log Rate Limiter - Spec

## 1. Overview

Sliding Window Log 記錄每個請求的精確時間戳，判斷時往回看固定時間長度的窗口。
精確度最高，沒有固定窗口邊界突刺問題，但記憶體使用較多。

**API 節點**: `POST /api/sliding-window-log/ping`

---

## 2. User Story

- 作為 API 使用者，我打 `/api/sliding-window-log/ping`，過去 60 秒內未超限時得到 `200 OK`
- 作為 API 使用者，過去 60 秒內請求數達到上限時得到 `429 Too Many Requests`
- 作為開發者，過期的時間戳自動清除，不需要手動維護

---

## 3. Business Rules

1. **窗口大小**: 往回看 `WINDOW_SIZE` 秒（預設 60）
2. **請求上限**: 窗口內最多允許 `MAX_REQUESTS` 個請求（預設 10）
3. **時間戳記錄**: 每次請求將當前 Unix timestamp（毫秒）加入 Redis Sorted Set
4. **過期清除**: 每次請求前先移除窗口外（`now - WINDOW_SIZE` 之前）的時間戳
5. **計數方式**: 清除後計算 Sorted Set 的元素數量，判斷是否超限
6. **Redis 儲存**: 使用 Redis Sorted Set，key 為 `rate_limit:sliding_window_log:{client_id}`，整體 TTL 設為 `WINDOW_SIZE`
7. **原子性**: 清除舊記錄、計數、加入新記錄需為原子操作（使用 Redis Lua script）

---

## 4. Acceptance

### 4.1 正常流量

- [ ] 打 `/api/sliding-window-log/ping`，過去 60 秒內未超限時回傳 `200 OK`
- [ ] Response body 包含 `current_count` 與 `limit` 欄位
- [ ] 連續打 10 次，全部 `200 OK`

### 4.2 超出限制

- [ ] 過去 60 秒內累計第 11 次請求回傳 `429 Too Many Requests`
- [ ] `429` response body 包含 `retry_after` 欄位

### 4.3 滑動窗口行為

- [ ] 60 秒前的請求不計入當前計數
- [ ] 早期請求滑出窗口後，新請求可再次通過（無需等到整個窗口結束）

### 4.4 原子性

- [ ] 高併發下，窗口內計數不超過 `MAX_REQUESTS`
