---
title: cancan 團購管理系統
created: 2026-05-12
type: ai-collaboration
category: 工具開發
tags: [Google Apps Script, Google Sheets, Web App, HTML Template, AI協作, vibe-coding, 團購]
status: 已完成
---

# cancan 團購 — 從 Google Form 到設計版出貨單的 AI 協作紀錄

> 🤖 **我用 AI 做了什麼**：把每檔次團購訂單整理的痛苦從「下午 4 小時手動」變成「全自動」，加上每筆訂單一鍵生成的設計版宅配出貨單 PDF。
> ⏱ **沒有 AI 的話**：找工程師外包估計 3-6 萬，外加每次有修改要 ticket。自己做？我連 Apps Script 是什麼都不太確定。
> ✅ **最終成果**：一份 Google Sheet 上自動跑 6 個分析分頁、即時 email 通知、設計版宅配 PDF（用 Apps Script Web App + HTML 模板，瀏覽器列印零月費）。

> cancan 食物設計工作室每季做一次團購，整理訂單跟出貨單是每次最累的事。試了 AI 一起做，從 Google Sheet 自動化開始，最後做到品牌設計版 PDF 出貨單。

---

## 一、為什麼要做這件事

cancan 的團購流程一直長這樣：

1. 開 Google Form 給客人填
2. Form 回應自動進 Google Sheet
3. **然後就卡住了** — 大量手動工作：
   - 加總每個品項的買賣量（要跟廠商叫貨）
   - 加總每筆訂單的金額（要算給客人匯款）
   - 套折扣規則（乾熟老朋友滿 6 項折 \$100）
   - 監控酸種麵包總量（150 顆上限）跟紅隼三號白酒（12 瓶上限）
   - 追蹤誰匯款了、誰沒匯（後五碼對帳）
   - 整理出貨資料給包貨同仁
   - **每筆宅配訂單要做一張包裹清單**

每檔次團購 30–50 筆訂單，這套流程大概要花一個下午（4 小時左右）整理。而且**出錯率很高** — 某個品項加錯、某個客人少匯款沒抓到、出貨單寫錯地址。

最痛的是「**宅配出貨單**」。同仁要一筆一筆從 Sheet 抓資料、填到一個範本上、印出來夾進包裹。沒有設計感、容易出錯、也不像「cancan 品牌」會送出去的東西。

**目標**：
- 訂單整理全自動
- 即時知道誰下單、誰沒匯款、上限快滿了沒
- 每筆訂單**一鍵**產生品牌設計版的出貨單 PDF
- **零月費**（cancan 是小工作室，能不花就不花）

---

## 二、技術選型

| 工具/技術 | 選的理由 |
|---|---|
| **Google Apps Script** | 跟我們現有的 Google Form / Sheet 完全整合、免費、無伺服器、沒額外帳號要管 |
| **Apps Script Web App** | 想做出貨單 PDF，這是**唯一**能在不付 PDF 轉換服務（PDFShift 月費 \$9）下做到設計版輸出的路徑 |
| **HTML + 內嵌 CSS** | 設計版出貨單的「設計力」靠這個。用瀏覽器原生列印（Cmd+P）轉 PDF 品質完美 |
| **Claude Design** | 出貨單視覺由 claude.ai 的 artifact 模式產出（A4 直式、Fraunces serif + Noto Sans TC、品牌橘 #BD4218），再請 AI 套上 placeholder 變模板 |
| **html2canvas** | 「存為圖片」按鈕用這個 CDN lib，把整張 sheet 轉 PNG（方便貼 IG / LINE） |

> 選型原則：**能用 Google 自家家族服務就用**（已經在用 Sheet、Drive、Gmail，多 onboarding 一個服務都是負擔）。需要設計力但 Google 工具做不到的（出貨單 PDF），才往外找方案。

---

## 三、外部服務與金鑰

| 服務 / 模型 | 用途 | 類型 | 備註 |
|---|---|---|---|
| Google Apps Script | 整套後端邏輯 + Web App 部署 | 平台 | 免費，每天 100 封 email 限制 |
| Google Sheets API（內建） | 讀寫試算表 | 平台 | Apps Script 已內建 |
| Google Drive API（內建） | 抓 logo 圖檔 + 存放 PDF | 平台 | 同上 |
| Gmail API（內建） | 寄即時通知 email | 平台 | 同上 |
| html2canvas CDN | 「存為圖片」按鈕用 | 外部 lib | `cdnjs.cloudflare.com` 載入，免費無 key |
| Google Fonts | Fraunces / Noto Sans TC / JetBrains Mono | 外部 lib | 模板用 `<link>` 載入，免費無 key |
| Drive Logo File | cancan logo 圖檔 | Drive 檔案 | `CONFIG.LOGO_FILE_ID` 指定，自動 base64 嵌入 |
| Drive PDF Folder | 出貨單 PDF 存放位置 | Drive 資料夾 | 使用者 Cmd+P 時自己選擇位置（系統不自動上傳） |

> ✅ 完全沒有付費 API、沒有信用卡綁定、沒有 token 要管。所有 secret 都不存在。

---

## 四、系統架構

```
[Google Form]
   ↓ 客人填表
[表單回應分頁]
   │  ← 同仁可在最右側加「後五碼」欄當對帳 source of truth
   ↓ onFormSubmit auto trigger
[Apps Script: computeAllOrders_ + 6 個 build 函式]
   ↓
[6 個分析分頁]
   ├── 儀表板（KPI、上限警示、Todos）
   ├── 訂單明細（每筆一列，最右邊有「📄 出貨單」HYPERLINK）
   ├── 品項採購總（含自取/宅配分欄方便叫貨）
   ├── 通路分析（甜甜圈圖表）
   ├── 匯款對帳（待對帳/已對帳分類）
   └── 異常清單

[Apps Script Web App: doGet(e)]
   ↑ 訂單明細點「📄 開啟」（HYPERLINK 帶 row 參數）
   ↓ renderPackingSlipHtml_ 把訂單資料填入 packing_slip_template.html
   ↓ HtmlService 渲染 → 瀏覽器分頁
   ↓ 使用者 Cmd+P / 點「📄 存為 PDF」按鈕
[品牌設計版 PDF / PNG]

[email 通知（Gmail）]
   ↑ onFormSubmit 同步觸發
   ↓ 有異常/超量/接近上限才寄信
[支援多收件人]
```

---

## 五、推進方式

整個系統是分**多個 session 逐步堆疊**起來的，不是一次到位。每個 session 解一塊痛點：

### 第一階段：先把 Google Sheet 整理自動化

從最痛的開始：每次要花 4 小時整理的訂單明細。

我說：「我有一份 Google Form，回應放在 Google Sheet，可以幫我做 Apps Script 自動整理嗎？」

AI 第一輪沒急著寫 code，反而問了好幾個關鍵問題：
- 有沒有折扣規則？怎麼套？
- 上限品項是哪些？怎麼算？
- 客戶後五碼怎麼對帳？

這把我逼著想清楚**規則本身**（不只是「做出來」而是「規則寫清楚」）。最後 CONFIG 區塊就是這次對話的產物，把所有「會變的東西」抽出來放最上面。

### 第二階段：發現 Google Form 標題有玄機

腳本第一次跑時，「酸種麵包」這個品項抓不到。一查才發現，Google Form 上的「題組標題」跟「sheet 上看到的欄位名稱」不一定一樣 — Form 編輯介面看到的可能是 section divider，不是 question title。

AI 想出的解法：**容錯式比對**。比對題目關鍵字、選項關鍵字、實際資料內容三層 fallback，總有一層會中。

### 第三階段：通路分析 + 圓餅圖

加了「通路分析」分頁，用 Apps Script 的 Charts 功能畫甜甜圈圖。但這裡有個坑：`sh.clear()` 清不掉圖表，要 `sh.getCharts().forEach(c => sh.removeChart(c))` 手動清。AI 第一次寫忘記、第二次重整就堆兩個圖。修。

### 第四階段：通知系統 + 多收件人

加了 onFormSubmit 觸發 email 通知。但只在「異常/超量/接近上限」才寄，避免每筆訂單都轟炸。順手也接了 onEdit — 同仁在表單回應的後五碼欄打字後 2 秒，訂單明細的對帳狀態就自動更新。

### 第五階段（重點）：宅配出貨單從「醜」變「精品」

這是這次的高潮。本來用 Apps Script 內建的 `DocumentApp` 產 Google Doc，但**醜**。標題就是黑字白底、品項一條一條列出來、看起來像收據。客人收到包裹打開看到這張紙的瞬間，cancan 的品牌感歸零。

跟 AI 開了一個獨立的設計討論，得到結論：
- Google Doc 的設計天花板就到這了
- 真正能呈現「精品品牌」的方法只有兩條：
  1. 付費 PDFShift 服務（\$9/月）轉 HTML → PDF
  2. 自架 Apps Script Web App，用瀏覽器原生列印（**零月費**）

選了第 2 條。流程：

1. 先在 claude.ai 用 artifact mode 設計一份 HTML 出貨單（cancan logo、Fraunces serif、橘色品牌色塊、印刷對位記號）
2. 把 HTML 抽出 `{{placeholder}}` 做成模板
3. Apps Script `doGet(e)` 讀訂單資料 → 填模板 → 回傳 HTML
4. 訂單明細最右邊放 HYPERLINK 公式指向 Web App URL
5. 點開就是設計版頁面，Cmd+P 存 PDF

第一次 deploy 卡在 URL 一直壞掉 — 原來 Apps Script 的「新增部署作業」每次都建新 URL，舊的 URL 失效後就出現 Drive 的「無法開啟」錯誤。解法：永遠用「**管理部署作業 → 編輯 → 新版本**」（URL 不變）。

### 第六階段：防呆 + 細節打磨

- 訂單明細的「📄 出貨單」欄改成 IF 公式：沒後五碼顯示「⏳ 待收款」（不能點），有後五碼才顯示「📄 開啟」 — 避免誤對未匯款客人出貨
- 模板加「📄 存為 PDF」+「🖼️ 存為圖片」浮動按鈕
- 動態 setTitle 讓 PDF 預設檔名是「客戶名_出貨單_#03」
- @page :first 處理跨頁列印（第一頁 full bleed、第二頁開頭加 margin）
- 品名「nowrap」+ tag 欄位拉寬，防斷行
- 「乾熟老朋友滿 6 項折 \$100」自動偵測並顯示折扣 block

---

## 六、踩到的坑，讓我更懂的事

### 坑一：Apps Script Web App 兩種「部署」差很多

原本以為「重新部署一次就好」。結果連續發生 URL 變來變去、HYPERLINK 全部失效。

- **「新增部署作業」** = 全新 deployment，**URL 會變**
- **「管理部署作業 → 編輯 → 新版本」** = 沿用同一個 deployment，**URL 不變**

> 帶走的原則：**第一次部署用「新增」，之後永遠用「編輯 → 新版本」**。每次改 code 都這樣，URL 不會變，HYPERLINK 不會失效。

### 坑二：`ScriptApp.getService().getUrl()` 多 deployment 時會抓錯

我以為腳本可以「自動知道」當前 Web App URL。但有多個 deployment 時，這個函式會回傳哪一個是不確定的（有時是已封存的舊版）。

結果：訂單明細的 HYPERLINK 指向已死掉的 URL，點下去就是 Drive 錯誤頁。

解法：在 `CONFIG.WEB_APP_URL` **hardcode** 一份正確的 URL，腳本優先用這個。auto-detect 當 fallback。

> 帶走的原則：**Web App URL 不要依賴 auto-detect，hardcode 進 config**。

### 坑三：HtmlService iframe 沙盒 → PDF 預設檔名抓錯

我在模板裡寫了 `document.title = '{客戶名}_出貨單_#03'`，希望 Cmd+P 時瀏覽器自動帶這個檔名。結果還是顯示「cancan 出貨單」 — 因為 Apps Script Web App 把 HTML 包在 iframe 裡，瀏覽器列印用的是**外框 title**，不是 iframe 裡的。

解法：在 `doGet()` 用 `HtmlService.createHtmlOutput(html).setTitle("客戶名_出貨單_#03")` 動態設外框 title。

> 帶走的原則：**Apps Script Web App 要動態檔名，必須在 doGet 設 setTitle**，HTML 裡寫 `document.title` 沒用。

### 坑四：列印跨頁時樣式破版

品項多的訂單會佔兩頁。第一頁底部把 TOTAL 標籤切走、第二頁開頭只有「NT$ 4,660」金額，加上 footer-wrap 的 `margin-top: auto` 把地址區塊推到 sheet 底部，造成第二頁中間一片空白。

解法：
- 給 `.menu-total` / `.menu-discount` 加 `page-break-inside: avoid`
- 列印時把 `footer-wrap` 的 `margin-top` 改成固定 14mm
- `@page :first { margin: 0 }` 讓第一頁 full bleed，第二頁起加 top margin

> 帶走的原則：**列印 CSS 跟螢幕 CSS 是兩套**，要分開測試。`@page :first` 是處理多頁列印的好工具。

### 坑五：分頁名稱加 emoji 跟鎖頭 icon 重複

我給每個分頁加了 emoji（📊 訂單明細、📦 品項採購總...）覺得有設計感。後來啟用「Warning Only 保護」後，每個分頁名旁邊都多了一個 🔒 鎖頭 icon。鎖頭 + emoji + 文字 = 太雜亂。

拿掉 emoji。鎖頭專心表達「自動產生別亂改」、純文字名稱乾淨。

> 帶走的原則：**裝飾性 emoji 用在沒有其他視覺暗示的地方才有意義**。已經有功能性 icon（鎖頭），裝飾就多餘。

---

## 七、最終樣貌

整套系統現在跑在 cancan 第二波團購的正式 Google Sheet 上。同仁的流程：

1. **客人填 Form** → 後台自動建好訂單、計算金額、套折扣、檢查上限
2. **客人匯款** → 同仁在「表單回應分頁」的「後五碼」欄填 5 位數 → 2 秒內訂單明細 / 匯款對帳 / 儀表板全部同步
3. **看現況** → 打開儀表板，KPI + Todos 一目瞭然
4. **跟廠商叫貨** → 看「品項採購總」分頁，自取/宅配分欄
5. **包貨出宅配** → 訂單明細最右邊點「📄 開啟」 → 新分頁是設計版出貨單 → Cmd+P 存 PDF（檔名自動是「葉丁綸_出貨單_#03」）→ 列印 → 夾進包裹
6. **異常通知** → 異常訂單會收到 email（地址漏填、酸種超量、選「其他」取貨等）

設計版出貨單長相：
- 左上角真實 cancan logo（米色紙底、輕微紙紋）
- 右上 colophon「Food Design Studio · TAIPEI · EST. 2024」+ 檔次標籤
- 大字 italic「{客戶名}，你好。」+ 訂單編號
- tasting menu 風格品項清單（品名 / 數量 / 單價 / 保存方式有色點）
- 自動折扣 block（有觸發折扣才出現）
- 大字斜體 TOTAL
- 底部地址 + IG QR code（真實可掃）
- 印刷對位記號（四角小直角線）

---

## 八、如果繼續往下

每個下一步都是「**真的會繼續做的事**」，不是空想：

- **LINE / Telegram 即時通知** — 補強 email 通知（LINE Notify 已停服，要走 LINE Messaging API 或 Telegram Bot）。目標：手機鈴一下就知道新訂單來了
- **跨檔次營運分析** — 哪個品項回購率高、哪個通路客人忠誠度高、客戶 LTV 排行
- **自家團購網頁取代 Google Form** — Form 做不到搶購進度條、品項照片預覽、品項分類 hover、行動裝置友善的選購流程。網頁送進來的訂單還是寫進同一份 Google Sheet，後端腳本繼續用

---

## Takeaway

### 這個案例展示了什麼

**把 AI 當「整個團隊」用，不只是 code generator。**

整個 5 個 session 的開發過程中，AI 扮演了：
- **業務分析師**：第一輪追問我規則細節，逼我想清楚 source of truth 是什麼
- **架構師**：設計 CONFIG 抽象、kept 字典用時間戳當 key（不是 name+phone）、normalize at source pattern
- **設計師**：用 claude.ai artifact 出 HTML 模板雛形，包含字體選擇、版面決策理由
- **debug 夥伴**：4 個坑都是 AI 跟我一起 trace 出來的（特別是 Web App deployment URL 那個）
- **文件撰寫**：CLAUDE.md / README.md 都是 AI 寫的，包括 12 條 Common Pitfalls

我的角色主要是：
- **判斷邊界**：「這個視覺夠精品嗎？」「這個防呆策略 OK 嗎？」
- **提供現場反饋**：截圖告訴 AI 哪裡破版、哪裡看起來怪
- **做最終取捨**：付費 vs 自架、保留 vs 移除某些設計元素

### 可移植性

**完全可以複製到**：
- 任何用 Google Form + Google Sheet 收訂單的小規模團購
- 想做設計版出貨單但不想付月費的 brand
- 需要「客戶/同仁分權限編輯」的試算表協作場景

**這個案例的特殊條件**：
- cancan 是小規模團購（每檔次 30-50 筆），所以 Apps Script 的執行限制（6 分鐘 / 函式、100 封 email / 天）夠用。如果是 500+ 筆的大團購，可能要拆批處理或改架構
- HTML 模板要靠 Claude Design 出視覺。如果讀者沒有「會用 claude.ai artifact mode」的能力，可以改用免費的 HTML email 模板生成器當起點
- Web App 部署為「具有 Google 帳戶的所有人」存取等級。如果做的是公開購物車（任何人都能填），要評估資安

### 如果你也要做，先問 AI 什麼

1. **第一句先給情境，不要直接要 code**：
   > 「我有一份 Google Form 收訂單，回應自動進 Google Sheet。我每次手動整理要花 4 小時，痛點是 A / B / C。可以用 Apps Script 幫我做哪些自動化？先告訴我可能的方向，不用寫 code。」

2. **第二句問抽象，不要問實作**：
   > 「我有 X 條業務規則（折扣 / 上限 / 通路）。怎麼設計 config 結構讓未來改規則只動一個地方、不用動 code？」

這兩句下來，你會得到一份「**有架構觀點**」的初稿，而不是一份「**符合表面需求**」的程式碼。後續所有 iteration 都好做很多。

---

_開發階段：2026 Q2 第二波團購_
_紀錄日期：2026-05-12_
