# 控制策略抽象：Strategy Pattern 在能源管理的應用

> **Part 4 — 控制迴路篇 | Article 16**
>
> 系列：從零打造工業控制協定框架：給軟體工程師的 CSP Library 實戰系列

---

## 前言

上一篇我們討論了為什麼控制迴路要放在邊緣。現在進入核心問題：邊緣控制器裡的控制邏輯該怎麼組織？

一座儲能系統可能需要支援十幾種運行模式：定功率輸出、自動頻率調節、電壓無功控制、離網模式、排程控制......如果把這些邏輯全部寫成 `if-elif` 的巨型函式，你會得到一個數千行的怪物，而且每次新增模式都要冒著破壞既有功能的風險。

csp_lib 的解決方案是 GoF 的 **Strategy Pattern**，但針對工業控制場景做了精心的調整。這篇文章將深入剖析這套抽象設計。

---

## 1. Strategy Pattern 在工業控制的特殊需求

在經典的 GoF Strategy Pattern 中，你有一個 `Context` 物件持有一個 `Strategy` 引用，不同的策略實現不同的演算法。概念上這很簡單，但在工業控制場景中，我們需要額外考慮幾個問題：

1. **生命週期管理**：策略的切換不只是換個演算法。例如進入離網模式時需要斷開 ACB（自動斷路器），離開時需要等待同步信號後才能閉合——策略需要 `on_activate` 和 `on_deactivate` 鉤子。

2. **執行模式差異**：有些策略需要每秒執行一次（AFC），有些只在事件觸發時執行（離網切換），有些則是兩者的混合。

3. **不可變輸出**：策略的輸出（功率指令）必須是不可變的。一旦產生，任何下游的修改都必須產生新的物件——這確保了保護鏈可以安全地追蹤修改歷史。

4. **上下文隔離**：策略不應該直接存取設備。它只應該基於注入的上下文資訊做決策，這保證了策略的可測試性和可組合性。

---

## 2. Strategy 抽象基礎類別

csp_lib 的策略抽象定義在 `csp_lib/controller/core/strategy.py`：

```python
from abc import ABC, abstractmethod
from csp_lib.controller.core import Command, ExecutionConfig, StrategyContext


class Strategy(ABC):
    """
    策略抽象基礎類別

    所有策略必須繼承此類別並實作：
    - execution_config: 回傳執行配置
    - execute(): 執行策略邏輯並回傳 Command
    """

    @property
    @abstractmethod
    def execution_config(self) -> ExecutionConfig:
        """回傳策略的執行配置"""
        pass

    @abstractmethod
    def execute(self, context: StrategyContext) -> Command:
        """
        執行策略邏輯

        Args:
            context: 執行時上下文，包含 last_command、soc 等狀態

        Returns:
            Command: 策略輸出命令
        """
        pass

    @property
    def suppress_heartbeat(self) -> bool:
        """
        是否在此策略啟用期間暫停心跳寫入。
        預設 False。旁路模式等需要完全停止控制器輸出的策略應覆寫為 True。
        """
        return False

    async def on_activate(self) -> None:
        """策略啟用時呼叫 (可選覆寫)"""
        pass

    async def on_deactivate(self) -> None:
        """策略停用時呼叫 (可選覆寫)"""
        pass
```

這個抽象類別有幾個重要的設計決策：

### 2.1 兩個必須實作的方法

- `execution_config`：告訴執行器「我該怎麼被執行」
- `execute()`：接收上下文，回傳指令

### 2.2 兩個可選的生命週期鉤子

- `on_activate()`：策略被啟用時呼叫，可以做初始化
- `on_deactivate()`：策略被停用時呼叫，可以做清理

注意這兩個方法是 `async` 的——因為生命週期操作可能涉及 I/O（例如控制斷路器）。

### 2.3 suppress_heartbeat 屬性

這個看起來不起眼的屬性其實很關鍵。當 `suppress_heartbeat` 為 `True` 時，`SystemController` 會暫停心跳寫入服務。這用於 Bypass 模式：操作人員接管控制權時，控制器不應該繼續向設備發送任何信號，包括心跳。

---

## 3. StrategyContext：策略的執行環境

策略的 `execute()` 方法接收一個 `StrategyContext` 物件，這是策略唯一的資訊來源：

```python
from dataclasses import dataclass, field
from datetime import datetime
from typing import Any, Optional

from csp_lib.controller.core.command import Command, SystemBase


@dataclass
class StrategyContext:
    """策略執行時上下文 (唯讀)"""

    last_command: Command = field(default_factory=Command)
    soc: Optional[float] = None
    system_base: Optional[SystemBase] = None
    current_time: Optional[datetime] = None
    extra: dict[str, Any] = field(default_factory=dict)

    def percent_to_kw(self, p_percent: float) -> float:
        """將百分比轉換為 kW"""
        if self.system_base is None:
            raise ValueError("system_base is not set in StrategyContext")
        return p_percent * self.system_base.p_base / 100

    def percent_to_kvar(self, q_percent: float) -> float:
        """將百分比轉換為 kVar"""
        if self.system_base is None:
            raise ValueError("system_base is not set in StrategyContext")
        return q_percent * self.system_base.q_base / 100
```

### 3.1 固定欄位 vs extra 字典

StrategyContext 有幾個固定欄位：

| 欄位 | 型別 | 說明 |
|------|------|------|
| `last_command` | `Command` | 上一次策略輸出的指令 |
| `soc` | `float?` | 儲能系統的 SOC（%） |
| `system_base` | `SystemBase?` | 系統基準值（用於百分比換算） |
| `current_time` | `datetime?` | 當前時間（由 Executor 注入） |

其他所有資訊都放在 `extra` 字典中。這個設計是刻意的——不同的策略需要不同的輸入資訊（AFC 需要頻率、QV 需要電壓、逆送保護需要電表功率），強制把它們都變成固定欄位會讓 StrategyContext 膨脹到無法管理。

### 3.2 百分比轉換輔助方法

`percent_to_kw()` 和 `percent_to_kvar()` 看起來簡單，卻解決了一個常見的混淆問題。AFC 策略計算出的功率是百分比（例如 60% 額定功率），但設備需要的是絕對值（例如 300kW）。這兩個方法統一了轉換邏輯，避免每個策略各自實現時出錯。

`SystemBase` 本身也是不可變的：

```python
@dataclass(frozen=True)
class SystemBase(ConfigMixin):
    """系統基準值，用於百分比與絕對值轉換"""
    p_base: float = 0.0  # 有功功率基準值 (kW)
    q_base: float = 0.0  # 無功功率基準值 (kVar)
```

---

## 4. Command：不可變的策略輸出

策略的輸出是一個 `Command` 物件：

```python
@dataclass(frozen=True)
class Command:
    """策略輸出命令 (不可變)"""

    p_target: float = 0.0  # 有功功率目標值 (kW)
    q_target: float = 0.0  # 無功功率目標值 (kVar)

    def with_p(self, p: float) -> Command:
        """建立新 Command，替換 P 值"""
        return dataclasses.replace(self, p_target=p)

    def with_q(self, q: float) -> Command:
        """建立新 Command，替換 Q 值"""
        return dataclasses.replace(self, q_target=q)

    def __str__(self) -> str:
        return f"Command(P={self.p_target:.1f}kW, Q={self.q_target:.1f}kVar)"
```

### 4.1 為什麼要 frozen=True

`Command` 使用 `frozen=True` 使其成為不可變物件。這不是學術性的堅持，而是安全需求：

```python
# 保護鏈中的追蹤
result = ProtectionResult(
    original_command=command,        # 策略原始輸出
    protected_command=protected,     # 保護後的版本
    triggered_rules=["soc_protection"],
)

# 因為 Command 是 frozen 的，我們可以確信：
assert result.original_command.p_target == 500.0  # 永遠不會被意外修改
```

如果 `Command` 是可變的，保護鏈可能不小心修改了 `original_command`，導致無法追蹤「原本想輸出什麼、被保護規則改成了什麼」。

### 4.2 with_p() / with_q() 模式

由於 `Command` 不可變，修改時需要建立新物件。`with_p()` 和 `with_q()` 提供了流暢的 API：

```python
# 保護規則中的用法
def evaluate(self, command: Command, context: StrategyContext) -> Command:
    if context.soc >= 95 and command.p_target < 0:
        # SOC 過高，禁止充電，但保留 Q 值
        return command.with_p(0.0)
    return command
```

---

## 5. 內建策略深度解析

csp_lib 提供了六種內建策略，覆蓋了儲能系統最常見的運行模式。

### 5.1 PQModeStrategy：固定功率輸出

最簡單也最常用的策略——輸出固定的 P 和 Q 值：

```python
@dataclass
class PQModeConfig(ConfigMixin):
    """PQ 模式配置"""
    p: float = 0.0  # kW
    q: float = 0.0  # kVar


class PQModeStrategy(Strategy):
    """根據配置輸出固定的 P/Q 值"""

    def __init__(self, config: Optional[PQModeConfig] = None):
        self._config = config or PQModeConfig()

    @property
    def execution_config(self) -> ExecutionConfig:
        return ExecutionConfig(mode=ExecutionMode.PERIODIC, interval_seconds=1)

    def execute(self, context: StrategyContext) -> Command:
        return Command(p_target=self._config.p, q_target=self._config.q)

    def update_config(self, config: PQModeConfig) -> None:
        """支援動態更新配置"""
        self._config = config
```

PQModeStrategy 的 `execute()` 甚至沒有使用 `context` 參數——它只是忠實地輸出配置中的值。看起來簡單，但配合排程系統，它可以實現時間電價套利：

```python
# 尖峰時段放電
peak_strategy = PQModeStrategy(PQModeConfig(p=500.0, q=0.0))

# 離峰時段充電
offpeak_strategy = PQModeStrategy(PQModeConfig(p=-500.0, q=0.0))
```

### 5.2 QVStrategy：電壓-無功功率控制

Volt-VAR Droop Control（電壓下垂控制）是電力系統中穩定電壓的標準方法：

```python
@dataclass
class QVConfig(ConfigMixin):
    """QV 模式配置"""
    nominal_voltage: float = 380.0  # 額定電壓 (V)
    v_set: float = 100.0            # 電壓設定值 (%)
    droop: float = 5.0              # 下垂係數 (%)
    v_deadband: float = 0.0         # 死區 (%)
    q_max_ratio: float = 0.5        # 最大無功功率比值


class QVStrategy(Strategy):
    def execute(self, context: StrategyContext) -> Command:
        voltage = context.extra.get("voltage")
        if voltage is None:
            return context.last_command  # 無資料時維持上次

        q_ratio = self._calculate_q_ratio(voltage)

        # 轉換為 kVar
        if context.system_base is not None:
            q_kvar = context.percent_to_kvar(q_ratio * 100)
        else:
            q_kvar = q_ratio

        # 保持 P 不變
        return Command(p_target=context.last_command.p_target, q_target=q_kvar)

    def _calculate_q_ratio(self, voltage: float) -> float:
        cfg = self._config
        v_pu = voltage / cfg.nominal_voltage
        v_set_pu = cfg.v_set / 100
        v_deadband_pu = cfg.v_deadband / 100
        droop_pu = cfg.droop / 100

        # 電壓過低 → 正 Q（提供無功，支撐電壓）
        if v_pu <= v_set_pu - v_deadband_pu:
            q_ratio = 0.5 * (v_set_pu - v_deadband_pu - v_pu) / (v_set_pu * droop_pu)
            return min(q_ratio, cfg.q_max_ratio)

        # 電壓過高 → 負 Q（吸收無功，降低電壓）
        if v_pu >= v_set_pu + v_deadband_pu:
            q_ratio = 0.5 * (v_set_pu + v_deadband_pu - v_pu) / (v_set_pu * droop_pu)
            return max(q_ratio, -cfg.q_max_ratio)

        # 死區內 → Q = 0
        return 0.0
```

QVStrategy 有幾個值得注意的設計：

1. **無電壓資料時回傳 `last_command`**——不是零，而是維持上次的輸出。這避免了感測器暫時離線時系統突然歸零的抖動。

2. **保持 P 不變**——QV 策略只控制 Q（無功），P（有功）由其他策略決定。這讓 QV 可以和 PQ 等策略組合使用。

3. **死區設計**——在設定值附近有一個小範圍不做調整，防止因為電壓微小波動導致 Q 不斷變化。

### 5.3 FPStrategy：頻率-功率控制（AFC）

AFC 是儲能系統的高價值應用。FPStrategy 使用分段線性插值來定義頻率-功率曲線：

```python
@dataclass
class FPConfig(ConfigMixin):
    """FP 模式配置"""
    f_base: float = 60.0

    # 頻率偏移量 (Hz)，需按升序排列
    f1: float = -0.5    # 最低頻率偏移
    f2: float = -0.25
    f3: float = -0.02   # 死區下限
    f4: float = 0.02    # 死區上限
    f5: float = 0.25
    f6: float = 0.5     # 最高頻率偏移

    # 功率百分比 (%)，需按降序排列
    p1: float = 100.0   # f1 時的功率 (最大放電)
    p2: float = 52.0
    p3: float = 9.0     # 死區內功率
    p4: float = -9.0
    p5: float = -52.0
    p6: float = -100.0  # f6 時的功率 (最大充電)
```

頻率-功率曲線的物理意義是：

```
功率 (%)
  100 |●
      |  \
   52 |   ●
      |     \
    9 |      ●──●
      |          \
  -52 |           ●
      |             \
 -100 |              ●
      +──┬──┬──┬──┬──┬──→ 頻率偏移 (Hz)
       -0.5 -0.25 0  0.25 0.5
```

- 頻率低於正常（<60Hz）→ 放電補償（P > 0）
- 頻率高於正常（>60Hz）→ 充電吸收（P < 0）
- 死區內（59.98~60.02Hz）→ 小量或零輸出

FPStrategy 的 `execute()` 方法展示了一個重要的模式——**百分比到絕對值的轉換**：

```python
def execute(self, context: StrategyContext) -> Command:
    frequency = context.extra.get("frequency")
    if frequency is None:
        return context.last_command

    p_percent = self._calculate_power(frequency)

    # 使用 system_base 轉換
    if context.system_base is not None:
        p_kw = context.percent_to_kw(p_percent)
    else:
        p_kw = p_percent  # 無 system_base 時直接輸出百分比

    return Command(p_target=p_kw, q_target=0.0)
```

### 5.4 IslandModeStrategy：離網模式

離網模式是最能體現生命週期管理重要性的策略：

```python
class IslandModeStrategy(Strategy):
    """離網模式策略 (Grid Forming / Island Mode)"""

    def __init__(self, relay: RelayProtocol, config: Optional[IslandModeConfig] = None):
        self._relay = relay
        self._config = config or IslandModeConfig()

    @property
    def execution_config(self) -> ExecutionConfig:
        # TRIGGERED 模式——不主動執行
        return ExecutionConfig(mode=ExecutionMode.TRIGGERED, interval_seconds=1)

    def execute(self, context: StrategyContext) -> Command:
        # 離網模式不發送功率命令，維持現狀
        return context.last_command

    async def on_activate(self) -> None:
        """啟用：切離 ACB"""
        await self._relay.set_open()

    async def on_deactivate(self) -> None:
        """停用：等待 sync_ok 後搭接 ACB"""
        timeout = self._config.sync_timeout
        elapsed = 0.0
        check_interval = 0.5

        while elapsed < timeout:
            if self._relay.sync_ok:
                break
            await asyncio.sleep(check_interval)
            elapsed += check_interval

        if not self._relay.sync_ok:
            logger.critical(
                f"等待 sync_ok 超時 ({timeout}s)，請手動處理 ACB"
            )
        else:
            await self._relay.set_close()
```

這個策略的 `execute()` 幾乎什麼都不做——真正的邏輯在 `on_activate()` 和 `on_deactivate()` 中：

- **啟用**：切開 ACB，系統進入孤島模式
- **停用**：等待 PCS 的 V/F 同步信號，確認後才閉合 ACB 重新併網

`on_deactivate()` 中的等待同步邏輯是一個典型的工業控制場景——你不能瞬間切換，必須等待物理系統達到穩定狀態。

### 5.5 BypassStrategy：旁路模式

Bypass 模式讓操作人員可以手動控制設備，控制器完全「放手」：

```python
class BypassStrategy(Strategy):
    """旁路策略：完全不發送任何指令"""

    @property
    def suppress_heartbeat(self) -> bool:
        return True  # 暫停心跳，讓設備知道控制器已釋放控制權

    @property
    def execution_config(self) -> ExecutionConfig:
        return ExecutionConfig(mode=ExecutionMode.TRIGGERED, interval_seconds=1)

    def execute(self, context: StrategyContext) -> Command:
        return context.last_command
```

關鍵設計：`suppress_heartbeat = True`。當控制器進入 Bypass 模式時，它不只停止發送功率指令，還停止心跳寫入。設備端的 watchdog 會因此觸發，進入安全模式——這是一個明確的「控制權移交」信號。

### 5.6 StopStrategy：停機模式

最簡單的策略，但也是最重要的安全機制之一：

```python
class StopStrategy(Strategy):
    """停止策略：輸出 P=0, Q=0"""

    @property
    def execution_config(self) -> ExecutionConfig:
        return ExecutionConfig(mode=ExecutionMode.PERIODIC, interval_seconds=1)

    def execute(self, context: StrategyContext) -> Command:
        return Command(p_target=0.0, q_target=0.0)
```

注意 StopStrategy 和 BypassStrategy 的區別：

| 特性 | StopStrategy | BypassStrategy |
|------|-------------|----------------|
| 輸出 | P=0, Q=0（主動歸零） | 維持上次（不送新指令） |
| 執行模式 | PERIODIC（持續送零） | TRIGGERED（不主動執行） |
| 心跳 | 不暫停（保持控制權） | 暫停（釋放控制權） |
| 用途 | 緊急停機、無排程時 | 手動調試、維護模式 |

---

## 6. ExecutionMode：三種執行模式

策略的執行方式由 `ExecutionConfig` 定義：

```python
class ExecutionMode(Enum):
    PERIODIC = auto()    # 固定週期執行
    TRIGGERED = auto()   # 僅在外部觸發時執行
    HYBRID = auto()      # 週期執行，但可被提前觸發


@dataclass(frozen=True)
class ExecutionConfig:
    mode: ExecutionMode
    interval_seconds: int = 1
```

### 6.1 PERIODIC

適用於需要持續控制的策略。`StrategyExecutor` 每隔 `interval_seconds` 秒執行一次 `execute()`：

```python
# AFC 每秒計算一次功率
return ExecutionConfig(mode=ExecutionMode.PERIODIC, interval_seconds=1)
```

### 6.2 TRIGGERED

適用於只在特定事件發生時才需要執行的策略。`StrategyExecutor` 會等待外部呼叫 `trigger()`：

```python
# 離網模式不主動執行，由外部事件觸發
return ExecutionConfig(mode=ExecutionMode.TRIGGERED, interval_seconds=1)
```

### 6.3 HYBRID

結合兩者——正常時按週期執行，但可以被 `trigger()` 提前觸發。適合需要即時響應但也要定期更新的場景：

```python
# 排程策略：每 5 秒執行一次，但排程切換時立即觸發
return ExecutionConfig(mode=ExecutionMode.HYBRID, interval_seconds=5)
```

---

## 7. 撰寫你自己的策略

了解了抽象設計之後，讓我們實際撰寫一個自訂策略。假設需求是：**根據即時電價動態調整充放電功率**。

```python
from dataclasses import dataclass
from csp_lib.controller.core import (
    Strategy, StrategyContext, Command,
    ExecutionConfig, ExecutionMode, ConfigMixin
)


@dataclass
class DynamicPricingConfig(ConfigMixin):
    """動態電價策略配置"""
    max_discharge_kw: float = 500.0   # 最大放電功率
    max_charge_kw: float = 500.0      # 最大充電功率
    price_threshold_high: float = 3.0  # 高電價門檻 (元/kWh)
    price_threshold_low: float = 1.5   # 低電價門檻 (元/kWh)
    soc_min_for_discharge: float = 20.0  # 放電最低 SOC


class DynamicPricingStrategy(Strategy):
    """
    動態電價策略

    根據 context.extra["electricity_price"] 決定充放電：
    - 電價 > high_threshold → 放電（賣電套利）
    - 電價 < low_threshold → 充電（低價囤電）
    - 中間 → 待機
    """

    def __init__(self, config: DynamicPricingConfig | None = None):
        self._config = config or DynamicPricingConfig()

    @property
    def execution_config(self) -> ExecutionConfig:
        # 每 5 秒執行一次，但可由外部提前觸發（例如電價更新時）
        return ExecutionConfig(mode=ExecutionMode.HYBRID, interval_seconds=5)

    def execute(self, context: StrategyContext) -> Command:
        price = context.extra.get("electricity_price")
        if price is None:
            return context.last_command  # 無電價資料，維持上次

        soc = context.soc
        cfg = self._config

        # 高電價：放電
        if price >= cfg.price_threshold_high:
            if soc is not None and soc < cfg.soc_min_for_discharge:
                return Command(p_target=0.0, q_target=0.0)  # SOC 太低不放
            # 依電價比例決定放電功率
            ratio = min((price - cfg.price_threshold_high) / cfg.price_threshold_high, 1.0)
            p = cfg.max_discharge_kw * ratio
            return Command(p_target=p, q_target=0.0)

        # 低電價：充電
        if price <= cfg.price_threshold_low:
            ratio = min((cfg.price_threshold_low - price) / cfg.price_threshold_low, 1.0)
            p = -cfg.max_charge_kw * ratio  # 充電為負值
            return Command(p_target=p, q_target=0.0)

        # 中間區：待機
        return Command(p_target=0.0, q_target=0.0)
```

### 使用方式

```python
# 註冊策略
config = DynamicPricingConfig(
    max_discharge_kw=500.0,
    price_threshold_high=3.0,
    price_threshold_low=1.5,
)
controller.register_mode(
    "dynamic_pricing",
    DynamicPricingStrategy(config),
    ModePriority.SCHEDULE,
)
await controller.set_base_mode("dynamic_pricing")
```

### 可測試性

由於策略只依賴 `StrategyContext`，測試完全不需要 mock 設備：

```python
def test_dynamic_pricing_high_price():
    strategy = DynamicPricingStrategy(DynamicPricingConfig(
        max_discharge_kw=500.0,
        price_threshold_high=3.0,
    ))

    context = StrategyContext(
        soc=80.0,
        extra={"electricity_price": 4.5},
    )

    command = strategy.execute(context)
    assert command.p_target > 0  # 應該放電
    assert command.q_target == 0.0


def test_dynamic_pricing_no_data():
    strategy = DynamicPricingStrategy()
    last = Command(p_target=100.0, q_target=50.0)

    context = StrategyContext(last_command=last)
    command = strategy.execute(context)

    # 無電價資料時應維持上次指令
    assert command == last
```

---

## 8. 重點回顧

1. **Strategy 抽象提供了四個擴展點**：`execute()`（必須）、`execution_config`（必須）、`on_activate()`/`on_deactivate()`（可選）。這四個擴展點覆蓋了工業控制策略的所有需求。

2. **StrategyContext 是策略的唯一資訊來源**。固定欄位（`soc`、`last_command`、`system_base`）處理最常見的資訊，`extra` 字典處理策略特定的輸入。`percent_to_kw()` 等輔助方法統一了常見的轉換邏輯。

3. **Command 是 frozen 的不可變物件**。`with_p()` / `with_q()` 提供安全的修改 API。不可變性確保了保護鏈可以正確追蹤修改歷史。

4. **六種內建策略覆蓋主要場景**：PQ（定功率）、QV（電壓調節）、FP（頻率調節）、Island（離網）、Bypass（旁路）、Stop（停機）。每個策略都是 `Strategy` 的具體實現，可以單獨使用也可以組合使用。

5. **三種執行模式適配不同場景**：PERIODIC 用於持續控制，TRIGGERED 用於事件驅動，HYBRID 兩者兼顧。

6. **自訂策略簡單且可測試**。繼承 `Strategy`，實作兩個方法，就能融入 csp_lib 的整個控制迴路——包括保護鏈、模式管理、功率分配。

---

## 下篇預告

有了策略之後，下一個問題是：誰來決定什麼時候用哪個策略？答案是 `ModeManager`——一個支援 base mode 和 override 堆疊的模式管理器。下篇我們將深入解析遠端指令如何驅動模式切換，以及優先權機制如何確保安全策略永遠不會被低優先權模式覆蓋。
