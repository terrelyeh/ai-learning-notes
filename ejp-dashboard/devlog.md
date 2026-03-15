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

## 五、最終樣貌

**儀表板本體**（[ejp-pipeline-dashboard.vercel.app](https://ejp-pipeline-dashboard.vercel.app/)）：
- 密碼保護（`ejp@demo`），全站 Middleware 守衛
- 7 個 KPI 卡（總案件數、活躍案件、Pipeline 總額、Closed Won、勝率、加重預測、當月新增）
- 6 個圖表（Pipeline by Stage funnel、Vertical 分布、Stage vs 加重預測、地區 Top 10、業務員排行、案件來源）
- 案件列表（搜尋、多維篩選、點開看詳情）
- 日本地圖（地區案件熱點）
- 英/日雙語切換
- 基本 RWD（手機可用，平板以上最佳）

**交付物**：
- `/proposal.html`：分公司導入說明（繁中/日文）
- `/it-request.html`：IT 部門 Azure App Registration 申請文件（含 SSO 長期方案）
- `docs/sharepoint-deployment-guide.md`：開發者遷移指南（Google Sheets → SharePoint）

---

## 六、如果繼續往下

- **資料來源遷移**：Google Sheets 是 POC 用的，正式版應該接 SharePoint（分公司的 Excel 在那裡）。程式架構已預留好，換一個 `lib/sharepoint.js` 檔案就搞定，其他不動。
- **存取控制升級**：目前是共用密碼，長期應接 Microsoft SSO，只有公司帳號才能進。IT 申請文件已備妥，等 POC 驗證後的下一步。
- **資料由業務自己填**：現在是工程師（AI）控制格式，未來要讓業務人員能直接更新 Excel，需要定義欄位規格並做驗證提示。
- **定期 Email Report**：把儀表板每週快照自動寄給管理層，讓不登入的人也能收到資訊。

---

## Takeaway

**這個案例展示了什麼思路**：不是「我要做一個儀表板」然後從零開始設計，而是「我有一份 Sheets，我要讓它變得可見」——把已知的正確答案當起點，讓 AI 逐步逼近它。這個方式的效率遠高於從需求文件開始。

**可移植性**：這個方法適用於任何「資料已經在某個地方、只差一個好看介面」的場景。不適用於需要設計全新資料流或資料庫架構的情況。

**如果你也要做，先問 AI 什麼**：
- 「我有一份 Google Sheets，欄位是 A/B/C，幫我用 Next.js + Recharts 做一個可以顯示 [具體圖表] 的儀表板，部署在 Vercel」
- 「我的圖表上有些 Stage 顯示 0，但資料在 Sheets 裡確實有值，可能是什麼原因？幫我列出 checklist 逐一排查」

_開發日期：2026-03-11_
