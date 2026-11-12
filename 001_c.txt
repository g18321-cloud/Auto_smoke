/**
 * ============================================================================
 * 行動端 AI 輔助目測判煙系統 (Mobile Ringelmann Assistant) - GAS 後端核心 (Code.gs)
 * ============================================================================
 * 功能包含：
 * 1. Web App 路由與行動端直式畫面 Viewport 設定
 * 2. Google Sheets 自動建表與參數外部化讀取 (系統參數表 / 固定源判煙紀錄 / 移動源烏賊車紀錄)
 * 3. Google Drive 自動路徑巡覽與建立 (預設路徑：GAS/smoke) 及證據照片存檔
 * ============================================================================
 */

const DEFAULT_SYSTEM_CONFIG = [
  ['參數名稱 (Key)', '參數數值 (Value)', '單位', '參數說明 (Description)'],
  ['DRIVE_FOLDER_PATH', 'GAS/smoke', '路徑', 'Google Drive 證據照片存放主路徑 (自動建立 GAS 與 smoke 資料夾)'],
  ['USE_DATE_SUBFOLDER', 'TRUE', '布林值', '是否於 GAS/smoke 下自動依日期建立 YYYYMMDD_判煙存證 子資料夾 (TRUE/FALSE)'],
  ['SUN_BACK_ANGLE_LIMIT', '70', '度 (°)', '固定源法規：背向太陽容許半角 (身後 140° 範圍即夾角 <= 70° 為順光合格)'],
  ['SUN_SIDE_ANGLE_LIMIT', '105', '度 (°)', '固定源法規：側光注意上限角度 (超過此角度判定為逆光違規)'],
  ['STATIONARY_INTERVAL_SEC', '15', '秒 (s)', '固定源連續判煙法規節拍秒數 (每 15 秒震動提醒紀錄一次)'],
  ['STATIONARY_TOTAL_COUNTS', '24', '筆', '固定源連續判煙預設總筆數 (24 筆 = 6 分鐘連續觀測)'],
  ['STATIONARY_OPACITY_WARN', '40', '%', '固定源不透光率警戒閾值 (預設 40% 對應林格曼 2 號黑煙)'],
  ['MOBILE_OPACITY_WARN', '30', '%', '移動源 (烏賊車) 瞬間黑煙不透光率警報觸發閾值 (%)'],
  ['MOBILE_BUFFER_SEC', '5', '秒 (s)', '移動源循環緩衝錄影保留秒數 (Pre-trigger Ring Buffer)'],
  ['MOBILE_BUFFER_FPS', '10', 'FPS', '移動源循環緩衝每秒取樣影格數'],
  ['WHITE_SMOKE_RATIO', '1.12', '倍率', '白煙 (水蒸氣) 反轉偵測閾值：煙羽亮度 / 背景亮度高於此倍率時提醒白煙'],
  ['RINGELMANN_G1_MIN', '15', '%', '林格曼 1 級不透光率下限 (標準 20%)'],
  ['RINGELMANN_G2_MIN', '35', '%', '林格曼 2 級不透光率下限 (標準 40%)'],
  ['RINGELMANN_G3_MIN', '55', '%', '林格曼 3 級不透光率下限 (標準 60%)'],
  ['RINGELMANN_G4_MIN', '75', '%', '林格曼 4 級不透光率下限 (標準 80%)'],
  ['RINGELMANN_G5_MIN', '95', '%', '林格曼 5 級不透光率下限 (標準 100%)']
];

const SHEET_NAMES = {
  CONFIG: '系統參數表',
  STATIONARY: '固定源判煙紀錄',
  MOBILE: '移動源烏賊車紀錄'
};

/**
 * GAS Web App HTTP GET 入口 (對齊 smartcard 原生直出架構，解除沙盒限制)
 */
function doGet(e) {
  return HtmlService.createHtmlOutputFromFile('index')
    .setTitle('行動端 AI 輔助目測判煙系統')
    .addMetaTag('viewport', 'width=device-width, initial-scale=1.0, maximum-scale=1.0, user-scalable=no, viewport-fit=cover')
    .setXFrameOptionsMode(HtmlService.XFrameOptionsMode.ALLOWALL);
}

/**
 * 取得或自動建立專案綁定的 Google Spreadsheet
 * 若為容器綁定腳本則直接使用 getActiveSpreadsheet()；
 * 若為獨立腳本則於 GAS/smoke 資料夾中自動建立或讀取已綁定之試算表。
 */
function getOrCreateSpreadsheet() {
  let ss = null;
  try {
    ss = SpreadsheetApp.getActiveSpreadsheet();
  } catch (err) {
    ss = null;
  }
  if (ss) return ss;

  const props = PropertiesService.getScriptProperties();
  const savedId = props.getProperty('SMOKE_SPREADSHEET_ID');
  if (savedId) {
    try {
      ss = SpreadsheetApp.openById(savedId);
      return ss;
    } catch (err) {
      // 若檔案已被刪除則重新建立
    }
  }

  // 在 Google Drive 的 GAS/smoke 資料夾下建立資料庫試算表
  const smokeFolder = getOrCreateDriveFolderByPath('GAS/smoke');
  ss = SpreadsheetApp.create('行動端AI輔助目測判煙系統_雲端資料庫');
  const file = DriveApp.getFileById(ss.getId());
  file.moveTo(smokeFolder);
  props.setProperty('SMOKE_SPREADSHEET_ID', ss.getId());
  return ss;
}

/**
 * 美化表頭列並凍結首列
 */
function styleHeaderRow_(sheet, colCount, bgColor) {
  const headerRange = sheet.getRange(1, 1, 1, colCount);
  headerRange.setBackground(bgColor)
    .setFontColor('#ffffff')
    .setFontWeight('bold')
    .setHorizontalAlignment('center');
  sheet.setFrozenRows(1);
}

/**
 * 確保三大工作表皆已初始化並具備表頭 (供手動執行或系統自動呼叫)
 */
function ensureSheetsInitialized() {
  const ss = getOrCreateSpreadsheet();

  // 1. 系統參數表
  let configSheet = ss.getSheetByName(SHEET_NAMES.CONFIG);
  if (!configSheet) {
    configSheet = ss.insertSheet(SHEET_NAMES.CONFIG);
    configSheet.getRange(1, 1, DEFAULT_SYSTEM_CONFIG.length, 4).setValues(DEFAULT_SYSTEM_CONFIG);
    styleHeaderRow_(configSheet, 4, '#0f172a');
    configSheet.setColumnWidth(1, 220);
    configSheet.setColumnWidth(2, 140);
    configSheet.setColumnWidth(3, 90);
    configSheet.setColumnWidth(4, 420);
  }

  // 2. 固定源判煙紀錄
  let statSheet = ss.getSheetByName(SHEET_NAMES.STATIONARY);
  if (!statSheet) {
    statSheet = ss.insertSheet(SHEET_NAMES.STATIONARY);
    const headers = [[
      '紀錄編號', '稽查時間', '污染源名稱/編號', '稽查備註',
      'GPS緯度', 'GPS經度', '鏡頭方位角(°)', '鏡頭仰角(°)',
      '太陽方位角(°)', '太陽仰角(°)', '太陽夾角(°)', '日照光線判定',
      '天空背景亮度(I_bg)', '煙羽採樣亮度(I_plume)', '實測不透光率(%)',
      'AI建議級數', '人工判定級數', '連續觀測筆數', '超標筆數(>=2級)',
      '最高級數', '平均不透光率(%)', '連續判煙明細(JSON)',
      '雲端照片連結', '雲端資料夾連結'
    ]];
    statSheet.getRange(1, 1, 1, headers[0].length).setValues(headers);
    styleHeaderRow_(statSheet, headers[0].length, '#0369a1');
  }

  // 3. 移動源烏賊車紀錄
  let mobSheet = ss.getSheetByName(SHEET_NAMES.MOBILE);
  if (!mobSheet) {
    mobSheet = ss.insertSheet(SHEET_NAMES.MOBILE);
    const headers = [[
      '紀錄編號', '捕捉時間', '車牌號碼/特徵', '稽查路段/備註',
      'GPS緯度', 'GPS經度', '鏡頭方位角(°)',
      '路面基準亮度(I_road)', '峰值黑煙亮度(I_peak)', '最大不透光率峰值(%)',
      '對應林格曼級數', '警戒閾值(%)', '合規判定結果',
      '三連拍合成圖連結', '車牌識別幀連結', '起煙瞬間幀連結', '最大黑煙峰值幀連結',
      '雲端資料夾連結'
    ]];
    mobSheet.getRange(1, 1, 1, headers[0].length).setValues(headers);
    styleHeaderRow_(mobSheet, headers[0].length, '#b45309');
  }

  // 移除預設空白的「工作表1」或「Sheet1」（若還有其他工作表存在）
  const defaultSheet = ss.getSheetByName('工作表1') || ss.getSheetByName('Sheet1');
  if (defaultSheet && ss.getSheets().length > 1) {
    try {
      ss.deleteSheet(defaultSheet);
    } catch (e) {}
  }

  Logger.log('✅ Google Sheets 三大工作表初始化完成: ' + ss.getUrl());
  return { ss, configSheet, statSheet, mobSheet };
}

/**
 * 依據路徑字串 (例如 'GAS/smoke' 或 'GAS\\smoke') 遞迴取得或建立 Google Drive 資料夾 (供手動執行或系統自動呼叫)
 */
function getOrCreateDriveFolderByPath(folderPathStr) {
  const cleanPath = String(folderPathStr || 'GAS/smoke')
    .replace(/\\/g, '/')
    .split('/')
    .map(p => p.trim())
    .filter(Boolean);

  let currentFolder = DriveApp.getRootFolder();
  for (let i = 0; i < cleanPath.length; i++) {
    const segName = cleanPath[i];
    const iter = currentFolder.getFoldersByName(segName);
    if (iter.hasNext()) {
      currentFolder = iter.next();
    } else {
      currentFolder = currentFolder.createFolder(segName);
    }
  }
  Logger.log('📁 Google Drive 路徑就緒: ' + folderPathStr + ' (' + currentFolder.getUrl() + ')');
  return currentFolder;
}

/**
 * [GAS 編輯器快捷執行函式] 一鍵初始化 Google Sheets 資料庫與 Google Drive (GAS/smoke)
 */
function initSystemEnvironment() {
  const folder = getOrCreateDriveFolderByPath('GAS/smoke');
  const res = ensureSheetsInitialized();
  Logger.log('🎉 系統環境初始化完畢！');
  Logger.log('試算表網址: ' + res.ss.getUrl());
  Logger.log('Drive 照片路徑 (GAS/smoke): ' + folder.getUrl());
  return {
    sheetUrl: res.ss.getUrl(),
    folderUrl: folder.getUrl()
  };
}

/**
 * 取得照片實際存放之目標資料夾 (預設 GAS/smoke，並支援日期子資料夾)
 */
function getTargetPhotoFolder_(basePath, useDateSubfolder) {
  const mainFolder = getOrCreateDriveFolderByPath(basePath || 'GAS/smoke');
  if (!useDateSubfolder) {
    return mainFolder;
  }
  const tz = Session.getScriptTimeZone() || 'Asia/Taipei';
  const dateStr = Utilities.formatDate(new Date(), tz, 'yyyyMMdd');
  const subName = dateStr + '_判煙存證';
  const iter = mainFolder.getFoldersByName(subName);
  if (iter.hasNext()) {
    return iter.next();
  }
  return mainFolder.createFolder(subName);
}

/**
 * 將前端傳入之 DataURL (Base64 JPEG/PNG) 轉存至指定 Google Drive 資料夾
 */
function saveBase64ImageToDrive_(dataUrl, fileName, folder) {
  if (!dataUrl || typeof dataUrl !== 'string' || !dataUrl.includes('base64,')) {
    return '';
  }
  const parts = dataUrl.split('base64,');
  const mimeMatch = parts[0].match(/data:(.*?);/);
  const mimeType = mimeMatch ? mimeMatch[1] : 'image/jpeg';
  const byteCharacters = Utilities.base64Decode(parts[1]);
  const blob = Utilities.newBlob(byteCharacters, mimeType, fileName);
  const file = folder.createFile(blob);
  try {
    file.setSharing(DriveApp.Access.ANYONE_WITH_LINK, DriveApp.Permission.VIEW);
  } catch (e) {
    // 若網域政策限制公開分享則保留預設權限
  }
  return file.getUrl();
}

/**
 * [前端 API 1] 讀取 Google Sheets 系統參數表與雲端連線資訊
 */
function getConfigFromSheet() {
  const { ss, configSheet } = ensureSheetsInitialized();
  const data = configSheet.getDataRange().getValues();
  const configMap = {};

  for (let i = 1; i < data.length; i++) {
    const key = String(data[i][0] || '').trim();
    const rawVal = data[i][1];
    if (!key) continue;

    if (String(rawVal).toUpperCase() === 'TRUE') {
      configMap[key] = true;
    } else if (String(rawVal).toUpperCase() === 'FALSE') {
      configMap[key] = false;
    } else if (rawVal !== '' && !isNaN(Number(rawVal))) {
      configMap[key] = Number(rawVal);
    } else {
      configMap[key] = String(rawVal).trim();
    }
  }

  // 確保 GAS/smoke 資料夾預先就緒
  const folderPath = configMap.DRIVE_FOLDER_PATH || 'GAS/smoke';
  const mainFolder = getOrCreateDriveFolderByPath(folderPath);

  return {
    ok: true,
    config: configMap,
    sheetUrl: ss.getUrl(),
    sheetName: ss.getName(),
    driveFolderPath: folderPath,
    driveFolderUrl: mainFolder.getUrl(),
    serverTime: Utilities.formatDate(new Date(), Session.getScriptTimeZone() || 'Asia/Taipei', 'yyyy-MM-dd HH:mm:ss')
  };
}

/**
 * [前端 API 2] 儲存判煙稽查紀錄至 Google Sheets 並上傳照片至 Google Drive (GAS/smoke)
 */
function saveInspectionRecord(payload) {
  if (!payload || typeof payload !== 'object') {
    throw new Error('無效的判煙紀錄資料格式');
  }

  const { ss, configSheet, statSheet, mobSheet } = ensureSheetsInitialized();
  const cfgRes = getConfigFromSheet();
  const cfg = cfgRes.config || {};

  const folderPath = cfg.DRIVE_FOLDER_PATH || 'GAS/smoke';
  const useDateSub = cfg.USE_DATE_SUBFOLDER !== false;
  const targetFolder = getTargetPhotoFolder_(folderPath, useDateSub);
  const folderUrl = targetFolder.getUrl();

  const tz = Session.getScriptTimeZone() || 'Asia/Taipei';
  const now = new Date();
  const timestampStr = payload.timestamp || Utilities.formatDate(now, tz, 'yyyy-MM-dd HH:mm:ss');
  const fileTimeTag = Utilities.formatDate(now, tz, 'yyyyMMdd_HHmmss');
  const recordId = payload.id || ('SMK-' + fileTimeTag + '-' + Math.floor(100 + Math.random() * 900));

  if (payload.mode === 'STATIONARY') {
    // 固定污染源判煙紀錄與照片上傳
    const sourceName = payload.sourceName || '未命名煙囪';
    const safeName = sourceName.replace(/[\\/:*?"<>|]/g, '_');
    const photoFileName = `${fileTimeTag}_固定源_${safeName}_${payload.finalGrade ?? payload.aiGrade}級.jpg`;
    const photoUrl = saveBase64ImageToDrive_(payload.photoDataUrl, photoFileName, targetFolder);

    const row = [
      recordId,
      timestampStr,
      sourceName,
      payload.note || '',
      payload.lat ?? '',
      payload.lng ?? '',
      payload.heading ?? '',
      payload.pitch ?? '',
      payload.sunAzimuth ?? '',
      payload.sunAltitude ?? '',
      payload.sunDiffAngle ?? '',
      payload.sunStatusText || '',
      payload.skyLuma ?? '',
      payload.plumeLuma ?? '',
      payload.opacity ?? '',
      payload.aiGrade ?? '',
      payload.finalGrade ?? '',
      payload.totalCounts ?? 0,
      payload.exceedCounts ?? 0,
      payload.maxGrade ?? 0,
      payload.avgOpacity ?? 0,
      JSON.stringify(payload.readings || []),
      photoUrl,
      folderUrl
    ];
    statSheet.appendRow(row);

    return {
      ok: true,
      mode: 'STATIONARY',
      recordId: recordId,
      photoUrl: photoUrl,
      folderUrl: folderUrl,
      sheetUrl: ss.getUrl(),
      savedAt: timestampStr
    };
  } else {
    // 移動污染源 (烏賊車) 單張存證照片上傳 (相容舊版三連拍欄位)
    const plate = payload.plateNumber || '待辨識車牌';
    const safePlate = plate.replace(/[\\/:*?"<>|]/g, '_');

    const photoUrl = saveBase64ImageToDrive_(
      payload.photoDataUrl || payload.collageDataUrl,
      `${fileTimeTag}_烏賊車存證_${safePlate}_${payload.peakOpacity}pct.jpg`,
      targetFolder
    );
    const frame1Url = payload.frame1DataUrl ? saveBase64ImageToDrive_(payload.frame1DataUrl, `${fileTimeTag}_1車牌幀_${safePlate}.jpg`, targetFolder) : '';
    const frame2Url = payload.frame2DataUrl ? saveBase64ImageToDrive_(payload.frame2DataUrl, `${fileTimeTag}_2起煙幀_${safePlate}.jpg`, targetFolder) : '';
    const frame3Url = payload.frame3DataUrl ? saveBase64ImageToDrive_(payload.frame3DataUrl, `${fileTimeTag}_3峰值幀_${safePlate}_${payload.peakOpacity}pct.jpg`, targetFolder) : '';

    const row = [
      recordId,
      timestampStr,
      plate,
      payload.locationNote || '',
      payload.lat ?? '',
      payload.lng ?? '',
      payload.heading ?? '',
      payload.roadBaselineLuma ?? '',
      payload.peakLuma ?? '',
      payload.peakOpacity ?? '',
      payload.ringelmannGrade ?? '',
      payload.warnThreshold ?? 30,
      payload.isExceed ? '超標舉發' : '未超標',
      photoUrl,
      frame1Url,
      frame2Url,
      frame3Url,
      folderUrl
    ];
    mobSheet.appendRow(row);

    return {
      ok: true,
      mode: 'MOBILE',
      recordId: recordId,
      photoUrl: photoUrl,
      frameUrls: [frame1Url, frame2Url, frame3Url].filter(Boolean),
      folderUrl: folderUrl,
      sheetUrl: ss.getUrl(),
      savedAt: timestampStr
    };
  }
}

/**
 * [前端 API 3] 讀取最近 N 筆雲端稽查紀錄供手機端抽屜檢視
 */
function getRecentCloudRecords(limit) {
  const maxRows = Number(limit) || 15;
  const { ss, statSheet, mobSheet } = ensureSheetsInitialized();
  const records = [];

  // 讀取固定源
  const statLast = statSheet.getLastRow();
  if (statLast > 1) {
    const startRow = Math.max(2, statLast - maxRows + 1);
    const numRows = statLast - startRow + 1;
    const rows = statSheet.getRange(startRow, 1, numRows, 24).getValues();
    rows.forEach(r => {
      records.push({
        id: r[0],
        timestamp: String(r[1]),
        mode: 'STATIONARY',
        title: r[2] || '固定源煙囪',
        note: r[3] || '',
        lat: r[4],
        lng: r[5],
        opacity: r[14],
        grade: r[16] !== '' ? r[16] : r[15],
        sunStatusText: r[11],
        photoUrl: r[22],
        folderUrl: r[23]
      });
    });
  }

  // 讀取移動源
  const mobLast = mobSheet.getLastRow();
  if (mobLast > 1) {
    const startRow = Math.max(2, mobLast - maxRows + 1);
    const numRows = mobLast - startRow + 1;
    const rows = mobSheet.getRange(startRow, 1, numRows, 18).getValues();
    rows.forEach(r => {
      records.push({
        id: r[0],
        timestamp: String(r[1]),
        mode: 'MOBILE',
        title: r[2] || '烏賊車蒐證',
        note: r[3] || '',
        lat: r[4],
        lng: r[5],
        opacity: r[9],
        grade: r[10],
        statusText: r[12],
        photoUrl: r[13],
        folderUrl: r[17]
      });
    });
  }

  records.sort((a, b) => (a.timestamp < b.timestamp ? 1 : -1));
  return {
    ok: true,
    sheetUrl: ss.getUrl(),
    records: records.slice(0, maxRows)
  };
}
