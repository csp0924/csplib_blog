# 從 REST API 到 Modbus：當後端工程師遇上工業協定

> **從零打造工業控制協定框架：給軟體工程師的 CSP Library 實戰系列**
>
> Part 1 — 觀念轉換篇 | Article 01
>
> 上一篇：（系列首篇） | [下一篇：工業場域的時間觀 >>>](./02-industrial-timing.md)

---

## 目錄

1. [為什麼後端工程師該關心工業協定？](#為什麼後端工程師該關心工業協定)
2. [REST API vs Modbus：全面比較](#rest-api-vs-modbus全面比較)
3. [關鍵觀念差異拆解](#關鍵觀念差異拆解)
4. [暫存器模型詳解](#暫存器模型詳解)
5. [csp_lib 的抽象方式：讓 Modbus 變得 Pythonic](#csp_lib-的抽象方式讓-modbus-變得-pythonic)
6. [從 URL 路由到暫存器映射](#從-url-路由到暫存器映射)
7. [重點回顧](#重點回顧)
8. [下篇預告](#下篇預告)

---

## 為什麼後端工程師該關心工業協定？

如果你是一位有 2-3 年經驗的後端工程師，你大概已經對 REST API、JSON、HTTP status code 這些概念駕輕就熟。你能用 FastAPI 或 Django 在一個下午寫出一套 CRUD API，能用 SQLAlchemy 操作資料庫，能用 Redis 做快取。這些都是你的舒適圈。

但有一天，你的主管跟你說：「我們接了一個儲能案場的專案，需要透過 Modbus 跟 PCS（功率調節系統）和 BMS（電池管理系統）溝通。」

你打開 Google 搜尋「Modbus protocol」，看到的第一個詞是「register」（暫存器）。你心想：「這不就是記憶體位址嗎？應該不難吧。」

然後你翻開設備的通訊協議書，看到這樣的表格：

| 位址   | 名稱       | 資料型態   | 倍率  | 單位 | 說明         |
|--------|-----------|-----------|------|------|-------------|
| 5000   | 有功功率   | Float32   | 0.1  | kW   | 即時輸出功率 |
| 5002   | 無功功率   | Float32   | 0.1  | kVar | 即時無功功率 |
| 5004   | SOC       | UInt16    | 0.1  | %    | 電池電量     |
| 5005   | 運行狀態   | UInt16    | -    | -    | 0:停機 1:運行 2:故障 |

你開始意識到，這裡沒有 JSON，沒有 URL，沒有 HTTP method。你面對的是一個 1979 年誕生的二進制協定，而它至今仍是工業自動化領域最廣泛使用的通訊標準。

**這就是這個系列文章想幫你解決的問題**：如何用你已有的後端知識，快速理解工業協定的核心概念，並透過 csp_lib 這套 Python 框架，將這些概念轉化為你熟悉的程式模式。

工業控制不再是只有電機系畢業的人才能碰的領域。隨著 IoT、智慧電網、儲能系統的蓬勃發展，越來越多軟體團隊需要直接與工業設備溝通。你的 Python 技能、你對非同步程式的理解、你的系統設計經驗——這些在工業場域一樣珍貴，甚至更加稀缺。

---

## REST API vs Modbus：全面比較

讓我們從你最熟悉的 REST API 開始，一項一項對比：

| 面向 | REST API | Modbus |
|------|----------|--------|
| **誕生年份** | 2000 年（Roy Fielding 論文） | 1979 年（Modicon 公司） |
| **傳輸層** | HTTP/HTTPS（TCP port 80/443） | TCP port 502（Modbus TCP）或 RS-485 串列埠（RTU） |
| **資料格式** | JSON / XML（文字） | 二進制暫存器（16-bit words） |
| **請求模型** | Client-Server（請求/回應） | Client-Server（主站/從站） |
| **定址方式** | URL 路徑 + HTTP Method | Unit ID + Function Code + Register Address |
| **連線模式** | 通常無狀態（每次請求獨立） | 持久連線（維持 TCP 或串列埠連線） |
| **錯誤處理** | HTTP Status Code（200, 404, 500...） | Exception Code（0x01-0x06） |
| **資料發現** | OpenAPI/Swagger 文件 | 設備通訊協議書（通常是 PDF） |
| **認證授權** | OAuth2, JWT, API Key | 無（靠網路隔離保護） |
| **並行能力** | 多工（HTTP/2, keep-alive） | 單工（一問一答，尤其 RTU） |
| **典型延遲** | 50-500ms（含網路） | 5-50ms（區域網路） |

看到這張表，你可能覺得 Modbus 簡陋得令人驚訝。沒有認證？沒有版本控制？連資料格式都是固定寬度的二進制？

但這正是 Modbus 在工業界稱霸四十餘年的原因：**簡單、可靠、可預測**。在一個感測器可能只有 8KB RAM 的環境裡，JSON 解析器是奢侈品。在一個控制迴路必須在 100 毫秒內完成的場景中，HTTP 的 header 開銷是不可接受的。

---

## 關鍵觀念差異拆解

### 1. HTTP 是無狀態的，Modbus 連線是持久的

身為後端工程師，你習慣了 HTTP 的無狀態模型。每個 request 都是獨立的，server 不需要記住上一次 request 的內容。你靠 session、JWT 來維持狀態。

```python
# 你熟悉的 REST 模式：每次請求獨立
import httpx

async def get_device_power():
    async with httpx.AsyncClient() as client:  # 建立連線
        response = await client.get("https://api.example.com/devices/pcs_01/power")
        return response.json()  # {"active_power": 50.5, "unit": "kW"}
    # 連線關閉
```

Modbus 完全不同。你與設備建立一條 TCP 連線後，這條連線會**一直保持**。你透過這條連線不斷地發送讀取指令、接收回應，就像持續對話一樣。

```python
# Modbus 模式：持久連線
from csp_lib.modbus.clients import PymodbusTcpClient
from csp_lib.modbus.config import ModbusTcpConfig

config = ModbusTcpConfig(host="192.168.1.100", port=502)
client = PymodbusTcpClient(config)

await client.connect()  # 建立持久連線

# 連線保持，持續讀取
registers = await client.read_holding_registers(address=5000, count=2, unit_id=1)
registers = await client.read_holding_registers(address=5002, count=2, unit_id=1)
# ... 每秒讀一次，連線不斷

await client.disconnect()  # 手動斷開
```

為什麼要持久連線？因為工業設備的連線建立成本很高。一個 RS-485 串列匯流排上可能掛了 32 個設備，它們共用一條物理線路。如果每次讀取都要重新建立連線，匯流排的利用率會大幅下降。

### 2. REST 用 JSON，Modbus 用二進制暫存器

在 REST 的世界裡，你讀到的是人類可讀的 JSON：

```json
{
    "device_id": "pcs_01",
    "active_power": 50.5,
    "reactive_power": 12.3,
    "soc": 85.2,
    "status": "running"
}
```

在 Modbus 的世界裡，你讀到的是一串 16-bit 整數：

```
[0x4249, 0x0000, 0x4144, 0xCCCD, 0x0352, 0x0001]
```

是的，就是這樣。沒有欄位名稱，沒有資料型態標記，沒有結構資訊。你必須**事先知道**每個位址存放的是什麼資料、什麼型態、怎麼解碼。

這就像是回到了 C 語言的 `struct` 時代。你必須知道「位址 5000-5001 是一個 IEEE 754 單精度浮點數，代表有功功率，單位是 kW，原始值要乘以 0.1」。

在 csp_lib 中，這個「知道」被編碼成了型態系統：

```python
from csp_lib.modbus import ModbusCodec
from csp_lib.modbus.types import Float32, UInt16, UInt32

codec = ModbusCodec()

# 編碼：Python 值 → 暫存器
registers = codec.encode(Float32(), 50.5)
print(registers)  # [0x4249, 0x0000]

# 解碼：暫存器 → Python 值
value = codec.decode(Float32(), [0x4249, 0x0000])
print(value)  # 50.5

# 整數型態
registers = codec.encode(UInt16(), 852)
print(registers)  # [852]

# 32-bit 整數需要 2 個暫存器
registers = codec.encode(UInt32(), 0x12345678)
print(registers)  # [0x1234, 0x5678]
value = codec.decode(UInt32(), registers)
print(value)  # 305419896
```

`ModbusCodec` 的設計遵循「簡潔 API 封裝複雜邏輯」的原則。你不需要手動處理位元組順序（byte order）或暫存器順序（register order），codec 會根據設定幫你處理。

### 3. REST 有 URL，Modbus 有暫存器位址

REST API 用 URL 來定位資源：

```
GET /api/v1/devices/pcs_01/points/active_power
```

你可以從 URL 看出這是在讀取 `pcs_01` 設備的 `active_power` 點位。語意清晰，自我描述。

Modbus 用數字位址：

```
Function Code: 0x03 (Read Holding Registers)
Starting Address: 5000
Quantity: 2
Unit ID: 1
```

位址 5000 是什麼？不看文件的話，沒有人知道。這就是為什麼 csp_lib 要做一層抽象，把數字位址映射成有意義的名稱（我們稍後會詳細介紹 `ReadPoint`）。

### 4. REST 錯誤碼 vs Modbus 例外碼

你習慣了 HTTP 狀態碼的豐富語意：

| HTTP Status | 含義 |
|------------|------|
| 200 OK | 成功 |
| 400 Bad Request | 請求格式錯誤 |
| 404 Not Found | 資源不存在 |
| 500 Internal Server Error | 伺服器內部錯誤 |
| 503 Service Unavailable | 服務暫時不可用 |

Modbus 的例外碼就精簡多了：

| Exception Code | 含義 | 對應 HTTP 類比 |
|---------------|------|---------------|
| 0x01 | Illegal Function（不支援的功能碼） | 405 Method Not Allowed |
| 0x02 | Illegal Data Address（位址超出範圍） | 404 Not Found |
| 0x03 | Illegal Data Value（值不合法） | 422 Unprocessable Entity |
| 0x04 | Server Device Failure（設備故障） | 500 Internal Server Error |
| 0x06 | Server Device Busy（設備忙碌） | 503 Service Unavailable |

而更常見的「錯誤」其實是**完全沒有回應**——設備斷線了、匯流排被佔用了、或者設備根本不存在。這種情況在 HTTP 世界裡是 timeout，在 Modbus 世界裡卻是日常。

---

## 暫存器模型詳解

Modbus 定義了四種資料區域，這是理解整個協定的基石：

| 資料區域 | 存取權限 | 資料單位 | 功能碼（讀/寫） | 典型用途 |
|---------|---------|---------|---------------|---------|
| **Coils** | 讀/寫 | 1 bit（布林） | FC 01 / FC 05, 0F | 開關控制（啟動、停止） |
| **Discrete Inputs** | 唯讀 | 1 bit（布林） | FC 02 / - | 數位輸入（門磁、限位開關） |
| **Input Registers** | 唯讀 | 16 bits（整數） | FC 04 / - | 量測值（溫度、電壓、電流） |
| **Holding Registers** | 讀/寫 | 16 bits（整數） | FC 03 / FC 06, 10 | 設定值與狀態（功率設定、運行模式） |

用後端的概念來類比：

- **Coils** 像是資料庫的布林欄位（`is_active BOOLEAN`），可讀可寫
- **Discrete Inputs** 像是只能 GET 的布林 endpoint
- **Input Registers** 像是只能 GET 的感測器數據 endpoint
- **Holding Registers** 像是可以 GET 和 PUT 的設定 endpoint

一個暫存器（register）固定是 16 bits，也就是一個 `UInt16`（0-65535）。但現實世界的資料不全都能塞進 16 bits：

- 一個 `Float32`（如有功功率 50.5 kW）需要 **2 個暫存器**
- 一個 `UInt32`（如累計發電量）需要 **2 個暫存器**
- 一個 `Float64`（如高精度計量值）需要 **4 個暫存器**

csp_lib 的型態系統完整覆蓋了這些需求：

```python
from csp_lib.modbus.types import Int16, UInt16, Int32, UInt32, Int64, UInt64, Float32, Float64

# 每個型態知道自己佔幾個暫存器
print(UInt16().register_count)   # 1
print(Float32().register_count)  # 2
print(UInt32().register_count)   # 2
print(Float64().register_count)  # 4
print(Int64().register_count)    # 4
```

這個設計的精妙之處在於：**型態本身攜帶了編解碼邏輯**。你不需要記住「Float32 要用 struct 的 `>f` 格式、佔 2 個暫存器、大端序」這些細節。你只需要宣告 `Float32()`，剩下的交給型態系統。

還有一個後端工程師容易忽略的坑：**位元組順序（Byte Order）和暫存器順序（Register Order）**。

同一個 32-bit 的值 `0x12345678`，不同設備可能用不同的方式存放：

| 排列方式 | 暫存器 0 | 暫存器 1 | 說明 |
|---------|---------|---------|------|
| Big Endian, High First | 0x1234 | 0x5678 | 最常見（AB CD） |
| Big Endian, Low First | 0x5678 | 0x1234 | 有些設備用（CD AB） |
| Little Endian, High First | 0x3412 | 0x7856 | 較少見 |
| Little Endian, Low First | 0x7856 | 0x3412 | 較少見 |

csp_lib 透過 `ByteOrder` 和 `RegisterOrder` 兩個列舉來處理這個問題：

```python
from csp_lib.modbus.enums import ByteOrder, RegisterOrder
from csp_lib.modbus import ModbusCodec
from csp_lib.modbus.types import UInt32

codec = ModbusCodec()

# 預設：大端序 + 高位優先（最常見）
regs = codec.encode(UInt32(), 0x12345678)
# [0x1234, 0x5678]

# 某些設備需要低位優先
regs = codec.encode(
    UInt32(), 0x12345678,
    register_order=RegisterOrder.LOW_FIRST,
)
# [0x5678, 0x1234]
```

---

## csp_lib 的抽象方式：讓 Modbus 變得 Pythonic

到目前為止，我們一直在底層打轉：暫存器、位元組順序、功能碼。這些概念很重要，但你不會想在每次讀取設備時都手動處理它們。

csp_lib 的核心設計理念是：**將 Modbus 的底層細節封裝成 Python 開發者熟悉的模式**。

這個框架分為 8 層架構，從底到頂：

```
Layer 1  Core          基礎工具（日誌、錯誤、生命週期）
Layer 2  Modbus        資料型態、客戶端、編解碼
Layer 3  Equipment     設備抽象、點位、轉換、告警、排程
Layer 4  Controller    控制策略（PQ/QV/FP/孤島...）
Layer 5  Manager       設備管理、告警持久化、資料上傳
Layer 6  Integration   設備註冊、情境建構、指令路由
Layer 7  Storage       MongoDB、Redis
Layer 8  Additional    叢集、監控、通知、GUI
```

作為初學者，你只需要關注 Layer 2（Modbus）和 Layer 3（Equipment）。其他層是進階功能，我們會在後續文章中逐步介紹。

讓我們看看 Layer 2 的核心元件——`ModbusCodec`。它的源碼出奇地簡潔：

```python
# 來自 csp_lib/modbus/codec.py

class ModbusCodec:
    """Modbus 編解碼器"""

    def encode(
        self,
        data_type: ModbusDataType,
        value: Any,
        byte_order: ByteOrder | None = None,
        register_order: RegisterOrder | None = None,
    ) -> list[int]:
        """編碼單一值"""
        try:
            return data_type.encode(
                value,
                byte_order or ByteOrder.BIG_ENDIAN,
                register_order or RegisterOrder.HIGH_FIRST,
            )
        except ModbusEncodeError as e:
            raise e
        except Exception as e:
            raise ModbusEncodeError(f"編碼失敗: {e}") from e

    def decode(
        self,
        data_type: ModbusDataType,
        registers: list[int],
        byte_order: ByteOrder | None = None,
        register_order: RegisterOrder | None = None,
    ) -> Any:
        """解碼暫存器列表"""
        try:
            return data_type.decode(
                registers,
                byte_order or ByteOrder.BIG_ENDIAN,
                register_order or RegisterOrder.HIGH_FIRST,
            )
        except ModbusDecodeError as e:
            raise e
        except Exception as e:
            raise ModbusDecodeError(f"解碼失敗: {e}") from e
```

注意幾個設計重點：

1. **委派模式**：`ModbusCodec` 本身不做編解碼，而是委派給 `data_type`（如 `Float32`、`UInt16`）。這讓新增型態時不需要修改 codec。
2. **合理預設值**：不指定時，預設使用大端序 + 高位優先，這是 Modbus 標準的預設值。
3. **統一的錯誤處理**：所有非預期的例外都被包裝成 `ModbusEncodeError` 或 `ModbusDecodeError`，保持錯誤層級的一致性。

---

## 從 URL 路由到暫存器映射

現在來看最關鍵的抽象：**如何把暫存器位址映射成有意義的命名點位**。

在 REST API 中，你用路由定義資料的存取方式：

```python
# FastAPI 路由
@app.get("/devices/{device_id}/power")
async def get_power(device_id: str):
    return {"active_power": 50.5, "unit": "kW"}
```

在 csp_lib 中，你用 `ReadPoint` 和 `WritePoint` 做同樣的事：

```python
from csp_lib.equipment.core import ReadPoint, WritePoint, PointMetadata
from csp_lib.equipment.core import pipeline, ScaleTransform, RoundTransform, EnumMapTransform
from csp_lib.modbus.types import Float32, UInt16

# 讀取點位：就像定義一個 GET endpoint
active_power = ReadPoint(
    name="active_power",                              # endpoint 名稱
    address=5000,                                     # 暫存器位址（相當於 URL path）
    data_type=Float32(),                              # 資料型態（相當於 response schema）
    pipeline=pipeline(ScaleTransform(0.1), RoundTransform(1)),  # 資料轉換管線
    metadata=PointMetadata(unit="kW", description="有功功率"),   # 後設資料
)

# SOC（電池電量）
soc = ReadPoint(
    name="soc",
    address=5004,
    data_type=UInt16(),
    pipeline=pipeline(ScaleTransform(0.1), RoundTransform(1)),
    metadata=PointMetadata(unit="%", description="電池荷電狀態"),
)

# 運行狀態（數值映射為字串）
status = ReadPoint(
    name="device_status",
    address=5005,
    data_type=UInt16(),
    pipeline=pipeline(EnumMapTransform(
        mapping={0: "STOP", 1: "RUN", 2: "FAULT"},
        default="UNKNOWN",
    )),
    metadata=PointMetadata(description="設備運行狀態"),
)
```

讓我拆解這段程式碼，逐一對應到你熟悉的概念：

| ReadPoint 屬性 | REST API 類比 | 說明 |
|---------------|-------------|------|
| `name` | URL path | 點位的唯一識別名稱 |
| `address` | Resource ID | Modbus 暫存器起始位址 |
| `data_type` | Response Schema | 資料的二進制格式（Float32, UInt16 等） |
| `pipeline` | Response Serializer | 原始值到最終值的轉換鏈 |
| `metadata` | API Documentation | 單位、描述等補充資訊 |

### 轉換管線（Pipeline）：你的資料中介層

`pipeline` 是 csp_lib 中一個特別精巧的設計。它等同於 REST API 中的 middleware 或 serializer——在原始資料和最終呈現之間，做一系列的轉換。

```python
from csp_lib.equipment.core import pipeline, ScaleTransform, RoundTransform, ClampTransform

# 溫度轉換管線：raw * 0.1 - 40，然後四捨五入到小數點後 1 位
temp_pipeline = pipeline(
    ScaleTransform(magnitude=0.1, offset=-40),
    RoundTransform(decimals=1),
)

# 假設暫存器原始值是 650
result = temp_pipeline.process(650)  # (650 * 0.1) - 40 = 25.0
print(result)  # 25.0
```

為什麼需要轉換？因為工業設備為了節省暫存器空間，經常把浮點數「壓縮」成整數。一個溫度值 `25.3°C` 會被存成 `253`（整數），你讀取後需要乘以 `0.1` 才能得到真實值。有些設備更複雜，會加上偏移量——比如 `-40°C` 到 `+80°C` 的範圍，存為 `0-1200`，你需要 `raw * 0.1 - 40` 才能還原。

csp_lib 提供了豐富的內建轉換步驟：

```python
from csp_lib.equipment.core import (
    ScaleTransform,       # 縮放：value * magnitude + offset
    RoundTransform,       # 四捨五入
    EnumMapTransform,     # 數值→字串映射
    ClampTransform,       # 值域限制
    BoolTransform,        # 數值→布林
    BitExtractTransform,  # 位元欄位提取
    InverseTransform,     # 反向縮放（寫入用）
)
```

這些轉換都是 **frozen dataclass**（不可變資料類別），遵循 csp_lib 的配置模式。不可變意味著建立後不會被意外修改——這在工業系統中很重要，因為你不希望某段程式碼意外改了轉換倍率導致功率設定值偏差。

### WritePoint：反向操作

讀取有 `ReadPoint`，寫入自然有 `WritePoint`。寫入時你可能還需要值驗證：

```python
from csp_lib.equipment.core import WritePoint, RangeValidator, InverseTransform

# 功率設定：寫入前驗證範圍，並做反向縮放
power_setpoint = WritePoint(
    name="active_power_setpoint",
    address=6000,
    data_type=Float32(),
    validator=RangeValidator(min_value=-500.0, max_value=500.0),
    pipeline=pipeline(InverseTransform(magnitude=0.1)),
    metadata=PointMetadata(unit="kW", description="有功功率設定"),
)
```

`RangeValidator` 會在寫入前確認值在合法範圍內。想像一下，如果你不小心發送了 `50000 kW` 的功率指令給一台額定 500 kW 的設備，後果可能是設備跳脫保護、甚至硬體損壞。在 REST API 中，這就像是請求驗證（request validation），但在工業場景中，驗證不只是回傳 422 錯誤那麼簡單——它可能關乎設備安全。

---

## 重點回顧

經過這篇文章的介紹，讓我們整理幾個核心觀念：

1. **Modbus 是一個簡單但強大的二進制協定**。它沒有 JSON 的便利性，但在資源受限的工業環境中，簡單就是最大的優勢。

2. **暫存器是 Modbus 的基本資料單位**。一個暫存器 = 16 bits。複雜的資料型態（Float32, UInt32）會跨越多個暫存器。

3. **Modbus 連線是持久的**。不像 HTTP 的請求-回應模型，你需要維護長期連線，並處理斷線重連。

4. **csp_lib 的 `ModbusCodec` 封裝了編解碼細節**。你只需要指定資料型態（如 `Float32()`），不需要手動處理 `struct.pack/unpack`。

5. **`ReadPoint` / `WritePoint` 是暫存器到 Python 物件的映射**。它們就像是 REST API 的路由定義，把無意義的位址數字轉化為有語意的命名點位。

6. **`pipeline` 是資料轉換的中介層**。它串聯多個轉換步驟，處理倍率、偏移、四捨五入等工業場景中常見的資料前處理。

7. **安全驗證在工業場景中不可或缺**。`RangeValidator` 不只是為了程式正確性，更是為了保護物理設備不被錯誤指令損壞。

如果你只記住一件事，那就是：**Modbus 協定本身很簡單，困難的是理解它的思維模式與 REST API 的根本差異**。一旦你完成了這個觀念轉換，你會發現 csp_lib 的設計讓工業通訊變得跟寫 CRUD API 一樣直觀。

---

## 下篇預告

在下一篇文章中，我們將深入探討一個常常被後端工程師忽略的主題：**時間**。

工業系統對時間的要求和 Web 應用完全不同。當你的 REST API 回應從 200ms 變成 500ms 時，使用者可能只是多等了 0.3 秒。但當你的 Modbus 讀取從 100ms 變成 2 秒時，可能意味著控制迴路失效、設備進入保護模式、甚至觸發告警。

我們會探討：
- 硬即時、軟即時、近即時的分類
- csp_lib 的 `ReadScheduler` 如何排程大量點位的讀取
- 為什麼 Python 的 `asyncio` 比你想像的更適合工業軟即時場景
- 連線失敗的時間感知錯誤處理

[下一篇：工業場域的時間觀 >>>](./02-industrial-timing.md)

---

> 本文為「從零打造工業控制協定框架：給軟體工程師的 CSP Library 實戰系列」的第 01 篇。
> 完整系列文章請參閱系列目錄。
