# BleTest.cs 與 callConnect.cs 修改詳細紀錄總結

本文檔詳細記錄了在 Windows PC / Windows Editor 環境下，針對藍芽 BLE 連線系統 (`BleTest.cs`) 與 UI 連線控制橋接器 (`callConnect.cs`) 所進行的所有修改、新增與刪除內容，以及相應的設計原因。

---

## 核心修改背景與目標

1. **支援雙裝置（Dual BLE Devices）同時連線與切換**：原有的 BLE 架構僅能單一裝置連線，現已擴充支援雙藍芽裝置（如主感測器與握力器 `IHPSS_v` / `Grip_`）同時掃描、連線與數據讀取。
2. **MAC 位址白名單機制（whitemac.txt）**：避免周邊無關藍芽裝置干擾，提供動態載入、動態寫入與預備白名單註解功能。
3. **跨場景單例生命週期管理與動態綁定**：確保切換 Unity 場景（如 `Main`, `Main2`, `Level1` ~ `Level5`, `Gait_analysis`）時 BLE 連線不中斷，並自動重新綁定當前場景的 Manager/UI。
4. **卡爾曼濾波（Kalman Filter）雙路獨立平滑化處理**：對裝置一與裝置二接收到的 7 通道 Raw Data 分別進行即時平滑化與單位轉換。
5. **UI 狀態與多國語言（繁中/台語/日語/英語）同步**：支援更完整的斷線判斷、分頁開閉與裝置連接數量提示。

---

## 一、 BleTest.cs 修改細節

### 1. 刪除了什麼？（What was deleted?）與刪除原因（Why?）

| 刪除/註解項目 | 刪除原因 (Why) |
| :--- | :--- |
| **舊有單一裝置連線邏輯** | 原程式僅支援單一裝置 (`deviceId`, `ble`)，無法因應遊戲中需同時對應雙感測器（如雙腳或握力器）的需求。已全數重構為雙路併行機制。 |
| ** Update 內舊有直接單筆 Upload Data 寫入** (`//gamemanagers.GetComponent<apiPreUploadData>().rawDataStringList.Add(rawString);`) | 原先在 `Update` 每一幀直接寫入 API 佇列容易造成主執行緒塞車或不同步，改為透過獨立緩衝區 `rawDataStringListBuf1` / `rawDataStringListBuf2` 在 `ReadBleData` 執行緒中收集，再批次寫入。 |
| **硬編碼單一設備掃描結束條件** | 原先掃描到單一裝置即結束，無法同時搜尋並配對第二台裝置；現調整為同時等待裝置 1 與裝置 2 分別判定。 |

---

### 2. 修改了什麼？（What was modified?）與修改原因（Why?）

| 修改項目 | 修改內容與細節 | 修改原因 (Why) |
| :--- | :--- | :--- |
| **`Start()` 函數與場景切換銷毀邏輯** | 增加 `GameObject.FindGameObjectsWithTag("BLE")` 檢查。若已存在舊有的 BLE 物件，則呼叫其 `onNewScene()` 並將新場景產生的重複 BLE 實體銷毀 (`Destroy(this.gameObject.transform.parent.gameObject)`)。 | 實現 DontDestroyOnLoad 單例模式，防止切換場景時產生重複的 BLE 管理器，並保持跨場景的藍芽連線不中斷。 |
| **`onNewScene()` 跨場景動態綁定** | 擴充 `switch (currentSceneName)`，涵蓋 `Main`, `Main2`, `Level1`~`Level5`, `Level5_`, `Gait_analysis` 等所有場景。 | 切換場景時自動將 BLE 實體綁定至該場景的 `callConnect`, `Practise`, `Kick`, `M_Tiptoe`, `BoatMove`, `test`, `H_Char`, `gaitAnalysisProcess` 等組件。 |
| **`ScanBleDevices()` 裝置比對與配對** | 改為比對名稱是否包含 `targetDeviceName` (`IHPSS_v`) 或 `targetDeviceNameB` (`Grip_`)，並分別賦值給 `deviceId` 與 `deviceId2`。 | 支援彈性識別不同類型的硬體裝置，且避免將同一台裝置重複設定給 Device 1 與 Device 2。 |
| **`ConnectBleDevice()` 執行緒連線** | 根據 `isDevice1Grip` / `isDevice2Grip` 布林值，動態挑選 Service UUID (`serviceUuid` 或 `serviceUuidB`) 及 Characteristic UUID。 | 握力器 (`Grip_`) 與標準感測器 (`IHPSS_v`) 使用不同的 UUID 服務通道，必須動態匹配才能成功建立 GATT 服務連線。 |
| **`ReadBleData()` 數據接收與處理解析** | 使用 `BLE.ReadBytesAndDeviceId()` 同時取得 Byte 陣列與發送端的 Device ID，並比對 `deviceIdPure` 與 `deviceIdPure2` 進行雙通道卡爾曼過濾。 | 解決多裝置共享同一讀取執行緒時數據混淆的問題，確保裝置 1 與裝置 2 的數據獨立過濾與緩衝。 |
| **`CleanUp()` 安全釋放** | 新增對 `ble2.Close()` 及 `connectionThread2.Abort()` 的清理邏輯。 | 防止 Unity Editor 或關閉 EXE 遊戲時，第二條藍芽連線執行緒殘留導致 Unity 當機或藍芽 Port 被鎖死。 |

---

### 3. 新增了什麼？（What was added?）與新增原因（Why?）

| 新增項目 | 新增內容與細節 | 新增原因 (Why) |
| :--- | :--- | :--- |
| **雙藍芽控制器與過濾器變數** | 新增 `ble2`, `connectionThread2`, `deviceId2`, `deviceIdPure2`, `bleValueInts2`, 以及 7 組卡爾曼濾波器 (`kffb1`~`kffb7`)。 | 為第二台 BLE 裝置提供獨立的連線控制與訊號濾波降噪通道。 |
| **裝置類型標記** | 新增 `isDevice1Grip` 與 `isDevice2Grip` 布林標記。 | 自動區分連接的是一般感測器還是握力器，進而決定 UUID 通道。 |
| **`nowMainDevice` 與 `changeMainDevice()`** | 新增主裝置切換變數與 API。 | 當遊戲僅需參考其中一台裝置作為主動作來源時（例如主要傳輸給遊戲控制器的 Value），可動態切換裝置 0 或裝置 1。 |
| **MAC 白名單管理功能** (`loadWhiteListMac`, `writeWhitemacInputfield`, `writeWhitelistPre`) | 讀寫 `whitemac.txt` 檔案，支援過濾 `, - :` 格式，並提供自動預寫入 `\\mac` 功能。 | 允許現場部署時手動設定允許連線的 MAC 地址，避免展場環境中鄰近其他藍芽設備干擾。 |
| **UI 數據顯示擴充** | 在 `Update()` 中擴充 `TextShowBleValue`，同時顯示 Raw Data、Kalman 濾波值與轉換後的物理數值。 | 方便開發與除錯人員在 UI 上即時檢視兩台裝置的訊號品質與轉換數值。 |

---

## 二、 callConnect.cs 修改細節

### 1. 刪除了什麼？（What was deleted?）與刪除原因（Why?）

| 刪除/註解項目 | 刪除原因 (Why) |
| :--- | :--- |
| **`startConnect()` 中 `Connect_Switch` 的 Toggle 檢查** (`// if (Connect_Switch.GetComponent<Toggle>().isOn)...`) | 舊邏輯使用 UI Toggle 開關控制連線狀態，容易造成 UI 開關與實際藍芽連線狀態不同步。刪除該判斷後，改為點擊直接觸發連線頁面開啟 (`connectPage[page].SetActive(true)`)。 |
| **`Start()` 中強制關閉 connectPage 的迴圈** | 防止 Unity 初始化時關閉潛在需要的 UI 頁面，改由場景加載後依連線狀態動態控制。 |

---

### 2. 修改了什麼？（What was modified?）與修改原因（Why?）

| 修改項目 | 修改內容與細節 | 修改原因 (Why) |
| :--- | :--- | :--- |
| **`Update()` 內的 Windows 平台狀態監控** | 分別針對 Windows (`BLEss`) 與 Mobile (`BLEs`) 做平台分流，依據 `BLEss.findTargetDeviceCount` (0/1/2) 控制 `scaningFind1Obj` / `scaningFind2Obj` / `SearchPage` / `done1Page` / `done2Page` 的顯示狀態。 | 讓 UI 搜尋畫面能精確反應目前 PC 端已找到的裝置數量 (1 台還是 2 台)，提升使用者體驗。 |
| **`closeConnect()` 多國語言 UI 狀態更新** | 透過 `PlayerPrefs.GetString("Language")` 判斷傳統中文、台語、日語、英語，動態設定 `connectStatus.text` ("已連接"/"未連接"/"接続済み"/"Not connect") 與 `connectedNum.text` ("X個裝置"/"X台の裝置"/"X devices")。 | 滿足多國語言版本的需求，讓連線狀態文字符合當前語系設定。 |
| **`closeConnect()` 遮罩按鈕與警告頁面控制** | 依據 `connectedNum` (0/1/2) 動態設置 `warnWhenCon0`, `warnWhenCon1` 與 `btNotConCoverBtn_1`, `btNotConCoverBtn_2` 的顯示/隱藏。 | 防止玩家在感測器數量未達標時（例如只連線 1 台但遊戲需要 2 台）誤按開始遊戲按鈕。 |
| **`disconnect()` 斷線處理函式** | 新增斷線判斷與註解 `// 【斷線判斷點 3：使用者手動點擊 UI 畫面上的 disconnect 按鈕】`，調用 `BLEmobile.OnUserClickDisconnect()` 或顯示連線按鈕。 | 規範斷線流程，確保點擊斷線按鈕後正確清空狀態並還原 UI 介面。 |

---

### 3. 新增了什麼？（What was added?）與新增原因（Why?）

| 新增項目 | 新增內容與細節 | 新增原因 (Why) |
| :--- | :--- | :--- |
| **`setDonePageClosed(int index)`** | 新增 `done1PageClosed` 與 `done2PageClosed` 標記位元控制。 | 允許使用者手動關閉「已連線 1 個」或「已連線 2 個」的彈出通知頁面後，不會在 `Update()` 每幀被重新強制開啟。 |
| **`saveBTwhitelist()`** | 新增對 `BLEtestobjects` 的 `writeWhitemacInputfield()` 呼叫。 | 供 UI 介面上的「儲存白名單」按鈕綁定，將使用者在 InputField 輸入的 MAC 地址保存至 `whitemac.txt`。 |
| **斷線狀態標記 `disconnecting`** | 新增 `disconnecting` 布林標記。 | 在執行斷線操作時，暫停 `closeConnect()` 自動隱藏頁面的邏輯，防止 UI 發生閃爍或狀態衝突。 |

---

## 三、 系統整體運作流程圖解 (Architecture Overview)

```
[UI / callConnect.cs]
       │
       ├─► 1. 點擊掃描 ──► BleTest.StartScanHandler() ──► 啟動 ScanBleDevices 執行緒
       │                                                         │ (比對 whitemac.txt)
       │                                                         ▼
       │                                                找到 1 或 2 台裝置
       │                                                         │
       ├─► 2. 點擊連線 ──► BleTest.StartConHandler()  ──► 啟動 ConnectBleDevice 執行緒
       │                                                         │ (區分 IHPSS_v / Grip_ UUID)
       │                                                         ▼
       │                                                雙路 BLE 成功連線
       │                                                         │
       ├─► 3. 讀取數據 ◄── ReadBleData() 執行緒 ◄─────────────────┘
       │                        │
       │                        ├──► 雙路獨立 KalmanFilter 濾波 (kff1~7 / kffb1~7)
       │                        └──► 寫入 rawDataStringListBuf1 / Buf2 (存 CSV / API)
       │
       └─► 4. 場景切換 ──► DontDestroyOnLoad 保留 BleTest 單例
                                └──► 新場景呼叫 onNewScene() 自動重新綁定 Manager
```

---
*記錄時間：2026-09-02*
