# Amazon Banana Studio — 開發紀錄

> 🤖 **我用 AI 做了什麼**：從零打造一款 AI 電商產品圖生成工具，涵蓋 8 種 Amazon 版位、4 階段 AI pipeline、前端費用預估、按需合規檢查
> ⏱ **沒有 AI 的話**：理解 Amazon 8 種版位規範 + 前後端架構 + Gemini 多模態串接 + UI 設計，至少需要 2-3 週全職開發
> ✅ **最終成果**：已上線工具（amazon-bs.vercel.app），含使用指南、API 費用說明頁、產業最佳實踐文件，已交付電商部門試用

> 專為 Amazon 賣家打造的 AI 產品攝影工具。上傳去背產品圖 → 選版位 → AI 自動生成符合 Amazon 規範的商業攝影級圖片。

## 需求背景

Amazon 賣家上架一個 SKU，需要準備大量不同規格的圖片：主圖要純白背景且產品佔比 85%+、副圖要情境照、A+ Content 要 970×600 橫幅、旗艦店 Banner 要 3000×600 寬幅⋯⋯每種版位都有不同的尺寸要求和合規規範。

傳統做法是攝影師拍攝 + 設計師修圖，每個 SKU 成本動輒數千元。核心挑戰不只是「接一個 AI 生圖 API」，而是把 Amazon 的圖片規範（8 種版位各自的規則）內嵌到 AI 的行為中，讓生出來的圖「開箱即合規」。

## 技術選型

| 層級 | 技術 | 選擇理由 |
|---|---|---|
| 前端框架 | React 19 + TypeScript 5.8 | 生態系成熟，AI 生成的程式碼品質最穩定 |
| 建置工具 | Vite 6 | 快速 HMR，Vercel 原生支援 |
| 樣式 | Tailwind CSS 4 | Utility-first，AI 生成 UI 時不需維護獨立 CSS |
| 動畫 | Motion 12 (Framer Motion) | `AnimatePresence` 處理條件渲染的進出場 |
| 後端 | Vercel Serverless Functions | 零設定部署，API Key 安全隔離在 server-side |
| 儲存 | IndexedDB（瀏覽器端） | 圖片不上傳第三方，保護商業機密，FIFO 50 張上限 |
| AI SDK | `@google/genai` | Server-side only，輕量，支援 structured output |
| 部署 | Vercel | `npx vercel --prod` 一鍵部署，自動 CI/CD |

## 外部服務與金鑰

| 服務 / 模型 | 用途 | 類型 | 備註 |
|---|---|---|---|
| `gemini-3.1-pro-preview` | 提示詞優化、合規檢測、產品分析 | AI Model | 多模態理解，看懂圖片+做邏輯推理 |
| `gemini-3.1-flash-image-preview` | 產品圖生成 | AI Model | 支援 14 種 aspect ratio + 1K/2K 解析度 |
| `GEMINI_API_KEY` | 所有 Gemini API 呼叫 | API Key | 存放於 Vercel env / `.env.local` |
| Vercel | 部署 + Serverless runtime | Platform | GitHub 連動，push 即自動部署 |
| Google AI Studio | API 管理 + Billing + spending cap | Platform | Monthly cap 設定在 AI Studio Settings |

> ⚠️ `GEMINI_API_KEY` 必須來自**已啟用 Billing** 的 Google Cloud 專案。圖片生成模型（Flash Image）沒有免費額度，純付費。Google AI Studio 有 Tier 制 spending cap（Tier 1 = $250/月），與 Cloud Billing Budgets 是獨立的兩套系統。

## 系統架構

```
                          ┌─────────────────────────────────┐
                          │    Vercel Serverless (api/)      │
                          │                                  │
  React 19 (SPA)          │  analyze-product ──→ Gemini Pro  │
  ┌─────────────┐         │  optimize-prompt ──→ Gemini Pro  │
  │ App.tsx     │ ──API──→│  generate-image  ──→ Gemini Flash│──→ base64 images
  │ (~2000 行)  │         │  check-compliance──→ Gemini Pro  │
  └──────┬──────┘         └─────────────────────────────────┘
         │
         ▼
    IndexedDB (瀏覽器)
    ├── 生成圖片 (FIFO 50 張)
    └── metadata (prompt, 版位, 時間)
```

- **前端不持有 API Key**：所有 AI 呼叫都經過 `/api/*` serverless function
- **IndexedDB 而非雲端儲存**：產品圖是商業機密，不上傳第三方
- **4 個 API endpoint 各司其職**：分析 → 優化 → 生成 → 檢查，pipeline 式串接

## 核心流程

### 生圖主流程

```
使用者上傳產品圖
    │
    ▼ (背景自動)
analyze-product → 辨識品類 + 推薦描述
    │
    ▼ (使用者點「生成」)
optimize-prompt → 使用者描述 + 版位規範 + 參考圖 → 專業 prompt
    │              輸入：產品圖 + 參考圖 + 文字描述 + 版位 + 排版意圖
    │              輸出：optimizedPrompt (英文) + aspectRatio
    ▼
generate-image × N 張 → Promise.allSettled + 自動重試
    │              輸入：產品圖 + 參考圖 + optimizedPrompt + ratio + resolution
    │              輸出：base64 圖片陣列
    ▼
存入 IndexedDB → 顯示預估費用 → 等待使用者觸發合規檢查
```

### 關鍵設計：Amazon 規範嵌入點

版位規範不是在「合規檢查」時才介入，而是在 **Step 1 的 prompt 優化** 就內嵌。`optimize-prompt.ts` 的 `PLACEMENT_RULES` 包含 8 種版位的完整規則（純白背景、產品佔比、禁止文字⋯），生成的 prompt 會自動帶入合規約束 → 圖片「天生合規」。合規檢查是二次驗證，不是唯一防線。

## 技術重點

1. **Aspect Ratio 安全閥**：Gemini 只接受 14 種特定比例（1:1, 3:2, 4:3, 16:9, 21:9 等）。`normalizeAspectRatio()` 將任意比例映射到最近的有效值，避免 API 回 400。新增版位時必須確認比例是否在 `VALID_RATIOS` 中。

2. **前端費用預估**：圖片生成成本是確定性的（1K=$0.067/張, 2K=$0.101/張），用前端公式 `固定開銷 $0.036 + 張數 × 每張成本` 即可精準估算。不走後端，是因為 `usage_metadata` 裡的 thinking tokens 不可預測，反而不如固定公式準確。

3. **合規檢查從自動改為手動**：初版每次生圖後自動跑合規（多花一次 Gemini Pro 呼叫），但主圖和旗艦店類的合規需求差異大，且 prompt 優化已內嵌規範，多數圖片本來就合規。改為手動觸發（hover 圖片 → 點合規按鈕），降低 API 成本。

4. **錯誤碼轉發**：Gemini API 回 429 spending cap 時，後端原本統一包成 500，前端無法顯示正確提示。修正後 4 個 API endpoint 都轉發原始 HTTP status（429/403/503），前端 `friendlyError()` 根據 status + message 關鍵字匹配中文提示。

5. **多產品模式的控制流**：`multiProductMode` 是獨立開關（由上往下的控制流），開啟 → 版位鎖定旗艦店 + 展開配角上傳區。避免了「下方操作觸發上方 UI 變化」的 UX 反模式。

## 實作步驟

### 步驟一：4 階段 AI Pipeline

每個階段是獨立的 Vercel Serverless Function（`api/` 目錄），各自負責一種 AI 任務：

```typescript
// api/optimize-prompt.ts — 核心：版位規範嵌入
const PLACEMENT_RULES: Record<string, string> = {
  '主圖 (Main Image)': `
    Must use PURE WHITE background (RGB 255, 255, 255).
    Product must fill ≥85% of the frame.
    NO text, NO logos, NO watermarks, NO props...
  `,
  'A+ 橫幅 (970×600)': `
    A+ Content Module 06. Actual 970×600px (≈16:10).
    Generate at 16:9, crop height afterward...
  `,
  // ... 8 種版位
};
```

### 步驟二：Aspect Ratio 安全閥

```typescript
// api/generate-image.ts
const VALID_RATIOS = new Set([
  '1:1','1:4','1:8','2:3','3:2','3:4','4:1','4:3',
  '4:5','5:4','8:1','9:16','16:9','21:9'
]);

function normalizeAspectRatio(ratio: string): string {
  if (VALID_RATIOS.has(ratio)) return ratio;
  const [w, h] = ratio.split(':').map(Number);
  const target = w / h;
  // 找最近的有效比例...
  return closestRatio;
}
```

### 步驟三：前端費用預估

```typescript
// src/App.tsx
const estimateCost = (imgCount: number, size: string): number => {
  const IMAGE_COST: Record<string, number> = { '1K': 0.067, '2K': 0.101 };
  const OVERHEAD_FIXED = 0.036; // prompt optimization (Pro model)
  return OVERHEAD_FIXED + imgCount * (IMAGE_COST[size] || 0.067);
};
```

### 步驟四：錯誤碼轉發 + 中文化

```typescript
// api/*.ts — 所有 4 個 endpoint 統一處理
catch (error: any) {
  const httpCode = error?.status || error?.httpStatusCode || (() => {
    try { return JSON.parse(error?.message)?.error?.code; }
    catch { return undefined; }
  })();
  const status = [400, 401, 403, 429, 503].includes(httpCode) ? httpCode : 500;
  return res.status(status).json({ error: error.message });
}

// src/lib/api.ts — 前端友善提示
function friendlyError(status: number, data: any): string {
  if (code === 429 && msg.includes('spending cap'))
    return 'API 月度花費已達上限。請至 Google AI Studio → Settings 調整。';
  // ...
}
```

## 踩坑紀錄

### Bug 1：Gemini 收緊 Aspect Ratio 驗證

- **症狀**：圖片生成突然全部失敗，回 400 錯誤。程式碼沒改過。
- **根本原因**：Gemini API 在某次更新後收緊了 aspect ratio 驗證，只接受 14 種特定比例。原本能通過的 `5:1`（旗艦店 Banner）被拒絕。
- **解法**：加入 `normalizeAspectRatio()` 安全閥，將非標準比例映射到最近的有效值（`5:1` → `4:1`）。
- **教訓**：依賴外部 API 時，即使你的程式碼沒改，API 的行為可能改變。加一層 normalization 防禦層。

### Bug 2：Spending cap 錯誤被吞掉

- **症狀**：使用者點「合規」按鈕，顯示「AI 服務暫時異常」，但實際原因是 API 月度花費達上限。
- **根本原因**：後端 catch block 統一回 `status(500)`，原始的 429 錯誤碼被丟失。前端的 `friendlyError()` 看到 500 就顯示通用錯誤訊息。
- **解法**：4 個 API endpoint 都改為轉發原始 HTTP status，前端根據 `status + message` 匹配精確的中文提示。
- **教訓**：錯誤處理要保留原始資訊，不能在中間層統一吞掉。

### Bug 3：friendlyError() 的 .includes() crash

- **症狀**：點合規按鈕，console 報 `o.includes is not a function`（minified 錯誤）。
- **根本原因**：`friendlyError()` 的 fallback chain `parsed?.error?.message || parsed?.error || ...`，當 `parsed.error` 是 object（而不是 string）時，`msg` 拿到 object，呼叫 `.includes()` 就 crash。
- **解法**：加入明確的類型檢查 `typeof rawMsg === 'string' ? rawMsg : ''`。
- **教訓**：TypeScript 的 `const msg: string = ...` 只是型別標註，不是 runtime guarantee。API 回傳的 error 結構不可預期，要防禦性處理。

### Bug 4：Google AI Studio vs Cloud Billing 的花費上限

- **症狀**：用 `gcloud billing budgets list` 查詢，回 0 筆預算。但使用者確實設了月度上限。
- **根本原因**：Google AI Studio 的 Monthly spending cap 和 Google Cloud Billing Budgets 是**兩套獨立系統**。用 `gcloud` 查的是後者，使用者設的是前者。
- **解法**：引導使用者到 AI Studio → Settings 查看和調整 cap。同時把前端錯誤提示從「至 Cloud Console」改為「至 AI Studio → Settings」。
- **教訓**：Google 的 AI 服務有兩個入口（AI Studio vs Vertex AI），各自有獨立的帳單管理。搞混會找不到設定。

## 延伸課題 / 未來可能

- **使用者管理 / Access Control**：目前是單一 API Key 共用，多人使用會撞 spending cap
- **圖片精修（Image Refinement）**：基於已生成的圖片微調（Gemini 目前沒有 seed/deterministic generation，無法精確控制）
- **更多產業的 Best Practice**：目前只有網通設備，可擴展到食品、美妝、3C 等
- **批次生成模式**：一次上傳多個 SKU，自動跑完所有版位

## Takeaway

### 這個案例展示了什麼

**「情境驅動開發」+ AI 全棧實作**的協作模式：人提出使用場景和邊界條件，AI 負責架構設計和全部程式碼。每個功能都是從真實使用情境反推出來的，而不是從技術能力正推。

關鍵是「追問邊界條件」：
- 「如果參考圖有文字怎麼辦？」→ 演化出三層優先級的佈局系統
- 「合規檢查真的每次都要跑嗎？」→ 改成按需觸發，省 50% API 成本
- 「API 花費上限查不到？」→ 發現 AI Studio 和 Cloud Billing 是兩套系統

### 可移植性

**可直接複製的部分**：
- 4 階段 AI pipeline 的架構（分析 → 優化 → 生成 → 檢查）適用於任何「AI 生成 + 品質把關」的場景
- Aspect Ratio normalization 的防禦策略適用於所有 Gemini 圖片生成
- 前端費用預估的做法適用於成本確定性高的 AI API（圖片生成、TTS 等）

**這個案例的特殊條件**：
- Gemini Flash Image 的圖片生成品質是核心（換模型效果不同）
- Amazon 的規範結構化程度高，適合嵌入 prompt（其他平台不一定）

### 如果你也要做，先問 AI 什麼

1. **「我要做一個 [領域] 的 AI 工具，使用者是 [角色]，痛點是 [問題]。我想用 [AI 模型]。你建議怎麼架構 API pipeline？」**

2. **「這個 AI API 有哪些已知限制（rate limit、支援的參數範圍、計費方式）？我要怎麼在程式碼裡做防禦？」**

_開發日期：2026-03-26 ~ 2026-03-27_
