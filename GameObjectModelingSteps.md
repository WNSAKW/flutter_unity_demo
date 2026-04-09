# 關節炎數位介入遊戲化APP開發專案

- **Topic**：遊戲物件建模步驟紀錄
- **Author**：AW'z
- **Update**：2026/04/09 Wed.
- **Version**：v1.0

---

## 目錄
- [角色建模流程](#角色建模流程)
- [小物件建模流程](#小物件建模流程)
- [檔案命名規則](#檔案命名規則)
- [資料夾結構建議](#資料夾結構建議)
- [Task List](#TaskList)
- [Version Record](#VersionRecord)

---

## 角色建模流程

### 1. 產出平面3D參考圖

**工具**：Google Gemini  
**目的**：產生清晰的 T-pose 3D 風格角色圖

#### Prompt 版本

**Prompt_ver.01（Basic Ver.）**
```bash
一隻3D的[動物角色]，可愛風格，圓潤可愛，大眼睛，細節豐富
背景：純白色背景，什麼東西都不要
角色呈現標準 T-pose，正面視角，全身完整顯示
高品質，乾淨，適合用來生成3D模型
```

**使用建議**：
- 將 `[動物角色]` 替換為具體名稱，例如：可愛兔子、胖胖狐狸、小熊等
- 可加入風格關鍵字：Q版、卡通、黏土風、迪士尼風等
- 強烈建議加上「純白色背景」與「T-pose」
- 人物參考風格：party animal、動物森友會

---

### 2. 產出3D模型

**工具**：Tripo.ai  
**輸入**：Gemini 產出的參考圖

**步驟**：
1. 進入 Tripo.ai，上傳 Gemini 生成的圖片
2. 調整產出設定（建議參數待持續更新）
3. 點擊 Generate
4. 產出後預覽模型，確認無明顯破面或畸形
5. Export 格式選擇：**FBX (for Mixamo)**
6. 下載後存入本地端

**目前推薦設定**：待更新  
**常見問題處理**：待補充

**存放位置**：`Models/Characters/Raw/`

---

### 3. 產出 Rig 與動畫

**工具**：Adobe Mixamo

**詳細步驟**：
1. 進入 [Mixamo](https://www.mixamo.com/)
2. 上傳 Tripo.ai 輸出的 FBX 檔案
3. **角度確認**：確保角色正面正確（可旋轉調整）
4. **Skeleton 調整**：
   - 檢查並微調骨骼位置（特別注意頭部、手掌、腳掌比例）
   - 確認手臂與腿部骨骼長度合理
5. 選擇想要的動作（可先下載 Idle / Walk / Jump 等基礎動作測試）
6. 設定動畫參數（Frame Rate、Skin 等）
7. 下載帶有 Rig 的模型與動畫（格式：**FBX**）

**下載選項建議**：
- Format: FBX Binary (.fbx)
- Skin: With Skin
- Frames: 完整動作長度

**存放位置**：`Models/Characters/Rigged/`

---

## 小物件建模流程

**目前狀態**：流程開發中

### 預計標準流程

1. **概念圖產生**  
   - 使用 Gemini 產生多角度參考圖
   - 建議包含：正面、側面、45度角、細節特寫

2. **3D模型生成**  
   - 主要工具：Tripo.ai（單張圖）
   - 也可使用 Blender + AI 插件輔助建模

3. **模型優化**（如需要）  
   - 使用 Blender 進行：
     - 修復破面
     - 調整拓樸（Topology）
     - 簡化面數
     - UV 展開與貼圖製作

4. **匯出格式**
   - 建議格式：**FBX**（含貼圖）或 **GLB**（適合即時引擎）

**存放位置**：`Models/Props/`

---

## 檔案命名規則

**建議統一命名格式**：

- **角色**：
  - Raw 模型：`Character_[動物名稱]_Raw_v01.fbx`
  - Rigged 模型：`Character_[動物名稱]_Rigged_[動作名稱]_v01.fbx`

- **小物件**：
  - `Prop_[物件名稱]_v01.fbx`
  - `Prop_[物件名稱]_Texture_v01.png`

**範例**：
- `Character_Rabbit_Raw_v01.fbx`
- `Character_Fox_Rigged_Idle_v02.fbx`
- `Prop_Mushroom_v01.fbx`

---

## 資料夾結構建議
```bash
ProjectName/
├── Models/
│   ├── Characters/
│   │   ├── Raw/           ← Tripo.ai 原始輸出
│   │   └── Rigged/        ← Mixamo 帶 rig 與動畫
│   └── Props/             ← 所有小物件
├── Reference/
│   └── Images/            ← Gemini 等產出的參考圖
├── Animations/            ← 額外動作檔案（可選）
└── Documents/
    └── GameObjectModelingSteps.md
```
---

## TaskList
- [ ] 補充 Tripo.ai 最佳產出參數與經驗
- [ ] 建立小物件完整建模流程與推薦工具
- [ ] 整理常用角色 Prompt 模板庫
- [ ] 記錄各動物角色在 Mixamo 的 Skeleton 調整技巧
- [ ] 測試不同 AI 工具在小物件生成的效果
- [ ] 決定遊戲引擎（Unity / Unreal / Godot）後調整匯出設定

---

## VersionRecord

| Date       | Version | Update Contents | Author   |
|------------|------|------------------------------------|--------|
| 2026/04/09 | v1.0 | 建立完整文件，包含流程、命名規則與資料夾結構 | AW'z  |

---

此文件將作為專案中所有遊戲物件建模的標準作業流程（SOP）參考。

如有流程調整或新發現，請隨時更新此文件並修改版本號。

---

**Notice**：  
保持文件更新，能讓後續開發與團隊合作更加順暢！
