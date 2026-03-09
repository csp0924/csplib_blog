# 組裝的智慧：Factory × Registry × Mediator × Facade

> **從零打造工業控制協定框架：給軟體工程師的 CSP Library 實戰系列**
>
> Part 0.5 — 設計模式活用篇 | Article 00h
>
> [<<< 上一篇：反應的藝術：Observer × Chain of Responsibility × State](./00g-patterns-reactive.md) | [下一篇：從 REST API 到 Modbus：當後端工程師遇上工業協定 >>>](./01-rest-to-modbus.md)

---

## 目錄

1. [開場：大型系統的組裝挑戰](#開場大型系統的組裝挑戰)
2. [Factory Pattern — 設備的標準化生產線](#factory-pattern--設備的標準化生產線)
3. [Registry Pattern — 設備的電話簿](#registry-pattern--設備的電話簿)
4. [Mediator Pattern — 系統控制器作為中央協調者](#mediator-pattern--系統控制器作為中央協調者)
5. [Facade Pattern — 簡化複雜子系統的門面](#facade-pattern--簡化複雜子系統的門面)
6. [Proxy + Lazy Init — 延遲初始化的連線代理](#proxy--lazy-init--延遲初始化的連線代理)
7. [22 種模式大總表](#22-種模式大總表)
8. [重點回顧](#重點回顧)
9. [系列正文預告：從 Part 1 開始](#系列正文預告從-part-1-開始)

---

## 開場：大型系統的組裝挑戰

想像你是一座儲能電站的系統工程師。你手上有：

- 4 台 PCS（功率調節系統），型號 SUN2000-100KTL
- 16 組 BMS（電池管理系統），每 4 組對應 1 台 PCS
- 1 台電表
- 1 台環境監測器

每台設備都有幾十個暫存器要讀取，各有不同的告警規則，需要連接不同的 Modbus 位址。而且，你不只在一個案場部署——下個案場可能是 8 台 PCS + 32 組 BMS，型號還不一樣。

問題來了：如何讓建立設備的過程**標準化**且**可重複**？如何讓幾十台設備被**統一查詢**和**統一控制**？如何讓使用者面對的是一個簡潔的控制介面，而不是內部幾十個元件之間錯綜複雜的依賴關係？

這就是「組裝模式」要解決的問題。本篇介紹六個模式，它們在 csp_lib 中分工合作：

- **Factory**：標準化設備的建立流程
- **Registry**：提供設備的查詢索引
- **Mediator**：協調多個子系統之間的互動
- **Facade**：為複雜系統提供簡潔的外部介面
- **Proxy + Lazy Init**：延遲昂貴資源的初始化

---

## Factory Pattern — 設備的標準化生產線

### 日常比喻

特斯拉的超級工廠裡，每台 Model 3 都從同一條產線出來。工廠不需要知道這台車會被賣到哪裡、車主是誰——它只負責根據 BOM（物料清單）組裝出一台符合規格的車。如果客戶要紅色，產線參數調一下；如果是出口版，方向盤換個位置。但核心的組裝流程是標準化的。

Factory 模式就是：**把物件的建立過程封裝起來，呼叫者只需要提供配置，不需要知道建立的細節。**

### 問題場景

你需要建立一台 `AsyncModbusDevice`。它需要：讀取點位、寫入點位、告警評估器、聚合管線、能力綁定、設備配置、Modbus 客戶端。而且同一型號的設備，點位定義是一樣的，只有 device_id 和 unit_id 不同。

你的第一直覺：

```python
# 反面教材：每次手動建立設備
pcs_1 = AsyncModbusDevice(
    config=DeviceConfig(device_id="pcs_1", unit_id=1),
    client=tcp_client,
    always_points=(p_point, q_point, soc_point, status_point),
    rotating_points=((temp_1, temp_2), (voltage_1, voltage_2)),
    write_points=(p_cmd, q_cmd, start_cmd, stop_cmd),
    alarm_evaluators=(soc_alarm, temp_alarm, fault_alarm),
    aggregator_pipeline=pipeline,
    capability_bindings=(pq_binding, heartbeat_binding),
)

# 第二台 PCS...複製貼上，改 device_id 和 unit_id
pcs_2 = AsyncModbusDevice(
    config=DeviceConfig(device_id="pcs_2", unit_id=2),
    client=tcp_client,
    always_points=(p_point, q_point, soc_point, status_point),  # 完全一樣
    # ... 二十行一模一樣的程式碼
)
```

### 直覺解法的問題

- 同型號的設備要複製貼上幾十次
- 修改一個點位定義需要改所有設備的建立程式碼
- 如果設備的暫存器位址有固定步幅（如每台偏移 100），手動計算容易出錯
- 沒有「設備範本」的概念——型號的知識散落在建構式呼叫中

### 模式登場：Factory

將「型號的定義」（Template）和「實例的建立」（Factory）分離。Template 定義了型號的共通特徵，Factory 負責根據 Template 和個別配置生產實例。

### csp_lib 真實程式碼

csp_lib 用兩個層級實現了 Factory 模式：`EquipmentTemplate`（範本）和 `DeviceFactory`（工廠）。

首先是範本——一個 frozen dataclass：

```python
# csp_lib/equipment/template/definition.py

@dataclass(frozen=True)
class EquipmentTemplate:
    """不可變設備模型範本"""
    model: str
    always_points: tuple[ReadPoint, ...] = ()
    rotating_points: tuple[tuple[ReadPoint, ...], ...] = ()
    write_points: tuple[WritePoint, ...] = ()
    alarm_evaluators: tuple[AlarmEvaluator, ...] = ()
    aggregator_pipeline: AggregatorPipeline | None = None
    capability_bindings: tuple[CapabilityBinding, ...] = ()
    description: str = ""

    def __post_init__(self) -> None:
        """驗證所有能力綁定的點位名稱確實存在於範本中"""
        if not self.capability_bindings:
            return
        read_names: set[str] = set()
        for p in self.always_points:
            read_names.add(p.name)
        for group in self.rotating_points:
            for p in group:
                read_names.add(p.name)
        write_names = {p.name for p in self.write_points}

        for binding in self.capability_bindings:
            cap = binding.capability
            for slot in cap.write_slots:
                actual = binding.point_map[slot]
                if actual not in write_names:
                    raise ConfigurationError(
                        f"Template '{self.model}': capability '{cap.name}' slot '{slot}' "
                        f"maps to write point '{actual}' which does not exist."
                    )
```

注意 `__post_init__` 的驗證邏輯：建立範本時就檢查能力綁定的點位是否存在。這是**Fail Fast**原則——在部署前就抓到配置錯誤，而不是在運行中才發現某個寫入點位不存在。

然後是工廠：

```python
# csp_lib/equipment/template/factory.py

class DeviceFactory:
    """從 EquipmentTemplate 建立 AsyncModbusDevice 實例"""

    @staticmethod
    def create(
        template: EquipmentTemplate,
        config: DeviceConfig,
        client: AsyncModbusClientBase,
        *,
        overrides: dict[str, PointOverride] | None = None,
        address_offset: int = 0,
    ) -> AsyncModbusDevice:
        always_points = template.always_points
        rotating_points = template.rotating_points
        write_points = template.write_points
        alarm_evaluators = template.alarm_evaluators

        # 套用覆寫（重新命名點位）
        if overrides:
            name_map = _build_name_map(overrides)
            always_points = _apply_overrides_to_read_points(always_points, overrides)
            rotating_points = tuple(
                _apply_overrides_to_read_points(group, overrides)
                for group in rotating_points
            )
            write_points = _apply_overrides_to_write_points(write_points, overrides)
            alarm_evaluators = _apply_overrides_to_evaluators(alarm_evaluators, name_map)

        # 套用位址偏移
        if address_offset != 0:
            always_points = _apply_offset_to_read_points(always_points, address_offset)
            rotating_points = tuple(
                _apply_offset_to_read_points(group, address_offset)
                for group in rotating_points
            )
            write_points = _apply_offset_to_write_points(write_points, address_offset)

        return AsyncModbusDevice(
            config=config,
            client=client,
            always_points=always_points,
            rotating_points=rotating_points,
            write_points=write_points,
            alarm_evaluators=alarm_evaluators,
            aggregator_pipeline=template.aggregator_pipeline,
            capability_bindings=template.capability_bindings,
        )
```

`create()` 方法做了兩件巧妙的事：

1. **Point Override**：同一個範本可以在不同實例中重新命名點位。這在 IO 模組場景中很常見——同一型號的 IO 板，接不同感測器時，`input_0` 在一台設備裡叫 `temperature`，在另一台叫 `humidity`。
2. **Address Offset**：同型號設備的暫存器位址通常有固定偏移。例如 BMS_1 起始位址 1000，BMS_2 起始位址 1100，偏移量 100。

批次建立更加強大：

```python
    @staticmethod
    def create_stride(
        template: EquipmentTemplate,
        base_config: DeviceConfig,
        client_factory: Callable[[DeviceConfig], AsyncModbusClientBase],
        count: int,
        stride: int,
        *,
        id_format: str = "{base_id}_{index}",
        overrides: dict[str, PointOverride] | None = None,
    ) -> list[AsyncModbusDevice]:
        """固定步幅批次建立設備"""
        instances = [
            replace(base_config, device_id=id_format.format(
                base_id=base_config.device_id, index=i + 1
            ))
            for i in range(count)
        ]
        offsets = [i * stride for i in range(count)]

        return DeviceFactory.create_batch(
            template=template,
            instances=instances,
            client_factory=client_factory,
            overrides=overrides,
            address_offsets=offsets,
        )
```

一行程式碼建立 16 組 BMS：

```python
bms_devices = DeviceFactory.create_stride(
    template=bms_template,
    base_config=DeviceConfig(device_id="bms", unit_id=1),
    client_factory=lambda cfg: shared_client,
    count=16,
    stride=100,  # 每台偏移 100 個暫存器
)
# 產出: bms_1 (offset=0), bms_2 (offset=100), ..., bms_16 (offset=1500)
```

注意 `client_factory` 參數的設計：工廠不直接接收客戶端實例，而是接收一個**工廠函式**。這讓你可以根據配置決定客戶端——有些設備共用同一條 TCP 連線，有些各自獨立。

### 如果不用 Factory

```python
# 反面教材：手動建立 16 台 BMS
bms_1 = AsyncModbusDevice(config=..., always_points=(
    replace(soc_point, address=soc_point.address + 0),
    replace(voltage_point, address=voltage_point.address + 0),
    ...
))
bms_2 = AsyncModbusDevice(config=..., always_points=(
    replace(soc_point, address=soc_point.address + 100),
    replace(voltage_point, address=voltage_point.address + 100),
    ...
))
# ... 複製貼上 16 次，祈禱不要算錯偏移量
```

### 何時不該用

- **只建立一個實例**：如果一個型號只有一台設備，直接建構比引入 Template + Factory 更簡單。
- **建立邏輯經常變動**：如果每台設備的建立流程都不一樣，Factory 無法標準化，反而增加了抽象層的負擔。
- **配置極其簡單**：如果物件只有兩三個參數，Factory 的價值有限。

### 練習題

1. `_apply_offset_to_read_points()` 是如何使用 `dataclasses.replace()` 實現不可變更新的？
2. `create_batch()` 中，如果 `address_offsets` 長度與 `instances` 不一致會怎樣？

---

## Registry Pattern — 設備的電話簿

### 日常比喻

你的手機通訊錄就是一個 Registry。你不需要記住每個人的電話號碼，只需要搜尋名字就能找到。通訊錄還支援分組——「家人」、「同事」、「客戶」——讓你能按類別查詢。

Registry 模式就是：**一個集中式的索引，讓你能用各種條件快速找到需要的物件。**

### 問題場景

你有 21 台設備（4 PCS + 16 BMS + 1 電表）。控制策略需要：

- 找到「所有 PCS」來下發功率指令
- 找到「所有有回應的 BMS」來計算總 SOC
- 找到「具備 PQ 控制能力的設備」來分配功率

你的第一直覺：

```python
# 反面教材：到處維護設備列表
pcs_list = [pcs_1, pcs_2, pcs_3, pcs_4]
bms_list = [bms_1, bms_2, ..., bms_16]
meter = meter_1

def get_responsive_pcs():
    return [p for p in pcs_list if p.is_responsive]

def get_all_bms():
    return bms_list

# 每個需要查詢設備的模組都要拿到這些列表
# 新增一台設備？改所有引用這些列表的地方
```

### 直覺解法的問題

- 設備列表的引用散落在多個模組中
- 新增/移除設備需要修改所有引用
- 沒有統一的「按特徵查詢」機制
- 無法在 runtime 動態新增設備

### 模式登場：Registry

一個集中式索引，維護雙向映射：設備 ID ↔ 設備實例，設備 ID ↔ traits（標籤）。所有需要查詢設備的模組都指向同一個 Registry。

### csp_lib 真實程式碼

`csp_lib/integration/registry.py` 的 `DeviceRegistry` 是一個精心設計的雙向索引：

```python
# csp_lib/integration/registry.py

class DeviceRegistry:
    """Trait-based 設備查詢索引"""

    def __init__(self) -> None:
        self._devices: dict[str, AsyncModbusDevice] = {}      # device_id → 設備實例
        self._device_traits: dict[str, set[str]] = {}          # device_id → traits
        self._trait_devices: dict[str, set[str]] = {}          # trait → device_ids
        self._metadata: dict[str, dict[str, Any]] = {}         # device_id → 靜態 metadata
```

四張索引表，各司其職。`_devices` 是正向索引（ID → 物件），`_trait_devices` 是反向索引（trait → IDs）。

註冊設備：

```python
    def register(
        self,
        device: AsyncModbusDevice,
        traits: list[str] | None = None,
        metadata: dict[str, Any] | None = None,
    ) -> None:
        did = device.device_id
        if did in self._devices:
            raise ValueError(f"Device '{did}' is already registered.")
        self._devices[did] = device
        self._device_traits[did] = set()
        self._metadata[did] = dict(metadata) if metadata else {}
        for trait in traits or []:
            self._add_trait_index(did, trait)
```

`raise ValueError` 而非靜默覆蓋——這是防禦性設計。在工業系統中，兩台設備用了同一個 ID 是嚴重的配置錯誤，必須大聲喊出來。

查詢 API 設計得很有層次：

```python
    def get_device(self, device_id: str) -> AsyncModbusDevice | None:
        """依 ID 查詢設備"""
        return self._devices.get(device_id)

    def get_devices_by_trait(self, trait: str) -> list[AsyncModbusDevice]:
        """依 trait 查詢所有設備（按 device_id 排序，確保確定性）"""
        ids = self._trait_devices.get(trait, set())
        return [self._devices[did] for did in sorted(ids)]

    def get_responsive_devices_by_trait(self, trait: str) -> list[AsyncModbusDevice]:
        """依 trait 查詢所有 is_responsive=True 的設備"""
        return [d for d in self.get_devices_by_trait(trait) if d.is_responsive]

    def get_first_responsive_device_by_trait(self, trait: str) -> AsyncModbusDevice | None:
        """依 trait 取得第一台 responsive 設備"""
        devices = self.get_responsive_devices_by_trait(trait)
        return devices[0] if devices else None
```

幾個值得注意的設計決策：

1. **排序保證確定性**：`sorted(ids)` 確保多次呼叫返回相同順序。在控制系統中，確定性（determinism）比效能更重要。
2. **漸進式過濾**：`get_devices_by_trait` → `get_responsive_devices_by_trait` → `get_first_responsive_device_by_trait`，每一層加一個過濾條件。
3. **Capability 查詢**：除了字串 trait，還支援結構化的 `Capability` 查詢：

```python
    def get_devices_with_capability(self, capability: Capability | str) -> list[AsyncModbusDevice]:
        """取得具備指定能力的所有設備"""
        return sorted(
            [d for d in self._devices.values() if d.has_capability(capability)],
            key=lambda d: d.device_id,
        )

    def get_responsive_devices_with_capability(
        self, capability: Capability | str
    ) -> list[AsyncModbusDevice]:
        """取得具備指定能力且 responsive 的設備"""
        return [d for d in self.get_devices_with_capability(capability) if d.is_responsive]
```

Trait 和 Capability 的差別：Trait 是外部標籤（「這台設備是 PCS」），Capability 是內部能力（「這台設備支援 PQ 控制」）。一個是運維人員的分類邏輯，一個是設備自身的功能描述。

雙向索引的內部維護：

```python
    def _add_trait_index(self, device_id: str, trait: str) -> None:
        """建立 device_id ↔ trait 的雙向索引"""
        self._device_traits[device_id].add(trait)
        if trait not in self._trait_devices:
            self._trait_devices[trait] = set()
        self._trait_devices[trait].add(device_id)

    def _remove_trait_index(self, device_id: str, trait: str) -> None:
        """移除 device_id ↔ trait 的雙向索引"""
        self._device_traits[device_id].discard(trait)
        if trait in self._trait_devices:
            self._trait_devices[trait].discard(device_id)
            if not self._trait_devices[trait]:
                del self._trait_devices[trait]  # 清理空集合
```

清理空集合 (`del self._trait_devices[trait]`) 是防止記憶體洩漏的好習慣——移除最後一台 PCS 後，`"pcs"` 這個 trait 的索引也應該被刪除。

### 如果不用 Registry

```python
# 反面教材：每個模組各自維護設備引用
class ControlStrategy:
    def __init__(self, pcs_list, bms_list, meter):
        self._pcs_list = pcs_list  # 建構式注入
        self._bms_list = bms_list
        self._meter = meter

class AlarmManager:
    def __init__(self, all_devices):
        self._devices = all_devices  # 又是另一份列表

class Dashboard:
    def __init__(self, pcs_list, bms_list, meter):
        self._pcs_list = pcs_list  # 又是另一份列表
        # 新增一台設備？改三個模組的建構式
```

### 何時不該用

- **設備數量極少且固定**：3 台設備直接用變數引用就好，Registry 的索引維護成本不值得。
- **不需要按條件查詢**：如果你永遠只需要「取得所有設備」，一個 list 就夠了。
- **多執行緒環境**：csp_lib 的 `DeviceRegistry` 沒有加鎖，因為它是單執行緒 asyncio 環境。如果你在多執行緒環境用，需要自己加鎖。

### 練習題

1. `DeviceRegistry.__contains__()` 讓你可以用 `"pcs_1" in registry` 語法，它的實作是什麼？
2. `unregister()` 是如何確保雙向索引一致性的？

---

## Mediator Pattern — 系統控制器作為中央協調者

### 日常比喻

機場的塔台是一個 Mediator。飛機不會直接跟其他飛機溝通（「嘿，737，我要降落了，你讓一下」），而是全部透過塔台協調。塔台知道每架飛機的位置、狀態、意圖，統一排程起降順序。沒有塔台的話，每架飛機都要和其他所有飛機通訊——N 架飛機需要 N(N-1)/2 條通訊鏈路。

Mediator 模式就是：**元件之間不直接通訊，而是透過一個中央協調者統籌調度。**

### 問題場景

你的系統有這些元件：

- `ContextBuilder`：從設備讀取值建構策略上下文
- `StrategyExecutor`：執行當前的控制策略
- `ProtectionGuard`：套用保護規則
- `CommandRouter`：將命令路由到設備
- `ModeManager`：管理多種控制模式的優先權
- `HeartbeatService`：維持與設備的心跳

如果沒有 Mediator，這些元件需要直接互相引用：

```python
# 反面教材：元件之間直接依賴
executor.set_context_provider(context_builder)
executor.set_on_command(lambda cmd: protection.apply(cmd)
    .then(lambda result: router.route(result.protected_command)))
mode_manager.set_on_change(lambda old, new: executor.set_strategy(new))
# ... 每加一個元件，連線指數增長
```

### 模式登場：Mediator

`SystemController` 作為中央協調者，所有元件只和它互動。它知道所有元件的存在，負責在適當的時機呼叫適當的元件。

### csp_lib 真實程式碼

`csp_lib/integration/system_controller.py` 的 `SystemController` 是一個經典的 Mediator：

```python
# csp_lib/integration/system_controller.py

class SystemController(AsyncLifecycleMixin):
    def __init__(self, registry: DeviceRegistry, config: SystemControllerConfig) -> None:
        self._registry = registry
        self._config = config

        # 模式管理
        self._mode_manager = ModeManager(on_strategy_change=self._on_strategy_change)

        # 保護鏈
        self._protection_guard = ProtectionGuard(config.protection_rules)

        # Context 建構器
        self._context_builder = ContextBuilder(
            registry,
            config.context_mappings,
            system_base=config.system_base,
            capability_mappings=config.capability_context_mappings or None,
        )

        # Command 路由器
        self._command_router = CommandRouter(
            registry,
            config.command_mappings,
            capability_mappings=config.capability_command_mappings or None,
        )

        # 策略執行器
        self._executor = StrategyExecutor(
            context_provider=self._build_context,
            on_command=self._on_command,
        )
```

注意 `SystemController` 是如何將自己的方法（`self._build_context`、`self._on_command`、`self._on_strategy_change`）注入到各個元件中的。元件不需要知道彼此——它們只知道「有一個回呼會被呼叫」。

控制流程的核心協調邏輯：

```python
    def _build_context(self) -> StrategyContext:
        """建構策略上下文，注入 system_alarm 旗標"""
        context = self._context_builder.build()

        # 檢查所有設備的告警狀態
        has_alarm = any(dev.is_protected for dev in self._registry.all_devices)

        if self._config.alarm_mode == "per_device":
            context.extra[self._config.system_alarm_key] = False
        else:
            context.extra[self._config.system_alarm_key] = has_alarm

        self._cached_context = context
        return context

    async def _on_command(self, command: Command) -> None:
        """命令回呼：套用保護鏈 → 處理告警 → 路由到設備"""
        context = self._cached_context
        if context is None:
            context = self._build_context()

        # 套用保護鏈
        result = self._protection_guard.apply(command, context)
        protected_command = result.protected_command

        # 評估事件驅動 overrides
        if self._event_overrides:
            await self._evaluate_event_overrides(context)

        # 路由到設備
        if self._config.power_distributor is not None:
            snapshots = self._build_device_snapshots()
            per_device = self._config.power_distributor.distribute(
                protected_command, snapshots
            )
            await self._command_router.route_per_device(protected_command, per_device)
        else:
            await self._command_router.route(protected_command)
```

這段程式碼清楚展示了 Mediator 的價值：控制流程是 `ContextBuilder → StrategyExecutor → ProtectionGuard → CommandRouter`，但這四個元件都不知道彼此的存在。它們的協調完全由 `SystemController` 負責。

策略變更時的協調更能看出 Mediator 的角色：

```python
    async def _on_strategy_change(self, old: Strategy | None, new: Strategy | None) -> None:
        """ModeManager 通知策略變更"""
        resolved = self._resolve_strategy()
        logger.info(f"Strategy change: {old} -> {resolved}")
        await self._executor.set_strategy(resolved)

        # 依據策略的 suppress_heartbeat 控制心跳服務
        if self._heartbeat is not None and resolved is not None:
            if resolved.suppress_heartbeat:
                self._heartbeat.pause()
            else:
                self._heartbeat.resume()
```

ModeManager 說「策略變了」→ SystemController 通知 StrategyExecutor 換策略 → 同時通知 HeartbeatService 暫停或恢復。ModeManager 不知道 HeartbeatService 的存在，HeartbeatService 也不知道 ModeManager 的存在。Mediator 在中間做了跨元件的協調。

### 如果不用 Mediator

```
ContextBuilder ←──→ StrategyExecutor ←──→ ProtectionGuard
       ↑                    ↑                      ↑
       │                    │                      │
       ├────────→ ModeManager ←──────→ CommandRouter
       │                    ↑                      │
       │                    │                      │
       └──→ HeartbeatService ←─────────────────────┘

每個元件需要知道其他 N-1 個元件，修改一個介面影響所有人
```

有 Mediator：

```
ContextBuilder ──→ SystemController ←── ModeManager
                        ↑ ↓
ProtectionGuard ──→ SystemController ←── HeartbeatService
                        ↑ ↓
CommandRouter ────→ SystemController ←── StrategyExecutor

每個元件只認識 SystemController
```

### 何時不該用

- **元件很少**：如果只有 2-3 個元件，直接依賴比引入 Mediator 更清晰。
- **Mediator 變成 God Object**：如果所有邏輯都塞進 Mediator，它就變成了一個「上帝物件」——什麼都知道、什麼都做。需要把邏輯委派給專門的元件。
- **效能敏感**：所有通訊都經過 Mediator 的間接層會增加延遲。在微秒級的控制迴路中，直接呼叫更快。

### 練習題

1. `SystemController._resolve_strategy()` 是如何根據 override 和 base mode 組合決定最終策略的？
2. `SystemController.health()` 是如何聚合所有設備的健康狀態為一個系統級報告的？

---

## Facade Pattern — 簡化複雜子系統的門面

### 日常比喻

你去飯店吃飯，只需要跟服務生說「我要一份牛排套餐」。服務生會幫你處理：點菜→廚房備料→烹飪→擺盤→上菜。你不需要知道後廚有幾個料理台、牛排要煎幾分鐘、配菜怎麼準備。服務生就是 Facade——一個簡單的介面，背後是複雜的子系統。

### SystemController 同時也是 Facade

`SystemController` 除了是 Mediator（內部協調元件），同時也是 Facade（對外提供簡潔介面）。

看它對外暴露的 API：

```python
# 使用者只需要這幾行就能啟動完整控制系統
controller = SystemController(registry, config)
controller.register_mode("pq", pq_strategy, ModePriority.SCHEDULE)
await controller.set_base_mode("pq")
async with controller:
    await asyncio.Event().wait()
```

使用者不需要知道內部有 ContextBuilder、ProtectionGuard、StrategyExecutor、CommandRouter、ModeManager、HeartbeatService 這些元件。它們全部被 `SystemController` 的 Facade 遮蔽了。

對外的模式管理 API 是委派模式：

```python
    def register_mode(self, name: str, strategy: Strategy, priority: int,
                      description: str = "") -> None:
        """註冊模式，並驗證策略所需的 capabilities"""
        required = strategy.required_capabilities
        if required:
            for cap in required:
                devices = self._registry.get_devices_with_capability(cap)
                if not devices:
                    logger.warning(
                        f"Strategy '{strategy}' requires capability '{cap.name}' "
                        "but no registered device has it."
                    )
        self._mode_manager.register(name, strategy, priority, description)

    async def set_base_mode(self, name: str | None) -> None:
        await self._mode_manager.set_base_mode(name)

    async def push_override(self, name: str) -> None:
        await self._mode_manager.push_override(name)
```

注意 `register_mode()` 比 `ModeManager.register()` 多做了一件事：驗證策略所需的 capabilities 是否有對應的設備。這就是 Facade 的價值——它不只是簡單的轉發，還加入了跨元件的驗證邏輯。

唯讀屬性提供了受控的內部窺視：

```python
    @property
    def registry(self) -> DeviceRegistry:
        return self._registry

    @property
    def effective_mode_name(self) -> str | None:
        mode = self._mode_manager.effective_mode
        return mode.name if mode is not None else None

    @property
    def protection_status(self) -> ProtectionResult | None:
        return self._protection_guard.last_result
```

使用者可以查看狀態，但無法直接操作內部元件（例如不能直接修改 ProtectionGuard 的規則，只能透過 SystemControllerConfig）。

### Mediator vs Facade 的區別

在 csp_lib 中，`SystemController` 同時扮演了兩個角色：

| 角色 | 朝向 | 功能 |
|------|------|------|
| **Mediator** | 內部 | 協調 ContextBuilder、ProtectionGuard、StrategyExecutor 等元件之間的互動 |
| **Facade** | 外部 | 對使用者暴露 `register_mode()`、`set_base_mode()` 等簡潔 API |

這在實務中很常見——頂層控制器同時是內部的協調者和外部的門面。

---

## Proxy + Lazy Init — 延遲初始化的連線代理

### 日常比喻

你辦了一張信用卡，但你不會在辦卡的時候就把信用額度全部花掉。信用卡是你的「消費代理」——它代表了你的支付能力，但實際的銀行扣款只有在你刷卡的時候才會發生。

Proxy + Lazy Init 就是：**提供一個代理物件，真正的資源到第一次使用時才初始化。**

### 問題場景

你的系統使用 pymodbus 作為 Modbus 通訊庫。但 pymodbus 是 optional dependency——不是所有使用者都需要 Modbus（有些人可能只用模擬設備開發）。如果在 `import` 時就載入 pymodbus，沒裝的人連 `import csp_lib` 都會失敗。

### csp_lib 真實程式碼

`csp_lib/modbus/clients/client.py` 展示了三層 Lazy Init：

**第一層：模組層級的延遲 import**

```python
# csp_lib/modbus/clients/client.py

# pymodbus 為 optional dependency，僅在實際使用時才載入
_AsyncModbusTcpClient: type[AsyncModbusTcpClient] | None = None
_AsyncModbusSerialClient: type[AsyncModbusSerialClient] | None = None


def _ensure_pymodbus_imported() -> None:
    """確保 pymodbus 已載入，只執行一次實際 import"""
    global _AsyncModbusTcpClient, _AsyncModbusSerialClient

    if _AsyncModbusTcpClient is not None:
        return  # 已載入，直接返回

    try:
        from pymodbus.client import (
            AsyncModbusSerialClient,
            AsyncModbusTcpClient,
        )
        _AsyncModbusTcpClient = AsyncModbusTcpClient
        _AsyncModbusSerialClient = AsyncModbusSerialClient
    except ImportError as e:
        raise ImportError(
            "Pymodbus client requires 'pymodbus' package. "
            "Install with: uv pip install csp_lib[modbus]"
        ) from e
```

模組層級的 `_AsyncModbusTcpClient` 是 `None`，直到第一次呼叫 `_ensure_pymodbus_imported()`。這意味著 `import csp_lib.modbus` 不會觸發 pymodbus 的載入——只有在實際建立客戶端時才會。

**第二層：實例層級的延遲建立**

```python
class PymodbusTcpClient(AsyncModbusClientBase):
    def __init__(self, config: ModbusTcpConfig) -> None:
        self._config = config
        self._client: AsyncModbusTcpClient | None = None  # 初始為 None

    def _get_client(self) -> AsyncModbusTcpClient:
        """取得或建立 pymodbus 客戶端"""
        if self._client is None:
            _ensure_pymodbus_imported()           # 第一層：確保 pymodbus 已載入
            assert _AsyncModbusTcpClient is not None
            self._client = _AsyncModbusTcpClient(  # 第二層：建立客戶端實例
                host=self._config.host,
                port=self._config.port,
                timeout=self._config.timeout,
                retries=0,
            )
        return self._client
```

`PymodbusTcpClient.__init__()` 不做任何 I/O——它只記住配置。真正的 pymodbus 客戶端在第一次 `_get_client()` 時才建立。這讓你可以先建構所有設備物件（配置階段），再逐一連線（啟動階段）。

**第三層：共用資源的 Singleton per Port**

RTU 客戶端更進一步——多個 `PymodbusRtuClient` 實例如果使用同一個串口，會共用同一個底層客戶端和請求佇列：

```python
# 模組層級的共用資源池
_rtu_instances: dict[str, tuple[AsyncModbusSerialClient, ModbusRequestQueue, int]] = {}
_rtu_instances_lock = asyncio.Lock()

class PymodbusRtuClient(AsyncModbusClientBase):
    async def _acquire_resources(self):
        """取得共用的 pymodbus 客戶端和請求佇列"""
        async with _rtu_instances_lock:
            if self._port in _rtu_instances:
                client, queue, ref_count = _rtu_instances[self._port]
                _rtu_instances[self._port] = (client, queue, ref_count + 1)
                return client, queue

            # 建立新的客戶端和請求佇列
            _ensure_pymodbus_imported()
            client = _AsyncModbusSerialClient(
                port=self._config.port,
                baudrate=self._config.baudrate,
                # ...
            )
            queue = ModbusRequestQueue(self._queue_config)
            _rtu_instances[self._port] = (client, queue, 1)
            return client, queue

    async def _release_shared_resources(self) -> None:
        """釋放共用資源的參考計數"""
        async with _rtu_instances_lock:
            if self._port not in _rtu_instances:
                return
            client, queue, ref_count = _rtu_instances[self._port]
            if ref_count <= 1:
                await queue.stop()
                if client.connected:
                    client.close()
                del _rtu_instances[self._port]
            else:
                _rtu_instances[self._port] = (client, queue, ref_count - 1)
```

這是 Proxy + Lazy Init + Reference Counting 的組合：

1. **Proxy**：`PymodbusRtuClient` 是 `AsyncModbusClientBase` 的代理——讀寫操作被轉發到共用的底層客戶端
2. **Lazy Init**：底層客戶端在第一次 `connect()` 時才建立
3. **Reference Counting**：追蹤有多少個 `PymodbusRtuClient` 共用同一個底層客戶端，最後一個斷線時才真正關閉

為什麼需要 reference counting？因為 RS-485 串口一個 port 只能有一個連線。16 台 BMS 共用一個串口，但它們要能獨立 connect/disconnect。Reference counting 確保只有最後一台設備斷線時才關閉串口。

TCP 版本也有同樣的設計——`SharedPymodbusTcpClient` 用於 TCP-RS485 轉換器場景，同一個 IP:Port 的多台設備共用連線和請求佇列。

### 如果不用 Proxy + Lazy Init

```python
# 反面教材：import 時就載入所有依賴
import pymodbus  # 沒裝 pymodbus 的人直接 ImportError

# 反面教材：建構時就連線
class EagerClient:
    def __init__(self, host, port):
        self._client = AsyncModbusTcpClient(host=host, port=port)
        # 建構 20 台設備 = 建立 20 個 TCP 連線
        # 其中 16 台共用同一個串口？那就開 16 個串口... 不對，一個串口只能開一個

# 反面教材：沒有 reference counting
class NaiveSharedClient:
    _instances = {}

    async def disconnect(self):
        # 第一台設備 disconnect 就把共用連線關了
        # 其他 15 台設備全部斷線！
        del self._instances[self._port]
```

### 何時不該用

- **資源很便宜**：如果初始化成本極低（例如建立一個記憶體內的物件），Lazy Init 增加了不必要的 `if None` 檢查。
- **需要 Eager Validation**：如果你想在啟動時就驗證所有連線是否可用（Fail Fast），Lazy Init 會延遲錯誤的發現。
- **生命週期簡單**：如果每個使用者都有自己的資源（不共用），Reference Counting 的複雜度不值得。

### 練習題

1. `SharedPymodbusTcpClient` 使用 `"host:port"` 作為共用資源的 key，為什麼不直接用 host？
2. 如果 `connect()` 中建立連線失敗，`_acquire_resources()` 已經增加了 ref_count——程式碼是如何回滾的？

---

## 22 種模式大總表

綜合 Part 0.5 系列所有文章，以下是 csp_lib 中使用的設計模式總覽：

| # | 模式 | 分類 | csp_lib 位置 | 解決的問題 |
|---|------|------|-------------|-----------|
| 1 | **Strategy** | 行為型 | `controller/strategies/*.py` | 多種控制策略的可切換 |
| 2 | **Template Method** | 行為型 | `core/lifecycle.py` | AsyncLifecycleMixin 的生命週期模板 |
| 3 | **Observer** | 行為型 | `equipment/device/events.py` | 設備事件的發布訂閱 |
| 4 | **Chain of Responsibility** | 行為型 | `controller/system/protection.py` | 保護規則的鏈式套用 |
| 5 | **State** | 行為型 | `cluster/election.py` | 叢集選舉的狀態機 |
| 6 | **Command** | 行為型 | `controller/core.py` (`Command`) | 功率指令的不可變封裝 |
| 7 | **Iterator** | 行為型 | `equipment/device/scheduler.py` | 輪詢點位的排程 |
| 8 | **Adapter** | 結構型 | `modbus/codec.py` | 暫存器 ↔ Python 類型轉換 |
| 9 | **Pipeline** | 結構型 | `equipment/processing/` | 聚合器管線 |
| 10 | **Builder** | 創建型 | `integration/context_builder.py` | StrategyContext 的組裝 |
| 11 | **Factory** | 創建型 | `equipment/template/factory.py` | 設備的標準化建立 |
| 12 | **Registry** | 結構型 | `integration/registry.py` | 設備的集中查詢索引 |
| 13 | **Mediator** | 行為型 | `integration/system_controller.py` | 子系統的中央協調 |
| 14 | **Facade** | 結構型 | `integration/system_controller.py` | 複雜系統的簡潔介面 |
| 15 | **Proxy** | 結構型 | `modbus/clients/client.py` | 共用連線的代理 |
| 16 | **Lazy Init** | 創建型 | `modbus/clients/client.py` | 延遲載入 optional dependency |
| 17 | **Singleton (per key)** | 創建型 | `modbus/clients/client.py` | 每個串口/端點一個連線 |
| 18 | **Circuit Breaker** | 穩定性 | `core/resilience.py` | 故障隔離與自動恢復 |
| 19 | **Frozen Dataclass** | 結構型 | 全專案 | 不可變配置物件 |
| 20 | **Protocol** | 結構型 | `controller/protocol.py` | Runtime-checkable 介面 |
| 21 | **Event-Driven Override** | 行為型 | `controller/system/event_override.py` | 事件驅動的模式切換 |
| 22 | **Reference Counting** | 資源管理 | `modbus/clients/client.py` | 共用資源的生命週期管理 |

> 這些模式不是刻意堆砌的——每一個都是為了解決工業控制場域中的真實問題而引入的。最好的設計模式是你在解決問題的過程中「自然發現」的，而不是事先從書上挑選的。

---

## 重點回顧

| 模式 | 一句話 | csp_lib 應用 | 關鍵好處 |
|------|--------|-------------|---------|
| **Factory** | 標準化生產 | `DeviceFactory` + `EquipmentTemplate` | 一行建 16 台設備 |
| **Registry** | 集中查詢 | `DeviceRegistry` | 雙向索引、按 trait/capability 查詢 |
| **Mediator** | 中央協調 | `SystemController`（內部） | 元件解耦、流程集中 |
| **Facade** | 簡潔門面 | `SystemController`（外部） | 使用者只需 4 行程式碼 |
| **Proxy + Lazy Init** | 延遲初始化 | `PymodbusTcpClient`、`SharedPymodbusTcpClient` | optional dependency、資源共用 |

組裝模式的核心理念是：**讓簡單的事情簡單做，讓複雜的事情成為可能。** Factory 讓建立設備變成一行程式碼，Registry 讓查詢設備變成一個函式呼叫，Mediator 讓多元件協調變成一個類別的責任，Facade 讓使用者無需理解內部複雜度。

從 Part 0.5 設計模式篇的六篇文章中，我們看到了 22 種設計模式在一個真實的工業控制框架中的應用。這些模式不是學院派的理論——它們每一個都在解決部署到真實案場時遇到的真實問題：

- 設備通訊不可靠 → Circuit Breaker
- 保護規則因案場而異 → Chain of Responsibility
- 幾十台同型號設備 → Factory + Template
- 多種控制策略切換 → Strategy + ModeManager
- 事件驅動的非同步處理 → Observer

---

## 系列正文預告：從 Part 1 開始

設計模式篇到此告一段落。從下一篇開始，我們將進入系列正文——Part 1「觀念轉換篇」。

如果你是一位後端工程師，從未接觸過工業協定，Part 1 會帶你從最熟悉的 REST API 出發，逐步理解 Modbus、IEC 104、IEC 61850 這些工業協定的核心概念。你會發現，它們和你日常使用的 HTTP 有驚人的相似之處——也有關鍵的差異。

[下一篇：從 REST API 到 Modbus：當後端工程師遇上工業協定 >>>](./01-rest-to-modbus.md)
