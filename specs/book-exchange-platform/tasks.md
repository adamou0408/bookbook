# 任務清單：書籍買賣與交流平台（book-exchange-platform）

> **約定**：每個任務有 ID、對應 User Story、依賴關係、可平行群組、測試策略。`[P-X]` 表示同 group 可平行；`[depends: N]` 表示需先完成任務 N。

## * 來源

- 計畫：[./plan.md](./plan.md)
- Spec：[./spec.md](./spec.md)（v1.1）

---

## Phase 0：基礎建設 + 外部依賴啟動（P0 阻擋項）

| ID | 任務 | User Story | 平行群 | 依賴 | 測試策略 |
|----|------|-----------|--------|------|----------|
| T01 | 啟動 ECPay 商家申請（KYC、合約、API key 申請） | 全部 | P0-A | - | 商家審核通過收信 |
| T02 | 申請 Line Login channel（messaging API + Login 權限） | A/B/C | P0-A | - | sandbox 登入 e2e |
| T03 | 申請 Google OAuth client | A/B/C | P0-A | - | sandbox 登入 e2e |
| T04 | 建立 Supabase 專案 + 設定 PITR + 基線 schema migration runner | 全部 | P0-A | - | drizzle migrate dry-run |
| T05 | 建立 Cloudflare R2 bucket + Cloudflare Images + DNS / WAF | A/B/C | P0-A | - | 上傳測試檔成功 |
| T06 | 建立 Vercel 專案 + Git 整合 + Preview Deploy + env 結構 | 全部 | P0-A | - | first commit deploy 成功 |
| T07 | Sentry / PostHog / Inngest 建立帳號與 SDK 金鑰 | D | P0-A | - | 測試事件能進入 dashboard |
| T08 | 建立 Next.js 14 App Router 專案骨架 + TypeScript 嚴格模式 + Tailwind + shadcn/ui | 全部 | - | T06 | `pnpm build` 通過 |
| T09 | 設定 Drizzle ORM + 基礎 migration（users 空表）+ CI migration check | 全部 | - | T04, T08 | drizzle generate + apply |
| T10 | GitHub Actions：lint / typecheck / unit test / migration check / deploy preview | 全部 | - | T08 | PR pipeline 全綠 |

---

## Phase 1：身分與帳號（identity）

| ID | 任務 | User Story | 平行群 | 依賴 | 測試策略 |
|----|------|-----------|--------|------|----------|
| T11 | 設計 `users`、`user_pii` schema（含加密欄位 pgcrypto） | A/B/C/D | - | T09 | drizzle migrate + 單元測試 |
| T12 | 整合 Auth.js v5（Line + Google provider） | A/B/C | - | T02, T03, T11 | e2e：完成第三方登入 |
| T13 | 註冊流程頁面（含個資填寫、預設取書超商選擇） | A/B/C | - | T12 | e2e：註冊 → ≤ 90s 達成 |
| T14 | 個人頁基本資訊（我的暱稱、預設取書超商、第三方綁定狀態） | A/B/C | - | T13 | e2e + a11y |
| T15 | 「下載個資」「刪除帳號」自助介面（PDPA 合規） | A/B/C | P1-A | T13 | e2e：請求 → 24h 內收檔 |
| T16 | 未成年人（< 18 歲）家長同意機制 | A/B/C | P1-A | T13 | 單元測試 + e2e 引導頁 |

---

## Phase 2：書籍 catalog + 媒體上傳（catalog / media）

| ID | 任務 | User Story | 平行群 | 依賴 | 測試策略 |
|----|------|-----------|--------|------|----------|
| T17 | 設計 `categories`、`books`、`risk_categories` schema（含 risk_level） | A/B/D | - | T09 | drizzle migrate |
| T18 | 串接 Google Books API + 國家圖書館 ISBN fallback service | A/B | - | T17 | 單元測試 + 真實 ISBN 抽樣 |
| T19 | 圖片上傳服務：前端壓縮 + R2 直傳 + Cloudflare Images variant | A/B | - | T05 | 整合測試：> 10MB 拒絕、≤ 300KB 達成 |
| T20 | 類別 taxonomy 管理（後台）+ 預設高風險清單種子資料（童書、教材、漫畫、絕版書） | D | - | T17 | 管理員後台 e2e |

---

## Phase 3：刊登（listing）

| ID | 任務 | User Story | 平行群 | 依賴 | 測試策略 |
|----|------|-----------|--------|------|----------|
| T21 | 設計 `listings`、`listing_photos` schema（含 detail_badge_eligible） | B | - | T17 | drizzle migrate |
| T22 | ISBN 條碼掃描頁（手機相機）+ 自動補全 | B | - | T18 | e2e：掃碼 → 自動填表 |
| T23 | 極簡刊登表單（低風險路徑：1 張封面照 + 三段書況勾選 + 售價） | B | - | T19, T21 | e2e：≤ 2min 達成 |
| T24 | 高風險刊登單一補件步驟（CONFLICT-001 + 003：版權頁 + 版次 + 童書內頁照） | B | - | T23 | e2e：高風險 ≤ 3min 達成 |
| T25 | 「書況詳實」徽章邏輯（≥ 3 張書況照→ detail_badge_eligible=true） | A/B | - | T23 | 單元測試 |
| T26 | 定價輔助：建議售價（新書定價 × 折數區間）+ 雙顯示（買家總成本 / 賣家實拿） | B | - | T23 | 單元測試 |
| T27 | 我的刊登管理（編輯、下架、補件） | B | - | T23 | e2e |

---

## Phase 4：搜尋與瀏覽（search）

| ID | 任務 | User Story | 平行群 | 依賴 | 測試策略 |
|----|------|-----------|--------|------|----------|
| T28 | 全文檢索 schema（pg_trgm + tsvector + index） | A/C | - | T21 | EXPLAIN ANALYZE：P95 < 1s |
| T29 | 公開瀏覽頁（**未登入**可進入，SSR / ISR） | C | - | T28 | e2e：未登入完成搜尋 |
| T30 | 三種排序：總成本最低 / 書況最佳 / 距離最近（地理排序使用 earthdistance） | A/C | - | T28 | 單元測試 + e2e |
| T31 | 搜尋條件：書名 / 作者 / 系列名 / ISBN | A/C | P4-A | T28 | 單元測試 |
| T32 | 追蹤清單與符合條件的新刊登 push 通知 | A | P4-A | T28, T13 | 整合測試 |
| T33 | k6 壓力測試確認搜尋 P95 ≤ 1s | C | - | T28 | 跑壓測 + 報告 |

---

## Phase 5：訂單 + 金流 + 物流 + escrow

| ID | 任務 | User Story | 平行群 | 依賴 | 測試策略 |
|----|------|-----------|--------|------|----------|
| T34 | 設計 `orders`、`payments`、`escrows`、`shipments`、`payouts` schema | A/B/C | - | T11, T21 | drizzle migrate + 狀態機單測 |
| T35 | 訂單狀態機實作（XState 或 enum + service） | A/B/C | - | T34 | 單元測試覆蓋全部轉移 |
| T36 | 整合 ECPay 金流（信用卡 / ATM / 超商代收 / callback 驗章） | A/C | - | T01, T34 | sandbox e2e |
| T37 | 整合 ECPay 超商物流（B 寄件→店到店→C 取書） | A/B/C | - | T01, T34 | sandbox e2e |
| T38 | escrow 狀態機（hold → 7d_release / early_release / refund） + Inngest 排程 | B/C | - | T35, T07 | 時間旅行測試 |
| T39 | **提早結案按鈕**（CONFLICT-005）：取貨後 24h 才顯示，點選即觸發 escrow release | C | - | T38 | e2e：時間冷藏 24h 後可見 |
| T40 | 服務費計算 + 雙顯示同步（與 T26 對齊） | B/C | - | T26, T34 | 單元測試 |
| T41 | 撥款給賣家（payouts）+ 對帳排程（每日跑） | B | - | T38 | 對帳腳本 + 異常告警 |
| T42 | **服務費電子發票**（CONFLICT-006）：ECPay 發票 API，僅就 service_fee 開票 | C/D | - | T36, T40 | sandbox 開立 1 張並複核 |

---

## Phase 6：互動（messages / reviews / watchlist）

| ID | 任務 | User Story | 平行群 | 依賴 | 測試策略 |
|----|------|-----------|--------|------|----------|
| T43 | 站內訊息（thread + message + 雙方匿名遮罩） | A/B/C | P6-A | T13, T34 | e2e：全程不洩漏個資 |
| T44 | 評價系統（取貨後可評，1~5 星 + 文字） | A/B/C | P6-A | T34 | 單元測試 + e2e |
| T45 | seller_credit 預留欄位（CONFLICT-005，MVP 不啟用，但 schema 與寫入要存在） | B | P6-A | T34 | drizzle migrate + 寫入測試 |

---

## Phase 7：爭議 + 檢舉 + moderation

| ID | 任務 | User Story | 平行群 | 依賴 | 測試策略 |
|----|------|-----------|--------|------|----------|
| T46 | 設計 `disputes`、`reports`、`moderation_queue` schema | A/C/D | - | T34 | drizzle migrate |
| T47 | 買家鑑賞期內發起爭議流程 + 證據上傳 | C | - | T46 | e2e |
| T48 | 高風險刊登自動進入複審佇列（CONFLICT-001 分級審核） | B/D | - | T24, T46 | 單元測試 |
| T49 | 後台複審工作流：核准 / 下架 / 要求補件 | D | - | T48 | e2e |

---

## Phase 8：後台管理 + 分層 RBAC + 個資存取記錄

| ID | 任務 | User Story | 平行群 | 依賴 | 測試策略 |
|----|------|-----------|--------|------|----------|
| T50 | 設計 `admin_users` + RBAC（tier1 / senior / legal）+ TOTP | D | - | T11 | 單元測試 + e2e |
| T51 | 後台基礎 layout + 角色路由 guard | D | - | T50 | e2e：角色切換驗證 403 |
| T52 | tier1 客服視角（暱稱 / 訂單 / 訊息，PII 全遮蔽） | D | - | T51 | e2e |
| T53 | senior 客服視角：點開個資觸發 audit_log + pii_access_log + 通知當事人 | D | - | T51, T54 | e2e + 單測 |
| T54 | 設計 `audit_logs`（append-only）+ `pii_access_logs` schema + RLS policy | D | - | T11 | RLS 阻擋 UPDATE/DELETE 測試 |
| T55 | legal 角色：個資匯出 + 主管核准 + 第三方揭露記錄 | D | - | T53 | e2e |
| T56 | 使用者個人頁的「我的個資存取紀錄」（自助查閱 pii_access_logs） | C | - | T54 | e2e |
| T57 | 後台指標儀表板（MAU、刊登完成率、糾紛率、UX 達標率、GMV） | D | - | T07, T34 | 整合 PostHog + DB query |

---

## Phase 9：通知 + PWA

| ID | 任務 | User Story | 平行群 | 依賴 | 測試策略 |
|----|------|-----------|--------|------|----------|
| T58 | Web Push（VAPID）訂閱與發送 | A/B/C | P9-A | T13 | e2e：手機收推播 |
| T59 | 通知中心頁 + 已讀狀態 | A/B/C | P9-A | T58 | e2e |
| T60 | PWA manifest + service worker + 安裝提示 | A/B/C | P9-A | T08 | Lighthouse PWA score ≥ 90 |

---

## Phase 10：上線前驗收 + 安全 + 種子計畫

| ID | 任務 | User Story | 平行群 | 依賴 | 測試策略 |
|----|------|-----------|--------|------|----------|
| T61 | 第三方滲透測試（OWASP Top 10）+ 修補 | 全部 | - | T01-T60 | 報告無 high/critical |
| T62 | DB 還原演練 + 對帳腳本驗收 + 上線前 checklist 全綠 + 種子刊登（30 KOL × 200 本） | 全部 | - | T61 | go/no-go 會議簽核 |

---

## 任務統計

- **總任務數**：62
- **預估平行群**：P0-A（外部依賴申請，6 平行）、P1-A（合規 2 平行）、P4-A（搜尋細項 2 平行）、P6-A（互動 3 平行）、P9-A（通知 PWA 3 平行）
- **關鍵路徑**：T01 → T11 → T34 → T38 → T42 → T62（金流發票鏈是最長依賴鏈）
- **每個任務都對應至少 1 個 User Story**：✅
- **測試策略已明確**：✅（單測 / 整合 / e2e / 壓測 / RLS policy 測試）
