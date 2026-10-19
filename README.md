# 我的股票帳本

個人使用的台股投資紀錄網站，部署在 GitHub Pages。
記錄買進、賣出、現金股利（含股利再投資），自動計算手續費、證交稅、損益，並每天自動帶入收盤價。

> **給未來的 Claude**：這份文件記錄了這個專案的所有需求與設計決定。請先讀完再修改 `index.html`，並保留下面「已確認的需求」裡的行為，除非使用者要求改變。

---

## 檔案

| 檔案 | 用途 |
|---|---|
| `index.html` | 整個網站，單一 HTML 檔（畫面、計算、內建股票清單都在裡面，不依賴其他檔案） |
| `.github/workflows/update-prices.yml` | GitHub Actions 排程，每個交易日抓收盤價 |
| `prices.json` | 由排程自動產生的收盤價，**不要手動編輯或改名** |
| `README.md` | 本說明文件 |

---

## 已確認的需求（2026/10 與使用者討論定案）

### 範圍
- 市場：台股上市、上櫃、興櫃、ETF（含 ETN、特別股、TDR、REITs，不含權證）。美股之後再說。
- 單一帳戶。
- 交易類型只有三種：**買進、賣出、現金股利**（不做股票股利、減資、分割）。
- 整股、零股都要支援，數量可用「股」或「張」輸入。

### 計算規則
- **成本計算：平均成本法。**
- **成交金額 = 股數 × 價格，無條件捨去到整數**（與券商一致，例：30 股 × 107.15 = 3,214）。
- **手續費 = 成交金額 × 0.1425% × 折扣，無條件捨去**；低於最低收費時收最低收費。
  - 使用者的券商折扣是 **2.8 折**（預設值 `feeDisc: 0.28`），由實際對帳單驗證過。
  - 整股（股數為 1000 倍數）最低 20 元，零股最低 1 元。
  - 交易紀錄有 `feeAuto` 標記：折扣改變時，只重算自動計算的手續費，使用者手動改過的不動。
- **證交稅（賣出時）**：股票 0.3%、當沖 0.15%、ETF/ETN/REITs 0.1%、債券 ETF（代號 B 結尾）0%。
  - 債券 ETF 停徵至 2026/12/31；行政院已通過延長至 2036 年的草案，但截至 2026/10 尚未三讀。若沒延長，需到設定改成 0.1%。
- **未實現損益**預設扣除「現在賣出」的預估手續費與證交稅（設定可關閉）。
- **總報酬（含息）** = 未實現 + 已實現 + 累計現金股利；報酬率分母是「投入本金」＝自掏腰包的買進成本（不含股利再投資）。
- 漲跌顏色：**紅漲綠跌**（台股慣例）。

### 股利再投資
- 使用者有開股利再投資。記「現金股利」時勾選再投資、填買進價格，系統自動算可買股數與手續費，並產生一筆連結的買進紀錄（`reOf` 指向股利那筆的 id）。
- 刪除或修改股利紀錄時，連結的再投資買進一起處理。

### 現價
- 主要來源：`prices.json`（GitHub Actions 每天產生）。網頁開啟時讀取並自動帶入。
- 使用者也可在持股卡片手動改現價，按「儲存我改的現價」。
- 每筆現價記錄來源 `src`：`auto`（收盤價）、`manual`（自己填）、未標記（新增買進時用的買進價）。
  - 規則：自動收盤價日期較新就覆蓋；同一天時，手動填的優先。
- 每次現價更新會記一筆「資產走勢」快照（日期、市值、成本）。
- 首頁螢幕下方顯示「持股收盤價日期」，是使用者持股實際用到的收盤價日期（上市、上櫃公布時間可能不同，所以會顯示範圍，例如 10/16～10/19）。

### 股票清單
- 內建於 `index.html`（`STOCK_DATA`），來源是證交所 ISIN 公開資料，2026/10/19 版本，共 2,770 檔。
- 使用者可在「設定 → 股票清單」貼上證交所 ISIN 頁面內容（strMode=2 上市、4 上櫃、5 興櫃）來新增，存在瀏覽器 `localStorage`。
- 搜尋：輸入代號或名稱關鍵字（例「台灣」）即跳出選單，選了自動帶入代號與名稱。

### 介面
- 使用者覺得第一版「太複雜」，所以刻意簡化：只有 **首頁、紀錄、設定** 三個分頁。
- 首頁：上方「螢幕」只顯示三個數字（總市值、總共賺了含股利、累計股利）＋持股卡片＋資產走勢圖。
- 外觀：像素風老電腦桌面（參考使用者提供的 Canva 模板：米色、黃色視窗、黑色像素外框、草地地面）。
- 字體：中文用 Huninn（粉圓），**數字用 Space Mono**（使用者反映像素數字字體 VT323 看不清楚，已換掉）。
- 手續費、稅、備註收在「修改手續費、稅或加備註」裡，平常不顯示。
- 手動改的手續費與自動算的不同時，會推算折扣並提示「設為預設」。

---

## 資料存放

- 所有交易紀錄存在使用者**瀏覽器的 localStorage**，不會上傳到 GitHub。
  - `twLedger.data.v1`：交易、現價、走勢、設定
  - `twLedger.stocks.v1`：使用者自行新增的股票清單
- 換裝置要用「設定 → 匯出備份（.json）」再匯入。也有「複製文字備份」和「匯出 Excel（.csv）」。
- **改版時務必保持 localStorage 的 key 與資料格式相容**，不然使用者的紀錄會讀不到。若要改格式，需寫轉換程式。

資料格式（`twLedger.data.v1`）：
```js
{
  txs: [
    { id, type:"buy",  date:"YYYY-MM-DD", code, shares, price, fee, feeAuto, note, seq, reOf? },
    { id, type:"sell", date, code, shares, price, fee, feeAuto, tax, taxAuto, day /*當沖*/, note, seq },
    { id, type:"div",  date, code, amount /*實收金額*/, note, seq }
  ],
  prices: { "0050": { p:108.4, d:"2026-10-19", src:"auto"|"manual" } },
  snaps:  [ { d:"2026-10-19", mv, cost } ],
  settings: { feeRate:0.001425, feeDisc:0.28, discV:2, minBoard:20, minOdd:1,
              taxStock:0.003, taxDay:0.0015, taxETF:0.001, taxBond:0, unrealIncludeSell:true },
  sample: false,      // true 表示目前是範例資料
  lastBackup: "YYYY-MM-DD"
}
```

---

## 自動收盤價的運作方式

網頁無法直接讀證交所 API（對方不允許跨網站讀取，已實測），所以改用 GitHub Actions：

1. 每個交易日台灣時間 17:07、21:07，以及隔天早上 08:07 執行 `update-prices.yml`（也可在 Actions 分頁按 Run workflow 手動執行）。
2. 抓取：
   - 上市：`https://openapi.twse.com.tw/v1/exchangeReport/STOCK_DAY_ALL`（欄位 `Code`、`ClosingPrice`、`Date` 為民國年 `1151016`，已確認）
   - 上市備援：`https://www.twse.com.tw/rwd/zh/afterTrading/STOCK_DAY_ALL?response=json`（實測回傳的是開頭帶 BOM 的 CSV，程式會自動判斷 JSON 或 CSV）。openapi 曾在 2026/10/19 晚上仍停在 10/16 的資料，所以加了這個來源，同一檔取日期較新的。
   - 上櫃：`https://www.tpex.org.tw/openapi/v1/tpex_mainboard_daily_close_quotes`
   - 興櫃：`https://www.tpex.org.tw/openapi/v1/tpex_esb_latest_statistics`
   - 上櫃、興櫃的欄位名稱沒有實際確認過，程式用多個候選欄位名稱自動判斷。若抓到 0 筆，執行紀錄會印出實際欄位名稱。
3. 每個來源下載失敗會重試 3 次（櫃買中心的檔案約 5MB，曾發生連線中斷 IncompleteRead）。
4. 先讀入上一次的 `prices.json` 再覆蓋新抓到的價格，所以某個來源整個失敗時，那些股票會保留上次的價格，不會消失。
5. 存成 `prices.json` 並 commit。
6. **用 `GITHUB_TOKEN` 推送的 commit 不會觸發 GitHub Pages 重新發布**（2026/10/19 實際遇到：repository 的 prices.json 有更新，網站卻一直是舊的），所以最後一步用 `gh api -X POST repos/{repo}/pages/builds` 主動要求重新發布，workflow 需要 `pages: write` 權限。

`prices.json` 格式：
```json
{ "updated": "2026-10-19T09:07:00Z",
  "dates": { "twse": "2026-10-19", "tpex": "2026-10-19", "esb": "2026-10-19" },
  "p": { "0050": [108.4, "2026-10-19"] } }
```

需要的 GitHub 設定：Settings → Actions → General → Workflow permissions 選 **Read and write permissions**；Settings → Pages 選 main 分支 /(root)。

---

## 程式結構（index.html 內）

- `STOCK_DATA`：內建股票清單（代號｜名稱｜市場+類型｜產業索引）
- `calcFee` / `calcTax` / `V()`：手續費、證交稅、成交金額（無條件捨去）
- `inferDisc`：由實際手續費推算折扣
- `compute()`：依日期排序跑平均成本，算出持股、損益、股利
- `loadAuto` / `applyAuto`：讀 `prices.json` 並套用
- `vHome` / `vRecs` / `vSet`：三個分頁的畫面
- `openTx` / `recalc`：記一筆的對話框與即時計算
- `sample()`：第一次開啟時顯示的範例資料（會提示使用者清除）

---

## 改版注意事項

- 保持**單一 HTML 檔**。
- 保持 localStorage key 與資料格式相容。
- 改完要實際測試：手續費（例 0050 30 股 × 107.15 → 成交 3,214、手續費 1、付出 3,215；00631L 100 股 × 36.06 → 3,606、1、3,607）。
- 使用者是非工程背景，說明請用白話、步驟要具體。
