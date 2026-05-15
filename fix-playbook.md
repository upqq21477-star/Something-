# 修法 Playbook — 地標爭霸 v1.4

> 對應規格文件 Section 14（診斷流程實戰手冊）+ Section 19（白名單監控優先級）
> 每種 Root Cause 提供：確認方式 → 修法步驟 → 涉及腳本 → 驗證方式

> ⚠️ **本文件定位（Agent 讀取規則）**
> - 本文件為**工程師人工修法參考指南**，**不作 Agent 優先度評分依據**。
> - Agent 的信心度計算與優先度評分以 `SKILL.md` Section 七的規則為**唯一準則**。
> - Agent 在 REPORT 步驟的「修法方向」欄位可引用本文件的修法步驟，但不得將本文件的「基礎分 / 條件分」欄位作為 SKILL.md 評分邏輯的替代或覆蓋來源。

---

## 🔴 Root Cause A：ListenerImbalance（Listener Leak）

**確認方式**
```
anomaly.json events[] 出現：
  system="LIFECYCLE" eventType="ListenerImbalance"
  detail 含 addCount=4（或更高）

或 finalSnapshot.listenerBalance > 3
```

**spanFrames 區分修法重點**
- `spanFrames < 60`：瞬間崩壞型 → OnDisable 未被呼叫（GameObject 被隱藏但未 Disable）
- `spanFrames > 300`：累積型 → Prefab Pool 的實例未釋放，長時間累積

**修法步驟**
```
Step 1：找到 detail 欄位中的 pageId（例如 Hair_12345678）
Step 2：在對應 Controller（HairPageController / EyebrowPageController）確認：
        - OnEnable 中是否有 AddListener
        - OnDisable 中是否有配對的 RemoveListener
Step 3：若 RemoveListener 存在但仍 leak：
        - 確認 OnDisable 是否實際被呼叫（加 Debug.Log 驗證）
        - 檢查是否有 SetActive(false) 但沒有觸發 OnDisable 的路徑
Step 4：若是 Pool 累積型：
        - 確認 prefab 回收時有呼叫 Disable 而非只是 SetActive(false)
```

**涉及腳本**
- `HairPageController.cs`：OnEnable / OnDisable
- `EyebrowPageController.cs`：OnEnable / OnDisable
- `ListenerBalanceDetector.cs`：確認 pageId 格式（`$"{pageName}_{GetInstanceID()}"`）

**驗證方式**
```
修復後重新執行診斷流程（Section 13 步驟 1-7）：
- 進入商店頭髮 → 切換分頁 → 切換背包 → 回商店
- F8 → 確認 anomaly.json 無 ListenerImbalance
- ListenerBalanceDetector.GetBalance("Hair") 應回傳 0 或 1
```

---

## 🟠 Root Cause B：InvalidTransition（State Machine 非法轉換）

**確認方式**
```
anomaly.json events[] 出現：
  system="SHOP_STATE" eventType="InvalidTransition"
  detail 含 old={X} new={Y}（Y 不在合法轉換表內）
```

**常見非法路徑**
```
Closed → HairTab（跳過 Open 步驟）
HairTab → HairTab（重複進入同一 tab）
Open → Closed → HairTab（Closed 後直接跳 HairTab）
```

**修法步驟**
```
Step 1：讀取 detail 欄位的 old / new / source
        source 指向呼叫 SetState 的位置（如 ShopTabController.OnTabClicked）
Step 2：在 source 對應方法中，確認 SetState 前是否有合法的 state 前置條件
Step 3：若是「直接跳到 Tab 但未先 Open」：
        - 在進入 Tab 邏輯前加入 if (shopState != "Open") return;
        - 或加入 state guard：強制先過渡到 Open 再到 Tab
Step 4：ShopStateTracer.Trace() 呼叫位置確認：
        ⚠️ 必須在賦值前呼叫：
        ShopStateTracer.Trace(shopState, newState, "SourceName"); // ← 先
        shopState = newState;                                      // ← 後
```

**涉及腳本**
- `ShopStateTracer.cs`：ValidTransitions HashSet（若需新增合法路徑，在此擴充）
- 呼叫 SetState 的業務邏輯腳本（從 detail.source 定位）

**驗證方式**
```
修復後：F8 → anomaly.json 無 InvalidTransition
商店可正常開啟、切換分頁、關閉
```

---

## 🟡 Root Cause C：DuplicateWrite（UI 競爭覆寫）

**確認方式**
```
anomaly.json events[] 出現：
  system="UI_TEXT" eventType="DuplicateWrite"
  detail 含 writer1={A} writer2={B} 且 frame 相同
```

**典型情境**
```
frame=18601 HoverController 寫入 text=檢視（中文）
frame=18601 LocalizationManager 寫入 text=VIEW（英文）
→ 兩個系統同 frame 爭搶同一 TMP_Text，最後結果取決於執行順序
```

**修法步驟**
```
Step 1：確認 writer1 / writer2 分別是哪兩個系統
Step 2：判斷哪個系統「不應該在這個時機寫入」
        - Hover 系統：應在 OnPointerEnter 時寫，OnPointerExit 時還原
        - Localization：應在語系切換或 ShopOpen 時統一刷新
Step 3：在 Hover 的 OnPointerExit 確認是否正確還原文字
Step 4：若問題是 Hover 殘留中文值沒有清除：
        - 在 HoverController.OnPointerExit 加入：
          shopViewButton.text = LocalizationManager.GetText("shop_view_button");
Step 5：在修改白名單 UI 的旁邊加入 Observe 呼叫（觀測，不替換邏輯）：
        UITextAnomalyDetector.Observe("shop_view_button", value, "SourceName");
```

**涉及腳本**
- `UITextAnomalyDetector.cs`：WatchTargets 白名單
- HoverController（具體腳本名依專案而定）
- `LocalizationManager.cs`：Refresh 方法中需呼叫 UITextAnomalyDetector.SetLanguage(lang)

**驗證方式**
```
切換語系 → Hover 商店按鈕 → 移出 → 確認文字未殘留中文
F8 → anomaly.json 無 DuplicateWrite / LocalizationMismatch
```

---

## 🟡 Root Cause D：PageResetMissing（分頁未歸零）

**確認方式**
```
anomaly.json events[] 出現：
  system="PAGE" eventType="PageResetMissing"
  detail 含「切換分類 {A}→{B} 但 renderedPage={N} 未更新」
```

**典型情境**
```
用戶在頭髮分類滾動到第 3 頁 → 切換到眉毛分類
→ renderedPage 仍停在 3，眉毛顯示第 3 頁（UI 不刷新）
```

**修法步驟**
```
Step 1：找到 OnTabChanged 或切換分類的呼叫方法
Step 2：在切換後加入 page 歸零呼叫：
        PageRefreshTracer.Trace(0, newCategory, 0, "OnTabChanged");
Step 3：確認 render 層實際接收到 page=0 的指令並重新渲染
Step 4：若 Tracer 顯示 renderedPage=0 但 UI 仍停留：
        → 問題在 render 層未正確響應，不是 Tracer 問題
```

**涉及腳本**
- `PageRefreshTracer.cs`：Trace 呼叫（觀測用）
- 商店分頁切換邏輯（具體腳本依專案而定）

---

## 🔵 Root Cause E：HighFrequencyAnomaly（高頻事件）

**確認方式**
```
anomaly.json events[] 出現：
  system="FREQUENCY" eventType="HighFrequencyAnomaly"
  detail 含 event={EventId} count={N} frame={F}
```

**修法步驟**
```
Step 1：確認 EventId（HairScroll / EyebrowScroll / ShopHover / MatchSettle）
Step 2：找到對應的事件來源方法
Step 3：確認是否有 debounce / throttle 機制
        - Scroll：只記錄 ScrollBegin / ScrollEnd 邊界，不記錄每 delta
        - Hover：不應在每個 OnPointerEnter frame 觸發業務邏輯
Step 4：Release Build 確認：FrequencyDetector 是否被 #if 正確移除
        #if UNITY_EDITOR || DEVELOPMENT_BUILD || QA_BUILD
```

**涉及腳本**
- `FrequencyDetector.cs`：Threshold（5）/ CooldownFrames（30）可依需求調整
- ⚠️ Release Build 必須完整移除（#if 包圍），不只是「不輸出 log」

---

## 🔴 Root Cause F：CriticalDump（關鍵流程失敗）

**確認方式**
```
anomaly.json events[] 出現：
  system="SNAPSHOT" eventType="CriticalDump"
anomaly.json 的 finalSnapshot 欄位有完整快照
```

**修法步驟**
```
Step 1：讀取 finalSnapshot：
        {
          "trigger": "ShopOpen_Failed",
          "shopState": "HairTab",
          "listenerBalance": 4,
          "frame": 18601,
          "time": "12:01:33.001"
        }
Step 2：trigger 指出失敗的關鍵流程
Step 3：listenerBalance 值：
        - > 3 → 同時有 ListenerImbalance，優先修 Root Cause A
        - ≤ 3 → State Machine 問題，優先修 Root Cause B
Step 4：shopState 顯示失敗時的 state → 對照合法轉換表確認問題路徑
```

---

## 🔵 Root Cause G：DesyncDetected（雙方狀態不同步）

**確認方式**
```
anomaly.json events[] 出現：
  system="SYNC" eventType="DesyncDetected"
  detail 含 tick={N} prev={hashA} curr={hashB}
```

**修法步驟（單邊）**
```
Step 1：記錄 divergence tick N
Step 2：在 tick N 附近確認 authoritative state 計算邏輯：
        - 是否有 float 未整數化就參與計算
        - 是否有 timestamp 或 VFX state 混入 hash
Step 3：確認 StateHashCollector.ComputeHash() 欄位符合白名單規則
        （只使用整數化欄位，見 event-types.md）
```

**修法步驟（雙方 ZIP 對比，MatchCorrelate 路徑）**
```
Step 1：對齊兩份 stateFingerprints（以 tick 為 key）
Step 2：找第一個 hash 不一致的 tick → divergence point
Step 3：在 divergence tick 前的 tick 範圍內，逐一確認
        雙方執行路徑是否相同（同一 game event 觸發時序）
Step 4：常見原因：
        - 網路封包延遲導致事件執行順序不同
        - 一方執行了本地預測但另一方回滾
        - float 計算在不同 CPU/IL2CPP 結果不同
```

---

## ⚪ StormDetected 特殊處理

**含義**
```
診斷系統本身觸發 storm 保護：60 frame 內 anomaly 超過 10 次
→ 後續事件以 1/5 sampling 記錄，部分事件已丟棄
→ causalChains 可能不完整
```

**處理方式**
```
1. 不要把 StormDetected 視為 root cause
2. 先依已有事件找最早的 WARN → 作為候選 root cause
3. 修復後重新測試，確認完整序列（無 StormDetected）
4. 優先度提升：有 StormDetected 代表當時異常密度極高
```

---

## 優先度評分矩陣

| Root Cause | 基礎分 | +條件 |
|---|---|---|
| CriticalDump | 8 | +1 若 listenerBalance > 3 |
| ListenerImbalance | 6 | +2 若 spanFrames < 60（瞬間崩壞）|
| InvalidTransition | 6 | +1 若有 CriticalDump 跟隨 |
| DuplicateWrite | 4 | +1 若影響商店主流程 |
| DesyncDetected | 5 | +2 若雙方 ZIP 對比確認 |
| PageResetMissing | 3 | — |
| HighFrequencyAnomaly | 2 | +1 若 Release Build 未移除 |
| StormDetected | 加分項 | 任何 root cause +1（代表高密度異常）|

**最高分 10，≥ 7 須立即處理，4–6 本週內修，≤ 3 排入 backlog。**
