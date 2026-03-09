# 工業協定全景圖：Modbus、IEC 61850、IEC 104、MQTT 與 OPC UA

> **從零打造工業控制協定框架：給軟體工程師的 CSP Library 實戰系列**
> Part 1 — 觀念轉換篇 | Article 04

---

## 前言：為什麼有這麼多協定？

如果你來自 Web 開發的世界，你可能會覺得困惑：「HTTP 不就夠了嗎？為什麼工業界需要這麼多通訊協定？」

答案很簡單：**因為工業環境的需求比 Web 複雜得多。**

一個智慧電網系統可能同時需要：

- **毫秒級的即時控制** — 保護繼電器必須在 4ms 內動作
- **確定性的資料傳輸** — 功率命令不能「偶爾」到達
- **數十年的向後相容** — 1979 年的設備到現在還在運行
- **跨廠商的互操作** — A 廠的 PCS 要跟 B 廠的 BMS 對話
- **低頻寬環境** — RS-485 串列通訊只有 9600 baud

沒有任何單一協定能同時滿足所有需求。這就是為什麼不同的應用場景催生了不同的協定，而理解它們的定位和取捨，是每位跨入 OT 領域的軟體工程師必備的知識。

---

## 協定總覽：一張表看懂五大協定

| 特性 | Modbus TCP/RTU | IEC 61850 | IEC 60870-5-104 | MQTT / Sparkplug B | OPC UA |
|------|---------------|-----------|-----------------|-------------------|--------|
| **誕生年代** | 1979 (RTU) / 1999 (TCP) | 2003 | 2000 | 1999 (MQTT) / 2016 (Sparkplug) | 2008 |
| **傳輸層** | TCP 或 RS-485 序列 | TCP (MMS)、L2 Ethernet (GOOSE/SMV) | TCP/IP | TCP/IP (MQTT Broker) | TCP/IP |
| **資料模型** | 平坦的暫存器表（無語意） | 階層式物件模型（IED → LD → LN → DO → DA） | 資訊物件（Information Object） | Topic + Payload（自定義或 Sparkplug） | 節點（Node）構成的位址空間 |
| **典型延遲** | 10-100ms | < 4ms (GOOSE)、10-100ms (MMS) | 50-200ms | 100ms-數秒 | 10-100ms |
| **安全性** | 無（明文傳輸） | 有（IEC 62351） | 可選（TLS） | TLS + 認證 | 內建加密、認證、授權 |
| **複雜度** | 極低 | 極高 | 中等 | 低（MQTT）/ 中（Sparkplug） | 高 |
| **主要場景** | 工廠自動化、儲能、太陽能 | 變電站自動化 | 電力 SCADA 遠端監控 | IIoT、雲端對接 | 工廠自動化、跨平台整合 |
| **csp_lib 支援** | 完整支援 | 規劃中（設備抽象層可擴展） | 規劃中 | 可透過 Storage 層對接 | 規劃中 |

接下來，讓我們逐一深入。

---

## Modbus TCP/RTU：最廣泛採用的工業協定

### 為什麼 Modbus 能活 40+ 年？

Modbus 是 1979 年由 Modicon（現在的 Schneider Electric）發明的。它能存活超過 40 年，靠的不是技術先進，而是 **極致的簡單**：

- 協定規格只有幾十頁（對比 IEC 61850 的數千頁）
- 實作一個 Modbus master 只需要幾百行程式碼
- 幾乎所有工業設備都支援

### 資料模型：暫存器就是一切

Modbus 的資料模型極為簡單 — 就是一張「暫存器表」：

| 類型 | 位址範圍 | 大小 | 讀/寫 | 用途 |
|------|---------|------|------|------|
| Coils | 00001-09999 | 1 bit | R/W | 數位輸出（繼電器開關） |
| Discrete Inputs | 10001-19999 | 1 bit | R | 數位輸入（按鈕狀態） |
| Input Registers | 30001-39999 | 16 bit | R | 類比輸入（感測器讀值） |
| Holding Registers | 40001-49999 | 16 bit | R/W | 類比輸出/設定值 |

一個暫存器就是 16-bit 的整數。沒有資料型別的概念、沒有語意標記、沒有結構定義。如果你需要表示一個 32-bit 浮點數，你得用 **兩個連續的暫存器** 來存放，而且不同廠商對位元組順序的約定可能不同。

這正是 csp_lib 的資料型別系統要解決的問題。

### csp_lib 的完整 Modbus 支援

#### 資料型別系統

csp_lib 提供了完整的 Modbus 資料型別，處理所有位元組順序和暫存器順序的組合：

```python
from csp_lib.modbus.types import (
    Int16,      # 16-bit 有號整數（1 暫存器）
    UInt16,     # 16-bit 無號整數（1 暫存器）
    Int32,      # 32-bit 有號整數（2 暫存器）
    UInt32,     # 32-bit 無號整數（2 暫存器）
    Int64,      # 64-bit 有號整數（4 暫存器）
    UInt64,     # 64-bit 無號整數（4 暫存器）
    Float32,    # IEEE 754 單精度浮點（2 暫存器）
    Float64,    # IEEE 754 雙精度浮點（4 暫存器）
    DynamicInt, # 動態長度有號整數
    DynamicUInt,# 動態長度無號整數
    ModbusString, # 字串類型
)

from csp_lib.modbus.enums import ByteOrder, RegisterOrder

# Float32 佔用 2 個暫存器
f32 = Float32()
print(f32.register_count)  # 2

# 編碼：將浮點數轉為暫存器值
registers = f32.encode(
    value=3.14,
    byte_order=ByteOrder.BIG_ENDIAN,
    register_order=RegisterOrder.HIGH_FIRST,
)
print(registers)  # [16584, 62914] — 兩個 16-bit 暫存器值

# 解碼：將暫存器值還原為浮點數
value = f32.decode(
    registers=[16584, 62914],
    byte_order=ByteOrder.BIG_ENDIAN,
    register_order=RegisterOrder.HIGH_FIRST,
)
print(f"{value:.2f}")  # 3.14
```

為什麼 `ByteOrder` 和 `RegisterOrder` 都需要？因為 Modbus 的世界沒有統一標準：

- **ByteOrder** 決定單一暫存器內的位元組排列（Big-Endian `AB` vs Little-Endian `BA`）
- **RegisterOrder** 決定多暫存器之間的排列（High-First `AB CD` vs Low-First `CD AB`）

不同廠商的設備可能使用不同組合。Schneider Electric 慣用 Big-Endian + High-First，而某些 SMA 的太陽能逆變器使用 Big-Endian + Low-First。csp_lib 的型別系統讓你可以在點位定義層級處理這些差異，而不是在業務邏輯裡到處做 byte-swapping。

#### 功能碼

Modbus 使用功能碼（Function Code）來區分操作類型：

```python
from csp_lib.modbus.enums import FunctionCode

# 讀取功能碼
FunctionCode.READ_COILS              # 0x01 — 讀取線圈
FunctionCode.READ_DISCRETE_INPUTS    # 0x02 — 讀取離散輸入
FunctionCode.READ_HOLDING_REGISTERS  # 0x03 — 讀取保持暫存器（最常用）
FunctionCode.READ_INPUT_REGISTERS    # 0x04 — 讀取輸入暫存器

# 寫入功能碼
FunctionCode.WRITE_SINGLE_COIL       # 0x05 — 寫入單一線圈
FunctionCode.WRITE_SINGLE_REGISTER   # 0x06 — 寫入單一暫存器
FunctionCode.WRITE_MULTIPLE_COILS    # 0x0F — 寫入多個線圈
FunctionCode.WRITE_MULTIPLE_REGISTERS # 0x10 — 寫入多個暫存器
```

在 csp_lib 中，`ReadPoint` 預設使用 `READ_HOLDING_REGISTERS (0x03)`，`WritePoint` 預設使用 `WRITE_MULTIPLE_REGISTERS (0x10)`。這些預設值覆蓋了大約 90% 的使用場景：

```python
from csp_lib.equipment.core.point import ReadPoint, WritePoint
from csp_lib.modbus.types import UInt16

# ReadPoint 預設 function_code = READ_HOLDING_REGISTERS
temp = ReadPoint(name="temperature", address=100, data_type=UInt16())

# 如果需要讀取 Input Register，明確指定
analog_input = ReadPoint(
    name="current_sensor",
    address=200,
    data_type=UInt16(),
    function_code=FunctionCode.READ_INPUT_REGISTERS,
)
```

#### 三種客戶端

csp_lib 提供三種 Modbus 客戶端，對應不同的通訊場景：

```python
from csp_lib.modbus.clients import (
    PymodbusTcpClient,        # TCP 獨立連線
    SharedPymodbusTcpClient,  # TCP 共用連線（TCP-RS485 轉換器）
    PymodbusRtuClient,        # RS-485 串列 RTU
)
```

**`PymodbusTcpClient`** — 最常見的選擇。每台設備一個 TCP 連線：
```
[csp_lib] ─── TCP ───→ [PCS (192.168.1.100:502)]
          ─── TCP ───→ [BMS (192.168.1.101:502)]
```

**`SharedPymodbusTcpClient`** — 當一個 TCP-RS485 轉換器後面掛了多台設備時使用。多個 `AsyncModbusDevice` 共用同一條 TCP 連線，用 `unit_id` 區分設備：
```
[csp_lib] ─── TCP ───→ [TCP-RS485 Gateway] ─── RS485 ───→ [Device A (unit=1)]
                                             ─── RS485 ───→ [Device B (unit=2)]
                                             ─── RS485 ───→ [Device C (unit=3)]
```

**`PymodbusRtuClient`** — 直接透過 RS-485 序列埠通訊。配備 `ModbusRequestQueue` 做請求排隊，因為 RS-485 是半雙工的（一次只能一個裝置傳輸）。

### Modbus 的局限

Modbus 的簡單也是它的弱點：

1. **沒有自描述能力** — 你必須從設備手冊查詢每個暫存器的意義
2. **沒有內建安全機制** — 所有通訊都是明文，任何人都能發送寫入命令
3. **效率低** — 每次請求最多讀取 125 個暫存器，大量點位需要多次請求
4. **沒有事件通知** — 只能靠輪詢（polling），不能由設備主動推送

---

## IEC 61850：電力系統的王者標準

### 為什麼電力系統需要自己的標準？

變電站自動化有著極其嚴苛的要求：

- **保護動作**必須在 **4ms 以內**完成（GOOSE 訊息）
- **電力品質監測**需要 **每秒 4000 個取樣點**（SMV 取樣值）
- **不同廠商的 IED**（Intelligent Electronic Device）必須能互操作

Modbus 無法滿足這些需求，因此 IEC 61850 應運而生。

### 三合一的通訊架構

IEC 61850 不是單一協定，而是三種通訊機制的組合：

| 服務 | 傳輸方式 | 延遲 | 用途 |
|------|---------|------|------|
| **MMS**（Manufacturing Message Specification） | TCP/IP | 10-100ms | 監控讀寫、設定管理 |
| **GOOSE**（Generic Object Oriented Substation Events） | L2 Ethernet（直接） | < 4ms | 即時保護、聯鎖 |
| **SMV**（Sampled Measured Values） | L2 Ethernet（直接） | < 1ms | 電流/電壓取樣值 |

GOOSE 和 SMV 跳過了整個 TCP/IP 堆疊，直接在 Ethernet 的 Layer 2 傳輸。這是它們能達到毫秒級延遲的關鍵。

### 豐富的資料模型

與 Modbus 的「暫存器表」相比，IEC 61850 的資料模型是一棵完整的語意樹：

```
Server（伺服器）
  └─ Logical Device（邏輯設備）
       └─ Logical Node（邏輯節點）—— 例如 MMXU（三相電力量測）
            └─ Data Object（資料物件）—— 例如 TotW（總有效功率）
                 └─ Data Attribute（資料屬性）—— 例如 mag.f（浮點值）
```

這意味著 IEC 61850 的資料點自帶語意。`MMXU.TotW.mag.f` 你不查手冊也知道它是「三相量測節點的總有效功率的浮點值」。

### csp_lib 如何與 IEC 61850 相容？

雖然 csp_lib 目前不直接支援 IEC 61850 協定，但其設備抽象層的設計已經考慮了可擴展性。關鍵在於 `DeviceProtocol` 介面：

```python
from csp_lib.equipment.device.protocol import DeviceProtocol

@runtime_checkable
class DeviceProtocol(Protocol):
    """設備通用協定 — 所有設備類型的最小公開介面"""

    @property
    def device_id(self) -> str: ...

    @property
    def is_connected(self) -> bool: ...

    @property
    def is_responsive(self) -> bool: ...

    @property
    def latest_values(self) -> dict[str, Any]: ...

    async def read_once(self) -> dict[str, Any]: ...

    async def write(self, name: str, value: Any, **kwargs: Any) -> WriteResult: ...

    def on(self, event: str, handler: AsyncHandler) -> Callable[[], None]: ...

    def health(self) -> HealthReport: ...
```

一個假想的 IEC 61850 設備實作可能長這樣：

```python
class AsyncIEC61850Device:
    """IEC 61850 設備（概念示意）"""

    @property
    def device_id(self) -> str:
        return self._ied_name

    async def read_once(self) -> dict[str, Any]:
        # 透過 MMS 讀取邏輯節點的資料物件
        values = {}
        for ln in self._logical_nodes:
            data = await self._mms_client.read(f"{ln.ref}")
            values.update(data)
        return values

    async def write(self, name: str, value: Any, **kwargs) -> WriteResult:
        # 透過 MMS 寫入控制命令（含 SBO 安全鎖定機制）
        await self._mms_client.operate(name, value)
        return WriteResult(success=True, point_name=name, value=value)
```

由於上層的 Manager、Controller、Integration 層都只依賴 `DeviceProtocol` 介面而非具體的 `AsyncModbusDevice` 類別，替換底層協定不需要修改任何上層程式碼。這就是**依賴反轉**在 OT 領域的具體應用。

---

## IEC 60870-5-104：SCADA 遠端監控

### 為什麼 SCADA 需要專用協定？

IEC 104（簡稱）是電力系統 SCADA（Supervisory Control And Data Acquisition）的標準遠端通訊協定。它的設計場景是：

- 控制中心與遠端變電站之間的 **長距離通訊**
- 頻寬有限的 WAN 環境（傳統上可能只有 9600bps 專線）
- 需要 **時間戳記** 的事件回報

### 平衡模式 vs 非平衡模式

IEC 104 的前身 IEC 101 有兩種通訊模式：

- **非平衡模式（Unbalanced）**：主站輪詢從站，類似 Modbus
- **平衡模式（Balanced）**：雙方都可以主動發起通訊

IEC 104 基於 TCP/IP，天然支援平衡模式。從站可以 **主動推送** 事件（如告警、狀態變化），不需要主站輪詢。這是它相比 Modbus 的一大優勢。

### 資訊物件

IEC 104 的資料單位是 **資訊物件（Information Object）**，每個物件有：

- **IOA（Information Object Address）**：類似 Modbus 的暫存器位址
- **型別識別碼（Type ID）**：明確定義資料型別（單點、雙點、浮點、時間戳記等）
- **品質描述（Quality Descriptor）**：標記資料是否有效、是否被替代

「品質描述」是一個很 OT 的概念 — IT 世界很少會在每個資料點上標記「這個數值是否可信」。但在 OT 世界，一個感測器可能故障了但還在回傳數值，你必須有機制區分「正常數值」和「故障數值」。

### 與 csp_lib 的關係

IEC 104 的「資訊物件」概念與 csp_lib 的 `ReadPoint` 有很高的對應性。兩者都是「具有位址和型別的命名資料點」。如果未來 csp_lib 要支援 IEC 104，`PointDefinition` 的結構可以自然延伸：

```python
# 現有的 Modbus 點位定義
@dataclass(frozen=True)
class PointDefinition:
    name: str
    address: int              # Modbus register address
    data_type: ModbusDataType  # 資料型別
    function_code: FunctionCode | None = None
    byte_order: ByteOrder = ByteOrder.BIG_ENDIAN
    register_order: RegisterOrder = RegisterOrder.HIGH_FIRST

# 概念上的 IEC 104 點位定義（未來擴展方向）
# address → IOA
# data_type → Type ID 映射
# 新增 quality 欄位
```

---

## MQTT 與 Sparkplug B：通往雲端的橋樑

### MQTT：輕量級的發布/訂閱

MQTT（Message Queuing Telemetry Transport）不是工業專用協定，但它在 IIoT（工業物聯網）領域被廣泛採用，原因是：

- **極低的頻寬開銷** — 最小封包只有 2 bytes
- **發布/訂閱模式** — 解耦了生產者和消費者
- **QoS 保證** — 三種品質等級（0: 最多一次, 1: 至少一次, 2: 恰好一次）
- **遺囑機制（Last Will）** — 設備離線時自動通知

### Sparkplug B：讓 MQTT 懂工業

原生 MQTT 只定義了傳輸機制，不定義資料格式。`topic: factory/line1/temp` 的 payload 是 JSON、Protobuf、還是原始 bytes？MQTT 不管。

**Sparkplug B** 是 Eclipse Foundation 定義的 MQTT 應用層規範，為工業場景加入了：

- **標準化的 Topic 結構**：`spBv1.0/{group_id}/{message_type}/{edge_node_id}/{device_id}`
- **Birth/Death 證書**：設備上線時發布 NBIRTH（Node Birth），離線時 Broker 自動發布 NDEATH（Node Death）
- **Protobuf 編碼的 Payload**：統一的資料格式，包含時間戳記、型別、品質
- **指標（Metric）**的概念：類似 csp_lib 的 `ReadPoint`

### MQTT 在 csp_lib 架構中的位置

MQTT 不適合做底層設備控制（延遲不確定、依賴 Broker），但非常適合做 **資料上行**：

```
                    ┌──────────┐
                    │  Cloud   │  Level 5
                    │ Platform │
                    └────▲─────┘
                         │ MQTT (Sparkplug B)
                    ┌────┴─────┐
                    │  MQTT    │  Broker
                    │  Broker  │
                    └────▲─────┘
                         │
   ┌─────────────────────┴──────────────────────┐
   │            csp_lib (Edge)                  │  Level 2-3
   │                                            │
   │  [Modbus] ←→ [Equipment] → [Storage] → [MQTT Publisher]
   │                                            │
   └────────────────────────────────────────────┘
```

csp_lib 的事件系統可以自然地對接 MQTT 發布：

```python
import json

async def publish_to_mqtt(payload):
    """將設備值變化發布到 MQTT"""
    topic = f"csp/{payload.device_id}/{payload.point_name}"
    message = json.dumps({
        "value": payload.new_value,
        "timestamp": payload.timestamp.isoformat(),
        "device_id": payload.device_id,
    })
    await mqtt_client.publish(topic, message, qos=1)

# 註冊到 csp_lib 的事件系統
device.on("value_change", publish_to_mqtt)
```

---

## OPC UA：通用翻譯官

### 資訊模型：不只是資料，還有語意

OPC UA（Open Platform Communications Unified Architecture）的最大野心是成為 **工業通訊的通用語言**。它的核心不是傳輸協定（雖然也定義了），而是 **資訊模型（Information Model）**。

OPC UA 的位址空間是一張有向圖，每個節點有：

- **NodeId**：唯一識別
- **BrowseName**：人類可讀的名稱
- **NodeClass**：物件、變數、方法、事件等
- **References**：節點之間的關係

這意味著你可以「瀏覽」一台 OPC UA 伺服器，就像瀏覽檔案系統一樣，自動發現它提供了哪些資料點。Modbus 做不到這點 — 你必須事先知道暫存器位址。

### 內建的安全機制

OPC UA 從一開始就把安全設計納入：

- **加密**：支援 AES-128/256
- **認證**：X.509 憑證
- **授權**：細粒度的存取控制
- **稽核**：操作日誌

對比 Modbus 的完全無安全機制，OPC UA 在安全性上有質的飛躍。

### 複雜度的代價

OPC UA 的強大帶來了高昂的複雜度：

- 規格文件超過 **1200 頁**
- 完整實作一個 OPC UA Server 是一項巨大的工程
- 嵌入式設備可能沒有足夠的運算資源
- 配置憑證和安全策略本身就需要專業知識

這是為什麼 Modbus 在簡單場景中依然不可取代 — **不是所有問題都需要最強大的解決方案**。

---

## 什麼時候該用哪個？決策樹

做協定選型時，你可以按照以下邏輯思考：

```
你的需求是什麼？
│
├─ 簡單的設備讀寫（感測器/變頻器/逆變器）
│   ├─ 設備支援 Modbus？ → ✅ Modbus TCP/RTU
│   └─ 需要更多安全性？ → 考慮 OPC UA
│
├─ 變電站自動化（保護/量測/控制）
│   └─ → IEC 61850（業界強制要求）
│
├─ 電力 SCADA 遠端監控
│   └─ → IEC 60870-5-104
│
├─ 邊緣設備到雲端的資料上傳
│   ├─ 需要標準化？ → MQTT + Sparkplug B
│   └─ 簡單就好？ → MQTT + 自定義 Payload
│
└─ 跨廠商、跨平台的設備整合
    └─ → OPC UA
```

在實際專案中，**混合使用多種協定是常態**。一個儲能系統可能同時使用：

- **Modbus** 與 PCS/BMS 通訊（設備控制層）
- **MQTT** 將運行資料上傳到雲端（資料上行層）
- **REST API** 接收調度指令（指令下行層）

---

## csp_lib 的協定無關設計

csp_lib 雖然目前專注於 Modbus，但其架構設計是 **協定無關（protocol-agnostic）** 的。這是怎麼做到的？

### 抽象層的解耦

看看 csp_lib 的依賴方向：

```
Manager/Integration 層
        │
        ▼ 依賴於
DeviceProtocol（介面）
        ▲
        │ 實作
AsyncModbusDevice（具體類別）
```

上層只認識 `DeviceProtocol` 介面，不認識 `AsyncModbusDevice` 具體類別。這意味著你可以為任何協定實作一個新的設備類別，只要它滿足 `DeviceProtocol` 的契約。

### ReadPoint/WritePoint 的可移植性

csp_lib 的點位定義雖然目前綁定了 Modbus 的概念（暫存器位址、功能碼），但核心的資料處理管線是協定無關的：

```python
from csp_lib.equipment.core.transform import ScaleTransform, EnumMapTransform
from csp_lib.equipment.core.pipeline import ProcessingPipeline

# 這個管線不關心資料來自 Modbus、IEC 61850 還是 OPC UA
status_pipeline = ProcessingPipeline(steps=(
    EnumMapTransform(
        mapping={0: "STOP", 1: "RUN", 2: "FAULT"},
        default="UNKNOWN",
    ),
))

# ScaleTransform 純粹是數學轉換，與協定無關
temperature_pipeline = ProcessingPipeline(steps=(
    ScaleTransform(magnitude=0.1, offset=-40),  # raw → 攝氏溫度
    RoundTransform(decimals=1),
))
```

`ScaleTransform` 不在乎你的原始值是從 Modbus 暫存器讀來的，還是從 OPC UA 節點讀來的。它只做一件事：`value * magnitude + offset`。這種設計讓轉換邏輯可以在不同協定之間復用。

### 事件系統的協定無關性

csp_lib 的事件系統同樣不綁定任何協定。無論底層是什麼協定，上層消費者收到的都是相同的事件格式：

```python
from csp_lib.equipment.device.events import (
    EVENT_VALUE_CHANGE,     # 值變化
    EVENT_CONNECTED,        # 設備連線
    EVENT_DISCONNECTED,     # 設備斷線
    EVENT_ALARM_TRIGGERED,  # 告警觸發
    EVENT_ALARM_CLEARED,    # 告警解除
    ValueChangePayload,
)

# 這段程式碼不關心設備用什麼協定通訊
async def log_changes(payload: ValueChangePayload):
    print(f"[{payload.device_id}] {payload.point_name}: "
          f"{payload.old_value} → {payload.new_value}")

device.on(EVENT_VALUE_CHANGE, log_changes)
```

### DeviceConfig：統一的設備配置

所有設備共享統一的配置結構：

```python
from csp_lib.equipment.device.config import DeviceConfig

config = DeviceConfig(
    device_id="pcs-001",          # 設備唯一識別碼
    unit_id=1,                     # Modbus unit ID（其他協定可重新詮釋）
    read_interval=1.0,             # 讀取間隔（秒）
    reconnect_interval=5.0,        # 重連間隔（秒）
    disconnect_threshold=5,        # 連續失敗閾值
)
```

`read_interval`、`reconnect_interval`、`disconnect_threshold` 這些參數是通訊的通用概念，不局限於 Modbus。任何需要定期輪詢的協定都需要這些設定。

---

## 實戰範例：完整的 Modbus 設備定義

讓我們把所有概念串起來，看一個完整的 PCS（功率轉換系統）設備定義：

```python
from csp_lib.modbus.types import Float32, UInt16, Int16
from csp_lib.modbus.enums import ByteOrder, RegisterOrder, FunctionCode
from csp_lib.modbus.clients import PymodbusTcpClient
from csp_lib.equipment.core.point import (
    ReadPoint, WritePoint, PointMetadata, RangeValidator,
)
from csp_lib.equipment.core.transform import (
    ScaleTransform, RoundTransform, EnumMapTransform,
)
from csp_lib.equipment.core.pipeline import ProcessingPipeline
from csp_lib.equipment.device.base import AsyncModbusDevice
from csp_lib.equipment.device.config import DeviceConfig

# ── 讀取點位定義 ──

pcs_read_points = [
    # 有效功率（kW）
    ReadPoint(
        name="active_power",
        address=0x0000,
        data_type=Float32(),
        pipeline=ProcessingPipeline(steps=(
            RoundTransform(decimals=2),
        )),
        metadata=PointMetadata(unit="kW", description="即時有效功率"),
    ),
    # 無效功率（kVar）
    ReadPoint(
        name="reactive_power",
        address=0x0002,
        data_type=Float32(),
        pipeline=ProcessingPipeline(steps=(
            RoundTransform(decimals=2),
        )),
        metadata=PointMetadata(unit="kVar", description="即時無效功率"),
    ),
    # 設備狀態
    ReadPoint(
        name="status",
        address=0x0010,
        data_type=UInt16(),
        pipeline=ProcessingPipeline(steps=(
            EnumMapTransform(
                mapping={0: "STANDBY", 1: "RUNNING", 2: "FAULT", 3: "MAINTENANCE"},
                default="UNKNOWN",
            ),
        )),
        metadata=PointMetadata(description="設備運行狀態"),
    ),
    # 機櫃溫度
    ReadPoint(
        name="cabinet_temp",
        address=0x0020,
        data_type=Int16(),
        pipeline=ProcessingPipeline(steps=(
            ScaleTransform(magnitude=0.1),
            RoundTransform(decimals=1),
        )),
        metadata=PointMetadata(unit="°C", description="機櫃內部溫度"),
    ),
]

# ── 寫入點位定義 ──

pcs_write_points = [
    WritePoint(
        name="p_target",
        address=0x0100,
        data_type=Float32(),
        validator=RangeValidator(min_value=-500.0, max_value=500.0),
        metadata=PointMetadata(unit="kW", description="目標有效功率"),
    ),
    WritePoint(
        name="q_target",
        address=0x0102,
        data_type=Float32(),
        validator=RangeValidator(min_value=-250.0, max_value=250.0),
        metadata=PointMetadata(unit="kVar", description="目標無效功率"),
    ),
]

# ── 組裝設備 ──

async def main():
    client = PymodbusTcpClient(host="192.168.1.100", port=502)
    config = DeviceConfig(
        device_id="pcs-001",
        unit_id=1,
        read_interval=1.0,
        disconnect_threshold=5,
    )

    async with AsyncModbusDevice(
        config=config,
        client=client,
        always_points=pcs_read_points,
        write_points=pcs_write_points,
    ) as pcs:
        # 讀取一次
        values = await pcs.read_once()
        print(f"有效功率: {values['active_power']} kW")
        print(f"設備狀態: {values['status']}")
        print(f"機櫃溫度: {values['cabinet_temp']} °C")

        # 寫入功率目標
        result = await pcs.write("p_target", 100.0)
        print(f"寫入結果: {result}")
```

這段程式碼展示了 csp_lib 如何將 OT 世界的複雜性（暫存器位址、位元組順序、資料型別轉換、值域驗證）封裝在清晰的 Python API 背後，讓 IT 背景的工程師也能自然地與工業設備互動。

---

## 重點回顧

1. **工業協定的多樣性源自需求的多樣性** — 沒有銀彈，不同場景需要不同協定。
2. **Modbus 是入門首選** — 簡單、普及，csp_lib 提供完整支援。
3. **IEC 61850 是電力系統的王者** — 豐富的資料模型和毫秒級延遲，但複雜度極高。
4. **IEC 104 專注 SCADA 遠端監控** — 支援事件推送和時間戳記。
5. **MQTT + Sparkplug B 是 IIoT 的首選** — 輕量、靈活，適合雲端對接。
6. **OPC UA 追求大一統** — 強大的資訊模型和安全機制，但代價是複雜度。
7. **csp_lib 的協定無關設計** — 透過 `DeviceProtocol` 介面和資料處理管線的解耦，為未來支援更多協定預留了空間。
8. **混合使用是常態** — 實際專案通常在不同層級使用不同協定。

---

## 下一篇預告

觀念轉換篇到這裡告一段落。從下一篇開始，我們進入 **Part 2 — 核心實作篇**，捲起袖子寫程式碼。第一站：深入 csp_lib 的 Modbus 資料型別系統 — 為什麼一個 16-bit 暫存器需要這麼多型別？`ByteOrder` 和 `RegisterOrder` 的排列組合到底有幾種？如何處理不同廠商的「方言」？

> **Article 05：Modbus 資料型別系統 — 從 16-bit 暫存器到工程值的旅程**
