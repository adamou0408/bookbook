# Changelog

This file logs every spec state transition. Auto-updated by /req-* commands.

## 2026-04-08
- Project initialized via req-init.sh
- `book-exchange-platform` intake 建立 via /req-intake
- `book-exchange-platform` research.md 建立 via /req-research（feasibility: Yellow，無重複）
- `book-exchange-platform` spec.md v1.0 建立 via /req-translate；產出 4 個 persona（family-reader, book-provider, book-seeker, platform-admin）、4 個 User Stories、24 項驗收條件、6 個開放問題
- `book-exchange-platform` spec.md v1.0 → v1.1 via /req-iterate：解決 6 個開放問題（商業模式=純買賣、目標族群=所有台灣讀者、物流=僅超商店到店、上線範圍=全台同步、提案者=使用者本人、「好用」定義=4 項可量測指標）
- `book-exchange-platform` /req-detect-conflicts：偵測 6 個衝突（4 high / 2 med），CONFLICT-006 為 v1.1 新增發票/服務費決策浮現
- `book-exchange-platform` /req-resolve-conflict CONFLICT-001：採用方案 3（分級審核），理由「法律風險 + KPI 雙重大」
- `book-exchange-platform` /req-resolve-conflict CONFLICT-002：採用方案 3（冷啟動錨點 + 雙顯示 + 多排序），理由「保留未來空間」
- `book-exchange-platform` /req-resolve-conflict CONFLICT-003：採用方案 3（童書類別額外必填內頁照 + 徽章鼓勵），理由「與 CONFLICT-001 協同，共用高風險類別 taxonomy」
- `book-exchange-platform` /req-resolve-conflict CONFLICT-004：採用方案 3（分層 RBAC 一線／資深／法遵 + 使用者可查的個資存取紀錄），理由「使用者信任公開」
- `book-exchange-platform` /req-resolve-conflict CONFLICT-005：採用方案 3（預設 7 天 + 取貨 24h 後可提早結案 + 信用分數預留），理由「守住消保法底線」
- `book-exchange-platform` /req-resolve-conflict CONFLICT-006：採用方案 2（服務費開票模型），理由「平台身分清晰」。⚠️ 必須在 /req-plan 前由法律／會計顧問驗證符合台灣稅法；若否決需回頭重新決策
- `book-exchange-platform` 全部 6 個衝突已解決，spec 狀態 draft → 可進入 /req-review
- `book-exchange-platform` /req-review：審核通過（Approve），spec 狀態 in-review → approved；審核紀錄 reviews/REVIEW-book-exchange-platform-2026-04-08.md；審核者＝Spec 擁有者本人；⚠️ /req-plan 啟動前需法律顧問驗證 CONFLICT-006 模型
