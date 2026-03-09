# 沒有設備也能開發：測試驅動的協定開發

> **Part 2 — 協定轉接層 | Article 10**
> 系列：從零打造工業控制協定框架：給軟體工程師的 CSP Library 實戰系列

---

## 前言

如果你寫 Web 後端，你的「設備」就是資料庫——隨時可以啟動一個 Docker container 來測試。但如果你寫工業控制軟體，你的「設備」是一台要價數百萬的 PCS、一組佔地半個倉庫的電池櫃、或是一面掛在變電站裡的配電盤。

**你不可能在每個開發者的桌上放一台 PCS。**

這是工業控制軟體開發最大的痛點之一：你需要開發的軟體是為了控制某個設備，但開發過程中你根本沒有那個設備。等到設備到場了，你才發現程式碼有 bug，但這時候修改成本已經很高——因為你可能正在一個偏遠的案場，身邊只有一台筆電和不穩定的網路。

csp_lib 從一開始就將「可測試性」作為核心設計原則。本篇文章將完整介紹 csp_lib 的測試策略：從最輕量的 Mock 到完整的 Modbus 模擬伺服器，讓你在沒有任何實體設備的情況下，也能自信地開發和驗證程式碼。

---

## 1. 挑戰：為沒有的設備開發軟體

讓我們先梳理「沒有設備」帶來的具體挑戰：

### 1.1 開發階段的困境

| 階段 | 挑戰 | 傳統做法 |
|------|------|----------|
| 需求分析 | 只有一份 PDF 規格書和 register map | 讀文件、猜測行為 |
| 開發 | 無法驗證讀寫邏輯 | 等到現場再測 |
| 單元測試 | 無法測試 Modbus 通訊 | 跳過測試... |
| 整合測試 | 無法測試完整控制流程 | 在案場通宵 debug |
| 回歸測試 | 改了程式碼不知道有沒有影響 | 再去一次案場 |

### 1.2 工業控制的特殊測試需求

跟一般的 Web 應用相比，工業控制軟體有一些特殊的測試需求：

1. **時序敏感**：控制迴路有固定的執行週期（通常 1-5 秒），時序不對會造成問題
2. **狀態機行為**：設備有多種狀態（停機、運行、故障、保護），狀態轉換邏輯複雜
3. **邊界條件**：SOC 到 0% 或 100% 時的行為、通訊中斷時的 fallback、數值溢位等
4. **安全關鍵**：錯誤的控制命令可能造成設備損壞或安全事故

### 1.3 測試金字塔

csp_lib 的測試策略遵循經典的測試金字塔，但針對工業控制場景做了調整：

```
         ╱╲
        ╱  ╲         實機測試（案場）
       ╱    ╲        — 最終驗證，不應在此發現 bug
      ╱──────╲
     ╱        ╲       整合測試 + 模擬器
    ╱          ╲      — modbus_server 模擬完整通訊
   ╱────────────╲
  ╱              ╲     單元測試 + Mock
 ╱                ╲    — 快速、隔離、大量
╱══════════════════╲
```

---

## 2. csp_lib 的測試哲學

csp_lib 的測試策略建立在三個層次之上：

### 層次一：Mock 設備（單元測試）

使用 `unittest.mock` 建立假的設備物件，測試 Controller、Manager、Strategy 等上層邏輯。這一層不涉及任何 Modbus 通訊。

**優點**：極快（毫秒級）、無外部依賴、可精確控制所有行為
**適用**：策略邏輯、告警判斷、狀態機、邊界條件

### 層次二：Simulator Server（整合測試）

使用 `csp_lib.modbus_server` 啟動一個真正的 Modbus TCP 伺服器，搭配設備模擬器（PCS、電表、太陽能等）。讓 `AsyncModbusDevice` 連線到模擬器進行完整的讀寫測試。

**優點**：驗證完整通訊堆疊、模擬真實行為
**適用**：設備通訊、資料轉換、讀取排程

### 層次三：Transforms & Evaluators（純邏輯測試）

csp_lib 將「資料轉換」和「告警判斷」設計為純函數式的元件。它們不依賴任何 I/O，可以直接 `apply()` 或 `evaluate()` 來測試。

**優點**：最純粹的單元測試、零依賴
**適用**：ScaleTransform、BitMaskAlarmEvaluator 等

---

## 3. Mock 設備模式

csp_lib 的測試套件中大量使用 Mock 設備來測試上層邏輯。讓我們看看核心的 Mock 模式。

### 3.1 基本 Mock 工廠函式

在 csp_lib 的整合測試中，有一個 `_make_device()` 工廠函式，它是所有 Mock 設備的起點：

```python
from unittest.mock import AsyncMock, MagicMock, PropertyMock


def make_device(
    device_id: str,
    values: dict | None = None,
    responsive: bool = True,
    connected: bool = True,
    protected: bool = False,
) -> MagicMock:
    """
    建立一個 Mock 設備物件

    這個 Mock 物件模擬了 AsyncModbusDevice 的核心介面：
    - device_id: 設備識別碼
    - is_responsive: 是否回應（通訊正常）
    - is_connected: 是否已連線
    - is_protected: 是否處於保護模式
    - latest_values: 最新讀取值
    - write(): 非同步寫入方法
    - on(): 事件訂閱方法
    """
    dev = MagicMock()

    # 唯讀屬性：使用 PropertyMock 模擬
    type(dev).device_id = PropertyMock(return_value=device_id)
    type(dev).is_responsive = PropertyMock(return_value=responsive)
    type(dev).is_connected = PropertyMock(return_value=connected)
    type(dev).is_protected = PropertyMock(return_value=protected)
    type(dev).is_healthy = PropertyMock(
        return_value=connected and responsive and not protected
    )
    type(dev).latest_values = PropertyMock(return_value=values or {})
    type(dev).active_alarms = PropertyMock(return_value=[])

    # 非同步方法：使用 AsyncMock
    dev.write = AsyncMock()

    # 事件訂閱：返回取消訂閱函式
    unsub_fn = MagicMock()
    dev.on = MagicMock(return_value=unsub_fn)

    return dev
```

### 3.2 為什麼用 PropertyMock？

你可能注意到我們用 `type(dev).device_id = PropertyMock(...)` 而不是直接 `dev.device_id = "pcs_01"`。這是因為在 csp_lib 中，`device_id` 是一個 `@property`，直接賦值會覆蓋掉 property 描述器。`PropertyMock` 正確地模擬了 property 的行為。

```python
# 錯誤做法：直接賦值
dev = MagicMock()
dev.device_id = "pcs_01"  # 這會變成一個普通屬性，不是 property

# 正確做法：PropertyMock
dev = MagicMock()
type(dev).device_id = PropertyMock(return_value="pcs_01")
# 現在 dev.device_id 每次存取都會回傳 "pcs_01"
```

### 3.3 進階 Mock 場景

#### 場景 1：模擬設備值變化

```python
def make_device_with_changing_values(
    device_id: str,
    value_sequence: list[dict],
) -> MagicMock:
    """
    建立一個值會隨時間變化的 Mock 設備

    每次存取 latest_values 會回傳序列中的下一組值。
    適合測試策略在不同輸入下的行為。
    """
    dev = MagicMock()
    type(dev).device_id = PropertyMock(return_value=device_id)
    type(dev).is_responsive = PropertyMock(return_value=True)
    type(dev).is_connected = PropertyMock(return_value=True)
    type(dev).is_protected = PropertyMock(return_value=False)
    type(dev).is_healthy = PropertyMock(return_value=True)

    # side_effect 讓每次呼叫回傳不同值
    type(dev).latest_values = PropertyMock(side_effect=value_sequence)

    dev.write = AsyncMock()
    return dev


# 使用範例
device = make_device_with_changing_values(
    "pcs_01",
    value_sequence=[
        {"active_power": 0.0, "soc": 80.0},    # 第 1 次讀取
        {"active_power": 25.0, "soc": 79.5},   # 第 2 次讀取
        {"active_power": 50.0, "soc": 79.0},   # 第 3 次讀取
    ],
)

# 每次存取 latest_values 會依序回傳
print(device.latest_values)  # {"active_power": 0.0, "soc": 80.0}
print(device.latest_values)  # {"active_power": 25.0, "soc": 79.5}
print(device.latest_values)  # {"active_power": 50.0, "soc": 79.0}
```

#### 場景 2：模擬通訊失敗

```python
import pytest
from csp_lib.core.errors import CommunicationError


def make_failing_device(
    device_id: str,
    fail_on_write: bool = False,
    fail_on_read: bool = False,
) -> MagicMock:
    """
    建立一個會失敗的 Mock 設備

    用於測試錯誤處理邏輯：
    - Controller 在設備寫入失敗時的行為
    - Manager 在設備不回應時的 fallback 策略
    """
    dev = MagicMock()
    type(dev).device_id = PropertyMock(return_value=device_id)
    type(dev).is_connected = PropertyMock(return_value=True)
    type(dev).is_responsive = PropertyMock(return_value=not fail_on_read)
    type(dev).is_protected = PropertyMock(return_value=False)
    type(dev).is_healthy = PropertyMock(return_value=not fail_on_read)
    type(dev).latest_values = PropertyMock(return_value={})
    type(dev).active_alarms = PropertyMock(return_value=[])

    if fail_on_write:
        dev.write = AsyncMock(
            side_effect=CommunicationError("Write failed: device not responding")
        )
    else:
        dev.write = AsyncMock()

    return dev


# 使用範例：測試 Controller 的錯誤處理
@pytest.mark.asyncio
async def test_controller_handles_write_failure():
    device = make_failing_device("pcs_01", fail_on_write=True)

    # 嘗試寫入應該不會讓 Controller crash
    with pytest.raises(CommunicationError):
        await device.write("p_setpoint", 50.0)

    # 驗證 write 確實被呼叫了
    device.write.assert_called_once_with("p_setpoint", 50.0)
```

#### 場景 3：驗證寫入命令

```python
@pytest.mark.asyncio
async def test_strategy_sends_correct_command():
    """
    測試策略是否發出正確的控制命令

    這是最常見的 Mock 測試模式：
    1. 建立 Mock 設備，設定初始值
    2. 執行策略邏輯
    3. 驗證 write 被呼叫的參數
    """
    # Arrange
    pcs = make_device("pcs_01", values={
        "active_power": 0.0,
        "soc": 50.0,
    })

    meter = make_device("meter_01", values={
        "active_power": -30.0,  # 負值 = 從電網購電 30kW
    })

    # Act
    # 假設策略是：讓 PCS 放電來抵消購電
    target_power = -meter.latest_values["active_power"]  # 30.0 kW

    await pcs.write("p_setpoint", target_power)

    # Assert
    pcs.write.assert_called_once_with("p_setpoint", 30.0)

    # 也可以檢查呼叫次數
    assert pcs.write.call_count == 1

    # 或檢查所有呼叫的歷史
    # pcs.write.assert_any_call("p_setpoint", 30.0)
```

---

## 4. modbus_server 模擬器

Mock 適合測試上層邏輯，但它完全跳過了 Modbus 通訊堆疊。當你需要測試「設備模板定義是否正確」、「資料轉換是否能處理真實的暫存器值」時，你需要一個真正的 Modbus TCP 伺服器。

csp_lib 的 `modbus_server` 模組提供了完整的模擬基礎設施。

### 4.1 SimulatorDataBlock 概念

`SimulatorDataBlock` 是 `modbus_server` 的核心橋接元件。它連接了 pymodbus 的 DataStore 和 csp_lib 的 `RegisterBlock`：

```python
# SimulatorDataBlock 的角色
#
# pymodbus TCP Server                  csp_lib Simulator
# ┌─────────────────┐                  ┌────────────────┐
# │ ModbusTcpServer  │                  │ PCSSimulator   │
# │                  │                  │                │
# │  ┌────────────┐  │  getValues()     │ RegisterBlock  │
# │  │ DataBlock  │──┼──────────────────┤  ._registers[] │
# │  │            │  │  setValues()     │                │
# │  └────────────┘  │←─────────────────┤  on_write()    │
# │                  │                  │                │
# └─────────────────┘                  └────────────────┘
#
# Client 讀取 register → DataBlock.getValues() → RegisterBlock.get_raw()
# Client 寫入 register → DataBlock.setValues() → Simulator.on_write()
```

### 4.2 啟動一個模擬 Modbus 伺服器

```python
import asyncio

from csp_lib.modbus_server import SimulationServer
from csp_lib.modbus_server.config import ServerConfig
from csp_lib.modbus_server.simulator.pcs import PCSSimulator, default_pcs_config

async def run_simulator():
    """啟動一個 PCS 模擬器"""

    # 1. 建立模擬器配置
    pcs_config = default_pcs_config(
        device_id="pcs_01",
        unit_id=10,
        base_address=0,
    )

    # 2. 建立模擬器實例
    pcs_sim = PCSSimulator(config=pcs_config)

    # 3. 建立模擬伺服器
    server_config = ServerConfig(host="127.0.0.1", port=5020, tick_interval=1.0)
    server = SimulationServer(config=server_config)
    server.add_simulator(pcs_sim)

    # 4. 啟動（使用 AsyncLifecycleMixin）
    async with server:
        print("模擬伺服器已啟動在 127.0.0.1:5020")
        print(f"PCS 模擬器 unit_id={pcs_sim.unit_id}")

        # 伺服器會持續運行...
        # tick loop 定期呼叫 pcs_sim.update()
        # 模擬 SOC 變化、功率斜率等行為
        await server.serve()

# asyncio.run(run_simulator())
```

### 4.3 設備模擬器種類

csp_lib 提供了多種預建的設備模擬器：

| 模擬器 | 類別 | 模擬行為 |
|--------|------|----------|
| PCS | `PCSSimulator` | P/Q 斜率控制、SOC 追蹤、告警管理 |
| 太陽能 | `SolarSimulator` | 日照曲線、功率擾動 |
| 負載 | `LoadSimulator` | 基礎負載 + 隨機擾動 |
| 電表 | `PowerMeterSimulator` | 電壓/頻率擾動、功率量測 |
| 發電機 | `GeneratorSimulator` | 啟停延遲、功率斜率、轉速 |

### 4.4 PCSSimulator 的模擬行為

`PCSSimulator` 是最複雜的模擬器，它模擬了真實 PCS 的核心行為：

```python
from csp_lib.modbus_server.config import PCSSimConfig, AlarmPointConfig, AlarmResetMode
from csp_lib.modbus_server.simulator.pcs import PCSSimulator, default_pcs_config

# 建立帶有告警配置的 PCS
alarm_configs = (
    AlarmPointConfig(
        alarm_code="OVER_TEMP",
        bit_position=0,
        reset_mode=AlarmResetMode.AUTO,     # 條件消失自動清除
    ),
    AlarmPointConfig(
        alarm_code="DC_BUS_FAULT",
        bit_position=1,
        reset_mode=AlarmResetMode.MANUAL,   # 需要手動 reset
    ),
)

pcs_config = default_pcs_config(
    device_id="pcs_01",
    unit_id=10,
    alarm_points=alarm_configs,
)

sim_config = PCSSimConfig(
    capacity_kwh=200.0,   # 200 kWh 電池
    p_ramp_rate=50.0,     # 50 kW/s 功率斜率
    q_ramp_rate=30.0,     # 30 kVar/s 無功斜率
    tick_interval=1.0,    # 1 秒更新週期
)

pcs = PCSSimulator(config=pcs_config, sim_config=sim_config)

# 模擬行為：
# 1. Client 寫入 p_setpoint = 100.0
#    → pcs.on_write("p_setpoint", 0.0, 100.0)
#    → 內部 RampBehavior.target = 100.0

# 2. 每次 tick (update()):
#    → p_actual 以 50 kW/s 趨近 100.0
#    → tick 1: p_actual = 50.0
#    → tick 2: p_actual = 100.0 (到達目標)

# 3. SOC 追蹤：
#    → 放電 100kW，200kWh 電池
#    → ΔSOC = -100 * 1 / 200 / 3600 * 100 ≈ -0.014% / tick

# 4. 觸發告警：
#    pcs.trigger_alarm("OVER_TEMP")
#    → alarm_register_1 的 bit 0 被設為 1
```

### 4.5 微電網聯動模擬

`MicrogridSimulator` 可以將多個設備模擬器連結在一起，模擬微電網的功率平衡：

```python
from csp_lib.modbus_server.config import MicrogridConfig
from csp_lib.modbus_server.microgrid import MicrogridSimulator
from csp_lib.modbus_server.simulator.pcs import PCSSimulator, default_pcs_config
from csp_lib.modbus_server.simulator.power_meter import (
    PowerMeterSimulator,
    default_power_meter_config,
)

# 建立多個模擬器
pcs = PCSSimulator(config=default_pcs_config("pcs_01", unit_id=10))
meter = PowerMeterSimulator(
    config=default_power_meter_config("meter_01", unit_id=20),
)

# 建立微電網聯動
microgrid_config = MicrogridConfig(
    grid_voltage=380.0,
    grid_frequency=60.0,
    voltage_noise=2.0,
    frequency_noise=0.02,
)

microgrid = MicrogridSimulator(config=microgrid_config)
microgrid.add_pcs(pcs)
microgrid.set_grid_meter(meter)

# 當 PCS 放電 50kW 時，電表的讀數會自動反映功率變化
# 微電網模擬器會協調所有設備的電壓、頻率等共享參數
```

---

## 5. 測試 Transforms：純邏輯的單元測試

csp_lib 的 Transform 設計是測試友善的典範——它們是不可變的 `frozen=True` dataclass，只有一個 `apply()` 方法，沒有任何副作用。

### 5.1 ScaleTransform 測試

```python
from csp_lib.equipment.core import ScaleTransform, RoundTransform

class TestScaleTransform:
    def test_basic_scale(self):
        """基本縮放轉換"""
        transform = ScaleTransform(magnitude=0.1, offset=0)
        assert transform.apply(500) == 50.0

    def test_scale_with_offset(self):
        """縮放 + 偏移：常見的溫度轉換"""
        # 溫度轉換: raw * 0.1 - 40
        transform = ScaleTransform(magnitude=0.1, offset=-40)
        assert transform.apply(650) == 25.0    # (650 * 0.1) + (-40) = 25.0°C
        assert transform.apply(400) == 0.0     # (400 * 0.1) + (-40) = 0.0°C
        assert transform.apply(900) == 50.0    # (900 * 0.1) + (-40) = 50.0°C

    def test_identity_transform(self):
        """恆等轉換（預設值）"""
        transform = ScaleTransform()  # magnitude=1.0, offset=0.0
        assert transform.apply(42) == 42.0

    def test_negative_scale(self):
        """負倍率：反轉符號"""
        transform = ScaleTransform(magnitude=-1.0, offset=0)
        assert transform.apply(100) == -100.0

    def test_type_error(self):
        """非數值輸入應拋出 TypeError"""
        transform = ScaleTransform(magnitude=0.1)
        with pytest.raises(TypeError):
            transform.apply("not_a_number")
```

### 5.2 EnumMapTransform 測試

```python
from csp_lib.equipment.core import EnumMapTransform

class TestEnumMapTransform:
    def test_known_values(self):
        """已知值的映射"""
        transform = EnumMapTransform(
            mapping={0: "STOP", 1: "RUN", 2: "FAULT"},
            default="UNKNOWN",
        )
        assert transform.apply(0) == "STOP"
        assert transform.apply(1) == "RUN"
        assert transform.apply(2) == "FAULT"

    def test_unknown_value(self):
        """未知值使用預設值"""
        transform = EnumMapTransform(
            mapping={0: "STOP", 1: "RUN"},
            default="UNKNOWN",
        )
        assert transform.apply(99) == "UNKNOWN"

    def test_real_world_pcs_modes(self):
        """真實 PCS 操作模式範例"""
        pcs_mode = EnumMapTransform(
            mapping={
                0: "STANDBY",
                1: "CHARGE",
                2: "DISCHARGE",
                3: "GRID_FORMING",
                4: "FAULT",
                5: "MAINTENANCE",
            },
            default="UNKNOWN",
        )
        assert pcs_mode.apply(2) == "DISCHARGE"
        assert pcs_mode.apply(4) == "FAULT"
```

### 5.3 Pipeline 組合測試

```python
from csp_lib.equipment.core.pipeline import ProcessingPipeline
from csp_lib.equipment.core import ScaleTransform, RoundTransform, ClampTransform

class TestProcessingPipeline:
    def test_chained_transforms(self):
        """多步驟轉換管線"""
        pipeline = ProcessingPipeline(
            steps=(
                ScaleTransform(magnitude=0.1, offset=-40),  # raw → 工程值
                RoundTransform(decimals=1),                   # 四捨五入
                ClampTransform(min_value=-20, max_value=80),  # 限制範圍
            ),
        )

        # 正常值
        assert pipeline.apply(650) == 25.0   # 650 * 0.1 - 40 = 25.0

        # 過高值被截斷
        assert pipeline.apply(1300) == 80.0  # 1300 * 0.1 - 40 = 90.0 → clamped to 80.0

        # 過低值被截斷
        assert pipeline.apply(100) == -20.0  # 100 * 0.1 - 40 = -30.0 → clamped to -20.0

    def test_bit_extract_pipeline(self):
        """位元提取管線"""
        from csp_lib.equipment.core import BitExtractTransform

        pipeline = ProcessingPipeline(
            steps=(
                BitExtractTransform(bit_offset=0, bit_length=1),
            ),
        )

        assert pipeline.apply(0b00000001) is True
        assert pipeline.apply(0b00000000) is False
        assert pipeline.apply(0b11111110) is False
```

---

## 6. 測試告警評估器

告警系統是工業控制中最關鍵的元件之一。csp_lib 提供了三種告警評估器，每種都可以獨立測試。

### 6.1 BitMaskAlarmEvaluator 測試

```python
from csp_lib.equipment.alarm import (
    AlarmDefinition,
    AlarmLevel,
    BitMaskAlarmEvaluator,
)

class TestBitMaskAlarmEvaluator:
    def setup_method(self):
        """每個測試前建立評估器"""
        self.evaluator = BitMaskAlarmEvaluator(
            point_name="fault_code",
            bit_alarms={
                0: AlarmDefinition(
                    code="OVER_TEMP",
                    name="Over-temperature",
                    level=AlarmLevel.WARNING,
                ),
                1: AlarmDefinition(
                    code="OVER_CURRENT",
                    name="Over-current",
                    level=AlarmLevel.ALARM,
                ),
                2: AlarmDefinition(
                    code="COMM_FAULT",
                    name="Communication fault",
                    level=AlarmLevel.INFO,
                ),
            },
        )

    def test_no_alarm(self):
        """所有位元都是 0 — 沒有告警"""
        result = self.evaluator.evaluate(0b000)
        assert result == {
            "OVER_TEMP": False,
            "OVER_CURRENT": False,
            "COMM_FAULT": False,
        }

    def test_single_alarm(self):
        """單一告警觸發"""
        result = self.evaluator.evaluate(0b001)  # bit 0 = 1
        assert result["OVER_TEMP"] is True
        assert result["OVER_CURRENT"] is False
        assert result["COMM_FAULT"] is False

    def test_multiple_alarms(self):
        """多個告警同時觸發"""
        result = self.evaluator.evaluate(0b101)  # bit 0 和 bit 2 = 1
        assert result["OVER_TEMP"] is True
        assert result["OVER_CURRENT"] is False
        assert result["COMM_FAULT"] is True

    def test_all_alarms(self):
        """所有告警觸發"""
        result = self.evaluator.evaluate(0b111)
        assert all(result.values())

    def test_none_value(self):
        """None 值 — 返回空結果"""
        result = self.evaluator.evaluate(None)
        assert result == {}

    def test_get_alarms(self):
        """取得所有告警定義"""
        alarms = self.evaluator.get_alarms()
        assert len(alarms) == 3
        codes = {a.code for a in alarms}
        assert codes == {"OVER_TEMP", "OVER_CURRENT", "COMM_FAULT"}
```

### 6.2 ThresholdAlarmEvaluator 測試

```python
from csp_lib.equipment.alarm.evaluator import (
    ThresholdAlarmEvaluator,
    ThresholdCondition,
    Operator,
)

class TestThresholdAlarmEvaluator:
    def test_temperature_thresholds(self):
        """溫度閾值告警"""
        evaluator = ThresholdAlarmEvaluator(
            point_name="temperature",
            conditions=[
                ThresholdCondition(
                    alarm=AlarmDefinition(
                        code="HIGH_TEMP_WARNING",
                        name="溫度偏高",
                        level=AlarmLevel.WARNING,
                    ),
                    operator=Operator.GT,
                    value=40.0,
                ),
                ThresholdCondition(
                    alarm=AlarmDefinition(
                        code="HIGH_TEMP_ALARM",
                        name="溫度過高",
                        level=AlarmLevel.ALARM,
                    ),
                    operator=Operator.GT,
                    value=55.0,
                ),
            ],
        )

        # 正常溫度
        result = evaluator.evaluate(25.0)
        assert result["HIGH_TEMP_WARNING"] is False
        assert result["HIGH_TEMP_ALARM"] is False

        # 偏高（WARNING）
        result = evaluator.evaluate(45.0)
        assert result["HIGH_TEMP_WARNING"] is True
        assert result["HIGH_TEMP_ALARM"] is False

        # 過高（WARNING + ALARM）
        result = evaluator.evaluate(60.0)
        assert result["HIGH_TEMP_WARNING"] is True
        assert result["HIGH_TEMP_ALARM"] is True

    def test_soc_boundaries(self):
        """SOC 邊界告警"""
        evaluator = ThresholdAlarmEvaluator(
            point_name="soc",
            conditions=[
                ThresholdCondition(
                    alarm=AlarmDefinition(code="LOW_SOC", name="SOC 過低"),
                    operator=Operator.LT,
                    value=10.0,
                ),
                ThresholdCondition(
                    alarm=AlarmDefinition(code="HIGH_SOC", name="SOC 過高"),
                    operator=Operator.GT,
                    value=95.0,
                ),
            ],
        )

        assert evaluator.evaluate(50.0) == {"LOW_SOC": False, "HIGH_SOC": False}
        assert evaluator.evaluate(5.0) == {"LOW_SOC": True, "HIGH_SOC": False}
        assert evaluator.evaluate(98.0) == {"LOW_SOC": False, "HIGH_SOC": True}
```

---

## 7. pytest + asyncio 模式

csp_lib 大量使用 async/await，因此測試也需要支援非同步。以下是常用的 pytest + asyncio 測試模式。

### 7.1 基本非同步測試

```python
import pytest

@pytest.mark.asyncio
async def test_device_write():
    """測試設備寫入"""
    device = make_device("pcs_01", values={"soc": 50.0})

    # Act
    await device.write("p_setpoint", 100.0)

    # Assert
    device.write.assert_called_once_with("p_setpoint", 100.0)
```

### 7.2 Fixture 模式

```python
import pytest

@pytest.fixture
def pcs_device():
    """PCS Mock 設備 fixture"""
    return make_device(
        "pcs_01",
        values={
            "active_power": 0.0,
            "reactive_power": 0.0,
            "soc": 50.0,
            "voltage": 380.0,
            "frequency": 60.0,
        },
    )

@pytest.fixture
def meter_device():
    """電表 Mock 設備 fixture"""
    return make_device(
        "meter_01",
        values={
            "active_power": -30.0,  # 購電 30kW
            "voltage": 380.2,
            "frequency": 60.01,
        },
    )

@pytest.fixture
def offline_device():
    """離線設備 fixture"""
    return make_device(
        "pcs_02",
        responsive=False,
        connected=False,
    )


class TestDeviceRegistry:
    def test_register_devices(self, pcs_device, meter_device):
        """測試設備註冊"""
        from csp_lib.integration.registry import DeviceRegistry

        registry = DeviceRegistry()
        registry.register(pcs_device)
        registry.register(meter_device)

        assert len(registry.devices) == 2
        assert registry.get("pcs_01") is pcs_device

    def test_healthy_devices(self, pcs_device, offline_device):
        """測試健康設備過濾"""
        from csp_lib.integration.registry import DeviceRegistry

        registry = DeviceRegistry()
        registry.register(pcs_device)
        registry.register(offline_device)

        healthy = [d for d in registry.devices.values() if d.is_healthy]
        assert len(healthy) == 1
        assert healthy[0].device_id == "pcs_01"
```

### 7.3 Parametrize 模式

```python
@pytest.mark.parametrize(
    "raw_value, expected",
    [
        (0, 0.0),
        (500, 50.0),
        (1000, 100.0),
        (650, 65.0),
    ],
)
def test_scale_transform_parametrized(raw_value, expected):
    """參數化測試：多組輸入/輸出"""
    transform = ScaleTransform(magnitude=0.1, offset=0)
    assert transform.apply(raw_value) == expected


@pytest.mark.parametrize(
    "fault_register, expected_alarms",
    [
        (0b000, []),
        (0b001, ["OVER_TEMP"]),
        (0b010, ["OVER_CURRENT"]),
        (0b011, ["OVER_TEMP", "OVER_CURRENT"]),
        (0b111, ["OVER_TEMP", "OVER_CURRENT", "COMM_FAULT"]),
    ],
)
def test_bitmask_alarm_parametrized(fault_register, expected_alarms):
    """參數化測試：各種告警組合"""
    evaluator = BitMaskAlarmEvaluator(
        point_name="fault_code",
        bit_alarms={
            0: AlarmDefinition(code="OVER_TEMP", name="OT"),
            1: AlarmDefinition(code="OVER_CURRENT", name="OC"),
            2: AlarmDefinition(code="COMM_FAULT", name="CF"),
        },
    )
    result = evaluator.evaluate(fault_register)
    triggered = [code for code, active in result.items() if active]
    assert sorted(triggered) == sorted(expected_alarms)
```

### 7.4 Arrange-Act-Assert 範例

```python
@pytest.mark.asyncio
async def test_complete_control_flow():
    """
    完整控制流程測試

    Arrange: 建立 Mock 設備和策略
    Act:     執行策略邏輯
    Assert:  驗證結果
    """
    # ---- Arrange ----
    pcs = make_device("pcs_01", values={
        "active_power": 0.0,
        "soc": 50.0,
    })
    meter = make_device("meter_01", values={
        "active_power": -50.0,  # 購電 50kW
    })

    from csp_lib.integration.registry import DeviceRegistry
    registry = DeviceRegistry()
    registry.register(pcs)
    registry.register(meter)

    # ---- Act ----
    # 簡化的策略邏輯：PCS 放電來抵消購電
    meter_power = meter.latest_values["active_power"]
    target_discharge = -meter_power  # 50.0 kW

    # 確保不超過 SOC 限制
    soc = pcs.latest_values["soc"]
    if soc > 10.0:  # SOC > 10% 才允許放電
        await pcs.write("p_setpoint", target_discharge)

    # ---- Assert ----
    pcs.write.assert_called_once_with("p_setpoint", 50.0)
```

---

## 8. 整合測試與 CI/CD 考量

### 8.1 不需要真實設備的整合測試

利用 `modbus_server`，你可以在 CI/CD 環境中運行完整的整合測試：

```python
import pytest

@pytest.fixture
async def simulation_server():
    """
    CI/CD 友善的模擬伺服器 fixture

    使用 csp_lib.modbus_server 建立模擬環境。
    所有通訊都是 localhost，不需要真實設備。
    """
    from csp_lib.modbus_server import SimulationServer
    from csp_lib.modbus_server.config import ServerConfig
    from csp_lib.modbus_server.simulator.pcs import PCSSimulator, default_pcs_config

    pcs_sim = PCSSimulator(config=default_pcs_config("pcs_01", unit_id=10))

    server = SimulationServer(
        config=ServerConfig(host="127.0.0.1", port=15020, tick_interval=0.5)
    )
    server.add_simulator(pcs_sim)

    async with server:
        yield server, pcs_sim


# 注意：下面的測試需要 pymodbus 依賴
# 在 CI/CD 中可用 SKIP_CYTHON=1 安裝測試版本
#
# @pytest.mark.asyncio
# async def test_real_modbus_communication(simulation_server):
#     server, pcs_sim = simulation_server
#
#     # 用真正的 Modbus client 連線
#     from pymodbus.client import AsyncModbusTcpClient
#     async with AsyncModbusTcpClient("127.0.0.1", port=15020) as client:
#         # 讀取 SOC（Float32, address=8, unit=10）
#         result = await client.read_holding_registers(8, count=2, slave=10)
#         assert result is not None
#
#         # 寫入 P setpoint（Float32, address=0, unit=10）
#         # ... encode float to registers ...
#         await client.write_registers(0, [0x4248, 0x0000], slave=10)
```

### 8.2 CI/CD 配置建議

```yaml
# .github/workflows/test.yml 中的測試配置重點
#
# 1. 單元測試（所有環境）：
#    uv run pytest tests/ -v -k "not integration"
#
# 2. 整合測試（需要 pymodbus）：
#    uv run pytest tests/ -v -k "integration"
#
# 3. 測試矩陣：
#    - Ubuntu + Windows（跨平台驗證）
#    - SKIP_CYTHON=1（避免 CI 編譯問題）
```

### 8.3 測試分層策略

```python
# conftest.py 中的 marker 定義
# pytest.ini 或 pyproject.toml:
# [tool.pytest.ini_options]
# markers = [
#     "unit: 單元測試（快速，無外部依賴）",
#     "integration: 整合測試（需要模擬伺服器）",
#     "slow: 慢速測試（涉及 asyncio.sleep）",
# ]

@pytest.mark.unit
def test_transform():
    """純邏輯，毫秒級完成"""
    t = ScaleTransform(magnitude=0.1)
    assert t.apply(100) == 10.0

@pytest.mark.integration
@pytest.mark.asyncio
async def test_device_communication():
    """需要模擬伺服器，秒級完成"""
    # ... 啟動模擬器，連線，讀寫 ...
    pass

@pytest.mark.slow
@pytest.mark.asyncio
async def test_ramp_behavior():
    """涉及時間模擬，可能需要數秒"""
    # ... 測試功率斜率控制 ...
    pass
```

### 8.4 測試覆蓋率關注點

在工業控制軟體中，測試覆蓋率的重點應該放在：

| 優先順序 | 關注領域 | 原因 |
|----------|----------|------|
| P0 | 保護邏輯（Protection） | 安全關鍵 |
| P0 | 告警評估（Alarm Evaluator）| 影響設備安全 |
| P1 | 資料轉換（Transform） | 錯誤值會導致誤判 |
| P1 | 策略邏輯（Strategy） | 核心業務邏輯 |
| P2 | 通訊錯誤處理 | 現場環境不穩定 |
| P2 | 邊界條件（SOC 0%/100%） | 常見的實機問題 |
| P3 | 正常流程 | 基本驗證 |

```python
# P0 級別測試範例：保護邏輯
@pytest.mark.parametrize(
    "soc, should_allow_discharge",
    [
        (50.0, True),   # 正常
        (10.0, True),   # 邊界
        (9.9, False),   # 低於保護閾值
        (0.0, False),   # 極端
        (100.0, True),  # 滿電
    ],
)
def test_soc_protection_discharge(soc, should_allow_discharge):
    """SOC 保護邏輯必須 100% 覆蓋"""
    min_soc = 10.0
    allowed = soc >= min_soc
    assert allowed == should_allow_discharge
```

---

## 9. 重點整理與下篇預告

### 本篇重點

1. **「沒有設備」是工業控制開發的常態，而非例外**。好的框架必須讓開發者在沒有實體設備的情況下也能有效地開發和測試。

2. **三層測試策略**：
   - **Mock 設備**（`make_device()`）：測試上層策略、Controller、Manager
   - **Simulator Server**（`modbus_server`）：測試完整的 Modbus 通訊堆疊
   - **Pure Logic Tests**：直接測試 Transform、Alarm Evaluator 等純函數元件

3. **Mock 技巧**：
   - 用 `PropertyMock` 模擬 `@property`
   - 用 `AsyncMock` 模擬 `async` 方法
   - 用 `side_effect` 模擬值序列或錯誤

4. **Simulator 系統**：
   - `PCSSimulator` 模擬功率斜率、SOC 追蹤、告警管理
   - `MicrogridSimulator` 實現多設備聯動模擬
   - `SimulatorDataBlock` 橋接 pymodbus 和 csp_lib

5. **測試模式**：
   - Arrange-Act-Assert 結構化測試
   - `@pytest.mark.asyncio` 支援非同步測試
   - `@pytest.mark.parametrize` 減少重複程式碼
   - Fixture 共享測試資源

6. **CI/CD 友善**：所有測試都可以在 GitHub Actions 中運行，不需要 VPN 或實體設備

### 給後端工程師的行動建議

- **從 Transform 測試開始**：它們最簡單、最純粹，是熟悉 csp_lib 測試模式的最佳起點
- **建立你的 `make_device()` 工廠**：根據你的專案需求客製化，加入常用的預設值
- **善用 parametrize**：工業控制的邊界條件很多，parametrize 可以系統性地覆蓋
- **在 CI 中跑整合測試**：利用 `modbus_server` 在 CI 中驗證完整流程，不要等到案場才發現問題

### 下篇預告

Part 2（協定轉接層）的旅程到此告一段落。下一篇文章我們將進入 **Part 3 — 設備抽象層**，深入探討 csp_lib 如何將底層的暫存器讀寫包裝成高階的設備抽象。我們會看到 `AsyncModbusDevice` 的完整架構——從連線管理、讀取排程、事件發射到告警處理，理解一個成熟的設備抽象層需要考慮哪些面向。

---

> **系列導航**
> - 上一篇：Article 09 — MQTT 與 Sparkplug B：工業物聯網的雲端橋樑
> - **本篇：Article 10 — 沒有設備也能開發：測試驅動的協定開發**
> - 下一篇：Article 11 — AsyncModbusDevice：設備抽象的完整架構
