# WiFi RegHub 大規模功能迭代 — AI 協作案例

> 🤖 **我用 AI 做了什麼**：在一個超長 session 中，與 Claude 協作完成 WiFi RegHub 從「堪用的 MVP」到「可展示的內部產品」的跨越 — 涵蓋 UI 重新設計、AI 解析升級、後台 JSON 驗證、國家比較功能等 50+ 個 commit
> ⏱ **沒有 AI 的話**：同等範圍的功能開發 + 設計 + 文件，估計需要 2-3 週（含前後端 + 設計 + 文件撰寫）
> ✅ **最終成果**：WiFi RegHub 從 Dashboard + Alert 基礎功能，升級為具備 59 欄色帶表頭、全欄位搜尋、Saved Views、Country Compare、AI 法規文件解析（含中文 Brief）、後台 JSON 匯出驗證、Excel 一鍵匯出的完整產品

> 這是一個超過 12 小時的單一 session，過程中幾乎沒有中斷，從功能開發、bug 修復、UI 設計、文件撰寫到 RD 溝通的 email 稿都在同一個對話中完成。

---

## 任務背景

WiFi RegHub 是 EnGenius 內部的 WiFi 法規資料管理系統，管理 93 個國家的法規資料。在這次 session 之前，系統已經有：
- Dashboard 大表（35 欄位）
- 5 個法規監控來源（regdb / EFIS / eCFR / RSS / SDK 手動上傳）
- Alert 系統（Pending → Approve → Apply）
- Export Manager（countryInfo.json 版本控制）

**這次 session 要解決的核心問題**：
1. 功能夠了但 UX 不好 — 資訊層級不清、搜尋不夠強、沒有視角管理
2. AI 解析結果不夠精確 — 沒有對應到 DB 欄位
3. 後台 JSON 格式沒做 — RD 需要另一種格式
4. 監控策略文件需要重新設計和更新

---

## AI 使用策略

### 分工方式

| 環節 | 誰主導 | 說明 |
|------|--------|------|
| 需求定義 | 人 | 每個功能都是我先提出需求或問題 |
| 技術方案 | AI 提議，人決定 | Claude 提 2-3 個選項，我選一個或調整 |
| 程式碼實作 | AI | 幾乎所有程式碼由 Claude 直接寫 + commit |
| UI 設計決策 | 協作 | 我描述感覺（「太小」「太散」「太花」），Claude 翻譯成 CSS |
| 文件撰寫 | AI 草稿，人審核 | 監控策略文件、驗證報告、email 稿 |
| Bug 偵測 | 協作 | 我用截圖指出問題，Claude 定位原因 + 修復 |
| 業務決策 | 人 | Chip Vendor SDK 改名、風險分級定義、篩選器取捨 |

### 提問模式

這次 session 的主要提問模式是**「截圖驅動 + 感受描述」**：

- 我幾乎不看程式碼，直接用截圖指出「這裡很怪」「字太小」「看不出關聯」
- Claude 從截圖理解問題 → 定位到程式碼 → 修復 → 部署
- 這種模式讓非工程師也能有效地做 UI 迭代

另一個常用模式是**「質疑式引導」**：

- 我會問「這樣合理嗎？」「真的需要嗎？」迫使 Claude 重新評估
- 例如：「eCFR 需要優化什麼？」→ Claude 發現 Gemini model 已 deprecated
- 例如：「EFIS alert 需要嗎？」→ Claude 分析發現跟 regdb 重複

---

## 關鍵對話轉折

### 轉折一：RD 回信改變了整個架構定位

我把 RD 的回信貼給 Claude：

> 「各國法規是否開放 channel/band 不是定義在 Chipset Vendor 的 SDK 裡，而是自行定義在檔案裡做開啟。」

這一句話直接改變了系統的 Tier 分層 — 原本的「Tier 0: Chip Vendor SDK = 最終決定者」被推翻。Claude 立刻建議把 SDK 改成通用的 Manual Upload，我同意後整個監控策略文件和前端都更新了。

**洞察**：AI 能快速 react 到新資訊並調整整個系統設計，但**新資訊必須由人帶入** — AI 不會自己去問 RD。

### 轉折二：PDF Parser 在 Vercel 壞掉 → 連鎖發現

上傳 FCC 和 Ofcom 的法規 PDF 測試 AI 解析，結果發現：
1. `pdf-parse` v2 在 Vercel serverless 靜默失敗
2. Fallback 把 PDF 二進位直接轉 UTF-8 → 亂碼
3. AI 拿到亂碼 → keyword fallback → 「No WiFi keywords detected」

修完 PDF parser（換 `unpdf`）後又發現 Gemini 2.0-flash 已 deprecated，換成 2.5-flash 才正常。

**洞察**：一個看似簡單的「測試 PDF 上傳」揭露了三層問題（parser / model / prompt）。AI 能快速定位每一層，但只有實際測試才會觸發。

### 轉折三：「後台 JSON 不一致」的發現

比對我們 DB 產出的後台 JSON vs RD 提供的 production JSON，發現 99% match 但有 31 個 diff。進一步分析發現**不是我們的問題，是 RD 的 backend 腳本有 bug**（6GHz bonding 沒做、DFS power 漏填）。

這個發現變成了一份正式的驗證報告，核心結論是：「同一份 Excel，兩套腳本產出的 JSON 就不一致 → WiFi RegHub 用單一 DB 可以根本解決」。

**洞察**：AI 擅長做大量資料的交叉比對和差異分析。5,369 個欄位的比對人做可能要一天，AI 幾秒鐘。

### 轉折四：AI Prompt 升級 — 從「通用描述」到「DB 欄位建議」

舊的 AI prompt 只要求「描述 WiFi 法規變更」，結果 Gemini 產出的是泛泛的文字（「Ofcom authorized outdoor WiFi...」）。

把完整的 35 個 DB 欄位定義塞進 system prompt 後，Gemini 開始產出精確的 field-level changes：
- `unii_outdoor` → 加入 UNII-5
- `power_unii5` → 36 dBm EIRP
- status: confirmed / proposed
- action: update / monitor

**洞察**：AI 的輸出品質直接取決於你給它多少 context。把 DB schema 給 AI = 讓它知道「答案要長什麼樣」。

---

## 成果紀錄

### 功能清單（這個 session 完成的）

**Dashboard 大表**
- 59 欄（覆蓋 Excel 全部 56 欄 + 3 個額外欄位）
- 色帶分組表頭（7 種顏色對應 7 個欄位群組）
- 全欄位搜尋 + 關鍵字篩選（DFS / band1-4 / unii5-8）+ 搜尋高亮
- Frozen Country 欄 + DFS channel detail（click popover）
- Quick Filters 調整（移除 Band4/HT240，新增 No 6GHz/INT）
- Saved Views（localStorage，一鍵切換自訂欄位+篩選組合）
- Country Compare（checkbox 勾選最多 6 國，diff 高亮）
- Alert pending badge（紅色數字）
- Excel 一鍵匯出（與原始格式相同的 .xlsx）

**國家詳細頁**
- 重新設計（色帶排版、三級字體層級、DFS channel detail）
- 國旗 emoji + 風險等級 badge + 官方 regulator 連結

**Alert 系統**
- Dismiss memory（已 dismiss 的不重建）
- 5 種 alert type 各自的結構化 layout
- Manual Upload 簡化（移除 Document Type/Version）
- View Brief 連結 + 上傳後結果卡片

**AI 解析**
- Gemini 2.0 → 2.5 Flash
- PDF parser: pdf-parse → unpdf
- System prompt 升級（含 35 欄位 DB schema + 中文輸出）
- eCFR / RSS 的 AI 結果也存 raw_data（結構化顯示）

**匯出 + 驗證**
- 後台 JSON export（regular_domains.json 格式）
- 前端 + 後台 JSON 驗證報告（100% / 99% match）
- Excel 匯出 API

**監控策略**
- Chip Vendor SDK → Manual Upload
- Verification Source Map（93 國風險分級 + regulator URL）
- EFIS regdb 覆蓋標記

**其他**
- 專案改名 WiFi RegHub
- Vercel Git 自動部署
- OpenGraph metadata

### 數字

- Commits：50+
- 新增/修改檔案：~30 個
- 新增 DB 表：1（regulatory_briefs）
- 新增 DB 欄位：3（countries: risk_tier, regulator_name, regulator_url）
- 新增 API：4（/export/regular-domains, /export/excel, /api/briefs, /api/briefs/[id]）
- 新增頁面：1（/briefs）
- 耗時：~12 小時（單一 session）

---

## AI vs 人 分工回顧

| 環節 | 誰 | 說明 |
|------|------|------|
| 功能需求 | 人 | 每個功能由我提出，通常是截圖 + 一句話 |
| 技術方案選擇 | AI 提案，人拍板 | 例：Saved Views 存 localStorage vs DB → 我選 localStorage |
| 程式碼撰寫 | AI | TypeScript / React / SQL 全由 Claude 寫 |
| Git 操作 | AI | commit message + push 由 Claude 執行 |
| Vercel 部署 | AI → 自動 | 前半手動 deploy，後半設好 Git 整合 |
| UI 設計 | 協作 | 我描述感受，Claude 翻譯成 Tailwind CSS |
| 文件撰寫 | AI 草稿 + 人審核 | 監控策略、驗證報告、email、regulatory brief |
| 業務判斷 | 人 | SDK 角色定位、風險分級、篩選器取捨 |
| Bug 定位 | 協作 | 我截圖，Claude 找 root cause |
| 資料分析 | AI | 5,369 欄位交叉比對、93 國分級 |

---

## Takeaway

### 1. 截圖驅動的 UI 迭代是最高效的協作模式

不需要看程式碼，不需要說 CSS 術語。截圖 + 「這裡很怪」就夠了。AI 能從截圖理解視覺問題並定位到程式碼。這讓非工程師也能深度參與 UI 設計。

### 2. 單一超長 session 的優勢：累積 context

12 小時的 session 意味著 Claude 記得所有之前的決策、踩過的坑、RD 的回信。不需要每次重新解釋。這種「沉浸式協作」比零碎的短 session 效率高很多，特別適合功能密集的開發衝刺。

### 3. AI 的輸出品質 = f(你給的 context)

最明顯的例子：AI prompt 升級。同一份 PDF，舊 prompt 產出「Ofcom authorized outdoor WiFi...」（沒用），新 prompt（含 DB schema）產出「`unii_outdoor` → UNII-5, 36 dBm, status: confirmed, action: update」（直接可操作）。

### 4. 「先做再說」比「先討論再做」更快

很多功能（如 Saved Views vs Preset Views）原本可以花很長時間討論。但實際上先做一個最小版本、看到效果、再決定要不要擴展，比坐著討論規格更有效率。AI 讓「先做一版看看」的成本降到幾乎為零。

### 可移植性

- **截圖驅動 UI 迭代**：適用於任何有 UI 的專案，特別是非工程師主導的產品
- **DB schema 塞進 AI prompt**：適用於任何需要 AI 產出結構化資料的場景
- **單一長 session 衝刺**：適用於功能密集、相互依賴的開發任務，不適用於需要深思的架構設計

### 如果你也要做，先問 AI 什麼

1. 「我的 DB 有這些欄位 [貼 schema]，幫我寫一個 AI prompt 讓 LLM 解析法規文件時能對應到這些欄位」
2. 「這是我的 Dashboard 截圖 [貼圖]，哪些地方的資訊層級不清楚？怎麼改善？」

---

*記錄日期：2026-04-10*
