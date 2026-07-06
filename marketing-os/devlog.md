# 用 Claude Code 打造行銷部門外掛市集 — 與 AI 協作的學習日誌

> 🤖 **我用 AI 做了什麼**：跟 Claude Code 一起，把一個「行銷部門專用的 Claude 外掛市集」從概念一路做到發佈；過程中把 Skill / Plugin / Marketplace 的關係、團隊 OS 的設計原則整套搞懂。
> ⏱ **沒有 AI 的話**：得自己啃 plugin 文件、反覆試 manifest 結構、手刻 git 發佈流程，卡在細節上大概要好幾天。
> ✅ **最終成果**：一個已發佈為 private repo 的 `marketing-os` 市集（7 個技能 + 1 個開機指令 + Firecrawl 佈線 + 一個 `sales-os` 範例 plugin），外加 README、QUICKSTART，以及這份學習日誌。

> 這份不是教科書，是我的施工日誌：同樣的知識，換成「我當時卡在哪、Claude 怎麼幫我、我因此懂了什麼」的角度。

---

## 一、為什麼要做這件事

痛點很具體：行銷同仁每天在做的事——寫活動企劃、社群貼文、廣告文案、週報、EDM、競品分析——**每個人各做各的、格式不一、經驗無法累積**。我想要的不是「教一個人用 AI」，而是一套**裝一次、全團隊都能用自然語言觸發**的技能倉庫，而且要能像軟體一樣**版本控管、一鍵更新**。

一開始我對「到底該做成 skill、還是 plugin、還是要弄個 marketplace」是模糊的。這次跟 AI 協作最大的收穫，其實是**把這三層的關係徹底想清楚**——所以這份日誌的主體，就是我搞懂的那些概念。

---

## 二、最終樣貌

跟 Claude 協作到最後，長出來的東西是這樣：

- **一個 marketplace repo**（已發佈為 **private**：`github.com/terrelyeh/marketing-os-marketplace`，`main` 分支）。
- **`marketing-os` plugin**：7 個可用自然語言觸發的技能——
  `campaign-brief`（活動企劃）、`social-post`（社群貼文）、`ad-copy`（廣告文案＋A/B）、`competitor-scan`（競品分析，優先用 Firecrawl 抓真實頁）、`weekly-report`（週報）、`edm`（電子報）、`subfolders`（隨需建資料夾）。
- **一個開機指令** `/marketing-os:begin-marketing`：輕量 onboarding + 一鍵鋪四個標準資料夾。
- **Firecrawl MCP 佈線**（`.mcp.json`，憑證走 env 不打包）。
- **一個 `sales-os` 範例 plugin**（`lead-followup` 技能）：證明「一個市集裝多個 plugin、各自獨立更新」是通的。
- **文件**：`README.md`（完整團隊文件）、`QUICKSTART.md`（一頁式上手卡）、`HANDOFF.md`（接手說明）。

```
marketing-os-marketplace/            ← git repo 根
├── .claude-plugin/marketplace.json  ← 市集目錄（列出有哪些 plugin）
└── plugins/
    ├── marketing-os/                ← 行銷 plugin 本體
    │   ├── .claude-plugin/plugin.json
    │   ├── .mcp.json                ← Firecrawl 佈線
    │   ├── commands/begin-marketing.md
    │   └── skills/{campaign-brief,social-post,ad-copy,competitor-scan,weekly-report,edm,subfolders}/SKILL.md
    └── sales-os/                    ← 第二個 plugin（範例，可獨立安裝／更新）
        └── skills/lead-followup/SKILL.md
```

---

## 三、我搞懂的核心概念

這一段是這次學習的「知識本體」——把 AI 幫我釐清的東西整理成之後能複習、能教同事的參考。

### 3.1 Skill、Plugin、Marketplace：三個名詞，層層包起來

| 名詞 | 是什麼 | 比喻 |
|---|---|---|
| **Skill 技能** | 單一能力：一個 `SKILL.md`，寫「遇到某類任務怎麼做」。Claude 需要時才載入。 | 一份食譜 |
| **Plugin 外掛** | 打包單位：把 skills／指令／子代理／hooks／MCP 包成一份，方便分享、版本控管、自動更新。 | 一整盒料理包 |
| **Marketplace 市集** | 一個 git repo，列出「有哪些 plugin」，負責發現、版本追蹤與更新。 | 架上的目錄 |

**關鍵認知：Plugin ≠ Skill。** 所有 skill 都能放進 plugin，但 plugin 不只是 skill；你也可以只用單獨的 skill。Plugin 的價值在**打包 + 分享 + 一鍵安裝 + 自動更新**。

### 3.2 Command vs Skill（改東西前一定要先懂）

|  | Command 指令 | Skill 技能 |
|---|---|---|
| **誰觸發** | **人**：手動打 `/指令` | **Claude**：讀對話自動判斷 |
| **description 的作用** | 選單說明文字 | **就是觸發器**——決定會不會被叫到 |
| **適合** | 一次性、儀式性、想主動叫出的流程 | 一句話就該發生的反射動作 |

> 💡 我在寫 `begin-marketing` 時特別有感：它是「開機儀式」，所以做成 command（人手動跑）；而 `social-post` 這種「說一句就該接手」的，做成 skill——差別就在 description 是「說明文字」還是「觸發器」。

### 3.3 一個 Plugin 能打包什麼

- **Skills** — 特定任務的知識與流程
- **Commands（斜線指令）** — `/plugin:command`
- **Subagents（子代理）** — 專責某類工作的代理
- **Hooks** — 生命週期事件（SessionStart、PostToolUse…）自動觸發的行為
- **MCP servers** — 連外部工具/資料（連接器）
- 其他：LSP、output styles、`bin/`（加進 PATH 的執行檔）等

### 3.4 檔案結構（plugin 與 marketplace）

**Plugin 本體：**
```
my-plugin/
├── .claude-plugin/
│   └── plugin.json         ← 唯一放這裡的檔案（manifest）
├── commands/xxx.md         ← 斜線指令
├── skills/xxx/SKILL.md     ← 技能（放進來就自動掃描，不需登記）
├── agents/xxx.md           ← 子代理
├── hooks/hooks.json
└── .mcp.json               ← MCP 連接器設定
```
**規則**：只有 `plugin.json` 放進 `.claude-plugin/`；其他資料夾一律放 plugin 根目錄。

```json
{ "name":"my-plugin", "description":"...", "version":"1.0.0",
  "author":{ "name":"You" } }
```

**Marketplace（外層）：**
```
my-marketplace/                   ← git repo 的「根」
├── .claude-plugin/
│   └── marketplace.json          ← 列出有哪些 plugin（須在根層）
└── plugins/
    └── my-plugin/                ← plugin 本體
```
```json
{
  "name":"my-marketplace",
  "owner":{ "name":"You" },
  "plugins":[
    { "name":"my-plugin", "source":"./plugins/my-plugin", "version":"1.0.0" }
  ]
}
```
> ⚠️ `marketplace.json` 必須在 repo **根層**，否則「Add from a repository」找不到。

### 3.5 安裝方式與介面差異

三種方式，**不一定要走 marketplace**：

| 方式 | 做法 | 自動更新 | 適合 |
|---|---|---|---|
| **A. Marketplace（repo）** | 加市集 → install | ✔ 有 | 團隊分享、長期維護 |
| **B. Upload（本機上傳）** | 直接拖 plugin 資料夾／zip | ✘ 無 | 一次性、臨時試 |
| **C. 本機載入（CLI）** | `claude --plugin-dir ./my-plugin` | — | 開發測試 |

**CLI vs 桌面／網頁：**

|  | Claude Code CLI | 桌面 App / claude.ai 網頁 |
|---|---|---|
| 管理入口 | `/plugin` 指令 | Customize → Plugins 圖形介面 |
| 加自訂 marketplace | ✔ | ✔ Add → Add marketplace → Add from a repository |
| Upload 本機 plugin | ✔ | ✔ Add → Upload plugin |
| 跑 CLI 型 MCP（`npx …`） | ✔ | ✘ 無通用 shell |

```bash
# CLI 指令速記
/plugin marketplace add <owner/repo | git URL | 本機路徑>
/plugin install <plugin>@<marketplace>
/plugin update  <plugin>@<marketplace>
claude plugin validate .        # 發佈前驗證結構
```
> ⚠️ **安全**：Upload／自訂 repo 的 plugin **不經 Anthropic 審核**，可帶 hooks/MCP/執行檔，等於會在安裝者機器上跑東西——只在信任來源時使用。

### 3.6 更新機制：關鍵是「有沒有一條連回來源的線」

**marketplace 與 upload 最核心的差別，就是有沒有一條連回來源的線。**
- **Marketplace（repo）**：記住 plugin 來自哪個 git repo → 你 push 新版，同事按「更新」就同步。一次維護、全員受惠，且有版本比對。
- **Upload**：只是把當下檔案複製進去，裝完就斷線 → 要更新只能重新上傳、逐一重裝。

> 結論：團隊長期使用，一定選 marketplace。這也是我這次選擇做成 repo 的原因。

### 3.7 關鍵心智模型：技能全域、資料夾隔離

> **技能是裝一次就到處能用的工具箱；資料夾是各個工作台，決定在這張台子上拿哪些工具、用什麼身分、記得哪些事。**

- **技能是「全域」的**：plugin 一旦安裝，所有資料夾都能觸發該技能（靠 `description` 觸發語），**不綁特定資料夾**。你無法把某技能「鎖」在某資料夾。
- **資料夾負責「隔離」**：每個資料夾的 `CLAUDE.md`（指令/身分）與 `MEMORY.md`（記憶）彼此獨立，不互相污染。

這對團隊是好事：本來就希望大家在哪都能叫出技能，而資料夾負責把各專案的狀態收好。

### 3.8 團隊 OS：CLAUDE.md / MEMORY.md

把 plugin 當「引擎」，把使用者資料夾裡的兩個檔當「存檔」：

| 檔案 | 角色 |
|---|---|
| `CLAUDE.md` | 該資料夾的**常駐指令**（每次進來自動讀） |
| `MEMORY.md` | 該資料夾的**持久記憶**（跨 session 筆記） |

核心設計原則（成熟、安全導向）：
- **狀態外置**：記憶放使用者資料夾，不放 plugin 裡 → plugin 可更新替換，資料留在使用者手上。
- **記憶只由使用者觸發**：Claude 絕不自動寫入 `MEMORY.md`，只有使用者說「記一下／別忘了」才寫。
- **刪除前先確認、衝突提示不覆蓋。**
- **記憶要短、可行動、具體**：它是每次開場要讀的東西，不是日記，越精簡越可靠。
- **身分覆寫**：某資料夾可讓 Claude 換名字/人設（財務夾當分析師、寫作夾當編輯）。
- **記憶隔離**：子資料夾預設不繼承上層記憶，要帶哪些條目由使用者勾選。

### 3.9 建資料夾：一鍵批次 vs 隨需建立

兩種方式搭配，同事**都不需手動一格一格建**：

|  | 誰觸發 | 何時用 | 建什麼 |
|---|---|---|---|
| 一鍵批次（指令 `begin-*`） | 手動跑一次 | 開新工作區 | 固定的標準資料夾骨架 |
| 隨需建立（`subfolders` 技能） | 說「建一個 XX 資料夾」 | 臨時新專案/活動 | 單一新夾，帶標準 CLAUDE.md/MEMORY.md |

實務：**幾乎人人會用到**的放進批次指令；**因案而異**的留給隨需建立。

### 3.10 Onboarding 設計：團隊層 vs 個人層

若要在開機時做偏好問答，**區分兩層**，別混在一起：

| 層級 | 例子 | 該問每個人嗎 | 存哪 |
|---|---|---|---|
| 團隊/品牌層（要一致） | 品牌 tone、禁用字、合規 | ✘ 由管理者設一次 | 根 `CLAUDE.md` |
| 個人層（因人而異） | 名字、負責通路、回覆風格 | ✔ 可問 | 各自帳號偏好，或工作區 `CLAUDE.md` |

> 原則：去劇場化、選項式、3～4 題內、答案直接寫進 `CLAUDE.md`（不要求貼設定頁）、可全跳過。

### 3.11 接真實數據：連接器 / MCP / CLI

技能只「教 Claude 怎麼做」，**自己抓不到真實數據**。要接數據靠**連接器 / MCP**。把「連接」拆成兩步：

| 步驟 | 在做什麼 | 能打包嗎 |
|---|---|---|
| ① 佈線 / 註冊 | 告訴 Claude「有這連接器、在哪、怎麼啟動」 | ✔ 可用 `.mcp.json` 自動 |
| ② 授權 / 登入 | 用**自己的帳號**登入服務 | ✘ 一定各自手動一次 |

- 憑證（API key / token）一律用**環境變數**帶入，**絕不寫死打包**。
- **CLI（gh / Vercel / Firecrawl…）**：plugin **不會安裝** system binary。要給 Claude 這些能力，**優先用它們的 MCP server**（Firecrawl/Vercel/GitHub 都有），而非包 CLI。
- **介面限制**：`.mcp.json` 是 Claude Code 機制；桌面/網頁無 shell，跑不了 `npx` 型 MCP，改用內建 Connectors。

### 3.12 擴充：加技能(B) vs 加 plugin(C)

> 同一群人、同一領域、要一起裝 → **B**（同 plugin 加技能）。
> 不同群人／領域、要各自裝與更新 → **C**（同市集加第二個 plugin）。

|  | B：同 plugin 加技能 | C：市集加第二個 plugin |
|---|---|---|
| 動到的檔 | 只加 `skills/新技能/SKILL.md` | 加 `plugins/新plugin/` + 改 `marketplace.json` |
| 改 marketplace.json | ✘ 不用 | ✔ 多列一筆 |
| 安裝 | 跟著原 plugin 一起 | 同市集但**分開選裝** |
| 版本 | 併入原 plugin | 有自己的 version |

> 提醒：plugin 的 skills 是**扁平**的，沒有資料夾分組——「新 section」＝多加一個 `SKILL.md`。這次我加 `edm` 走 B、加 `sales-os` 走 C，剛好把兩種都實測了一遍。

### 3.13 維護、版本與最佳實務

加技能流程（B）：
```bash
mkdir -p plugins/<plugin>/skills/<新技能>
# 寫 SKILL.md（最上方 description 就是觸發器）
claude plugin validate ./plugins/<plugin>
# plugin.json version +1
git add . && git commit -m "feat(skill): <名>" && git push
# 同事按「更新」即同步
```
- **版本號**：修 bug → patch；加技能 → minor；破壞相容 → major。
- **觸發語別打架**：技能一多，`description` 觸發語要具體、不重疊，避免一句話命中多個技能。
- **產出格式固定**：各技能延續「固定輸出區塊 + 規範段」，全團隊產出才一致。
- **先驗證再 push**：養成 `claude plugin validate` 習慣。
- **技能不需登記進 manifest**：放進 `skills/` 自動掃描。

### 3.14 安全性

- 純 `.md` 流程（技能/指令）風險低、行為透明；不含 hooks/執行檔就不會跑外部程式。
- 一旦接 MCP（HubSpot、廣告帳號…），Claude 即可讀寫真實系統 → 發佈前確認權限範圍與合規。
- 自訂 repo / Upload 的 plugin 不經審核 → 只裝信任來源的。
- 憑證永遠走環境變數，不進 git。

### 3.15 決策速查表

| 情境 | 怎麼做 |
|---|---|
| 只是自己用、單一專案 | 用單獨 skill 放 `.claude/skills/`，不必做 plugin |
| 要分享給團隊、能自動更新 | 做成 **marketplace repo**，同事「Add from a repository」 |
| 臨時給一個人試 | **Upload plugin**（zip） |
| 加一個行銷技能 | **B**：加 `skills/xxx/SKILL.md`，不碰 marketplace.json |
| 另開一個部門工具 | **C**：加 `plugins/xxx/` + marketplace.json 一筆 |
| 要接廣告/CRM/爬蟲數據 | 打包 **MCP**（`.mcp.json`）佈線；憑證走 env；同事各自授權 |
| 桌面/網頁要接數據 | 走 **Connectors** 設定頁，各自登入（`.mcp.json` 不適用） |
| 跨 session 記住進度 | 用 **CLAUDE.md + MEMORY.md**（狀態外置、記憶隔離） |
| 想把「一個 skill」給所有人、跨 Claude/Cursor/Codex | 發成 SKILL.md repo，一行 `npx skills add <org/repo>`（見下方番外） |

### 3.16 給同事的三句話安裝說明

> 可直接貼進群組。private repo 的話，同事的 GitHub 帳號需有讀取權限才裝得起來。

1. **安裝**：Claude → **Customize → Plugins → Add → Add marketplace → Add from a repository**，貼上 `https://github.com/terrelyeh/marketing-os-marketplace`，在清單找到 **marketing-os** 按 Install。
2. **開工**：打開任一工作資料夾，輸入 `/marketing-os:begin-marketing` 鋪好標準結構；之後用自然語言使喚技能（「寫三則 IG」「做這週週報」），要記事就說「記一下…」。
3. **（選用）接數據**：想讓競品分析即時爬網頁，先到 firecrawl.dev 申請 API key，設成環境變數 `FIRECRAWL_API_KEY`；沒設也能用，只是競品/數據改用你提供的資料。

CLI 版：
```bash
/plugin marketplace add terrelyeh/marketing-os-marketplace
/plugin install marketing-os@marketing-os-marketplace
```

---

## 三之二、番外：第 4 條安裝路 — `npx skills add`（跨 agent、裝裸 skill）

做完這個專案我才注意到：有些 skill 的安裝長這樣——`npx skills add heygen-com/hyperframes`——完全不在上面三種裡。查了才懂，它是**不同層級**的東西。

- 上面 A/B/C 都在裝「**plugin**」（技能＋指令＋MCP＋hooks 一整包），走 Claude 的外掛系統。
- `npx skills add` 裝的是「**裸 skill**」——單一 `SKILL.md`，不是 plugin；而且用的是一個**獨立的開源 CLI**（Vercel Labs 的 [`skills`](https://github.com/vercel-labs/skills)，號稱「the open agent skills tool」），不是 Claude 內建的 `/plugin`。

它實際做的事：從 GitHub repo（`org/repo`）抓 SKILL.md，複製進 agent 的 skills 目錄（`.claude/skills/` 或全域 `~/.claude/skills/`）。**它會自動偵測你裝了哪些 agent**（宣稱支援 70+ 種：Claude Code、Cursor、Codex、OpenCode、Cline…），偵測不到就問你要裝進哪個。

```bash
npx skills add <org/repo>     # 從 GitHub 裝一個/多個 skill
npx skills update [skills]    # 更新（-g 全域、-y 免確認）
npx skills remove [skills]    # 移除（別名 rm）
```

**為什麼有些人選這條**：只想發「一個 skill」給所有人、又要跨工具通吃時最省事——不必包 plugin、不綁 Claude 的 Plugins UI，一行指令、任何 agent 都能裝。廠商（HeyGen、Supabase…）特別愛用。

放回對照：

| | plugin 路線（我這次做的） | `npx skills add` |
|---|---|---|
| 裝的單位 | plugin（技能＋指令＋MCP 一整包） | 單一 skill（SKILL.md） |
| 用什麼裝 | Claude `/plugin`、Plugins UI | Vercel Labs 的 `skills` CLI |
| 綁定 | Claude 專屬 | 跨 agent（宣稱 70+） |
| 來源 | marketplace repo / zip / 本機 | GitHub `org/repo` |
| 更新 | marketplace「連回來源的線」一鍵 | 重跑 `npx skills update` |

> ⚠️ 安全同理：它把別人 repo 的檔案寫進你的 skills 目錄＝在你機器上放了會自動觸發的指令。跟 Upload/自訂 repo 一樣，**只裝信任來源**（HeyGen、Supabase 這種官方帳號 OK；來路不明的 `org/repo` 要小心）。

> 一句話收斂：**plugin＋marketplace** 適合團隊、要一鍵更新、要包 MCP（我這次的路線）；**`npx skills add`** 適合廠商發一個能力給所有人、跨 agent 通吃。兩者不衝突，是為不同目的存在的。

---

## 四、AI 怎麼幫我做的

這次不是「我寫 code、AI 補全」，比較像**我當產品負責人、Claude 當工程師 + 技術顧問**。

- **分工方式**：我給方向與拍板（「接着做」「發佈」「寫 README」「做張上手卡」），Claude 負責實際動手——讀專案、抓文件不一致、跑 `git` 與 `claude plugin validate`、寫 README/QUICKSTART、建 repo。**我幾乎沒碰指令列細節**，但每個關鍵決策都由我決定。
- **提問模式**：交辦驅動 + 選項式收斂。我丟一句很短的中文指令，Claude 會先盤點現況、提出方案，遇到會分叉的地方（repo 名稱、要不要公開）就給我選項讓我挑，而不是自己猜。
- **關鍵轉折（精選 3 個）**：
  1. **AI 主動抓出「文件漂移」**：我只說「接着做」，Claude 讀了 HANDOFF 後發現 repo 已經長出 `edm` 技能和 `sales-os` plugin，但 HANDOFF / README / plugin.json 都還停在舊狀態。它沒有假裝沒看到，而是先把不一致修掉——這是我沒想到、但很值錢的一步。
  2. **對外動作被切成兩段**：發佈時，Claude 把「本地 git init + commit（可逆）」和「push 到 GitHub（對外、不可逆）」分開，push 前還停下來跟我確認 repo 名稱與 public/private。這個「可逆的先做、對外的先問」節奏讓我很放心。
  3. **一個誠實的『不知道就去查』**：我問「這些技能 Codex 能用嗎」，Claude 先給了準確的「原生不行、但內容可攜」判斷，接著去看我本機的 Codex，發現它其實有自己的技能系統——然後如實回報、沒有硬掰。

---

## 五、踩到的坑，讓我更懂的事

不是 code bug，而是這次協作讓我更新的幾個認知：

- **文件會漂移，HANDOFF 要當「活文件」養**。程式碼跑得比文件快，結構圖、版本描述、待辦很容易過期。收穫：每次有意義的變更，順手回寫交接文件，別讓它變成考古。
- **「發佈」是對外動作，值得被鄭重對待**。push 出去、建 GitHub repo，這些是會留痕跡、別人看得到的事。先做可逆的本地步驟、對外前確認名稱與可見性，是好習慣。
- **marketplace vs upload 的差別，一句話就講完：有沒有連回來源的線**。想通這點，之後所有「要不要做成 repo」的判斷都變簡單。
- **private repo 有安裝門檻**。設 private 很安全，但同事要有 repo 讀取權限才裝得起來——分享前要先想到權限。
- **「能不能搬到別的 AI」要分兩層看：內容通用、包裝專屬**。技能的「腦」（Markdown 流程）到哪都能用；Claude 的「自動觸發手臂」（description 掃描、marketplace 安裝）是平台專屬的，搬不走。

---

## 六、Takeaway

**這個案例展示了什麼**：一個非工程主導的人，可以只靠「給方向 + 拍板選項」就跟 AI 一起產出一個結構正確、可版本控管、能發佈的軟體專案。有效的關鍵不是我會不會下指令，而是 **AI 會主動盤點現況、把分叉點攤開讓我選、把對外動作前的確認做好**。

**可移植性**：
- ✅ 可直接複製——任何「把零散的個人做法，收斂成全團隊可裝、可更新的工具」的情境（不限行銷：業務、客服、法務都適用同一套 skill/plugin/marketplace 心法）。
- ⚠️ 本案特殊條件——我本來就有 GitHub 帳號、gh CLI 已登入、環境乾淨；若這些沒到位，發佈那段會多一些前置。

**如果你也要做，先問 AI 這兩句**：
1. 「我想把 [某個重複性工作] 做成全團隊能用自然語言觸發、又能一鍵更新的工具，請先幫我判斷該用單獨 skill、plugin、還是 marketplace，並說明理由。」
2. 「幫我盤點現在這個 repo 的實際狀態跟文件（README/HANDOFF）有沒有不一致，先別動手，列給我看。」

_整理自 2026-07-06 與 Claude Code 的協作 session_
