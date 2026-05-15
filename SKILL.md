---
name: bug-diagnosis-landmark
description: |
  地標爭霸（Steam 2P 對戰）專用 Bug 診斷 Skill。
  當開發者收到玩家回報的 Bug ZIP 包，或貼入 anomaly.json / report.json / metadata.txt
  等診斷資料時，立即觸發本 Skill。涵蓋：ZIP 內容解析、Anomaly 事件分類、
  因果鏈重建、Root Cause 定位、修法方向輸出（含優先度評分）。
  適用關鍵詞：bug report、F8、anomaly.json、ListenerImbalance、InvalidTransition、
  DuplicateWrite、desync、商店卡死、UI 錯位、listener leak、地標爭霸 debug。
  任何涉及地標爭霸遊戲異常、診斷資料分析的需求，都應觸發本 Skill。
---

# Bug Diagnosis Skill — 地標爭霸 v1.4

> 本 Skill 對應規格文件：BugReport_Unified_Spec_v1_4.md  
> 不呼叫任何外部 API，所有分析在 context 內完成。

---

## 一、輸入格式識別

開發者可能提供以下任一形式，Skill 均需支援：

| 輸入形式 | 處理方式 |
|---|---|
| ZIP 檔路徑（已解壓） | 讀取目錄下所有檔案 |
| 貼入 anomaly.json 內容 | 直接解析 JSON |
| 貼入 metadata.txt 內容 | 提取 matchId、playerId、timestamp |
| 純文字 bug 描述 | 轉為結構化問題假設，列出需補充資料 |
| 多份 ZIP（雙方視角） | 執行 MatchCorrelate 路徑（需 matchId 一致）|

**若輸入不完整**：明確列出「缺少哪些欄位」，嚴禁補完不存在的內容。

---

## 二、分析流程（Pipeline）

```
Step 1: PARSE      → 提取所有欄位，建立內部資料結構
Step 2: CLASSIFY   → 依 eventType 分 bucket
Step 3: CHAIN      → 重建因果鏈（讀取 causalChains 或手動推斷）
Step 4: ROOT CAUSE → 定位主要 root cause
Step 5: REPORT     → 輸出結構化診斷摘要 + 修法方向
```

每個 Step 的詳細規則見下方對應章節。

---

## 三、Step 1：PARSE — 欄位提取

### 必讀檔案優先順序

```
1. metadata.txt       ← 快速摘要，先讀
2. anomaly.json       ← 核心診斷資料
3. report.json        ← 玩家描述 + 系統資訊
4. match.json         ← matchId + stateFingerprints
5. player.log（尾段）  ← 補充 stack trace
```

### 提取目標欄位

**metadata.txt**
- matchId、playerId、opponentId
- timestamp（ZIP 生成時間）
- gameVersion

**anomaly.json**
- `events[]`：完整 anomaly 事件序列
  - 每筆欄位：`system`、`eventType`、`detail`、`time`、`frame`
- `causalChains[]`：因果鏈（v1.3+ 才有）
  - 每條鏈：`rootCause`、`sequence[]`、`spanFrames`
- `finalSnapshot`：CriticalSnapshot 快照（若有）
  - `trigger`、`shopState`、`listenerBalance`、`frame`

**match.json**
- `stateFingerprints[]`：`{tick, hash}` 序列

**report.json**
- `title`、`description`（玩家描述）
- `platform`、`os`、`unityVersion`
- `networkStats`：`pingAvg`、`pingPeak`、`packetLoss`
- `tickSync`：`localTick`、`remoteTick`、`offset`

---

## 四、Step 2：CLASSIFY — 事件分類

> 完整 Canonical 事件類型表請讀 `event-types.md`

依 `eventType` 將 `events[]` 分入以下 bucket：

| Bucket | EventType | 嚴重度 |
|---|---|---|
| 🔴 CRITICAL | `CriticalDump` | ERROR |
| 🟠 LIFECYCLE | `ListenerImbalance` | WARN |
| 🟠 STATE | `InvalidTransition` | WARN |
| 🟡 UI | `DuplicateWrite`、`LocalizationMismatch` | WARN |
| 🟡 SYNC | `DesyncDetected` | WARN |
| 🟡 PAGE | `PageResetMissing` | WARN |
| 🔵 FREQUENCY | `HighFrequencyAnomaly` | WARN |
| ⚪ GUARD | `StormDetected` | WARN（診斷系統自保護） |
| ℹ️ INFO | `ScrollBegin`、`ScrollEnd` | INFO（背景資訊） |

**分類後輸出**：
```
[事件統計]
- CriticalDump: N 筆
- ListenerImbalance: N 筆
- InvalidTransition: N 筆
- DuplicateWrite: N 筆
- DesyncDetected: N 筆
- PageResetMissing: N 筆
- StormDetected: N 筆（若有，代表部分事件已被 sampling 丟棄）
```

---

## 五、Step 3：CHAIN — 因果鏈重建

### 路徑 A：anomaly.json 已含 causalChains（v1.3+）

直接讀取，輸出每條鏈：
```
Chain #1
  rootCause: ListenerImbalance
  sequence: ListenerImbalance → InvalidTransition → CriticalDump
  spanFrames: 342
  解讀：[依 spanFrames 規則判讀]
```

**spanFrames 判讀規則**（來源：規格文件 Section 14）：
- `spanFrames < 60`  → 瞬間崩壞型，OnDisable 未正確呼叫
- `spanFrames > 300` → 累積型 leak，長時間 prefab pool 未釋放

### 路徑 B：causalChains 為空或缺失

手動從 `events[]` 依時間/frame 序列推斷：

```
推斷規則：
1. 找最早的 WARN/ERROR 事件 → 候選 rootCause
2. 看後續事件 frame 是否在 rootCause frame 後 60～600 frame 內連發
3. 若有 CriticalDump，確認其 frame 是否在鏈的末端
4. StormDetected 存在時 → 標注「部分事件可能因 sampling 遺失，因果鏈不完整」
```

---

## 六、Step 4：ROOT CAUSE — 定位

> 修法細節查 `fix-playbook.md`

### 前置：複合型 Bug 偵測

**在進入決策樹前，先執行這個檢查：**

```
若以下任兩種（或以上）eventType 同時出現在 events[] 中：
  - ListenerImbalance
  - DuplicateWrite
  - InvalidTransition

→ 標注「⚠️ 複合型 Bug」
→ 輸出警告：「多個異常系統同時觸發，最早出現的事件不一定是根源。
             建議逐一排除而非只修最早那筆，修復後需重新完整測試。」
→ 仍繼續決策樹，但在 REPORT 的 Root Cause 欄位加注「複合型，信心度下調」
```

### 決策樹

```
Q1: causalChains 是否有 rootCause？
  ├─ YES → 直接取 rootCause eventType，進入對應修法
  └─ NO  → 進 Q2

Q2: 有 CriticalDump (finalSnapshot)？
  ├─ YES → 讀 finalSnapshot.listenerBalance
  │         ├─ balance > 3 → root = ListenerImbalance
  │         └─ balance ≤ 3 → root = InvalidTransition（State Machine 問題）
  └─ NO  → 進 Q3

Q3: 最早出現的 WARN 事件是？
  ├─ ListenerImbalance → root = Listener Leak
  ├─ InvalidTransition → root = State Machine
  ├─ DuplicateWrite   → root = UI 競爭覆寫
  ├─ PageResetMissing → root = 分頁未歸零
  └─ DesyncDetected   → root = Desync（需雙方 Fingerprint 對比）

Q4: StormDetected 是否存在？
  └─ YES → 標注「診斷資料不完整，建議先解決 Q3 的 rootCause 後重測」
```

---

## 七、Step 5：REPORT — 輸出格式

### 標準輸出結構

```markdown
## 🔍 地標爭霸 Bug 診斷報告

**Match ID**：{matchId}
**Player**：{playerId}
**時間**：{timestamp}
**玩家描述**：{description}

---

### 📊 事件統計
{bucket 統計表}

---

### 🔗 因果鏈
{causalChains 或手動推斷結果}
{若為手動推斷，標注「推斷，非系統輸出」}

---

### 🎯 Root Cause
**主要原因**：{eventType} — {一句話說明}
**複合型 Bug**：是 / 否
**診斷信心度**：{百分比} — {說明依據}

---

### 🛠 修法方向
{對應 fix-playbook 的修法步驟，最多 3 步，具體到哪個腳本哪個方法}

---

### ⚠️ 注意事項
{StormDetected 警告 / 資料缺失說明 / 複合型 Bug 警告 / 需補充資料請求}

---

### 📋 優先度評分
| 項目 | 分數 |
|---|---|
| 嚴重度（ERROR=3, WARN=2, INFO=1）| {n} |
| 可重現性（有因果鏈=+2, 有 Snapshot=+1）| {n} |
| 影響範圍（商店=高, UI=中, Sync=視情況）| {n} |
| **總分** | **{total}/10** |
```

### 信心度計算規則

診斷信心度從 **100%** 開始，依條件扣減：

| 條件 | 扣減 |
|---|---|
| `StormDetected` 存在（部分事件已 sampling 丟棄） | -20% |
| `causalChains` 為空，Root Cause 來自手動推斷 | -15% |
| 只有單邊 ZIP（無雙方對比，僅限 Desync 類 bug） | -10% |
| 複合型 Bug（多個 WARN bucket 同時觸發） | -15% |
| `finalSnapshot` 缺失（無 CriticalDump）| -10% |
| player.log 最後 100 行無相關 stack trace | -5% |

**信心度說明文字規則**：
- 85–100%：「資料完整，結論可直接作為修法依據」
- 65–84%：「資料有缺口，建議修法後重新測試確認」
- < 65%：「資料不足以確定根源，建議補充診斷包再行動」

---

## 八、特殊路徑

### 雙方 ZIP 對比（MatchCorrelate）

**前提**：兩份 ZIP 的 `matchId` 相同。

```
1. 對齊兩份 stateFingerprints（以 tick 為 key）
2. 找第一個 hash 不一致的 tick → divergence tick
3. 輸出：「在 tick={N} 雙方狀態開始分歧，hash A={X}, hash B={Y}」
4. 建議：查看 divergence tick 前後的 gameplay 邏輯
```

**定位到 divergence tick 後：Subsystem 縮小範圍**

單一整合 hash 只能告訴你「不同步了」，無法告訴你「哪裡不同步」。
開發者需要在 `GameplayStateSnapshot` 中拆分欄位分組，事後對比：

```
目前 ComputeHash() 混合所有欄位 → 無法區分是哪個 subsystem 爆掉

建議拆分記錄方向（未來版本）：
  positionHash   = hash(tileCoordX, tileCoordY, posXInt, posYInt)
  combatHash     = hash(hpInt, ownerId)
  resourceHash   = hash(resource)

→ 哪個 sub-hash 先分歧，就是哪個 subsystem 的問題
```

> ⚠️ 這是未來改進方向，需配合 `StateHashCollector.cs` 修改，**不是現版本功能**。
> 現版本只能定位 divergence tick，縮小到 subsystem 需開發者手動在該 tick 加 Debug.Log。

**當前單邊診斷方法（不需對方 ZIP）**：

```
看本地 stateFingerprints 序列：
- 找連續兩筆 hash 跳變的位置（DesyncDetected 會標記）
- 在該 tick 附近的 player.log 搜尋異常 state 計算
- 確認 ComputeHash() 的輸入欄位是否有 float 未整數化
```

> ⚠️ 單邊 ZIP 仍有診斷價值（確認本地 state 是否異常跳變）。雙方對比為加分路徑，非必要條件。

### 純文字 bug 描述（無 ZIP）

純文字回報是最常見的輸入形式，不能只給三行處理就結束。

**Step 1：關鍵詞萃取 → 初步分類**

| 描述關鍵詞 | 懷疑方向 |
|---|---|
| 「卡死」「按了沒反應」「按鈕失效」 | InvalidTransition / ListenerImbalance |
| 「文字變成別的語言」「顯示奇怪文字」 | DuplicateWrite / LocalizationMismatch |
| 「我看到的跟對方不同」「畫面不同步」 | DesyncDetected |
| 「滾動後 UI 停在奇怪地方」 | PageResetMissing / HighFrequencyAnomaly |
| 「商店打不開」「進了又跳出」 | InvalidTransition / CriticalDump |

**Step 2：追問清單（依懷疑方向選擇對應問題）**

通用必問（所有 bug 均需）：
```
□ 在哪個操作後發生？（例：點了頭髮分類後、切換背包再回商店後）
□ 能穩定重現嗎？還是偶發？
□ 第一次發生是在什麼時間點（遊戲開始多久後）？
□ 對手在線還是已離開？
```

若懷疑 ListenerImbalance / InvalidTransition：
```
□ 有沒有反覆進出商店或切換分頁？切了幾次？
□ 異常發生前有沒有切換到背包再回來？
□ 異常後商店是完全無法點擊，還是部分功能正常？
```

若懷疑 DuplicateWrite / LocalizationMismatch：
```
□ 目前語系設定是什麼？
□ 文字是在 Hover（滑鼠移到按鈕上）後才變？還是一進商店就錯？
□ 移開滑鼠後文字有沒有恢復？
```

若懷疑 DesyncDetected：
```
□ 哪個具體畫面不同？（HP、地標位置、資源數量？）
□ 是從遊戲開始就不同，還是進行到某個時間點後才分歧？
□ 雙方是否都有按 F8？
```

**Step 3：輸出格式**

```markdown
## 🔍 純文字回報初步分析（推論）

> ⚠️ 以下結論為推論，無診斷包支撐，信心度上限 50%。
> 請引導玩家按 F8 重現並提交 ZIP 後，執行完整診斷。

**懷疑方向**：{eventType}
**推論依據**：{描述中哪個關鍵詞觸發}

**建議玩家操作**：
1. 重現步驟：{具體重現步驟}
2. 重現後立即按 F8
3. 將 ZIP 上傳到 [Discord / Google Form]

**需要補充的問題**：
{從追問清單選出最關鍵的 2–3 題}
```

---

## 九、算力控制原則

- **禁止逐行掃描 player.log 全文**：只讀最後 100 行，除非玩家描述指向特定時間點
- **禁止對 stateFingerprints 做全量比對**：找前 20 筆有 hash 跳變的記錄即停
- **禁止重複解析同一 JSON**：PARSE 步驟一次完成，後續 step 讀取快取結果
- **INFO 級事件（ScrollBegin/End）不進入因果鏈分析**：僅作背景參考

---

## 十、參考資料

| 檔案 | 使用時機 |
|---|---|
| `event-types.md` | 需查詢完整 Canonical 事件表、觸發條件、system 欄位、RootEligible 判斷 |
| `fix-playbook.md` | 確認每種 root cause 的修法步驟、涉及腳本、修改位置（**僅引用修法步驟，不作 Agent 評分來源**）|

---

## 十一、Agent Canon 聲明（必讀）

### SKILL.md 是 Agent 的唯一規範來源

```
優先層級（高 → 低）：
  1. SKILL.md（本文件）        ← Agent 行為規範、信心度計算、評分邏輯的唯一準則
  2. event-types.md            ← Canonical 事件類型表（RootEligible 判斷依據）
  3. BugReport_Unified_Spec_v1_4.md  ← runtime 實作細節（Agent 需要時查閱）
  4. fix-playbook.md           ← 人工修法參考（Agent 只引用修法步驟，不用評分部分）
```

**禁止行為**：

- ❌ 使用 `fix-playbook.md` 的「優先度評分矩陣」覆蓋或替換 SKILL.md Section 七的信心度計算
- ❌ 把 `fix-playbook.md` 的「基礎分 / 條件分」與 SKILL.md 的優先度分數混用
- ❌ 因找不到因果鏈就跳過 Step 3 直接進 Step 4

### Priority Algorithm（Agent 遇到矛盾時適用）

```
if (SKILL.md 有明確規則) → 以 SKILL.md 為準
else if (event-types.md 有 Canonical 定義) → 以 event-types.md 為準
else if (BugReport_Unified_Spec_v1_4.md 有對應 section) → 引用 spec 說明
else → 標注「依據不足，信心度 -20%」
```

若兩份文件在同一問題上有出入，**以 SKILL.md 為準，並在報告的注意事項欄位標注衝突**。
