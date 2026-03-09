# 協定轉接器架構：用 Adapter Pattern 統一設備介面

> **從零打造工業控制協定框架：給軟體工程師的 CSP Library 實戰系列**
> Part 2 -- 協定轉接層 | Article 05

---

## 前言

如果你曾在後端開發中整合過第三方支付或物流 API，應該很熟悉這種情境：每家廠商的介面長得不一樣，回傳格式各異，你得為每一家寫一套轉接邏輯。現在把場景搬到工業控制 -- 你的系統需要同時管理來自 10 個不同廠牌的變流器（PCS）、電表（Meter）、儲能電池（BMS），每一台設備的 Modbus 暫存器表完全不同。

這正是 csp_lib 在 Layer 2（Modbus）與 Layer 3（Equipment）所解決的核心問題。本篇文章將帶你深入理解 csp_lib 如何運用 Adapter Pattern 的精神，將千差萬別的設備協定統一成一致的程式介面。

---

## 1. 痛點：每台設備都說不同的方言

想像你正在開發一套儲能系統（BESS）的能源管理平台。你手邊有三台設備：

| 設備 | 廠牌 | 有功功率 (Active Power) 的表示方式 |
|------|------|-----------------------------------|
| 變流器 A | 廠牌甲 | Float32，暫存器 5000-5001，單位 kW |
| 變流器 B | 廠牌乙 | Int16，暫存器 200，原始值需乘以 0.1，單位 kW |
| 電表 C | 廠牌丙 | UInt32，暫存器 3000-3001，低位暫存器在前 |

三台設備量測的是同一個物理量 -- 有功功率，但在通訊協定層面卻完全不同：

- **資料類型不同**：Float32 vs Int16 vs UInt32
- **暫存器位址不同**：5000 vs 200 vs 3000
- **數值轉換不同**：有的需要乘以 0.1，有的直接就是工程值
- **位元組順序不同**：有的高位在前（Big-Endian），有的低位在前（Little-Endian）
- **暫存器順序不同**：32-bit 值拆成兩個 16-bit 暫存器時，高位暫存器先傳還是低位先傳

如果你用最直覺的方式開發，程式碼大概會長這樣：

```python
# 硬編碼地獄 -- 不要學
async def read_active_power(device_type: str, client, unit_id: int) -> float:
    if device_type == "brand_a":
        regs = await client.read_holding_registers(5000, 2, unit_id)
        return struct.unpack(">f", struct.pack(">HH", regs[0], regs[1]))[0]
    elif device_type == "brand_b":
        regs = await client.read_holding_registers(200, 1, unit_id)
        return struct.unpack(">h", struct.pack(">H", regs[0]))[0] * 0.1
    elif device_type == "brand_c":
        regs = await client.read_holding_registers(3000, 2, unit_id)
        return struct.unpack(">I", struct.pack(">HH", regs[1], regs[0]))[0]
    # ... 每新增一個廠牌就多一個 elif
```

這段程式碼有幾個致命問題：

1. **Open-Closed Principle 違反**：每新增一台設備就得修改這個函式
2. **關注點混淆**：位元組操作、數值轉換、業務邏輯全擠在一起
3. **無法測試**：每個分支的邏輯纏繞在條件判斷中
4. **無法複用**：另一個點位（如無功功率）得重寫一遍類似的邏輯

---

## 2. Adapter Pattern：從 GoF 到工業場景

### 經典 GoF Adapter

在 GoF（Gang of Four）設計模式中，Adapter 的定義很清晰：

> 將一個類別的介面，轉換成客戶端所期望的另一個介面。Adapter 讓原本因為介面不相容而不能合作的類別，可以一起運作。

經典的 Adapter 通常是這樣：

```python
# 經典 Adapter -- 概念示意
class Target(Protocol):
    def request(self) -> str: ...

class Adaptee:
    def specific_request(self) -> str:
        return "specific data"

class Adapter:
    def __init__(self, adaptee: Adaptee):
        self._adaptee = adaptee

    def request(self) -> str:
        return self._adaptee.specific_request()
```

### 工業場景的挑戰

但在工業控制領域，Adapter Pattern 面臨更複雜的挑戰：

1. **多維度差異**：不只是「介面不同」，而是資料類型、位址、位元組順序、數值轉換全部都不同
2. **組合爆炸**：如果為每種設備寫一個完整的 Adapter 類別，數量會爆炸
3. **部分共用**：不同設備之間有些邏輯是共用的（例如 Float32 的編解碼邏輯），不應該重複實作

csp_lib 的解法不是寫一堆 Adapter 類別，而是將「轉接」的工作拆解成多個可組合的小元件，透過**宣告式配置**來組合它們。

---

## 3. csp_lib 的分層轉接架構

csp_lib 將轉接工作拆成兩個清楚的層次：

```
Layer 3 (Equipment)
  ReadPoint / WritePoint      <- 統一的點位介面
  ProcessingPipeline          <- 數值轉換管線
  AsyncModbusDevice           <- 設備抽象

Layer 2 (Modbus)
  ModbusDataType              <- 資料類型轉接（Int16, Float32...）
  ModbusCodec                 <- 編解碼器
  AsyncModbusClientBase       <- 通訊客戶端
```

**Layer 2 (Modbus)** 負責解決「暫存器的原始 bytes 如何轉換成 Python 數值」的問題。

**Layer 3 (Equipment)** 負責解決「Python 數值如何進一步轉換成有業務意義的工程值」的問題。

這兩層各自承擔一部分的 Adapter 職責，合在一起就能處理任何設備的差異。

---

## 4. ModbusDataType：資料類型轉接器

`ModbusDataType` 是 csp_lib 中最底層的 Adapter 元件。它是一個抽象基底類別（ABC），定義了從暫存器原始值到 Python 值之間的轉換介面：

```python
from abc import ABC, abstractmethod
from typing import Any

class ModbusDataType(ABC):
    """Modbus 資料類型抽象基底類別"""

    @property
    @abstractmethod
    def register_count(self) -> int:
        """所需的暫存器數量（每個暫存器 16 bits）"""

    @abstractmethod
    def encode(self, value: Any, byte_order, register_order) -> list[int]:
        """將 Python 值編碼為暫存器值列表"""

    @abstractmethod
    def decode(self, registers: list[int], byte_order, register_order) -> Any:
        """將暫存器值列表解碼為 Python 值"""
```

這個介面只做三件事：

1. **告訴你需要幾個暫存器** -- `register_count`
2. **把 Python 值變成暫存器值** -- `encode()`
3. **把暫存器值變回 Python 值** -- `decode()`

### 為什麼這很重要？

因為 Modbus 的世界只有一種基本單位：16-bit 暫存器。不管你的資料是整數、浮點數還是字串，在傳輸層面都是一組 16-bit 暫存器值。`ModbusDataType` 就是把「暫存器值 <-> Python 值」這個轉換封裝成統一的介面。

---

## 5. 型別系統作為轉接器

csp_lib 提供了一整套的具體型別實作，每一個都是 `ModbusDataType` 的子類別：

| 型別 | 暫存器數 | 範圍 | 典型用途 |
|------|---------|------|---------|
| `Int16` | 1 | -32768 ~ 32767 | 有號短整數量測值 |
| `UInt16` | 1 | 0 ~ 65535 | 狀態碼、無號短整數 |
| `Int32` | 2 | -2^31 ~ 2^31-1 | 累計電量（有號） |
| `UInt32` | 2 | 0 ~ 2^32-1 | 累計電量（無號） |
| `Float32` | 2 | IEEE 754 | 功率、電壓、電流 |
| `Float64` | 4 | IEEE 754 | 高精度量測 |
| `Int64` | 4 | -2^63 ~ 2^63-1 | 大範圍有號整數 |
| `UInt64` | 4 | 0 ~ 2^64-1 | 大範圍無號整數 |

回到前面的例子，現在我們可以用宣告式的方式來表達不同設備的差異：

```python
from csp_lib.modbus import Int16, UInt32, Float32

# 設備 A：有功功率用 Float32 表示，位於暫存器 5000
# 設備 B：有功功率用 Int16 表示，位於暫存器 200，原始值需乘以 0.1
# 設備 C：有功功率用 UInt32 表示，位於暫存器 3000，低位暫存器在前

# 三種資料類型，三個不同的 Adapter 實例
device_a_type = Float32()     # 占 2 個暫存器
device_b_type = Int16()       # 占 1 個暫存器
device_c_type = UInt32()      # 占 2 個暫存器
```

每個型別實例都知道自己需要幾個暫存器、如何編碼、如何解碼。你不需要自己去搞 `struct.pack/unpack`，型別系統已經幫你處理了。

讓我們看看 `Float32` 的核心實作邏輯：

```python
class Float32(ModbusDataType):
    """IEEE 754 單精度浮點數，占用 2 個暫存器"""

    @property
    def register_count(self) -> int:
        return 2

    def encode(self, value, byte_order, register_order) -> list[int]:
        packed = struct.pack(f"{byte_order.value}f", float(value))
        return split_to_registers(packed, 2, byte_order, register_order)

    def decode(self, registers, byte_order, register_order) -> float:
        packed = assemble_from_registers(registers, 2, byte_order, register_order)
        return struct.unpack(f"{byte_order.value}f", packed)[0]
```

注意幾個設計要點：

- **byte_order 和 register_order 由外部傳入**，型別本身不綁定特定的位元組順序
- **encode/decode 是對稱的**，確保資料的一致性
- **錯誤驗證內建**，例如 `Int16` 會檢查值是否在 -32768 ~ 32767 範圍內

---

## 6. Transform Pipeline：數值轉換管線

資料從暫存器解碼成 Python 值之後，通常還需要進一步轉換才能得到有業務意義的工程值。例如：

- 暫存器原始值 `500`，乘以 `0.1` 後才是實際的功率 `50.0 kW`
- 溫度感測器的原始值 `650`，需要乘以 `0.1` 再減 `40` 得到 `25.0 度C`
- 功率因數需要四捨五入到小數第二位

csp_lib 的 Transform Pipeline 就是處理這些「後處理」邏輯的機制。

### TransformStep 介面

每一個轉換步驟都實作 `TransformStep` 協定：

```python
class TransformStep(Protocol):
    """轉換步驟介面"""
    def apply(self, value: Any) -> Any: ...
```

### 內建的轉換步驟

csp_lib 提供了多種實用的轉換步驟：

**ScaleTransform -- 線性縮放**

```python
from csp_lib.equipment.core import ScaleTransform

# result = value * magnitude + offset
scale = ScaleTransform(magnitude=0.1, offset=0.0)
scale.apply(500)   # -> 50.0

# 溫度轉換：raw * 0.1 - 40
temp_scale = ScaleTransform(magnitude=0.1, offset=-40)
temp_scale.apply(650)  # -> 25.0
```

**RoundTransform -- 四捨五入**

```python
from csp_lib.equipment.core import RoundTransform

round_step = RoundTransform(decimals=1)
round_step.apply(50.0123)  # -> 50.0
```

**EnumMapTransform -- 數值映射**

```python
from csp_lib.equipment.core import EnumMapTransform

status_map = EnumMapTransform(
    mapping={0: "STOP", 1: "RUN", 2: "FAULT"},
    default="UNKNOWN",
)
status_map.apply(1)  # -> "RUN"
status_map.apply(99) # -> "UNKNOWN"
```

**ClampTransform -- 值域限制**

```python
from csp_lib.equipment.core import ClampTransform

clamp = ClampTransform(min_value=0.0, max_value=100.0)
clamp.apply(150.0)  # -> 100.0
clamp.apply(-10.0)  # -> 0.0
```

**BitExtractTransform -- 位元欄位提取**

```python
from csp_lib.equipment.core import BitExtractTransform

# 從狀態暫存器提取第 0 個位元（布林值）
is_running = BitExtractTransform(bit_offset=0, bit_length=1)
is_running.apply(0b00000101)  # -> True

# 從狀態暫存器提取 bit 8-11（4-bit 數值）
mode = BitExtractTransform(bit_offset=8, bit_length=4)
mode.apply(0x0300)  # -> 3
```

### 組合管線

單一的轉換步驟用處有限，真正強大的是把它們串起來。`pipeline()` 函式是用來組合多個步驟的便捷方法：

```python
from csp_lib.equipment.core import ScaleTransform, RoundTransform, pipeline

# 建立管線：原始值 -> 乘以 0.1 -> 四捨五入到小數第 1 位
power_pipeline = pipeline(
    ScaleTransform(magnitude=0.1),
    RoundTransform(decimals=1),
)

# 處理資料
power_pipeline.process(503)  # 503 * 0.1 = 50.3 -> round(50.3, 1) = 50.3
power_pipeline.process(507)  # 507 * 0.1 = 50.7 -> round(50.7, 1) = 50.7
```

`pipeline()` 回傳的是 `ProcessingPipeline` 實例，它的 `process()` 方法會依序執行每個步驟：

```python
@dataclass(frozen=True)
class ProcessingPipeline:
    steps: tuple[TransformStep, ...]

    def process(self, raw_value: Any) -> Any:
        value = raw_value
        for step in self.steps:
            value = step.apply(value)
        return value
```

這裡使用了標準的 Pipeline Pattern（也稱為 Chain of Responsibility 的變體），每個步驟只負責一件事，組合起來就能處理任意複雜的轉換需求。

### 為什麼是 frozen dataclass？

你可能注意到所有的 Transform 都使用 `@dataclass(frozen=True)`。這是刻意的設計：

1. **不可變性**：轉換邏輯一旦定義就不該被修改，避免執行期間的意外改動
2. **可雜湊**：frozen dataclass 可以作為 dict 的 key 或放入 set
3. **執行緒安全**：不可變物件天然是執行緒安全的

---

## 7. ReadPoint 與 WritePoint：統一的設備介面

有了 `ModbusDataType` 處理底層編解碼，`Pipeline` 處理數值轉換，現在我們需要一個東西把它們黏在一起。這就是 `ReadPoint` 和 `WritePoint` 的角色。

### ReadPoint：讀取點位

```python
from csp_lib.equipment.core import ReadPoint, ScaleTransform, RoundTransform, pipeline
from csp_lib.equipment.core.point import PointMetadata
from csp_lib.modbus import Float32, UInt16

# 設備 A：Float32 直接就是 kW，不需要額外轉換
power_a = ReadPoint(
    name="active_power",
    address=5000,
    data_type=Float32(),
    pipeline=pipeline(RoundTransform(1)),
    metadata=PointMetadata(unit="kW", description="Active power output"),
)

# 設備 B：Int16 原始值需乘以 0.1 才是 kW
power_b = ReadPoint(
    name="active_power",
    address=200,
    data_type=UInt16(),
    pipeline=pipeline(ScaleTransform(0.1), RoundTransform(1)),
    metadata=PointMetadata(unit="kW", description="Active power output"),
)
```

注意，兩個 `ReadPoint` 的 `name` 都是 `"active_power"`。這就是 Adapter Pattern 的精髓 -- 不管底層差異多大，對外暴露的都是同一個語義名稱。上層程式碼（如控制策略）只需要透過 `"active_power"` 這個名稱來取值，完全不需要知道底層是 Float32 還是 Int16。

### WritePoint：寫入點位

```python
from csp_lib.equipment.core import WritePoint
from csp_lib.equipment.core.point import RangeValidator

# 功率設定點：Float32，範圍 -100 ~ 100 kW
p_set = WritePoint(
    name="p_set",
    address=6000,
    data_type=Float32(),
    validator=RangeValidator(min_value=-100.0, max_value=100.0),
    metadata=PointMetadata(unit="kW", description="Active power setpoint"),
)
```

`WritePoint` 除了 `data_type` 之外，還支援 `validator` 用於寫入前的值驗證。這確保了你不會意外寫入超出設備允許範圍的數值 -- 在工業控制中，這種防護至關重要。

### DeviceConfig：設備層級配置

每個設備實例還需要一些獨立於點位定義的配置：

```python
from csp_lib.equipment.device import DeviceConfig

config = DeviceConfig(
    device_id="pcs_01",         # 設備唯一識別碼
    unit_id=1,                  # Modbus 設備位址（Slave ID）
    address_offset=0,           # 位址偏移（某些 PLC 使用 1-based 定址）
    read_interval=1.0,          # 讀取間隔（秒）
    disconnect_threshold=5,     # 連續失敗幾次後視為斷線
)
```

`DeviceConfig` 同樣是 frozen dataclass，設定完就不再改變。

---

## 8. EquipmentTemplate 與 DeviceFactory：多型號支援

當你的系統需要管理大量設備時（例如 20 台同型號的變流器），逐一定義每台設備的點位會很繁瑣。csp_lib 提供了 `EquipmentTemplate` 和 `DeviceFactory` 來解決這個問題。

### EquipmentTemplate：設備模型範本

`EquipmentTemplate` 封裝了一個設備型號的完整定義：

```python
from csp_lib.equipment.template import EquipmentTemplate

pcs_template = EquipmentTemplate(
    model="SUN2000-100KTL",
    description="Huawei 100kW PCS",
    always_points=(
        ReadPoint(name="active_power", address=5000, data_type=Float32(),
                  pipeline=pipeline(ScaleTransform(0.1), RoundTransform(1))),
        ReadPoint(name="soc", address=5034, data_type=UInt16(),
                  pipeline=pipeline(ScaleTransform(0.1))),
    ),
    write_points=(
        WritePoint(name="p_set", address=6000, data_type=Float32(),
                   validator=RangeValidator(min_value=-100.0, max_value=100.0)),
    ),
)
```

一個 Template 定義了所有「與型號相關」的資訊，但不包含「與實例相關」的資訊（如 IP 位址、unit_id）。

### DeviceFactory：批次建立設備

`DeviceFactory` 可以從 Template 快速建立設備實例：

```python
from csp_lib.equipment.template import DeviceFactory

# 建立單一設備
device = DeviceFactory.create(
    template=pcs_template,
    config=DeviceConfig(device_id="pcs_01", unit_id=1),
    client=tcp_client,
)

# 批次建立：10 台同型號設備
configs = [
    DeviceConfig(device_id=f"pcs_{i:02d}", unit_id=i)
    for i in range(1, 11)
]
devices = DeviceFactory.create_batch(
    template=pcs_template,
    instances=configs,
    client_factory=lambda cfg: tcp_client,
)
```

更進階的場景是 `create_stride` -- 當多台設備共用同一條 Modbus 線路，但暫存器位址是等間距偏移的（例如 BMS 子模組）：

```python
# 10 個 BMS 子模組，位址間距 100
sub_bms_devices = DeviceFactory.create_stride(
    template=sub_bms_template,
    base_config=DeviceConfig(device_id="sub_bms", unit_id=1),
    client_factory=lambda cfg: shared_client,
    count=10,
    stride=100,
    id_format="{base_id}_{index}",
)
# 產生 sub_bms_1, sub_bms_2, ..., sub_bms_10
# 位址偏移分別為 0, 100, 200, ..., 900
```

---

## 9. 全景圖：資料如何從暫存器變成工程值

讓我們追蹤一個完整的資料流程，看看各元件如何協作：

```
Modbus 暫存器 [0x4248, 0x0000]
       |
       | (1) AsyncModbusClientBase.read_holding_registers()
       v
Python list: [0x4248, 0x0000]
       |
       | (2) ModbusDataType(Float32).decode()
       |     -- 組合暫存器 bytes，解析 IEEE 754
       v
Python float: 50.0
       |
       | (3) ProcessingPipeline.process()
       |     -- ScaleTransform(0.1) -> 5.0
       |     -- RoundTransform(1)   -> 5.0
       v
工程值: 5.0 kW
       |
       | (4) 存入 device.latest_values["active_power"]
       |     -- 觸發 value_change 事件
       v
上層程式碼使用
```

每一層只做自己的事：

- **Client** 負責把 bytes 從設備搬到程式裡
- **DataType** 負責把暫存器值解碼成 Python 原生型別
- **Pipeline** 負責把原始數值轉成有意義的工程值
- **Device** 負責排程讀取、管理狀態、發送事件

這就是 csp_lib 版本的 Adapter Pattern -- 不是一個大 Adapter 類別，而是多個小元件各司其職，透過宣告式配置組合在一起。

---

## 10. 重點回顧

1. **工業設備的異質性**是真實存在的痛點：同一物理量在不同設備上可能有完全不同的表示方式
2. **經典 Adapter Pattern 不夠用**：工業場景的差異是多維度的，需要更細粒度的抽象
3. **csp_lib 的解法是分層轉接**：
   - Layer 2 的 `ModbusDataType` 處理暫存器 <-> Python 值的轉換
   - Layer 3 的 `ProcessingPipeline` 處理原始值 -> 工程值的轉換
   - `ReadPoint/WritePoint` 將兩者黏合成統一的點位介面
4. **宣告式配置**取代了大量的 if-else 邏輯：新增設備只需要新增一組 Point 定義
5. **不可變設計**（frozen dataclass）確保了轉換邏輯的可靠性和執行緒安全性
6. **EquipmentTemplate + DeviceFactory** 解決了大規模部署時的重複定義問題

---

## 下一篇預告

在下一篇文章中，我們將深入 Layer 2 的 Modbus 客戶端實作。你將會看到 csp_lib 如何封裝 TCP 與 RTU 兩種通訊模式、SharedPymodbusTcpClient 如何用參考計數管理共用連線，以及 ModbusCodec 的編解碼實戰。

> **Article 06：Modbus TCP/RTU 實戰 -- 從零封裝到生產級客戶端**
