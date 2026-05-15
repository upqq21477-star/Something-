# Codex 交付任務說明 — 地標爭霸 Bug Report & Runtime Diagnosis System

> 版本：v1.2  
> 對應規格：BugReport_Unified_Spec_v1_4.md  
> 最後更新：2026-05-15  
> 變更：新增 Phase 0 架構偵察、業主確認機制、Integration Point 清單流程

---

## ⛔ 最高優先規則：未經確認，不得執行任何實作

**你拿到這份文件後，第一件事不是寫程式。**

請依照以下順序執行：

```
Step 1 → 執行 Phase 0（架構偵察）
Step 2 → 產出 INSTRUMENTATION_MAP.md 草稿
Step 3 → 列出完整行動計畫，等待業主文字確認
Step 4 → 收到「確認開始」後，才進入 Phase 1 實作
```

**任何腳本的建立、任何現有檔案的修改，都必須在 Step 4 之後才能發生。**

---

## Phase 0：架構偵察（所有工作的第一步）

### 目的

本系統的所有 Detector 需要插入遊戲現有腳本的特定位置。  
在插入前，必須先確認這些位置確實存在，以及它們的真實名稱與結構。  
**不得假設、不得猜測、不得自行創造插入點。**

---

### 0-A. 掃描目標

請搜尋專案所有 `.cs` 檔案，找出以下四個系統的真實腳本位置。

#### 掃描目標 1：商店狀態切換

搜尋關鍵字（依序嘗試）：
```
"Shop"、"Store"、"SetState"、"ChangeCategory"、"PageType"、"ShopState"
```

需要找出：
- 負責切換商店狀態的腳本名稱（`.cs` 檔名）
- 實際方法名稱與參數型別
- 這個方法在哪裡被呼叫（caller）

---

#### 掃描目標 2：UI 頁面生命週期

搜尋關鍵字：
```
"HairPage"、"EyebrowPage"、"PageController"、"PageView"、"OnEnable"、"OnDisable"
```

需要找出：
- Hair / Eyebrow 相關頁面的實際腳本名稱
- 頁面開啟與關閉時執行的方法（OnEnable / OnDisable 或其他等效方法）

---

#### 掃描目標 3：主遊戲 Tick / 主迴圈

搜尋關鍵字：
```
"FixedUpdate"、"Update"、"Tick"、"GameLoop"、"Coroutine"
```

需要找出：
- 主要遊戲邏輯每幀執行的入口腳本
- 是否有固定頻率 Tick，或是 Coroutine chain
- 這個入口所在的腳本名稱與方法名稱

---

#### 掃描目標 4：事件系統

搜尋關鍵字：
```
"EventBus"、"EventBinder"、"AddListener"、"Subscribe"、"Invoke"、"UnityEvent"
```

需要找出：
- 專案是否有集中的事件系統
- 如果有，事件是用字串 key、enum 還是 Type 作為識別
- AddListener / RemoveListener 的呼叫集中在哪些腳本

---

### 0-B. 產出：INSTRUMENTATION_MAP.md

完成掃描後，建立 `INSTRUMENTATION_MAP.md`，格式如下：

```markdown
# INSTRUMENTATION_MAP — 地標爭霸 Diagnostics 插入點清單

產出時間：{產出日期}
掃描方式：靜態程式碼搜尋

---

## Detector 插入點

| Detector | 真實腳本 | 真實方法 | 插入位置 | 每秒估計呼叫次數 | 備註 |
|---|---|---|---|---|---|
| ShopStateTracer | （填入或 NOT FOUND）| （填入）| 方法開頭，賦值前 | （估計）| |
| ListenerBalanceDetector | （填入或 NOT FOUND）| OnEnable / OnDisable | 方法開頭 | （估計）| |
| StateHashCollector | （填入或 NOT FOUND）| （填入）| （填入）| （估計）| |
| FrequencyDetector | （填入或 NOT FOUND）| （填入）| （填入）| （估計）| |

---

## 事件系統現狀

- 是否有集中事件系統：（是 / 否 / 不確定）
- 事件識別方式：（字串 / enum / Type / 不確定）
- AddListener 主要集中位置：（腳本名稱或 NOT FOUND）

---

## 未找到的項目（NOT FOUND 清單）

以下插入點未能在現有腳本中確認，需業主補充：

- （列出所有 NOT FOUND 的項目）
```

**填寫規則：**
- 找不到對應腳本時，一律寫 `NOT FOUND`，**不得自行創造或假設名稱**
- 找到多個候選時，**全部列出**，不得自行選擇
- 每秒呼叫次數若無法確定，寫 `待確認`

---

### 0-B-2. 發現 NOT FOUND 時的處理規則

掃描過程中若無法確認插入點，依以下情況處理，**不得自行決定或跳過**：

**情況 A：某個 Detector 的插入點完全不存在**
```
→ 在 INSTRUMENTATION_MAP 該欄位標記 NOT FOUND
→ 在偵察報告中列出：「此 Detector 無法整合，原因：找不到對應插入點」
→ 停止，等待業主決定：跳過此 Detector，或提供正確腳本名稱
```

**情況 B：找到插入點，但結構與 Spec 假設不符**
```
（例如：state 是用 enum 而非字串管理，或 UI 切換由 Animator 驅動而非 SetState）
→ 在備註欄描述實際結構（不得假設等價）
→ 列出需要調整的 Detector 內容
→ 停止，等待業主確認後才修改 Detector 設計
```

**情況 C：找到多個候選腳本**
```
→ 全部列出，附上各自的方法簽名
→ 不自行選擇
→ 停止，等待業主指定使用哪一個
```

---

### 0-C. 業主確認步驟（必須等待，不得跳過）

完成 `INSTRUMENTATION_MAP.md` 後，輸出以下內容，**然後停止，等待業主回覆**：

---

**[Codex 偵察報告 — 等待確認]**

```
Phase 0 偵察完成。

INSTRUMENTATION_MAP.md 已產出，摘要如下：

✅ 已找到的插入點：（列出）
⚠️  找到多個候選、需業主確認的項目：（列出）
❌ 未找到的插入點（NOT FOUND）：（列出）

---

預計行動計畫（Phase 1 開始後的完整步驟）：

1. 建立 EventRingBuffer.cs（無依賴，最先建）
2. 建立 AnomalyStormGuard.cs
3. 建立 BugPackageData.cs
4. 建立 ListenerBalanceDetector.cs
   → 插入點：{INSTRUMENTATION_MAP 中確認的腳本與方法}
5. 建立 CriticalSnapshot.cs
6. 建立 ShopStateTracer.cs
   → 插入點：{INSTRUMENTATION_MAP 中確認的腳本與方法}
7. 建立 StateHashCollector.cs
   → 插入點：{INSTRUMENTATION_MAP 中確認的腳本與方法}
8. 建立 AnomalyCausalChain.cs
9. 建立 MatchInfoProvider.cs
10. 建立 NetworkStatsCollector.cs
11. 建立 TickSyncManager.cs
12. 建立 LocalZipPacker.cs
13. 建立 BrowserLauncher.cs
14. 建立 BugReportManager.cs
15. 建立 DiagnosticsBootstrap.cs

---

❗ NOT FOUND 項目需要業主提供以下資訊後才能繼續：

（逐條列出需要業主確認的問題）

---

請回覆「確認開始」或針對上述問題提供補充資訊。
在收到明確指示前，不會進行任何程式碼修改。
```

---

## 0. 先讀這裡：兩份文件的定位邊界

你會拿到兩份核心文件，**它們的用途完全不同**，不得混用：

| 文件 | 定位 | 你的任務 |
|---|---|---|
| `BugReport_Unified_Spec_v1_4.md` | Unity C# **runtime 腳本規格**，在遊戲內執行 | **實作**為 .cs 檔案 |
| `bug-diagnosis-skill/SKILL.md` | Codex Agent 的**分析指令集**，在開發環境執行 | **不產出任何 .cs**，保持原樣 |

**SKILL.md 裡的所有內容（信心度計算、追問清單、REPORT 格式）均為 Agent 分析邏輯，不是 Unity UI 或 runtime 程式碼。不要嘗試把它實作成 C#。**

---

## 1. 任務目標

在現有 Unity 專案中，實作 `BugReport_Unified_Spec_v1_4.md` 定義的完整 Bug Report & Runtime Diagnosis System。

**範圍**：
- 建立所有 Tier 1（必做）腳本
- 建立所有 Tier 2（強烈建議）腳本
- Tier 3 暫不實作，等待指示

**不在範圍內**：
- 修改任何現有 game logic 腳本
- 接管現有 state 管理
- 建立後端、API、或任何雲端服務
- 實作 Discord Webhook（Tier 2 選配，需另行指示才做）

---

## 2. 實作優先順序

### Tier 1：必做，按順序完成

| 順序 | 腳本 | 依賴 |
|---|---|---|
| 1 | `EventRingBuffer.cs` | 無（其他腳本依賴它，最先建）|
| 2 | `AnomalyStormGuard.cs` | 依賴 EventRingBuffer |
| 3 | `BugPackageData.cs` | 無（DTO 定義，其他腳本依賴）|
| 4 | `ListenerBalanceDetector.cs` | 依賴 EventRingBuffer |
| 5 | `CriticalSnapshot.cs` | 依賴 EventRingBuffer、ListenerBalanceDetector |
| 6 | `ShopStateTracer.cs` | 依賴 EventRingBuffer、CriticalSnapshot |
| 7 | `StateHashCollector.cs` | 依賴 EventRingBuffer |
| 8 | `AnomalyCausalChain.cs` | 依賴 EventRingBuffer |
| 9 | `MatchInfoProvider.cs` | 無 |
| 10 | `NetworkStatsCollector.cs`（Tier 2，但 BugReportManager 依賴）| 無 |
| 11 | `TickSyncManager.cs`（Tier 2，但 BugReportManager 依賴）| 無 |
| 12 | `LocalZipPacker.cs` | 依賴 BugPackageData、AnomalyCausalChain |
| 13 | `BrowserLauncher.cs` | 無 |
| 14 | `BugReportManager.cs` | 依賴所有上述腳本 |
| 15 | `DiagnosticsBootstrap.cs` | 依賴所有上述腳本（最後建，統一初始化入口）|

> ⚠️ **注意順序調整**：CriticalSnapshot（5）移到 ShopStateTracer（6）之前。
> 原因：ShopStateTracer.Trace() 內部呼叫 CriticalSnapshot.Dump()，
> 必須先有 CriticalSnapshot 才能讓 ShopStateTracer 編譯通過。

### Tier 2：Tier 1 全部完成後

| 腳本 | 說明 |
|---|---|
| `FrequencyDetector.cs` | ⚠️ 必須完整 `#if` 包圍（見下方注意事項 A）|
| `UITextAnomalyDetector.cs` | 白名單原則，不全域監控 |
| `PageRefreshTracer.cs` | 分頁歸零追蹤 |
| `ReportPanel.cs` | UI 面板 |

---

## 2.5 分階段落地順序（Integration Phases）

**不要在 gameplay 中一次接入所有 Detector。** 按以下四個階段逐步整合，每個 Phase 完成後驗收才進下一個。

### Phase 1：驗證事件流（最優先）
```
接入：EventRingBuffer + ShopStateTracer
目標：在商店 SetState 旁加入 ShopStateTracer.Trace()，
      確認 InvalidTransition 事件能正確寫入 EventRingBuffer
插入點：依 INSTRUMENTATION_MAP.md 確認的腳本與方法
驗收：觸發一次非法 state 轉換，GetAll() 回傳對應事件
```

### Phase 2：驗證 Anomaly Pipeline
```
接入：ListenerBalanceDetector（在 HairPageController / EyebrowPageController 的 OnEnable/OnDisable）
目標：確認 ListenerImbalance 事件能寫入 buffer，AnomalyStormGuard 正常 gate
插入點：依 INSTRUMENTATION_MAP.md 確認的腳本與方法
驗收：重複 AddListener 超過 3 次，確認觸發 LIFECYCLE/ListenerImbalance
```

### Phase 3：驗證 Dump Pipeline
```
接入：CriticalSnapshot + StateHashCollector + LocalZipPacker
目標：F8 按下後能產出完整 ZIP
驗收：ZIP 包含 6 個必要檔案，anomaly.json 有正確事件序列
```

### Phase 4：完整 F8 流程
```
接入：AnomalyCausalChain + BugReportManager + BrowserLauncher
目標：F8 → ZIP → 開啟瀏覽器的完整流程可用
驗收：執行 Section 14 的標準復現流程，確認因果鏈出現在 anomaly.json
```

---

## 3. 關鍵實作注意事項

以下是 spec 中容易被忽略或截斷的地方，實作前必讀。

### A. FrequencyDetector 的 #if 守衛

Release Build 必須**完整移除**，不只是「不輸出 log」：

```csharp
public static void Record(EventId eventId)
{
#if UNITY_EDITOR || DEVELOPMENT_BUILD || QA_BUILD
    // 所有邏輯都在這裡
    // 包含 count 計算、threshold 比對、LogAnomaly 呼叫
#endif
}
```

`#if` 必須包圍整個方法體，而不只是 LogAnomaly 那一行。

### B. EventRingBuffer 有兩個入口，不得混用

```csharp
// 一般 Detector 呼叫：通過 AnomalyStormGuard gate
public void LogAnomaly(string system, string eventType, string detail)

// 僅供 AnomalyStormGuard 自身呼叫：繞過 gate，防止遞迴
public void LogAnomalyDirect(string system, string eventType, string detail)
```

**所有 Detector（ListenerBalanceDetector、ShopStateTracer 等）只能呼叫 `LogAnomaly`，永遠不能呼叫 `LogAnomalyDirect`。**

### C. ShopStateTracer.Trace() 的呼叫時序

必須在賦值前呼叫，順序不能顛倒：

```csharp
// ✅ 正確
ShopStateTracer.Trace(shopState, "HairTab", "SourceName.Method");
shopState = "HairTab";

// ❌ 錯誤（Trace 拿到的是新值，無法判斷 old state）
shopState = "HairTab";
ShopStateTracer.Trace(shopState, "HairTab", "SourceName.Method");
```

### D. Thread Ownership 規則（v1.4 嚴格執行）

| 規則 | 說明 |
|---|---|
| 禁止 worker thread 碰 `UnityEngine.Object` | 截圖 encode 在主執行緒完成後，只傳 `byte[]` 給 worker |
| 禁止 `Task.Run` 缺少 `CancellationToken` | 所有背景 task 必須可被 `OnDestroy` 中止 |
| 禁止 `BuildPackageData` 裡呼叫 `FindWithTag` | 所有組件參考在 `Awake` 快取，F8 觸發時直接讀快取 |

### E. StateHashCollector 欄位白名單

進入 `ComputeHash()` 的欄位**只能是整數**：

```csharp
// ✅ 允許
s.tileCoordX, s.tileCoordY      // 整數座標
s.posXInt = (int)(pos.x * 100)  // 整數化後的位置
s.hpInt   = Mathf.RoundToInt(hp) // 整數化後的血量
s.resource, s.ownerId

// ❌ 禁止（float 跨平台不一致，會造成誤報 desync）
s.position.x, s.velocity, s.hp（float 原始值）
```

### F. Section 17 的 Skill 分層已整合

Spec Section 17 提到 UnzipSkill / AnomalyParseSkill / BugClassifySkill 等，**這些已整合進 `bug-diagnosis-skill/SKILL.md`**，不需要實作為 C# 腳本，也不需要建立對應檔案。

### G. ShopStateTracer → CriticalSnapshot 的單向依賴是刻意設計

`ShopStateTracer.Trace()` 內部直接呼叫 `CriticalSnapshot.Dump()`。
這是規格明確定義的單向依賴，**不是錯誤，不需要重構**：

```csharp
// ShopStateTracer.cs — 這是正確的，不要移除
if (!ValidTransitions.Contains((oldState, newState)))
{
    EventRingBuffer.Instance?.LogAnomaly(...);
    CriticalSnapshot.Dump("ShopOpen_Failed", newState); // ← 刻意直接呼叫
}
```

不要引入 EventBus 或 callback 來解耦這個關係，會增加不必要的複雜度。

### H. 所有 static Detector 必須實作 Reset()

Unity Editor 在 Play Mode 重啟（Domain Reload）時，static 欄位可能殘留上一次的狀態，
導致 Detector 出現 duplicate subscribe 或 stale cache。

每個 static Detector 必須有明確的 `Reset()` 方法：

```csharp
// 範例（每個 static Detector 都需要）
public static class ListenerBalanceDetector
{
    private static readonly Dictionary<string, int> _balance = new();

    public static void Reset() => _balance.Clear(); // ← 必須有這個
}
```

`DiagnosticsBootstrap.cs` 負責在初始化時統一呼叫所有 `Reset()`。

### I. DiagnosticsBootstrap.cs 規格

這是新增腳本，spec 未定義，由 Codex 自行實作，需符合以下規格：

```csharp
public static class DiagnosticsBootstrap
{
    // AfterSceneLoad 確保在所有 MonoBehaviour Awake 之後執行
    [RuntimeInitializeOnLoadMethod(RuntimeInitializeLoadType.AfterSceneLoad)]
    private static void Initialize()
    {
        // 1. 清除所有 static Detector 的殘留狀態
        ListenerBalanceDetector.Reset();
        ShopStateTracer.Reset();        // 若有 static 欄位
        UITextAnomalyDetector.Reset();  // 若有 static 欄位
        AnomalyStormGuard.Reset();
        FrequencyDetector.Reset();      // #if 保護
        PageRefreshTracer.Reset();      // ✅ 確認納入：有 _lastCategory / _lastRenderedPage 靜態欄位，
                                        //    必須在 Play Mode 重啟時重置，否則殘留舊分頁狀態。

        // 2. 確認 MonoBehaviour Singleton 已就緒（不要在這裡建立）
        Debug.Assert(EventRingBuffer.Instance != null,
            "[Diagnostics] EventRingBuffer 未初始化，請確認 Scene 中有掛載");
        Debug.Assert(StateHashCollector.Instance != null,
            "[Diagnostics] StateHashCollector 未初始化");
    }
}
```

**重要**：Bootstrap 只做初始化與驗證，不建立任何 GameObject 或 Singleton。
MonoBehaviour Singleton（EventRingBuffer、StateHashCollector）仍由 Scene 中的物件負責建立。

### K. MatchInfoProvider Contract（v1.4 補充）

`MatchInfoProvider` 在 Spec Section 18（腳本清單）列為 Tier 1 必做，但原始規格未定義類別結構。  
Codex 實作時需符合以下契約：

```csharp
/// <summary>
/// MatchInfoProvider
/// 用途：提供當前對局的 matchId / playerId / opponentId 給 BugReportManager
/// 插入點：Network 層 Match 開始回調（OnMatchStart / OnRoomJoined / OnGameBegin）
/// 插入方式：在 Match 開始時呼叫 SetMatchInfo()
/// 算力影響：非 hot path，僅 Match 開始時執行一次
/// </summary>
public class MatchInfoProvider : MonoBehaviour
{
    public static MatchInfoProvider Instance;

    public string MatchId     { get; private set; } = "";
    public string PlayerId    { get; private set; } = "";
    public string OpponentId  { get; private set; } = "";

    void Awake() => Instance = this;

    /// <summary>Match 開始時由網路層呼叫，快取對局識別資料。</summary>
    public void SetMatchInfo(string matchId, string playerId, string opponentId)
    {
        MatchId    = matchId;
        PlayerId   = playerId;
        OpponentId = opponentId;
    }
}
```

**Source of Truth（業主確認後填入）：**

| 欄位 | 候選來源 | 狀態 |
|---|---|---|
| `MatchId` | `PhotonNetwork.CurrentRoom.Name` 或 `NetworkManager.MatchId` | ⚠️ 待業主確認 |
| `PlayerId` | `PhotonNetwork.LocalPlayer.NickName` 或 `NetworkManager.LocalPlayerId` | ⚠️ 待業主確認 |
| `OpponentId` | 對應對手玩家的 NetworkId | ⚠️ 待業主確認 |

**初始化時機**：
- `SetMatchInfo()` 必須在 F8 可能被觸發之前呼叫（最晚在 Match 開始時）
- 若 F8 按下時 `MatchInfoProvider.Instance` 為 null，`BugReportManager` 的 `BuildPackageData` 會用空字串填入，ZIP 的 match ID 將為空白
- `DiagnosticsBootstrap` 的 `Debug.Assert` 目前**不驗證** `MatchInfoProvider`（因為 Match 可能尚未開始），這是刻意設計

每個 Detector 腳本頂部，必須附上不超過 10 行的中文說明，格式如下：

```csharp
/// <summary>
/// [Detector 名稱]
/// 用途：（這個 Detector 在偵測什麼）
/// 插入點：（插入在哪個腳本的哪個方法）
/// 插入方式：（如何呼叫，呼叫時機）
/// 算力影響：（是否為 hot path，每秒約呼叫幾次）
/// </summary>
```

這是強制要求，目的是讓非實作者能在之後快速理解每個 Detector 的作用與位置。

---

## 4. 驗收標準

每個 Tier 完成後，執行以下檢查：

### Tier 1 驗收

```
□ 所有 Tier 1 .cs 檔案在同一 Unity 專案能無錯誤編譯
□ EventRingBuffer.Instance 在 Awake 正確初始化
□ AnomalyStormGuard.ShouldLog() 被 LogAnomaly 呼叫（不被 LogAnomalyDirect 呼叫）
□ DiagnosticsBootstrap 的 Assert 在 Play Mode 啟動時無錯誤
□ Play Mode 重啟後，所有 static Detector 狀態乾淨（無殘留 balance / state）
□ BugReportManager 按 F8 觸發後能產出 ZIP 到 BugReports 目錄
□ ZIP 內包含：metadata.txt / report.json / match.json / anomaly.json / screenshot.png / player.log
□ LocalZipPacker 使用背景執行緒（PackAsync），不 block 主執行緒
□ 截圖 encode 在主執行緒完成，只傳 byte[] 給背景執行緒
```

### Phase 驗收（依 Section 2.5 順序）

```
Phase 1 □ 觸發非法 state 轉換 → EventRingBuffer.GetAll() 回傳 InvalidTransition 事件
Phase 2 □ AddListener 超過 3 次 → 觸發 LIFECYCLE/ListenerImbalance
Phase 3 □ F8 產出 ZIP，anomaly.json 格式正確，6 個必要檔案齊全
Phase 4 □ anomaly.json 的 causalChains 有正確因果鏈
```

### Tier 2 驗收

```
□ FrequencyDetector 在 Release Build 下完整移除（驗證方式：切換 Build Mode 確認無殘留呼叫）
□ UITextAnomalyDetector 只監控白名單（shop_view_button / shop_watch_label / shop_category_label）
□ ReportPanel 能正常顯示截圖預覽、輸入標題描述、觸發 BugReportManager
```

### 靜默失敗禁止清單

```
□ ZIP 生成失敗時，玩家必須看到明確提示（見 spec Section 16）
□ Fallback A 驗收：ZIP 失敗後 BugReports/ 目錄有散檔（screenshot / json / log 分別存在）
□ Fallback B 驗收：即使 JSON 生成失敗，screenshot.png 必須成功存入磁碟
□ Fallback C 驗收：遊戲內顯示提示文字，包含 Match ID（玩家看得到，不是只有 console log）
□ 禁止任何 catch 區塊為空
□ 禁止 Task.Run 沒有 CancellationToken
```

---

## 5. 不要做的事

```
❌ 不在 Phase 0 完成並取得業主確認前，開始任何實作
❌ 不修改任何現有 game logic 腳本
❌ 不接管任何 state（Tracer / Detector 只觀測，不替換邏輯）
❌ 不建立後端或 API endpoint
❌ 不把 SKILL.md 的內容實作為 C#
❌ 不在 Release Build 保留 FrequencyDetector 的任何程式碼
❌ 不在 ComputeHash() 裡放入 float 原始值
❌ 不在 worker thread 呼叫任何 UnityEngine.Object
❌ 不全域監控所有 TMP_Text（白名單原則）
❌ 不全域 trace 所有 OnClick / OnPointerEnter
❌ 不引入 EventBus 來解耦 Detector（現有規模不需要，ShopStateTracer 直接呼叫 CriticalSnapshot 是刻意設計）
❌ 不讓任何 Detector 控制或修改 gameplay state（if(anomaly) FixState() 嚴格禁止）
❌ 不讓 Detector 之間互相依賴（除 ShopStateTracer→CriticalSnapshot 這個規格內的單向依賴）
❌ 不在 DiagnosticsBootstrap 裡建立 GameObject 或 Singleton
❌ 不在 INSTRUMENTATION_MAP 找不到插入點時自行假設或創造腳本名稱
```

---

## 6. 參考文件索引

| 文件 | 用途 |
|---|---|
| `BugReport_Unified_Spec_v1_4.md` Section 3 | Rule 1–4 偵測邏輯 |
| `BugReport_Unified_Spec_v1_4.md` Section 4 | 所有 Detector 的參考實作 |
| `BugReport_Unified_Spec_v1_4.md` Section 5 | ZIP 格式規格（必讀才能實作 LocalZipPacker）|
| `BugReport_Unified_Spec_v1_4.md` Section 10–11 | Thread ownership + Async 規則 |
| `BugReport_Unified_Spec_v1_4.md` Section 13 | Canonical EventType 表（唯一來源）|
| `BugReport_Unified_Spec_v1_4.md` Section 16 | ZIP 失敗 Fallback 流程 |
| `bug-diagnosis-skill/SKILL.md` | Agent 分析工具（不實作，僅參考）|
| `INSTRUMENTATION_MAP.md` | Phase 0 產出，Detector 真實插入點清單（由 Codex 建立）|
