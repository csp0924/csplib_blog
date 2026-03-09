# IEC 60870-5-104 入門：電力系統的通訊骨幹

> **從零打造工業控制協定框架：給軟體工程師的 CSP Library 實戰系列**
> Part 2 -- 協定轉接層 | Article 07

---

## 前言

在前兩篇文章中，我們深入探討了 Modbus 協定 -- 一個簡單、普及、幾乎所有工業設備都支援的通訊協定。但當場景從「廠內設備通訊」擴展到「電力系統調度」時，Modbus 就開始力不從心了。

想像你正在開發一套區域電網的能源管理系統（EMS），需要從變電所取得即時的電力資料、向儲能系統下達調度指令、並在故障發生時在毫秒級別收到告警。這時候，你需要的是一個專為電力系統設計的協定 -- **IEC 60870-5-104**（以下簡稱 IEC 104）。

本篇文章將帶你理解 IEC 104 的核心概念、它與 Modbus 的關鍵差異，以及 csp_lib 的協定無關抽象如何讓你的系統同時支援多種協定。

---

## 1. IEC 104 是什麼？

IEC 60870-5-104 是國際電工委員會（IEC）制定的電力系統遠端監控通訊標準。它是 IEC 60870-5-101（串列通訊版本）的 TCP/IP 延伸，主要用於：

- **變電所自動化**：從遠端監控和控制變電所設備
- **SCADA 系統**：電力調度中心與各場站之間的資料交換
- **分散式能源管理**：風力發電、太陽能、儲能系統的調度
- **電網互聯**：不同電力公司之間的資料交換

在台灣，台電的 SCADA 系統大量使用 IEC 104 協定。如果你的儲能系統需要接入台電的調度，幾乎一定會碰到 IEC 104。

### IEC 104 vs Modbus：定位差異

| 特性 | Modbus | IEC 104 |
|------|--------|---------|
| 設計年代 | 1979 | 1995（101）/ 2000（104） |
| 設計目的 | 工廠設備通訊 | 電力系統遠端監控 |
| 適用範圍 | 廠內 LAN | 廣域網路（WAN） |
| 資料模型 | 暫存器 (16-bit) | 資訊物件（typed） |
| 通訊模式 | 主從輪詢 | 事件驅動 + 輪詢 |
| 可靠性機制 | 無 | 序號確認、連線監督 |
| 時間戳 | 無（需自行處理） | 協定內建 |
| 標準化程度 | 簡單但自由度大 | 嚴格定義 |

簡單來說：Modbus 是工業界的 HTTP/1.0，IEC 104 是電力界的 WebSocket。

---

## 2. 協定結構：APCI + ASDU

IEC 104 的封包結構分為兩層：

```
+-------------------------------------------+
| APCI (Application Protocol Control Info)   |
|  - 起始位元組 (0x68)                       |
|  - 長度                                    |
|  - 控制欄位 (4 bytes)                      |
+-------------------------------------------+
| ASDU (Application Service Data Unit)       |
|  - 類型識別 (Type ID)                      |
|  - 資訊物件數量                            |
|  - 傳輸原因 (Cause of Transmission)        |
|  - 公共位址 (Common Address)               |
|  - 資訊物件 (Information Objects)          |
+-------------------------------------------+
```

### APCI：傳輸控制層

APCI 負責連線管理和可靠傳輸，定義了三種訊框格式：

**I-format（Information）**：攜帶實際資料的訊框

```
| 起始 0x68 | 長度 | 發送序號(SSN) | 接收序號(RSN) | ASDU... |
```

每個 I-format 訊框都有**發送序號（SSN）**和**接收序號（RSN）**。這就像 TCP 的 sequence number -- 確保資料不會遺失或亂序。

**S-format（Supervisory）**：確認訊框

```
| 起始 0x68 | 長度 | 0x01 | 0x00 | 接收序號(RSN) |
```

當一方收到多個 I-format 訊框後，可以用一個 S-format 一次確認。這比逐一確認更有效率。

**U-format（Unnumbered）**：連線管理訊框

```
| 起始 0x68 | 長度 | 控制碼 | 0x00 | 0x00 | 0x00 |
```

U-format 用於連線管理，包含三對命令：

- **STARTDT（Start Data Transfer）**：啟動資料傳輸
- **STOPDT（Stop Data Transfer）**：停止資料傳輸
- **TESTFR（Test Frame）**：連線測試（心跳）

### ASDU：應用資料層

ASDU 是實際承載業務資料的部分。它的核心概念是**資訊物件（Information Object）**-- 每個物件代表一個量測值或控制命令。

#### 常用的 Type ID

IEC 104 定義了上百種 Type ID，但實務中最常用的只有十幾種：

**監視方向（Monitor Direction）-- 從設備到控制中心：**

| Type ID | 名稱 | 說明 | 對應 Modbus |
|---------|------|------|------------|
| M_SP_NA_1 (1) | 單點資訊 | 0 或 1（開/關） | Discrete Input |
| M_SP_TB_1 (30) | 單點資訊（帶時標） | 同上 + 時間戳 | -- |
| M_DP_NA_1 (3) | 雙點資訊 | 00/01/10/11 四種狀態 | -- |
| M_ME_NA_1 (9) | 量測值（歸一化） | -1.0 ~ +1.0 | Input Register |
| M_ME_NB_1 (11) | 量測值（標度化） | -32768 ~ 32767 | Holding Register |
| M_ME_NC_1 (13) | 量測值（短浮點） | IEEE 754 float | Float32 |
| M_ME_TF_1 (36) | 量測值（短浮點+時標） | 同上 + 時間戳 | -- |
| M_IT_NA_1 (15) | 累計量 | 電度量、累積值 | UInt32 |

**控制方向（Control Direction）-- 從控制中心到設備：**

| Type ID | 名稱 | 說明 |
|---------|------|------|
| C_SC_NA_1 (45) | 單命令 | 開/關 |
| C_DC_NA_1 (46) | 雙命令 | 開/關/無效/不確定 |
| C_SE_NA_1 (48) | 設定值命令（歸一化） | -1.0 ~ +1.0 |
| C_SE_NB_1 (49) | 設定值命令（標度化） | -32768 ~ 32767 |
| C_SE_NC_1 (50) | 設定值命令（短浮點） | IEEE 754 float |
| C_IC_NA_1 (100) | 總召喚 | 要求設備回報所有資料 |

#### Cause of Transmission（傳輸原因）

每個 ASDU 都帶有「傳輸原因」，說明為什麼要傳這筆資料：

| CoT | 名稱 | 說明 |
|-----|------|------|
| 1 | 週期 | 定期上報 |
| 2 | 背景掃描 | 背景資料掃描 |
| 3 | 突發 | 資料變化時主動上報 |
| 5 | 被請求 | 回應總召喚或查詢 |
| 6 | 啟動 | 命令啟動 |
| 7 | 啟動確認 | 命令啟動確認 |
| 10 | 結束啟動 | 啟動完成 |
| 20 | 被詢問 | 回應總召喚 |

---

## 3. 與 Modbus 的關鍵差異

### 差異一：事件驅動 vs 輪詢

**Modbus** 是純粹的主從輪詢模式。主站定期向從站發送讀取請求，從站回應資料。如果你想知道一個值有沒有變化，只能靠不斷輪詢。

```
Modbus:
  主站: "位址 100 的值是多少？"
  從站: "42"
  (等 1 秒)
  主站: "位址 100 的值是多少？"
  從站: "42"
  (等 1 秒)
  主站: "位址 100 的值是多少？"
  從站: "57"  <- 終於變了！但你等了 1 秒才發現
```

**IEC 104** 支援事件驅動通訊。當設備端的值發生變化時，它會**主動**把新值推送給控制端（Cause of Transmission = 突發/Spontaneous）。

```
IEC 104:
  [連線建立, STARTDT]
  控制端: C_IC_NA_1 (總召喚) <- 第一次取得全部資料
  設備端: M_ME_NC_1 (active_power=42.0, CoT=被詢問)
  設備端: M_ME_NC_1 (voltage=220.5, CoT=被詢問)
  ...
  (過了一段時間，值改變了)
  設備端: M_ME_NC_1 (active_power=57.0, CoT=突發) <- 主動推送！
```

這個差異對系統設計的影響非常大：

- **延遲**：IEC 104 可以做到接近即時的事件通知，Modbus 的延遲取決於輪詢間隔
- **頻寬**：IEC 104 只在值變化時傳輸，Modbus 不管值有沒有變都要傳
- **故障偵測**：電力系統的保護需要毫秒級的故障偵測，Modbus 做不到

### 差異二：序號確認機制

Modbus 的 TCP 版本雖然跑在 TCP 上（TCP 本身有重傳機制），但 Modbus 協定層面沒有額外的可靠性保證。如果應用層發生錯誤（例如回應的功能碼不對），Modbus 只能靠上層邏輯處理。

IEC 104 在應用層就有完整的序號確認機制：

```
控制端 -> 設備端: I-frame SSN=0, RSN=0 (第 0 個封包)
設備端 -> 控制端: I-frame SSN=0, RSN=1 (確認收到第 0 個)
控制端 -> 設備端: I-frame SSN=1, RSN=1 (第 1 個封包)
設備端 -> 控制端: S-frame RSN=2        (確認收到第 0, 1 個)
```

如果序號不連續，代表中間有封包遺失，雙方可以偵測到並重新同步。

### 差異三：General Interrogation（總召喚）

IEC 104 有一個很重要的機制叫**總召喚（General Interrogation, GI）**。當控制端第一次連線（或重新連線）時，可以發送一個 C_IC_NA_1 命令，要求設備端回報所有資料點的當前值。

```python
# 概念示意（非 csp_lib 程式碼）
# 控制端發送總召喚
send_general_interrogation(common_address=1)

# 設備端回應所有量測值
# M_SP_NA_1: breaker_status = CLOSED
# M_ME_NC_1: active_power = 42.0
# M_ME_NC_1: voltage = 220.5
# ...（可能有數百個資料點）
# 最後回應 GI 結束確認
```

這讓控制端可以在連線建立時快速獲得完整的系統狀態，之後只需要靠事件驅動更新變化的值。

Modbus 沒有類似的機制 -- 你必須自己逐一讀取每個暫存器來建立初始狀態。

### 差異四：時間戳

IEC 104 的許多 Type ID 都內建時間戳（如 M_ME_TF_1）。時間戳精確到毫秒，記錄的是**事件發生的時間**，而非資料到達控制端的時間。

這在電力系統中至關重要 -- 當故障發生時，你需要知道精確的故障時序（哪個保護先動作、哪個斷路器先跳脫），才能正確分析故障原因。

Modbus 的資料不帶時間戳。如果你需要時間資訊，只能在收到資料時自行記錄，但這記錄的是「收到時間」而非「發生時間」，在高延遲的網路中差異可能很大。

---

## 4. 資料模型對照

讓我們把 IEC 104 的資料模型和 Modbus 的暫存器模型做一個對照：

### 單點資訊 vs 線圈/離散輸入

```
IEC 104: M_SP_NA_1 (Type ID = 1)
  - 資訊物件位址 (IOA): 1001
  - 值: 0 或 1
  - 品質描述詞: {invalid, blocked, substituted, not_topical, overflow}

Modbus: Coil / Discrete Input
  - 位址: 0x0000
  - 值: 0 或 1
  - 品質資訊: 無
```

注意 IEC 104 的單點資訊帶有**品質描述詞（Quality Descriptor）**-- 它告訴你這個值是否可信。例如 `invalid` 表示感測器故障，`substituted` 表示值已被人工替代。Modbus 沒有這個概念，一個暫存器值就是一個數字，你無法從協定層面知道它是否可靠。

### 量測值 vs 暫存器

```
IEC 104: M_ME_NC_1 (Type ID = 13, 短浮點)
  - IOA: 2001
  - 值: IEEE 754 float (4 bytes)
  - 品質描述詞
  - 可選時間戳 (M_ME_TF_1)

Modbus: Holding Register (Float32)
  - 位址: 5000-5001
  - 值: 2 個 16-bit 暫存器組成的 IEEE 754 float
  - 品質資訊: 無
  - 時間戳: 無
```

### 命令物件 vs 寫入暫存器

```
IEC 104: C_SE_NC_1 (Type ID = 50, 設定值命令-短浮點)
  - IOA: 3001
  - 值: IEEE 754 float
  - 選擇/執行旗標
  - 命令確認機制（啟動 -> 啟動確認 -> 執行 -> 執行確認）

Modbus: Write Multiple Registers (FC 0x10)
  - 位址: 6000-6001
  - 值: 2 個 16-bit 暫存器
  - 確認: 簡單的回應封包
```

IEC 104 的命令有一套完整的**選擇-執行（Select Before Operate, SBO）**機制：

```
控制端: C_SE_NC_1 (SELECT, value=50.0)    -- "我想設定功率為 50 kW"
設備端: C_SE_NC_1 (ACTCON, value=50.0)     -- "收到，已準備好"
控制端: C_SE_NC_1 (EXECUTE, value=50.0)    -- "執行！"
設備端: C_SE_NC_1 (ACTTERM, value=50.0)    -- "已執行完畢"
```

這個四步驟的確認機制比 Modbus 的「寫入-回應」安全得多，特別是在控制高壓斷路器這種不容出錯的場景。

---

## 5. csp_lib 的協定無關抽象

雖然 csp_lib 目前的底層實作聚焦於 Modbus，但它的上層抽象（Layer 3 以上）是**協定無關**的。這意味著即使底層換成 IEC 104，上層的邏輯不需要改動。

### ReadPoint / WritePoint 的協定無關性

回憶 Article 05 中介紹的 `ReadPoint`：

```python
from csp_lib.equipment.core import ReadPoint, ScaleTransform, RoundTransform, pipeline
from csp_lib.modbus import Float32

active_power = ReadPoint(
    name="active_power",
    address=5000,
    data_type=Float32(),
    pipeline=pipeline(ScaleTransform(0.1), RoundTransform(1)),
)
```

這裡的 `address` 在 Modbus 世界是暫存器位址，但概念上它就是「一個資料點的定址」。在 IEC 104 中，等價的概念是 IOA（Information Object Address）。

`data_type` 在 Modbus 世界決定了如何編解碼暫存器，但在 IEC 104 中，Type ID 本身就定義了資料格式（M_ME_NC_1 就是 float）。

`pipeline` 則完全與協定無關 -- 不管資料是從 Modbus 暫存器還是 IEC 104 資訊物件取得的，後續的縮放和四捨五入邏輯都是一樣的。

### 概念映射

如果我們要在 csp_lib 的點位模型上映射 IEC 104，概念上會是這樣：

```python
# 假想的 IEC 104 點位定義（概念示意）
# 實際 csp_lib 尚未實作 IEC 104 底層，但抽象層已可容納

# Modbus 版本
modbus_power = ReadPoint(
    name="active_power",
    address=5000,            # Modbus 暫存器位址
    data_type=Float32(),     # Modbus 資料類型
    pipeline=pipeline(ScaleTransform(0.1), RoundTransform(1)),
)

# IEC 104 版本（概念）
# 如果實作了 IEC104DataType，結構會非常相似：
# iec104_power = ReadPoint(
#     name="active_power",
#     address=2001,           # IOA (Information Object Address)
#     data_type=MeNc1(),      # M_ME_NC_1 短浮點量測值
#     pipeline=pipeline(ScaleTransform(0.1), RoundTransform(1)),
# )
```

關鍵觀察：上層程式碼（控制策略、告警評估、資料上傳）只看到 `"active_power"` 這個名稱和最終的工程值。它不在乎這個值是從 Modbus 暫存器來的還是從 IEC 104 資訊物件來的。

### Transform Pipeline 的通用性

Transform Pipeline 是完全協定無關的。同樣的管線定義可以用在任何協定的資料後處理：

```python
from csp_lib.equipment.core import (
    ScaleTransform, RoundTransform, EnumMapTransform,
    ClampTransform, pipeline,
)

# 這些管線不管資料來源是什麼協定都能用
power_pipeline = pipeline(ScaleTransform(0.1), RoundTransform(1))
temp_pipeline = pipeline(ScaleTransform(0.1, offset=-40), RoundTransform(1))
status_pipeline = pipeline(EnumMapTransform({0: "STOP", 1: "RUN", 2: "FAULT"}))
soc_pipeline = pipeline(ScaleTransform(0.1), ClampTransform(0.0, 100.0))
```

### Command 的協定無關性

csp_lib 的 `Command` 物件同樣是協定無關的：

```python
from csp_lib.controller.core import Command

# Command 只描述「要做什麼」，不描述「怎麼做」
cmd = Command(p_target=500.0, q_target=100.0)
```

`Command` 表達的是業務語義（「把有功功率設定為 500 kW」），而非協定細節（「往暫存器 6000 寫入 Float32 值 500.0」或「發送 C_SE_NC_1 到 IOA 3001 值 500.0」）。

協定相關的翻譯工作由 `CommandRouter` 在更下層處理 -- 它知道如何把一個 `Command` 映射成具體的 Modbus 寫入操作或 IEC 104 命令。

---

## 6. 實務考量：何時選擇 IEC 104？

### 選擇 IEC 104 的場景

| 場景 | 原因 |
|------|------|
| 接入電力公司 SCADA | 這是台電等電力公司的標準 |
| 變電所自動化 | IEC 61850 + IEC 104 是業界標準 |
| 跨區域調度 | IEC 104 設計用於廣域網路 |
| 需要事件驅動通知 | 故障告警需要即時性 |
| 需要時間戳精確到毫秒 | 故障序列分析 |
| 法規要求 | 許多國家的電網併網規範要求 IEC 104 |

### 繼續使用 Modbus 的場景

| 場景 | 原因 |
|------|------|
| 廠內設備通訊 | Modbus 最簡單，幾乎所有設備都支援 |
| 感測器數據採集 | 不需要事件驅動，定期輪詢就夠了 |
| 快速原型開發 | Modbus 的學習曲線最低 |
| 預算有限 | Modbus 是免費的，某些 IEC 104 實作需要授權費 |

### 混合架構：Gateway 模式

在實務中，最常見的架構是混合使用兩種協定：

```
                         ┌─────────────┐
  台電 SCADA ──IEC 104──>│   Gateway   │──Modbus TCP──> PCS (變流器)
                         │  (你的系統)  │──Modbus TCP──> Meter (電表)
  調度中心  ──IEC 104──> │             │──Modbus RTU──> BMS (電池)
                         └─────────────┘
```

你的系統作為一個 **Gateway**（閘道器）：

- **南向（Southbound）**：用 Modbus 跟現場設備通訊（這是 csp_lib 目前的強項）
- **北向（Northbound）**：用 IEC 104 跟電力調度系統通訊

csp_lib 的架構天然支援這種模式：

1. Layer 2-3 負責南向的 Modbus 通訊
2. Layer 4-5 的控制策略和管理器是協定無關的
3. 北向的 IEC 104 可以作為 Layer 8（Additional）的模組實作，讀取設備的 `latest_values` 並透過 IEC 104 上報

```python
# 概念示意：Gateway 模式
# 南向：csp_lib 的 Modbus 設備讀取
async with device_manager:
    # device_manager 管理所有 Modbus 設備
    # 每台設備持續讀取最新值

    # 北向：IEC 104 Server（概念）
    # iec104_server = IEC104Server(port=2404)
    # for device in device_manager.devices:
    #     values = device.latest_values
    #     iec104_server.update_data_point(
    #         ioa=mapping[device.device_id]["active_power"],
    #         value=values["active_power"],
    #     )
    pass
```

---

## 7. IEC 104 的連線管理

IEC 104 的連線管理比 Modbus TCP 複雜得多。理解這些機制有助於你在做系統設計時做出正確的決策。

### 連線建立流程

```
1. TCP 三向交握（標準 TCP，port 2404）
2. 控制端發送 STARTDT act（U-format）
3. 設備端回應 STARTDT con（U-format）
4. 控制端發送 C_IC_NA_1（總召喚）
5. 設備端回應所有資料（I-format, CoT=被詢問）
6. 設備端發送 C_IC_NA_1（總召喚結束）
7. 進入正常運行：事件驅動上報 + 定期上報
```

### 連線監督：三個定時器

IEC 104 定義了三個定時器來監督連線健康狀態：

| 定時器 | 名稱 | 典型值 | 說明 |
|--------|------|--------|------|
| t1 | Send timeout | 15s | 發送後等待確認的超時 |
| t2 | Confirm timeout | 10s | 收到 I-frame 後多久要發確認 |
| t3 | Test timeout | 20s | 沒有資料交換時的心跳間隔 |

當 t3 超時（一段時間沒有任何資料交換），一方會發送 TESTFR act，另一方回應 TESTFR con。如果 TESTFR 也沒有回應（t1 超時），就判定連線已斷。

相比之下，Modbus TCP 完全沒有連線監督機制。如果對方「靜悄悄地」斷線了（例如網路線拔掉但沒有 TCP RST），你只能等到下次讀取超時才會發現。

### 備援連線

IEC 104 標準定義了**備援連線（Redundancy Group）**機制。控制端可以同時維持兩條到同一設備端的 TCP 連線，一條 active、一條 standby。當 active 連線斷掉時，standby 可以立即接手，實現無縫切換。

這種級別的可靠性是 Modbus 無法提供的，也是電力系統選擇 IEC 104 的重要原因之一。

---

## 8. 對軟體工程師的啟示

即使你目前的專案只用 Modbus，理解 IEC 104 仍然有價值。以下是幾個可以帶回日常開發的啟示：

### 啟示一：事件驅動優於輪詢

IEC 104 的事件驅動模式比 Modbus 的輪詢模式更高效。在你的應用層設計中，也應該優先考慮事件驅動。

csp_lib 在 Modbus 之上已經實現了事件驅動的抽象 -- `AsyncModbusDevice` 雖然底層用輪詢讀取，但它會自動比對值的變化並觸發 `value_change` 事件：

```python
# 雖然底層是 Modbus 輪詢，但上層看到的是事件驅動
async def on_value_change(payload):
    print(f"{payload.point_name}: {payload.old_value} -> {payload.new_value}")

device.on("value_change", on_value_change)
```

這個設計讓上層程式碼不需要關心底層是輪詢還是推送。

### 啟示二：品質資訊很重要

IEC 104 的每個資料點都帶有品質描述詞，這提醒我們：一個數值的「可信度」和數值本身一樣重要。

在設計工業系統時，你應該考慮：

- 這個值是真實量測的，還是感測器故障後的保持值？
- 這個值是自動讀取的，還是人工替代的？
- 這個值的時間戳是什麼時候？是最新的嗎？

csp_lib 的 `AsyncModbusDevice` 透過 `is_responsive` 屬性和 `disconnect_threshold` 機制提供了基本的品質判斷。

### 啟示三：協定無關的抽象層是值得投資的

csp_lib 的分層架構讓更換底層協定成為可能。這不是過度設計 -- 在工業系統的生命周期中（通常 15-20 年），底層協定更換是很正常的事。

如果你的業務邏輯直接依賴 Modbus 的暫存器位址和資料格式，等到需要支援 IEC 104 時，就必須大幅重寫。但如果你的業務邏輯只依賴 `"active_power"` 這個語義名稱，切換協定就只是更換底層驅動的事。

---

## 9. 重點回顧

1. **IEC 60870-5-104** 是電力系統 SCADA 的通訊標準，專為廣域監控設計
2. **協定結構**分為 APCI（傳輸控制）和 ASDU（應用資料），比 Modbus 複雜但也更可靠
3. **與 Modbus 的核心差異**：
   - 事件驅動 vs 輪詢
   - 序號確認機制提供可靠傳輸
   - 總召喚（GI）機制實現快速狀態同步
   - 內建時間戳支援故障序列分析
   - 品質描述詞標示資料可信度
4. **csp_lib 的抽象是協定無關的**：ReadPoint/WritePoint、Pipeline、Command 都不依賴特定協定
5. **Gateway 模式**是最常見的混合架構：南向 Modbus、北向 IEC 104
6. **設計啟示**：事件驅動優於輪詢、品質資訊不可忽略、協定無關抽象值得投資

---

## 下一篇預告

到這裡，我們已經完成了 Part 2（協定轉接層）的所有內容。接下來的 Part 3 將進入 csp_lib 的設備管理層，探討 AsyncModbusDevice 的完整生命週期管理、事件系統的設計、以及告警評估機制的實作。

> **Article 08：設備生命週期管理 -- AsyncModbusDevice 的 async with 魔法**
