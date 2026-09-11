# CLAUDE.md

本檔案為 Claude Code 在此子資料夾工作時的指引。此資料夾**本身是獨立 git 儲存庫**，不受根目錄工作區規則約束（除語言等全域偏好）。

## 這是什麼

**亞馬遜商品組合上架收支平衡計算機**，單檔前端、無後端。2026-09-11 依使用者要求建置，姊妹專案是 `資料儀表板/amazon-cost-calculator`（單一商品美日雙站成本瀑布試算），但計算層級完全不同：`amazon-cost-calculator` 算「一件商品該賣多少錢、能開多少採購價」，本工具算「一整批多分類、多商品的組合，整體收支是否平衡、哪些該補貨、商品在賣場結構裡扮演什麼角色」。

使用者原始需求逐項對應（詳見 `~/.claude/plans/amazon-rosy-thunder.md`，若已被清除則以下方各節為準）：

- 「平台有規定上架總金額不能超過金額」→ 這**不是** Amazon 官方硬性規則，是使用者自訂的可選庫存投入金額警戒線（`state.capEnabled`/`state.capAmount`），比對對象為新增的選填「建議售價」欄位 × 庫存數量加總，未填售價的商品不計入比對但仍正常計入毛利/淨利。
- 「公司最低獲利金額才能開銷金額」→ 公司每週／每月固定支出門檻（`state.weeklyThreshold`/`state.monthlyThreshold`），拿來跟全部商品「抽成後淨利」加總比較，判斷整體收支獲利／打平臨界／虧損三態。
- 「分類抽成百分比」→ 套用在使用者輸入的「毛利」上，算出 Amazon 抽成後淨利：`淨利 = 毛利 × (1 − 分類抽成%)`，淨利才是拿去跟門檻比較的數字（不是毛利本身）。
- 「品名/庫存數量/銷售數量(週)/銷售數量(月)/毛利(週)/毛利(月)」→ 商品六欄位原樣保留，另外新增「所屬分類」（下拉）與「建議售價」（選填，新增欄位，見上）。
- 「補貨所需天數」→ 全域前置期輸入，跟每項商品「庫存可撐天數」比對分三級：緊急（可撐天數 < 前置期）／注意（< 前置期×1.5）／正常。

**2026-09-11 使用者中途追加需求**（在原計畫核准後、實作過程中補充）：AI 診斷提示詞裡也要包含「商品的搭配建議」，並提供使用者給的賣場商品結構角色定義（主力／輔助／附屬／聯想／刺激商品）。因此新增「商品組合陳列角色」整個區塊，見下方「商品角色分類」一節。

不套用序號授權，公開免費工具（使用者明確選擇），無可攜式桌面版 exe。是否推公開 GitHub repo／GitHub Pages 尚待使用者本機驗證後另外確認（依 [[pref-confirm-before-deploy-new-experimental-tool]] 記憶慣例，新實驗性工具不自動上線）。

## 架構

`index.html` 單一 IIFE `<script>`，型態部分沿用 `amazon-cost-calculator`／`restaurant-feasibility-calculator` 已驗證的模式（單一計算源＋扇出渲染＋BYOK AI診斷整包＋PDF靜態報告＋跑馬燈＋PWA），但表單改成「分類＋商品」兩張可動態新增/刪除列的表格，而非固定滑桿：

- `state` — `{weeklyThreshold, monthlyThreshold, restockLeadDays, capEnabled, capAmount, categories:[{id,name,feePct}], products:[{id,name,categoryId,price,stock,weeklySales,monthlySales,weeklyProfit,monthlyProfit}], _catSeq, _prodSeq}`，整包存 `localStorage` key `amazonListingMixState`。分類/商品刪除用事件委派（`tbody` 上綁一個 `input`/`change`/`click` listener，用 `closest('tr').dataset.id` 找目標），只有新增/刪除列或切換範例時才整段 `renderCategories()`/`renderProducts()` 重建 DOM，單純編輯欄位值不重建（避免輸入中失焦）。
- `calculate()` — 核心計算源，回傳 `{perProduct, totals, catRollup}`。`perProduct[i].weeklyNet/monthlyNet` = 毛利 ×（1－該分類抽成%）；`daysLeft` = 庫存 ÷（週銷量÷7），週銷量為 0 時為 `null`（不計入補貨警示，避免除以零）；`listingValue` = 售價×庫存（售價未填則為 `null`，不計入 `totals.listingTotal`）。
- `classifyProductRoles(perProduct)` — 商品組合陳列角色分類引擎，見下節。
- `PRESETS` — 5 組虛構情境（居家小家電／3C配件／戶外露營／美妝保養／寵物用品），刻意設計成分別展示「全數健康達標」「緊急+注意兩級補貨警示」「上架金額超過自訂上限」「未達獲利門檻(虧損)」「多分類混合+零銷售商品邊界情況」五種不同狀態，`applyPreset()` 直接整批覆寫 `state.categories`/`state.products`/門檻/上限/前置期後重算。首次開啟（無 localStorage）會自動套用 `PRESETS[0]`，避免空白畫面。
- `AI_PROVIDERS`／`callLLM`／BYOK 設定面板 — 逐字比照 `amazon-cost-calculator` 已驗證的實作（Claude 需要 `anthropic-dangerous-direct-browser-access` header；429/500/503/529 自動重試3次）。金鑰只存 localStorage（`amazonListingMixApiConfig`），不經任何後端。
- `buildDiagnosisPrompt()` — 除了整體收支/補貨警示/分類彙總，**每次請求都固定附上商品角色分類清單，並明確要求 AI 給出商品搭配陳列建議**（不論使用者點選哪個角度按鈕），這是使用者中途追加的需求，不是選用角度之一。`AI_PROMPT_PRESETS` 5組快速角度改為：整體收支健檢／補貨優先順序建議／滯銷商品處置建議／分類抽成優化／新品上架規劃。
- `ruleBasedDiagnosis()` — 無金鑰或AI失敗時的規則式 fallback，除了收支/補貨句子，也會動態帶入實際的主力商品名稱＋刺激或聯想商品名稱組一句搭配陳列建議（例如「可考慮把刺激商品「X」擺放在主力商品附近陳列或搭售」），純規則式、不需要 AI 也看得到搭配建議。

### 商品角色分類（`classifyProductRoles`）

依使用者提供的賣場商品結構角色定義（主力／輔助／附屬／聯想／刺激商品，逐字保留在 `ROLE_DEFS`），用簡化的 ABC 分析＋分類關聯性＋價格銷量規則推算：

1. 依 `monthlyNet` 由大到小排序，累計佔全部正淨利加總的比例。
2. 累計比例 < 60% 的商品 → **主力商品**，同時記下它們所屬的分類（`coreCatIds`）。
3. 累計比例 60%–85%：若該商品分類已出現過主力商品 → **聯想商品**（同分類、關聯性強）；否則 → **輔助商品**（不同分類、拓寬品項寬度）。
4. 累計比例 ≥85% 的尾端商品，再依售價／銷量二次判斷：售價落在全體售價後四分之一（最低 25%）**且**月銷量 ≥ 全體中位數 → **刺激商品**（低價高頻帶流量）；其餘 → **附屬商品**。
5. 全部商品淨利加總 ≤0（例如虧損或無資料）時，直接全部歸為附屬商品，不跑分級（避免除以零／負值比例失真）。

這是**簡化規則式分類，非嚴謹商品管理理論**，UI 與 manual.html 皆已明確標註「僅供參考」。分類結果會傳進 AI 診斷 prompt，讓 AI 可以在此基礎上補充或指出分類不合理之處。

### 5 組範例的設計動機（供之後調整範例時參考）

| 範例 | 分類數 | 商品數 | 刻意展示的狀態 |
|---|---|---|---|
| 居家小家電組合 | 2 | 6 | 全數正常（無補貨警示、未超上限、獲利達標）——健康基準組 |
| 3C配件組合 | 2 | 5 | USB-C快充頭庫存15件週銷20＝5.3天(緊急)；運動手錶錶帶14天(注意)；未啟用上限 |
| 戶外露營用品組合 | 2 | 5 | 客單價高，庫存×售價加總 455,235 > 自訂上限 350,000（超過） |
| 美妝保養組合 | 2 | 5 | 固定支出門檻設較高（週5,000/月20,000），抽成後淨利週4,293/月17,986皆未達標（虧損） |
| 寵物用品組合 | 3 | 6 | 三分類混合抽成(15%/15%/20%)；「寵物羽毛逗貓棒」週銷月銷皆為0，示範無銷售紀錄不算除以零；另有緊急+注意各一項補貨警示 |

以上數字已用 Playwright 在 `preview_start`（port 8814）逐一點開五組範例並讀 `#kpiGrid`/`#restockTbody` 文字內容核對，跟手算結果完全吻合（含分類混合抽成、庫存可撐天數、上限比對）；新增/刪除分類與商品、刪除「有商品使用中」的分類會被擋下、PDF報告產生（stub `window.print`）、手機寬度(375px)與桌機寬度(1411px)皆已驗證無橫向溢出，也一併驗證過。

**已修過一個真實 CSS bug**：`.results-columns` 是 `grid-template-columns:1.1fr 1fr` 的 grid，裡面的 `.panel-box` 沒設 `min-width:0` 時，grid item 預設 `min-width:auto` 會被內部表格（`table` 的 `min-width:640px`）撐開，導致 `.table-scroll` 的 `overflow-x:auto` 失效、整個 section 橫向溢出桌面版視窗（1411px 視窗量到 1485px scrollWidth）。修法是在 `.panel-box` 與 `.table-scroll` 都加 `min-width:0`，讓 flex/grid 子項可以正常收縮、內部改用自己的橫向捲動。之後若在別的工具遇到「表格用 `.table-scroll` 包了但整頁還是橫向溢出」，優先檢查外層是不是 flex/grid 容器缺了 `min-width:0`。

## 週邊功能（比照 amazon-cost-calculator 已驗證的實作）

- **PDF匯出**（`#pdfExportBtn`）— 走「獨立靜態報告」路線：`buildPrintReport()` 用目前 `state`／`calculate()` 現組一份純靜態 HTML（整體收支概況＋分類別彙總＋補貨警示＋商品組合陳列角色＋若曾產生過的AI診斷）塞進 `#printReportRoot`，`@media print` 只把該區塊／`#pdfWatermark` 設回可見。浮水印 `#wmImg` 的 base64 data URI 直接複用 `amazon-cost-calculator/index.html` 裡的同一張「馬克老師」品牌圖（用 Python 腳本字串替換注入，未進對話視窗，見下方指令）。
- **頂部跑馬燈** — 獨立 IIFE，逐字複製 `amazon-cost-calculator` 的實作，`MARQUEE_CHECK_URL` 沿用同一顆共用 Google Apps Script 端點，localStorage key 改為 `amazonListingMixCalcMarquee`。
- **`manual.html`** — 獨立頁面，內容依本工具操作流程改寫（快速範例/全域設定/分類抽成/商品角色分類邏輯說明/AI診斷含固定搭配陳列建議/PDF匯出/PWA/隱私/警語），創作者資料區塊逐字比照姊妹專案。
- **PWA** — `manifest.json`＋`service-worker.js`（network-first＋同源快取備援）＋`icons/`（PIL 產生，深色森林墨綠底＋白色天秤圖案，象徵「收支平衡」，192/512/maskable-512/apple-touch-icon 四種尺寸，產生腳本 `_gen_icons.py` 用完即刪未進 repo）；安裝按鈕 `#installBtn`＋`#toast`，安裝腳本逐字沿用 [[pwa-install-rollout]] 記載已修好的版本。
- **訪客計數器** — `visitor-badge.laobi.icu`，`page_id=m255525.amazonlistingmixcalculator`，放 footer。

## localStorage

- `amazonListingMixState` — 全部門檻/上限/分類/商品資料（reload 後還原）。
- `amazonListingMixApiConfig` — `{provider, model, apiKey, extra}`，只存本機瀏覽器。
- `amazonListingMixActivePreset` — 目前套用中的範例 id；手動編輯任一欄位會清空（`clearActivePreset()`）。
- `amazonListingMixCalcMarquee` — 跑馬燈內容快取。

「重設為範例一」（`resetToPresetOne()`）直接呼叫 `applyPreset(PRESETS[0], true)`，會整批覆寫分類/商品/門檻，不像 `amazon-cost-calculator` 只重設數值滑桿——因為本工具的分類/商品本身是使用者自由增減的清單，沒有獨立於範例之外的「基準假設」概念。AI 設定不受影響。

## 指令

無建置步驟。直接開啟 `index.html`（`file://`）或用伺服器託管即可。

預覽伺服器：port `8814`（工作區根目錄 `.claude/launch.json` 的 `amazon-listing-mix-calculator` 項目），用 Preview MCP 的 `preview_start` 啟動；若該 MCP 不可用，退回 `python -m http.server 8814 --directory 資料儀表板/amazon-listing-mix-calculator` 暫起、測完關閉。

若要重新產生浮水印注入（例如浮水印圖片要換新），沿用以下作法（不要把 base64 貼進對話視窗）：
```python
import re
src = open('../amazon-cost-calculator/index.html', encoding='utf-8').read()
m = re.search(r'id="wmImg" src="(data:image/png;base64,[^"]+)"', src)
html = open('index.html', encoding='utf-8').read().replace(現有的data-uri, m.group(1))
open('index.html', 'w', encoding='utf-8').write(html)
```
