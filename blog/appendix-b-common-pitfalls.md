# 附錄 B：常見陷阱與除錯技巧

> **從零打造工業控制協定框架：給軟體工程師的 CSP Library 實戰系列**
> 附錄 | Appendix B

---

工業控制軟體開發有許多「踩過才知道」的坑。這篇附錄收集了使用 CSP Library 時最常遇到的陷阱，涵蓋 Modbus 通訊、Python async 程式設計、控制策略、測試、部署和效能六大面向。每個陷阱都附有錯誤範例、正確做法和背後的原因說明。

建議你先快速瀏覽一遍，留個印象。等到實際開發時遇到問題，再回來查找對應的段落。

---

## 1. Modbus 通訊陷阱

### 1.1 位元組順序混淆

這是 Modbus 開發中最常見也最隱蔽的問題。不同廠商的設備可能使用不同的位元組順序（byte order）和暫存器順序（register order），即使它們都「遵循 Modbus 標準」。

```python
from csp_lib.modbus.enums import ByteOrder, RegisterOrder

# Modbus 標準定義：Big-Endian + High-First
# 但不是所有設備都遵守！

# 常見組合 1（最標準）：大端序 + 高位暫存器在前
byte_order = ByteOrder.BIG_ENDIAN        # ">"
register_order = RegisterOrder.HIGH_FIRST  # AB CD

# 常見組合 2（某些歐系 PLC）：大端序 + 低位暫存器在前
byte_order = ByteOrder.BIG_ENDIAN         # ">"
register_order = RegisterOrder.LOW_FIRST   # CD AB

# 常見組合 3（某些美系設備）：小端序 + 低位暫存器在前
byte_order = ByteOrder.LITTLE_ENDIAN      # "<"
register_order = RegisterOrder.LOW_FIRST   # DC BA
```

**症狀**：讀到的數值看起來不對，但又不是完全隨機的。例如應該讀到 `1000.0` 卻讀到 `3.57e-43` 或 `1069547520`。

**排除方法**：
1. 用 Wireshark 抓取 Modbus 封包，觀察原始暫存器值
2. 查閱設備手冊中的「Byte Order」或「Word Order」章節
3. 如果手冊沒寫，用已知值（例如讓設備輸出固定值）逐一嘗試四種組合

### 1.2 位址偏移（0-based vs 1-based）

Modbus 協定規範中，暫存器位址是從 0 開始的。但許多設備手冊和組態軟體使用 1-based 位址。這導致了持續數十年的混亂。

```python
# 設備手冊寫「保持暫存器 40001」
# 在 Modbus 協定中，這其實是位址 0（功能碼 0x03）
# 因為 40001 = 4xxxx 基底 + 位址 1，而位址 1 在協定中是 0

# CSP Library 使用 address_offset 處理這個問題
config = DeviceConfig(
    device_id="pcs_01",
    unit_id=1,
    address_offset=0,  # 預設值：位址直接使用（0-based）
    # address_offset=-1,  # 如果手冊用 1-based 位址，設 -1
)
```

**經驗法則**：
- 如果手冊寫位址 `0x0000`，通常是 0-based，`address_offset=0`
- 如果手冊寫位址 `40001` 或 `1`，可能是 1-based，嘗試 `address_offset=-1`
- 不確定的話，讀取一個已知值的暫存器，看偏移多少

### 1.3 Unit ID 與閘道器

在點對點的 Modbus TCP 連線中，Unit ID 通常設為 1 或 0 就能運作。但當你的連線經過 Modbus 閘道器（Gateway）時，Unit ID 就變得至關重要——它決定了閘道器要將請求轉發到哪個下游設備。

```python
# 直連設備：Unit ID 通常是 1
config_direct = DeviceConfig(device_id="pcs_01", unit_id=1, ...)

# 透過閘道器連接多台設備：每台設備有不同的 Unit ID
config_pcs = DeviceConfig(device_id="pcs_01", unit_id=1, ...)
config_bms = DeviceConfig(device_id="bms_01", unit_id=2, ...)
config_meter = DeviceConfig(device_id="meter_01", unit_id=3, ...)
# 以上三台設備共用同一個閘道器 IP，但 unit_id 不同
```

**常見錯誤**：把所有設備的 Unit ID 都設為 1，導致閘道器只回應其中一台設備的資料。

### 1.4 首次讀取超時

很多工業設備在上電或通訊初始化後，需要一段時間才能回應 Modbus 請求。如果你的程式在連線後立即發送讀取命令，很可能會收到超時錯誤。

```python
# 錯誤做法：連線後立刻讀取
async with AsyncModbusDevice(config) as device:
    value = await device.read("voltage")  # 可能超時！

# 正確做法：利用 CSP Library 的重試機制，或加入啟動延遲
async with AsyncModbusDevice(config) as device:
    await asyncio.sleep(1)  # 給設備一點反應時間
    value = await device.read("voltage")
```

CSP Library 內建的 `CircuitBreaker` 和 `RetryPolicy` 可以優雅地處理這種暫時性失敗，不需要手動加 `sleep`。但你需要知道這個現象存在，才能正確設定重試參數。

### 1.5 通訊頻率過高

Modbus 設備（尤其是 RTU 模式下的 PLC）有處理速度上限。如果你以過高的頻率輪詢，會導致設備回應變慢、甚至完全無回應。

**建議**：
- Modbus TCP 設備：輪詢間隔不低於 100ms
- Modbus RTU 設備：輪詢間隔不低於 200ms
- 多設備共用串口：考慮使用 CSP Library 的 `SharedClient` 進行排程

---

## 2. Async 非同步程式設計陷阱

### 2.1 忘記 await

這是 Python async 開發中最經典的錯誤。忘記 `await` 不會產生語法錯誤，只會得到一個 coroutine 物件而不是結果。

```python
# 錯誤：忘記 await，result 是 coroutine 物件
result = device.write("p_set", 50.0)
print(result)  # <coroutine object AsyncModbusDevice.write at 0x...>

# 正確：加上 await
result = await device.write("p_set", 50.0)
print(result)  # WriteResult(success=True, ...)
```

**症狀**：程式沒有報錯，但操作沒有實際執行。Python 會在垃圾回收時發出 `RuntimeWarning: coroutine was never awaited` 警告，但這個警告很容易被忽略。

**預防**：啟用 `mypy` 型別檢查，它能偵測到未 await 的 coroutine 賦值。

### 2.2 在 async 中執行阻塞操作

Event loop 是單執行緒的。如果你在 async 函式中執行 CPU 密集或阻塞 I/O 的操作，整個 event loop 會被凍結，所有其他的非同步任務都會停滯。

```python
# 錯誤：在 async 函式中做 CPU 密集運算
async def process_data(data: list[float]) -> float:
    # 這會阻塞 event loop！
    return sum(x ** 2 for x in data)  # 如果 data 有百萬筆...

# 正確：將 CPU 密集工作交給執行緒池
import asyncio

async def process_data(data: list[float]) -> float:
    loop = asyncio.get_running_loop()
    result = await loop.run_in_executor(None, _compute, data)
    return result

def _compute(data: list[float]) -> float:
    return sum(x ** 2 for x in data)
```

**常見的阻塞操作**：
- `time.sleep()`（應使用 `asyncio.sleep()`）
- 同步的檔案 I/O（`open()` / `read()` / `write()`）
- 同步的 HTTP 請求（`requests.get()`，應使用 `httpx.AsyncClient`）
- 大量的數值計算

### 2.3 資源清理遺漏

CSP Library 的核心元件都實作了 `AsyncLifecycleMixin`，必須正確地啟動和停止。如果忘記清理，可能導致連線洩漏、背景任務殘留。

```python
# 錯誤：沒有清理資源
device = AsyncModbusDevice(config)
await device.start()
data = await device.read("voltage")
# 忘記 await device.stop() -> 連線洩漏！

# 正確：使用 async context manager
async with AsyncModbusDevice(config) as device:
    data = await device.read("voltage")
# 離開 with 區塊時自動呼叫 stop()

# 也正確：手動管理，但要確保 stop 被呼叫
device = AsyncModbusDevice(config)
try:
    await device.start()
    data = await device.read("voltage")
finally:
    await device.stop()
```

**原則**：凡是繼承 `AsyncLifecycleMixin` 的類別（`AsyncModbusDevice`、`DeviceManager`、`SystemController` 等），一律使用 `async with` 語法。

### 2.4 並行 vs 循序的選擇錯誤

需要同時讀取多台設備時，逐一 await 會浪費大量等待時間：

```python
# 慢：循序讀取三台設備，總耗時 = t1 + t2 + t3
v1 = await device1.read("voltage")
v2 = await device2.read("voltage")
v3 = await device3.read("voltage")

# 快：並行讀取，總耗時 = max(t1, t2, t3)
v1, v2, v3 = await asyncio.gather(
    device1.read("voltage"),
    device2.read("voltage"),
    device3.read("voltage"),
)
```

但要注意：如果多台設備共用同一個 Modbus 連線（例如透過 `SharedClient`），則不能真正並行，因為 Modbus 是請求-回應協定。CSP Library 的 `ReadScheduler` 已經考慮了這一點。

---

## 3. 控制策略陷阱

### 3.1 未檢查 Context 中的 None 值

策略的 `execute()` 方法接收的 `context` 物件中，某些欄位可能是 `None`（例如設備離線時）。直接運算會導致 `TypeError`。

```python
# 錯誤：直接使用可能為 None 的值
class MyStrategy:
    async def execute(self, context) -> StrategyResult:
        # 如果 context.extra["grid_voltage"] 是 None，這裡會爆炸
        power = context.extra["grid_voltage"] * context.extra["grid_current"]
        return StrategyResult(commands=[...])

# 正確：先檢查 None
class MyStrategy:
    async def execute(self, context) -> StrategyResult:
        voltage = context.extra.get("grid_voltage")
        current = context.extra.get("grid_current")

        if voltage is None or current is None:
            # 資料不完整，回傳空命令或保持現狀
            return StrategyResult(commands=[], reason="incomplete data")

        power = voltage * current
        return StrategyResult(commands=[...])
```

### 3.2 策略中的無窮迴圈

`execute()` 方法應該是「計算一次，回傳結果」。如果你在裡面寫了 `while True` 等待某個條件成立，會導致整個策略執行引擎卡住。

```python
# 錯誤：在 execute 中等待
class BadStrategy:
    async def execute(self, context) -> StrategyResult:
        while context.extra.get("soc") < 90:
            await asyncio.sleep(1)  # 永遠不會更新！
        return StrategyResult(commands=[...])

# 正確：根據當前狀態做決策，由外部排程器負責重複呼叫
class GoodStrategy:
    async def execute(self, context) -> StrategyResult:
        soc = context.extra.get("soc", 0)
        if soc < 90:
            # 繼續充電
            return StrategyResult(commands=[SetPowerCommand("pcs_01", 100.0)])
        else:
            # 充飽了，停止
            return StrategyResult(commands=[SetPowerCommand("pcs_01", 0.0)])
```

### 3.3 忽略 system_base 為 None 的情況

在某些運行模式下，`context.system_base` 可能為 `None`（例如系統剛啟動、尚未取得電網基準值時）。如果你的策略計算依賴 `system_base`，必須處理這個邊界情況。

```python
class PQStrategy:
    async def execute(self, context) -> StrategyResult:
        if context.system_base is None:
            return StrategyResult(
                commands=[],
                reason="system_base not available yet"
            )
        # 正常計算...
```

---

## 4. 測試陷阱

### 4.1 缺少 @pytest.mark.asyncio

CSP Library 的測試沒有配置全域 asyncio mode。每個非同步測試函式都需要加上裝飾器：

```python
import pytest

# 錯誤：缺少裝飾器，測試不會被正確執行
async def test_device_read():
    ...

# 正確：加上 @pytest.mark.asyncio
@pytest.mark.asyncio
async def test_device_read():
    async with AsyncModbusDevice(config) as device:
        result = await device.read("voltage")
        assert result is not None
```

**症狀**：測試顯示 passed 但實際上什麼都沒執行（因為 pytest 把它當成普通函式，回傳了一個 coroutine 物件，而 coroutine 物件是 truthy）。

### 4.2 Mock 設備的 PropertyMock 陷阱

當你 mock `AsyncModbusDevice` 的屬性時，需要使用 `PropertyMock`：

```python
from unittest.mock import MagicMock, PropertyMock, AsyncMock

# 錯誤：直接設定 attribute，可能不會觸發 property getter
mock_device = MagicMock()
mock_device.device_id = "pcs_01"  # 這會覆蓋 property descriptor

# 正確：使用 PropertyMock
mock_device = MagicMock(spec=AsyncModbusDevice)
type(mock_device).device_id = PropertyMock(return_value="pcs_01")
mock_device.read = AsyncMock(return_value=100.0)
mock_device.write = AsyncMock(return_value=True)
```

### 4.3 測試之間的狀態汙染

如果多個測試共用 mock 物件或全域狀態，一個測試的副作用可能影響另一個測試的結果：

```python
# 不好：模組層級的共用物件
shared_device = MagicMock()

def test_read():
    shared_device.read.return_value = 100
    ...

def test_write():
    # shared_device.read.return_value 還是 100！
    ...

# 好：使用 pytest fixture 確保隔離
@pytest.fixture
def mock_device():
    device = MagicMock(spec=AsyncModbusDevice)
    device.read = AsyncMock(return_value=0)
    device.write = AsyncMock(return_value=True)
    return device

def test_read(mock_device):
    mock_device.read.return_value = 100
    ...

def test_write(mock_device):
    # 全新的 mock_device，不受上一個測試影響
    ...
```

---

## 5. 部署陷阱

### 5.1 防火牆阻擋 Modbus 連接埠

Modbus TCP 預設使用 port 502。在企業網路環境中，這個連接埠通常被防火牆阻擋。

```bash
# 檢查 port 是否通暢
# Linux
nc -zv 192.168.1.100 502

# Windows
Test-NetConnection -ComputerName 192.168.1.100 -Port 502
```

**解決方式**：聯絡網路管理員開通 port 502，或使用非標準連接埠（需要在設備端和程式端同時修改）。

### 5.2 SELinux 阻止串口存取

在 RHEL/CentOS 系統上使用 Modbus RTU（串口通訊）時，SELinux 可能會阻止 Python 存取 `/dev/ttyS*` 或 `/dev/ttyUSB*`：

```bash
# 查看 SELinux 是否阻擋
sudo ausearch -m AVC -ts recent | grep python

# 臨時解決（不建議用於生產）
sudo setenforce 0

# 永久解決：建立 SELinux policy module
sudo setsebool -P domain_can_mmap_files 1
# 或建立自訂 policy
```

同時確保使用者有串口的存取權限：

```bash
sudo usermod -a -G dialout $USER
```

### 5.3 時間同步問題

工業控制系統中，控制器和設備之間的時間不同步會導致：
- 資料時間戳不一致，分析困難
- 排程策略在錯誤的時間點執行
- 告警記錄的時序混亂

```bash
# 確保所有節點使用 NTP 同步時間
sudo systemctl enable chronyd
sudo systemctl start chronyd

# 檢查時間偏差
chronyc tracking
```

### 5.4 生產環境缺少錯誤處理

開發階段可能讓例外直接往上拋，但生產環境中，未處理的例外會導致整個系統停止運作。

```python
# 危險：生產環境中不處理例外
async def main():
    async with SystemController(config) as ctrl:
        await ctrl.run()  # 一個例外就讓整個系統掛掉

# 安全：加入錯誤處理和重啟邏輯
async def main():
    while True:
        try:
            async with SystemController(config) as ctrl:
                await ctrl.run()
        except KeyboardInterrupt:
            logger.info("收到停止信號，正常關閉")
            break
        except CommunicationError as e:
            logger.warning(f"通訊中斷，10 秒後重試: {e}")
            await asyncio.sleep(10)
        except Exception as e:
            logger.error(f"非預期錯誤，30 秒後重試: {e}")
            await asyncio.sleep(30)
```

---

## 6. 效能陷阱

### 6.1 過多的輪轉點位群組

CSP Library 的 `ReadScheduler` 支援將點位分組輪轉讀取。但如果群組太多、每組太小，協定開銷（每次 Modbus 請求的表頭和延遲）會大幅降低效率。

**建議**：
- 盡量合併相鄰位址的點位到同一群組
- 每個群組包含 10-50 個暫存器
- 輪轉群組數量控制在 3-5 個以內

### 6.2 MongoDB 寫入放大

如果每次讀取設備資料都立即寫入 MongoDB，會產生大量的小型寫入操作，嚴重影響 MongoDB 和網路效能。

```python
# 不好：每次讀取都寫入
async def on_data_update(device_id, data):
    await mongo_collection.insert_one({"device": device_id, **data})

# 好：使用 CSP Library 的 DataUploadManager 進行批次上傳
# DataUploadManager 會在記憶體中累積資料，定期批次寫入
async with DataUploadManager(config) as uploader:
    uploader.submit(device_id, data)
    # 內部會自動批次處理
```

### 6.3 Redis 記憶體持續增長

使用 Redis Streams 儲存設備資料時，如果不設定 MAXLEN，Stream 會無限增長直到記憶體耗盡。

```python
# 不好：Stream 無限增長
await redis.xadd("device:pcs_01:data", data)

# 好：限制 Stream 長度
await redis.xadd("device:pcs_01:data", data, maxlen=10000)
# 或使用近似裁剪（效能更好）
await redis.xadd("device:pcs_01:data", data, maxlen=10000, approximate=True)
```

**建議**：在 Redis 中設定 `maxmemory` 和 `maxmemory-policy`，防止記憶體溢出。

---

## 7. 除錯工具與技巧

### 7.1 Loguru 模組層級日誌

CSP Library 使用 loguru 並支援按模組設定日誌等級，這在除錯時非常有用：

```python
from csp_lib.core import set_level, configure_logging

# 初始化日誌
configure_logging(level="INFO")

# 只對 Modbus 通訊開啟 DEBUG 日誌
set_level("DEBUG", module="csp_lib.modbus")

# 只對設備層開啟 DEBUG
set_level("DEBUG", module="csp_lib.equipment")

# 其他模組維持 INFO 等級，避免日誌洪水
```

### 7.2 Wireshark 抓取 Modbus 封包

Wireshark 內建 Modbus 協定解析器，是排查通訊問題的利器：

1. 在 Wireshark 中設定擷取過濾器：`tcp port 502`
2. 開始擷取後，執行你的程式
3. 在顯示過濾器輸入 `modbus` 或 `mbtcp`
4. 檢查每個請求和回應的功能碼、位址、資料

**重點檢查項目**：
- 請求的功能碼是否正確（0x03 = Read Holding Registers）
- 起始位址是否正確（注意 0-based vs 1-based）
- 回應的例外碼（如 0x02 = Illegal Data Address）

### 7.3 健康檢查端點

CSP Library 的 `HealthCheckable` protocol 讓你可以查詢各元件的健康狀態：

```python
from csp_lib.core import HealthStatus

# 檢查單一設備
report = await device.health_check()
print(report.status)     # HealthStatus.HEALTHY / DEGRADED / UNHEALTHY
print(report.details)    # 詳細資訊

# 在 GUI 模組中，健康狀態會自動暴露為 REST API 端點
# GET /api/health -> 所有元件的健康狀態摘要
```

### 7.4 常用除錯指令速查

```bash
# 執行單一測試並顯示完整輸出
uv run pytest tests/modbus/test_client.py -v -s

# 執行測試到第一個失敗就停止
uv run pytest tests/ -x

# 顯示最慢的 10 個測試
uv run pytest tests/ --durations=10

# 檢查型別錯誤
uv run mypy csp_lib/modbus/ --show-error-codes

# 查看某個函式被哪些地方引用
# （在專案根目錄執行）
grep -rn "from csp_lib.core import.*CircuitBreaker" csp_lib/
```

---

## 快速查找表

| 症狀 | 可能原因 | 排查步驟 |
|------|---------|----------|
| 讀到的數值不合理 | 位元組順序錯誤 | 用 Wireshark 檢查原始資料 |
| 讀取都回傳 None | 位址偏移錯誤 | 嘗試 address_offset=-1 |
| 連線成功但讀取超時 | Unit ID 錯誤 | 確認閘道器上的設備 ID 對映 |
| coroutine object 而非值 | 忘記 await | 啟用 mypy 型別檢查 |
| Event loop 卡住 | 阻塞操作在 async 中 | 用 `run_in_executor` 包裝 |
| 測試通過但行為不對 | 缺少 @pytest.mark.asyncio | 確認所有 async test 都有裝飾器 |
| 生產環境記憶體增長 | Redis Stream 無 MAXLEN | 設定 maxlen 參數 |
| 連線被拒 | 防火牆阻擋 port 502 | 用 `nc` 或 `Test-NetConnection` 測試 |
| 策略不執行 | context 值為 None | 在 execute() 中加入 None 檢查 |

---

**系列導航**

- [附錄 A：開發環境建置指南](appendix-a-dev-environment.md)
- [附錄 C：延伸學習資源](appendix-c-resources.md)
