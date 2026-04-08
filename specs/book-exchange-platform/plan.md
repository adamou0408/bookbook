# 技術計畫：書籍買賣與交流平台（book-exchange-platform）

> **欄位標記說明**：標題前有 `*` 為**必填**段落（即使結論是「無」也要明確寫出）。

## * 狀態：`approved`（2026-04-08 由 Spec 擁有者核准）

## * 來源規格

- Spec：[./spec.md](./spec.md)（v1.1，approved）
- 調研：[./research.md](./research.md)
- 衝突決議：[CONFLICT-001](../../conflicts/CONFLICT-001.md) ~ [CONFLICT-006](../../conflicts/CONFLICT-006.md)（6/6 resolved）
- CONFLICT-006 法律驗證：✅ 已通過（2026-04-08）

## * 計畫總覽

| 項目 | 內容 |
|------|------|
| 工作量 | **XL**（greenfield 全棧 + 金流 + 物流 + 後台 + 法遵） |
| 任務數 | **62 個任務**（分 11 個 Phase） |
| 預估循環 | 多輪 `/req-implement`，建議 Phase 0 ~ Phase 11 依序執行 |
| MVP 範圍 | 全台同步上線，僅超商店到店，純買賣（服務費開票） |
| 技術負責人 | 待技術選型確認後由 Spec 擁有者指派 |

## * 技術選型

### 核心 Stack（一句話理由）

| 層級 | 選擇 | 理由（與 spec 哪條需求對應） |
|------|------|------|
| 前端框架 | **Next.js 14 (App Router) + React 18 + TypeScript** | Mobile-first RWD + 未登入即可瀏覽（SSR/ISR）滿足角色 C「不需登入」與 P95 ≤ 1s 搜尋目標 |
| 樣式 | **Tailwind CSS + shadcn/ui** | 快速 RWD、繁中介面在地化、shadcn 元件含 a11y |
| 後端 | **Next.js Route Handlers + tRPC**（同 monorepo） | 單一部署、型別安全、降低 greenfield infra 複雜度（research 建議 4） |
| 資料庫 | **PostgreSQL 16**（Supabase 託管） | pg_trgm + tsvector 全文檢索、earthdistance 地理排序、RLS 預備 |
| ORM | **Drizzle ORM** | 型別嚴格、支援 edge runtime、migration 可審查 |
| 認證 | **Auth.js v5（NextAuth）** + Line + Google Provider | Line 在台灣家庭族群滲透率高（research 建議 3） |
| 物件儲存 | **Cloudflare R2 + Cloudflare Images** | 低出口費用、內建壓縮（單張 ≤ 300KB 需求） |
| 金流 | **綠界 ECPay**（信用卡 + ATM + 超商代收） | 支援台灣電子發票串接（CONFLICT-006 服務費開票模型） |
| 物流 | **綠界物流 API（超商店到店）** | 與金流同一個服務商，降低串接面 |
| 書籍資料 | **Google Books API** + **國家圖書館 ISBN** fallback | ISBN 自動補全（角色 B 驗收條件） |
| 推播 | **Web Push API（VAPID）** + Line Notify（選用） | PWA 不依賴 native app；Line Notify 為冷啟動加分 |
| 背景排程 | **Inngest**（管理 escrow 自動撥款、通知、過期處理） | 比 Vercel Cron 更適合 stateful 工作流，可以重試與觀測 |
| 觀測性 | **Sentry**（錯誤）+ **PostHog**（產品分析 / 漏斗）+ **Vercel Analytics**（Web Vitals） | 對應成功指標目標 4：UX 達標率量測 |
| 部署 | **Vercel**（前後端）+ **Supabase**（DB / Storage 備援）+ **Cloudflare**（DNS / R2 / WAF） | 單一 PaaS 降 infra 成本（research 建議 4） |
| CI/CD | **GitHub Actions** + Vercel Preview Deploys | PR 自動部署、跑 lint/test/migration check |

### 不採用的選項與理由

- **Native App（iOS/Android）**：MVP 不做。Mobile-first PWA 已可滿足主要場景，避免雙平台開發成本。
- **微服務架構**：MVP 採 modular monolith。Greenfield 階段優先收斂運維面積。
- **自架金流**：不考慮。台灣金流監管嚴格，必須走持牌業者。
- **Firebase**：放棄。Firestore 不利複雜搜尋與地理查詢；台灣金流整合社群資源較弱。

## * 架構設計

### 高階拓樸

```
┌────────────────────────────────────────────────────────┐
│  Client（Mobile-first PWA, Next.js App Router, RSC）     │
│  - 未登入瀏覽 / 搜尋（公開頁面，SSR + ISR）                 │
│  - 登入後互動（刊登、購買、訂單、後台）                       │
└────────────────────────────────────────────────────────┘
              │ tRPC over HTTPS / HSTS
              ▼
┌────────────────────────────────────────────────────────┐
│  Edge / Node Runtime（Vercel Functions）                 │
│  ┌─────────────┐ ┌─────────────┐ ┌──────────────────┐  │
│  │ catalog svc │ │ listing svc │ │ search svc       │  │
│  ├─────────────┤ ├─────────────┤ ├──────────────────┤  │
│  │ order svc   │ │ payment svc │ │ escrow svc       │  │
│  ├─────────────┤ ├─────────────┤ ├──────────────────┤  │
│  │ dispute svc │ │ message svc │ │ admin svc (RBAC) │  │
│  ├─────────────┤ ├─────────────┤ ├──────────────────┤  │
│  │ audit svc   │ │ pii-log svc │ │ invoice svc      │  │
│  └─────────────┘ └─────────────┘ └──────────────────┘  │
└────────────────────────────────────────────────────────┘
       │           │           │           │
       ▼           ▼           ▼           ▼
   PostgreSQL   R2 / Images  ECPay / 物流   Inngest
   (Supabase)   (Cloudflare) (綠界)        (背景工作)
```

### Domain 模組（每個對應一個 service 資料夾）

1. **identity**：使用者帳號、第三方登入、KYC
2. **catalog**：類別 taxonomy（含風險分級）、ISBN 對應
3. **listing**：刊登流程（含分級表單、書況徽章）
4. **media**：圖片上傳、壓縮、CDN 配發
5. **search**：全文檢索、多排序、追蹤清單
6. **order**：購物、訂單狀態機
7. **payment**：金流串接、發票（服務費）
8. **escrow**：託管資金狀態機、7 天 + 提早結案
9. **logistics**：超商店到店物流
10. **dispute**：爭議與退款
11. **review**：評價與信用分數預留欄位
12. **message**：站內訊息（雙方匿名）
13. **moderation**：檢舉、下架、複審佇列
14. **admin**：後台 + 分層 RBAC + 指標儀表板
15. **audit**：審計日誌（不可竄改）
16. **pii**：個資存取記錄（使用者可查）
17. **notification**：站內訊息、推播、Email

### 關鍵狀態機

**訂單狀態機**：

```
created → paid → seller_shipping → in_transit → arrived_at_store
        → picked_up → (early_close OR 7d_window) → completed
                   → disputed → (refunded OR closed_seller_won)
```

**Escrow 狀態機**：

```
held → (auto_release_after_7d OR early_release_by_buyer)
     → released_to_seller (扣除服務費)
     → 同時觸發 invoice svc 開立服務費電子發票
held → disputed → refunded_to_buyer
```

## * 資料模型變更

> 本 spec 為 host repo 的**第一個** schema，全為新增。所有變更皆 forward-only migration（drizzle-kit）。

### 核心資料表（簡述，欄位細節留待 implement 階段）

| 表 | 用途 | 重點欄位 | 索引 |
|---|---|---|---|
| `users` | 一般使用者 | id, line_id, google_id, display_name, kyc_state, suspended_at | line_id, google_id |
| `user_pii` | 個資（加密欄位） | user_id, real_name_enc, phone_enc, default_cvs_store | user_id |
| `admin_users` | 後台管理員 | id, email, role(`tier1`/`senior`/`legal`), totp_secret | email |
| `categories` | 書籍類別 + 風險分級 | id, name, parent_id, **risk_level**(`low`/`high`) | parent_id |
| `books` | ISBN 主檔 | isbn13, title, author, publisher, edition, cover_url | isbn13 |
| `listings` | 刊登 | id, seller_id, book_id, condition_tier, price, status, **detail_badge_eligible** | seller_id, status |
| `listing_photos` | 書況照片 | id, listing_id, kind(`cover`/`condition`/`copyright_page`/`inner_page`), r2_key | listing_id |
| `orders` | 訂單 | id, buyer_id, listing_id, total_amount, service_fee, status, paid_at, picked_up_at, completed_at | buyer_id, listing_id |
| `escrows` | 託管資金 | id, order_id, amount, status, hold_until, early_released_at | order_id |
| `payments` | 金流 | id, order_id, ecpay_trade_no, method, amount, status | order_id |
| `payouts` | 撥款給賣家 | id, escrow_id, seller_id, amount_after_fee, ecpay_payout_ref | seller_id |
| `invoices` | 電子發票（**僅服務費**） | id, order_id, ecpay_invoice_no, amount(=service_fee), issued_at | order_id |
| `shipments` | 物流 | id, order_id, cvs_send_store, cvs_recv_store, ecpay_logistics_id, status | order_id |
| `disputes` | 爭議 | id, order_id, raised_by, reason, evidence_urls, resolution, decided_by | order_id |
| `messages` | 站內訊息 | id, thread_id, sender_id, body, attachment_url, created_at | thread_id |
| `reviews` | 評價 | id, order_id, reviewer_id, reviewee_id, stars, comment | reviewee_id |
| `watchlists` | 追蹤清單 | id, user_id, kind(`title`/`series`/`isbn`), keyword | user_id |
| `notifications` | 通知 | id, user_id, kind, payload, read_at | user_id |
| `reports` | 檢舉 | id, reporter_id, target_type, target_id, reason, status | status |
| `moderation_queue` | 高風險複審佇列 | id, listing_id, priority, assignee_id, decision | priority |
| `audit_logs` | 審計（**append-only**） | id, actor_type, actor_id, action, entity, payload_jsonb, ip, created_at | actor_id, entity |
| `pii_access_logs` | 使用者可查的個資存取記錄 | id, target_user_id, accessor_admin_id, accessor_role, reason, accessed_at, third_party_disclosed_to | target_user_id |
| `risk_categories` | 高風險類別清單（CONFLICT-001） | category_id, added_by, reason | - |
| `seller_credit` | 賣家信用分數（CONFLICT-005 預留欄位，MVP 不啟用） | seller_id, score, completed_orders, dispute_count | seller_id |

### 不可逆性評估

- 全部為**新增**資料表，**沒有不可逆變更**（因為沒有既有資料需遷移）。
- ⚠️ `audit_logs` 與 `pii_access_logs` 為 **append-only**，schema 上需禁止 UPDATE/DELETE（透過 RLS + DB role 限制），需在 implement 階段以 PG policy 落實。

## * 部署影響評估

> CONSTITUTION 必填項；本 spec 為 greenfield，多數項目為「首次建立」。

### 1. 健康檢查（Health Check）
- **新增**：`/api/health`（liveness）+ `/api/health/ready`（readiness：DB ping、R2 ping、ECPay sandbox ping）
- 部署前 Vercel Preview 會跑這兩支端點作為 smoke test。
- 影響：**首次建立**，無回退影響。

### 2. Schema 變更（向後相容性）
- 全部為新增 table，**向後相容**（沒有舊版需相容）。
- Migration 工具：drizzle-kit generate + drizzle-kit migrate；CI 強制 review。
- 不可逆變更：無（全為 forward additive）。
- ⚠️ `audit_logs` / `pii_access_logs` 的 append-only policy 一旦上線不可降級為可寫；屬於**設計層面**不可逆，但對部署無 rollback 風險。

### 3. 設定 / 環境變數
- **新增**以下 env vars（依環境區分 dev/staging/prod）：
  - `DATABASE_URL`（Supabase）
  - `AUTH_SECRET`、`AUTH_LINE_*`、`AUTH_GOOGLE_*`
  - `R2_*`（accountId, accessKey, bucket）
  - `CF_IMAGES_*`
  - `ECPAY_MERCHANT_ID`、`ECPAY_HASH_KEY`、`ECPAY_HASH_IV`、`ECPAY_INVOICE_*`、`ECPAY_LOGISTICS_*`
  - `INNGEST_*`、`SENTRY_DSN`、`POSTHOG_KEY`
- 部署前 checklist 需確認所有 env 已設定，缺漏會使 readiness 失敗。

### 4. 資源需求
- DB：Supabase Pro（≥ 4GB RAM、PITR 啟用）
- Vercel：Pro tier（含 Edge Config + Analytics）
- R2：估計第一年 < 100GB
- ECPay：需開立**正式商家帳號**並完成 KYC（不是 sandbox）→ 屬於**外部依賴前置作業**，需提前 2 週啟動
- 觀測性：Sentry team + PostHog growth

### 5. 持久性 / 資料安全
- DB：Supabase PITR（最近 7 天可回復至任意時點）
- R2：開啟 versioning + 跨區複寫
- 加密：高度機密欄位採用 PG `pgcrypto` symmetric encryption；金鑰存於 Vercel env（未來改為 HashiCorp Vault / KMS）
- 備份演練：上線前需做一次 restore drill

### 6. 觀測性
- Logs：Vercel Functions logs → Logflare → BigQuery（≥ 1 年保存以符合審計要求）
- Metrics：PostHog（漏斗）、Vercel Analytics（Web Vitals）
- Tracing：Sentry performance + Vercel OpenTelemetry
- Alerting：Sentry → Slack；ECPay 對帳異常 → Email

> ✅ 部署影響六大項全部填寫完整，無留白。

## * 風險評估

| # | 風險 | 可能性 | 影響 | 緩解策略 |
|---|------|------|------|--------|
| R1 | ECPay 商家審核延遲導致 MVP 上線時程被卡 | 高 | 高 | Phase 0 立即啟動商家申請；技術上預留 sandbox 環境讓開發不被卡 |
| R2 | 雙邊市場冷啟動失敗（供給不足→打擊體驗） | 高 | 高 | Phase 11 加入「種子刊登」程序：邀請 30 個家長 KOL 預先刊登 200 本以上 |
| R3 | escrow 7 天滯留資金壓力（同時 N 筆訂單卡在 escrow） | 中 | 中 | 估算 MVP 啟動 6 個月內最多 1,000 筆同時 escrow → 預留營運準備金 |
| R4 | 個資洩漏導致 PDPA 罰則 | 低 | 極高 | DB 欄位加密 + 分層 RBAC + 用戶可查的 PII 存取記錄 + 上線前 pen-test |
| R5 | 高風險類別（盜版漫畫、絕版書）人工複審量爆量 | 中 | 中 | 複審佇列加入優先序與配額；後續可加上 ML 預篩 |
| R6 | 童書售出含個資內頁未塗銷 → 兒少資料外流 | 中 | 高 | 賣家上架流程加入「請塗銷個資內頁」提醒；買家舉報快速通道 |
| R7 | Line / Google OAuth 政策變動 | 低 | 中 | 雙 provider 並存 + 留 email / OTP 備援 |
| R8 | 第三方金流爭議 | 中 | 中 | 對帳排程每日跑、不一致即告警；保留 ECPay 原始 callback payload ≥ 1 年 |
| R9 | 服務費開票模型遭稅務局異議 | 低（已驗） | 極高 | 已由法律顧問驗證（CONFLICT-006）；保留稽核 trail |
| R10 | 圖片儲存爆量（每張上傳 5~10MB → 壓縮後 ≤ 300KB） | 中 | 中 | 上傳即經 Cloudflare Images 自動壓縮；前端拒絕 > 10MB 原圖 |

**最高風險**：R1 — ECPay 商家審核延遲（可能性高 × 影響高）。緩解：Phase 0 第一週立即啟動商家申請，與技術開發並行。

## * 成功驗收

實作完成的判定條件（與 spec.md 成功指標對齊）：

1. **功能完整性**：spec.md 36 條驗收條件全部通過 e2e 測試（playwright）。
2. **效能達標**：
   - 搜尋 P95 ≤ 1s（k6 壓測）
   - 註冊 P95 ≤ 90s（PostHog 漏斗）
   - 刊登 P95 ≤ 2min（PostHog 漏斗）
3. **安全性**：上線前完成 OWASP Top 10 自評 + 1 次第三方 pen-test。
4. **法遵**：CONFLICT-006 服務費開票模型實際開出 1 張電子發票並通過會計師複核。
5. **指標儀表板**：管理員後台可查 MAU、刊登完成率、糾紛率、UX 達標率、GMV 五項。

## * 計畫後續

- 接 `/req-implement` 開始執行 [tasks.md](./tasks.md)。
- API 細節見 [contracts.md](./contracts.md)。
- 上線前最後一次 `/req-review`（已 approved 不需重跑，但建議在 Phase 11 前加一次 retrospective）。
