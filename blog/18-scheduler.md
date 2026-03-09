# 排程與功率分配：從單機到多機協調控制

> **Part 4 — 控制迴路篇 | Article 18**
>
> 系列：從零打造工業控制協定框架：給軟體工程師的 CSP Library 實戰系列

---

## 前言

前三篇我們從 Edge-First 架構談到策略抽象、再到模式管理。但到目前為止，所有的討論都隱含了一個假設：系統只有一台 PCS（功率變換器）。

現實世界不是這樣的。一座 MW 級儲能電站通常包含多台 PCS、多組 BMS、電表、繼電保護器等設備。當策略算出「放電 1500kW」時，這 1500kW 要怎麼分配到三台 500kW 的 PCS？如果其中一台離線了呢？

這篇文章將帶你走完 csp_lib 從單機到多機協調控制的完整路徑。

---

## 1. 從單台設備到設備叢集

讓我們先看一個典型的儲能系統配置：

```
       ┌─────────┐
       │  電表    │ ← 量測匯流排功率、頻率、電壓
       └────┬────┘
            │
    ════════╪══════════ 匯流排
            │
    ┌───────┼───────┐
    │       │       │
┌───┴──┐ ┌─┴────┐ ┌┴─────┐
│PCS-01│ │PCS-02│ │PCS-03│ ← 各自 500kW 額定
└───┬──┘ └──┬───┘ └──┬───┘
    │       │        │
┌───┴──┐ ┌──┴───┐ ┌──┴───┐
│BMS-01│ │BMS-02│ │BMS-03│ ← 各自對應的電池管理系統
└──────┘ └──────┘ └──────┘
```

在這個系統中：
- 策略需要的輸入（SOC、頻率、電壓）來自多台設備
- 策略的輸出（功率指令）需要分配到多台設備
- 不同設備可能有不同的狀態（有些離線、有些告警）

csp_lib 用四個元件處理這個問題：

| 元件 | 職責 |
|------|------|
| `DeviceRegistry` | 設備註冊與查詢 |
| `ContextBuilder` | 多設備資料聚合 → StrategyContext |
| `PowerDistributor` | 系統級 Command → 各設備 Command |
| `CommandRouter` | 各設備 Command → Modbus 寫入 |

---

## 2. DeviceRegistry：設備管理的基礎

`DeviceRegistry` 是一個 trait-based 的設備查詢索引。它維護 `device_id ↔ trait` 的雙向索引：

```python
from csp_lib.integration import DeviceRegistry

registry = DeviceRegistry()

# 註冊設備，附加 traits 和 metadata
registry.register(
    pcs_01,
    traits=["pcs"],
    metadata={"rated_p": 500.0, "rated_s": 600.0},
)
registry.register(
    pcs_02,
    traits=["pcs"],
    metadata={"rated_p": 1000.0, "rated_s": 1200.0},
)
registry.register(
    bms_01, traits=["bms"],
    metadata={"capacity_kwh": 1000.0},
)
registry.register(
    bms_02, traits=["bms"],
    metadata={"capacity_kwh": 2000.0},
)
registry.register(meter, traits=["meter"])
```

### 2.1 Trait 是什麼

Trait 是設備的角色標籤。同一台設備可以有多個 trait：

```python
# 一台整合型設備同時是 PCS 和 BMS
registry.register(hybrid_unit, traits=["pcs", "bms"])
```

後續的 ContextBuilder 和 CommandRouter 都是透過 trait 來定位設備，而非 device_id。這讓系統配置可以抽象化——你不需要硬編碼「pcs_01」、「pcs_02」，只需要說「所有 pcs 設備」。

### 2.2 Metadata

Metadata 是設備的靜態資訊，在註冊時提供，不會隨運行改變：

```python
# 取得設備的 metadata
meta = registry.get_metadata("pcs_01")
# {"rated_p": 500.0, "rated_s": 600.0}
```

PowerDistributor 使用 metadata 中的額定功率來計算分配比例。

### 2.3 查詢 API

DeviceRegistry 提供多種查詢方式：

```python
# 依 trait 查詢所有設備
all_pcs = registry.get_devices_by_trait("pcs")

# 只取 responsive（通訊正常）的設備
alive_pcs = registry.get_responsive_devices_by_trait("pcs")

# 取第一台 responsive 設備
primary_meter = registry.get_first_responsive_device_by_trait("meter")

# 依 capability 查詢
devices_with_soc = registry.get_devices_with_capability("soc_readable")

# 所有已註冊設備
all_devices = registry.all_devices
```

所有查詢結果都按 `device_id` 排序，確保確定性——這在分配功率時很重要，避免因為排序不穩定導致每次分配結果不同。

---

## 3. ContextBuilder：將多台設備聚合為一個上下文

策略的 `execute()` 方法只接收一個 `StrategyContext`，但資料來源可能是多台設備。`ContextBuilder` 負責這個「多到一」的聚合。

### 3.1 ContextMapping：定義映射規則

```python
from csp_lib.integration.schema import ContextMapping, AggregateFunc

mappings = [
    # BMS SOC：多台 BMS 取平均
    ContextMapping(
        point_name="soc",
        context_field="soc",
        trait="bms",
        aggregate=AggregateFunc.AVERAGE,
    ),

    # PCS 總有功功率：多台 PCS 加總
    ContextMapping(
        point_name="active_power",
        context_field="extra.total_power",
        trait="pcs",
        aggregate=AggregateFunc.SUM,
    ),

    # 電表頻率：單一電表讀取
    ContextMapping(
        point_name="frequency",
        context_field="extra.frequency",
        trait="meter",
        aggregate=AggregateFunc.FIRST,
    ),

    # 電表電壓
    ContextMapping(
        point_name="voltage",
        context_field="extra.voltage",
        trait="meter",
        aggregate=AggregateFunc.FIRST,
    ),

    # 電表功率（用於逆送保護）
    ContextMapping(
        point_name="active_power",
        context_field="extra.meter_power",
        trait="meter",
        aggregate=AggregateFunc.FIRST,
    ),
]
```

### 3.2 聚合函式

`AggregateFunc` 支援五種內建聚合方式：

| 函式 | 說明 | 典型用途 |
|------|------|---------|
| `AVERAGE` | 平均值 | SOC（多組電池的平均電量） |
| `SUM` | 加總 | 總功率（多台 PCS 的功率加總） |
| `MIN` | 最小值 | 最低電池電壓 |
| `MAX` | 最大值 | 最高電池溫度 |
| `FIRST` | 第一台的值 | 電表讀值（通常只有一台電表） |

如果內建函式不夠用，可以提供自訂聚合：

```python
ContextMapping(
    point_name="cell_voltage",
    context_field="extra.voltage_spread",
    trait="bms",
    custom_aggregate=lambda values: max(values) - min(values),  # 電壓極差
)
```

### 3.3 context_field 的路徑語法

`context_field` 支援兩種寫法：

- `"soc"` → 設定 `context.soc`（直接欄位）
- `"extra.frequency"` → 設定 `context.extra["frequency"]`（extra 字典）

固定欄位（`soc`、`system_base`）用於最常見的資訊，其他都放在 `extra` 中。

### 3.4 ContextBuilder 的容錯機制

```python
class ContextBuilder:
    def _resolve_value(self, mapping: ContextMapping) -> Any:
        # 1. 讀取原始值
        if mapping.device_id is not None:
            raw = self._read_single_device(mapping)
        else:
            raw = self._read_trait_aggregate(mapping)

        # 2. 無有效值 → 使用預設值
        if raw is None:
            return mapping.default

        # 3. 套用轉換函式（出錯則用預設值）
        if mapping.transform is not None:
            try:
                raw = mapping.transform(raw)
            except Exception:
                return mapping.default

        return raw
```

每個步驟都有 fallback：
- 設備離線？回傳 `None`
- 所有設備都沒有該點位？回傳 `mapping.default`
- Transform 函式出錯？回傳 `mapping.default`

這確保了 ContextBuilder 永遠不會因為單一設備的問題而崩潰。

---

## 4. PowerDistributor：功率分配

策略算出的是系統級的功率指令（例如 1500kW），但每台 PCS 需要接收各自的指令。`PowerDistributor` 負責這個分配。

### 4.1 Protocol 定義

```python
@runtime_checkable
class PowerDistributor(Protocol):
    def distribute(
        self, command: Command, devices: list[DeviceSnapshot]
    ) -> dict[str, Command]:
        """
        分配功率

        Args:
            command: 系統級命令（已經過 ProtectionGuard）
            devices: 可用設備的狀態快照

        Returns:
            device_id → Command 的映射
        """
        ...
```

### 4.2 ProportionalDistributor：按額定比例分配

最常用的分配器。假設你有兩台 PCS：

```python
from csp_lib.integration.distributor import ProportionalDistributor

distributor = ProportionalDistributor(rated_key="rated_p")

# registry 中的 metadata:
# pcs_01: rated_p=500kW
# pcs_02: rated_p=1000kW
# 總計: 1500kW

# 系統指令: 放電 900kW
# pcs_01 分得: 900 * (500/1500) = 300kW
# pcs_02 分得: 900 * (1000/1500) = 600kW
```

原始碼很簡潔：

```python
class ProportionalDistributor:
    def __init__(self, rated_key: str = "rated_p"):
        self._rated_key = rated_key

    def distribute(self, command: Command, devices: list[DeviceSnapshot]) -> dict[str, Command]:
        n = len(devices)
        if n == 0:
            return {}

        total_rated = sum(d.metadata.get(self._rated_key, 0.0) for d in devices)
        if total_rated <= 0:
            # 全部設備無額定值 → fallback 均分
            return EqualDistributor().distribute(command, devices)

        result = {}
        for d in devices:
            ratio = d.metadata.get(self._rated_key, 0.0) / total_rated
            result[d.device_id] = Command(
                p_target=command.p_target * ratio,
                q_target=command.q_target * ratio,
            )
        return result
```

### 4.3 EqualDistributor：均等分配

最簡單的分配方式，適用於所有設備規格相同的場景：

```python
class EqualDistributor:
    def distribute(self, command: Command, devices: list[DeviceSnapshot]) -> dict[str, Command]:
        n = len(devices)
        if n == 0:
            return {}
        p_each = command.p_target / n
        q_each = command.q_target / n
        return {
            d.device_id: Command(p_target=p_each, q_target=q_each)
            for d in devices
        }
```

### 4.4 SOCBalancingDistributor：SOC 平衡分配

進階分配器——在比例分配的基礎上，根據各設備的 SOC 偏差調整分配：

```python
class SOCBalancingDistributor:
    """
    SOC 平衡分配：
    - 放電 (P > 0)：SOC 較高的設備多放
    - 充電 (P < 0)：SOC 較低的設備多充
    """

    def __init__(
        self,
        rated_key: str = "rated_p",
        soc_capability: str = "soc_readable",
        soc_slot: str = "soc",
        gain: float = 2.0,
    ):
        self._rated_key = rated_key
        self._soc_capability = soc_capability
        self._soc_slot = soc_slot
        self._gain = gain
```

演算法的核心是：

```
avg_soc = mean(所有設備的 SOC)

對每台設備:
    soc_deviation = (device_soc - avg_soc) / 100
    若放電: weight = rated * (1 + gain * soc_deviation)
    若充電: weight = rated * (1 - gain * soc_deviation)
```

舉例：兩台 500kW PCS，SOC 分別是 80% 和 60%，要放電 800kW：

```
avg_soc = 70%
PCS-A (SOC=80%): deviation = +0.10, weight = 500 * (1 + 2*0.10) = 600
PCS-B (SOC=60%): deviation = -0.10, weight = 500 * (1 - 2*0.10) = 400
total_weight = 1000

PCS-A 分得: 800 * 600/1000 = 480kW (多放，因為 SOC 高)
PCS-B 分得: 800 * 400/1000 = 320kW (少放，因為 SOC 低)
```

這樣做的好處是：經過一段時間的運行，各設備的 SOC 會趨向一致，避免「一台用光、一台還滿」的不均衡狀態。

### 4.5 自訂分配器

由於 `PowerDistributor` 是一個 Protocol，你可以輕鬆實作自己的分配器：

```python
class PriorityDistributor:
    """
    優先使用特定設備，其餘作為備援
    """

    def __init__(self, primary_ids: list[str], rated_key: str = "rated_p"):
        self._primary_ids = primary_ids
        self._rated_key = rated_key

    def distribute(self, command: Command, devices: list[DeviceSnapshot]) -> dict[str, Command]:
        # 優先分配給 primary 設備
        primaries = [d for d in devices if d.device_id in self._primary_ids]
        secondaries = [d for d in devices if d.device_id not in self._primary_ids]

        primary_capacity = sum(d.metadata.get(self._rated_key, 0) for d in primaries)
        remaining_p = command.p_target
        remaining_q = command.q_target
        result = {}

        # 先填滿 primary
        for d in primaries:
            rated = d.metadata.get(self._rated_key, 0)
            if primary_capacity > 0:
                ratio = rated / primary_capacity
                p = min(remaining_p * ratio, rated)
                result[d.device_id] = Command(p_target=p, q_target=remaining_q * ratio)

        # 超出 primary 容量的部分分配給 secondary
        used_p = sum(c.p_target for c in result.values())
        overflow_p = command.p_target - used_p
        if overflow_p > 0 and secondaries:
            for d in secondaries:
                result[d.device_id] = Command(
                    p_target=overflow_p / len(secondaries),
                    q_target=0.0,
                )

        return result
```

---

## 5. CommandRouter：將指令寫入設備

PowerDistributor 的輸出是 `dict[str, Command]`——每台設備對應一個 Command。`CommandRouter` 負責把這些 Command 轉換為實際的 Modbus 寫入。

### 5.1 CommandMapping：定義寫入規則

```python
from csp_lib.integration.schema import CommandMapping

command_mappings = [
    # P 指令 → 寫入所有 pcs trait 設備的 p_set 點位
    CommandMapping(
        command_field="p_target",
        point_name="p_set",
        trait="pcs",
    ),
    # Q 指令 → 寫入所有 pcs trait 設備的 q_set 點位
    CommandMapping(
        command_field="q_target",
        point_name="q_set",
        trait="pcs",
    ),
]
```

### 5.2 廣播模式 vs 分配模式

CommandRouter 支援兩種路由模式：

**廣播模式**（無 PowerDistributor）：

```python
# 所有 pcs 設備收到相同的值
await command_router.route(command)
# pcs_01.write("p_set", 1500.0)
# pcs_02.write("p_set", 1500.0)
# pcs_03.write("p_set", 1500.0)
```

**分配模式**（有 PowerDistributor）：

```python
# 每台設備收到各自的分配值
per_device = distributor.distribute(command, snapshots)
await command_router.route_per_device(command, per_device)
# pcs_01.write("p_set", 500.0)
# pcs_02.write("p_set", 500.0)
# pcs_03.write("p_set", 500.0)
```

### 5.3 安全防護

CommandRouter 內建多重安全防護：

```python
async def _write_single(self, device_id: str, point_name: str, value: object) -> None:
    device = self._registry.get_device(device_id)

    if device is None:
        logger.warning(f"Device '{device_id}' not found, skipping write.")
        return

    if device.is_protected:
        logger.warning(f"Device '{device_id}' is protected (alarm), skipping write.")
        return

    if not device.is_responsive:
        logger.warning(f"Device '{device_id}' is not responsive, skipping write.")
        return

    await self._safe_write(device, point_name, value)

@staticmethod
async def _safe_write(device, point_name: str, value: object) -> None:
    try:
        await device.write(point_name, value)
    except DeviceError:
        logger.warning(
            f"Write failed for device '{device.device_id}' point '{point_name}'.",
            exc_info=True,
        )
```

三道防線：
1. **設備不存在**？跳過（可能已被移除）
2. **設備告警**（is_protected）？跳過（不對異常設備發指令）
3. **設備離線**（not is_responsive）？跳過（發了也收不到）
4. **寫入失敗**？記錄但不中斷其他設備的寫入

---

## 6. HeartbeatService：心跳看門狗

心跳服務是一個容易被忽略但至關重要的元件。它讓設備端知道「控制器還活著」。

### 6.1 為什麼需要心跳

想像一個場景：控制器程式因為 OOM 被 kernel 殺掉，但設備仍然按照最後收到的指令持續運行。如果最後的指令是「放電 500kW」，設備會一直放電直到電池完全耗盡。

心跳機制解決了這個問題：控制器定期向設備寫入心跳值，設備端的 watchdog 計時器會監控這個值。如果心跳停止超過設定的超時時間，設備韌體自動進入安全模式（停機或低功率）。

### 6.2 使用方式

**方式一：明確映射**

```python
from csp_lib.integration.schema import HeartbeatMapping, HeartbeatMode

config = SystemControllerConfig(
    heartbeat_mappings=[
        HeartbeatMapping(
            point_name="heartbeat",
            trait="pcs",
            mode=HeartbeatMode.TOGGLE,  # 交替寫入 0 和 1
        ),
    ],
    heartbeat_interval=1.0,  # 每秒一次
)
```

**方式二：能力發現（Capability-based）**

```python
config = SystemControllerConfig(
    use_heartbeat_capability=True,  # 自動找出所有具備 HEARTBEAT 能力的設備
    heartbeat_capability_mode=HeartbeatMode.INCREMENT,  # 遞增計數
    heartbeat_capability_increment_max=65535,
)
```

### 6.3 三種心跳模式

| 模式 | 行為 | 設備端驗證 |
|------|------|-----------|
| `TOGGLE` | 交替 0 / 1 | 檢查值是否有變化 |
| `INCREMENT` | 0, 1, 2, ..., max, 0, ... | 檢查值是否遞增 |
| `CONSTANT` | 固定值（如 1） | 任何非零值表示在線 |

### 6.4 策略感知的暫停/恢復

HeartbeatService 會感知當前策略的 `suppress_heartbeat` 屬性：

```python
# SystemController 在策略切換時自動控制心跳
async def _on_strategy_change(self, old, new):
    resolved = self._resolve_strategy()
    await self._executor.set_strategy(resolved)

    if self._heartbeat is not None and resolved is not None:
        if resolved.suppress_heartbeat:
            self._heartbeat.pause()   # Bypass 模式：暫停心跳
        else:
            self._heartbeat.resume()  # 其他模式：恢復心跳
```

暫停心跳是一個明確的「控制權移交」信號。設備端的 watchdog 會因此觸發，進入安全模式——這正是操作員接手時想要的行為。

### 6.5 心跳的容錯

```python
@staticmethod
async def _safe_write(device, point_name: str, value: int) -> None:
    try:
        await device.write(point_name, value)
    except DeviceError:
        logger.warning(
            f"Heartbeat write failed: device='{device.device_id}' point='{point_name}'",
            exc_info=True,
        )
```

單一設備的心跳寫入失敗不會影響其他設備。如果某台設備持續收不到心跳，它會自行進入安全模式——這是期望的行為。

---

## 7. 完整範例：多機協調控制

讓我們把所有元件串在一起，建立一個完整的多機控制系統：

```python
import asyncio
from csp_lib.integration import DeviceRegistry, ContextMapping
from csp_lib.integration.schema import (
    AggregateFunc, CommandMapping, HeartbeatMapping, HeartbeatMode,
)
from csp_lib.integration.system_controller import SystemController, SystemControllerConfig
from csp_lib.integration.distributor import ProportionalDistributor
from csp_lib.controller.core import SystemBase
from csp_lib.controller.strategies.pq_strategy import PQModeStrategy, PQModeConfig
from csp_lib.controller.strategies.fp_strategy import FPStrategy, FPConfig
from csp_lib.controller.strategies.bypass_strategy import BypassStrategy
from csp_lib.controller.system import ModePriority
from csp_lib.controller.system.protection import (
    SOCProtection, SOCProtectionConfig,
    ReversePowerProtection,
)


async def main():
    # ============ 1. 建立設備 Registry ============

    registry = DeviceRegistry()

    # 假設 pcs_01, pcs_02, bms_01, bms_02, meter 已建立
    registry.register(
        pcs_01, traits=["pcs"],
        metadata={"rated_p": 500.0, "rated_s": 600.0},
    )
    registry.register(
        pcs_02, traits=["pcs"],
        metadata={"rated_p": 1000.0, "rated_s": 1200.0},
    )
    registry.register(bms_01, traits=["bms"])
    registry.register(bms_02, traits=["bms"])
    registry.register(meter, traits=["meter"])

    # ============ 2. 配置 SystemController ============

    config = SystemControllerConfig(
        # --- Context 映射 ---
        context_mappings=[
            ContextMapping(
                point_name="soc",
                context_field="soc",
                trait="bms",
                aggregate=AggregateFunc.AVERAGE,
            ),
            ContextMapping(
                point_name="frequency",
                context_field="extra.frequency",
                trait="meter",
                aggregate=AggregateFunc.FIRST,
            ),
            ContextMapping(
                point_name="active_power",
                context_field="extra.meter_power",
                trait="meter",
                aggregate=AggregateFunc.FIRST,
            ),
        ],

        # --- Command 映射 ---
        command_mappings=[
            CommandMapping(
                command_field="p_target",
                point_name="p_set",
                trait="pcs",
            ),
            CommandMapping(
                command_field="q_target",
                point_name="q_set",
                trait="pcs",
            ),
        ],

        # --- 功率分配 ---
        power_distributor=ProportionalDistributor(rated_key="rated_p"),

        # --- 系統基準 ---
        system_base=SystemBase(p_base=1500.0, q_base=1500.0),

        # --- 保護規則 ---
        protection_rules=[
            SOCProtection(SOCProtectionConfig(
                soc_high=95.0,
                soc_low=5.0,
                warning_band=5.0,
            )),
            ReversePowerProtection(threshold=0.0),
        ],

        # --- 心跳 ---
        heartbeat_mappings=[
            HeartbeatMapping(
                point_name="heartbeat",
                trait="pcs",
                mode=HeartbeatMode.TOGGLE,
            ),
        ],
        heartbeat_interval=1.0,

        # --- 告警模式 ---
        auto_stop_on_alarm=True,
    )

    # ============ 3. 建立 Controller 並註冊模式 ============

    controller = SystemController(registry, config)

    controller.register_mode(
        "pq",
        PQModeStrategy(PQModeConfig(p=1000.0, q=0.0)),
        ModePriority.SCHEDULE,
        description="定功率放電 1000kW",
    )

    controller.register_mode(
        "afc",
        FPStrategy(FPConfig(f_base=60.0)),
        ModePriority.SCHEDULE,
        description="AFC 頻率調節",
    )

    controller.register_mode(
        "bypass",
        BypassStrategy(),
        ModePriority.MANUAL,
        description="維護模式",
    )

    # ============ 4. 啟動 ============

    await controller.set_base_mode("pq")

    async with controller:
        print("SystemController 已啟動")
        print(f"生效模式: {controller.effective_mode_name}")
        print(f"健康狀態: {controller.health().status}")

        # 控制迴路自動運行：
        # 每秒: ContextBuilder → PQModeStrategy → ProtectionGuard → ProportionalDistributor → CommandRouter
        #
        # pcs_01 (500kW / 1500kW) 收到: p_set = 1000 * 1/3 ≈ 333.3kW
        # pcs_02 (1000kW / 1500kW) 收到: p_set = 1000 * 2/3 ≈ 666.7kW

        await asyncio.Event().wait()


asyncio.run(main())
```

### 7.1 運行時的資料流

讓我們追蹤一個完整的控制週期：

```
1. ContextBuilder.build()
   ├─ bms_01.latest_values["soc"] = 82.0
   ├─ bms_02.latest_values["soc"] = 78.0
   ├─ AVERAGE → soc = 80.0
   ├─ meter.latest_values["frequency"] = 59.95
   └─ meter.latest_values["active_power"] = 200.0
   → StrategyContext(soc=80.0, extra={"frequency": 59.95, "meter_power": 200.0})

2. PQModeStrategy.execute(context)
   → Command(p_target=1000.0, q_target=0.0)

3. ProtectionGuard.apply(command, context)
   ├─ SOCProtection: SOC=80% → 正常範圍，不修改
   └─ ReversePowerProtection: 1000 > 200+0=200 → clamp to 200kW
   → Command(p_target=200.0, q_target=0.0)  # 受逆送保護限制

4. ProportionalDistributor.distribute(...)
   ├─ pcs_01: 200 * 500/1500 = 66.7kW
   └─ pcs_02: 200 * 1000/1500 = 133.3kW

5. CommandRouter.route_per_device(...)
   ├─ pcs_01.write("p_set", 66.7)
   └─ pcs_02.write("p_set", 133.3)
```

注意步驟 3 中，逆送保護把 1000kW 的策略輸出限制到 200kW。這就是保護鏈的價值——即使策略配置了 1000kW 放電，保護規則確保不會超過電表功率（200kW），避免逆送。

### 7.2 設備離線時的行為

假設 pcs_01 突然離線（通訊中斷）：

```
ContextBuilder.build()
   # pcs_01 的值仍然可以從 latest_values 讀取（上次的快照）
   # 但 is_responsive = False

ProportionalDistributor.distribute(...)
   # SystemController._build_device_snapshots() 會過濾掉 non-responsive 設備
   # 只有 pcs_02 參與分配
   # pcs_02 收到全部 200kW

CommandRouter.route_per_device(...)
   # pcs_01 因 is_responsive=False 被跳過
   # pcs_02.write("p_set", 200.0)

HeartbeatService
   # pcs_01 因 is_responsive=False 跳過心跳寫入
   # pcs_01 的 watchdog 會超時 → 自動進入安全模式
```

一切自動處理，不需要任何手動介入。

---

## 8. 重點回顧

1. **DeviceRegistry 是設備管理的基礎**。Trait-based 索引讓配置可以抽象化——不需要硬編碼 device_id，用 trait 描述設備角色。Metadata 記錄靜態資訊供功率分配使用。

2. **ContextBuilder 聚合多設備資料為單一上下文**。五種內建聚合函式（AVERAGE / SUM / MIN / MAX / FIRST）加上自訂聚合，覆蓋了所有資料聚合需求。完善的 fallback 機制確保單一設備異常不會影響整體。

3. **PowerDistributor 將系統級指令分配到多台設備**。ProportionalDistributor 按額定容量比例分配，SOCBalancingDistributor 在此基礎上根據 SOC 偏差做動態調整。Protocol-based 設計讓自訂分配器簡單易實現。

4. **CommandRouter 負責最終的設備寫入**。支援廣播模式和分配模式，內建三重安全防護（設備不存在 / 告警 / 離線），單一寫入失敗不影響其他設備。

5. **HeartbeatService 是控制器存活的證明**。三種心跳模式（TOGGLE / INCREMENT / CONSTANT）適配不同設備需求。策略感知的 pause/resume 機制確保 Bypass 模式時正確移交控制權。

6. **所有元件的設計都遵循同一原則：局部故障不擴散**。設備離線時跳過、寫入失敗時記錄但繼續、資料缺失時用預設值。這在管理多台設備時尤其重要——你不能因為一台設備的問題讓整個系統停擺。

---

## 下篇預告

至此，Part 4 控制迴路篇告一段落。我們從 Edge-First 的架構理念出發，經歷了策略抽象、模式管理、保護機制，最終到多機協調控制。下一篇將進入 Part 5，探討資料持久化與上傳——當控制迴路穩定運行之後，如何將運行數據可靠地儲存到 MongoDB，以及如何設計批次上傳機制來應對網路不穩定的邊緣環境。
