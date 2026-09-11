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

**2026-09-11 同日第二輪追加需求**（工具已上線一版之後，使用者再提出四項）：
1. 「再增加一個成本欄位」→ 每項商品新增「商品成本」（選填，固定以台幣輸入，代表國內進貨成本，與售價幣別無關）。
2. 「可以輸出PDF 或是word」→ 新增 Word（.docx）匯出，內容與 PDF 報告一致。
3. 「輸出會在文件增加浮水印」＋使用者直接附上一張新的「馬克老師」吉祥物圖檔 → PDF 與 Word 都要有浮水印（見下方「浮水印」一節，換成使用者這次提供的圖）。
4. 「在商品清單增加上傳csv 或是excel，上傳後會自動辨識將資料放到適當欄位，並可以下載其內容」→ 商品清單支援 CSV／Excel 匯入（自動欄位比對＋分類自動建立，取代目前商品清單）與 CSV 匯出。
5.（同一輪稍後追加）「售價可以選擇美金/日幣，增加與台幣轉換匯率」→ 「建議售價」欄位改為固定以 USD 或 JPY 輸入（不再是無單位的台幣數字），全域設定新增「美元兌台幣匯率」「日圓兌台幣匯率」兩個欄位，換算後的台幣金額才拿去跟上架金額上限比對、算入 KPI；「商品成本」維持純台幣、不受此影響。5 組範例的售價數字因此整批重新設計成貼近真實的 USD／JPY 售價（不是把原本 NT$ 數字硬貼幣別標籤），並同步調整對應的上架金額上限，讓「camping 超過上限」「home 未超過上限」的示範狀態維持成立。

## 架構

`index.html` 單一 IIFE `<script>`，型態部分沿用 `amazon-cost-calculator`／`restaurant-feasibility-calculator` 已驗證的模式（單一計算源＋扇出渲染＋BYOK AI診斷整包＋PDF靜態報告＋跑馬燈＋PWA），但表單改成「分類＋商品」兩張可動態新增/刪除列的表格，而非固定滑桿：

- `state` — `{weeklyThreshold, monthlyThreshold, restockLeadDays, capEnabled, capAmount, usdToTwd, jpyToTwd, categories:[{id,name,feePct}], products:[{id,name,categoryId,price,priceCurrency,cost,stock,weeklySales,monthlySales,weeklyProfit,monthlyProfit}], _catSeq, _prodSeq}`，整包存 `localStorage` key `amazonListingMixState`。分類/商品刪除用事件委派（`tbody` 上綁一個 `input`/`change`/`click` listener，用 `closest('tr').dataset.id` 找目標），只有新增/刪除列或切換範例時才整段 `renderCategories()`/`renderProducts()` 重建 DOM，單純編輯欄位值不重建（避免輸入中失焦）。`price` 固定以 `priceCurrency`（`"USD"`／`"JPY"`，預設 `USD`）輸入；`cost` 固定台幣、無幣別欄位。
- `calculate()` — 核心計算源，回傳 `{perProduct, totals, catRollup}`。`perProduct[i].weeklyNet/monthlyNet` = 毛利 ×（1－該分類抽成%），**不受幣別/匯率影響**；`daysLeft` = 庫存 ÷（週銷量÷7），週銷量為 0 時為 `null`（不計入補貨警示，避免除以零）；`priceTWD` = `price × rateForCurrency(priceCurrency)`（`rateForCurrency()` 依幣別回傳 `state.usdToTwd` 或 `state.jpyToTwd`）；`listingValue` = `priceTWD × 庫存`（售價未填則為 `null`，不計入 `totals.listingTotal`）；`costValue` = `cost × 庫存`（成本未填則為 `null`，不計入新增的 `totals.costTotal`）。`classifyProductRoles()` 的售價百分位判斷改用 `priceTWD` 而非原始 `price`，避免 USD 與 JPY 數字直接混比。
- **舊版 localStorage 相容性**：`loadState()` 讀到沒有 `usdToTwd`/`jpyToTwd` 欄位的舊存檔（本次功能上線前存的）時會自動補上預設匯率（31.5／0.21）。這是真的踩過的 bug——沒補的話 `rateForCurrency()` 會回傳 `undefined||0 = 0`，導致舊資料的 `priceTWD` 全部變 0，上架金額 KPI 卡靜默顯示 NT$0 而不報錯，Playwright 測試時實際踩到才發現。
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

| 範例 | 分類數 | 商品數 | 售價幣別 | 刻意展示的狀態 |
|---|---|---|---|---|
| 居家小家電組合 | 2 | 6 | USD | 全數正常（無補貨警示、未超上限、獲利達標）——健康基準組；上架金額294,377 < 自訂上限350,000 |
| 3C配件組合 | 2 | 5 | USD | USB-C快充頭庫存15件週銷20＝5.3天(緊急)；運動手錶錶帶14天(注意)；未啟用上限 |
| 戶外露營用品組合 | 2 | 5 | USD | 上架金額（庫存×售價已換算台幣）469,267 > 自訂上限400,000（超過） |
| 美妝保養組合 | 2 | 5 | JPY | 固定支出門檻設較高（週5,000/月20,000），抽成後淨利週4,293/月17,986皆未達標（虧損） |
| 寵物用品組合 | 3 | 6 | USD＋JPY混合 | 三分類混合抽成(15%/15%/20%)＋同一組內混用美金與日幣售價（示範per-product幣別）；「寵物羽毛逗貓棒」週銷月銷皆為0，示範無銷售紀錄不算除以零；另有緊急+注意各一項補貨警示；上架金額97,696 < 上限100,000 |

2026-09-11 第二輪追加需求後，5 組範例的售價全部重新設計成貼近真實的 USD／JPY 零售價格（例如電動打蛋器 $24.99、露營帳篷 $89.99、面膜組 ¥1480），而非把原本無幣別的 NT$ 數字直接貼上幣別標籤（那樣會出現「$899的打蛋器」這種失真情境）；`weeklyProfit`/`monthlyProfit`（毛利，跟幣別無關）完全不動，所以收支平衡/補貨警示的既有驗證結果不受影響，只有跟「售價」相關的上架金額/上限比對/角色分類售價百分位需要重新驗證。`cost`（商品成本，台幣）用「售價換算台幣後的 45%~55%」估算，是虛構參考數字。

以上數字已用 Playwright 在 `preview_start`（port 8814）逐一點開五組範例並讀 `#kpiGrid`/`#restockTbody` 文字內容核對，跟手算結果完全吻合（含分類混合抽成、庫存可撐天數、上限比對、USD/JPY匯率換算）；新增/刪除分類與商品、刪除「有商品使用中」的分類會被擋下、手動編輯商品的售價/成本/幣別即時更新KPI、PDF報告產生（stub `window.print`）、手機寬度(375px)與桌機寬度(1411px)皆已驗證無橫向溢出，也一併驗證過。

**已修過一個真實 CSS bug**：`.results-columns` 是 `grid-template-columns:1.1fr 1fr` 的 grid，裡面的 `.panel-box` 沒設 `min-width:0` 時，grid item 預設 `min-width:auto` 會被內部表格（`table` 的 `min-width:640px`）撐開，導致 `.table-scroll` 的 `overflow-x:auto` 失效、整個 section 橫向溢出桌面版視窗（1411px 視窗量到 1485px scrollWidth）。修法是在 `.panel-box` 與 `.table-scroll` 都加 `min-width:0`，讓 flex/grid 子項可以正常收縮、內部改用自己的橫向捲動。之後若在別的工具遇到「表格用 `.table-scroll` 包了但整頁還是橫向溢出」，優先檢查外層是不是 flex/grid 容器缺了 `min-width:0`。

## 週邊功能（比照 amazon-cost-calculator 已驗證的實作）

- **PDF匯出**（`#pdfExportBtn`）— 走「獨立靜態報告」路線：`buildPrintReport()` 用目前 `state`／`calculate()` 現組一份純靜態 HTML（整體收支概況含匯率設定與總庫存成本＋分類別彙總＋補貨警示＋商品組合陳列角色＋若曾產生過的AI診斷）塞進 `#printReportRoot`，`@media print` 只把該區塊／`#pdfWatermark` 設回可見。
- **Word匯出**（`#wordExportBtn`，2026-09-11 新增）— 用 `docx@9.7.1` CDN（`https://cdn.jsdelivr.net/npm/docx@9.7.1/dist/index.iife.min.js`，`loadDocxLib()` 懶載入，逐字比照 `phoenix-loan-generator`/`text-organizer-studio` 已驗證的 CDN 載入方式），但**不是**沿用它們的 DOM-walker 手法（`_docxItemsFromContainer()` 那套），而是 `buildWordBlob()` 直接依 `calculate()` 的資料重新組出 `docx.Paragraph`/`docx.Table`（`buildDocxTable()` 小工具函式），跟 `buildPrintReport()` 的 HTML 版本是兩份平行但邏輯一致的程式碼，內容涵蓋整體收支概況／分類別彙總／補貨警示／商品組合陳列角色／AI診斷（若有）。這是本工作區第一個真正產生 `.docx`（透過 `docx.Packer.toBlob()`）且內嵌浮水印圖片的工具——`buildWatermarkImageRun()` 用 `docx.ImageRun` 的 `floating:{behindDocument:true, wrap:{type:docx.TextWrappingType.NONE}}` 把浮水印放進 `docx.Header`（每頁頁首重複出現）。**這個 API 用法在寫入正式程式碼前，已先在一個獨立沙箱頁面（非本 repo 內）用 Playwright 實測過**：確認 `type:"png"` 為必填欄位、`data` 可直接吃 base64 字串（不需转 Uint8Array）、`floating`/`Header`/`HeadingLevel`/`Table` 的 API 形狀都跟 docx.js 官方文件一致；正式整合後又用 JSZip 解開實際產生的 `.docx`，確認 `word/document.xml` 含 `Heading1`／`<w:tbl>`、`word/header1.xml` 含圖片參照（`<pic:pic`/`a:blip`）、`word/media/` 底下真的有 png——不是只看「沒有 JS 錯誤」就假設成功。
- **CSV／Excel 匯入匯出**（2026-09-11 新增）— `PapaParse 5.4.1` + `xlsx(SheetJS) 0.18.5`，皆從 cdnjs 懶載入（`loadCsvLibs()`，跟 `資料儀表板/Dashboard` 用的同一組版本一致，唯獨 Dashboard 是頁面一載入就靜態 `<script>` 引入、本工具改成使用者第一次按「上傳」或「下載」才動態載入，因為 CSV 匯入在本工具是次要功能非主要工作流）。`PRODUCT_FIELD_ALIASES` 定義每個欄位的多組別名（中英文皆有），`normalizeHeader()`／`mapRowHeaders()` 做欄位標題模糊比對；分類名稱用 `findOrCreateCategoryByName()` 比對現有分類或自動新增（預設抽成15%）；匯入行為是**取代**整份商品清單（分類清單不動，包含匯入時新建的分類）。CSV 下載（`downloadProductsCsv()`）與匯入共用同一組欄位（品名/分類/建議售價/售價幣別/商品成本/庫存數量/週銷量/月銷量/週毛利/月毛利），確保可以下載→離線編輯→重新上傳來回互通；下載檔案開頭加 UTF-8 BOM（`"﻿"`）避免 Excel 開啟中文亂碼，逐字比照 `product-title-generator` 既有慣例。已用 Playwright 建構假的 CSV（含別名標題＋部分欄位留空）與假的 XLSX（用 `XLSX.utils.aoa_to_sheet`/`XLSX.write` 現組二進位）實際觸發 `<input type=file>` 的 `change` 事件驗證過，包含新分類自動建立、幣別辨識、必填欄位缺漏時的容錯。
- **浮水印圖片**（2026-09-11 換新）— 使用者本次直接提供一張新的「馬克老師 AI‧工具‧學習‧成長」吉祥物圖檔（`C:\Users\mark_\Downloads\ChatGPT Image 2026年8月10日 下午05_42_45.png`，1536×1024）。沒有直接處理這個 1.85MB 原始檔，而是發現 `行銷內容工具/new-product-strategy-studio/watermark-source.png`（480×315，RGBA已去背透明）跟已 byte-for-byte 比對確認是同一張圖的正確處理版本（該專案 CLAUDE.md 記載「已去背透明background，裁掉透明邊界後縮到寬480px」），所以直接複製那個檔案當來源，而非重新去背。用 PIL 把它的 alpha 通道整體乘 0.32 烤進一張新的 `watermark-source.png`（本專案根目錄，199KB），同一張淡化後的圖同時餵給 PDF 的 `<img>` 與 Word 的 `docx.ImageRun`（docx.js 沒有現成的「透明度」參數，所以淡化必須烤進圖片本身，不能像 PDF 那樣單靠 CSS `opacity` 控制）。**架構重構**：浮水印 base64 data URI 原本直接寫死在 `<img id="wmImg" src="...">` 屬性裡（跟 `amazon-cost-calculator` 一樣），這次改成 JS 常數 `var WATERMARK_DATA_URI = "data:image/png;base64,...";`（緊跟在 `PRESET_KEY` 之後宣告），初始化時用 `$("wmImg").src = WATERMARK_DATA_URI` 寫回 `<img>`，`docx.ImageRun` 則用 `WATERMARK_DATA_URI.split(",")[1]` 拿掉 prefix 取原始 base64——這樣同一份圖片資料只需要在檔案裡存一份，不必為了 PDF 和 Word 各複製一次巨大字串。PDF 端 CSS 也一併微調成跟 `new-product-strategy-studio` 一致的比例（`#pdfWatermark img{width:42%;min-width:200px;max-width:380px;}`，不再疊加額外的 wrapper `opacity`，因為透明度已經烤進圖片）。
- **頂部跑馬燈** — 獨立 IIFE，逐字複製 `amazon-cost-calculator` 的實作，`MARQUEE_CHECK_URL` 沿用同一顆共用 Google Apps Script 端點，localStorage key 改為 `amazonListingMixCalcMarquee`。
- **`manual.html`** — 獨立頁面，內容依本工具操作流程改寫（快速範例/全域設定/售價幣別與匯率換算/分類抽成/匯入匯出商品清單/商品角色分類邏輯說明/AI診斷含固定搭配陳列建議/PDF與Word匯出/PWA/隱私/警語），創作者資料區塊逐字比照姊妹專案。
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

若要重新產生浮水印注入（例如浮水印圖片要換新），流程是：(1) 準備好處理過的來源圖（去背/裁切/淡化，PIL）存成 `watermark-source.png`；(2) 用 Python 腳本把它 base64 編碼後，正則取代 `index.html` 裡 `var WATERMARK_DATA_URI = "data:image/png;base64,...";` 這一行——**不要把 base64 貼進對話視窗**：
```python
import re, base64
b64 = base64.b64encode(open('watermark-source.png', 'rb').read()).decode('ascii')
html = open('index.html', encoding='utf-8').read()
html = re.sub(r'var WATERMARK_DATA_URI = "data:image/png;base64,[^"]+";',
              'var WATERMARK_DATA_URI = "data:image/png;base64,' + b64 + '";', html, count=1)
open('index.html', 'w', encoding='utf-8').write(html)
```
`<img id="wmImg" src="">` 與 `docx.ImageRun` 的資料來源都指到這同一個 JS 常數（見上方「浮水印圖片」一節），改一次全部生效。
