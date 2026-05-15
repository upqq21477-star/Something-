# 地標爭霸 Debug / Runtime Diagnosis System 分析與使用說明

## 系統定位

這套系統是為 Steam 遊戲《地標爭霸》設計的：

- Runtime 異常診斷系統（Runtime Diagnosis）
- Bug 回報封裝系統（Bug Report Package）
- Root Cause 分析流程（Diagnosis Skill）

其核心目標：

> 專門抓「不會 crash，但會讓遊戲狀態悄悄壞掉」的問題。

例如：

- 商店 UI 卡死
- Listener 重複註冊
- UI 文字互相覆寫
- State Machine 非法跳轉
- 分頁未重置
- Desync
- Hover/UI race condition

---

# 核心設計理念

## 1. 完全無 API / Backend

系統主架構：

```text
玩家遊戲端
    ↓
Runtime Detector
    ↓
本地生成 ZIP
    ↓
玩家手動上傳
    ↓
Agent + Skill 分析
```

優點：

- 不需要 API Key
- 不需要後端維運
- 不需要 DB
- 不需要 Webhook
- 不需要 infra 成本

非常適合：

- Indie Game
- Steam 遊戲
- 小團隊
- 單機 / P2P 遊戲

---

## 2. 低算力優先（避免 Lag）

Spec 非常強調：

> Debug 系統本身不能造成新的效能問題。

因此大量使用：

### 白名單監控

只監控特定 UI：

```text
shop_view_button
shop_watch_label
shop_category_label
```

避免全域掃描。

---

### Ring Buffer

```text
EventRingBuffer
```

只保留固定數量異常：

```text
最多 50 筆
```

避免：

- GC 爆增
- 記憶體成長
- Log 洗版

---

### 僅記錄異常

不是每 frame 記錄。

而是：

```text
只有違規才寫入 anomaly
```

例如：

```text
正常 Scroll → 不記錄
異常高頻 Scroll → 才記錄
```

---

### Sampling 保護（Storm Guard）

當異常暴增時：

```text
60 frame 內 > 10 anomaly
```

系統進入：

```text
1/5 sampling mode
```

避免 debug 系統自己造成 lag。

---

# 系統架構分析

## 分層架構

目前設計接近：

```text
Runtime Layer
    ↓
Detector Layer
    ↓
Packaging Layer
    ↓
Agent Layer
    ↓
Skill Layer
```

與：

```text
Agent + Skill 下統一完成
```

完全相容。

---

# Runtime Diagnosis Layer

## EventRingBuffer

用途：

```text
保存最近異常事件
```

特性：

- 固定容量
- O(1)
- 不成長
- 避免 GC

---

## ListenerBalanceDetector

檢查：

```text
AddListener
RemoveListener
是否失衡
```

適合抓：

- OnEnable 重複 AddListener
- OnDisable 未 RemoveListener
- Object Pool 生命周期錯誤

---

## FrequencyDetector

檢查：

```text
同 frame
同事件
是否超過閾值
```

適合抓：

- Infinite refresh
- Recursive event
- Hover loop
- UI feedback loop

---

## ShopStateTracer

追蹤：

```text
State Machine transition
```

例如：

```text
Closed → Open
Open → HairTab
```

非法：

```text
Closed → HairTab
```

---

## UITextAnomalyDetector

檢查：

```text
同 frame
不同系統
同時寫同一個 TMP_Text
```

適合抓：

- Hover 殘留
- Localization race
- UI 互蓋

---

## StateHashCollector

建立：

```text
Desync Fingerprint
```

注意：

```text
不能直接 hash float
```

需：

```text
先量化成 int
再 hash
```

避免跨平台差異。

---

# Bug Package ZIP

## ZIP 結構

```text
BugPackage.zip
```

內含：

```text
metadata.txt
report.json
anomaly.json
match.json
player.log
screenshot.png
```

---

## anomaly.json

核心資料：

```text
events[]
causalChains[]
finalSnapshot
```

為 AI diagnosis 的主要來源。

---

# Skill 系統分析

SKILL.md 本身已經很接近：

```text
AI diagnosis workflow spec
```

## Pipeline

### Step 1：PARSE

解析：

- metadata
- anomaly
- report
- match
- player.log

---

### Step 2：CLASSIFY

分類：

```text
STATE
UI
SYNC
LIFECYCLE
```

---

### Step 3：CHAIN

重建因果鏈：

```text
ListenerImbalance
→ InvalidTransition
→ CriticalDump
```

---

### Step 4：ROOT CAUSE

計算：

```text
最可能根因
```

---

### Step 5：REPORT

輸出：

- 修法方向
- 涉及腳本
- 驗證方式
- 優先級

---

# 為什麼適合 Agent + Skill

## Runtime 很輕

遊戲端只做：

```text
Detect
Trace
Snapshot
Package
```

不做 Heavy Analysis。

---

## Agent 負責推理

Agent 可做：

```text
Cross-event correlation
Root cause ranking
Repair suggestion
```

---

## Skill 負責規則化

例如：

```text
InvalidTransition
→ 對應哪種修法
```

因此：

```text
Runtime 很輕
AI 很重
```

工作切割合理。

---

# 玩家使用流程

## 玩家遇到問題

例如：

- 商店卡死
- UI 錯亂
- 同步異常

按下：

```text
F8
```

系統會：

```text
生成 ZIP
```

玩家上傳給開發者即可。

---

# 開發者分析流程

## Step 1

讀取：

```text
metadata.txt
```

確認：

- 玩家
- 版本
- matchId

---

## Step 2

讀：

```text
anomaly.json
```

檢查：

```text
CriticalDump
ListenerImbalance
InvalidTransition
DuplicateWrite
```

---

## Step 3

重建 causal chain：

```text
Listener leak
→ UI 重複 refresh
→ State 被覆寫
→ 商店卡死
```

---

# 建議補強

## Detector Priority Level

建議：

```text
Critical
Normal
Disabled
```

方便 Release Build 控制。

---

## Conditional Compile

```csharp
#if DEBUG_DIAGNOSIS
#endif
```

避免正式版 overhead。

---

## 統一事件格式

```json
{
  "system": "SHOP_STATE",
  "eventType": "InvalidTransition",
  "frame": 123,
  "detail": {}
}
```

方便 Agent 分析。

---

# 總結

這套系統的核心特點：

```text
低成本
低算力
無 API
無 Backend
高診斷能力
```

它不是傳統 Crash Reporting。

而是：

```text
Runtime Anomaly Diagnosis System
```

真正價值：

```text
能抓到「沒有 crash 的壞狀態」
```

並且非常適合：

```text
Runtime Layer
→ Packaging Layer
→ Agent Layer
→ Skill Layer
```

後續直接接入：

- AI diagnosis
- Root cause ranking
- 修法推薦
- anomaly clustering
