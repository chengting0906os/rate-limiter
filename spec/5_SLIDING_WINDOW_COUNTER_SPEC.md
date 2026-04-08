# [5] Sliding Window Counter Rate Limiter - Spec

## 1. Overview

Sliding Window Counter 是 Fixed Window 的改良版，用前後兩個窗口的計數加權估算當前滑動窗口的請求數。
記憶體用量低（只存兩個計數器），精確度介於 Fixed Window 與 Sliding Window Log 之間。

**API 節點**: `POST /api/sliding-window-counter/ping`

---

## 2. User Story

- 作為 API 使用者，我打 `/api/sliding-window-counter/ping`，估算後未超限時得到 `200 OK`
- 作為 API 使用者，估算後超限時得到 `429 Too Many Requests`
- 作為開發者，只需維護兩個計數器，記憶體負擔低於 Sliding Window Log

---

## 3. Business Rules

1. **窗口大小**: 每個時間窗口為 `WINDOW_SIZE` 秒（預設 60）
2. **請求上限**: 估算後最多允許 `MAX_REQUESTS` 個請求（預設 10）
3. **加權計算公式**:
   ```
   elapsed = now % WINDOW_SIZE          # 當前窗口已過秒數
   weight = elapsed / WINDOW_SIZE       # 前窗口權重（0~1）
   estimated = prev_count * (1 - weight) + curr_count
   ```
4. **計數器更新**: 請求通過後，當前窗口計數 +1
5. **窗口切換**: 進入新窗口時，上一窗口計數移至 `prev_count`，`curr_count` 歸零
6. **Redis 儲存**: 使用兩個 key：
   - `rate_limit:sliding_window_counter:{client_id}:curr` （當前窗口計數，TTL = `WINDOW_SIZE * 2`）
   - `rate_limit:sliding_window_counter:{client_id}:prev` （前一窗口計數，TTL = `WINDOW_SIZE * 2`）
7. **原子性**: 讀取兩個計數器、計算、決策、更新需為原子操作（使用 Redis Lua script）

---

## 4. Acceptance

### 4.1 正常流量

- [ ] 打 `/api/sliding-window-counter/ping`，估算未超限時回傳 `200 OK`
- [ ] Response body 包含 `estimated_count`、`limit` 欄位
- [ ] 連續打 10 次，全部 `200 OK`

### 4.2 超出限制

- [ ] 估算超限時回傳 `429 Too Many Requests`
- [ ] `429` response body 包含 `retry_after` 欄位

### 4.3 加權行為

- [ ] 窗口剛開始時（elapsed 接近 0），前窗口計數的影響權重高
- [ ] 窗口接近結束時（elapsed 接近 WINDOW_SIZE），前窗口計數影響趨近於 0
- [ ] 跨窗口時，`prev_count` 正確更新為上一窗口的 `curr_count`

### 4.4 原子性

- [ ] 高併發下，估算計數正確，不超過 `MAX_REQUESTS`
