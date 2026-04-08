# API 合約：書籍買賣與交流平台（book-exchange-platform）

> 本文件描述對外與內部 API 介面。實作採 **tRPC**，但本文件以 REST 風格列出便於人工審閱與第三方理解；tRPC procedure 名稱與此 1:1 對應。
>
> **慣例**：
> - 所有公開讀取（瀏覽 / 搜尋 / 商品詳情）**不需登入**。
> - 寫入與個資相關 endpoint 必須登入；管理後台須額外通過 RBAC role guard。
> - 所有 4xx / 5xx 回應使用統一錯誤格式：`{ code, message, requestId }`。
> - 時間欄位使用 ISO 8601 with timezone。

---

## 1. 認證 / 帳號（identity）

| 方法 | 路徑 | 用途 | 認證 |
|------|------|------|------|
| GET | `/api/auth/[...nextauth]` | Auth.js v5 universal handler（Line / Google） | - |
| POST | `/api/users/me/profile` | 註冊後補資料（暱稱、預設取書超商） | 已登入 |
| GET | `/api/users/me` | 取得自身基本資訊 | 已登入 |
| POST | `/api/users/me/data-export` | 申請下載個資（PDPA） | 已登入 |
| POST | `/api/users/me/delete` | 申請刪除帳號 | 已登入 |
| GET | `/api/users/me/pii-access-logs` | 查詢誰存取過我的個資（CONFLICT-004） | 已登入 |

---

## 2. 書籍 catalog（catalog）

| 方法 | 路徑 | 用途 | 認證 |
|------|------|------|------|
| GET | `/api/books/lookup?isbn=...` | ISBN 查詢（含 Google Books / 國圖 fallback） | - |
| GET | `/api/categories` | 列出全部類別（含 risk_level） | - |

---

## 3. 刊登（listing）

| 方法 | 路徑 | 用途 | 認證 |
|------|------|------|------|
| POST | `/api/listings` | 建立刊登（極簡或高風險路徑） | 已登入 |
| GET | `/api/listings/:id` | 取得單一刊登 | - |
| PATCH | `/api/listings/:id` | 編輯刊登 | 賣家本人 |
| POST | `/api/listings/:id/take-down` | 下架自己的刊登 | 賣家本人 |
| POST | `/api/listings/:id/photos` | 預先請求 R2 上傳 URL（pre-signed） | 賣家本人 |
| GET | `/api/listings/me` | 我的刊登列表 | 已登入 |
| GET | `/api/listings/:id/pricing-hint` | 取得建議售價 + 雙顯示金額 | - |

**關鍵 request body：POST /api/listings**

```json
{
  "bookId": "uuid",
  "categoryId": "uuid",
  "conditionTier": "good|normal|defect",
  "price": 120,
  "photos": [
    { "kind": "cover", "r2Key": "..." },
    { "kind": "condition", "r2Key": "..." },
    { "kind": "copyright_page", "r2Key": "..." },
    { "kind": "inner_page", "r2Key": "..." }
  ]
}
```

服務端會依 `categoryId` 對應的 `risk_level` 自動驗證高風險類別必填欄位（CONFLICT-001 + 003）。

---

## 4. 搜尋（search）

| 方法 | 路徑 | 用途 | 認證 |
|------|------|------|------|
| GET | `/api/search?q=...&sort=...&lat=...&lng=...` | 全文檢索 + 多排序 | - |
| GET | `/api/search/popular` | 熱門關鍵字 | - |
| POST | `/api/watchlists` | 加入追蹤 | 已登入 |
| DELETE | `/api/watchlists/:id` | 取消追蹤 | 已登入 |

**`sort` 列舉**：`total_cost_asc` / `condition_best` / `distance_asc`

效能 SLA：P95 ≤ 1s（一般文字搜尋條件）。

---

## 5. 訂單 / 付款 / 物流 / escrow

| 方法 | 路徑 | 用途 | 認證 |
|------|------|------|------|
| POST | `/api/orders` | 建立訂單（鎖定 listing） | 已登入 |
| GET | `/api/orders/:id` | 取得訂單與狀態 | 買賣家任一 |
| GET | `/api/orders/me` | 我的訂單列表 | 已登入 |
| POST | `/api/orders/:id/early-close` | **提早結案按鈕**（取貨後 24h 才開放） | 買家 |
| POST | `/api/orders/:id/dispute` | 鑑賞期內發起爭議 | 買家 |
| POST | `/api/payments/ecpay/callback` | ECPay 金流 callback（驗章必要） | webhook |
| POST | `/api/logistics/ecpay/callback` | ECPay 物流 callback（取貨通知） | webhook |
| GET | `/api/orders/:id/invoice` | 取得服務費電子發票（CONFLICT-006） | 買家 |

**訂單狀態列舉**（與 plan.md 狀態機一致）：

```
created | paid | seller_shipping | in_transit | arrived_at_store
| picked_up | completed | disputed | refunded | closed_seller_won
```

**`POST /api/orders/:id/early-close` 守則**：

- 必須是訂單買家本人。
- 訂單狀態必須是 `picked_up`。
- 取貨時間（`picked_up_at`）距現在必須 **≥ 24 小時**。
- 否則回 `403 EARLY_CLOSE_NOT_ALLOWED`。

---

## 6. 站內訊息

| 方法 | 路徑 | 用途 | 認證 |
|------|------|------|------|
| GET | `/api/messages/threads` | 我的訊息串列表 | 已登入 |
| GET | `/api/messages/threads/:id` | 訊息內容 | 雙方任一 |
| POST | `/api/messages/threads/:id` | 發訊息 | 雙方任一 |

**遮罩規則**：所有訊息渲染時 server-side 過濾真實姓名、地址、電話 pattern；觸發即標記為待管理員審查。

---

## 7. 評價 / 檢舉

| 方法 | 路徑 | 用途 | 認證 |
|------|------|------|------|
| POST | `/api/reviews` | 完成交易後評價 | 已登入 |
| GET | `/api/users/:id/reviews` | 看某使用者收到的評價 | - |
| POST | `/api/reports` | 檢舉刊登 / 使用者 / 訊息 | 已登入 |

---

## 8. 後台（admin，需 RBAC）

| 方法 | 路徑 | tier1 | senior | legal |
|------|------|------|------|------|
| GET | `/api/admin/queue/moderation` | ✅ | ✅ | ✅ |
| POST | `/api/admin/listings/:id/take-down` | ✅ | ✅ | ✅ |
| POST | `/api/admin/users/:id/suspend` | ❌ | ✅ | ✅ |
| GET | `/api/admin/users/:id` | 暱稱/訂單 only | + 個資（觸發 PII log） | + 個資 |
| POST | `/api/admin/users/:id/pii-export` | ❌ | ❌ | ✅（需主管核准） |
| GET | `/api/admin/dashboard/metrics` | ✅ | ✅ | ✅ |
| POST | `/api/admin/disputes/:id/resolve` | ❌ | ✅ | ✅ |
| GET | `/api/admin/audit-logs` | ❌ | ✅ | ✅ |
| POST | `/api/admin/risk-categories` | ❌ | ✅ | ✅ |

**所有 admin 寫操作必須**：
1. 寫入 `audit_logs`（actor_admin_id, action, entity, ip）
2. 若涉及個資讀取，同步寫入 `pii_access_logs`，欄位包含原因（reason）與當事人通知（除非法律明禁）

---

## 9. 通知

| 方法 | 路徑 | 用途 | 認證 |
|------|------|------|------|
| POST | `/api/notifications/web-push/subscribe` | 訂閱 Web Push | 已登入 |
| GET | `/api/notifications` | 通知中心 | 已登入 |
| POST | `/api/notifications/:id/read` | 標記已讀 | 已登入 |

---

## 10. 系統 / 健康檢查

| 方法 | 路徑 | 用途 | 認證 |
|------|------|------|------|
| GET | `/api/health` | liveness | - |
| GET | `/api/health/ready` | readiness（DB / R2 / ECPay sandbox ping） | - |

---

## 錯誤碼總覽（節錄）

| code | HTTP | 說明 |
|------|------|------|
| `UNAUTHORIZED` | 401 | 未登入 |
| `FORBIDDEN_ROLE` | 403 | RBAC 角色不足 |
| `EARLY_CLOSE_NOT_ALLOWED` | 403 | 取貨後未滿 24h |
| `LISTING_HIGH_RISK_FIELDS_REQUIRED` | 400 | 高風險類別缺少版權頁 / 內頁照 |
| `IMAGE_TOO_LARGE` | 413 | 上傳超過 10MB |
| `ECPAY_CALLBACK_INVALID_SIG` | 400 | callback 驗章失敗 |
| `PII_EXPORT_REQUIRES_APPROVAL` | 403 | legal 未取得主管核准 |

---

## Webhook 與外部相依

| 來源 | 路徑 | 驗證機制 |
|------|------|----------|
| ECPay 金流 | `/api/payments/ecpay/callback` | CheckMacValue HMAC |
| ECPay 物流 | `/api/logistics/ecpay/callback` | CheckMacValue HMAC |
| ECPay 發票 | `/api/payments/ecpay/invoice/callback` | CheckMacValue HMAC |
| Inngest events | `/api/inngest` | Inngest signing key |

所有 webhook callback **必須**：
1. 驗章成功才落地。
2. 原始 payload 寫入 `audit_logs`（保留 ≥ 1 年）。
3. idempotent（同一 trade_no 重複進來不會重算）。
