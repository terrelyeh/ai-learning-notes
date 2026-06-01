---
title: Network AI-Assistant Skill v3 · 從工具到產品：pivot、雙 Agent、與製造紀律
created: 2026-06-01
type: ai-collaboration
category: 工具開發
tags: [AI-協作, Claude-Code, Codex, Plugin-Development, SKILL.md, Marketplace, 品質閘, Pivot, 商業模式]
status: 已完成
---

# Network AI-Assistant Skill v3 — 從工具到產品：pivot、雙 Agent、與製造紀律

> 🤖 **我用 AI 做了什麼**：把 v2 那個「能分享的 demo 工具」收斂成一個定位清楚的產品——兩個 plugin 合併成單一 `engenius-craft-ai`、補上 **Claude Code + OpenAI Codex 雙 agent** 支援、加了一道「新功能進來前先驗證真 API」的**品質閘**，還把整套裝法做成一鍵精靈。
> ⏱ **沒有 AI 的話**：我不會自己去 probe 100+ 個上游 API、寫一個 doc↔API drift 驗證器、搞跨 agent 的可移植性、做 pre-commit 把關、或把同一份 skill 打包成兩個 agent 的 marketplace。這一輪純工程外包大概要一個月，而且「該不該收斂範圍、護城河在哪」這種判斷根本外包不出去。
> ✅ **最終成果**：v2 的 demo 工具 → 一個團隊可正式試用的產品：單一 plugin、雙 agent、品質閘擋住文件漂移、一鍵安裝精靈、多把 key 秒切。順手把商業模式、護城河、發布架構都想清楚了。

> 接續 [Network AI-Assistant Skill v2](../network-ai-assistant-skill-v2/devlog.html) — v2 把工具變成「可分享、可切帳號、可雙平台部署」；v3 是把它從**工具**變成**有定位、能規模化、品質不會隨成長而崩**的產品。

---

## 一、為什麼要做這件事

v2 之後，工具「能用、能分享」了，但很快撞到四件不是「再加功能」能解決的事：

1. **同事現場 demo 卡住了。** 使用者問了一個很廣的問題（沒指定是哪個 org / network），AI 為了「周全」開始對上百個 org 逐一掃描，幾分鐘過去畫不出 dashboard。這不是效能問題——是 **AI 沒有先收斂範圍**，而且當下的 persona 規範根本沒被遵守。
2. **API 文件在騙人。** 我手動跑了 22 個高風險 API，抓到 **11 個「文件說的」跟「實際回的」對不上**。最誇張的是用量統計：文件聲稱 summary 等於明細加總，實測差了 **37 到 880 倍**。一個建立在錯文件上的 AI，講出來的話也是錯的。
3. **兩包要裝兩次很煩。** 之前是兩個 plugin（資料 skills + dashboard-builder），團隊試用要裝兩包、各自更新，光解釋就卡住。
4. **客群其實同時用兩種 AI。** 主要使用者不只用 Claude Code，也用 OpenAI Codex。一套東西只能跑一個 agent，等於放掉一半的人。

所以 v3 的主軸不是「加 feature」，是**把工具收斂成產品**：定位清楚（「AI 網管顧問」）、規模化（雙 agent + 品質閘）、不會越長越爛。

---

## 二、技術選型 / 關鍵決策

這一輪的「選型」幾乎都是**決策**，不是選工具：

| 決策 | 選的理由 |
|---|---|
| **兩個 plugin 合併成一個 `engenius-craft-ai`** | 團隊裝一包就好；dashboard-builder 變成並列的一個 skill。安裝心智成本砍半。 |
| **雙 agent 不做「跨工具安裝器」、也不改 MCP** | 實測發現 Codex 用的是**同一種 `SKILL.md` 格式**，而我們的 skill 因為把 transport 自包在 `_shared/`、不依賴 harness，所以**直接可移植**。結論：一份 source、兩個薄打包入口，不是維護兩套。 |
| **自寫 `validate-skill-docs.py`（doc↔API drift 驗證）** | 與其相信文件，不如每次拿真 API 回應對照。配 pre-commit gate，文件漂移直接擋 commit。 |
| **判斷留散文、水管固化成積木** | 一度想把整條 pipeline 寫死成腳本，但那會殺掉 AI 的判斷力。最後分層：「要不要畫、畫什麼」留在 persona 散文；「解 ID、並行撈、自包 canvas」固化成可重用積木。 |
| **named API profiles + `use-key.py`** | 多把 key（staging / prod / 各自的）一行指令秒切，不用每次手改設定檔。 |

> **選型原則（延續 v2）**：能用 Python / shell / 既有標準做的就不引框架；能讓 AI 自己判斷的就不要寫死。差別是 v3 多了一條——**「能驗證的就一定要驗證」**：API 不信文件信實測，文件不信記憶信 drift gate。

---

## 三、外部服務與金鑰

| 服務 / 模型 | 用途 | 類型 | 備註 |
|---|---|---|---|
| EnGenius Cloud API | skill 撈資料 / 診斷 / 讀設定 | API | staging + prod，api-key header |
| Claude（Claude Code） | 主要 agent runtime | AI Model | 第一種裝法 |
| gpt-5.5（OpenAI Codex CLI） | 第二種 agent runtime | AI Model | 雙軌支援、語氣略不同 |
| `API_KEY` 等 | 存在 `~/.claude/engenius_env.json`（named profiles）| 金鑰 | 兩個 agent 共用，不進 repo |
| Vercel | 主站 + 文件自動部署 | Token | push 即部署 |

> ⚠️ 一個學到的事：金鑰**設一次、兩個 agent 共用**（同一份 `engenius_env.json`），但「沒設好」時要給**會引導的錯誤訊息**，而不是丟一句 `API_KEY required` 讓使用者卡住。

---

## 四、系統架構

核心觀念：**一份來源，兩個外包裝。**

```
                   api-skills/  ← 唯一正本（skills + _shared transport + 4 份規範文件）
                       │
        ┌──────────────┴──────────────┐
   Claude Code plugin            Codex marketplace
   (.claude-plugin/)             (codex-marketplace/，腳本產生)
        │                              │
   /plugin install              codex plugin marketplace add
        └──────────────┬──────────────┘
                  同一套 SKILL.md + persona/design/house-rules
                  ＋ 品質閘（PROBE → AUTHOR → VALIDATE）守著入口
```

- **skill 本體只寫一次**；兩個 agent 的安裝包都是從同一份自動產生（Codex 那包由 `build-codex-marketplace.py` 生，還掛 pre-commit 防它過期）。
- **4 份規範文件**（persona 怎麼講話 / design 視覺 / house-rules 平台事實 / memory）是讓 AI「像顧問而不是 API 包裝」的關鍵。
- **品質閘**守在「新 skill / API 進來」的入口：先 probe 真 API、過 drift 檢查，才准寫進文件。

---

## 五、推進方式

這一輪跟 AI 的協作，幾個有效的模式：

- **用「壓力測試的失敗」當起點**：同事 demo 卡住那次，我沒有只修表面（「並行撈快一點」），而是跟 AI 一起把它拆到根——發現真正的問題在上游（範圍沒收斂）。**把失敗當診斷樣本，比直接要解法更有用。**
- **先讓 AI 探真實，再寫文件**：每加一個 API / skill，固定先跑 probe 看真實回應，再寫文件。AI 很會「憑 spec 想像」，這一步是強迫它面對現實。
- **把判斷說清楚、讓 AI 固化水管**：我負責「該收斂範圍嗎、護城河在哪、license 怎麼設」這類判斷；AI 負責把「解 ID、並行、自包、驗證」這些重複的機制做成積木。
- **不急著動手、先確認意圖**：很多次我問一個方向（例如「要刪舊文件嗎」），AI 先盤點 + 分級 + 標出風險，等我核可才動。**破壞性操作一律先給清單。**

---

## 六、踩到的坑，讓我更懂的事

### 坑一：「並行不夠快」是表象，根因是範圍沒收斂
上百個 org × 模糊問題 = AI 對全部 fan-out，再怎麼並行都是在洪水裡並行。
> 帶走的原則：**先收斂、再並行**。收斂範圍（先問是哪個 org）在並行的上游，缺一不可。

### 坑二：文件憑想像寫，AI 就跟著錯
11 個 doc-drift bug，每個都是同一模式：文件照 spec / 印象寫，AI 看到的真實 API 是另一回事。
> 帶走的原則：**API 不信文件信實測**。後來把這條變成結構性的——pre-commit drift gate，漂移直接擋 commit，不靠人記得。

### 坑三：同一份 persona，Codex 的語氣卻飄掉
Codex 讀 persona 時只讀了前 220 行（檔案 378 行），剛好把「要主動給建議」那段切掉；加上它底層是 gpt-5.5、不是 Claude。
> 帶走的原則：**內容要對「只被讀一半」有抵抗力**——關鍵規則前置、或放進 always-on 的指令層。跨模型的語氣差異無法歸零，但可以縮小。

### 坑四：以為「做成 skill 就沒門檻」
一度焦慮：API 是公開的、架構攤在 repo、任何人都能照做、甚至套到別家平台。
> 帶走的原則：**護城河不在程式碼**。程式碼/架構可複製；真正不可複製的是**廠商關係（讓 RD 為你開 API）、通路、逐平台累積的驗證知識、客戶資料**。會打 API 是門檻，不是護城河。

---

## 七、最終樣貌

從 v2 的「能分享的 demo 工具」變成「團隊可正式試用的產品」：

- **單一 plugin** `engenius-craft-ai`，一行 `/plugin install` 裝齊全部能力
- **雙 agent**：同一套 skill 跑在 Claude Code 與 OpenAI Codex（實測都能撈資料、跨 skill 串接、動態生成 dashboard）
- **一鍵安裝精靈** `setup.py`：檢查套件 → 引導輸入 + 驗證 API key → 選 agent 安裝
- **品質閘**：新 API / skill 一律 PROBE → AUTHOR → VALIDATE，drift 擋 commit
- **本機 Dashboard 目錄頁**：每張動態生成的 dashboard 自動列出（即時預覽 / 搜尋 / 刪除 / 已部署徽章）
- **多把 key 秒切** + 設不好時會引導你的錯誤訊息
- 商業模式、護城河、發布架構（含「不刪歷史、用時間軸整合」這類決策）都想清楚並寫成內部 one-pager

---

## 八、如果繼續往下

- **團隊試用 rollout**：加 collaborator + 決定共用 key 還是各自一把，就能正式開測。
- **MSP 跨廠商**：架構「可移植到任何廠商」這件事，在「MSP 要一個看穿混合機隊（Meraki + EnGenius + …）的 AI 顧問」情境下，會從弱點變成唯一賣點。
- **動手改設定 + 記憶**：目前停在「讀 / 診斷 / 提案」；下一步是讓 AI 真的動手改 config（風險分級 + 人核可）並同步記一筆 audit log。
- **產品名定案 + 對外**：rotate 歷史裡的舊 key、prod 環境填好、名稱拍板，才談公開。

---

## Takeaway

**這個案例展示了什麼**：把一個「能動的工具」升級成「能規模化、品質不掉的產品」，靠的不是狂加功能，而是三件事——**收斂定位**（一個 plugin、一個角色）、**建立紀律**（新東西進來前先驗證真實，而不是信文件/記憶）、**想清楚護城河**（程式碼會被抄，關係與驗證知識不會）。AI 在這裡最有價值的不是寫 code，是當「把失敗拆到根、把判斷固化成機制」的協作者。

**可移植性**：
- ✅ 直接可複製：「先 probe 真實、再寫文件 + 用 gate 擋漂移」這套紀律，任何接外部 API 的專案都適用。
- ✅ 直接可複製：「判斷留散文、水管固化成積木」——凡是想保留 AI 判斷力、又要穩定產出的工具都受用。
- ⚠️ 情境限定：雙 agent 之所以幾乎零成本，是因為我們的 skill 早就把 transport 自包、不依賴 harness。如果你的工具深綁某個 agent 的內建能力，移植成本會高很多。

**如果你也要做，先問 AI 什麼**：
1. 「我這個專案接了哪些外部 API？幫我對每個 op 跑一次真實呼叫，把『文件說的』跟『實際回的』列出差異。」（找出你的 doc-drift）
2. 「假設這個工具會持續長大，幫我設計一道『新功能 / 新 API 進來前必須通過』的驗證關卡，並說明怎麼掛進 pre-commit。」（建立製造紀律）

_開發日期：2026-05-20 ~ 2026-06-01（接續 v2，本階段約 200+ commits）_
