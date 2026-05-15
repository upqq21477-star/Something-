# Canonical 事件類型表 — 地標爭霸 v1.4

> ⚠️ 本表為唯一來源（對應規格 Section 13）。
> 所有程式碼與 anomaly.json 的 system/eventType 欄位均以此為準。

---

## 完整事件類型表

| System | EventType | 觸發條件（Rule）| 嚴重度 | 對應 Detector | RootEligible |
|---|---|---|---|---|---|
| `LIFECYCLE` | `ListenerImbalance` | Rule 2：AddListener - RemoveListener > 3 | **WARN** | ListenerBalanceDetector | ✅ 可作鏈頭 |
| `FREQUENCY` | `HighFrequencyAnomaly` | Rule 1：同 frame 同事件超過 5 次 | **WARN** | FrequencyDetector | ✅ 可作鏈頭 |
| `SHOP` | `ScrollBegin` | Scroll 開始（僅邊界記錄）| INFO | 手動呼叫 | ❌ 不可作鏈頭 |
| `SHOP` | `ScrollEnd` | Scroll 結束（僅邊界記錄）| INFO | 手動呼叫 | ❌ 不可作鏈頭 |
| `SHOP_STATE` | `InvalidTransition` | Rule 4：(oldState, newState) ∉ validTransitions | **WARN** | ShopStateTracer | ✅ 可作鏈頭 |
| `UI_TEXT` | `DuplicateWrite` | Rule 3：同 frame、同 target、不同 source、不同 value | **WARN** | UITextAnomalyDetector | ✅ 可作鏈頭 |
| `UI_TEXT` | `LocalizationMismatch` | 非中文語系下，非 LocalizationManager 來源寫入白名單 UI | **WARN** | UITextAnomalyDetector | ✅ 可作鏈頭 |
| `PAGE` | `PageResetMissing` | 切換分類後 renderedPage 未歸零 | **WARN** | PageRefreshTracer | ✅ 可作鏈頭 |
| `SNAPSHOT` | `CriticalDump` | 關鍵流程失敗時，ShopStateTracer.Trace 偵測到 InvalidTransition | **ERROR** | CriticalSnapshot | ⚠️ 通常為鏈尾 |
| `SYNC` | `DesyncDetected` | 本地連續兩筆 stateFingerprint hash 異常跳變 | **WARN** | StateHashCollector | ✅ 可作鏈頭 |
| `GUARD` | `StormDetected` | 60 frame 內 LogAnomaly 超過 10 次，進入 1/5 sampling 模式 | **WARN** | AnomalyStormGuard | ❌ 不可作鏈頭（自保護 meta-event）|

### RootEligible 欄位說明

`RootEligible` 表示該事件能否成為 `AnomalyCausalChain` 分析的因果鏈頭（`rootCause`）。

| 標記 | 含義 |
|---|---|
| ✅ 可作鏈頭 | WARN / ERROR 級事件，`Analyze()` 可以從此事件開始建鏈 |
| ⚠️ 通常為鏈尾 | `CriticalDump` 通常是鏈的最終結果，不作起點；但若 buffer 裡只有 CriticalDump 則仍可作鏈頭 |
| ❌ 不可作鏈頭 | INFO 級事件（ScrollBegin / ScrollEnd）或系統 meta-event（StormDetected），被 `AnomalyCausalChain.InfoEvents` 過濾，不進入因果鏈起點判斷 |

> **對應程式碼**：`AnomalyCausalChain.cs` 的 `InfoEvents` HashSet 定義不可作鏈頭的事件名稱。

---

## Rule 定義速查

### Rule 1：頻率異常（Frequency Anomaly）
```
count(event, same_frame) > 5
→ 同一 frame 同類事件超過 5 次
→ 再觸發需等 30 frame（CooldownFrames）
```
**常見場景**：Scroll/Hover 在 120Hz 下被非法重複呼叫、Infinite loop refresh

### Rule 2：Listener 不平衡（Listener Imbalance）
```
(AddListener 次數 - RemoveListener 次數) > 3
→ balance = 4 時才觸發，閾值 3 以下為合法範圍
```
**閾值說明**：> 1 會誤報（prefab reuse、object pool、nested canvas 均屬合法）；> 3 為實務安全值  
**常見場景**：OnDisable 未正確執行、prefab pool 未釋放

### Rule 3：重複寫入（Duplicate Write）
```
same target（白名單內）
same frame
different source
different value（相同值不視為衝突）
```
**常見場景**：HoverController 與 LocalizationManager 同 frame 寫入同一 TMP_Text

### Rule 4：非法狀態轉換（Invalid Transition）
```
(oldState, nextState) ∉ ValidTransitions set
```
**合法轉換表（ShopStateTracer）**：
```
Closed     → Open
Open       → Closed / HairTab / EyebrowTab / CardTab
HairTab    → EyebrowTab / CardTab / Closed
EyebrowTab → HairTab / CardTab / Closed
CardTab    → HairTab / EyebrowTab / Closed
```
任何不在上表的轉換均為非法，例如 `Closed → HairTab`（跳過 Open）

---

## 白名單監控對象（UITextAnomalyDetector）

僅監控以下 targetId，不全域追蹤：
- `shop_view_button`
- `shop_watch_label`
- `shop_category_label`

---

## StateHashCollector — 允許進 hash 的欄位

```
✅ tileCoordX / tileCoordY（整數座標）
✅ posXInt = (int)(position.x * 100)
✅ posYInt = (int)(position.y * 100)
✅ hpInt   = Mathf.RoundToInt(hp)
✅ resource（整數）
✅ ownerId（整數）

❌ float 原始值（Vector3、float hp、float velocity）
❌ VFX / animation / physics / UI / timestamp 狀態
```
**禁止原因**：Unity 不保證跨平台 float 一致性（IL2CPP vs Mono / ARM vs x86）

---

## frame index 的重要性

```
無 frame（難以判斷競爭還是順序）：
  12:01:33 HoverController     修改 text=檢視
  12:01:33 LocalizationManager 修改 text=VIEW

有 frame（立即確認同幀競爭）：
  12:01:33 frame=18601 HoverController     修改 text=檢視
  12:01:33 frame=18601 LocalizationManager 修改 text=VIEW
           ↑ 同一 frame → 確認 Rule 3 DuplicateWrite
```

---

## AnomalyStormGuard 行為

```
正常模式（≤ 10 次 / 60 frame）：所有 anomaly 完整記錄
Storm 模式（> 10 次 / 60 frame）：
  → 寫入 StormDetected 事件（標記進入 storm）
  → 後續每 5 次只通過 1 次（1/5 sampling）
  → 視窗重置（60 frame 後）自動恢復正常模式
```

**診斷含義**：anomaly.json 出現 StormDetected 時，因果鏈可能不完整，部分事件已被丟棄。
