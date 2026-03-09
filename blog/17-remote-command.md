# 遠端指令與模式管理：ModeManager 深入解析

> **Part 4 — 控制迴路篇 | Article 17**
>
> 系列：從零打造工業控制協定框架：給軟體工程師的 CSP Library 實戰系列

---

## 前言

上一篇我們看到了如何用 Strategy Pattern 封裝各種控制策略。但一個儲能系統在實際運行中，需要在多種策略之間切換：白天用 PQ 定功率放電、下午用 AFC 做頻率調節、設備告警時自動停機、維護人員要求進入旁路模式......

誰來管理這些切換？切換的規則是什麼？如果兩個請求同時到達，誰優先？

這就是 `ModeManager` 要解決的問題。

---

## 1. 為什麼模式管理很重要

在沒有模式管理的系統中，你可能會寫出這樣的程式碼：

```python
# 危險的做法
if alarm_triggered:
    current_strategy = StopStrategy()
elif maintenance_requested:
    current_strategy = BypassStrategy()
elif schedule_says_afc:
    current_strategy = FPStrategy(config)
else:
    current_strategy = PQModeStrategy(default_config)
```

這段程式碼有幾個致命問題：

1. **優先權隱含在 if-elif 順序中**——新增模式時容易搞錯
2. **沒有切換通知機制**——舊策略的 `on_deactivate()` 不會被呼叫
3. **Override 難以管理**——維護模式結束後怎麼「回到之前的狀態」？
4. **多來源衝突**——排程系統和操作員同時下令時誰贏？

`ModeManager` 用一個乾淨的 base mode + override stack 模型解決了這些問題。

---

## 2. ModeManager 的核心模型

ModeManager 的設計理念是：**一個 base mode 處理正常運行，override 堆疊處理臨時插入**。

```
┌─────────────────────────────────────┐
│           Override Stack            │
│                                     │
│   ┌─────────────────────────────┐   │
│   │ __auto_stop__ (priority=101)│ ← 最高優先權
│   ├─────────────────────────────┤   │
│   │ bypass (priority=50)       │   │
│   └─────────────────────────────┘   │
│                                     │
├─────────────────────────────────────┤
│           Base Mode                 │
│                                     │
│   ┌─────────────────────────────┐   │
│   │ pq (priority=10)           │ ← 正常運行
│   └─────────────────────────────┘   │
└─────────────────────────────────────┘
```

規則很簡單：
- 有 override 時 → 取 override 中 priority 最高的那個
- 無 override 時 → 使用 base mode
- base mode 也可以是空的（此時策略為 None，不執行任何動作）

### 2.1 資料結構

每個註冊的模式由一個不可變的 `ModeDefinition` 描述：

```python
@dataclass(frozen=True)
class ModeDefinition:
    """模式定義"""
    name: str           # 唯一名稱
    strategy: Strategy  # 策略實例
    priority: int       # 優先等級（數值越大越優先）
    description: str = ""
```

ModeManager 內部維護三個狀態：

```python
class ModeManager:
    def __init__(self):
        self._modes: dict[str, ModeDefinition] = {}   # 所有已註冊模式
        self._base_mode_names: list[str] = []          # 基礎模式列表
        self._override_names: list[str] = []           # 活躍的 override 列表
```

---

## 3. ModePriority：優先等級設計

csp_lib 定義了三個預設的優先等級：

```python
class ModePriority(IntEnum):
    SCHEDULE = 10      # 排程控制
    MANUAL = 50        # 操作員手動控制
    PROTECTION = 100   # 安全保護
```

自動停機模式的優先權更高：

```python
_AUTO_STOP_MODE = "__auto_stop__"
# priority = ModePriority.PROTECTION + 1 = 101
```

這構成了一個清晰的優先權階梯：

| 優先權 | 模式 | 說明 |
|--------|------|------|
| 101 | `__auto_stop__` | 系統告警自動停機（最高，自動管理） |
| 100 | PROTECTION | 保護模式（如離網模式由 ACB 跳脫觸發） |
| 50 | MANUAL | 操作員手動覆寫 |
| 10 | SCHEDULE | 排程或遠端指令（正常運行） |

### 3.1 優先權的物理意義

這個數字不是隨便定的。它反映了儲能系統的安全層級：

- **安全保護 > 操作員 > 排程**。不論排程怎麼設定，操作員都可以覆寫。不論操作員怎麼操作，安全保護都可以覆寫。

- 自動停機 (101) > 保護模式 (100) 的原因是：即使系統正在離網模式運行，如果發生嚴重告警（例如電池過溫），仍然要強制停機。

---

## 4. SystemController 的模式管理 API

`SystemController` 將 `ModeManager` 的能力包裝成簡潔的 API：

### 4.1 基本用法：註冊與切換

```python
from csp_lib.integration.system_controller import SystemController, SystemControllerConfig
from csp_lib.integration import DeviceRegistry
from csp_lib.controller.strategies.pq_strategy import PQModeStrategy, PQModeConfig
from csp_lib.controller.strategies.bypass_strategy import BypassStrategy
from csp_lib.controller.strategies.fp_strategy import FPStrategy, FPConfig
from csp_lib.controller.system import ModePriority

# 建立 SystemController
registry = DeviceRegistry()
config = SystemControllerConfig(...)
controller = SystemController(registry, config)

# 註冊模式
controller.register_mode(
    "pq",
    PQModeStrategy(PQModeConfig(p=500.0)),
    ModePriority.SCHEDULE,
    description="定功率放電 500kW",
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
```

### 4.2 Base Mode 操作

```python
# 設定基礎模式
await controller.set_base_mode("pq")
# 現在生效的策略是 PQModeStrategy

# 切換基礎模式
await controller.set_base_mode("afc")
# PQModeStrategy.on_deactivate() 被呼叫
# FPStrategy.on_activate() 被呼叫
```

`set_base_mode()` 會自動觸發策略的生命週期管理——這是 ModeManager 透過 `_notify_change` 回呼通知 SystemController 完成的。

### 4.3 Override 操作

Override 是臨時的高優先權模式覆蓋：

```python
# 操作員要求進入維護模式
await controller.push_override("bypass")
# bypass 的優先權 (50) > pq 的優先權 (10)
# 生效策略變為 BypassStrategy
# 心跳服務被暫停（suppress_heartbeat=True）

# ... 維護作業 ...

# 維護結束，恢復正常控制
await controller.pop_override("bypass")
# 生效策略自動回到 base mode (PQModeStrategy)
# 心跳服務恢復
```

Override 的關鍵特性：
- **可以堆疊多個**：多個 override 同時活躍時，取 priority 最高的
- **不影響 base mode**：pop 之後自動恢復
- **同一個 override 不能重複推入**

```python
# 多重 override 範例
await controller.push_override("bypass")     # bypass (50) 生效
await controller.push_override("island")     # island (100) 生效（更高優先權）
await controller.pop_override("island")      # bypass (50) 重新生效
await controller.pop_override("bypass")      # 回到 base mode
```

### 4.4 策略切換時發生了什麼

當 ModeManager 偵測到 effective strategy 改變時，會觸發一連串動作：

```python
# SystemController 內部
async def _on_strategy_change(self, old: Strategy | None, new: Strategy | None):
    # 1. 解析實際該使用的策略
    resolved = self._resolve_strategy()

    # 2. 通知 StrategyExecutor 切換策略
    #    這會觸發 old.on_deactivate() → new.on_activate()
    await self._executor.set_strategy(resolved)

    # 3. 根據策略的 suppress_heartbeat 控制心跳
    if self._heartbeat is not None and resolved is not None:
        if resolved.suppress_heartbeat:
            self._heartbeat.pause()
        else:
            self._heartbeat.resume()
```

---

## 5. CascadingStrategy：多策略組合

有時候你需要同時運行多個策略。例如：PQ 策略控制 P，QV 策略控制 Q。csp_lib 透過多 base mode 和 `CascadingStrategy` 支援這個場景。

### 5.1 啟用多 base mode

```python
# 註冊兩個不同層面的策略
controller.register_mode(
    "pq", PQModeStrategy(PQModeConfig(p=600.0)), ModePriority.SCHEDULE
)
controller.register_mode(
    "qv", QVStrategy(QVConfig(nominal_voltage=380.0)), ModePriority.SCHEDULE
)

# 同時啟用兩個 base mode
await controller.set_base_mode("pq")
await controller.add_base_mode("qv")  # 新增而非替換
```

### 5.2 CascadingStrategy 的級聯邏輯

當多個 base mode 同時活躍且配置了 `capacity_kva` 時，SystemController 會自動建立 `CascadingStrategy`：

```python
# SystemControllerConfig 中設定容量上限
config = SystemControllerConfig(
    capacity_kva=1000.0,  # 最大 1000 kVA
    ...
)
```

CascadingStrategy 使用 delta-based clamping 確保高優先權策略的輸出不被修改：

```
第 1 層 (PQ): 輸出 P=600kW, Q=0
    累積: P=600kW, Q=0, S=600kVA

第 2 層 (QV): 想加 Q=900kVar
    新 S = √(600² + 900²) = 1082 kVA > 1000 kVA
    → 只縮放 QV 的 delta Q，不動 PQ 的 P
    → Q 限制 ≈ 800kVar

    最終: P=600kW, Q=800kVar, S≈1000kVA
```

這個設計保證了：
1. PQ 策略的 P=600kW 不會被 QV 策略的加入而改變
2. 總視在功率不超過設備額定容量
3. 後加入的策略只能使用「剩餘容量」

---

## 6. EventDrivenOverride：自動模式切換

除了手動 push/pop，csp_lib 還支援基於條件的自動 override——這是工業控制中極為重要的功能。

### 6.1 EventDrivenOverride 協定

```python
@runtime_checkable
class EventDrivenOverride(Protocol):
    @property
    def name(self) -> str:
        """對應 ModeManager 中的已註冊模式名稱"""
        ...

    @property
    def cooldown_seconds(self) -> float:
        """條件解除後的冷卻時間（防抖動）"""
        ...

    def should_activate(self, context: StrategyContext) -> bool:
        """評估是否應啟用此 override"""
        ...
```

SystemController 在每個控制週期中會評估所有已註冊的 EventDrivenOverride，自動管理 push/pop。

### 6.2 內建實作：AlarmStopOverride

自動停機是最常見的事件驅動 override。csp_lib 預設就會註冊它：

```python
class AlarmStopOverride:
    """告警自動停機"""

    def __init__(self, name: str = "__auto_stop__", alarm_key: str = "system_alarm"):
        self._name = name
        self._alarm_key = alarm_key

    @property
    def name(self) -> str:
        return self._name

    @property
    def cooldown_seconds(self) -> float:
        return 0.0  # 立即回復

    def should_activate(self, context: StrategyContext) -> bool:
        return context.extra.get(self._alarm_key, False) is True
```

在 SystemController 的建構子中：

```python
if config.auto_stop_on_alarm:
    self._mode_manager.register(
        _AUTO_STOP_MODE,
        StopStrategy(),
        ModePriority.PROTECTION + 1,  # 101，最高優先權
        "Auto stop on system alarm",
    )
    self.register_event_override(
        AlarmStopOverride(name=_AUTO_STOP_MODE, alarm_key=config.system_alarm_key)
    )
```

這意味著：當任何設備觸發告警時，系統會自動推入 StopStrategy（P=0, Q=0），且它的優先權 (101) 高於任何其他模式。告警解除後自動回復原來的控制模式。

### 6.3 ContextKeyOverride：通用條件觸發

`ContextKeyOverride` 是更通用的事件驅動 override 實作：

```python
class ContextKeyOverride:
    """根據 context.extra 中的 key 值觸發 override"""

    def __init__(
        self,
        name: str,
        context_key: str,
        activate_when: Callable[[Any], bool],
        cooldown_seconds: float = 5.0,
    ):
        self._name = name
        self._context_key = context_key
        self._activate_when = activate_when
        self._cooldown_seconds = cooldown_seconds

    def should_activate(self, context: StrategyContext) -> bool:
        value = context.extra.get(self._context_key)
        if value is None:
            return False
        return self._activate_when(value)
```

### 6.4 實際案例：ACB 跳脫自動進入離網模式

```python
from csp_lib.controller.system.event_override import ContextKeyOverride

# 1. 註冊離網模式
controller.register_mode(
    "island",
    IslandModeStrategy(relay, IslandModeConfig()),
    ModePriority.PROTECTION,
    description="離網模式",
)

# 2. 註冊自動切換條件
acb_override = ContextKeyOverride(
    name="island",
    context_key="acb_tripped",
    activate_when=lambda v: v is True,
    cooldown_seconds=2.0,  # ACB 恢復後等 2 秒再退出離網
)
controller.register_event_override(acb_override)
```

當 `context.extra["acb_tripped"]` 變為 `True` 時：
1. `ContextKeyOverride.should_activate()` 回傳 `True`
2. SystemController 自動呼叫 `push_override("island")`
3. IslandModeStrategy 的 `on_activate()` 被呼叫 → 切離 ACB

當 `acb_tripped` 變回 `False` 後：
1. 等待 `cooldown_seconds`（2 秒）確認穩定
2. 自動呼叫 `pop_override("island")`
3. IslandModeStrategy 的 `on_deactivate()` 被呼叫 → 等待同步後併網

### 6.5 防抖動機制

`cooldown_seconds` 是一個重要的設計。在工業環境中，信號可能會「彈跳」——ACB 狀態可能在短時間內快速切換。如果每次變化都觸發模式切換，系統會不斷進出離網模式，造成設備損耗。

cooldown 確保條件**持續**解除一段時間後才真正退出 override，防止抖動。

---

## 7. ProtectionGuard：最後的安全防線

即使模式管理確保了正確的策略在運行，策略的輸出仍然需要經過保護規則的檢查。

### 7.1 保護規則鏈

ProtectionGuard 按順序執行所有保護規則，每條規則可以修改或維持 Command：

```python
class ProtectionGuard:
    def apply(self, command: Command, context: StrategyContext) -> ProtectionResult:
        original = command
        current = command
        triggered = []

        for rule in self._rules:
            try:
                current = rule.evaluate(current, context)
                if rule.is_triggered:
                    triggered.append(rule.name)
            except Exception:
                # 規則本身出錯 → fail-safe
                current = Command(p_target=0.0, q_target=0.0)
                triggered.append(f"{rule.name}(fail-safe)")

        return ProtectionResult(
            original_command=original,
            protected_command=current,
            triggered_rules=triggered,
        )
```

### 7.2 三種內建保護規則

**SOCProtection**——防止過充過放：

```python
class SOCProtection(ProtectionRule):
    """
    P > 0 = 放電，P < 0 = 充電

    - SOC >= soc_high: 禁止充電 → clamp P >= 0
    - SOC <= soc_low: 禁止放電 → clamp P <= 0
    - 警戒區漸進限制
    """

    def evaluate(self, command: Command, context: StrategyContext) -> Command:
        soc = context.soc
        if soc is None:
            return command  # 無 SOC 資料時不介入

        # SOC 過高：禁止充電
        if soc >= self._config.soc_high:
            if command.p_target < 0:
                return command.with_p(0.0)
            return command

        # SOC 過低：禁止放電
        if soc <= self._config.soc_low:
            if command.p_target > 0:
                return command.with_p(0.0)
            return command

        # 警戒區：漸進限制
        warning_high = self._config.soc_high - self._config.warning_band
        if soc >= warning_high and command.p_target < 0:
            ratio = (self._config.soc_high - soc) / self._config.warning_band
            return command.with_p(command.p_target * ratio)

        return command
```

**ReversePowerProtection**——防止逆送：

```python
class ReversePowerProtection(ProtectionRule):
    """
    約束: p_target <= meter_power + threshold
    """

    def evaluate(self, command: Command, context: StrategyContext) -> Command:
        meter_power = context.extra.get(self._meter_power_key)
        if meter_power is None or command.p_target < 0:
            return command  # 充電不受限

        max_discharge = meter_power + self._threshold
        if max_discharge < 0:
            max_discharge = 0.0

        if command.p_target > max_discharge:
            return command.with_p(max_discharge)

        return command
```

**SystemAlarmProtection**——系統告警強制歸零：

```python
class SystemAlarmProtection(ProtectionRule):
    """system_alarm == True → 強制 P=0, Q=0"""

    def evaluate(self, command: Command, context: StrategyContext) -> Command:
        if context.extra.get(self._alarm_key, False):
            return Command(p_target=0.0, q_target=0.0)
        return command
```

### 7.3 保護規則的順序

保護規則的執行順序很重要。典型的配置是：

```python
config = SystemControllerConfig(
    protection_rules=[
        SOCProtection(SOCProtectionConfig(soc_high=95, soc_low=5)),
        ReversePowerProtection(threshold=0.0),
        SystemAlarmProtection(),
    ],
)
```

1. 先做 SOC 保護（限制充放電範圍）
2. 再做逆送保護（限制放電上限）
3. 最後做告警保護（最嚴格，直接歸零）

每一層規則都是在上一層的輸出上進一步限制。

---

## 8. 重點回顧

1. **ModeManager 用 base mode + override stack 模型管理策略切換**。Base mode 處理正常運行，override 處理臨時覆蓋。Pop override 後自動恢復。

2. **優先權反映安全層級**：SCHEDULE (10) < MANUAL (50) < PROTECTION (100) < AUTO_STOP (101)。高優先權永遠可以覆蓋低優先權。

3. **策略切換有完整的生命週期管理**。`on_deactivate()` → `on_activate()` 的呼叫順序保證了策略可以安全地做初始化和清理。

4. **CascadingStrategy 支援多策略同時運行**。Delta-based clamping 確保高優先權策略的輸出不被後續策略修改，且總功率不超過設備容量。

5. **EventDrivenOverride 實現自動模式切換**。`AlarmStopOverride` 處理告警停機，`ContextKeyOverride` 處理通用條件觸發。`cooldown_seconds` 防止信號抖動。

6. **ProtectionGuard 是最後的安全防線**。SOC 保護、逆送保護、告警保護按順序執行，保護規則出錯時 fail-safe 歸零。

---

## 下篇預告

到目前為止，我們討論的都是單機控制。但實際的儲能系統通常包含多台 PCS 和 BMS。下一篇我們將探討：如何把一個系統級的 Command 分配到多台設備？功率要怎麼按額定容量比例分配？心跳服務如何確保每台設備都知道控制器還活著？
