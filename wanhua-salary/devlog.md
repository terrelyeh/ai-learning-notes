---
title: 萬華世界薪資計算模組
created: 2026-04-23
type: ai-collaboration
category: 工具開發
tags: [Next.js, Python, salary, payroll, vibe-coding, GitHub-API]
status: 已完成
---

# 萬華世界薪資計算模組 — AI 協作開發紀錄

> 🤖 **我用 AI 做了什麼**：把一份手算 Excel 的薪資流程，做成「線上協作編輯 → 本地批次計算 → 線上發布給員工查看」的完整工具
> ⏱ **沒有 AI 的話**：以我的程度（行銷背景、會 vibe coding），純自己寫從零到能上線估計要 2–3 個月，且很容易卡在認證/列印/SSG 部署這些坑
> ✅ **最終成果**：14 位員工的 3 月薪資計算誤差全在 $200 內，平均 $43，管理層 100% 吻合；店長現在直接在 web 上補打卡，不用 LINE 來回對帳

> 一份從 Excel 公式手算進化到 SaaS 級工作流的內部工具。

---

## 一、為什麼要做這件事

我不是要做一個薪資系統，我是要解決三個一直在發生的痛點：

1. **每月對帳很痛** — POS 打卡常常漏一邊（只有上班沒下班、或反過來），月底結薪時要大家用 LINE 翻紀錄回想當天幾點到幾點。
2. **規則散落 Excel 公式** — 辛苦金、業績獎金、Winnie 店長業績、勞健保自付額、家人健保扣款，每個人的算法都是試算表裡的 IFS。我自己都常常忘記為什麼某格是這個數字。
3. **線上 / 線下不一致** — 店長在 LINE 群組講「這天高知夜 POS $0，實際包場 $68,870」，但這修正只活在訊息裡，下個月另一個人算薪資時又要重新問一次。

換句話說：問題不是「我要做薪資系統」，問題是**業務知識沒有單一可信來源**。

---

## 二、技術選型

| 工具 | 選的理由 |
|---|---|
| Next.js 16 (App Router, SSG) | 已用在儀表板，不想再多開一個技術棧 |
| Python (PEP 723 script + `uv`) | 薪資計算邏輯放本地 CLI 執行；單檔搞定相依（不用 venv 也不用裝 conda），給未來的我或會計同事執行很乾淨 |
| GitHub Contents API | 不想架資料庫。corrections / config / day-markers 全部當 JSON 存進 private repo，前端透過 GitHub API 寫檔。**省下一整個後端** |
| NextAuth v5 + Google OAuth + email 白名單 | 員工資料機密，不能讓任何人 Google 一下就看到 |
| 第二層密碼（薪資頁專用） | 薪資頁要再多一道密碼，連登入到儀表板的同事也不會誤點到 |
| `modern-screenshot` | Tailwind v4 用了 `oklab/oklch` 顏色，舊的 `html2canvas` / `html-to-image` 都不支援，截薪資條會變灰白色 |
| `jszip` | 批次下載 14 位員工薪資條圖片 |

> **選型原則**：能不加就不加。後端 → GitHub API 取代；DB → JSON 檔；計算 → 本地 CLI 推進階段才執行（不放雲端 serverless）。任何加進來的東西都得回答「你不能用 X 取代嗎」。

---

## 三、外部服務與金鑰

| 服務 / 模型 | 用途 | 類型 |
|---|---|---|
| Google OAuth | 第一層登入 + email 白名單檢查 | OAuth |
| `GOOGLE_CLIENT_ID` / `GOOGLE_CLIENT_SECRET` | NextAuth provider 設定 | Secret |
| `NEXTAUTH_SECRET` | JWT 簽章 | Secret |
| `ALLOWED_EMAILS` | 白名單 email 清單（逗號分隔） | Env |
| `SALARY_PAGE_PASSWORD` | 第二層 — 薪資頁密碼 | Secret |
| `GITHUB_TOKEN` | 寫 corrections / config / markers 用（Fine-grained PAT，`Contents: Read and write`） | Token |
| `TARGET_REVENUE_2026` | 年度目標 | Env |

> 沒有付費 API、沒有 DB 服務、沒有 Resend / LINE（規劃中但還沒做）。最低營運成本就是 Vercel 的免費方案 + GitHub。

---

## 四、系統架構

把資料流寫一次最快理解：

```
┌─────────────────────────────────────────────────────────────┐
│  iChef POS Excel ─▶ excel_to_json.py ─▶ data/YYYY-MM.json   │  (月初手動轉檔)
└─────────────────────────────────────────────────────────────┘
                          │
                          ▼
┌─────────────────────────────────────────────────────────────┐
│  /salary/admin  (Next.js)                                   │
│  店長線上編輯：補打卡 / 國定假日 / 每日營收覆寫 /            │
│  Winnie 月度時數 / 特殊調整                                  │
│  ─▶ POST /api/salary/corrections                            │
│         └─▶ GitHub Contents API ─▶ corrections.json (commit)│
└─────────────────────────────────────────────────────────────┘
                          │
                          ▼
┌─────────────────────────────────────────────────────────────┐
│  本地 CLI (執行者：管理者，月底跑一次)                        │
│  $ git pull                                                 │
│  $ uv run --script salary_calculator.py --month X --publish │
│   ├─▶ salary_core.py (套規則計算)                           │
│   ├─▶ public/docs/salary/YYYY-MM.xlsx  (進 git)             │
│   └─▶ data/salary-results/YYYY-MM.json (進 git)             │
│  $ git push                                                 │
└─────────────────────────────────────────────────────────────┘
                          │  (Vercel 自動部署)
                          ▼
┌─────────────────────────────────────────────────────────────┐
│  /salary/view  (Next.js)                                    │
│  員工線上查看：薪資總覽 + 個人薪資條 (drawer + 印章)         │
│  + 列印 PDF + 存圖片 + 批次 ZIP                              │
│  + 業績獎金計算明細 drawer (會計用)                          │
└─────────────────────────────────────────────────────────────┘
```

**三層認證守備**：
1. middleware 沒有 `req.auth` → 跳轉 `/login`（Google）
2. email 不在 `ALLOWED_EMAILS` 白名單 → signIn callback 拒絕
3. 進 `/salary/*` 還要有 `wanhua_salary_auth` cookie → 否則跳 `/salary-login`（密碼）

---

## 五、推進方式

整個專案橫跨約一個月（2026-03-29 → 2026-04-23），我把它拆成幾個 phase，**每個 phase 都先做最小能跑的版本，再回來補**：

**Phase 1：規則上紙面**（2 天）
先請 AI 把試算表的公式翻譯成自然語言文件（`docs/salary-rules-draft.html`），跟團隊對。這一步的目的是**把隱性規則顯性化**，不是寫 code。對完之後才有信心開工。

**Phase 2A：CLI 計算工具**（3 天）
Python 單檔（PEP 723）。先把計算邏輯寫對，輸出 Excel + JSON。**用 2026-02 真實資料當回歸測試** — AI 生第一版時故意先不告訴它「Winnie 不領 pool 份額」，看它能不能從規則文件推斷出來。它沒推到，這就是第一個關鍵轉折：規則文件不夠精確，要重寫。

**Phase 2B：上線 + 認證**（2 天）
直接套 Google OAuth + 雙層保護。中間踩到 middleware 無限重定向，下面會說。

**Phase 3：店長線上編輯**（5 天）
補打卡 → 特殊調整 → 每日營收覆寫 → 國定假日。每個區塊都是「檢視 / 編輯」雙模式 — 預設唯讀，點編輯才出現表單。這是因為店長不是工程師，誤觸是真實問題。

**Phase 4：薪資條 + 列印 + 圖片**（3 天）
一個元件，三種輸出：螢幕 drawer / A4 列印 / PNG 圖片 / ZIP 批次。技術難點全在 print CSS 和螢幕截圖工具的相容性，下面踩坑章節會講。

**Phase 5：規則調整**（最後 2 天）
業績獎金 $1.6M 門檻、辛苦金最低工時 3h → 4h、Winnie 月度時數 UI、業績獎金計算明細 drawer（給會計）。這些不是新功能，是**把隱性規則繼續顯性化**的延伸。

> **推進心法**：先做能跑的，後做正確的，最後做漂亮的。每個階段都先讓店長能用，再回頭補我自己看到的不完美。

---

## 六、踩到的坑，讓我更懂的事

### 坑 1：Winnie 的算法不是「特例」，是「混合」

最早 AI 把規則理解成「Winnie 走獨立公式 = 月營收 × 1%，不參與一般池」。實際上正確的規則是：**Winnie 的時數會進獎金池的分母（拉低每小時金額），但她本人不領 pool 份額，改領店長業績 1%**。

這個差異讓三月計算結果一開始所有人都偏高 6–8%。我跟 AI 說「再驗一次數字」，它才從 2026-02 的對照表反推出規則。

> **帶走的原則**：規則文件寫到「獨立計算」「特殊處理」這種詞時，要追問「對其他人的計算有沒有副作用」。隱性的副作用最會跑掉。

### 坑 2：middleware `startsWith("/salary")` 會把 `/salary-login` 也擋下

寫法是：「進 `/salary/*` 沒密碼 → 跳 `/salary-login`」。但 `/salary-login` 也 `startsWith("/salary")`，於是跳了又跳，無限重定向。

```ts
// 錯：
if (pathname.startsWith("/salary") && !cookie) → /salary-login

// 對：
if ((pathname === "/salary" || pathname.startsWith("/salary/")) && !cookie) → /salary-login
```

> **帶走的原則**：路由匹配陷阱在 Next.js / Express / 任何路由系統都會出現。寫條件之前先把「最接近的反例」舉出來。

### 坑 3：Tailwind v4 + `html2canvas` 互不相容

存薪資條圖片時，`html2canvas` 和 `html-to-image` 都會把 `oklab` / `oklch` 顏色解析成灰色或全黑。Tailwind v4 預設就用這些色彩空間。

換成 `modern-screenshot` 解決（它原生支援）。但 `modern-screenshot` 在 headless 瀏覽器（preview 環境）會 timeout — 真實瀏覽器才能正常工作。

> **帶走的原則**：CSS 工具鏈進化太快，截圖類庫沒跟上是常見陷阱。先確認 PoC 在你實際瀏覽器跑過再投入功能開發。

### 坑 4：SSG + GitHub API 寫檔有部署延遲

線上版透過 GitHub Contents API 寫 corrections / markers 後，必須等 Vercel 重新部署（1–2 分鐘）資料才會生效。`router.refresh()` 救不了，因為讀的是 build 時打包的檔。

我一開始花了 30 分鐘 debug「為什麼改完看不到」，最後是 commit 紀錄發現 Vercel 還在 building。

> **帶走的原則**：用 SSG + 外部寫檔架構時，UX 上要顯示「已儲存，1–2 分鐘後生效」字樣，不要假裝是即時的。

### 坑 5：列印「全部薪資條」會跑出空白頁

最早把 `.payslip-bulk-target` 放在 drawer 的 React tree 裡，列印時 fixed/overflow 互相干擾，A4 第一頁是空白、最後一頁也是空白。

解法：把列印容器搬到**根層級**（`<body>` 直接子元素），用 `body.printing-bulk` class toggle 控制顯示，print CSS 用 `visibility: hidden` 而不是 `display: none`（後者會破壞 page-break）。

> **帶走的原則**：print 樣式跟螢幕樣式是兩套思維。fixed/overflow/transform 在 print 媒體都不照預期。

### 坑 6：薪資條展開公式時，跟實際算法不一致

薪資條業績獎金列原本顯示「**月營收** × 2% = 池」，但實際 `salary_core.py` 用的是「**獎金基準**（含每日覆寫後的數字）× 2%」。兩個值通常一樣，但**有覆寫的月份**就會差個幾萬，會計核對會抓蟲。

> **帶走的原則**：把規則展示在 UI 上時，文字一定要跟程式碼用的變數同名同義。任何「差不多就好」的描述都會在某個邊界情境炸開。

---

## 七、最終樣貌

**店長視角**（`/salary/admin/2026-04`）
- 五個區塊都是「檢視 / 編輯」雙模式：正職目標工時、國定假日、管理層時數、每日營收覆寫、補打卡、特殊調整
- 單邊打卡警告會在補打卡上方自動列出，「補打卡」一鍵帶入日期/員工/已知時間，「忽略」立即存檔
- 改完按儲存 → 線上立即看到最新狀態（透過 GitHub Contents API commit）

**員工視角**（`/salary/view/2026-04`）
- 薪資總覽表 + 下載 Excel 全月報表
- 每位員工點「📄 薪資條」打開 drawer：每日出勤明細 + 加班費展開（目標 160h，加班 2h20m × $159/h）+ 業績獎金展開（基準 × 2% → 池 → ÷ 合格時數 → 個人）+ 公司印章
- 「🖼️ 存圖片」單張 PNG / 「📦 批次下載 ZIP」14 人一次打包 / 「🖨️ 列印」單份或全部 A4

**會計視角**（從「獎金池 2%」KPI 卡點「📊 查看計算明細」）
- 門檻狀態（✅ / ❌ 含金額對比）
- 公式區：基準 × 2% ÷ 合格時數 = 每小時
- 每人分配表（姓名/類別/打卡時數/進位小時/業績獎金），Winnie 標「店長」徽章

**規則文件**（`/docs/salary-rules-draft.html`，公開但需登入）
- v1.2 含完整公式 + FAQ + 常見情境

---

## 八、如果繼續往下

- **薪資條自動寄送** — 員工 email 欄位 + Resend API 批次寄；或 LINE Messaging API 推播（員工更常用 LINE）
- **參數版本化** — 現在改 `salary-config.json` 會影響所有未計算月份。加個 `effective_from` 日期，調薪時不會回頭污染歷史
- **`/salary/trend`** — 多月薪資趨勢比較
- **薪資條每人獨立 PDF** — 用 jsPDF + JSZip 生成個別 PDF（現在只能 PNG）

---

## Takeaway

**這個案例展示了什麼**

最值得分享的不是技術細節，是**「先把規則寫成文件，再讓 AI 翻譯成程式」這個反向工序**。

絕大多數人的順序是：先動手寫 code → 邊寫邊發現規則沒講清 → 改 code → 再改規則 → 永遠不一致。

我反過來：花 2 天先讓 AI 訪談我、把試算表公式翻譯成自然語言文件、跟團隊對齊。對齊之後 code 是相對機械的翻譯動作。後來業務規則調整（門檻、4h、Winnie 算法），都是先改文件 → 改設定 → 改 code，三邊都同步。

**可移植性**

這套「先規則 → 後 code」的工序非常通用：
- ✅ 任何「業務規則密集」的工具都適用：薪資、計費、退款、出貨規則、扣抵
- ✅ 任何要跨團隊溝通的內部工具都適用：規則文件本身就是溝通介質
- ❌ 純技術問題（API 串接、效能優化）不適用 — 那種題目沒有「規則文件」可以先寫

特殊條件：我有一個小團隊（3–4 人）需要對齊。如果只有自己用，這個 overhead 不一定划算。

**如果你也要做，先問 AI 什麼**

1. **「我把現在試算表裡所有計算規則描述給你聽，請你幫我整理成一份條列文件，並標出哪些地方我沒講清楚、需要補充。」**
   → 這是把隱性知識顯性化的起手式。

2. **「我要做一個工具讓 [角色 A] 線上編輯規則參數、[角色 B] 線上查看結果，但我不想架後端。請幫我評估『把資料存在 Git 倉庫的 JSON 檔，前端透過 GitHub API 讀寫』這個架構的優缺點。」**
   → 這是「在限制中做選型」的提問模式。AI 會列出所有 trade-off，你做決定。

---

_開發期間：2026-03-29 → 2026-04-23（約 26 天，跨多個 session）_
_整理日期：2026-04-23_
