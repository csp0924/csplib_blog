# Modbus TCP/RTU 實戰：從零封裝到生產級客戶端

> **從零打造工業控制協定框架：給軟體工程師的 CSP Library 實戰系列**
> Part 2 -- 協定轉接層 | Article 06

---

## 前言

上一篇我們談到 csp_lib 如何用 Adapter Pattern 統一不同設備的介面。但光有抽象介面是不夠的 -- 你還需要一個可靠的底層通訊客戶端，真正把資料從設備搬到你的程式裡。

本篇將深入 csp_lib 的 Layer 2（Modbus 層），帶你理解 Modbus TCP/RTU 的核心概念，並實際看看 csp_lib 如何將 pymodbus 封裝成生產級的非同步客戶端。如果你曾經用過 HTTP client library（如 `httpx` 或 `aiohttp`），本篇的架構思維會讓你很有共鳴。

---

## 1. Modbus 協定速覽

### 為什麼要學 Modbus？

Modbus 是 1979 年由 Modicon（現 Schneider Electric）提出的工業通訊協定。雖然已經 40 多歲了，但它至今仍是工業控制領域最廣泛使用的協定之一。原因很簡單：它**簡單、開放、無授權費用**。

幾乎所有的工業設備 -- 變流器、電表、BMS、溫溼度感測器 -- 都支援 Modbus。如果你要做工業軟體，Modbus 是你繞不過的第一課。

### 資料模型

Modbus 定義了四種資料區域：

| 資料區域 | 讀/寫 | 資料大小 | 功能碼（讀） | 功能碼（寫） | 典型用途 |
|---------|-------|---------|------------|------------|---------|
| Coils | 讀寫 | 1 bit | 0x01 | 0x05/0x0F | 開關控制 |
| Discrete Inputs | 唯讀 | 1 bit | 0x02 | -- | 開關狀態 |
| Holding Registers | 讀寫 | 16 bit | 0x03 | 0x06/0x10 | 設定值、量測值 |
| Input Registers | 唯讀 | 16 bit | 0x04 | -- | 量測值 |

在實際應用中，最常用的是 **Holding Registers (0x03/0x10)** 和 **Input Registers (0x04)**。大部分的量測數據（電壓、電流、功率）和控制命令（功率設定值、啟停命令）都存放在這兩種暫存器中。

### PDU 結構

Modbus 的請求封包（PDU）結構非常簡潔：

```
請求: [功能碼 1byte] [起始位址 2bytes] [數量 2bytes]
回應: [功能碼 1byte] [位元組數 1byte] [資料 N bytes]
```

例如，讀取 Holding Register 位址 100 開始的 2 個暫存器：

```
請求: [0x03] [0x00 0x64] [0x00 0x02]
回應: [0x03] [0x04] [0x12 0x34 0x56 0x78]
```

回應中的 4 個 bytes 就是 2 個暫存器的值：`0x1234` 和 `0x5678`。

### TCP vs RTU

Modbus 有兩種主要的傳輸方式：

| | Modbus TCP | Modbus RTU |
|---|-----------|-----------|
| 傳輸介質 | Ethernet (TCP/IP) | 串列埠 (RS-485/RS-232) |
| 定址 | IP + Port + Unit ID | Slave Address |
| 速度 | 通常 100Mbps | 通常 9600~115200 bps |
| 連線模型 | 一對一 TCP 連線 | 多設備共用匯流排 |
| 並行性 | 可多工 | 序列化（一次只能問一台） |

**Modbus TCP** 就是把 Modbus PDU 包在 TCP 封包裡傳輸。每個設備有獨立的 IP 位址和 port（預設 502），client 可以同時持有多個連線。

**Modbus RTU** 則是透過串列埠（如 RS-485）傳輸，多台設備共用一條匯流排。因為是共用匯流排，所以一次只能跟一台設備通訊 -- 這就像一條單線道的高速公路，車子必須排隊通過。

還有一種混合場景：**TCP-RS485 轉換器**。前端是 TCP 連線，後端接 RS-485 匯流排。雖然連線方式是 TCP，但底層仍然是序列化的 RS-485 通訊，所以同樣需要排隊機制。

---

## 2. csp_lib 的客戶端階層

理解了 Modbus 的基礎之後，讓我們看看 csp_lib 如何設計客戶端架構：

```
AsyncModbusClientBase (抽象基底類別)
  ├── PymodbusTcpClient      -- 獨立 TCP 連線，一對一
  ├── PymodbusRtuClient      -- RTU 串列埠，多設備共用 + 請求佇列
  └── SharedPymodbusTcpClient -- TCP 共用連線，多設備共用 + 請求佇列
```

### AsyncModbusClientBase：統一介面

所有客戶端都繼承自 `AsyncModbusClientBase`，它定義了完整的非同步操作介面：

```python
class AsyncModbusClientBase(ABC):
    """Modbus 非同步客戶端抽象介面"""

    @abstractmethod
    async def connect(self) -> None: ...

    @abstractmethod
    async def disconnect(self) -> None: ...

    @abstractmethod
    async def is_connected(self) -> bool: ...

    # 讀取操作
    @abstractmethod
    async def read_coils(self, address: int, count: int, unit_id: int = 1) -> list[bool]: ...

    @abstractmethod
    async def read_discrete_inputs(self, address: int, count: int, unit_id: int = 1) -> list[bool]: ...

    @abstractmethod
    async def read_holding_registers(self, address: int, count: int, unit_id: int = 1) -> list[int]: ...

    @abstractmethod
    async def read_input_registers(self, address: int, count: int, unit_id: int = 1) -> list[int]: ...

    # 寫入操作
    @abstractmethod
    async def write_single_coil(self, address: int, value: bool, unit_id: int = 1) -> None: ...

    @abstractmethod
    async def write_single_register(self, address: int, value: int, unit_id: int = 1) -> None: ...

    @abstractmethod
    async def write_multiple_coils(self, address: int, values: list[bool], unit_id: int = 1) -> None: ...

    @abstractmethod
    async def write_multiple_registers(self, address: int, values: list[int], unit_id: int = 1) -> None: ...

    # 支援 async context manager
    async def __aenter__(self) -> AsyncModbusClientBase:
        await self.connect()
        return self

    async def __aexit__(self, exc_type, exc_val, exc_tb) -> None:
        await self.disconnect()
```

幾個重要的設計決策：

1. **所有方法都接受 `unit_id` 參數**：unit_id 不是客戶端的屬性，而是每次呼叫時傳入的。這讓多個設備可以共用同一個客戶端連線。
2. **支援 async context manager**：`async with` 語法自動管理連線生命週期。
3. **回傳型別明確**：暫存器讀取回傳 `list[int]`，線圈讀取回傳 `list[bool]`。

---

## 3. ModbusTcpConfig 與 ModbusRtuConfig

在建立客戶端之前，你需要先準備連線設定。csp_lib 使用 frozen dataclass 來表示設定，確保設定一旦建立就不可修改：

### TCP 設定

```python
from csp_lib.modbus import ModbusTcpConfig

tcp_config = ModbusTcpConfig(
    host="192.168.1.100",         # 目標 IP
    port=502,                     # 連接埠（預設 502）
    timeout=0.5,                  # 通訊逾時（秒）
    byte_order=ByteOrder.BIG_ENDIAN,        # 位元組順序
    register_order=RegisterOrder.HIGH_FIRST, # 暫存器順序
)
```

### RTU 設定

```python
from csp_lib.modbus import ModbusRtuConfig
from csp_lib.modbus.enums import Parity

rtu_config = ModbusRtuConfig(
    port="/dev/ttyUSB0",          # 串口名稱（Linux）
    # port="COM3",                # 串口名稱（Windows）
    baudrate=9600,                # 鮑率
    parity=Parity.NONE,           # 校驗位元
    stopbits=1,                   # 停止位元
    bytesize=8,                   # 資料位元
    timeout=0.5,                  # 通訊逾時（秒）
)
```

兩個設定類別都有 `__post_init__` 驗證，在建立時就會檢查參數合理性：

```python
# 會拋出 ModbusConfigError
ModbusTcpConfig(host="", port=502)            # host 不可為空
ModbusTcpConfig(host="192.168.1.1", port=0)   # port 必須 1-65535
ModbusTcpConfig(host="192.168.1.1", timeout=-1) # timeout 必須為正數
```

這種「建構時驗證」的策略稱為 **Fail Fast** -- 與其讓無效設定在執行時造成奇怪的錯誤，不如在建立時就立刻失敗並給出清楚的錯誤訊息。

---

## 4. 三種客戶端的使用場景

### PymodbusTcpClient：一對一 TCP 連線

最單純的使用場景：你的程式直接透過 Ethernet 連到一台 Modbus TCP 設備。

```python
from csp_lib.modbus import PymodbusTcpClient, ModbusTcpConfig

client = PymodbusTcpClient(ModbusTcpConfig(host="192.168.1.100", port=502))

async with client:
    # 讀取暫存器 5000 開始的 2 個暫存器（例如 Float32）
    registers = await client.read_holding_registers(
        address=5000, count=2, unit_id=1
    )
    print(registers)  # [0x4248, 0x0000] -> Float32 = 50.0

    # 寫入暫存器 6000 開始的 2 個暫存器
    await client.write_multiple_registers(
        address=6000, values=[0x4248, 0x0000], unit_id=1
    )
```

`PymodbusTcpClient` 的實作很直接 -- 它是 pymodbus `AsyncModbusTcpClient` 的薄封裝。每個實例持有一個獨立的 TCP 連線，支援全雙工通訊（可以同時讀寫）。

關鍵設計：

- **Lazy 初始化**：pymodbus 客戶端在第一次 `connect()` 或 `_get_client()` 時才建立
- **Lazy import**：`pymodbus` 是 optional dependency，只有在實際使用客戶端時才會 import，確保在不需要 Modbus 功能的場景下不會因為缺少 pymodbus 而報錯
- **retries=0**：不在客戶端層做重試，重試策略交給上層（設備層）處理

### PymodbusRtuClient：串列埠通訊

當你使用 RS-485 串口連接設備時：

```python
from csp_lib.modbus import PymodbusRtuClient, ModbusRtuConfig

client = PymodbusRtuClient(ModbusRtuConfig(port="/dev/ttyUSB0", baudrate=9600))

async with client:
    registers = await client.read_holding_registers(
        address=100, count=1, unit_id=1
    )
```

RTU 客戶端有一個重要的特性：**Singleton per port**。同一個串口（如 `/dev/ttyUSB0`）無論你建立幾個 `PymodbusRtuClient` 實例，底層都共用同一個 pymodbus 連線和請求佇列。

為什麼需要這樣？因為 RS-485 是共用匯流排架構，同一條線路上的所有設備共用一個物理連線。如果兩個執行緒同時往同一個串口發送請求，訊號會衝突。所以 csp_lib 使用 `ModbusRequestQueue` 來序列化所有請求，確保一次只有一個請求在匯流排上傳輸。

資源管理採用**參考計數（reference counting）**：

```python
# 三個設備共用同一個串口
client_a = PymodbusRtuClient(rtu_config)  # ref_count: 0
client_b = PymodbusRtuClient(rtu_config)
client_c = PymodbusRtuClient(rtu_config)

await client_a.connect()  # ref_count: 1, 建立連線
await client_b.connect()  # ref_count: 2, 共用連線
await client_c.connect()  # ref_count: 3

await client_a.disconnect()  # ref_count: 2, 連線維持
await client_b.disconnect()  # ref_count: 1, 連線維持
await client_c.disconnect()  # ref_count: 0, 關閉連線並清理資源
```

### SharedPymodbusTcpClient：TCP-RS485 轉換器

這是一種混合場景，也是實務中非常常見的配置。你的控制器透過 TCP 連到一台 TCP-RS485 轉換器（如 Moxa NPort），轉換器後面接 RS-485 匯流排，匯流排上掛著多台設備。

```
[你的程式] --TCP--> [TCP-RS485 轉換器] --RS485--> [設備1 unit_id=1]
                                              --> [設備2 unit_id=2]
                                              --> [設備3 unit_id=3]
```

雖然連線方式是 TCP，但底層仍然是序列化的 RS-485。所以 `SharedPymodbusTcpClient` 的行為跟 `PymodbusRtuClient` 類似：

- 同一個 `host:port` 的多個實例**共用一個 TCP 連線**
- 所有請求透過 `ModbusRequestQueue` **序列化存取**
- 參考計數管理連線生命週期

```python
from csp_lib.modbus import SharedPymodbusTcpClient, ModbusTcpConfig

config = ModbusTcpConfig(host="192.168.1.12", port=502)

# 三台設備共用同一個 TCP 連線
client_1 = SharedPymodbusTcpClient(config)
client_2 = SharedPymodbusTcpClient(config)
client_3 = SharedPymodbusTcpClient(config)
```

**什麼時候用 PymodbusTcpClient，什麼時候用 SharedPymodbusTcpClient？**

| 場景 | 選擇 |
|-----|------|
| 每台設備有獨立的 IP | `PymodbusTcpClient` |
| 多台設備透過 TCP-RS485 轉換器 | `SharedPymodbusTcpClient` |
| 需要最大吞吐量 | `PymodbusTcpClient`（支援多工） |
| 設備在 RS-485 匯流排上 | `PymodbusRtuClient` |

---

## 5. ModbusCodec：編解碼器

前面提到的 `ModbusDataType`（如 `Float32`、`UInt32`）負責單一值的編解碼，但每次呼叫都需要傳入 `byte_order` 和 `register_order`。`ModbusCodec` 是一個更高階的封裝，讓編解碼操作更簡潔：

```python
from csp_lib.modbus import ModbusCodec, UInt32, Float32, Int16
from csp_lib.modbus.enums import ByteOrder, RegisterOrder

codec = ModbusCodec()

# === 編碼：Python 值 -> 暫存器列表 ===

# UInt32 編碼
regs = codec.encode(UInt32(), 0x12345678)
print(regs)  # [0x1234, 0x5678]  (預設 Big-Endian, High-First)

# Float32 編碼
regs = codec.encode(Float32(), 50.0)
print(regs)  # [0x4248, 0x0000]

# Int16 編碼
regs = codec.encode(Int16(), -100)
print(regs)  # [0xFF9C]

# === 解碼：暫存器列表 -> Python 值 ===

val = codec.decode(UInt32(), [0x1234, 0x5678])
print(val)  # 305419896 (0x12345678)

val = codec.decode(Float32(), [0x4248, 0x0000])
print(val)  # 50.0

val = codec.decode(Int16(), [0xFF9C])
print(val)  # -100
```

`ModbusCodec` 的 `encode()` 和 `decode()` 方法接受可選的 `byte_order` 和 `register_order` 參數。如果不指定，預設使用 `BIG_ENDIAN` 和 `HIGH_FIRST` -- 這是 Modbus 標準的預設值。

---

## 6. 位元組順序與暫存器順序陷阱

這是 Modbus 開發中最容易踩到的坑。理論上 Modbus 標準規定使用 Big-Endian，但實際上每家設備廠商可能有不同的實作。

### ByteOrder（位元組順序）

決定**單一暫存器內**位元組的排列方式：

```
Big-Endian (Modbus 標準):    高位元組在前
  值 0x1234 -> [0x12, 0x34]

Little-Endian:               低位元組在前
  值 0x1234 -> [0x34, 0x12]
```

### RegisterOrder（暫存器順序）

決定**多個暫存器之間**的排列方式（只在 32-bit 或 64-bit 資料類型時有影響）：

```
HIGH_FIRST (常見):  高位暫存器在前
  32-bit 值 0x12345678 -> [0x1234, 0x5678]
  記作 "AB CD"

LOW_FIRST:         低位暫存器在前
  32-bit 值 0x12345678 -> [0x5678, 0x1234]
  記作 "CD AB"
```

### 實務中的四種組合

將 ByteOrder 和 RegisterOrder 排列組合，就有四種可能的方式來表示一個 32-bit 的值：

| ByteOrder | RegisterOrder | 0x12345678 的暫存器表示 | 記法 |
|-----------|--------------|----------------------|------|
| Big-Endian | High-First | [0x1234, 0x5678] | AB CD |
| Big-Endian | Low-First | [0x5678, 0x1234] | CD AB |
| Little-Endian | High-First | [0x3412, 0x7856] | BA DC |
| Little-Endian | Low-First | [0x7856, 0x3412] | DC BA |

不同的設備廠商可能使用不同的組合。例如：

- **ABB**：通常使用 Big-Endian + High-First（最標準的方式）
- **某些 Schneider 設備**：使用 Big-Endian + Low-First
- **某些歐系 PLC**：使用 Little-Endian + Low-First

在 csp_lib 中，你可以在 `ModbusTcpConfig` / `ModbusRtuConfig` 中設定預設的 byte_order 和 register_order，也可以在 `ReadPoint` / `WritePoint` 中針對個別點位覆寫：

```python
from csp_lib.modbus import ModbusTcpConfig
from csp_lib.modbus.enums import ByteOrder, RegisterOrder

# 全域設定：這台設備使用 Big-Endian + Low-First
config = ModbusTcpConfig(
    host="192.168.1.100",
    byte_order=ByteOrder.BIG_ENDIAN,
    register_order=RegisterOrder.LOW_FIRST,
)
```

```python
from csp_lib.equipment.core import ReadPoint
from csp_lib.modbus import UInt32
from csp_lib.modbus.enums import ByteOrder, RegisterOrder

# 個別點位覆寫：這個點位特別使用 Little-Endian
special_point = ReadPoint(
    name="cumulative_energy",
    address=3000,
    data_type=UInt32(),
    byte_order=ByteOrder.LITTLE_ENDIAN,
    register_order=RegisterOrder.LOW_FIRST,
)
```

**實務建議**：拿到設備的 Modbus 通訊手冊後，第一件事就是確認 byte_order 和 register_order。如果手冊沒說清楚（很常見），最簡單的辦法是讀一個已知值（例如設備額定容量），用四種組合都試一遍，看哪一種解碼出來是對的。

---

## 7. 完整設備建立範例

讓我們把前面學到的所有概念串在一起，建立一個完整的設備：

```python
import asyncio
from csp_lib.equipment.core import (
    ReadPoint, WritePoint, ScaleTransform, RoundTransform, pipeline,
)
from csp_lib.equipment.core.point import PointMetadata, RangeValidator
from csp_lib.equipment.device import AsyncModbusDevice, DeviceConfig
from csp_lib.modbus import Float32, UInt16, PymodbusTcpClient, ModbusTcpConfig

# ============================================================
# Step 1: 建立 Modbus 客戶端
# ============================================================
client = PymodbusTcpClient(
    ModbusTcpConfig(host="192.168.1.100", port=502)
)

# ============================================================
# Step 2: 設備設定
# ============================================================
config = DeviceConfig(
    device_id="pcs_01",
    unit_id=1,
    read_interval=1.0,         # 每秒讀取一次
    disconnect_threshold=5,    # 連續失敗 5 次視為斷線
)

# ============================================================
# Step 3: 定義讀取點位
# ============================================================
active_power = ReadPoint(
    name="active_power",
    address=5000,
    data_type=Float32(),
    pipeline=pipeline(ScaleTransform(0.1), RoundTransform(1)),
    metadata=PointMetadata(unit="kW", description="Active power output"),
)

soc = ReadPoint(
    name="soc",
    address=5034,
    data_type=UInt16(),
    pipeline=pipeline(ScaleTransform(0.1)),
    metadata=PointMetadata(unit="%", description="Battery state of charge"),
)

# ============================================================
# Step 4: 定義寫入點位
# ============================================================
p_set = WritePoint(
    name="p_set",
    address=6000,
    data_type=Float32(),
    validator=RangeValidator(min_value=-100.0, max_value=100.0),
    metadata=PointMetadata(unit="kW", description="Active power setpoint"),
)

# ============================================================
# Step 5: 建立設備
# ============================================================
device = AsyncModbusDevice(
    config=config,
    client=client,
    always_points=[active_power, soc],
    write_points=[p_set],
)

# ============================================================
# Step 6: 使用設備
# ============================================================
async def main():
    # 註冊事件處理器
    async def on_value_change(payload):
        print(f"[{payload.device_id}] {payload.point_name}: "
              f"{payload.old_value} -> {payload.new_value}")

    device.on("value_change", on_value_change)

    # 使用 async context manager 管理生命週期
    async with device:
        print(f"Connected: {device.is_connected}")
        print(f"Latest values: {device.latest_values}")

        # 寫入功率設定值
        result = await device.write("p_set", 50.0, verify=True)
        print(f"Write result: {result.status.value}")

        # 讓設備持續運行 10 秒
        await asyncio.sleep(10)

    # 離開 context manager 後，設備自動斷線

asyncio.run(main())
```

這段程式碼展示了完整的流程：

1. **客戶端** (`PymodbusTcpClient`) 負責底層 TCP 通訊
2. **設定** (`DeviceConfig`) 定義設備身份和行為參數
3. **點位** (`ReadPoint` / `WritePoint`) 定義資料結構和轉換邏輯
4. **設備** (`AsyncModbusDevice`) 整合一切，提供自動讀取迴圈和事件驅動

### 設備啟動後發生了什麼？

當你執行 `async with device:` 時，設備會依序執行：

1. `connect()` -- 透過客戶端建立 Modbus 連線
2. `start()` -- 啟動定期讀取迴圈

讀取迴圈的每一輪會：

1. 根據 `always_points` 計算需要讀取的暫存器範圍
2. 透過客戶端發送 Modbus 讀取請求
3. 用 `ModbusDataType` 解碼暫存器值
4. 用 `ProcessingPipeline` 轉換為工程值
5. 比對前一次的值，如果有變化就觸發 `value_change` 事件
6. 等待 `read_interval` 秒後再次讀取

當你離開 `async with` 區塊時，設備會自動 `stop()` 並 `disconnect()`。

---

## 8. 錯誤處理

csp_lib 的 Modbus 模組定義了清晰的例外層級：

```
ModbusError (基礎)
  ├── ModbusEncodeError      -- 編碼失敗（值超出範圍、型別不符）
  ├── ModbusDecodeError      -- 解碼失敗（暫存器數量不足、格式錯誤）
  ├── ModbusConfigError      -- 設定錯誤（無效的 port、timeout）
  ├── ModbusCircuitBreakerError -- 斷路器開啟（設備連續失敗達閾值）
  └── ModbusQueueFullError   -- 請求佇列已滿（RTU/Shared 模式）
```

### ModbusEncodeError 與 ModbusDecodeError

這兩個是最常見的資料層錯誤。通常在開發階段就能發現：

```python
from csp_lib.modbus import ModbusCodec, Int16, UInt16
from csp_lib.modbus.exceptions import ModbusEncodeError, ModbusDecodeError

codec = ModbusCodec()

# 編碼錯誤：值超出範圍
try:
    codec.encode(Int16(), 40000)  # Int16 最大 32767
except ModbusEncodeError as e:
    print(e)  # "Int16 範圍為 -32768~32767，收到: 40000"

# 編碼錯誤：型別不符
try:
    codec.encode(UInt16(), "hello")
except ModbusEncodeError as e:
    print(e)  # "UInt16 需要整數，收到: str"

# 解碼錯誤：暫存器數量不足
try:
    codec.decode(Float32(), [0x4248])  # Float32 需要 2 個暫存器
except ModbusDecodeError as e:
    print(e)  # "Float32 需要 2 個暫存器，收到: 1"
```

### ModbusCircuitBreakerError

`PymodbusRtuClient` 和 `SharedPymodbusTcpClient` 的請求佇列內建斷路器（Circuit Breaker）機制。當某個 `unit_id` 的設備連續失敗達到閾值時，斷路器會打開，暫時拒絕該設備的所有請求：

```python
from csp_lib.modbus.exceptions import ModbusCircuitBreakerError

try:
    await client.read_holding_registers(100, 2, unit_id=3)
except ModbusCircuitBreakerError as e:
    print(f"設備 unit_id={e.unit_id} 的斷路器已開啟，暫時無法通訊")
```

這個機制避免了一台故障設備拖慢整條匯流排的問題 -- 如果某台設備一直不回應，與其每次都等到 timeout，不如直接跳過它，把通訊頻寬留給正常的設備。

### ModbusQueueFullError

在高負載場景下，如果請求佇列塞滿了（達到 `max_queue_size` 上限），新的請求會被拒絕：

```python
from csp_lib.modbus.exceptions import ModbusQueueFullError

try:
    await client.read_holding_registers(100, 2, unit_id=1)
except ModbusQueueFullError:
    print("請求佇列已滿，請稍後再試")
```

---

## 9. pymodbus 版本相容性

csp_lib 需要支援不同版本的 pymodbus（>= 3.0.0）。pymodbus 3.10.0 引入了一個 breaking change：`slave` 參數改名為 `device_id`。csp_lib 透過一個相容層來處理這個差異：

```python
# csp_lib 內部的相容層（使用者不需要關心）
from .compat import slave_kwarg

# 自動偵測 pymodbus 版本，回傳正確的參數名稱
# pymodbus < 3.10.0: {"slave": unit_id}
# pymodbus >= 3.10.0: {"device_id": unit_id}
response = await client.read_holding_registers(
    address=100, count=2, **slave_kwarg(unit_id)
)
```

這個設計讓使用者完全不需要擔心 pymodbus 版本問題 -- csp_lib 在內部自動處理。

---

## 10. 重點回顧

1. **Modbus 是工業通訊的基石**：理解功能碼、暫存器模型、TCP vs RTU 的差異是必備知識
2. **csp_lib 提供三種客戶端**，對應三種實務場景：
   - `PymodbusTcpClient`：獨立 TCP 連線，最單純
   - `PymodbusRtuClient`：串口共用匯流排，自動序列化和參考計數
   - `SharedPymodbusTcpClient`：TCP-RS485 轉換器場景
3. **ByteOrder + RegisterOrder 的組合**是最容易踩的坑，務必查閱設備手冊
4. **ModbusCodec** 提供便捷的高階編解碼 API
5. **完善的錯誤層級**讓你可以精確捕捉和處理不同類型的故障
6. **參考計數管理共用資源**，確保連線生命週期的正確性

---

## 下一篇預告

Modbus 雖然是工業界最普及的協定，但在電力系統的 SCADA 領域，IEC 60870-5-104 才是真正的主角。下一篇我們將介紹 IEC 104 協定的核心概念，並探討它與 Modbus 的差異、以及如何用 csp_lib 的抽象層來橋接不同的協定。

> **Article 07：IEC 60870-5-104 入門 -- 電力系統的通訊骨幹**
