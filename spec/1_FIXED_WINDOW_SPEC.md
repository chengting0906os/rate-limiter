# [1] Fixed Window Counter Rate Limiter - Spec

## 1. Overview

Fixed Window Counter 將時間切成固定大小的窗口（如每 60 秒一格），
每個窗口內計算請求數量，超出上限則拒絕。實作最簡單，但窗口邊界有流量突刺問題。

**API 節點**: `POST /api/fixed-window/ping`

---

## 2. User Story

- 作為 API 使用者，我打 `/api/fixed-window/ping`，當前窗口內未超限時得到 `200 OK`
- 作為 API 使用者，當前窗口請求數達到上限時得到 `429 Too Many Requests`
- 作為開發者，窗口結束後計數自動歸零，不需要手動重置

---

## 3. Business Rules

1. **窗口大小**: 每個時間窗口為 `WINDOW_SIZE` 秒（預設 60）
2. **請求上限**: 每個窗口內最多允許 `MAX_REQUESTS` 個請求（預設 10）
3. **窗口計算**: 窗口以 Unix timestamp 整除 `WINDOW_SIZE` 決定（`ts // WINDOW_SIZE`）
4. **計數重置**: 進入新窗口時計數從 0 開始，使用 Redis key TTL 自動過期實現
5. **Redis 儲存**: key 為 `rate_limit:fixed_window:{client_id}:{window_id}`，TTL 設為 `WINDOW_SIZE`
6. **原子性**: 計數的讀取與遞增需為原子操作（使用 Redis `INCR`）

---

## 4. Acceptance

### 4.1 正常流量

- [ ] 打 `/api/fixed-window/ping`，當前窗口未超限時回傳 `200 OK`
- [ ] Response body 包含 `current_count` 與 `limit` 欄位
- [ ] 同一窗口內連續打 10 次，全部 `200 OK`

### 4.2 超出限制

- [ ] 同一窗口內第 11 次請求回傳 `429 Too Many Requests`
- [ ] `429` response body 包含 `retry_after` 欄位，告知當前窗口剩餘秒數

### 4.3 窗口重置

- [ ] 進入新窗口後，計數歸零，請求可再次通過
- [ ] 新窗口的 `current_count` 從 1 開始（第一筆請求後）

### 4.4 原子性

- [ ] 高併發下，同一窗口內計數不超過 `MAX_REQUESTS`
