# 雲端自動化的兩種工具 — GitHub Actions vs Vercel Function

> 食料庫團隊 · 內部學習筆記 · 2026-05-11
>
> 一份 10 分鐘看完的入門。看完你會知道：兩者各自做什麼、怎麼搭、什麼時候選哪個。

---

## TL;DR

| | 內容 |
|---|---|
| **GitHub Actions** | 「在你不在的時候、自動做事」— 排程 / push / 手動 → 跑一連串指令 |
| **Vercel Function** | 「有人點按鈕、立刻回應」— 收 HTTP 請求 → 回一個 response |
| **共同點** | 都是 serverless、都跟 git 整合、都免費 tier 慷慨 |
| **關鍵差別** | Actions 等事件、跑批次；Function 等使用者、即時回應 |
| **互補關係** | 不是替代，可以一起用（食料庫就是這樣）|

---

## 01. 兩者的「一句話差別」

| | **GitHub Actions** | **Vercel Function** |
|---|---|---|
| **本質** | 雲端**批次任務**執行器 | 雲端 **API endpoint** |
| **誰觸發** | 時間 / git 事件 / 手動按按鈕 | **使用者的 HTTP request** |
| **誰在等結果** | 沒人在線上等 — 跑完寫 log | **使用者瀏覽器在等** — 必須立刻回 |
| **可以跑多久** | 最多 6 小時 | 通常 < 10 秒（serverless 限制） |
| **產出** | commit / 部署 / 通知 | HTTP response（JSON / HTML / redirect） |

簡單記法：

> 🛠 **Actions 是「工人」**，等工頭排班 / 鬧鐘響了開工。
> 🛎 **Function 是「櫃台」**，等客人按鈴立刻服務。

---

## 02. GitHub Actions — 是什麼？

GitHub 提供的一個「**事件 → 跑程式碼**」系統。

意思是：你定義「當 repo 發生 X 事情時、自動執行 Y 一系列指令」。X 可以是 push、有人開 PR、有 issue 被建立、每天早上 9 點到了、有人手動按按鈕…

每一份「X → Y」的對應寫法，就叫一個 **Workflow**（YAML 檔，放 `.github/workflows/`）。

### 完整層級

```
GitHub Actions                ← 平台（GitHub 內建）
└── Workflow                  ← .github/workflows/*.yml 一個檔 = 一個任務
    └── Event                 ← 何時觸發（push / cron / 手動 / issue...）
        └── Job               ← 一個 workflow 可以有多個 job
            └── Step          ← 一步一步要跑什麼指令
                └── Action    ← 可重用元件（checkout / setup-uv ...）
                    ↑ 跑在 Runner（GitHub 提供的雲端 VM）
```

> ⚠️ **常見混淆點**：「**Action**」（可重用元件）跟「**GitHub Actions**」（整個平台）名字一樣但**不是同一層**。Action 像「演員 / 道具」，GitHub Actions 像「整個劇場」。

### 劇場比喻

| 劇場 | ↔ | GitHub Actions |
|---|---|---|
| 整個劇場系統 | = | **GitHub Actions** 平台 |
| 一齣戲的劇本 | = | **Workflow**（.yml 檔） |
| 開演鈴聲 | = | **Event**（`on:` 那一段） |
| 一幕戲 | = | **Job** |
| 一句台詞 / 一個動作 | = | **Step** |
| 演員 / 道具（可重用） | = | **Action**（`uses: ...` 拉的元件） |
| 演出場地 | = | **Runner**（`ubuntu-latest` 之類） |

---

## 03. Vercel Function — 是什麼？

Vercel 提供的「**HTTP request → 跑程式碼 → 回 response**」系統。

俗稱 **Serverless Function** — 你不用架 server、不用管 24 小時在線；當有人打你的 endpoint URL 時，Vercel **臨時開一台 VM 跑你的 code、跑完關機**。

只要把一個 `.js` 或 `.py` 檔放進 `api/` 資料夾、push 上去，**那個檔案就自動變成一個 endpoint**：

```
api/report-typo.js  →  https://your-site.vercel.app/api/report-typo
api/hello.js        →  https://your-site.vercel.app/api/hello
```

### Vercel Function 在做什麼

```
┌─ 使用者瀏覽器 ─┐
│                │
│  fetch(...)    │ ─── POST → ┐
│                │             │
└────────────────┘             │
                               ▼
                  ┌─ Vercel Function（你寫的 .js）──┐
                  │                                  │
                  │  1. 讀 req.body                 │
                  │  2. 驗證資料                    │
                  │  3. 呼叫外部 API（DB、GitHub… ）  │
                  │  4. 組 response                 │
                  │                                  │
                  └──────────────┬───────────────────┘
                                 │
                                 ▼ 200 + JSON
                  ┌─ 使用者瀏覽器 ─┐
                  │                │
                  │  收到結果       │
                  │  更新 UI       │
                  │                │
                  └────────────────┘
```

### 最常見的使用情境

| 情境 | 例子 |
|---|---|
| **收前端表單** | 使用者填表單 → POST 到 function → 寫進資料庫 / 寄 email |
| **API proxy** | 不能在前端直接呼叫的 API（需 secret token）→ function 中轉 |
| **Auth callback** | OAuth 登入完跳回來 → function 換 token |
| **Webhook 接收** | Stripe / Slack 等服務的 webhook → function 處理 |
| **動態內容** | 動態生成 OG image / SEO meta / PDF |

---

## 04. 並排對比

| 維度 | **GitHub Actions** | **Vercel Function** |
|---|---|---|
| **觸發者** | 時間 / git 事件 / 手動 | HTTP request |
| **誰在等** | 沒人 — 開發者事後看 log | 使用者瀏覽器即時等 |
| **可跑多久** | 最多 6 小時 | 通常 < 10 秒（hobby tier） |
| **冷啟動** | ~30 秒（開 VM） | ~1-2 秒（warm < 100ms） |
| **典型場景** | CI/CD、批次、排程、爬蟲 | 表單、API、auth、webhook |
| **檔案位置** | `.github/workflows/*.yml` | `api/*.js` 或 `api/*.py` |
| **語言** | YAML + shell + 任何 CLI | JavaScript / Python / etc. |
| **回應格式** | log（給人看的文字輸出） | HTTP response（給 client 的資料） |
| **學習曲線** | 學 YAML 結構 + shell | 學 req/res 處理 |
| **免費額度** | Public repo 無上限；Private 2000 分鐘/月 | 100k 次呼叫 / 月 |

---

## 05. 食料庫專案：兩者怎麼一起用

最有 aha 感的部分 — 我們專案**同時用了兩者**，且**它們串成一條 pipeline**：

```
┌────────────────────────────────────────────────────────────────┐
│                                                                │
│  ① 使用者邊看逐字稿、發現錯字                                    │
│      ↓                                                         │
│  點 ✏️ icon → 填表單 → 送出                                      │
│      ↓ HTTP POST                                               │
│                                                                │
│  ┌──────────────────────────────────────────────┐              │
│  │  Vercel Function：api/report-typo.js         │              │
│  │  ─────────────────────────────────────       │              │
│  │  • 收前端 POST                                │              │
│  │  • 驗證資料（honeypot, rate limit, ...）      │              │
│  │  • 用 PAT 建 GitHub Issue                    │              │
│  │  • 回 200 給前端                              │              │
│  └─────────────────┬────────────────────────────┘              │
│                    ↓                                            │
│  ② GitHub Issue #N 出現（標籤 transcript-fix）                  │
│                                                                │
│      ↓ (累積中、管理者每週看一次)                                 │
│                                                                │
│  ③ 管理者進 Actions 頁面、填表單觸發 add-term-fix workflow      │
│      ↓                                                         │
│                                                                │
│  ┌──────────────────────────────────────────────┐              │
│  │  GitHub Actions：add-term-fix.yml            │              │
│  │  ─────────────────────────────────────       │              │
│  │  • 把規則寫進 transcribe.py 的 TERM_FIXES      │              │
│  │  • 跑 apply_term_fixes.py 套到所有逐字稿       │              │
│  │  • 重 build episodes.js                       │              │
│  │  • commit + push                              │              │
│  │  • close issue #N + 留 comment                │              │
│  └─────────────────┬────────────────────────────┘              │
│                    ↓                                            │
│  ④ Vercel 偵測 push → 自動部署                                  │
│      ↓                                                         │
│  ⑤ 30-60 秒後使用者重整網站，錯字消失                            │
│                                                                │
└────────────────────────────────────────────────────────────────┘

                ┌────────────────────────────┐
                │ Vercel Function 在這裡：     │
                │   即時回應使用者按鈕         │
                ├────────────────────────────┤
                │ GitHub Actions 在這裡：     │
                │   批次處理 + 部署            │
                └────────────────────────────┘
```

### 為什麼這樣分工？

| 階段 | 為什麼用 Vercel Function | 為什麼用 GitHub Actions |
|---|---|---|
| **使用者送出表單** | 必須即時回 200 給瀏覽器、< 1 秒 | ❌ 30 秒冷啟動太慢，使用者會以為壞了 |
| **批次修錯字** | ❌ 跑 retroactive script 要 1-2 分鐘，function 會 timeout | ✅ 可以跑久、又能 git commit + push |
| **接收外部 webhook** | ✅ 適合 | ❌ Actions 不能直接收 HTTP request |
| **每天爬資料** | ❌ 不能排程觸發 | ✅ cron 內建 |

→ **彼此填補對方做不到的事**。

---

## 06. 最小範例對比

### GitHub Actions — `打招呼` workflow

```yaml
# .github/workflows/hello.yml
name: 打招呼

on:
  push:
    branches: [main]

jobs:
  greet:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v5
      - run: echo "Hello, $(date)"
```

push 到 main → 自動跑 → 在 Actions 頁面看 log。

### Vercel Function — `打招呼` endpoint

```javascript
// api/hello.js
module.exports = function handler(req, res) {
  res.status(200).json({
    message: 'Hello',
    time: new Date().toISOString(),
  });
};
```

push 到 main → 自動部署 → `curl https://你的網址/api/hello` 立刻看到結果。

### 看出差別了嗎？

| | Actions | Function |
|---|---|---|
| **觸發** | git push | 任何人打那個 URL |
| **看結果** | 進 Actions 頁面看 log | 收到 JSON response |
| **執行時機** | push 後幾秒 | 被呼叫的當下 |
| **執行多久** | 看你寫的 step | 必須在 10 秒內結束 |

---

## 07. 什麼時候用哪個？

### 決策樹

```
                你需要自動化什麼？
                       │
       ┌───────────────┼───────────────┐
       ▼               ▼               ▼
   間歇/排程        對外 API       長期執行服務
   git 相關         即時回應       (24/7 連線)
       │               │               │
       ▼               ▼               ▼
 GitHub Actions  Vercel Function   自架 server
       │               │               │
       ▼               ▼               ▼
   寫 .yml        寫 api/*.js     Cloud Run / EC2

  例：             例：              例：
  • 每天爬資料     • 收前端 POST     • 24/7 機器人
  • PR 跑測試      • OAuth callback  • WebSocket 服務
  • 手動工具      • Webhook 接收     • 即時聊天
  • 排程備份      • API proxy       • 遊戲伺服器
```

### 速判規則

| 你的需求關鍵字 | 用哪個 |
|---|---|
| 「每天 / 每週 / 排程」 | **GitHub Actions** |
| 「使用者點了按鈕之後…」 | **Vercel Function** |
| 「跑超過 1 分鐘」 | **GitHub Actions** |
| 「< 5 秒內要有回應」 | **Vercel Function** |
| 「commit / push / PR 自動…」 | **GitHub Actions** |
| 「前端表單送資料」 | **Vercel Function** |
| 「對接外部 webhook」 | **Vercel Function** |
| 「批次處理一堆檔案」 | **GitHub Actions** |
| 「不能在前端暴露 token」 | **Vercel Function** 中轉 |
| 「給管理者用的工具」 | 簡單的 → **GitHub Actions**（workflow_dispatch）；複雜 UI → 加 **Vercel Function** |

### 兩個都不該選的情境

| 情境 | 為什麼 | 該用什麼 |
|---|---|---|
| 24/7 連線（聊天、遊戲）| 都是 serverless、用完即關 | 自架 server（Cloud Run、EC2）|
| 大量 GPU / CPU 計算 | 都不適合長時間重運算 | 專門平台（RunPod、Replicate）|
| 巨量資料儲存 | 都不是儲存服務 | Supabase / S3 / 自架資料庫 |

---

## 08. 總結

**一句話兩者差別**：

> 🛠 **GitHub Actions** 是「工人」— 等鬧鐘響 / 工頭叫，開工跑一份排程。
> 🛎 **Vercel Function** 是「櫃台」— 等客人按鈴，立刻回應。

**最強組合**：兩個一起用。

我們食料庫專案的修錯字流程就是經典範例：

```
使用者按按鈕 → Vercel Function（即時回 200）
                      ↓
              GitHub Issue 累積
                      ↓
            管理者觸發 Actions（批次處理）
                      ↓
                自動部署修正
```

→ **Function 處理「即時、外部」、Actions 處理「批次、git 相關」**。

懂這個區分，就懂為什麼大部分現代 web 專案兩個都會用。

---

_整理日期：2026-05-11 · 食料庫團隊內部學習筆記_
