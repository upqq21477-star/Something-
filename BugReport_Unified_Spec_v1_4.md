# Bug Report & Runtime Diagnosis System — 規格文件

> **適用遊戲**：地標爭霸（Steam PC 端，2 人連線對戰）
> **版本**：v1.4
> **最後更新**：2026-05-15

---

## 目錄

1. [目的與系統定位](#1-目的與系統定位)
2. [整體架構](#2-整體架構)
3. [Runtime Anomaly Rules（核心偵測規則）](#3-runtime-anomaly-rules核心偵測規則)
4. [Runtime Diagnosis Layer](#4-runtime-diagnosis-layer)
   - 4.1 EventRingBuffer
   - 4.2 ListenerBalanceDetector
   - 4.3 FrequencyDetector
   - 4.4 ShopStateTracer
   - 4.5 UITextAnomalyDetector
   - 4.6 PageRefreshTracer
   - 4.7 CriticalSnapshot
   - 4.8 StateHashCollector（Desync Fingerprint）
   - 4.9 AnomalyStormGuard（自保護 Sampling）
   - 4.10 AnomalyCausalChain（因果鏈追蹤）
   - Detector Budget
5. [Bug Package（ZIP 規格）](#5-bug-packagezip-規格)
   - 5.1 目錄結構
   - 5.2 metadata.txt
   - 5.3 report.json
   - 5.4 match.json
   - 5.5 anomaly.json
   - 5.6 screenshot.png
   - 5.7 player.log
6. [封裝層（F8 觸發流程）](#6-封裝層f8-觸發流程)
   - 6.1 BugReportManager
   - 6.2 NetworkStatsCollector
   - 6.3 TickSyncManager
   - 6.4 LocalZipPacker
   - 6.5 BrowserLauncher
7. [上傳模式](#7-上傳模式)
   - 7.1 無後端模式（主力）
   - 7.2 Discord Webhook 模式（選配）
   - 7.3 三模式對比
8. [UI 設計](#8-ui-設計)
9. [Build Mode 環境分級](#9-build-mode-環境分級)
10. [Thread Ownership 規格](#10-thread-ownership-規格)
11. [Async / Coroutine 注意事項](#11-async--coroutine-注意事項)
12. [Performance Rules](#12-performance-rules)
13. [Anomaly 事件類型規格（Canonical）](#13-anomaly-事件類型規格canonical)
14. [診斷流程（實戰手冊）](#14-診斷流程實戰手冊)
15. [安全性](#15-安全性)
16. [ZIP 生成失敗 Fallback](#16-zip-生成失敗-fallback)
17. [Agent + Skill 後處理分層設計](#17-agent--skill-後處理分層設計)
18. [腳本清單與優先順序](#18-腳本清單與優先順序)
19. [白名單監控優先級](#19-白名單監控優先級)
20. [適用階段](#20-適用階段)
21. [排除功能](#21-排除功能)

---

## 1. 目的與系統定位

### 兩種 bug 各需不同工具

| Bug 類型 | 傳統 Crash Log 能抓嗎？ | 需要什麼 |
|---|---|---|
| NullReferenceException | ✅ | 現有 Sentry / Crashlytics |
| 商店按鈕卡死（state 被悄悄改壞）| ❌ | **Runtime Anomaly Detection** |
| Listener 重複註冊累積 N 倍觸發 | ❌ | **Runtime Anomaly Detection** |
| Hover 覆蓋 Localization 文字 | ❌ | **Runtime Anomaly Detection** |
| 穿模、UI 錯位（需截圖） | ❌ | **Bug Report Upload** |
| desync（雙方狀態不同）| ❌ | **Bug Report Upload + Desync Fingerprint** |

**結論**：這兩套系統互補，缺一都有死角。本文件將兩層整合為一個系統。

### 系統核心模式

本系統採用**完全無後端（Backend-less）**設計：

```
舊模式（已移除）：Unity Client → Webhook / API → Cloud Database
新模式（主力）  ：Unity Client → 本地封裝 ZIP → 玩家手動上傳
```

- 遊戲只負責收集診斷資料與本地封裝
- 玩家自行選擇是否上傳
- 遊戲本身不持有任何 Secret / Token / API Key

### 設計原則

```
✅ 不重構現有 game logic
✅ 不接管 state
✅ 不全域監控（白名單原則）
✅ 不高頻記錄（只記異常）
✅ 玩家低負擔（F8 一鍵觸發）
✅ 雙方視角可對齊（透過 matchId）
✅ 零後端、零 API Key、零 infra 成本
```

---

## 2. 整體架構

```
Bug Report & Runtime Diagnosis System
│
├── Runtime Diagnosis Layer（主動找 root cause）
│   ├── EventRingBuffer          ← 只記 anomaly 事件（上限 50 筆）
│   ├── AnomalyStormGuard        ← 自保護 gate：storm 時切換 sampling，防二次 lag
│   ├── ListenerBalanceDetector  ← Rule 2：listener leak
│   ├── FrequencyDetector        ← Rule 1：高頻異常
│   ├── ShopStateTracer          ← Rule 4：非法 state 轉換
│   ├── UITextAnomalyDetector    ← Rule 3：重複寫入（白名單）
│   ├── PageRefreshTracer        ← 分頁未歸零
│   ├── CriticalSnapshot         ← 異常瞬間完整狀態快照（含 cooldown）
│   ├── StateHashCollector       ← Desync Fingerprint（每 30～60 tick hash）
│   └── AnomalyCausalChain       ← F8 時分析因果鏈，輸出 causalChains
│
└── 封裝層（收集 + 本地封裝）
    ├── BugReportManager         ← F8 觸發、截圖、資料彙整
    ├── NetworkStatsCollector    ← ping avg/peak、packet loss
    ├── TickSyncManager          ← localTick / remoteTick / offset
    ├── MatchInfoProvider        ← matchId / playerId / opponentId
    ├── LocalZipPacker           ← 打包 ZIP（PackAsync，背景執行緒）
    ├── BrowserLauncher          ← 開瀏覽器到上傳頁面
    └── ReportPanel              ← UI：描述輸入、截圖預覽、送出
```

### 資料流

```
F8 按下
  │
  ├─ [截圖] CaptureScreenshotAsync（非同步，不 block 主執行緒）
  ├─ [Diagnosis Layer] 讀取 EventRingBuffer（anomaly log）+ CriticalSnapshot
  ├─ [封裝層]
  │    → 讀 Player.log 最後 500 行
  │    → 取 NetworkStats 快照
  │    → 取 MatchInfo（matchId / playerId）
  │    → 取 TickSyncInfo（localTick / remoteTick）
  │    → 讀取 StateHashCollector ring buffer（stateFingerprints）
  └─ [LocalZipPacker] 組裝並產出 ZIP
       │
       ├── metadata.txt
       ├── report.json
       ├── match.json（含 stateFingerprints）
       ├── anomaly.json（含 finalSnapshot）
       ├── screenshot.png
       └── player.log
            ↓
       開 Windows Explorer 選中 ZIP + 開啟瀏覽器 → 玩家手動上傳
```

---

## 3. Runtime Anomaly Rules（核心偵測規則）

所有 Detector 的偵測邏輯基礎。

### Rule 1：頻率異常（Frequency Anomaly）

```
公式：count(event, window) > threshold

用途：
  - Listener leak（OnScroll 被呼叫 N 倍）
  - Infinite loop / recursive refresh
  - Event storm
```

### Rule 2：Listener 不平衡（Listener Imbalance）

```
公式：(listener_add_count - listener_remove_count) > 3

閾值說明：
  > 1 誤報太高（prefab reuse、object pool、nested canvas 等合法情況可能觸發）
  > 3 作為實務安全值，仍能抓到累積型 leak

用途：
  - AddListener 未配對 RemoveListener
  - OnDisable 未正確執行
```

### Rule 3：重複寫入（Duplicate Write）

```
公式：
  same target
  same frame
  different source
  different value（相同值不視為衝突）

用途：
  - Localization vs Hover 競爭覆蓋
  - 同 frame 多系統寫入同一 UI
```

### Rule 4：非法狀態轉換（Invalid Transition）

```
公式：(oldState, nextState) ∉ validTransitions

用途：
  - Dead state（商店卡死）
  - Invalid shop flow
```

---

## 4. Runtime Diagnosis Layer

### 4.1 EventRingBuffer（Anomaly-Focused）

> ⚠️ **這個 Buffer 只記錄 anomaly 事件，不記錄一般 gameplay 事件。**
> 一般 gameplay 事件（SkillFireball、ReceiveHit 等）由封裝層的 `BugReportManager` 另行記錄，兩者不混用。

```csharp
// EventRingBuffer.cs
public class EventRingBuffer : MonoBehaviour
{
    public static EventRingBuffer Instance;
    private const int Capacity = 50;
    private readonly EventEntry[] _buffer = new EventEntry[Capacity];
    private int _head = 0;

    void Awake() => Instance = this;

    // 只在異常時呼叫，正常路徑不寫入
    public void LogAnomaly(string system, string eventType, string detail)
    {
        _buffer[_head % Capacity] = new EventEntry
        {
            system    = system,
            eventType = eventType,
            detail    = detail,
            time      = DateTime.Now.ToString("HH:mm:ss.fff"),
            frame     = Time.frameCount
        };
        _head++;
    }

    public List<EventEntry> GetAll()
    {
        var result = new List<EventEntry>();
        int count = Mathf.Min(_head, Capacity);
        for (int i = 0; i < count; i++)
            result.Add(_buffer[(_head - count + i) % Capacity]);
        return result;
    }
}

[System.Serializable]
public struct EventEntry
{
    public string system, eventType, detail, time;
    public int frame;
}
```

**為何只記錄異常？**
每 frame 寫 log 在 60fps 下會產生大量 GC 與 IO 壓力。只在 Rule 1–4 觸發時寫入，可將 log 成本降至接近零。

---

### 4.2 ListenerBalanceDetector

最快確認是否為 event leak 的方法。本質是 `Dictionary<string, int>` 加減，只在 `OnEnable` / `OnDisable` 發生，**不是每 frame**，成本接近零。

```csharp
// ListenerBalanceDetector.cs
public static class ListenerBalanceDetector
{
    private static readonly Dictionary<string, int> _balance = new();

    public static void OnAdd(string pageId)
    {
        _balance.TryGetValue(pageId, out int count);
        _balance[pageId] = count + 1;

        // Rule 2：balance > 3 才警告
        if (_balance[pageId] > 3)
        {
            EventRingBuffer.Instance?.LogAnomaly("LIFECYCLE", "ListenerImbalance",
                $"[LEAK] page={pageId} addCount={_balance[pageId]} frame={Time.frameCount}");
        }
    }

    public static void OnRemove(string pageId)
    {
        _balance.TryGetValue(pageId, out int count);
        _balance[pageId] = Mathf.Max(0, count - 1);
    }

    public static int GetBalance(string pageId)
    {
        _balance.TryGetValue(pageId, out int count);
        return count;
    }

    /// <summary>Play Mode 重啟時由 DiagnosticsBootstrap 呼叫，清除殘留 balance。</summary>
    public static void Reset() => _balance.Clear();
}
```

**使用方式（頭髮／眉毛 Controller）：**

```csharp
public class HairPageController : MonoBehaviour
{
    private string PageId => $"Hair_{GetInstanceID()}";

    void OnEnable()
    {
        ListenerBalanceDetector.OnAdd(PageId);
        scrollRect.onValueChanged.AddListener(OnScroll);
        styleGrid.onClick.AddListener(OnCellClick);
    }

    void OnDisable()
    {
        ListenerBalanceDetector.OnRemove(PageId);
        scrollRect.onValueChanged.RemoveListener(OnScroll);
        styleGrid.onClick.RemoveListener(OnCellClick);
    }
}
```

**判讀方式：**

```
正常：
  Add (Hair, frame=100) → balance=1
  Remove (Hair, frame=200) → balance=0
  Add (Hair, frame=300) → balance=1  ← 第二次進入，正常

合法範圍（不觸發）：
  Add → balance=1 → balance=2 → balance=3  ← 閾值內，不報

異常（觸發 log）：
  Add → ... → balance=4  ← [LEAK] 輸出！
```

---

### 4.3 FrequencyDetector

唯一可能偏重的 Detector。原因：Scroll / Hover / MouseMove 可能高頻。
已用 Threshold + CooldownFrames 控制成本，不會 lag。

> ⚠️ **Release Build 應用 `#if` 完整移除**，不只是「不輸出 log」。
> 即使沒有 log 輸出，高頻 UI 操作下仍有寫入成本。

> 🔧 **v1.2 修正**：原版使用 `Dictionary<string, (int, int)>` 做 hot path key lookup，
> 在 Scroll / Hover 120Hz 場景下 string hash 成本不可忽略（IL2CPP 下亦有 boxing 風險）。
> 改用 **固定 enum + static array**，zero allocation，幾乎零成本。

```csharp
// FrequencyDetector.cs
public static class FrequencyDetector
{
    // ✅ 固定事件集合，不用 string key
    public enum EventId
    {
        HairScroll     = 0,
        EyebrowScroll  = 1,
        ShopHover      = 2,
        MatchSettle    = 3,
        // 新增事件：在此擴充 enum，並更新 _arraySize
    }

    private const int _arraySize = 4; // 與 EventId 值數量保持一致

    private struct FreqRecord { public int count, frame; }

    // ✅ 固定大小 array，不用 Dictionary，零 alloc
    private static readonly FreqRecord[] _records        = new FreqRecord[_arraySize];
    private static readonly int[]        _lastReportFrame = new int[_arraySize];

    private const int Threshold     = 5;  // 同 frame 超過 5 次視為異常
    private const int CooldownFrames = 30; // 同一 event 30 frame 內只報一次

    public static void Record(EventId eventId)
    {
#if UNITY_EDITOR || DEVELOPMENT_BUILD || QA_BUILD
        int id           = (int)eventId;
        int currentFrame = Time.frameCount;

        if (_records[id].frame == currentFrame)
            _records[id].count++;
        else
        {
            _records[id].count = 1;
            _records[id].frame = currentFrame;
        }

        if (_records[id].count > Threshold)
        {
            if (currentFrame - _lastReportFrame[id] >= CooldownFrames)
            {
                EventRingBuffer.Instance?.LogAnomaly("FREQUENCY", "HighFrequencyAnomaly",
                    $"[WARN] event={eventId} count={_records[id].count} frame={currentFrame}");
                _lastReportFrame[id] = currentFrame;
            }
        }
#endif
    }

    /// <summary>Play Mode 重啟時由 DiagnosticsBootstrap 呼叫，清除計數與 cooldown 殘留。</summary>
    public static void Reset()
    {
#if UNITY_EDITOR || DEVELOPMENT_BUILD || QA_BUILD
        System.Array.Clear(_records,         0, _records.Length);
        System.Array.Clear(_lastReportFrame, 0, _lastReportFrame.Length);
#endif
    }
}
```

**使用方式：**

```csharp
public void OnScrollDelta(float delta)
{
    // ✅ enum 呼叫，零 string alloc
    FrequencyDetector.Record(FrequencyDetector.EventId.HairScroll);
    // 原本邏輯...
}

// ✅ 只記錄 ScrollBegin / ScrollEnd 邊界
public void OnScrollBegin() =>
    EventRingBuffer.Instance?.LogAnomaly("SHOP", "ScrollBegin",
        $"category={currentCategory} frame={Time.frameCount}");

public void OnScrollEnd() =>
    EventRingBuffer.Instance?.LogAnomaly("SHOP", "ScrollEnd",
        $"category={currentCategory} frame={Time.frameCount}");
```

---

### 4.4 ShopStateTracer

**追蹤器，不是控制系統。** 只在偵測到非法轉換時記錄，不攔截或接管 state。
本質是 `HashSet.Contains()`，只在 `SetState()` 時發生，**不是每 frame**。

```csharp
// ShopStateTracer.cs
public static class ShopStateTracer
{
    // ✅ v1.4 修正：語義分離
    // LastValidState  = 最後一次合法轉換後的 state（CriticalSnapshot 讀此值）
    // LastAttemptedState = 最後一次嘗試轉換的目標 state（含非法嘗試，供排查用）
    public static string LastValidState    { get; private set; } = "Closed";
    public static string LastAttemptedState { get; private set; } = "Closed";

    private static readonly HashSet<(string from, string to)> ValidTransitions = new()
    {
        ("Closed",     "Open"),
        ("Open",       "Closed"),
        ("Open",       "HairTab"),
        ("Open",       "EyebrowTab"),
        ("Open",       "CardTab"),
        ("HairTab",    "EyebrowTab"),
        ("HairTab",    "CardTab"),
        ("HairTab",    "Closed"),
        ("EyebrowTab", "HairTab"),
        ("EyebrowTab", "CardTab"),
        ("EyebrowTab", "Closed"),
        ("CardTab",    "HairTab"),
        ("CardTab",    "EyebrowTab"),
        ("CardTab",    "Closed"),
    };

    /// <summary>
    /// 在現有 SetState 呼叫旁加一行 Trace，不替換原邏輯。
    /// oldState 由呼叫方傳入（賦值前的當前 state）。
    /// </summary>
    public static void Trace(string oldState, string newState, string source)
    {
        LastAttemptedState = newState; // 始終記錄嘗試值（含非法）

        // Rule 4：非法轉換才寫 log
        if (!ValidTransitions.Contains((oldState, newState)))
        {
            EventRingBuffer.Instance?.LogAnomaly("SHOP_STATE", "InvalidTransition",
                $"[ILLEGAL] old={oldState} new={newState} source={source} frame={Time.frameCount}");

            // 異常時觸發 Critical Snapshot；LastValidState 維持舊值，不更新
            CriticalSnapshot.Dump("ShopOpen_Failed", newState);
        }
        else
        {
            LastValidState = newState; // 只在合法轉換時更新
        }
    }

    /// <summary>Play Mode 重啟時由 DiagnosticsBootstrap 呼叫，重置兩個 state 欄位。</summary>
    public static void Reset()
    {
        LastValidState     = "Closed";
        LastAttemptedState = "Closed";
    }
}
```

**使用方式（侵入最小）：**

```csharp
// ⚠️ 必須在賦值前呼叫
ShopStateTracer.Trace(shopState, "HairTab", "ShopTabController.OnTabClicked"); // ← 只加這行
shopState = "HairTab";
```

---

### 4.5 UITextAnomalyDetector

只監控白名單內的高風險 UI 文字，**不全域接管 TMP_Text**。

> 🔧 **v1.2 修正**：原版在每次 `Observe()` 呼叫時執行 `ContainsChinese()` 字元迴圈掃描。
> 在 Hover / Scroll 高頻場景（每秒 300 次以上）下有 CPU 成本。
> 改為由 `LocalizationManager` 語系切換時主動推送語系狀態，`Observe()` 只做整數比較，零掃描成本。

```csharp
// UITextAnomalyDetector.cs
public static class UITextAnomalyDetector
{
    // 白名單：只監控這些目標
    private static readonly HashSet<string> WatchTargets = new()
    {
        "shop_view_button",
        "shop_watch_label",
        "shop_category_label"
    };

    private static readonly Dictionary<string, (int frame, string source, string value)> _lastWrite = new();

    // ✅ 語系 cache：由 LocalizationManager.Refresh 前呼叫，不在 Observe 裡做 runtime scan
    private static bool _isChinese = true;

    /// <summary>
    /// 語系切換時由 LocalizationManager 呼叫（一次性更新）。
    /// </summary>
    public static void SetLanguage(string lang)
    {
        _isChinese = lang is "zh" or "zh-CN" or "zh-TW";
    }

    public static void Observe(string targetId, string value, string source)
    {
        // 不在白名單 → 直接跳過，零成本
        if (!WatchTargets.Contains(targetId)) return;

        int currentFrame = Time.frameCount;

        // Rule 3：同 frame、同 target、不同 source 且值不同 → 重複寫入異常
        if (_lastWrite.TryGetValue(targetId, out var last))
        {
            if (last.frame == currentFrame && last.source != source && last.value != value)
            {
                EventRingBuffer.Instance?.LogAnomaly("UI_TEXT", "DuplicateWrite",
                    $"[CONFLICT] target={targetId} " +
                    $"writer1={last.source} value1={last.value} writer2={source} value2={value} frame={currentFrame}");
            }
        }

        // ✅ 語系不一致偵測：用 cache flag，不做 runtime 字元掃描
        if (!_isChinese && source != "LocalizationManager.Refresh")
        {
            // 非中文語系下，非 Localization 來源寫入白名單 UI → 可能是中文 Hover 殘留
            EventRingBuffer.Instance?.LogAnomaly("UI_TEXT", "LocalizationMismatch",
                $"[WARN] target={targetId} lang=non-zh unexpectedSource={source} value={value}");
        }

        _lastWrite[targetId] = (currentFrame, source, value);
    }

    /// <summary>Play Mode 重啟時由 DiagnosticsBootstrap 呼叫，清除 lastWrite 紀錄與語系快取。</summary>
    public static void Reset()
    {
        _lastWrite.Clear();
        _isChinese = true; // 重置為預設語系
    }
}
```

**使用方式：**

```csharp
// LocalizationManager 語系切換時主動推送（一次性）
UITextAnomalyDetector.SetLanguage(newLanguage);

// 現有邏輯不動，只加 Observe
shopViewButton.text = localizedValue;
UITextAnomalyDetector.Observe("shop_view_button", localizedValue, "LocalizationManager.Refresh");
```

---

### 4.6 PageRefreshTracer

直接對應「分頁停留第 1 頁 / UI 不刷新」問題。

```csharp
// PageRefreshTracer.cs
public static class PageRefreshTracer
{
    private static string _lastCategory = "None";
    private static int _lastRenderedPage = 0;

    /// <summary>
    /// 在切換分類後呼叫。renderedPage 傳入 render 層實際顯示的 page index。
    /// 只在「切換分類後 renderedPage 仍停在舊值」時警告。
    /// </summary>
    public static void Trace(int page, string category, int renderedPage, string source)
    {
        bool categoryChanged = category != _lastCategory;

        // WARN only：某些 UI 設計合法保留 page index，只輸出 WARN 而非 anomaly
        if (categoryChanged && renderedPage == _lastRenderedPage && renderedPage != 0)
        {
            EventRingBuffer.Instance?.LogAnomaly("PAGE", "PageResetMissing",
                $"[WARN] 切換分類 {_lastCategory}→{category} 但 renderedPage={renderedPage} 未更新 " +
                $"source={source} frame={Time.frameCount}");
        }

        _lastCategory = category;
        _lastRenderedPage = renderedPage;
    }

    /// <summary>Play Mode 重啟時由 DiagnosticsBootstrap 呼叫，清除分頁殘留狀態。</summary>
    public static void Reset()
    {
        _lastCategory     = "None";
        _lastRenderedPage = 0;
    }
}
```

---

### 4.7 CriticalSnapshot

比 event log 更重要。當關鍵流程失敗時，立即 dump 當下完整狀態。

> 🔧 **v1.2 修正**：原版無 cooldown，若 anomaly storm 觸發（每秒 30 次以上）會連續 alloc 匿名 object，造成 GC spike。
> 加入 `lastDumpFrame` cooldown（60 frame ≈ 1 秒），避免高頻觸發。

```csharp
// CriticalSnapshot.cs
public static class CriticalSnapshot
{
    // ✅ v1.4 修正：使用 [Serializable] class 取代匿名型別，JsonUtility.ToJson() 才能序列化
    [System.Serializable]
    private class SnapshotData
    {
        public string trigger;
        public string shopState;
        public string currentTab;   // = ShopStateTracer.LastValidState（合法狀態，不含非法嘗試）
        public int    listenerBalance;
        public int    frame;
        public string time;
    }

    public static SnapshotData LastDump { get; private set; }
    private static int _lastDumpFrame = -9999; // cooldown 起點
    private const int CooldownFrames = 60;     // 60 frame ≈ 1 秒

    public static void Dump(string trigger, string currentState)
    {
        // ✅ cooldown：避免 anomaly storm 造成連續 GC spike
        int frame = Time.frameCount;
        if (frame - _lastDumpFrame < CooldownFrames) return;
        _lastDumpFrame = frame;

        // ✅ v1.4 修正：讀取 LastValidState（合法轉換後的狀態）
        var snapshot = new SnapshotData
        {
            trigger         = trigger,
            shopState       = currentState,
            currentTab      = ShopStateTracer.LastValidState,
            listenerBalance = ListenerBalanceDetector.GetBalance("Hair"),
            frame           = frame,
            time            = DateTime.Now.ToString("HH:mm:ss.fff")
        };

        LastDump = snapshot;

        // ✅ [Serializable] class → JsonUtility.ToJson() 可正確序列化（匿名型別無法）
        // prettyPrint: false 避免高頻異常時額外 GC
        string json = JsonUtility.ToJson(snapshot, prettyPrint: false);

        EventRingBuffer.Instance?.LogAnomaly("SNAPSHOT", "CriticalDump", json);
        Debug.LogError($"[CriticalSnapshot] {trigger}\n{json}");
    }
}
```

**觸發範例輸出（anomaly.json 內的 finalSnapshot）：**

```json
{
  "trigger":         "ShopOpen_Failed",
  "shopState":       "HairTab",
  "currentTab":      "HairTab",
  "listenerBalance": 2,
  "frame":           18601,
  "time":            "12:01:33.001"
}
```

---

### 4.8 StateHashCollector（Desync Fingerprint）

> **定位聲明**：Desync Fingerprint 是診斷工具，不是 lockstep 框架。它只記錄每 N tick 的狀態 hash，F8 時序列化進 `match.json`，事後雙方對比找 divergence point。

#### 規格摘要

| 屬性 | 值 |
|---|---|
| 觸發時機 | 每 N tick，在 FixedUpdate 或 game loop tick 末端呼叫 |
| 觸發頻率 | 每 **30～60 tick** 計算一次，不是每 tick |
| 讀取對象 | authoritative gameplay state（整數化，見下方欄位規則）|
| 寫入對象 | 本地 `List<FingerprintEntry>`，上限 **200 筆**（ring buffer）|
| F8 觸發時 | 將整個 list 序列化進 `match.json["stateFingerprints"]` |
| Build Mode | **全 Build Mode 均啟用**（Development / QA / Release）|

**Build Mode 全開理由**：Desync 問題在 Release Build 才是真實問題；runtime 成本極低（每 30 tick 純整數運算），F8 觸發時才序列化，不影響幀率。

#### 可以進 hash 的欄位

```
✅ tile coordinate / grid coordinate
✅ (int)(position.x * 100)
✅ rounded hp（整數）
✅ resource integer
✅ owner id
```

#### 禁止進 hash 的欄位

```
❌ float 原始值（Vector3、float hp、float velocity）
❌ VFX 狀態
❌ interpolation 值
❌ UI 狀態
❌ animation state
❌ physics velocity
❌ timestamp
```

**禁止原因**：Unity 不保證跨平台 float 一致性（不同 CPU / SIMD / IL2CPP vs Mono / ARM vs x86），直接 hash float 會造成邏輯一致但 hash 不同的誤報。

#### 單邊可用原則

即使只有一方提交，`stateFingerprints` 仍有診斷價值（可確認本地 state 是否在某 tick 發生異常跳變）。雙方對比是加分路徑，不是必要條件。

#### 記憶體上限說明

10 分鐘對局 × 60fps / 30tick ≈ 1,200 筆；ring buffer 上限 200 筆（保留最近記錄），約 4KB，不影響記憶體。

```csharp
// StateHashCollector.cs
using UnityEngine;
using System.Collections.Generic;

public class StateHashCollector : MonoBehaviour
{
    public static StateHashCollector Instance;

    [System.Serializable]
    public struct FingerprintEntry
    {
        public int tick;
        public string hash;
    }

    private const int HashInterval    = 30;          // 每 30 tick 計算一次
    private const int RingBufferLimit = 200;         // ring buffer 上限 200 筆

    // ✅ v1.4 修正：使用固定大小 array ring buffer，取代 List.RemoveAt(0)（O(n) 移位）
    private readonly FingerprintEntry[] _ring = new FingerprintEntry[RingBufferLimit];
    private int  _ringHead  = 0;   // 下一個寫入位置（對 RingBufferLimit 取模）
    private int  _ringCount = 0;   // 已寫入筆數（達到 RingBufferLimit 後固定為 RingBufferLimit）
    private string _lastHash = "";

    void Awake() => Instance = this;

    /// <summary>
    /// 在 FixedUpdate 或 game loop tick 末端呼叫（每 tick 呼叫一次）。
    /// 只有在 currentTick % HashInterval == 0 時才計算 hash。
    /// </summary>
    public void OnTick(int currentTick, GameplayStateSnapshot state)
    {
        if (currentTick % HashInterval != 0) return;

        string hash = ComputeHash(state);

        // Divergence 偵測：同一方連續兩筆 hash 異常跳變
        if (_lastHash != "" && hash != _lastHash)
        {
            EventRingBuffer.Instance?.LogAnomaly("SYNC", "DesyncDetected",
                $"tick={currentTick} prev={_lastHash} curr={hash}");
            // 不觸發 CriticalSnapshot（避免過度雜訊）
        }

        _lastHash = hash;

        // ✅ 寫入 ring buffer（零移位，O(1)）
        _ring[_ringHead % RingBufferLimit] = new FingerprintEntry { tick = currentTick, hash = hash };
        _ringHead++;
        if (_ringCount < RingBufferLimit) _ringCount++;
    }

    /// <summary>F8 時呼叫，回傳整個 ring buffer（按時間順序）供序列化進 match.json。</summary>
    public List<FingerprintEntry> GetAll()
    {
        var result = new List<FingerprintEntry>(_ringCount);
        int start  = _ringCount < RingBufferLimit ? 0 : _ringHead % RingBufferLimit;
        for (int i = 0; i < _ringCount; i++)
            result.Add(_ring[(start + i) % RingBufferLimit]);
        return result;
    }

    private string ComputeHash(GameplayStateSnapshot s)
    {
        // 只使用整數化欄位，避免 float 跨平台不一致
        unchecked
        {
            int h = 17;
            h = h * 31 + s.tileCoordX;
            h = h * 31 + s.tileCoordY;
            h = h * 31 + s.posXInt;   // (int)(position.x * 100)
            h = h * 31 + s.posYInt;
            h = h * 31 + s.hpInt;     // rounded hp
            h = h * 31 + s.resource;
            h = h * 31 + s.ownerId;
            return ((uint)h).ToString("X8");
        }
    }
}

/// <summary>
/// 呼叫方在每 tick 末端從 authoritative gameplay state 讀取並填入此結構。
/// 所有欄位必須是整數化後的值，不得傳入 float 原始值。
/// </summary>
[System.Serializable]
public struct GameplayStateSnapshot
{
    public int tileCoordX, tileCoordY;
    public int posXInt, posYInt;   // (int)(position.x * 100)
    public int hpInt;              // Mathf.RoundToInt(hp)
    public int resource;
    public int ownerId;
}
```

---

---

### 4.9 AnomalyStormGuard（自保護 Sampling）

> **背景**：CriticalSnapshot 的 60 frame cooldown 只保護單一 Dump 點。若多個 Detector 同時大量觸發（異常 storm），EventRingBuffer 仍會連續寫入，LogAnomaly 的 string interpolation 成本會在主執行緒積累，形成「診斷系統本身造成二次 lag」的悖論。
>
> 本模組讓整個診斷 layer 具備**自保護能力**：偵測到 anomaly storm 時，自動切換至 sampling 模式，丟棄多餘的寫入，保護 runtime 不受 debug infra 拖累。

```csharp
// AnomalyStormGuard.cs
public static class AnomalyStormGuard
{
    // 滑動視窗設定
    private const int WindowFrames   = 60;  // 觀測視窗：60 frame ≈ 1 秒
    private const int StormThreshold = 10;  // 視窗內超過 10 次寫入 → 進入 storm 模式
    private const int SampleRate     = 5;   // storm 模式下每 5 次只通過 1 次

    private static int _windowStart  = 0;
    private static int _countInWindow = 0;
    private static bool _inStorm     = false;
    private static int _stormSampleCounter = 0;

    /// <summary>
    /// 所有 Detector 呼叫 EventRingBuffer.LogAnomaly 前，
    /// 先通過此 gate。回傳 false → 丟棄本次寫入。
    /// </summary>
    public static bool ShouldLog()
    {
        int frame = Time.frameCount;

        // 視窗重置
        if (frame - _windowStart >= WindowFrames)
        {
            _windowStart   = frame;
            _countInWindow = 0;
            _inStorm       = false;
        }

        _countInWindow++;

        if (_countInWindow > StormThreshold)
        {
            if (!_inStorm)
            {
                _inStorm = true;
                // storm 進入事件本身寫一筆（不再遞迴呼叫 guard）
                EventRingBuffer.Instance?.LogAnomalyDirect("GUARD", "StormDetected",
                    $"[WARN] 進入 anomaly storm 模式，切換為 1/{SampleRate} sampling frame={frame}");
            }

            // sampling：每 SampleRate 次通過一次
            _stormSampleCounter++;
            return _stormSampleCounter % SampleRate == 0;
        }

        return true;
    }

    /// <summary>Play Mode 重啟時由 DiagnosticsBootstrap 呼叫，清除視窗計數與 storm 狀態。</summary>
    public static void Reset()
    {
        _windowStart        = 0;
        _countInWindow      = 0;
        _inStorm            = false;
        _stormSampleCounter = 0;
    }
}
```

**對 EventRingBuffer 的配合修改：**

```csharp
// EventRingBuffer.cs（新增 LogAnomalyDirect，供 Guard 本身使用，繞過 gate）
public void LogAnomalyDirect(string system, string eventType, string detail)
{
    // 不通過 AnomalyStormGuard，直接寫入（只供 Guard 自身的 meta-event 使用）
    _buffer[_head % Capacity] = new EventEntry
    {
        system = system, eventType = eventType, detail = detail,
        time   = DateTime.Now.ToString("HH:mm:ss.fff"),
        frame  = Time.frameCount
    };
    _head++;
}

// LogAnomaly 加 gate（所有 Detector 的正常呼叫路徑）
public void LogAnomaly(string system, string eventType, string detail)
{
    if (!AnomalyStormGuard.ShouldLog()) return; // ← 新增
    LogAnomalyDirect(system, eventType, detail);
}
```

**各 Detector 的呼叫方式不變。** Gate 統一在 `LogAnomaly` 入口處理，Detector 代碼不需要修改。

**Storm 模式下的行為：**

```
正常模式（≤ 10 次/秒）：所有 anomaly 完整記錄
Storm 模式（> 10 次/秒）：
  → 寫入一筆 StormDetected 事件（標記進入 storm）
  → 後續每 5 次只通過 1 次（sampling 保留特徵樣本）
  → 視窗重置後自動恢復正常模式
```

---

### 4.10 AnomalyCausalChain（因果鏈追蹤）

> **背景**：孤立的 anomaly 事件需要開發者手動推理因果關係。例如：
> `ListenerImbalance` → 5 秒後 `InvalidTransition` → 10 秒後 `CriticalDump`
> 這條因果鏈在 anomaly.json 裡是三筆獨立記錄，開發者需自行串聯。
>
> 本模組在 F8 時自動分析 EventRingBuffer，找出時間相近的異常序列，生成因果鏈摘要，直接輸出進 `anomaly.json` 的 `causalChains` 欄位。
>
> **INFO 事件過濾**：`ScrollBegin` / `ScrollEnd` 等 INFO 級事件不會成為因果鏈的鏈頭（rootCause）。  
> 原因：Scroll 事件是邊界記錄，本身不是異常，不應成為診斷的起點。  
> 過濾規則由 `InfoEvents` HashSet 定義，可依需求擴充。  
> 對應事件表的 `RootEligible` 欄位（見 `event-types.md`）。

```csharp
// AnomalyCausalChain.cs
public static class AnomalyCausalChain
{
    // 兩筆 anomaly 相差不超過此 frame 數，視為同一因果鏈的連續事件
    private const int ChainWindowFrames = 600; // 600 frame ≈ 10 秒（@60fps）

    public struct ChainEntry
    {
        public string rootCause;        // 鏈頭 eventType
        public List<string> sequence;   // 事件序列（eventType 列表）
        public int startFrame;
        public int endFrame;
        public int spanFrames;          // 跨越幾個 frame
    }

    // INFO 級事件不作鏈頭（ScrollBegin / ScrollEnd 等邊界記錄事件不是 root cause）
    private static readonly HashSet<string> InfoEvents = new()
    {
        "ScrollBegin",
        "ScrollEnd",
    };

    /// <summary>
    /// F8 時呼叫。輸入整個 anomaly buffer，輸出因果鏈清單。
    /// 純計算，不寫任何 log，不影響 runtime。
    /// </summary>
    public static List<ChainEntry> Analyze(List<EventEntry> events)
    {
        var chains  = new List<ChainEntry>();
        if (events == null || events.Count == 0) return chains;

        int i = 0;
        while (i < events.Count)
        {
            // 鏈頭：找下一個 WARN 或 ERROR 級事件作為起點（跳過 INFO 事件）
            if (InfoEvents.Contains(events[i].eventType))
            {
                i++;
                continue;
            }

            var root = events[i];
            var seq  = new List<string> { root.eventType };
            int j    = i + 1;

            while (j < events.Count &&
                   events[j].frame - root.frame <= ChainWindowFrames)
            {
                seq.Add(events[j].eventType);
                j++;
            }

            // 只收錄長度 ≥ 2 的鏈（孤立事件不成鏈）
            if (seq.Count >= 2)
            {
                chains.Add(new ChainEntry
                {
                    rootCause  = root.eventType,
                    sequence   = seq,
                    startFrame = root.frame,
                    endFrame   = events[j - 1].frame,
                    spanFrames = events[j - 1].frame - root.frame
                });
            }

            i = j; // 跳過已被收入當前鏈的事件
        }

        return chains;
    }
}
```

**anomaly.json 新增 `causalChains` 欄位：**

```json
{
  "anomalyCount": 3,
  "events": [ ... ],
  "causalChains": [
    {
      "rootCause":  "ListenerImbalance",
      "sequence":   ["ListenerImbalance", "InvalidTransition", "CriticalDump"],
      "startFrame": 18543,
      "endFrame":   19203,
      "spanFrames": 660
    }
  ],
  "finalSnapshot": { ... }
}
```

**`LocalZipPacker.BuildAnomalyJson()` 已整合（見 Section 6.4）：**

causalChains 的生成與輸出已直接合併進 Section 6.4 的完整 `BuildAnomalyJson` 函式，輸出位置在 `events` 之後、`finalSnapshot` 之前。Codex 實作時以 Section 6.4 的完整函式為準，本節代碼片段僅供理解邏輯用，不需要另行插入。

**診斷流程配合（Section 14 延伸）：**

```
收到 anomaly.json 後第一步：
1. 看 causalChains 有幾條
2. 每條鏈的 rootCause 是什麼
3. spanFrames 多長（短鏈 < 60f → 瞬間崩壞 / 長鏈 > 300f → 累積型問題）
4. 再深入看 events 序列確認細節
```

---

### Detector Budget

| Detector | 觸發時機 | Hot Path | Release Build |
|---|---|---|---|
| ListenerBalanceDetector | OnEnable / OnDisable | 否 | 保留 |
| FrequencyDetector | 每次事件呼叫 | **是** | `#if` 完整移除 |
| ShopStateTracer | SetState 時 | 否 | 保留 |
| UITextAnomalyDetector | UI 寫入時（白名單） | 視白名單大小 | 保留 |
| PageRefreshTracer | 分頁切換時 | 否 | 保留 |
| CriticalSnapshot | 異常觸發時（含 cooldown）| 否 | 保留 |
| StateHashCollector | 每 30～60 tick | 否（純整數運算）| 保留（全 Build Mode）|
| AnomalyStormGuard | LogAnomaly 入口（gate）| **是**（輕量整數比較）| 保留 |
| AnomalyCausalChain | F8 時一次性分析 | 否（F8 only）| 保留 |

> ⚠️ **Detector 清單凍結：不新增任何 Detector，直至 Desync Fingerprint 實作完成。**

---

## 5. Bug Package（ZIP 規格）

### 5.1 目錄結構

```
BugReport_2026-05-15_04-31-22.zip
│
├── metadata.txt       ← 人類可讀摘要（快速預覽用）
├── report.json        ← 玩家提交資訊 + gameplay events
├── match.json         ← 對局資訊（含 stateFingerprints）
├── anomaly.json       ← 異常事件序列 + finalSnapshot
├── screenshot.png     ← F8 截圖
└── player.log         ← 最後 500 行 log
```

---

### 5.2 metadata.txt

**用途**：人類可讀的純文字摘要。開發者打開 ZIP 後第一眼快速確認重點，無需解析 JSON。

**格式範例：**

```
Report ID  : 4f8a2c1d-ae21-4b03-9c1f-007e3f8821cc
Timestamp  : 2026-05-15 04:31:22
Version    : 0.4.2
Match ID   : room_18273
Player     : p1  vs  p2
Anomalies  : 2  (ListenerImbalance, InvalidTransition)
Screenshot : screenshot.png
Log Lines  : 500
```

**設計原則：**
- 純 UTF-8，不依賴 JSON parser
- 顯示 anomaly 數量與類型，讓開發者第一眼判斷是否需要深入閱讀
- 不含敏感資訊（無 IP、無玩家帳號）

---

### 5.3 report.json

玩家提交的主觀描述資訊與 gameplay 事件記錄。

```json
{
  "reportId":       "4f8a2c1d-ae21-4b03-9c1f-007e3f8821cc",
  "title":          "Hair tab softlock",
  "category":       "UI",
  "description":    "Shop becomes unusable after switching tabs",
  "buildVersion":   "0.4.2",
  "timestamp":      "2026-05-15T04:31:22Z",
  "scene":          "Battle_Arena_01",
  "position":       "(12.3, 0.0, -5.8)",
  "playerState":    "attacking",
  "opponentState":  "rolling",
  "gameplayEvents": [
    "12:01:29.412 [P1] SkillFireball pos=(10,0,-4)",
    "12:01:31.022 [P2] ReceiveHit dmg=20",
    "12:01:32.100 [P2] StateChange rolling"
  ]
}
```

---

### 5.4 match.json

**用途**：記錄本場對局的同步資訊，供 desync 診斷與雙方資料關聯使用。`stateFingerprints` 統一嵌入此檔（見 Section 4.8 StateHashCollector 規格）。

```json
{
  "matchId":          "room_18273",
  "playerId":         "p1",
  "opponentId":       "p2",
  "localTick":        1842,
  "remoteTick":       1839,
  "tickDelta":        3,
  "timeSyncOffsetMs": 214,
  "matchDuration":    498,
  "network": {
    "pingAvg":        88,
    "pingPeak":       142,
    "packetLossAvg":  0.02,
    "packetLossPeak": 0.18,
    "disconnectCount": 0,
    "disconnectReason": ""
  },
  "stateFingerprints": [
    { "tick": 1800, "hash": "A81F22D1" },
    { "tick": 1830, "hash": "A81F22D1" },
    { "tick": 1860, "hash": "C4F09B3E" }
  ]
}
```

**欄位說明：**

| 欄位 | 說明 |
|---|---|
| `matchId` | 唯一場次 ID，供雙方 log 關聯 |
| `playerId` / `opponentId` | 提交方與對手識別碼 |
| `localTick` / `remoteTick` | F8 按下瞬間的雙方 tick 值 |
| `tickDelta` | tick 差值，desync 診斷核心指標 |
| `timeSyncOffsetMs` | 時間同步偏移量（毫秒）|
| `matchDuration` | 對局持續時間（秒）|
| `network` | 網路統計：avg / peak ping、掉包率、斷線次數 |
| `stateFingerprints` | Desync Fingerprint 輸出，每 30～60 tick 一筆，上限 200 筆（見 Section 4.8）|

> **無後端模式限制**：單方 log 無法自動與對方關聯。desync 診斷需請對方也按 F8，並透過 Match ID 手動對齊。

---

### 5.5 anomaly.json

核心診斷資料。使用 EventRingBuffer 的 EventEntry 格式（`system` + `eventType` + `detail` + `time` + `frame`），確保欄位與程式碼完全一致。`finalSnapshot` 為最後一次 CriticalSnapshot.Dump 的輸出，可為 null。

```json
{
  "anomalyCount": 2,
  "events": [
    {
      "system":    "LIFECYCLE",
      "eventType": "ListenerImbalance",
      "detail":    "[LEAK] page=Hair_12345 addCount=4 frame=18543",
      "time":      "12:01:30.123",
      "frame":     18543
    },
    {
      "system":    "SHOP_STATE",
      "eventType": "InvalidTransition",
      "detail":    "[ILLEGAL] old=HairTab new=HairTab source=ShopRefresh.Tick frame=18601",
      "time":      "12:01:33.001",
      "frame":     18601
    }
  ],
  "finalSnapshot": {
    "trigger":         "ShopOpen_Failed",
    "shopState":       "HairTab",
    "currentTab":      "HairTab",
    "listenerBalance": 2,
    "frame":           18601,
    "time":            "12:01:33.001"
  }
}
```

**記錄原則**：只記錄異常事件，不記錄正常 gameplay。異常類型的完整清單見 Section 13。

---

### 5.6 screenshot.png

**用途**：F8 觸發時的遊戲畫面截圖，保留 UI 可讀性與文字清晰度。

#### 縮放策略

| 解析度 | 處理方式 |
|---|---|
| ≤ 1920×1080 | 不縮放，保留原始解析度 |
| > 1920×1080 | 縮放至 1920×1080，使用 bilinear filtering |

> ⚠️ **禁止無條件縮至 1280×720。** UI bug 診斷依賴小型文字（Hover 狀態、語系標籤、按鈕文字），過度縮圖會導致無法辨識 UI 細節。

#### 格式與大小

| 格式 | 適用情境 |
|---|---|
| PNG | UI 問題（主要用途，無損壓縮）|
| JPG | 一般畫面問題（非 UI 細節）|

- 建議 ≤ 5MB
- 禁止：4K 原圖、無壓縮超大 PNG

---

### 5.7 player.log

**來源**：`Application.persistentDataPath`

#### 擷取策略

- 預設：最後 **500 行**
- 若觸發 CriticalSnapshot：額外保留 **CriticalSnapshot 前 100 行**

**原因**：某些 UI bug（ListenerLeak、Recursive Refresh、OnEnable 未解除）的 root cause 可能發生於數分鐘前。若 log window 過短，只剩最終崩壞結果而失去真正 trigger。

**維持 500 行，不縮短。** 目前無後端、無 telemetry，player.log 是唯一 runtime 記錄來源，診斷完整性優先於 ZIP 大小。

---

## 6. 封裝層（F8 觸發流程）

### 6.1 BugReportManager

F8 觸發、截圖、資料彙整、ZIP 啟動的主控腳本。

```csharp
// BugReportManager.cs
using UnityEngine;
using UnityEngine.SceneManagement;
using System.IO;
using System.Collections;
using System.Collections.Generic;

public class BugReportManager : MonoBehaviour
{
    public static BugReportManager Instance;
    [SerializeField] private GameObject reportPanelPrefab;

    // ✅ v1.4：Immutable Snapshot 模式 — 截圖 encode 在主執行緒完成後存為 byte[]
    // worker 只會拿到這份 pure managed data，不再碰 UnityEngine.Object
    private byte[] _screenshotPng = null;

    // ✅ v1.4：CancellationToken — 遊戲退出時安全中止背景 task
    private CancellationTokenSource _cts = new CancellationTokenSource();

    // ✅ v1.4：場景初始化時快取 LocalPlayer，不在 BuildPackageData 每次 FindWithTag
    private PlayerStateProvider _cachedPlayerState;

    // Gameplay event log（與 Anomaly Buffer 分開）
    // ✅ v1.2：改用 struct 儲存 timestamp，不在 GetRecentGameplay 做 string parse
    // 原版 DateTime.TryParse(line.Substring(0, 12)) 對格式長度有 bug（"12:01:29.412" 實為 12 字元但含毫秒不保證對齊）
    private struct GameplayEntry
    {
        public System.DateTime ts;
        public string text;
    }
    private Queue<GameplayEntry> _gameplayBuffer = new Queue<GameplayEntry>();
    private bool _isCooldown = false;

    void Awake()
    {
        Instance = this;
        // ✅ 場景初始化時快取，避免 BuildPackageData 裡每次 FindWithTag
        var player = GameObject.FindWithTag("LocalPlayer");
        if (player != null) _cachedPlayerState = player.GetComponent<PlayerStateProvider>();
    }

    void OnDestroy()
    {
        // ✅ v1.4：通知所有背景 task 停止，防止遊戲退出時 worker 仍寫檔
        _cts.Cancel();
        _cts.Dispose();
    }

    void Update()
    {
        if (Input.GetKeyDown(KeyCode.F8) && !_isCooldown)
            StartCoroutine(TriggerReport());
    }

    // 記錄一般 gameplay 事件（不寫 anomaly buffer）
    public void LogGameplay(string actor, string eventName, string extra = "")
    {
        var now = System.DateTime.Now;
        string ts = now.ToString("HH:mm:ss.fff");
        string line = $"{ts} [{actor}] {eventName}";
        if (!string.IsNullOrEmpty(extra)) line += $" {extra}";
        _gameplayBuffer.Enqueue(new GameplayEntry { ts = now, text = line });
        while (_gameplayBuffer.Count > 500) _gameplayBuffer.Dequeue();
    }

    public IEnumerator TriggerReport()
    {
        _isCooldown = true;
        // ✅ v1.4：截圖結果直接存為 byte[]（_screenshotPng），不再依賴臨時檔案路徑
        yield return StartCoroutine(CaptureScreenshotAsync(png => _screenshotPng = png));

        var panel = Instantiate(reportPanelPrefab);
        panel.GetComponent<ReportPanel>().Init(_screenshotPng);

        yield return new WaitForSeconds(10f);
        _isCooldown = false;
    }

    // ✅ v1.4：Immutable Snapshot 模式
    // EncodeToPNG 在主執行緒執行後回傳 byte[]（pure managed），不再回傳 path
    // worker thread 後續只會拿到此 byte[]，完全不碰 Texture2D / File
    private IEnumerator CaptureScreenshotAsync(System.Action<byte[]> onDone)
    {
        yield return new WaitForEndOfFrame();

        Texture2D tex = ScreenCapture.CaptureScreenshotAsTexture();

        // ✅ 主執行緒：Texture2D encode（Unity Object，不可跨 thread）
        byte[] png = tex.EncodeToPNG();
        UnityEngine.Object.Destroy(tex);

        // ✅ 備份截圖到磁碟（背景 IO，不 block 主執行緒）
        string dir  = Application.persistentDataPath + "/BugReports/";
        string path = dir + $"screenshot_{System.DateTime.Now:yyyyMMdd_HHmmss}.png";
        Directory.CreateDirectory(dir);
        _ = System.Threading.Tasks.Task.Run(() => File.WriteAllBytes(path, png), _cts.Token);

        onDone?.Invoke(png);
    }

    public async void SubmitLocalZip(string title, string description, string category)
    {
        // ✅ v1.4：BuildPackageData 在主執行緒完成（所有 Unity Object 讀取在此）
        // 回傳 pure DTO（BugPackageData），可安全傳入 worker
        var data = BuildPackageData(title, description, category);

        // ✅ v1.4：傳入 _screenshotPng（byte[]）與 _cts.Token
        // worker 不再讀 screenshotPath，不再碰任何 Unity Object
        string zipPath = await LocalZipPacker.PackAsync(data, _screenshotPng, _cts.Token);
        if (zipPath == null)
        {
            Debug.LogError("[BugReport] 打包失敗");
            return;
        }

        TryRevealInExplorer(zipPath);
        BrowserLauncher.Open(BrowserLauncher.TargetPage.Discord);
        OnLocalZipReady?.Invoke(zipPath, data.MatchId);
    }

    public event System.Action<string, string> OnLocalZipReady;

    private void TryRevealInExplorer(string path)
    {
#if UNITY_STANDALONE_WIN
        try
        {
            // ✅ 穩定寫法（確保以資料夾方式開啟）
            string arg = $"/select,\"{path.Replace("/", "\\")}\"";
            System.Diagnostics.Process.Start("explorer.exe", arg);
        }
        catch (System.Exception e)
        {
            Debug.LogWarning($"[BugReport] 開啟資料夾失敗：{e.Message}");
        }
#endif
    }

    public BugPackageData BuildPackageData(string title, string description, string category)
    {
        var d = new BugPackageData
        {
            ReportId     = System.Guid.NewGuid().ToString("N").Substring(0, 8),
            Timestamp    = System.DateTime.UtcNow.ToString("o"),
            AppVersion   = Application.version,
            Scene        = SceneManager.GetActiveScene().name,
            Title        = title,
            Description  = description,
            Category     = category
        };

        var match = MatchInfoProvider.Instance;
        if (match != null)
        {
            d.MatchId    = match.MatchId;
            d.PlayerId   = match.PlayerId;
            d.OpponentId = match.OpponentId;
        }

        // ✅ v1.4：改用 Awake 快取的 _cachedPlayerState，不每次 FindWithTag + GetComponent
        if (_cachedPlayerState != null)
        {
            d.Position    = _cachedPlayerState.transform.position.ToString("F1");
            d.PlayerState = _cachedPlayerState.State ?? "";
        }

        var tick = TickSyncManager.Instance;
        if (tick != null)
        {
            d.LocalTick        = tick.LocalTick;
            d.RemoteTick       = tick.RemoteTick;
            d.TickDelta        = tick.LocalTick - tick.RemoteTick;
            d.TimeSyncOffsetMs = tick.TimeSyncOffsetMs;
        }

        d.GameplayEvents    = GetRecentGameplay(60);
        d.AnomalyEvents     = EventRingBuffer.Instance?.GetAll() ?? new List<EventEntry>();
        d.StateFingerprints = StateHashCollector.Instance?.GetAll()
                              ?? new List<StateHashCollector.FingerprintEntry>();
        d.Network           = NetworkStatsCollector.Instance?.GetSnapshot() ?? new NetworkStatsSnapshot();
        d.LogTail           = ReadLogTail(500);
        return d;
    }

    private List<string> GetRecentGameplay(int seconds)
    {
        // ✅ v1.2：直接比較 struct DateTime，不做 string substring parse（原版有 parse fail 風險）
        var cutoff = System.DateTime.Now.AddSeconds(-seconds);
        var result = new List<string>();
        foreach (var entry in _gameplayBuffer)
        {
            if (entry.ts >= cutoff)
                result.Add(entry.text);
        }
        return result;
    }

    private string ReadLogTail(int lines)
    {
        try
        {
            string logPath = System.IO.Path.GetFullPath(
                Application.persistentDataPath + "/../Player.log");
            if (!File.Exists(logPath)) return "[log not found]";
            var all = File.ReadAllLines(logPath);
            int start = Mathf.Max(0, all.Length - lines);
            return string.Join("\n", all, start, all.Length - start);
        }
        catch (System.Exception e) { return $"[log read error: {e.Message}]"; }
    }
}

// 封裝資料的統一結構（對應 ZIP 各檔案的原始資料）
public class BugPackageData
{
    public string ReportId, Timestamp, AppVersion, Scene;
    public string Title, Description, Category;
    public string MatchId, PlayerId, OpponentId;
    public string Position, PlayerState, OpponentState;
    public int LocalTick, RemoteTick, TickDelta, TimeSyncOffsetMs;
    public List<string> GameplayEvents;
    public List<EventEntry> AnomalyEvents;
    public List<StateHashCollector.FingerprintEntry> StateFingerprints;
    public NetworkStatsSnapshot Network;
    public string LogTail;
}
```

---

### 6.2 NetworkStatsCollector

```csharp
// NetworkStatsCollector.cs
using UnityEngine;
using Mirror;

public class NetworkStatsCollector : MonoBehaviour
{
    public static NetworkStatsCollector Instance;

    public float PingAvg { get; private set; }
    public float PingPeak { get; private set; }
    public float PacketLossAvg { get; private set; }
    public float PacketLossPeak { get; private set; }
    public int DisconnectCount { get; private set; }
    public string LastDisconnectReason { get; private set; } = "";

    private float _pingSampleSum = 0f;
    private int _pingSampleCount = 0;
    private float _timer = 0f;

    void Awake() => Instance = this;

    void Update()
    {
        _timer += Time.deltaTime;
        if (_timer >= 1f)
        {
            _timer = 0f;
            float currentPing = (float)(NetworkTime.rtt * 1000);
            _pingSampleSum += currentPing;
            _pingSampleCount++;
            PingAvg = _pingSampleSum / _pingSampleCount;
            if (currentPing > PingPeak) PingPeak = currentPing;
        }
    }

    public void RecordDisconnect(string reason)
    {
        DisconnectCount++;
        LastDisconnectReason = reason;
        // 斷線屬於 gameplay event，寫進 BugReportManager 的 gameplay log
    }

    public NetworkStatsSnapshot GetSnapshot() => new NetworkStatsSnapshot
    {
        pingAvg          = (int)PingAvg,
        pingPeak         = (int)PingPeak,
        packetLossAvg    = PacketLossAvg,
        packetLossPeak   = PacketLossPeak,
        disconnectCount  = DisconnectCount,
        disconnectReason = LastDisconnectReason
    };
}

[System.Serializable]
public class NetworkStatsSnapshot
{
    public int pingAvg, pingPeak;
    public float packetLossAvg, packetLossPeak;
    public int disconnectCount;
    public string disconnectReason;
}
```

---

### 6.3 TickSyncManager

```csharp
// TickSyncManager.cs
using UnityEngine;

public class TickSyncManager : MonoBehaviour
{
    public static TickSyncManager Instance;

    public int LocalTick { get; private set; }
    public int RemoteTick { get; private set; }
    public int TimeSyncOffsetMs { get; private set; }

    void Awake() => Instance = this;
    void FixedUpdate() => LocalTick++;

    public void UpdateRemoteTick(int remoteTick, long remoteTimestampMs)
    {
        RemoteTick = remoteTick;
        long localMs = System.DateTimeOffset.UtcNow.ToUnixTimeMilliseconds();
        TimeSyncOffsetMs = (int)(localMs - remoteTimestampMs);

        if (Mathf.Abs(LocalTick - RemoteTick) > 5)
            Debug.LogWarning($"[TickSync] local={LocalTick} remote={RemoteTick} delta={LocalTick - RemoteTick}");
    }
}
```

---

### 6.4 LocalZipPacker

```csharp
// LocalZipPacker.cs
using System.IO;
using System.IO.Compression;
using System.Text;
using System.Collections.Generic;
using UnityEngine;

public static class LocalZipPacker
{
        // ✅ v1.2：ZIP 打包移至背景執行緒（避免 HDD / Steam Deck SD Card 造成主執行緒卡 0.5~2 秒）
        // ✅ v1.4：加入 screenshotPng（byte[]）參數，worker 不再需要碰 screenshotPath 的 File.Read
        //          加入 CancellationToken，遊戲退出時可安全中止
        public static async System.Threading.Tasks.Task<string> PackAsync(
            BugPackageData d,
            byte[] screenshotPng,
            System.Threading.CancellationToken token = default)
        {
            return await System.Threading.Tasks.Task.Run(
                () => PackInternal(d, screenshotPng), token);
        }

        /// <summary>
        /// 同步版本保留供 Editor 工具使用（非遊戲主路徑）。
        /// 遊戲主路徑請改用 PackAsync。
        /// </summary>
        public static string Pack(BugPackageData d, string screenshotPath)
        {
            byte[] png = File.Exists(screenshotPath) ? File.ReadAllBytes(screenshotPath) : null;
            return PackInternal(d, png);
        }

        // ✅ v1.4：內部實作接受 byte[]，worker thread 永遠不碰 Unity Object 或 File 系統讀取
        private static string PackInternal(BugPackageData d, byte[] screenshotPng)
    {
        try
        {
            string dir = Application.persistentDataPath + "/BugReports/";
            Directory.CreateDirectory(dir);
            string zipPath = dir + $"BugReport_{System.DateTime.Now:yyyy-MM-dd_HH-mm-ss}.zip";

            using (var zip = ZipFile.Open(zipPath, ZipArchiveMode.Create))
            {
                // 1. screenshot.png（直接寫 byte[]，不再讀 screenshotPath 檔案）
                if (screenshotPng != null && screenshotPng.Length > 0)
                {
                    var entry = zip.CreateEntry("screenshot.png", CompressionLevel.Fastest);
                    using var s = entry.Open();
                    s.Write(screenshotPng, 0, screenshotPng.Length);
                }

                // 2. metadata.txt（人類可讀摘要）
                WriteTextEntry(zip, "metadata.txt", BuildMetadata(d));

                // 3. report.json（玩家資訊 + gameplay events）
                WriteTextEntry(zip, "report.json", BuildReportJson(d));

                // 4. match.json（對局資訊 + stateFingerprints）
                WriteTextEntry(zip, "match.json", BuildMatchJson(d));

                // 5. anomaly.json（異常事件 + finalSnapshot）
                WriteTextEntry(zip, "anomaly.json", BuildAnomalyJson(d));

                // 6. player.log
                if (!string.IsNullOrEmpty(d.LogTail))
                    WriteTextEntry(zip, "player.log", d.LogTail);
            }

            Debug.Log($"[BugReport] ZIP 已生成：{zipPath}");
            return zipPath;
        }
        catch (System.Exception e)
        {
            Debug.LogError($"[BugReport] ZIP 打包失敗：{e.Message}");
            return null;
        }
    }

    private static void WriteTextEntry(ZipArchive zip, string entryName, string content)
    {
        var entry = zip.CreateEntry(entryName, CompressionLevel.Fastest);
        using var writer = new StreamWriter(entry.Open(), Encoding.UTF8);
        writer.Write(content);
    }

    // --- metadata.txt ---
    private static string BuildMetadata(BugPackageData d)
    {
        var anomalyTypes = new List<string>();
        foreach (var e in d.AnomalyEvents ?? new List<EventEntry>())
            if (!anomalyTypes.Contains(e.eventType))
                anomalyTypes.Add(e.eventType);

        return
            $"Report ID  : {d.ReportId}\n" +
            $"Timestamp  : {d.Timestamp}\n" +
            $"Version    : {d.AppVersion}\n" +
            $"Match ID   : {d.MatchId}\n" +
            $"Player     : {d.PlayerId}  vs  {d.OpponentId}\n" +
            $"Anomalies  : {d.AnomalyEvents?.Count ?? 0}" +
            (anomalyTypes.Count > 0 ? $"  ({string.Join(", ", anomalyTypes)})" : "") + "\n" +
            $"Screenshot : screenshot.png\n" +
            $"Log Lines  : 500\n";
    }

    // --- report.json ---
    private static string BuildReportJson(BugPackageData d)
    {
        var sb = new StringBuilder();
        sb.AppendLine("{");
        sb.AppendLine($"  \"specVersion\":    \"1.4\",");  // ✅ v1.4 新增
        sb.AppendLine($"  \"reportId\":      \"{d.ReportId}\",");
        sb.AppendLine($"  \"timestamp\":     \"{d.Timestamp}\",");
        sb.AppendLine($"  \"buildVersion\":  \"{d.AppVersion}\",");
        sb.AppendLine($"  \"title\":         \"{EscapeJson(d.Title)}\",");
        sb.AppendLine($"  \"category\":      \"{EscapeJson(d.Category)}\",");
        sb.AppendLine($"  \"description\":   \"{EscapeJson(d.Description)}\",");
        sb.AppendLine($"  \"scene\":         \"{d.Scene}\",");
        sb.AppendLine($"  \"position\":      \"{d.Position}\",");
        sb.AppendLine($"  \"playerState\":   \"{d.PlayerState}\",");
        sb.AppendLine($"  \"opponentState\": \"{d.OpponentState}\",");
        sb.AppendLine("  \"gameplayEvents\": [");
        var events = d.GameplayEvents ?? new List<string>();
        for (int i = 0; i < events.Count; i++)
        {
            string comma = i < events.Count - 1 ? "," : "";
            sb.AppendLine($"    \"{EscapeJson(events[i])}\"{comma}");
        }
        sb.AppendLine("  ]");
        sb.AppendLine("}");
        return sb.ToString();
    }

    // --- match.json（含 stateFingerprints）---
    private static string BuildMatchJson(BugPackageData d)
    {
        var sb = new StringBuilder();
        sb.AppendLine("{");
        sb.AppendLine($"  \"matchId\":          \"{d.MatchId}\",");
        sb.AppendLine($"  \"playerId\":         \"{d.PlayerId}\",");
        sb.AppendLine($"  \"opponentId\":       \"{d.OpponentId}\",");
        sb.AppendLine($"  \"localTick\":        {d.LocalTick},");
        sb.AppendLine($"  \"remoteTick\":       {d.RemoteTick},");
        sb.AppendLine($"  \"tickDelta\":        {d.TickDelta},");
        sb.AppendLine($"  \"timeSyncOffsetMs\": {d.TimeSyncOffsetMs},");
        sb.AppendLine("  \"network\": {");
        var n = d.Network ?? new NetworkStatsSnapshot();
        sb.AppendLine($"    \"pingAvg\":        {n.pingAvg},");
        sb.AppendLine($"    \"pingPeak\":       {n.pingPeak},");
        sb.AppendLine($"    \"packetLossAvg\":  {n.packetLossAvg:F3},");
        sb.AppendLine($"    \"packetLossPeak\": {n.packetLossPeak:F3},");
        sb.AppendLine($"    \"disconnectCount\":{n.disconnectCount},");
        sb.AppendLine($"    \"disconnectReason\":\"{EscapeJson(n.disconnectReason)}\"");
        sb.AppendLine("  },");
        sb.AppendLine("  \"stateFingerprints\": [");
        var fps = d.StateFingerprints ?? new List<StateHashCollector.FingerprintEntry>();
        for (int i = 0; i < fps.Count; i++)
        {
            string comma = i < fps.Count - 1 ? "," : "";
            sb.AppendLine($"    {{ \"tick\": {fps[i].tick}, \"hash\": \"{fps[i].hash}\" }}{comma}");
        }
        sb.AppendLine("  ]");
        sb.AppendLine("}");
        return sb.ToString();
    }

    // --- anomaly.json（EventEntry 結構 + causalChains + finalSnapshot）---
    private static string BuildAnomalyJson(BugPackageData d)
    {
        var sb = new StringBuilder();
        var events = d.AnomalyEvents ?? new List<EventEntry>();
        sb.AppendLine("{");
        sb.AppendLine($"  \"specVersion\":  \"1.4\",");  // ✅ v1.4 新增：供 Agent / Skill 版本判斷
        sb.AppendLine($"  \"anomalyCount\": {events.Count},");
        sb.AppendLine("  \"events\": [");
        for (int i = 0; i < events.Count; i++)
        {
            var e = events[i];
            string comma = i < events.Count - 1 ? "," : "";
            sb.AppendLine(
                $"    {{ \"system\": \"{e.system}\", \"eventType\": \"{e.eventType}\", " +
                $"\"detail\": \"{EscapeJson(e.detail)}\", \"time\": \"{e.time}\", \"frame\": {e.frame} }}{comma}");
        }
        sb.AppendLine("  ],");

        // ✅ v1.4 整合：causalChains — F8 時由 AnomalyCausalChain.Analyze() 生成，
        // 輸出在 events 之後、finalSnapshot 之前（供 Skill Step 3 Path A 讀取）
        var chains = AnomalyCausalChain.Analyze(events);
        sb.AppendLine("  \"causalChains\": [");
        for (int i = 0; i < chains.Count; i++)
        {
            var c = chains[i];
            string seq   = string.Join("\", \"", c.sequence);
            string comma = i < chains.Count - 1 ? "," : "";
            sb.AppendLine(
                $"    {{ \"rootCause\": \"{c.rootCause}\", " +
                $"\"sequence\": [\"{seq}\"], " +
                $"\"startFrame\": {c.startFrame}, " +
                $"\"endFrame\": {c.endFrame}, " +
                $"\"spanFrames\": {c.spanFrames} }}{comma}");
        }
        sb.AppendLine("  ],");

        // finalSnapshot：取 EventRingBuffer 中最後一筆 CriticalDump 的 detail
        var snapshot = events.FindLast(e => e.eventType == "CriticalDump");
        if (snapshot.detail != null)
            sb.AppendLine($"  \"finalSnapshot\": {snapshot.detail}");
        else
            sb.AppendLine("  \"finalSnapshot\": null");

        sb.AppendLine("}");
        return sb.ToString();
    }

    private static string EscapeJson(string s) =>
        s?.Replace("\\", "\\\\").Replace("\"", "\\\"").Replace("\n", "\\n") ?? "";
}
```

---

### 6.5 BrowserLauncher

```csharp
// BrowserLauncher.cs
using UnityEngine;

public static class BrowserLauncher
{
    // 填入 Discord 頻道連結或 Google Form 連結，二擇一
    private const string DiscordChannelUrl = "https://discord.com/channels/YOUR_SERVER/YOUR_CHANNEL";
    private const string GoogleFormUrl     = "https://forms.gle/YOUR_FORM_ID";

    public enum TargetPage { Discord, GoogleForm }

    public static void Open(TargetPage target = TargetPage.Discord)
    {
        string url = target == TargetPage.Discord ? DiscordChannelUrl : GoogleFormUrl;
        Application.OpenURL(url);
        Debug.Log($"[BugReport] 開啟瀏覽器：{url}");
    }
}
```

---

### 6.6 MatchInfoProvider（v1.4 補充定義）

> ⚠️ **v1.4 補充**：原始規格 Section 18 列為 Tier 1 必做腳本，但缺少類別定義。本節補完 contract。

```csharp
// MatchInfoProvider.cs
using UnityEngine;

/// <summary>
/// MatchInfoProvider
/// 用途：快取對局 matchId / playerId / opponentId，供 BugReportManager.BuildPackageData() 讀取
/// 插入點：Network 層 Match 開始回調（OnMatchStart / OnRoomJoined 等），由業主確認實際方法名
/// 插入方式：Match 開始時呼叫 SetMatchInfo()，F8 觸發前必須已呼叫過
/// 算力影響：非 hot path，Match 開始時執行一次
/// </summary>
public class MatchInfoProvider : MonoBehaviour
{
    public static MatchInfoProvider Instance;

    public string MatchId    { get; private set; } = "";
    public string PlayerId   { get; private set; } = "";
    public string OpponentId { get; private set; } = "";

    void Awake() => Instance = this;

    /// <summary>
    /// Match 開始時由網路層（Photon / Mirror / 自訂 NetworkManager）呼叫。
    /// Source of Truth 由業主確認並填入實際 API。
    /// </summary>
    public void SetMatchInfo(string matchId, string playerId, string opponentId)
    {
        MatchId    = matchId;
        PlayerId   = playerId;
        OpponentId = opponentId;
    }
}
```

**Source of Truth（業主確認後填入 SetMatchInfo 呼叫位置）：**

| 欄位 | 候選來源 | 確認狀態 |
|---|---|---|
| `MatchId` | `PhotonNetwork.CurrentRoom.Name` 或 `NetworkManager.MatchId` | ⚠️ 待確認 |
| `PlayerId` | `PhotonNetwork.LocalPlayer.NickName` 或本地玩家識別碼 | ⚠️ 待確認 |
| `OpponentId` | 對手玩家的 NetworkId 或 NickName | ⚠️ 待確認 |

**注意**：
- `DiagnosticsBootstrap` 的 `Debug.Assert` **不驗證** `MatchInfoProvider.Instance`（Match 可能尚未開始），這是刻意設計
- F8 按下時若 `MatchInfoProvider.Instance == null` 或 `MatchId == ""`，ZIP 的 matchId 欄位將為空字串，影響雙方 ZIP 的 MatchCorrelate 能力

---

### 7.1 無後端模式（主力）

**封裝流程：**

```
玩家按 F8
    ↓
CaptureScreenshotAsync（非同步，不 block 主執行緒）
    ↓
收集 Runtime Snapshot / Anomaly Buffer / MatchInfo / Log
    ↓
LocalZipPacker.Pack()  ← ⚠️ 主執行緒同步操作，約 100～300ms
    ↓
TryRevealInExplorer（Windows Explorer 選中 ZIP）
    ↓
BrowserLauncher.Open()（開 Discord / Google Form）
    ↓
玩家手動拖曳 ZIP 上傳
```

**玩家提交方案：**

| 方案 | 優點 | 缺點 |
|---|---|---|
| Discord | 最簡單，QA 可即時互動 | 不適合大量玩家，無法自動聚類 |
| Google Form | 結構化，可分類 | 無即時互動，有容量限制 |
| GitHub Issue | 可直接連結 commit | 僅限內部 QA，一般玩家門檻過高 |

**適合規模**：玩家數 < 50 人的朋友測試、內部 QA、Demo 版本。

---

### 7.2 Discord Webhook 模式（選配）

> 適用於早期封測（< 50 人），需要比無後端模式更低的玩家操作成本。
> 此模式在遊戲 client 中持有 webhook URL，有暴露風險，請評估後決定是否啟用。

```csharp
// DiscordWebhookUploader.cs（選配腳本，非主力路徑）
using System.Collections;
using System.Text;
using UnityEngine;
using UnityEngine.Networking;

public class DiscordWebhookUploader : MonoBehaviour
{
    [SerializeField] private string webhookUrl = "https://discord.com/api/webhooks/YOUR_WEBHOOK";

    public IEnumerator UploadReport(BugPackageData d, string screenshotPath)
    {
        string screenshotUrl = "";
        yield return StartCoroutine(UploadScreenshot(screenshotPath, url => screenshotUrl = url));

        int anomalyCount = d.AnomalyEvents?.Count ?? 0;
        string anomalySummary = anomalyCount > 0
            ? $"⚠️ Anomaly Events: `{anomalyCount}` detected"
            : "✅ No anomaly events";

        string message =
            $"**Bug Report #{d.ReportId}**\n" +
            $"Match: `{d.MatchId}` | Player: `{d.PlayerId}`\n" +
            $"Scene: `{d.Scene}` | Version: `{d.AppVersion}`\n" +
            $"Ping Peak: `{d.Network.pingPeak}ms` | Disconnect: `{d.Network.disconnectCount}`\n" +
            $"{anomalySummary}\n" +
            $"Screenshot: {screenshotUrl}\n\n" +
            $"```{d.Description}```";

        var bodyJson = $"{{\"content\":\"{EscapeJson(message)}\"}}";
        using var req = new UnityWebRequest(webhookUrl, "POST");
        byte[] bytes = Encoding.UTF8.GetBytes(bodyJson);
        req.uploadHandler   = new UploadHandlerRaw(bytes);
        req.downloadHandler = new DownloadHandlerBuffer();
        req.SetRequestHeader("Content-Type", "application/json");
        yield return req.SendWebRequest();

        if (req.result == UnityWebRequest.Result.Success)
            Debug.Log($"[BugReport] 上傳成功 ID={d.ReportId}");
        else
            Debug.LogError($"[BugReport] 上傳失敗: {req.error}");
    }

    private string EscapeJson(string s) =>
        s.Replace("\\", "\\\\").Replace("\"", "\\\"").Replace("\n", "\\n");

    private IEnumerator UploadScreenshot(string path, System.Action<string> callback)
    {
        if (!System.IO.File.Exists(path)) { callback(""); yield break; }
        var formData = new WWWForm();
        byte[] imgBytes = System.IO.File.ReadAllBytes(path);
        formData.AddBinaryData("file", imgBytes, "screenshot.png", "image/png");
        using var req = UnityWebRequest.Post(webhookUrl, formData);
        yield return req.SendWebRequest();
        callback(req.result == UnityWebRequest.Result.Success ? "[screenshot attached]" : "");
    }
}
```

---

### 7.3 三模式對比

| 模式 | 適合階段 | 玩家負擔 | 開發成本 | 資料查詢能力 | API Key |
|---|---|---|---|---|---|
| **無後端（LocalZip）** | 朋友測試 / Demo | 高（手動上傳）| 極低 | 無（人工翻閱）| ❌ 不需要 |
| **Discord Webhook** | 早期封測（< 50 人）| 低（一鍵）| 低 | 低（無法聚類）| ⚠️ webhook URL |
| **Supabase** | 正式封測 / 上線 | 低（一鍵）| 中 | 高（SQL 查詢）| ✅ 需要 |

> **建議路線**：LocalZip → Discord Webhook → Supabase，不要跳級。

---

## 8. UI 設計

### 觸發方式

- **F8 快捷鍵**（必做）：視覺 bug 一瞬間就消失，等玩家找按鈕已來不及
- **Report Issue 按鈕**（建議）：UI 右上角或 Esc 選單

### 回報面板（無後端模式）

```
┌──────────────────────────────────────┐
│  Report Issue                        │
│  Match: room_18273                   │
│  ── ⚠️ 2 anomaly events ──           │
│  [截圖預覽]                          │
│                                      │
│  標題（選填）：                      │
│  [___________________________________]│
│                                      │
│  描述問題（選填）：                  │
│  [___________________________________]│
│                                      │
│  [📦 產生回報包 + 開啟上傳頁面]      │
│                                      │
│  [取消]                              │
└──────────────────────────────────────┘

點擊後：
  ├── 顯示「正在生成回報包…」
  ├── 生成 BugReport_YYYY-MM-DD_HH-mm-ss.zip
  ├── Windows Explorer 選中 ZIP
  ├── 顯示「✅ 回報包已生成，請拖曳到 Discord 上傳」
  ├── 顯示 Match ID：room_18273（方便貼到 Discord / Form）
  └── 自動開瀏覽器 → Discord 頻道 / Google Form
```

> 面板上固定顯示 anomaly count（例如「⚠️ 2 anomaly events」），讓開發者第一眼就知道 Diagnosis Layer 有沒有抓到東西。

---

## 9. Build Mode 環境分級

```
Development Build（開發機）
  ├── 全部 Detector 開啟
  └── CriticalSnapshot prettyPrint: true（易讀）

QA Build（測試機）
  ├── ListenerBalanceDetector  ✓
  ├── FrequencyDetector        ✓
  ├── UITextAnomalyDetector（白名單）✓
  └── ShopStateTracer          ✓

Release Build（正式環境）
  ├── CriticalSnapshot         ✓（關鍵失敗必要）
  ├── NetworkStatsCollector    ✓（網路診斷必要）
  ├── StateHashCollector       ✓（全 Build Mode，見 Section 4.8）
  └── FrequencyDetector 等其餘 Detector 用 #if 完整移除
```

**建議實作（Compile Symbol）：**

```csharp
public static void Record(string eventKey)
{
#if UNITY_EDITOR || DEVELOPMENT_BUILD || QA_BUILD
    // detector 邏輯
#endif
}
```

> ⚠️ **QA Build 送測給玩家時須確認 `#if` 已正確設定**，否則玩家回報的 lag 可能是 FrequencyDetector 造成的，而非遊戲本身。

---

## 10. Thread Ownership 規格

> **v1.4 新增。** 本節是整個系統的執行緒安全基礎，所有程式碼改動必須以本節為第一校驗標準。

### 核心原則

```
Unity Object = Main Thread Owned Resource

擁有者決定 thread，不是「看起來能不能跑」。
```

### Ownership 邊界表

| 資源類型 | Owner | 禁止操作 |
|---|---|---|
| `Texture2D` / `RenderTexture` | **主執行緒** | 禁止在 Task.Run 中讀寫或 Encode |
| `GameObject` / `Transform` | **主執行緒** | 禁止跨 thread 讀取 position / scale |
| `TMP_Text` / `Text` | **主執行緒** | 禁止跨 thread 修改 .text |
| `Material` / `Mesh` | **主執行緒** | 禁止跨 thread 讀取 sharedMesh |
| `Animator` / `AudioSource` | **主執行緒** | 禁止跨 thread 呼叫任何方法 |
| `ScriptableObject` 實例 | **主執行緒** | 禁止跨 thread 讀寫欄位 |
| `byte[]` / `string` / struct | **任意 thread** | 無限制（pure managed data）|
| `FileStream` / `ZipArchive` | **Worker** | 禁止在主執行緒長時間同步執行 |
| `JSON string` / hash 計算 | **任意 thread** | 無限制 |

### Immutable Snapshot 模式（本系統強制規範）

Worker thread 永遠只碰 **snapshot DTO**，不碰 live Unity object。

```csharp
// ✅ 正確模式：主執行緒建立純資料快照，worker 只操作快照
void OnF8()
{
    // 主執行緒：從 Unity Object 提取資料 → 純 C# DTO
    BugPackageData snapshot = BuildPackageData(...); // 所有 UnityEngine.Object 讀取在此完成
    byte[] pngBytes = tex.EncodeToPNG();             // Texture2D encode 在主執行緒

    // Worker：只碰 snapshot 和 byte[]，不碰任何 Unity Object
    _ = Task.Run(() => LocalZipPacker.PackFromSnapshot(snapshot, pngBytes, token));
}

// ❌ 錯誤模式：worker 讀 live state
_ = Task.Run(() =>
{
    var pos = player.transform.position; // 危險：跨 thread 讀 Transform
    tex.EncodeToPNG();                   // 危險：跨 thread 操作 Texture2D
});
```

**`BugPackageData` 的設計保證**：此 class 中所有欄位均為 pure managed types（`string`、`int`、`List<struct>`），無任何 `UnityEngine.Object` 參考。主執行緒呼叫 `BuildPackageData()` 後，此物件可安全傳入 worker thread。

### Shutdown Safety（CancellationToken）

`Task.Run` 必須傳入 `CancellationToken`，防止遊戲退出時 worker 仍在寫檔。

```csharp
// BugReportManager.cs（v1.4 新增）
private CancellationTokenSource _cts = new CancellationTokenSource();

void OnDestroy()
{
    _cts.Cancel();   // 通知所有背景 task 停止
    _cts.Dispose();
}

// 呼叫方傳入 token
string zipPath = await LocalZipPacker.PackAsync(snapshot, pngBytes, _cts.Token);
```

```csharp
// LocalZipPacker.cs（v1.4 修正）
public static async Task<string> PackAsync(
    BugPackageData d, byte[] screenshotPng, CancellationToken token = default)
{
    return await Task.Run(() =>
    {
        token.ThrowIfCancellationRequested(); // ← 進入前先檢查
        return PackInternal(d, screenshotPng);
    }, token);
}
```

### Component 快取規則

`GetComponent<T>()` 跨越 native/managed boundary，不是免費呼叫。本系統所有 MonoBehaviour 必須快取常用組件參考。

```csharp
// ✅ 正確：Awake 快取，後續直接用
private TMP_Text _cachedLabel;
void Awake() => _cachedLabel = GetComponent<TMP_Text>();
void OnUpdate() => _cachedLabel.text = "..."; // 直接用快取

// ❌ 錯誤：每次呼叫都跨 native bridge
void OnUpdate() => GetComponent<TMP_Text>().text = "...";
```

**本系統受影響組件**：`ReportPanel`（TMP_Text 參考）、`NetworkStatsCollector`（Mirror API 呼叫）、`BugReportManager`（`FindWithTag` 應改為場景初始化時快取）。

---

## 11. Async / Coroutine 注意事項

> ⚠️ Rule 1–3 均基於「同 frame」假設，以下情況不適用。

```
不適用同 frame 規則的情況：
  ❌ coroutine resume（yield return null 後恢復）
  ❌ async/await callback
  ❌ tween completion（DOTween / LeanTween）
  ❌ Invoke / InvokeRepeating

這些情況下，同一 frame 偵測到多個 source 可能是合法的 callback 順序，
而非 race condition，請結合 frame index 與 source 名稱人工判斷。
```

---

## 12. Performance Rules

| 規則 | 說明 |
|---|---|
| ❌ 禁止每 frame 寫 log | Anomaly Buffer 只在異常觸發時寫入 |
| ❌ 禁止大量 string interpolation（非異常路徑）| 正常路徑不執行 |
| ❌ 禁止正式環境即時寫檔 | 只在 F8 觸發時寫入 ZIP |
| ❌ 禁止全域監控 TMP_Text | 白名單原則，只監控已知高風險 UI 元件 |
| ❌ 禁止 hot path 使用 string key（v1.2）| FrequencyDetector 改用 enum + static array |
| ❌ 禁止在 Thread 中呼叫 Texture2D API（v1.2）| EncodeToPNG 必須在主執行緒 |
| ❌ 禁止 ZIP 打包在主執行緒同步執行（v1.2）| 改用 PackAsync（Task.Run）|
| ✅ Anomaly Buffer 容量 50 筆 | 只記錄異常，接近零成本 |
| ✅ FrequencyDetector 使用 enum + static array（v1.2）| 零 alloc，hot path 安全 |
| ✅ FrequencyDetector Threshold + Cooldown | 防 log storm；Release Build 用 `#if` 完整移除 |
| ✅ LocalZipPacker.PackAsync 背景執行緒（v1.2）| 避免 HDD / SD Card 卡頓 |
| ✅ Desync Fingerprint 每 30～60 tick 計算一次 | 純整數運算，成本接近零 |
| ✅ stateFingerprints ring buffer 上限 200 筆 | 防止長時間掛機造成無限累積 |
| ✅ CriticalSnapshot 加 60 frame cooldown（v1.2）| 防 anomaly storm 造成 GC spike |
| ✅ UITextAnomalyDetector 語系用 cache flag（v1.2）| 不做 runtime 字元掃描 |
| ✅ GameplayBuffer 改用 struct 儲存 timestamp（v1.2）| 消除 string parse bug |
| ✅ AnomalyStormGuard gate（v1.3）| 整個診斷 layer 自保護：storm 時 1/5 sampling，防診斷系統造成二次 lag |
| ✅ AnomalyCausalChain 分析（v1.3）| F8 時一次性計算，不影響 runtime；輸出因果鏈到 anomaly.json |
| ❌ 禁止 worker thread 碰 UnityEngine.Object（v1.4）| 截圖 encode 在主執行緒完成後傳 byte[]，worker 只碰 pure DTO |
| ❌ 禁止 Task.Run 缺少 CancellationToken（v1.4）| 所有背景 task 必須可被 OnDestroy 中止 |
| ❌ 禁止 BuildPackageData 裡 FindWithTag（v1.4）| 組件參考在 Awake 快取，F8 觸發時直接讀快取 |
| ✅ Immutable Snapshot 模式（v1.4）| 主執行緒建立純 DTO 快照後傳入 worker，消除 race condition 與 state tearing 風險 |

---

## 13. Anomaly 事件類型規格（Canonical）

> ⚠️ **本表為唯一來源。** 所有程式碼（EventRingBuffer.LogAnomaly 呼叫）與資料格式（anomaly.json 的 `system` / `eventType` 欄位）均以本表為準。

### 事件類型表

| System | EventType | 觸發條件（Rule）| 嚴重度 |
|---|---|---|---|
| `LIFECYCLE` | `ListenerImbalance` | Rule 2：Add/Remove 不平衡 > 3 | **WARN** |
| `FREQUENCY` | `HighFrequencyAnomaly` | Rule 1：同 frame 超頻 | **WARN** |
| `SHOP` | `ScrollBegin` | Scroll 開始（僅邊界）| INFO |
| `SHOP` | `ScrollEnd` | Scroll 結束（僅邊界）| INFO |
| `SHOP_STATE` | `InvalidTransition` | Rule 4：非法 state 轉換 | **WARN** |
| `UI_TEXT` | `DuplicateWrite` | Rule 3：同 frame 多來源覆寫 | **WARN** |
| `UI_TEXT` | `LocalizationMismatch` | 語系與文字來源不一致 | **WARN** |
| `PAGE` | `PageResetMissing` | 切分類未歸零 | **WARN** |
| `SNAPSHOT` | `CriticalDump` | 關鍵流程失敗時 | **ERROR** |
| `SYNC` | `DesyncDetected` | Fingerprint 本地連續兩筆 hash 異常跳變 | **WARN** |
| `GUARD` | `StormDetected` | 60 frame 內 LogAnomaly 超過 10 次，進入 sampling 模式（v1.3）| **WARN** |

### Frame Index 的必要性

```
沒有 frame（難以判斷是競爭還是順序）：
  12:01:33 HoverController     修改 text=檢視
  12:01:33 LocalizationManager 修改 text=VIEW

有 frame（立即確認是同幀競爭）：
  12:01:33 frame=18601 HoverController     修改 text=檢視
  12:01:33 frame=18601 LocalizationManager 修改 text=VIEW
           ↑ 同一 frame → 確認 Rule 3 DuplicateWrite
```

---

## 14. 診斷流程（實戰手冊）

### 復現 Bug 的標準流程

```
1. 開啟遊戲，確認 EventRingBuffer 已啟動
2. 進入商店 → 造型 → 頭髮
3. 執行以下操作各 3 次：
   a. 點擊造型格
   b. 點擊空白區域
   c. 滾動滑鼠滾輪
4. 切換到 眉毛 分頁
5. 重複步驟 3
6. 切換到「背包」，再點回「商店」
7. 確認商店是否異常
8. 若異常：立即按 F8 上傳回報
```

### 從 anomaly.json 判讀 Root Cause

```
【第一步：看 causalChains（v1.3 新增）】
  causalChains 為空         → 孤立事件，直接看 events 序列
  rootCause = ListenerImbalance, spanFrames < 60
                            → 瞬間崩壞型，OnDisable 未正確呼叫
  rootCause = ListenerImbalance, spanFrames > 300
                            → 累積型 leak，長時間 prefab pool 未釋放
  sequence 含 CriticalDump  → 已有完整 finalSnapshot，直接讀取

【第二步：看個別事件】
看到 ListenerImbalance（LIFECYCLE）
→ ✅ 確認 Event Listener Leak
→ 修法：確保 OnDisable 正確呼叫 RemoveListener

看到 InvalidTransition（SHOP_STATE）
→ ✅ 確認 State Machine 有非法轉換
→ 修法：在原有 SetState 旁加入 ShopStateTracer.Trace()

看到 DuplicateWrite（UI_TEXT）在 ShopOpen 附近
→ ✅ 確認多個系統同時修改 UI
→ 修法：在白名單目標的修改旁加入 UITextAnomalyDetector.Observe()

看到 PageResetMissing（PAGE）
→ ✅ 確認分頁切換時 page 未歸零
→ 修法：在 OnTabChanged 中加入 PageRefreshTracer.Trace(0, newCategory, ...)

看到 CriticalDump（SNAPSHOT）
→ ✅ 讀取 anomaly.json 的 finalSnapshot 欄位
→ 可立即定位異常發生瞬間的完整狀態

看到 StormDetected（GUARD）
→ ⚠️ 診斷系統本身觸發 storm 保護
→ 代表當時有大量異常連發，causalChains 可能不完整（部分事件被 sampling 丟棄）
→ 修法優先度提高：先解決 rootCause，再重新測試確認完整序列
```

### 工程師注意事項

```
❌ 錯誤理解：用這套工具「接管」state 修改
✅ 正確理解：在現有邏輯「旁邊」觀測，找到問題再修原始代碼

原則：Tracer / Detector 只觀測，不替換現有邏輯。
```

---

## 15. 安全性

| 面向 | 說明 |
|---|---|
| 無 Secret | Client 不含 API Key / Webhook Token / DB Password |
| 無 Upload API | 根本不存在 upload server，無需 anti-spam / rate limit |
| 無遠端攻擊面 | 不存在 webhook endpoint 或 cloud function，無法被刷 storage |

> 選配的 Discord Webhook 模式因在 client 持有 webhook URL，有被逆向工程取得的風險，請評估後決定是否採用。

---

## 16. ZIP 生成失敗 Fallback

> ⚠️ **嚴禁靜默失敗。** 若 ZIP 生成失敗，玩家必須知道：發生了什麼、哪些檔案仍存在、接下來該怎麼做。

### 可能失敗原因

| 類型 | 範例 |
|---|---|
| Disk Full | 磁碟空間不足 |
| Permission Denied | 無寫入權限 |
| Compression Failure | ZipFile exception |
| File Lock | log 被其他進程占用 |

### Fallback 優先順序

**執行序列**（依序嘗試，不得跳過）：

```
1. ZIP 生成失敗
   → [自動 Retry 一次]（間隔 500ms）

2. Retry 仍失敗
   → [Fallback A] 保留散檔：BugReports/ 目錄下各別存入
     - screenshot.png（截圖）
     - anomaly.json（anomaly buffer）
     - match.json（fingerprint）
     - report.json（玩家輸入 + gameplay）
     - player.log（最後 500 行）

3. 開啟 BugReports 資料夾（Windows Explorer）
   → 讓玩家可手動壓縮上傳

4. [Fallback B] 若 JSON 生成也失敗
   → 仍保證 screenshot.png 存入磁碟（截圖是最低優先但最高價值的診斷資料）

5. [Fallback C] 遊戲內顯示提示（必須在 UI 上可見，不只是 console log）
```

**Fallback A**：直接開啟 BugReports 資料夾

即使 ZIP 失敗，仍保留散檔（screenshot / json / log），玩家可手動壓縮。

**Fallback B**：至少保留 screenshot

若 JSON / ZIP 失敗，仍保證 `screenshot.png` 成功保存，避免完全無診斷資料。

**Fallback C**：遊戲內提示（必須顯示於 UI，不得只寫 Debug.Log）

```
無法自動打包 Bug Report。
請手動上傳 BugReports 資料夾中的檔案。
Match ID：room_18273
```

避免玩家誤以為已成功送出。

---

## 17. Agent + Skill 後處理分層設計

> **適用時機**：此層不在遊戲內執行，不消耗遊戲算力。
> 開發者收到玩家提交的 ZIP 後，在開發環境透過 Agent + Skill 工具執行。

### 分層架構

```
[遊戲層] Unity Client
    ↓ F8 → 生成 ZIP
[封裝層] LocalZipPacker
    ↓ 玩家手動上傳（Discord / Google Form）
[後處理層] Agent + Skill（開發環境，非遊戲內）
    ↓
    ├── UnzipSkill          → 解壓縮，列舉內容
    ├── AnomalyParseSkill   → 解析 anomaly.json，提取 system / eventType 序列
    ├── MatchCorrelateSkill → 以 matchId 關聯雙方 ZIP（條件性，需雙方提交）★
    ├── BugClassifySkill    → 依 eventType 自動分類
    └── ReportGenerateSkill → 輸出結構化摘要，標記優先度
```

> ★ **MatchCorrelateSkill 屬於 Developer-only enhanced path，不是主系統路徑。**
> 無後端模式下，對手可能不知道要按 F8、已離線、或不會配合上傳。
> 主力診斷路徑應是 Desync Fingerprint（單邊即可提供資訊），雙方對比為條件性加分功能。

### ZIP 格式配合後處理的設計原則

| 原則 | 說明 |
|---|---|
| JSON 使用標準格式 | 無自定義 encoding，Agent 直接 JSON.parse |
| matchId 出現在多個檔案 | `report.json` 與 `match.json` 均含 matchId，供 Agent 索引 |
| anomaly eventType 使用固定 enum | 值固定（見 Section 13），BugClassifySkill 可直接 match |
| metadata.txt 提供快速摘要 | Agent 可先讀此檔決定是否需要深入解析 |
| 檔案命名含 timestamp | `BugReport_2026-05-15_04-31-22.zip`，便於批次排序 |
| stateFingerprints 標準化格式 | tick（整數）+ hash（8 位十六進位），MatchCorrelateSkill 可直接對比 |

### 後處理工作流程

```
開發者收到 ZIP 批次
    ↓
Agent 依 eventType 自動分類
    ↓
相同 matchId 的多個 ZIP → MatchCorrelateSkill（條件性，需雙方提交）
    ↓
stateFingerprints 對比 → 找出 divergence tick（條件性）
    ↓
依 ListenerImbalance / InvalidTransition / DuplicateWrite / DesyncDetected 分 bucket
    ↓
輸出結構化 bug report（含優先度評分）
    ↓
人工複審高優先度 bug
```

---

## 18. 腳本清單與優先順序

### Tier 1：必做

| 腳本 | 功能 |
|---|---|
| `EventRingBuffer.cs` | Anomaly-focused，容量 50 筆；含 LogAnomalyDirect（v1.3）|
| `AnomalyStormGuard.cs` | 自保護 gate：storm 時 1/5 sampling（v1.3）|
| `ListenerBalanceDetector.cs` | Rule 2：Listener leak |
| `ShopStateTracer.cs` | Rule 4：非法 state 轉換 |
| `CriticalSnapshot.cs` | 異常瞬間完整狀態快照（含 60f cooldown）|
| `StateHashCollector.cs` | Desync Fingerprint：每 30～60 tick hash，ring buffer 200 筆 |
| `AnomalyCausalChain.cs` | F8 時分析因果鏈，輸出 causalChains（v1.3）|
| `BugReportManager.cs` | F8 觸發、截圖、資料彙整、ZIP 啟動 |
| `BugPackageData.cs` | 所有資料類別定義 |
| `MatchInfoProvider.cs` | matchId / playerId / opponentId |
| `LocalZipPacker.cs` | 打包 ZIP（PackAsync，背景執行緒）|
| `BrowserLauncher.cs` | 開瀏覽器到 Discord / Google Form |

### Tier 2：強烈建議

| 腳本 | 功能 |
|---|---|
| `FrequencyDetector.cs` | Rule 1：高頻異常（Release 需 `#if` 移除）|
| `UITextAnomalyDetector.cs` | Rule 3：重複寫入（白名單）|
| `PageRefreshTracer.cs` | 分頁未歸零 |
| `NetworkStatsCollector.cs` | ping avg/peak、packet loss |
| `TickSyncManager.cs` | localTick / remoteTick / offset |
| `ReportPanel.cs` | UI：截圖預覽、標題／描述輸入、送出 |

### Tier 2（選配，Webhook 模式）

| 腳本 | 觸發條件 |
|---|---|
| `DiscordWebhookUploader.cs` | 選擇 Discord Webhook 模式時才加入 |

### Tier 3（按需）

| 腳本 | 觸發條件 |
|---|---|
| `SliderLoopDetector.cs` | 出現 Slider 異常循環 |
| `MatchSettlementTracer.cs` | Match state 出現非法轉換 |

---

## 19. 白名單監控優先級

| 優先 | 區域 | 監控工具 | 監控重點 |
|---|---|---|---|
| 🔴 最高 | 頭髮／眉毛頁面 | `ListenerBalanceDetector` | Listener lifecycle |
| 🔴 最高 | 商店 UI | `ShopStateTracer` | 非法 state 轉換 |
| 🟠 高 | 商店開啟流程 | `CriticalSnapshot` | ShopOpen 失敗快照 |
| 🟠 高 | Localization 切換 | `UITextAnomalyDetector` | 重複寫入白名單目標 |
| 🟡 中 | Match Settlement | `FrequencyDetector` | 異常高頻結算事件 |

---

## 20. 適用階段

### 非常適合

| 階段 | 適合度 |
|---|---|
| 個人開發 | ✅ 高 |
| 小型封測（< 50 人）| ✅ 高 |
| QA 測試 | ✅ 高 |
| Steam Demo | ✅ 高 |
| Early Access 初期 | 🔶 中高 |

### 不適合

| 階段 | 原因 |
|---|---|
| 大型 Live Service | 回報量太大，需後端聚類 |
| 高 DAU PvP | 缺乏集中分析能力 |
| 即時營運分析 | 無 analytics pipeline |

---

## 21. 排除功能

以下功能對 2 人對戰收益極低，現階段不做：

- **inventory snapshot**：2 人對戰通常無複雜 RPG 背包系統
- **procedural random seed**：無程序生成地圖則不需要
- **save file 上傳**：無持久化存檔則不需要
- **report status system**：早期開發，先做遊戲本體
- **screenshot base64 嵌入 JSON**：直接放進 ZIP，無此需要
- **MMO 等級 telemetry**：2 人對戰架構完全不需要
- **全域 TMP_Text 監控**：改用白名單原則
- **全域 event trace（所有 OnClick / OnPointerEnter）**：改用白名單原則

> ⚠️ 常見錯誤：遊戲還沒穩，先做一套企業級 bug platform，結果 debug 系統比遊戲本體還大。

---

*文件版本 v1.4 | 地標爭霸 Bug Report & Runtime Diagnosis System*
*最後更新：2026-05-15*
