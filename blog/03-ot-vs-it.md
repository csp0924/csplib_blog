# OT vs IT：兩個世界的碰撞與融合

> **從零打造工業控制協定框架：給軟體工程師的 CSP Library 實戰系列**
> Part 1 — 觀念轉換篇 | Article 03

---

## 前言：你以為的「系統」可能只有一半

身為後端工程師，你每天面對的是 API、資料庫、快取、容器化部署。這些都屬於 **IT（Information Technology，資訊技術）** 的範疇。但當你踏入工廠、電廠、變電站這些場域時，你會發現這裡運行著一整套截然不同的技術體系 —— **OT（Operational Technology，操作技術）**。

OT 的世界有自己的規則。這裡的「伺服器」是 PLC（可程式邏輯控制器），「資料庫」是暫存器（Register），「API」是 Modbus 或 IEC 61850，「部署」可能意味著爬上變電站的配電盤，把一條 RS-485 線接上去。

這篇文章將帶你認識 OT 與 IT 的本質差異，理解為什麼 Industry 4.0 需要兩者的融合，以及 csp_lib 如何作為連接這兩個世界的橋樑。

---

## OT 與 IT：基本定義

**IT（Information Technology）** 是我們熟悉的資訊技術：伺服器、網路、資料庫、雲端服務。核心目標是**資料的處理、儲存與傳輸**。

**OT（Operational Technology）** 是用來監控和控制實體設備與工業流程的硬體和軟體。核心目標是**讓實體世界的機器正確、安全地運作**。

一個簡單的類比：IT 管理的是「資訊流」，OT 管理的是「能量流」和「物質流」。

---

## 關鍵差異：兩張完全不同的優先序清單

下面這張表格是每位從 IT 跨入 OT 的工程師都必須銘記在心的：

| 面向 | OT（操作技術） | IT（資訊技術） |
|------|---------------|---------------|
| **最高優先** | **可用性（Availability）** — 系統不能停 | **機密性（Confidentiality）** — 資料不能洩 |
| **安全模型** | CIA → **AIC**（Availability > Integrity > Confidentiality） | CIA（Confidentiality > Integrity > Availability） |
| **設備生命週期** | **15-20 年** — 現場可能還在跑 Windows XP | **3-5 年** — 定期汰換升級 |
| **網路環境** | **氣隙隔離（Air-gapped）** — 理想上不接網路 | **連網** — 預設就在網路上 |
| **時間要求** | **確定性（Deterministic）** — 1ms 內必須回應 | **盡力而為（Best-effort）** — 可接受延遲 |
| **失敗後果** | **安全攸關（Safety-critical）** — 可能造成人員傷亡 | **資料攸關（Data-critical）** — 可能造成資料遺失 |
| **更新策略** | **極度保守** — 改了可能爆炸（字面意義） | **持續部署** — CI/CD、藍綠部署 |
| **通訊協定** | Modbus、IEC 61850、DNP3、OPC UA | HTTP、gRPC、AMQP、WebSocket |
| **作業系統** | 嵌入式 RTOS、VxWorks、裸機韌體 | Linux、Windows Server、容器 |
| **開發者** | 控制工程師、電機工程師 | 軟體工程師 |

讓我們深入理解幾個最關鍵的差異。

### 可用性至上：停機就是災難

在 IT 的世界，服務中斷 5 分鐘可能意味著幾筆訂單遺失。在 OT 的世界，系統中斷 5 分鐘可能意味著：

- 發電機組頻率失控，觸發連鎖跳脫
- 化工廠反應爐溫度失控
- 儲能系統電池過充引發熱失控

這就是為什麼 OT 系統的設計哲學是 **「永遠不能停」**。你在 IT 習以為常的 rolling restart、blue-green deployment，在 OT 是無法想像的奢侈。

### 20 年的設備生命週期

你寫的 Web 服務可能 3 年就重構了。但電廠裡的 PLC 可能從你出生前就在運行。這代表：

- 你可能需要跟 **1979 年發明的 Modbus 協定** 打交道
- 現場設備的韌體 **無法遠端更新**
- 通訊介面可能是 RS-485 序列埠，不是 Ethernet

### 確定性 vs 盡力而為

HTTP 請求慢了 200ms，使用者皺一下眉頭。但在 OT：

- 電力系統的保護繼電器必須在 **4ms 以內** 動作
- PLC 的掃描週期通常是 **1-10ms**
- Modbus RTU 的 frame gap 是 **3.5 個字元時間**（在 9600 baud 下約 3.6ms）

---

## 融合的時代：為什麼 OT 需要 IT？

儘管有這麼多差異，OT 和 IT 正在快速融合。這背後有幾個強大的推動力：

### Industry 4.0 與智慧製造

工廠不再只是「讓機器運轉」，而是要「讓機器聰明地運轉」。這需要從 OT 設備採集資料，用 IT 的方法（機器學習、數據分析）來優化製程。

### IIoT（工業物聯網）

感測器越來越便宜，邊緣運算越來越強大。每台設備都可以是一個數據源，但這些數據需要 IT 的基礎設施來匯聚、儲存、分析。

### 智慧電網與儲能系統（ESS）

這是 csp_lib 的核心應用場景。一套儲能系統需要：

- **OT 側**：透過 Modbus 控制 PCS（功率轉換系統）、讀取 BMS（電池管理系統）的 SOC（State of Charge）
- **IT 側**：將運行數據上傳 MongoDB、透過 Redis 做即時快取、用 REST API 提供監控介面

這兩側缺一不可。OT 確保系統安全運行，IT 確保數據被正確利用。

### 碳中和與再生能源

當太陽能、風能大量併入電網，電力調度變得極為複雜。你需要 OT 來控制設備，也需要 IT 來做即時調度演算法、氣象預測整合、電力交易平台對接。

---

## 軟體工程師的位置：OT-IT 的橋樑

如果你是一位有 2-3 年經驗的後端工程師，你在 OT-IT 融合中扮演的角色是什麼？

**你就是那座橋。**

控制工程師懂設備、懂協定、懂安全規範，但他們通常不擅長寫可維護的軟體架構。軟體工程師懂架構、懂測試、懂部署，但不懂工業協定和設備行為。

OT-IT 融合需要的是 **同時理解兩邊語言的人**。你不需要成為電機專家，但你需要理解：

- 為什麼 Modbus 暫存器是 16-bit 的
- 為什麼設備斷線不能只靠 retry 解決
- 為什麼告警必須有遲滯（hysteresis）機制
- 為什麼寫入命令必須有驗證

這正是 csp_lib 試圖降低的門檻。

---

## csp_lib 作為 OT-IT 橋樑

csp_lib 的 8 層架構設計，本質上就是一個 OT-IT 的轉譯層。讓我們從架構圖來看每一層的定位：

```
                          IT 世界
                    ┌─────────────────┐
         Layer 8   │   GUI (FastAPI)  │  ← REST API、Web 介面
                   ├─────────────────┤
         Layer 7   │  Storage (Mongo  │  ← 時序資料、歷史紀錄
                   │  / Redis)        │
                   ├─────────────────┤
         Layer 6   │  Integration     │  ← 資料聚合、指令路由
                   ├─────────────────┤
         Layer 5   │  Manager         │  ← 設備管理、告警持久化
         ──────────┼─────────────────┤──── OT-IT 邊界 ────
         Layer 4   │  Controller      │  ← 控制策略（PQ/QV/FP）
                   ├─────────────────┤
         Layer 3   │  Equipment       │  ← 設備抽象、點位、告警
                   ├─────────────────┤
         Layer 2   │  Modbus          │  ← 協定實作、資料編解碼
                   ├─────────────────┤
         Layer 1   │  Core            │  ← 日誌、生命週期、錯誤
                   └─────────────────┘
                          OT 世界
```

### Equipment 層（OT 面向）：與設備對話

Equipment 層是 csp_lib 最靠近 OT 的部分。它抽象了 Modbus 設備的所有細節：

```python
from csp_lib.modbus.types import Float32, UInt16
from csp_lib.modbus.enums import ByteOrder, RegisterOrder
from csp_lib.equipment.core.point import ReadPoint, WritePoint, PointMetadata
from csp_lib.equipment.core.transform import ScaleTransform, RoundTransform
from csp_lib.equipment.core.pipeline import ProcessingPipeline

# 定義一個 BMS 電池模組的讀取點位
soc_point = ReadPoint(
    name="soc",
    address=0x0000,
    data_type=UInt16(),
    byte_order=ByteOrder.BIG_ENDIAN,
    register_order=RegisterOrder.HIGH_FIRST,
    pipeline=ProcessingPipeline(steps=(
        ScaleTransform(magnitude=0.1),   # 暫存器值 * 0.1 = 實際 SOC%
        RoundTransform(decimals=1),       # 保留一位小數
    )),
    metadata=PointMetadata(unit="%", description="電池剩餘電量"),
)

# 定義 PCS 功率寫入點位
p_target_point = WritePoint(
    name="p_target",
    address=0x0010,
    data_type=Float32(),
    metadata=PointMetadata(unit="kW", description="目標有效功率"),
)
```

注意這裡的每個參數都來自 OT 世界的概念：
- `address=0x0000` — Modbus 暫存器位址
- `data_type=UInt16()` — 暫存器的資料型別
- `byte_order` — 位元組順序（OT 設備廠商各有各的實作）
- `ScaleTransform(magnitude=0.1)` — 原始暫存器值到工程值的轉換

### Manager/Integration 層（橋樑）：資料聚合與指令路由

Integration 層是真正的「翻譯官」。它將 OT 側的點位資料聚合成 IT 側可以理解的結構：

```python
from csp_lib.integration.schema import ContextMapping, AggregateFunc

# 將多台設備的 SOC 平均值映射到策略上下文
soc_mapping = ContextMapping(
    field="soc",
    trait="battery",           # 所有具備 "battery" 能力的設備
    point="soc",
    aggregate=AggregateFunc.AVERAGE,  # 多台電池取平均
)
```

### Storage 層（IT 面向）：持久化與快取

Storage 層是純粹的 IT 技術 — MongoDB 做時序資料儲存，Redis 做即時狀態快取。OT 工程師不需要關心這一層；IT 工程師則對這一層駕輕就熟。

### GUI 層（IT 面向）：REST API

FastAPI 驅動的 REST API，提供標準的 HTTP 介面。這是 IT 工程師最熟悉的領域，也是監控系統、SCADA 前端、行動 App 對接的介面。

---

## OT 環境的安全考量

### 為什麼不能「把所有東西都加防火牆」

IT 工程師面對安全問題的第一反應通常是：加密、防火牆、VPN。但在 OT 環境中，這些手段有致命的侷限：

1. **延遲不可接受**：加密/解密會增加通訊延遲。對於毫秒級要求的控制迴路，這可能導致系統不穩定。
2. **設備不支援**：20 年前的 PLC 沒有 TLS 能力。你不能要求它升級韌體。
3. **可用性衝突**：防火牆規則錯誤可能阻擋關鍵控制命令，後果可能是實體災害。

### Purdue 模型：OT 安全的分層架構

OT 安全的標準架構是 **Purdue Enterprise Reference Architecture**（ISA-95/IEC 62443）：

```
Level 5  ┌──────────────────┐  企業網路（ERP、Email）
         │   Enterprise     │
Level 4  ├──────────────────┤  IT 網路（商業系統）
         │   Business       │
         ╠══════════════════╡  ← DMZ（非軍事區）
Level 3  ├──────────────────┤  工廠營運（SCADA Server、Historian）
         │   Operations     │
Level 2  ├──────────────────┤  監控層（HMI、工程師工作站）
         │   Supervisory    │
Level 1  ├──────────────────┤  控制層（PLC、RTU）
         │   Control        │
Level 0  ├──────────────────┤  現場設備（感測器、驅動器）
         │   Process        │
         └──────────────────┘
```

csp_lib 通常運行在 **Level 2-3** 之間 — 它直接與 Level 1 的 PLC/RTU 通訊（透過 Modbus），同時將資料向上傳送到 Level 4-5 的 IT 系統（透過 MongoDB/Redis/REST API）。

這個定位意味著 csp_lib 必須同時滿足 OT 的可靠性要求和 IT 的數據需求。

---

## csp_lib 的錯誤層次結構：安全的第一道防線

在 OT 世界裡，**錯誤處理不是可選的，而是安全的核心機制**。一個未處理的例外可能導致控制迴路中斷，進而引發設備損壞甚至安全事故。

csp_lib 設計了明確的錯誤層次結構：

```python
from csp_lib.core.errors import (
    DeviceError,            # 設備層基礎例外
    DeviceConnectionError,  # 連線/斷線失敗
    CommunicationError,     # 讀寫逾時/解碼錯誤
    AlarmError,             # 告警觸發
    ConfigurationError,     # 配置無效（非設備層級）
)
```

這套設計的精髓在於 **每種錯誤都帶有語意**，讓上層可以做出正確的決策：

```python
import asyncio

async def safe_control_loop(device):
    """帶有分級錯誤處理的控制迴路"""
    while True:
        try:
            values = await device.read_once()
            # 正常處理...

        except DeviceConnectionError as e:
            # 連線失敗 → 進入安全模式，等待重連
            # 不要繼續發送控制命令！
            print(f"連線中斷: {e}, 進入安全模式")
            await asyncio.sleep(5)

        except CommunicationError as e:
            # 通訊錯誤 → 可能是暫時的，計數後重試
            # 連續多次失敗才判定為斷線
            print(f"通訊錯誤: {e}, 嘗試重試")

        except AlarmError as e:
            # 告警 → 記錄並通知，但不中斷迴路
            print(f"設備告警: {e.alarm_code}")

        except ConfigurationError as e:
            # 配置錯誤 → 這是程式 bug，必須修復
            print(f"配置錯誤: {e}")
            raise  # 不應該在運行時發生
```

注意 `DeviceError` 攜帶了 `device_id`，`AlarmError` 還額外攜帶 `alarm_code`。這讓上層不需要猜測「到底是哪台設備出了問題」— 錯誤本身就是自描述的。

對比 IT 世界常見的做法 — 用通用的 `Exception` 或 HTTP status code，OT 的錯誤處理要求更高的精確度。因為 **錯誤的分類直接決定了系統的安全回應策略**。

---

## AsyncLifecycleMixin：確保資源永不洩漏

在 OT 世界，「資源洩漏」不只是記憶體問題 — 一個未關閉的 Modbus 連線可能占用 RS-485 總線，導致其他設備無法通訊。csp_lib 用 `AsyncLifecycleMixin` 來解決這個問題：

```python
from csp_lib.core.lifecycle import AsyncLifecycleMixin

class AsyncLifecycleMixin:
    """Async 生命週期 Mixin — 子類別只需覆寫 _on_start/_on_stop"""

    async def start(self) -> None:
        await self._on_start()

    async def stop(self) -> None:
        await self._on_stop()

    async def _on_start(self) -> None:
        """子類別覆寫此方法以實作啟動邏輯"""

    async def _on_stop(self) -> None:
        """子類別覆寫此方法以實作停止邏輯"""

    async def __aenter__(self) -> Self:
        await self.start()
        return self

    async def __aexit__(self, *args) -> None:
        await self.stop()
```

這個設計有幾個 OT-specific 的考量：

### 1. 確定性的生命週期

每個元件都有明確的 `start()` 和 `stop()`。不存在「半啟動」或「不確定是否已關閉」的狀態。在 OT 環境中，**模糊的狀態是危險的**。

### 2. 例外安全的資源管理

`async with` 語法確保無論發生什麼錯誤，`stop()` 都會被呼叫：

```python
from csp_lib.equipment.device.base import AsyncModbusDevice
from csp_lib.equipment.device.config import DeviceConfig
from csp_lib.modbus.clients import PymodbusTcpClient

# 即使讀取過程中發生例外，設備也會被正確斷線
async def monitor_device():
    client = PymodbusTcpClient(host="192.168.1.100", port=502)
    config = DeviceConfig(device_id="pcs-001", unit_id=1)

    async with AsyncModbusDevice(
        config=config,
        client=client,
        always_points=[soc_point],
        write_points=[p_target_point],
    ) as device:
        # 設備已連線並開始讀取循環
        values = await device.read_once()
        print(f"SOC: {values.get('soc')}%")
    # ← 離開 with 區塊時，自動 stop + disconnect
```

### 3. 組合性

由於所有元件都遵循相同的生命週期協定，它們可以自然地組合：

```python
class EnergyStorageSystem(AsyncLifecycleMixin):
    """儲能系統 — 組合多個子元件"""

    def __init__(self, pcs_device, bms_device, data_manager):
        self.pcs = pcs_device
        self.bms = bms_device
        self.data_mgr = data_manager

    async def _on_start(self) -> None:
        await self.pcs.connect()
        await self.bms.connect()
        await self.pcs.start()
        await self.bms.start()
        await self.data_mgr.start()

    async def _on_stop(self) -> None:
        await self.data_mgr.stop()
        await self.pcs.stop()
        await self.bms.stop()
        await self.pcs.disconnect()
        await self.bms.disconnect()

# 整個系統可以用一行啟動/關閉
async with EnergyStorageSystem(pcs, bms, data_mgr) as ess:
    # 所有元件已啟動
    ...
# 所有元件已按正確順序關閉
```

注意 `_on_stop` 中的關閉順序：先停止資料管理（IT 側），再停止設備讀取，最後斷開連線（OT 側）。**順序很重要** — 你不希望在資料還在寫入時就斷開設備連線。

---

## 事件驅動：OT 資料流入 IT 系統

csp_lib 的 `AsyncModbusDevice` 使用事件系統將 OT 側的變化推送給 IT 側的消費者：

```python
from csp_lib.equipment.device.events import ValueChangePayload

async def on_value_change(payload: ValueChangePayload):
    """將 OT 設備的值變化轉發到 IT 系統"""
    print(
        f"[{payload.device_id}] {payload.point_name}: "
        f"{payload.old_value} → {payload.new_value} "
        f"@ {payload.timestamp}"
    )
    # 可以在這裡：
    # - 寫入 MongoDB（Storage 層）
    # - 更新 Redis 快取
    # - 透過 WebSocket 推送到前端

# 註冊事件監聽器
cancel = device.on("value_change", on_value_change)
```

這個事件系統巧妙地將 OT 的「輪詢式」資料採集轉換為 IT 習慣的「事件驅動」模式。OT 設備不會主動推送資料（Modbus 是 master-slave 架構），但 csp_lib 透過比對前後讀取值，自動產生 `value_change` 事件。

---

## 重點回顧

1. **OT 和 IT 有根本不同的優先序**：OT 以可用性和安全性為最高優先，IT 以機密性和資料完整性為最高優先。
2. **融合是必然趨勢**：Industry 4.0、智慧電網、儲能系統都需要 OT 和 IT 的協作。
3. **軟體工程師是關鍵橋樑**：你不需要成為電機專家，但需要理解 OT 世界的基本規則。
4. **csp_lib 的分層架構就是 OT-IT 橋樑**：底層（Layer 2-3）面向 OT，頂層（Layer 7-8）面向 IT，中間層（Layer 4-6）做翻譯。
5. **錯誤處理是安全的一部分**：明確的錯誤層次結構讓系統能根據錯誤類型做出正確的安全回應。
6. **生命週期管理不是 nice-to-have**：在 OT 環境中，未釋放的資源可能導致通訊癱瘓。`AsyncLifecycleMixin` 確保每個元件都有確定性的啟動和關閉。

---

## 下一篇預告

既然我們理解了 OT 和 IT 的差異與融合，下一篇我們將深入探討 OT 世界最核心的主題之一：**工業通訊協定**。Modbus、IEC 61850、IEC 104、MQTT、OPC UA — 這些協定各自解決什麼問題？什麼時候該用哪一個？csp_lib 為什麼選擇從 Modbus 開始？

> **Article 04：工業協定全景圖 — Modbus、IEC 61850、IEC 104、MQTT 與 OPC UA**
