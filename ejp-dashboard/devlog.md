# EJP Sales Pipeline Dashboard — 開發紀錄

> 🤖 **我用 AI 做了什麼**：用 Claude Code 從零建出一個部署在 Vercel、有登入保護、雙語切換、多維度圖表的業務管道儀表板 POC
> ⏱ **沒有 AI 的話**：找外包至少 2–4 週、預算數十萬日圓，且對方不懂業務邏輯還要來回溝通
> ✅ **最終成果**：儀表板上線（[ejp-pipeline-dashboard.vercel.app](https://ejp-pipeline-dashboard.vercel.app/)），搭配分公司導入提案網頁 + IT 申請文件，整套交付物一站完成

> 一個沒有工程師背景的業務人員，靠 AI 協作在幾天內完成原本要花幾週的工具開發。

---

## 一、為什麼要做這件事

EJP Japan 的業務 pipeline 資料散落在 Google Sheets 裡。每次開會都要手動整理數字、截圖貼到 PPT，沒有人能一眼看出哪個 Stage 的案件最多、哪個業務的 Pipeline 最健康。

不是不想用現成工具（Salesforce、HubSpot），而是：
- IT 採購流程太長，今天需要的明年才批
- 分公司規模小，付整套 CRM 的費用不合理
- 最重要的是：**資料就在 Google Sheets 裡，只差一個能看的介面**

所以目標很清楚：做一個 POC，證明「這件事值得繼續投資」，而不是先說服管理層、排預算、等 IT。

---

## 二、技術選型

| 工具 | 選的理由 |
|------|---------|
| **Next.js** | AI 最熟悉、部署最簡單、前後端一起搞定 |
| **Recharts** | React 生態裡圖表最好用，不用額外學 |
| **Google Sheets API** | 資料就在那裡，Service Account 5 分鐘接好 |
| **Vercel** | 直接推 GitHub 就部署，免費方案夠用，URL 給管理層看很專業 |
| **Tailwind CSS** | 沒有設計稿也能寫出看得過去的 UI |
| **Radix UI** | Dropdown、Modal 這些互動元件不用自己寫 |

> **選型原則**：在 AI 最熟悉的技術範圍內選，不是選「最好的」，而是選「AI 最能幫我的」。陌生的技術堆疊只會讓每個問題都變成 debug 地獄。

---

## 三、推進方式

整個過程不是「我想到什麼 AI 就做什麼」，而是**把現有的 Google Sheets 當成規格書，讓 AI 逼近這個已知的正確答案**。

具體做法是：先把 Sheets 的欄位名稱、Stage 值、資料格式全部告訴 AI，然後逐步要求它把每個欄位正確地對應到介面上。每加一個功能，都先確認資料有沒有正確讀進來，再確認圖表顯示對不對。

拆解任務的方式大概是這樣：
1. 先有一個能跑的骨架（API 讀 Sheets，前端顯示一個表格）
2. 再加圖表（Funnel、Bar、Pie 一個個加）
3. 再加互動（Filter、搜尋、語言切換）
4. 再加保護（登入頁、Middleware）
5. 最後加交付物（提案網頁、IT 申請文件）

每個階段都先讓它動起來，再優化細節。不試圖一次到位。

---

## 四、踩到的坑，讓我更懂的事

### 坑一：Stage 資料全部顯示 0（資料讀進來了，但 key 對不上）

**症狀**：Contact、Qualified、Evaluation、Quote 這幾個 Stage 的圖表全是 0，但 Sheets 裡明明有資料。

這個問題讓我理解了一件事：**程式碼裡有兩套 Stage 名稱在同時存在，它們必須完全一致**。Google Sheets 裡的原始值是 `"1 - Contact"`、`"2 - Qualified"`，程式用 `normalizeStage()` 把它轉成 `"Contact"`、`"Qualified"`。但常數設定檔裡的 key 還是舊的 `"Initial Contact"`、`"Qualified Opportunity"`，永遠對不到，所以查表結果是 0。

> **帶走的原則**：凡是「資料在但畫面是 0」的問題，先查「資料有沒有被轉換過」、「轉換後的結果跟查詢用的 key 是否完全一致」。這類 key mismatch 問題很難靠肉眼看程式碼發現，但症狀很典型。

---

### 坑二：Funnel 圖上有些 Bar 沒有數字標籤（空間不夠但 AI 不知道）

**症狀**：Pipeline Stage 圖的某幾個 Stage Bar 太窄，數字被塞進去卻看不見，視覺上就像那個 Stage 沒有標籤。

根本問題是：標籤預設放在 Bar 裡面，但當 Bar 的寬度小於一定比例時，文字就超出邊界或被截掉了。解法是**判斷 Bar 寬度，太窄就把標籤移到 Bar 右邊**。

> **帶走的原則**：圖表的「看不見」問題，不是資料問題，是 **overflow 行為**問題。只要可視區域有截切，裡面的內容就可能消失。解法通常是把元素搬到容器外面。

---

### 坑三：Proposal 頁面的語言切換破壞 HTML 結構（JS 換 innerHTML 的副作用）

**症狀**：中英文切換時，某個包含子元素的容器，切換後裡面的 HTML 結構消失了。

原因是：`data-ja` 屬性被加在一個有子元素的容器上，語言切換時 JS 直接覆蓋了整個 `innerHTML`，子元素當然也一起消失。

> **帶走的原則**：如果要讓 JS 安全地替換文字，`data-ja` 只能加在「裡面只有純文字、沒有其他子元素」的節點上。否則切換語言 = 刪掉結構。

---

## 五、Activity Log 與 Last Updated（2026-03-22 新增）

POC 功能穩定後，接下來遇到一個管理層很在意的問題：**「這個案子最後是什麼時候更新的？業務有在跟嗎？」**

### 為什麼不直接加一欄讓業務填日期

最初的想法是在 Pipeline 大表加一欄「最後更新時間」，請業務手動填。但人一定會忘記填。所以改用 **Apps Script 的 `onEdit` trigger**——只要業務編輯了 Pipeline 的任何欄位，R 欄自動寫入當前時間。零操作負擔。

### Activity Log 的設計思路

備註欄（Notes）有另一個問題：業務更新時會直接覆蓋舊的內容，歷史就不見了。

跟 AI 討論了幾種方案：

| 方案 | 做法 | 問題 |
|------|------|------|
| 同一格往下加 | 業務在 Notes 欄用日期前綴換行寫 | 格式混亂、容易誤刪 |
| 每人一份 Log sheet | 每個業務有自己的紀錄表 | 管理層要看全貌時拼不起來 |
| **共用 Activity Log sheet** | 全部人寫同一張表，用 author 欄區分 | ✅ 最簡單、Dashboard 好讀 |

最後選了共用 Activity Log，關鍵設計是**讓業務只需做兩件事：選案件名 → 寫備註**，其他全自動。

### Google Sheets 端的設定

這是這次最有趣的部分。Activity Log 有 5 欄，但業務只操作其中 2 欄：

| 欄 | 內容 | 來源 |
|----|------|------|
| A | Deal Name | **Dropdown**（來源 `Pipeline!B:B`，可打字搜尋） |
| B | Date | Apps Script 自動填 |
| C | Deal ID | INDEX/MATCH 公式自動帶 |
| D | Sales Rep | INDEX/MATCH 公式自動帶 |
| E | Note | 業務手動填 |

幾個關鍵的設計決定：

- **用 Deal Name 而不是 Deal ID 做 dropdown**——因為案子越來越多時，沒人記得 EJP-047 是哪個案子。用 Deal Name 做入口，業務打幾個字就能找到，Deal ID 讓公式自動帶出來就好。
- **B/C/D 欄設定 Protect range（Only you）**——防止業務不小心刪到公式。這是 Google Sheets 原生功能，不用寫程式碼。
- **Apps Script 同時處理兩件事**——Pipeline 的 `onEdit` 更新 R 欄時間，Activity Log 的 `onEdit` 自動填日期和公式。一個 function 搞定。

> **帶走的原則**：Google Sheets 的 Data Validation + INDEX/MATCH + Protect range 這三個原生功能組合起來，就能做出「半自動表單」的體驗，不需要 Google Forms 或任何外部工具。

### Dashboard 端怎麼呈現

Dashboard 改動反而不大：
- Deal List 表格多一欄 **Last Updated**
- DealModal 底部多一個 **Activity Timeline**（teal 色時間線，按時間倒序）
- 案件的 Last Updated 取 Pipeline R 欄和 Activity Log 最新日期中較新的那個
- Deal List 內建 Updated filter（Last 3/7/14/30 Days + Inactive），只影響表格不影響圖表

### 踩到的坑

#### 坑：Apps Script 按 ▶️ 執行噴錯

測試 Apps Script 時，在編輯器裡按了 ▶️ 執行按鈕，結果報 `TypeError: Cannot read properties of undefined (reading 'source')`。

原因很簡單：`onEdit(e)` 是事件觸發的，手動執行時沒有 event 物件（`e`），所以 `e.source` 是 undefined。正確的測試方式是直接回到 Google Sheets 編輯儲存格。

> **帶走的原則**：Google Apps Script 的 `onEdit` 不能手動測試，只能透過實際編輯觸發。如果要 debug，用 `Logger.log()` 配合觸發，然後到 Execution log 裡看結果。

#### 坑：Activity Log 更新了但 Last Updated 沒動

業務在 Activity Log 新增了一筆，但 Pipeline 大表的 Last Updated 欄還是空的。原因是 `onEdit` 只監聽被編輯的那張 sheet——編輯 Activity Log 不會觸發 Pipeline 的 `onEdit`。

解法：不在 Google Sheets 端處理，而是在 **Dashboard 端合併**——讀取兩邊的日期，取較新的那個。這樣不管業務是直接改 Pipeline 還是加 Activity Log，Dashboard 都能反映最新狀態。

> **帶走的原則**：當資料分散在多張 sheet 時，不要試圖用 Apps Script 讓它們完美同步。在讀取端（Dashboard）做合併，比在寫入端做連動更穩定、更好維護。

---

## 六、最終樣貌

**儀表板本體**（[ejp-pipeline-dashboard.vercel.app](https://ejp-pipeline-dashboard.vercel.app/)）：
- 密碼保護（`ejp@demo`），全站 Middleware 守衛
- 7 個 KPI 卡（總案件數、活躍案件、Pipeline 總額、Closed Won、勝率、加重預測、當月新增）
- 6 個圖表（Pipeline by Stage funnel、Vertical 分布、Stage vs 加重預測、地區 Top 10、業務員排行、案件來源）
- 案件列表（搜尋、多維篩選、點開看詳情、**表頭鎖定**）
- **Activity Timeline**：DealModal 內顯示該案件的所有歷史備註，按時間倒序
- **Updated filter**：在 Deal List 表格內篩選最近更新的案件（3/7/14/30 天 + Inactive）
- **Deal ID 模糊搜尋**：`EJP002` 和 `EJP-002` 都能找到同一筆
- **Model Summary tab**：所有 Model 的 QTY、單價、Standard Value 彙總；含月份分布 pivot（Model × 月份），支援 Close/Deploy Date 切換、季度小計
- **Sales Rep Summary tab**：業務員 × 月份 pivot，案件數數字可點擊帶 filter 跳回 Deal List
- 日本地圖（地區案件熱點）
- 英/日雙語切換
- 基本 RWD（手機可用，平板以上最佳）

**Google Sheets 端**：
- Pipeline sheet（A:R，18 欄）：R 欄 `last_updated` 由 Apps Script 自動填
- Activity Log sheet（A:E，5 欄）：Dropdown + INDEX/MATCH + Protect，業務只需操作 2 欄
- Price List sheet（A:C）：產品型號與單價對照
- Settings sheet：JPY 匯率

**交付物**：
- `/proposal.html`：分公司導入說明（繁中/日文）
- `/it-request.html`：IT 部門 Azure App Registration 申請文件（含 SSO 長期方案）
- `docs/sharepoint-deployment-guide.md`：開發者遷移指南（Google Sheets → SharePoint）

---

## 七、如果繼續往下

- **資料來源遷移**（等待 POC 確認中）：Google Sheets 是 POC 用的，分公司確認後接 SharePoint。程式架構已預留好，換一個 `lib/sharepoint.js` 就搞定，其他不動。IT 申請文件已備妥。
- **存取控制升級**：目前是共用密碼，長期應接 Microsoft SSO，只有公司帳號才能進。IT 申請文件已備妥，等 POC 驗證後的下一步。
- **Activity Log 進階**：目前 Activity Log 是手動填的，未來可以考慮讓 Dashboard 直接新增備註（需要 Google Sheets API 的寫入權限）。
- **定期 Email Report**：把儀表板每週快照自動寄給管理層，讓不登入的人也能收到資訊。

---

## Takeaway

**這個案例展示了什麼思路**：不是「我要做一個儀表板」然後從零開始設計，而是「我有一份 Sheets，我要讓它變得可見」——把已知的正確答案當起點，讓 AI 逐步逼近它。這個方式的效率遠高於從需求文件開始。

**Activity Log 的決策過程**：這次跟 AI 的討論模式是「先問需求（我要看到更新時間），再問方案（有哪些做法），再問取捨（要不要上資料庫）」。AI 建議不上 Supabase 的理由很清楚——資料量不大、業務流程還在 Google Sheets 上、多加一層同步只會增加維護負擔。這個「不做什麼」的決策，跟「做什麼」一樣重要。

**可移植性**：Google Sheets 的 Data Validation + INDEX/MATCH + Apps Script + Protect range 組合，可以套用在任何「多張 sheet 需要連動、但不想搬到資料庫」的場景。Activity Log 的設計模式（選名稱 → 自動帶其他欄位 → 鎖定公式欄）在 CRM、專案管理、客服紀錄等場景都能直接複製。

**如果你也要做，先問 AI 什麼**：
- 「我的 Google Sheets 有一張主表和一張紀錄表，我希望紀錄表選了案件名後自動帶出 ID 和負責人，要怎麼設定 Data Validation 和 INDEX/MATCH？」
- 「我想在 Dashboard 上顯示每個案件的最新更新時間，但更新可能來自不同的 sheet，Dashboard 端要怎麼合併？」

_開發日期：2026-03-11，最後更新：2026-03-22_
