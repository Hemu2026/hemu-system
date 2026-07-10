# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## 專案概述

和木餐飲集團的餐飲營運管理系統，管理「林家涼麵」（品牌 a）與「木谷食麵所」（品牌 b）兩間門市，以及對外批發子品牌「醬醬人 SauceMan」。系統採「前端單一 HTML 檔 + GAS 後端 + Google Sheets 資料庫 + LINE 推播」的輕量架構，沒有 build/lint/test 工具鏈——這是純手寫的 vanilla JS 單檔應用，沒有套件管理、沒有自動化測試。

## 本地檔案與部署

- `index.html`：主系統（登入、每日記錄、月報、薪資、原物料、生產紀錄、央廚等所有頁面），前端邏輯全部塞在一個 `<script>` 裡，函式與變數命名大量使用中文（例如 `格式化金額`、`渲染月報資料`、`儲存每日記錄`），修改時延續同樣的命名風格
- `sauceman.html`：對外批發網站「醬醬人 SauceMan」的獨立靜態頁面，跟 `index.html` 不共用 JS，是給批發客戶看的行銷頁+申請表單
- **本地並沒有 GAS 原始碼檔案**（`.gs`）。GAS 後端只存在 Google Apps Script 雲端專案（無 clasp 同步），修改後端一律要打開雲端編輯器操作
  - GAS 專案編輯器：`https://script.google.com/home/projects/1-j-vb--RuCtYWOotRmWf9HVxSK0iaXUHIdIhNJ-YoV0SG5KzimdiIyJl/edit`（注意是大寫 I 的 `diIyJl`）
  - 部署後的 exec URL 寫死在 `index.html` 的 `const API = '...'`，GAS 改完程式碼一定要「部署→管理部署作業→編輯現有部署→建立新版本→部署」，單純存檔（Cmd+S）不會讓已上線的 `/exec` 或 LINE webhook 吃到新程式碼

### 推送到 GitHub
```bash
cd /Users/gua/Downloads/hemu-system
cp "/Users/gua/Desktop/vs code/index.html" index.html   # 若是在桌面那份改的，先同步過來
git add index.html
git commit -m "說明"
git push origin main
```
`/Users/gua/Desktop/vs code/index.html` 是另一份工作檔（沒有版控），實際編輯常在那邊發生，push 前記得先同步覆蓋這裡的 `index.html`。

## 架構：GAS 後端檔案分工

- **程式碼.gs**：主程式，`doPost`/`doGet` 入口、`handleRequest` action 分派、核心 CRUD（`saveRecord`/`getRecords`/`updateRecord`/`getStaffProfiles`/`saveStaffProfile`/`deleteStaffProfile` 等）、B2B、原物料、生產紀錄、促銷試算（F1）、LINE 進貨記帳與補給官示警（B1）的計算函式都在這裡
- **GoogleLogin.gs**：Google OAuth 登入相關 + `deleteRecord(sheetName, id)`

前端呼叫後端一律透過 `呼叫API(參數)`（`index.html`），送 `{action: 'xxx', ...}` 到 `API` 這個 exec URL，GAS `handleRequest` 用 `switch(action)` 分派到對應函式。新增功能時，前端加呼叫、GAS 加 case + 對應函式、需要新分頁就用 `ensureXXXSheet_()` 模式確保表頭存在。

## 架構：Google Sheets 分頁對應用途

| 分頁 | 用途 |
|---|---|
| 每日記錄 | 每日營收明細（現金/LINE Pay門市/線上點餐/台灣Pay/Uber/熊貓/客單價等），A1、月報表、F1 都讀這張表當基準線 |
| 薪資 | 員工薪資紀錄，存檔後自動同步一筆扣款到資金管理（帳戶固定「富郁霖卡」） |
| 設定 | 各品牌固定成本、費率（LINE Pay/Uber/熊貓費率）、Claude API 金鑰等設定值 |
| 帳號 | 系統登入帳號（email/密碼/角色/品牌） |
| FundRecords | 資金流水記錄，欄位 `id, date, account, type, amount, toAccount, note`（注意是 camelCase 的 `toAccount`） |
| B2B客戶 / B2B訂單 | 醬醬人批發客戶申請與訂單 |
| 原物料 | 食材庫存主檔（進價/得率/進貨單位換算/目前庫存），B1 補給官的歷史均價比對也是靠掃「進貨申報」表反推 |
| 配方 | 商品成本拆解用的原物料配方，成本表即時重算依賴這張表 |
| 員工資料 | 完整人事資料（身分證/學歷/緊急聯絡人/勞健保） |
| 進貨申報 | LINE 拍照/文字進貨記帳的存放表，**這張表沒有表頭列**（第一列就是資料），讀取時不能用 `headers.indexOf()`，要用寫死的欄位 index |
| 員工訂購 | 員工內部訂購紀錄（用內部供應價結算，不是食材成本價） |
| 生產紀錄 | 商品生產登記，送出時自動扣原物料庫存、加商品庫存，刪除時回沖 |
| 月份成本 | 每月可調整的變動費用（電話/廣告/水電/其他/營業稅） |
| 促銷試算 | F1 平台利潤極大化專員的輸入輸出表，欄位見下方 F1 章節 |
| 請假紀錄 | C1 虛擬店長的請假登記表，欄位 `id, name, date, brand, registeredAt, payrollProcessed`，由 LINE 訊息「姓名 日期 請假」寫入，`savePayroll` 存檔後自動標記該員工當月紀錄為已核發 |
| 林家涼麵 / 木谷食麵所 | 「每日記錄」的 FILTER 分頁，自動篩出對應品牌資料，唯讀用途 |

## 紅線規則

1. **數據一律從源頭獨立計算，禁止直接複製或修改現成報表上的數字**。任何新報表/新 agent 的數字都必須重新從「每日記錄」「原物料」「進貨申報」等原始表格算出，不可以把別的報表算好的數字複製貼上或手動改寫進另一張表/另一個計算裡。
2. **A2 檔案長（未來的歷史紀錄/知識庫功能）的歷史紀錄只能新增版本，不得刪除**。設計 A2 時，任何寫入都是 append 新版本，不做覆寫或刪除既有紀錄的操作。

## LINE 推播格式規範

比照現有 A1（每日總管）、F1（促銷試算）的推播風格，之後每個新 agent（B1、C1...）的訊息格式都要遵循：

- **標題**：第一行是「【Agent 代稱】主題」的格式，例如「⚠【補給官】進價異常提醒」，emoji 放最前面表示訊息性質（✓ 正常完成 / ⚠ 異常示警 / 📊 報表類）
- **數字呈現**：金額一律用千分位（`格式化金額()` 風格，`Math.round` 後 `toLocaleString('zh-TW')`），百分比取小數點後 1 位並加 `%`，不顯示過多小數位造成閱讀負擔
- **內容結構**：標題 → 關鍵指標（每行一項，「項目：數值」格式）→ 需要時加一行結論/建議（如 F1 的「建議參加」「建議搭配加購」，B1 的漲幅百分比與歷史均價對照）
- **簡潔**：不加多餘的寒暄或裝飾文字，一則訊息只講一件事，需要示警時額外獨立發一則，不要跟正常通知混在一起

## Agent（虛擬員工）進度總覽

系統以「Google Sheets 為底層數據源、GAS 計算、LINE 推播」統一架構，逐步擴充多個職能 agent：

- **A1 小總管（幕僚長）**：✅ 已上線。每日讀取昨日「每日記錄」，推播營業額/來客數/客單價/預估毛利到 LINE
- **F1 平台利潤極大化專員**：✅ 已完成。「促銷試算」分頁 onEdit 觸發，自動算促銷後毛利率、稀釋幅度、參加建議、加購建議金額
- **B1 補給官**：✅ 已上線。LINE 進貨記帳當下即時比對歷史均價，漲幅超過 10% 推播示警
  - LINE 進貨記帳支援補記過去日期（2026-07-10新增）：訊息開頭加「昨天」「前天」或「M/D」「YYYY-MM-DD」（如「昨天 洋蔥300」），`extractPurchaseDate_` 解析後覆蓋預設的今天日期；不加前綴則跟以前一樣記今天。回覆訊息若非當天會標示「（日期：xxx）」
- **C1 虛擬店長**：🚧 簡化版已上線（請假登記+自動扣薪），完整版（人力門檻檢查+自動催代班）待實作
  - 已上線：員工 LINE 傳「姓名 日期 請假」（如「小美 7/15 請假」）自動比對員工名單歸屬品牌、寫入「請假紀錄」分頁、回覆確認訊息；格式錯誤或姓名比對不到會回覆失敗原因，不會默默漏記
  - 已上線：開啟薪資頁「新增員工薪資」選擇員工時，自動抓該員工當月未核發的請假紀錄，用「本薪÷當月天數×請假天數」試算建議扣款金額帶入「缺勤扣款」欄位（仍可手動覆蓋），並提示本月請假日期清單
  - 已上線：`savePayroll` 存檔後自動把該員工當月請假紀錄標記為已核發（`markLeavesProcessed_`），避免下個月重複帶入
  - 待實作：人力門檻檢查（基本班表：各店每天需要幾人）、人力不足時自動催代班公告
- **D1/E1 品質客服**：📋 待實作（央廚稽查/留言監控，決定先做簡化合併版）
- **A2 檔案長**：📋 待實作（知識庫/決策歸檔，排最後）

開發優先順序：F1 → B1 → C1 → D1/E1 → A2（F1、B1 已完成，C1 簡化版已完成）。新 agent 開發時優先沿用同一套「Sheets 分頁當輸入、GAS 函式當計算引擎、LINE 當輸出」模式，不重新造輪子。

## 已知的反覆故障模式

- **Sheet 表頭欄位名稱前後端拼法不一致**（camelCase vs snake_case，或欄位名稱打錯/重複）是這個系統最常見的 bug 來源，遇到「邏輯看起來對但結果沒發生」，先用 `curl -s -L "https://docs.google.com/spreadsheets/d/{sheetId}/gviz/tq?tqx=out:csv&sheet={分頁名}"` 直接確認 Sheet 實際表頭
- **onclick 裡直接塞 `JSON.stringify()` 的字串**會因為雙引號截斷屬性，凡是把 note/文字內容塞進 `onclick="..."` 的地方都要做 `.replace(/"/g,'&quot;')` 轉義
- **GAS 部署一定要建立新版本才生效**，光是編輯器存檔不影響已上線的 `/exec` 行為，這個坑反覆出現
- 操作 GAS 雲端 Monaco 編輯器時，`browser_type`/`.fill()` 對 textarea 會整個覆蓋文件內容，要用 `monaco.editor.getModels()` + `pushEditOperations()` 做精準片段替換，且先用 `indexOf` 確認錨點字串在原始碼中唯一，避免整檔字串取代造成內容暴增或重複
