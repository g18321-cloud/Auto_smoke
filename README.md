# 行動端 AI 輔助目測判煙系統 (Mobile Ringelmann Assistant)

一套專為智慧型手機直式全螢幕打造的「固定污染源（工廠煙囪）」與「移動污染源（烏賊車/柴油車）」目測判煙與蒐證系統。採用 **前後端解耦雙棲架構**（GitHub Pages 獨立前端 + Google Apps Script 雲端後端），支援手機即時後置鏡頭取景、感測器太陽合規檢核、動態不透光率測算、法規存證照片壓印，並具備 API 通行碼防濫用驗證與 Google Drive / Google Sheets 自動化存證歸檔。

---

## 🌐 線上正式體驗 (GitHub Pages)

👉 **正式網址**：[https://g18321-cloud.github.io/Auto_smoke/](https://g18321-cloud.github.io/Auto_smoke/)

*使用智慧型手機 Chrome 瀏覽器開啟，即可直接在網頁畫面中啟動後置鏡頭即時取景，享受 100% 全功能操作體驗！*

---

## 🌟 核心特色功能

1. **固定源連續判煙（煙囪）**：
   - **太陽合規檢核**：天文演算法結合手機電子羅盤與 GPS，即時檢核是否處於背向太陽 140° 合規順光範圍（🟢 順光 / 🟡 側光 / 🔴 逆光）。
   - **虛擬林格曼比色卡**：0～5 級標準網格與純灰階色塊一鍵切換。
   - **雙點採樣準星**：🔴 煙羽 ROI vs. 🔵 天空 ROI，即時計算灰階對比並自動提醒白煙（水蒸氣亮度反轉）。
   - **15 秒法規節拍計時器**：震動提醒記錄 24 筆連續觀測序列。

2. **移動源目測判煙（烏賊車）**：
   - **雙對照框精準比對**：🔴 車輛黑煙框 vs. 🔵 無煙基準框，精巧尺寸設計（支援 小/中/大 調整），避免框入車體。
   - **邊緣觸發與 5 秒冷卻防抖**：黑煙超標瞬間震動一次提醒，絕不持續高頻狂震。
   - **單張現場照片存證**：定格或拍照後直接壓印完整法規浮水印並上傳至雲端。

3. **雙模相機引擎（即時串流 + 原生快拍降級）**：
   - **WebRTC 即時取景**：對齊 `smartcard` 成功架構，內建三階平滑降級與互斥鎖防護，支援取景框中央一鍵手勢喚起鏡頭。
   - **原生相機快拍**：支援 HTML5 `capture="environment"` 原生後置相機快拍降級。

4. **🔒 資安授權防護門禁（防惡意調用與灌水）**：
   - **【對策 1：通行碼驗證 (API Secret Token)】**：後端嚴格比對 `apiToken`，符合系統參數表 `API_SECRET_KEY` 才允許寫入雲端，徹底防範外部惡意洗版。
   - **【對策 2：檔案大小安全檢查 (Payload Gate)】**：單張存證照片上限 4.5MB（約 3.3MB JPEG），超過自動阻擋以保護 Google Drive 空間。

5. **獨立「🎓 模擬教學」分頁**：
   - 卡通煙囪動態煙羽與柴油烏賊車行駛噴煙動畫，支援滑桿自由調控 0～100% 濃度，供人員培訓與室內演練，不干擾主視窗現場實測。

6. **雙棲雲端儲存架構（GAS + Drive + Sheets）**：
   - **Google Sheets**：自動初始化並寫入 `系統參數表`、`固定源判煙紀錄` 與 `移動源烏賊車紀錄`。
   - **Google Drive**：照片自動遞迴存檔至 `GAS/smoke/YYYYMMDD_判煙存證`。

---

## 📂 檔案目錄結構

```text
Auto_smoke/
├── 專案規劃書.md          # 系統詳細功能需求與規格 (本機保留，不上傳 GitHub)
├── MODLOG.md             # 完整開發日誌與異動紀錄 (本機保留，不上傳 GitHub)
├── AGENTS.md             # AI 團隊角色與協作職責分工 (本機保留，不上傳 GitHub)
├── README.md             # 專案公開說明與操作手冊
├── index.html            # 行動端前端主程式 (GitHub Pages 首頁)
├── Code.gs               # Google Apps Script 後端 (Sheets 讀寫、Drive 歸檔、REST API)
├── appsscript.json       # GAS 專案組態設定清單
├── 001_c.txt             # Code.gs 備份副本
└── 001_i.txt             # index.html 備份副本
```

---

## 🚀 雲端後端 (GAS) 部署指南

1. 開啟 [Google Apps Script 編輯器](https://script.google.com/) 並新增專案。
2. 建立 `Code.gs` 並貼入本專案之 `Code.gs` 代碼。
3. 建立 `index.html` 並貼入本專案之 `index.html` 代碼。
4. 在 GAS 編輯器選取執行 `initSystemEnvironment()` 函式，一鍵完成 Google Sheets 三大工作表初始化與 Google Drive `GAS/smoke` 資料夾建立。
5. 點擊右上角 **「部署」 → 「管理部署作業」 → 類型選「網頁應用程式」**：
   - **執行身分**：我
   - **誰可以存取**：所有人 (Anyone)
6. 取得 Web App 網址後，在手機端開啟 GitHub Pages（或 GAS 頁面），進入右上角「⚙️ 系統設定」填入 GAS 網址與通行碼即可開始雲端存證！
