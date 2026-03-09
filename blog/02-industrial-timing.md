# 工業場域的時間觀：為什麼你的 async/await 技能在這裡更重要

> **從零打造工業控制協定框架：給軟體工程師的 CSP Library 實戰系列**
>
> Part 1 — 觀念轉換篇 | Article 02
>
> [<<< 上一篇：從 REST API 到 Modbus](./01-rest-to-modbus.md) | 下一篇：（即將發布）

---

## 目錄

1. [時間為什麼在工業系統中如此重要？](#時間為什麼在工業系統中如此重要)
2. [三種即時性分類](#三種即時性分類)
3. [csp_lib 的時間管理策略](#csp_lib-的時間管理策略)
4. [AsyncModbusDevice 的讀取迴圈](#asyncmodbusdevice-的讀取迴圈)
5. [asyncio 的優勢：為什麼 Python 非同步適合工業軟即時](#asyncio-的優勢為什麼-python-非同步適合工業軟即時)
6. [持久連線管理：不只是保持連線那麼簡單](#持久連線管理不只是保持連線那麼簡單)
7. [時間感知的錯誤處理](#時間感知的錯誤處理)
8. [重點回顧](#重點回顧)
9. [下篇預告](#下篇預告)

---

## 時間為什麼在工業系統中如此重要？

在上一篇文章中，我們比較了 REST API 與 Modbus 的差異，理解了暫存器模型和資料型態。但有一個面向我們只輕輕帶過——**時間**。

對後端工程師來說，「效能」通常意味著「回應速度」和「吞吐量」。你的 API 回應時間從 100ms 變成 300ms，使用者可能完全無感。你的資料庫查詢從 50ms 變成 200ms，前端可以加個 loading spinner 撐過去。在 Web 的世界裡，「慢一點」通常只是體驗問題。

但在工業場域，時間有完全不同的含義。

想像一個儲能系統的場景：電網頻率從 60.00 Hz 突然下降到 59.85 Hz。這代表發電量不足、負載過重。儲能系統的 EMS（能源管理系統）必須在**幾百毫秒內**偵測到這個變化，計算出需要放電的功率，然後下達指令給 PCS（功率調節系統）。如果這個反應延遲了 5 秒，電網可能已經觸發了低頻卸載（UFR），部分用戶被斷電了。

再想像另一個場景：BMS（電池管理系統）回報某個電池模組的溫度異常升高。你的系統必須在溫度超過安全閾值**之前**偵測到趨勢、發出告警、甚至自動降低充放電功率。如果你的讀取週期是 10 秒一次，而溫度在 3 秒內就飆升到了危險值，你的系統就成了一個昂貴的事後諸葛亮。

**在工業系統中，時間不是效能指標，而是安全邊界。**

---

## 三種即時性分類

工業控制系統按照時間要求，可以分為三個等級：

### 硬即時（Hard Real-Time）：< 1ms

| 特性 | 說明 |
|------|------|
| 典型延遲 | 微秒到毫秒等級 |
| 代表系統 | PLC（可程式邏輯控制器）、馬達驅動器 |
| 程式語言 | IEC 61131-3（梯形圖、ST）、C/C++ |
| 錯過期限的後果 | 設備損壞、人身安全風險 |

這是 Python 完全不適合的領域。硬即時要求每個控制迴圈**必須**在固定時間內完成，沒有例外。Python 的垃圾回收器、GIL、甚至作業系統的排程器，都會引入不可預測的延遲。

一個典型的 PLC 掃描週期是 1-10ms。在這個時間內，它要讀取所有輸入、執行邏輯運算、更新所有輸出。這就像一個心跳，必須穩定、精確、永不停止。

**結論：不要用 Python 做硬即時控制。交給 PLC。**

### 軟即時（Soft Real-Time）：100ms - 1s

| 特性 | 說明 |
|------|------|
| 典型延遲 | 100 毫秒到 1 秒 |
| 代表系統 | SCADA、EMS、DCS 上位機 |
| 程式語言 | Python、Java、C# |
| 錯過期限的後果 | 效能降級、控制精度下降，但不會立即危險 |

**這是 csp_lib 的主戰場。**

軟即時意味著系統「應該」在期限內完成，但偶爾的延遲是可以容忍的。一個 EMS 的控制迴圈可能要求每秒執行一次，但偶爾一次迴圈花了 1.5 秒，系統不會崩潰——只是這次控制精度差了一點。

Python 的 asyncio 非常適合這個等級。它的事件迴圈延遲通常在微秒級，完全滿足 100ms-1s 的精度要求。而 Python 的開發效率、豐富的生態系統（資料庫連接、HTTP 客戶端、機器學習），讓它在 SCADA/EMS 層級有巨大的優勢。

### 近即時（Near Real-Time）：1s - 10s

| 特性 | 說明 |
|------|------|
| 典型延遲 | 1 秒到 10 秒 |
| 代表系統 | 監控面板、資料分析、報表系統 |
| 程式語言 | 任何語言 |
| 錯過期限的後果 | 資料略過時，顯示稍舊的數據 |

這是傳統後端工程師最熟悉的領域。定時從資料庫查詢最新數據、WebSocket 推播到前端、每隔幾秒更新圖表——你可能已經做過類似的事。

csp_lib 的 Layer 7（MongoDB、Redis）和 Layer 8（監控、GUI）就是服務這個等級的需求。

---

## csp_lib 的時間管理策略

理解了三種即時性分類後，讓我們看看 csp_lib 如何在軟即時的框架內管理時間。

### DeviceConfig：時間參數的中心

每個設備的時間行為都由 `DeviceConfig` 控制：

```python
from csp_lib.equipment.device import DeviceConfig

config = DeviceConfig(
    device_id="pcs_01",         # 設備唯一識別碼
    unit_id=1,                  # Modbus 站號
    read_interval=1.0,          # 每 1 秒讀取一次
    reconnect_interval=5.0,     # 斷線後每 5 秒嘗試重連
    disconnect_threshold=5,     # 連續 5 次讀取失敗視為斷線
    max_concurrent_reads=1,     # 最大並行讀取數
)
```

這裡有三個時間相關的參數，每個都對應一種故障場景：

| 參數 | 預設值 | 用途 | 對應故障場景 |
|------|--------|------|-------------|
| `read_interval` | 1.0s | 兩次讀取之間的間隔 | 控制讀取頻率，影響資料新鮮度 |
| `reconnect_interval` | 5.0s | 斷線後嘗試重連的間隔 | 避免狂打已故障的設備 |
| `disconnect_threshold` | 5 | 連續失敗幾次後視為斷線 | 區分「偶爾超時」和「真的斷了」 |

`read_interval` 的設定需要仔細思考。設得太短（比如 0.1 秒），Modbus 匯流排可能負荷過重，尤其是 RS-485 這種半雙工的物理介面。設得太長（比如 10 秒），關鍵數據可能過時。一般來說：

- 功率、頻率等需要快速反應的數據：0.5 - 1.0 秒
- SOC、溫度等變化較慢的數據：2 - 5 秒
- 設備資訊、序號等靜態數據：按需讀取即可

但等等——如果一台設備有 200 個點位，每秒讀取一次，每次讀取需要 50ms，那光是讀完所有點位就要 10 秒。這明顯不符合 1 秒的讀取間隔。

這就是 `ReadScheduler` 要解決的問題。

### ReadScheduler：分組輪替的排程藝術

`ReadScheduler` 是 csp_lib 解決「點位太多、時間太少」問題的核心元件。它的策略很直觀：

- **固定讀取（always_groups）**：每個週期都讀的關鍵點位
- **輪替讀取（rotating_groups）**：分批輪流讀的次要點位

```python
from csp_lib.equipment.transport import ReadScheduler, PointGrouper

grouper = PointGrouper()

scheduler = ReadScheduler(
    # 每次都讀：關鍵即時數據
    always_groups=grouper.group([
        active_power,   # 有功功率（控制必需）
        reactive_power, # 無功功率（控制必需）
        soc,            # 電池電量（控制必需）
        device_status,  # 運行狀態（安全必需）
    ]),

    # 輪替讀取：BMS 各模組的詳細資料
    rotating_groups=[
        grouper.group(sbms1_points),  # 第 1 組電池模組（16 個點位）
        grouper.group(sbms2_points),  # 第 2 組電池模組（16 個點位）
        grouper.group(sbms3_points),  # 第 3 組電池模組（16 個點位）
    ],
)
```

排程行為如下：

```
週期 1: always_groups + sbms1_points     ← 讀取 4 + 16 = 20 個點位
週期 2: always_groups + sbms2_points     ← 讀取 4 + 16 = 20 個點位
週期 3: always_groups + sbms3_points     ← 讀取 4 + 16 = 20 個點位
週期 4: always_groups + sbms1_points     ← 回到第 1 組（循環）
...
```

讓我們看看 `ReadScheduler` 的核心實作：

```python
# 來自 csp_lib/equipment/transport/scheduler.py

class ReadScheduler:
    """讀取排程器"""

    def __init__(
        self,
        always_groups: Sequence[ReadGroup] | None = None,
        rotating_groups: Sequence[Sequence[ReadGroup]] | None = None,
    ):
        self._always_groups: list[ReadGroup] = list(always_groups) if always_groups else []
        self._rotating_groups: list[list[ReadGroup]] = [
            list(g) for g in rotating_groups
        ] if rotating_groups else []
        self._rotating_index = 0

    def get_next_groups(self) -> list[ReadGroup]:
        """取得下一批要讀取的分組"""
        groups: list[ReadGroup] = list(self._always_groups)

        if self._rotating_groups:
            groups.extend(self._rotating_groups[self._rotating_index])
            self._rotating_index = (self._rotating_index + 1) % len(self._rotating_groups)

        return groups
```

設計上幾個值得注意的地方：

1. **`_rotating_index` 使用模數運算自動循環**：`(index + 1) % len(groups)` 確保永遠不會越界。簡單但防錯。
2. **`get_next_groups()` 有副作用（推進索引）**：如果你只想查看下一批而不推進，可以用 `peek_next_groups()`。
3. **`update_groups()` 支援動態更新**：設備在運行中可能需要增減點位，不用停機就能調整排程。

用一個後端的類比：`ReadScheduler` 就像是一個資料庫查詢排程器。你的核心業務查詢（如用戶認證）每次都執行，但統計報表查詢則分散到不同時間窗口，避免同時打滿資料庫連線。

---

## AsyncModbusDevice 的讀取迴圈

`ReadScheduler` 只負責「決定讀什麼」，真正的「讀取迴圈」在 `AsyncModbusDevice` 中。讓我們看看它的 `_read_loop` 方法如何協調時間：

```python
# 來自 csp_lib/equipment/device/base.py（簡化版）

class AsyncModbusDevice:

    async def _read_loop(self) -> None:
        """讀取循環（含自動重連）"""
        interval = self._config.read_interval
        reconnect_interval = self._config.reconnect_interval

        while not self._stop_event.is_set():
            start_time = time.monotonic()

            # 未連線時嘗試重連
            if not self._client_connected:
                try:
                    await self._client.connect()
                    self._client_connected = True
                    self._device_responsive = True
                    self._consecutive_failures = 0
                    await self._emitter.emit_await(
                        EVENT_CONNECTED,
                        ConnectedPayload(device_id=self._config.device_id),
                    )
                except Exception:
                    await asyncio.sleep(reconnect_interval)
                    continue

            try:
                await self.read_once()
            except Exception:
                pass  # read_once 已處理錯誤事件

            elapsed = time.monotonic() - start_time
            sleep_time = max(0, interval - elapsed)
            await asyncio.sleep(sleep_time)
```

這段程式碼有幾個精妙的時間控制技巧，值得後端工程師仔細學習：

### 技巧 1：動態調整睡眠時間

```python
elapsed = time.monotonic() - start_time
sleep_time = max(0, interval - elapsed)
await asyncio.sleep(sleep_time)
```

如果 `read_interval` 是 1.0 秒，而這次讀取花了 0.3 秒，那實際睡眠時間是 `1.0 - 0.3 = 0.7` 秒。這確保了**兩次讀取的起始時間間隔**儘可能接近 1.0 秒，而不是「讀完後再等 1.0 秒」。

如果讀取花了 1.2 秒（超過了 `read_interval`），`max(0, ...)` 確保不會出現負數的睡眠時間——直接進入下一次讀取。

這是後端定時任務中常見的「drift correction」技巧。你可能在寫 cron job 或 Celery beat 時沒太在意這件事，因為幾秒的偏移對 Web 應用無所謂。但在工業場域，讀取間隔的穩定性直接影響控制品質。

### 技巧 2：使用 time.monotonic() 而非 time.time()

`time.monotonic()` 是**單調遞增**的時鐘，不受系統時間調整影響。如果你用 `time.time()`，NTP 同步或手動校時可能導致計時錯誤——想像時鐘被往回撥了 2 秒，你的迴圈會突然多等 2 秒。

在後端開發中，你可能很少意識到這個差異。但在一個需要連續運行數月的工業系統中，`monotonic` 是唯一正確的選擇。

### 技巧 3：斷線時切換到重連間隔

```python
if not self._client_connected:
    try:
        await self._client.connect()
        # ...
    except Exception:
        await asyncio.sleep(reconnect_interval)  # 5 秒
        continue
```

設備斷線時，迴圈不再以 `read_interval`（1 秒）的頻率狂打連線請求，而是降級到 `reconnect_interval`（5 秒）。這避免了在設備故障期間產生大量無意義的連線嘗試。

類比到後端：這就像是 circuit breaker pattern（斷路器模式）。當下游服務不可用時，你不會繼續以原始頻率重試，而是拉長間隔，給對方恢復的時間。

---

## asyncio 的優勢：為什麼 Python 非同步適合工業軟即時

你可能會問：「用 Python 寫工業控制，不會太慢嗎？」

答案是：對於軟即時場景，asyncio 不只「夠快」，而且有獨特的優勢。

### 優勢 1：單執行緒事件迴圈 = 確定性的執行順序

asyncio 使用單執行緒的事件迴圈。這意味著**不存在競態條件（race condition）**。在多執行緒的環境中，兩個執行緒可能同時修改同一個設備的狀態，造成資料不一致。在 asyncio 中，這不可能發生——同一時間只有一個 coroutine 在執行。

```python
# asyncio：不需要 lock
class AsyncModbusDevice:
    async def _process_values(self, values: dict[str, Any]) -> None:
        """處理讀取到的值，發送變更事件"""
        for name, new_value in values.items():
            if name in self._disabled_points:
                continue
            old_value = self._latest_values.get(name)
            if old_value != new_value:
                self._emitter.emit(
                    EVENT_VALUE_CHANGE,
                    ValueChangePayload(
                        device_id=self._config.device_id,
                        point_name=name,
                        old_value=old_value,
                        new_value=new_value,
                    ),
                )
            self._latest_values[name] = new_value
```

注意這段程式碼沒有任何 `threading.Lock`。在多執行緒環境中，你需要用鎖保護 `_latest_values` 的讀寫。但在 asyncio 中，`_process_values` 不會被其他 coroutine 中斷（除非遇到 `await`），所以整個更新操作天然就是原子的。

對工業系統來說，確定性比性能更重要。你寧可系統慢一點但行為可預測，也不要快但偶爾出錯。

### 優勢 2：I/O 等待不浪費 CPU

Modbus 通訊是典型的 I/O 密集型操作。你發送一個讀取請求，然後等待設備回應。在這個等待期間（通常 5-50ms），CPU 完全空閒。

asyncio 的 `await` 讓你在等待 I/O 的同時，去處理其他事情：

```python
# 虛擬碼：展示 asyncio 的並行效果
async def manage_multiple_devices():
    """同時管理多台設備"""

    # 在後端世界，你可能這樣做：
    # 方法 A：用多執行緒（每台設備一個執行緒）
    # 方法 B：用 asyncio（更優雅）

    device_1 = AsyncModbusDevice(config_1, client_1, always_points=points_1)
    device_2 = AsyncModbusDevice(config_2, client_2, always_points=points_2)
    device_3 = AsyncModbusDevice(config_3, client_3, always_points=points_3)

    # 每台設備啟動自己的讀取迴圈
    # 在等待設備 1 回應的時候，可以處理設備 2 的請求
    async with device_1, device_2, device_3:
        # 三台設備同時運行
        await asyncio.sleep(3600)  # 運行 1 小時
```

如果用多執行緒管理 50 台設備，你會有 50 個執行緒，每個都在大部分時間裡阻塞在 I/O 等待上。GIL 的存在讓這些執行緒無法真正平行化 CPU 運算。

用 asyncio，50 台設備只需要 1 個執行緒。事件迴圈在各設備的 I/O 等待之間高效切換，CPU 利用率接近 100%（在有工作要做的前提下）。

### 優勢 3：asyncio.sleep() 的精確度

你可能擔心 `asyncio.sleep()` 的精確度。實測結果是：在現代作業系統上，`asyncio.sleep(1.0)` 的實際偏差通常在 1-5ms 以內。對於軟即時的 100ms-1s 精度要求來說，這完全足夠。

```python
import asyncio
import time

async def test_sleep_precision():
    """測試 asyncio.sleep 的精確度"""
    errors = []
    for _ in range(100):
        start = time.monotonic()
        await asyncio.sleep(0.1)  # 期望等待 100ms
        actual = time.monotonic() - start
        errors.append(abs(actual - 0.1) * 1000)  # 誤差轉為 ms

    print(f"平均誤差: {sum(errors)/len(errors):.2f} ms")
    print(f"最大誤差: {max(errors):.2f} ms")
    # 典型輸出：平均誤差 ~1-2ms, 最大誤差 ~5ms
```

---

## 持久連線管理：不只是保持連線那麼簡單

在上一篇中我們提到，Modbus 使用持久連線。但在實際運行中，「保持連線」遠比想像中複雜。

### TCP 獨立連線 vs 共用連線

csp_lib 提供兩種 TCP 客戶端：

```python
from csp_lib.modbus.clients import PymodbusTcpClient, SharedPymodbusTcpClient
from csp_lib.modbus.config import ModbusTcpConfig

# 獨立連線：每個設備一條 TCP 連線
# 適用場景：設備有獨立 IP（如乙太網直連的 PCS）
client_1 = PymodbusTcpClient(ModbusTcpConfig(host="192.168.1.101"))
client_2 = PymodbusTcpClient(ModbusTcpConfig(host="192.168.1.102"))

# 共用連線：多個設備共用一條 TCP 連線
# 適用場景：透過 TCP-RS485 轉換器連接多台設備
client_a = SharedPymodbusTcpClient(ModbusTcpConfig(host="192.168.1.12"))
client_b = SharedPymodbusTcpClient(ModbusTcpConfig(host="192.168.1.12"))
# client_a 和 client_b 共用同一條到 192.168.1.12 的 TCP 連線
```

為什麼需要 `SharedPymodbusTcpClient`？因為在工業現場，你經常會遇到 **TCP-RS485 轉換器**（也叫 Modbus Gateway）。這是一個小盒子，一端接乙太網，另一端接 RS-485 匯流排。匯流排上掛了多台設備（比如 10 台電表），但它們都通過同一個 IP 位址的 port 502 通訊。

如果你為每台電表建立獨立的 TCP 連線，這些連線會在轉換器內部打架——因為 RS-485 是半雙工的，同一時間只能有一個請求。`SharedPymodbusTcpClient` 使用 **Singleton per endpoint** 模式和請求佇列來序列化存取：

```python
# SharedPymodbusTcpClient 的核心概念
# 同一個 host:port 共用：
# 1. 同一個 TCP 連線
# 2. 同一個 ModbusRequestQueue（優先權排程 + 公平排程 + 斷路器）

# 使用引用計數管理生命週期
# 最後一個 disconnect() 時才真正關閉 TCP 連線
```

用後端的概念類比：`SharedPymodbusTcpClient` 就像資料庫連線池（connection pool），但它不是池化多條連線，而是讓多個消費者共用**一條**連線，並透過佇列來排程請求。

### RTU 客戶端的 Singleton per Port

RTU（串列通訊）也有類似的問題。一個串列埠（如 `/dev/ttyUSB0` 或 `COM1`）上可能掛了多台設備，它們透過不同的 `unit_id` 區分。csp_lib 的 `PymodbusRtuClient` 同樣使用 Singleton per port 模式：

```python
from csp_lib.modbus.clients import PymodbusRtuClient
from csp_lib.modbus.config import ModbusRtuConfig

# 同一個串列埠的多台設備
config_1 = ModbusRtuConfig(port="COM1", baudrate=9600)
config_2 = ModbusRtuConfig(port="COM1", baudrate=9600)

client_1 = PymodbusRtuClient(config_1)  # unit_id=1
client_2 = PymodbusRtuClient(config_2)  # unit_id=2

# 共用同一個實體串列埠連線
await client_1.connect()  # 建立串列埠連線，ref_count=1
await client_2.connect()  # 重用連線，ref_count=2

await client_1.disconnect()  # ref_count=1，連線保持
await client_2.disconnect()  # ref_count=0，關閉串列埠
```

引用計數（reference counting）確保最後一個使用者斷開時才真正關閉硬體資源。這是一個經典的資源管理模式，你在 Python 的 `__del__` 或 context manager 中可能見過類似的做法。

---

## 時間感知的錯誤處理

工業場域的錯誤處理和 Web 應用有一個根本差異：**你不能簡單地回傳一個錯誤碼就了事**。設備斷線不是「這個請求失敗了」，而是「從現在開始所有請求都會失敗，直到設備恢復為止」。

### 連續失敗計數與斷線判定

csp_lib 用連續失敗次數來區分「偶爾的通訊超時」和「設備真的斷了」：

```python
# 來自 csp_lib/equipment/device/base.py

class AsyncModbusDevice:
    def __init__(self, config, client, ...):
        # ...
        self._device_responsive = False      # 設備是否有回應
        self._consecutive_failures = 0       # 連續失敗次數
        self._last_failure_time = None       # 最後失敗時間

    def _handle_read_failure(self, error_msg: str) -> None:
        """處理讀取失敗：累加計數 + 記錄時間 + 發送事件"""
        self._consecutive_failures += 1
        self._last_failure_time = time.monotonic()
        self._emitter.emit(
            EVENT_READ_ERROR,
            ReadErrorPayload(
                device_id=self._config.device_id,
                error=error_msg,
                consecutive_failures=self._consecutive_failures,
            ),
        )

    async def _check_disconnect_threshold(self, error_msg: str) -> None:
        """達到斷線閾值時標記設備無回應"""
        if (self._consecutive_failures >= self._config.disconnect_threshold
                and self._device_responsive):
            self._device_responsive = False
            await self._emitter.emit_await(
                EVENT_DISCONNECTED,
                DisconnectPayload(
                    device_id=self._config.device_id,
                    reason=error_msg,
                    consecutive_failures=self._consecutive_failures,
                ),
            )
```

邏輯很直觀：

1. 每次讀取失敗，`_consecutive_failures` 加 1
2. 當連續失敗次數達到 `disconnect_threshold`（預設 5 次），標記設備為「無回應」
3. 一旦讀取成功，計數歸零，設備恢復為「有回應」

為什麼不是失敗一次就判定斷線？因為工業通訊環境中，偶爾的超時是正常現象。電磁干擾、匯流排忙碌、設備暫時性的處理延遲——這些都可能導致單次讀取失敗，但設備本身是正常的。設定一個閾值（如 5 次），讓系統有足夠的容錯空間。

### should_attempt_read：避免超時拖慢整個系統

這裡有一個很巧妙的設計：`should_attempt_read` 屬性。

```python
# 來自 csp_lib/equipment/device/base.py

@property
def should_attempt_read(self) -> bool:
    """是否應嘗試讀取"""
    if self._device_responsive:
        return True
    if self._last_failure_time is None:
        return True
    return (time.monotonic() - self._last_failure_time) >= self._config.reconnect_interval
```

當一台設備已經被標記為無回應時，繼續每秒讀取它是浪費時間的——每次讀取都會等到超時（通常 3-5 秒），這段時間 Modbus 匯流排是被佔用的。

`should_attempt_read` 讓上層排程器知道：「這台設備目前無回應，在 `reconnect_interval` 時間後再試就好。」這樣其他正常的設備就不會因為一台故障設備而被拖慢。

用後端的類比：這就像是 HTTP 客戶端的 circuit breaker。當下游服務持續 502 時，你的客戶端不會繼續以每秒 100 次的頻率重試，而是「打開斷路器」，等一段冷卻時間後再「半開」探測一次。

### 錯誤層級：語意化的例外類別

csp_lib 定義了清晰的錯誤層級結構：

```python
# 來自 csp_lib/core/errors.py

class DeviceError(Exception):
    """設備層基礎例外"""
    def __init__(self, device_id: str, message: str):
        self.device_id = device_id
        super().__init__(f"[{device_id}] {message}")

class DeviceConnectionError(DeviceError):
    """連線/斷線失敗"""

class CommunicationError(DeviceError):
    """讀寫逾時/解碼錯誤"""

class AlarmError(DeviceError):
    """告警觸發"""
    def __init__(self, device_id: str, alarm_code: str, message: str):
        self.alarm_code = alarm_code
        super().__init__(device_id, message)

class ConfigurationError(Exception):
    """配置無效（非設備層級）"""
```

注意兩個設計重點：

1. **每個 DeviceError 都帶有 `device_id`**。當你管理 50 台設備時，知道是哪台出了問題至關重要。
2. **`DeviceConnectionError` 和 `CommunicationError` 分開**。前者是「連不上」（TCP 連線失敗），後者是「連上了但通訊有問題」（讀取超時、解碼錯誤）。這個區分幫助上層做出不同的處理策略：連線失敗需要重連，通訊失敗可能只需要重試。

在後端開發中，你可能習慣了 `requests.ConnectionError` 和 `requests.Timeout` 的區分。csp_lib 的錯誤層級遵循同樣的原則，但多了 `device_id` 的上下文——因為在管理多台設備的場景中，「誰出了問題」和「出了什麼問題」同樣重要。

---

## 重點回顧

1. **工業系統的時間約束分為三級**。硬即時（< 1ms）交給 PLC，軟即時（100ms-1s）是 csp_lib 的戰場，近即時（1-10s）是傳統後端的地盤。

2. **`DeviceConfig` 的三個時間參數**控制了設備的讀取節奏、重連策略和斷線判定。`read_interval`、`reconnect_interval`、`disconnect_threshold` 各自對應不同的故障場景。

3. **`ReadScheduler` 用 always + rotating 模式解決大量點位的讀取效率問題**。關鍵數據每次都讀，次要數據輪流讀，確保在有限的時間窗口內取得最重要的資訊。

4. **`AsyncModbusDevice._read_loop` 展示了精確的時間控制技巧**：動態睡眠補償、monotonic 時鐘、斷線降級。這些技巧在 Web 開發中常被忽略，但在工業場域是必備知識。

5. **asyncio 的三個優勢**讓它非常適合工業軟即時：單執行緒確定性消除了競態條件、I/O 等待時間被充分利用、`asyncio.sleep()` 的精確度滿足需求。

6. **`SharedPymodbusTcpClient` 和 `PymodbusRtuClient`** 透過 Singleton 模式和引用計數，解決了多設備共用物理連線的資源管理問題。

7. **時間感知的錯誤處理**不只是「重試」那麼簡單。連續失敗計數、斷線閾值、重連間隔、`should_attempt_read`——這些機制組合在一起，形成了一套完整的故障偵測與恢復策略。

---

## 下篇預告

前兩篇文章完成了「觀念轉換」的基礎工作。你現在知道了 Modbus 的資料模型、csp_lib 的型態系統、以及時間管理的核心策略。

在接下來的文章中，我們將進入**實戰篇**。你會學到：

- 如何用 `AsyncModbusDevice` 實際建構一個完整的設備驅動
- 事件系統（`on` / `emit`）的運作原理
- 告警管理：如何定義告警條件、遲滯處理、狀態機轉換
- 用 `async with` 優雅地管理設備的生命週期

從觀念到程式碼，從理解到動手——下一篇開始，我們寫真正的工業控制程式。

下一篇：（即將發布）

---

> 本文為「從零打造工業控制協定框架：給軟體工程師的 CSP Library 實戰系列」的第 02 篇。
> 完整系列文章請參閱系列目錄。
