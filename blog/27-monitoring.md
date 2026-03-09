# 監控與告警：建立工業級的可觀測性

> **從零打造工業控制協定框架：給軟體工程師的 CSP Library 實戰系列**
>
> Part 6 — 系統整合篇 | Article 27
>
> [<<< 上一篇：部署策略](./26-deployment.md) | [下一篇：實戰案例 >>>](./28-ems-case-study.md)

---

## 目錄

1. [工業系統的可觀測性三支柱](#工業系統的可觀測性三支柱)
2. [系統資源監控](#系統資源監控)
3. [階層式健康檢查](#階層式健康檢查)
4. [告警管理](#告警管理)
5. [告警持久化與歷史](#告警持久化與歷史)
6. [通知系統](#通知系統)
7. [集中式日誌](#集中式日誌)
8. [重點回顧](#重點回顧)
9. [下篇預告](#下篇預告)

---

## 工業系統的可觀測性三支柱

如果你有後端開發的經驗，你大概聽過「可觀測性三支柱」（Three Pillars of Observability）：Metrics、Logs、Traces。在 Web 應用中，你可能用 Prometheus 收指標、用 ELK 收日誌、用 Jaeger 追蹤分散式請求。

工業控制系統的可觀測性需求類似，但側重點完全不同：

| 面向 | Web 應用 | 工業控制系統 |
|------|----------|-------------|
| **Metrics** | QPS、延遲、錯誤率 | 設備健康、通訊品質、功率輸出 |
| **Logs** | 請求日誌、應用日誌 | 控制迴路日誌、告警歷史、命令記錄 |
| **Events** | HTTP 錯誤、DB 連線失敗 | 告警觸發/解除、模式切換、保護動作 |

在工業場域，「Traces」的重要性相對較低（控制迴路不是分散式請求鏈），取而代之的是「Events」——特別是告警事件和狀態變化事件。

更根本的差異在於：**Web 應用的可觀測性主要用來除錯，工業系統的可觀測性直接關乎安全**。一個未被偵測到的通訊異常可能導致控制器對過時的資料做出錯誤決策；一個被忽略的溫度告警可能最終導致設備損壞。

---

## 系統資源監控

csp_lib 的 `monitor` 模組（Layer 8）提供系統資源的即時監控。它依賴 `psutil` 來收集主機的 CPU、記憶體、磁碟和網路指標。

### SystemMetricsCollector

```python
from csp_lib.monitor import SystemMetricsCollector, SystemMetrics

collector = SystemMetricsCollector()
metrics: SystemMetrics = collector.collect()

print(f"CPU: {metrics.cpu_percent}%")
print(f"RAM: {metrics.memory_used_mb:.0f}MB / {metrics.memory_total_mb:.0f}MB")
print(f"Disk: {metrics.disk_used_percent}%")
```

`SystemMetrics` 是一個包含以下資訊的資料類別：

| 欄位 | 型別 | 說明 |
|------|------|------|
| `cpu_percent` | `float` | CPU 使用率（%） |
| `memory_total_mb` | `float` | 總記憶體（MB） |
| `memory_used_mb` | `float` | 已用記憶體（MB） |
| `memory_percent` | `float` | 記憶體使用率（%） |
| `disk_used_percent` | `float` | 磁碟使用率（%） |

### 網路介面指標

對工業控制系統來說，網路品質是至關重要的指標。如果 Modbus TCP 連線的封包丟失率上升，控制迴路的品質會直接下降。

```python
from csp_lib.monitor import InterfaceMetrics

# 取得特定網路介面的指標
interface: InterfaceMetrics = collector.collect_interface("eth0")
print(f"TX bytes: {interface.bytes_sent}")
print(f"RX bytes: {interface.bytes_recv}")
print(f"TX errors: {interface.errs_out}")
print(f"RX errors: {interface.errs_in}")
```

### 模組健康收集

除了系統資源，`ModuleHealthCollector` 可以收集各個應用模組的運行狀態：

```python
from csp_lib.monitor import ModuleHealthCollector, ModuleHealthSnapshot, ModuleStatus

health_collector = ModuleHealthCollector()

# 註冊需要監控的模組
health_collector.register("device_manager", device_manager)
health_collector.register("alarm_manager", alarm_manager)

# 收集所有模組的健康快照
snapshots: list[ModuleHealthSnapshot] = health_collector.collect()
for snap in snapshots:
    print(f"{snap.name}: {snap.status}")
    # ModuleStatus.RUNNING / STOPPED / ERROR
```

### SystemMonitor：統一入口

`SystemMonitor` 整合了系統資源收集和模組健康監控，並提供定時收集和告警評估：

```python
from csp_lib.monitor import SystemMonitor, MonitorConfig, MetricThresholds

config = MonitorConfig(
    collect_interval=5.0,  # 每 5 秒收集一次
    thresholds=MetricThresholds(
        cpu_warning=80.0,     # CPU > 80% 觸發警告
        cpu_critical=95.0,    # CPU > 95% 觸發嚴重告警
        memory_warning=80.0,
        memory_critical=95.0,
        disk_warning=85.0,
        disk_critical=95.0,
    ),
)

monitor = SystemMonitor(config)
async with monitor:
    # monitor 會在背景定期收集指標
    await asyncio.Event().wait()
```

### 叢集監控聚合

在 HA 或分散式部署中，`ClusterMonitorAggregator` 可以聚合多個節點的指標：

```python
from csp_lib.monitor import ClusterMonitorAggregator, ClusterHealthSnapshot

aggregator = ClusterMonitorAggregator()

# 各節點透過 Redis 發布指標
# 聚合器收集所有節點的健康狀態
cluster_health: ClusterHealthSnapshot = aggregator.get_snapshot()
for node in cluster_health.nodes:
    print(f"Node {node.node_id}: CPU={node.cpu_percent}%")
```

---

## 階層式健康檢查

csp_lib 的健康檢查系統採用階層式設計。`HealthReport` 可以包含子報告，形成一棵健康狀態樹：

```python
from csp_lib.core.health import HealthCheckable, HealthStatus, HealthReport

class HealthReport:
    status: HealthStatus       # HEALTHY / DEGRADED / UNHEALTHY
    component: str             # 元件名稱
    message: str               # 訊息
    details: dict[str, Any]    # 詳細資訊
    children: list[HealthReport]  # 子元件的健康報告
```

### SystemController 的健康階層

`SystemController` 實作了 `health()` 方法，自動聚合所有已註冊設備的健康狀態：

```python
controller = SystemController(registry, config)
report = controller.health()
```

產生的報告結構類似：

```
SystemController: DEGRADED
├── PCS_1: HEALTHY
│   ├── communication: HEALTHY (latency=12ms)
│   └── alarms: HEALTHY (active=0)
├── PCS_2: UNHEALTHY
│   ├── communication: UNHEALTHY (timeout, 3 consecutive failures)
│   └── alarms: HEALTHY
├── BMS: HEALTHY
│   ├── communication: HEALTHY (latency=8ms)
│   └── alarms: HEALTHY
└── Meter: HEALTHY
    ├── communication: HEALTHY (latency=5ms)
    └── alarms: HEALTHY
```

### 狀態聚合規則

`SystemController` 的健康狀態聚合遵循以下規則：

```python
def health(self) -> HealthReport:
    children = [dev.health() for dev in self._registry.all_devices]
    if all(c.status == HealthStatus.HEALTHY for c in children):
        status = HealthStatus.HEALTHY
    elif any(c.status == HealthStatus.UNHEALTHY for c in children):
        status = HealthStatus.UNHEALTHY
    else:
        status = HealthStatus.DEGRADED
    return HealthReport(
        status=status,
        component="system_controller",
        details={"mode": self.effective_mode_name,
                 "alarmed": list(self._alarmed_devices)},
        children=children,
    )
```

簡單來說：
- **所有子元件 HEALTHY** → 系統 HEALTHY
- **任一子元件 UNHEALTHY** → 系統 UNHEALTHY
- **其他情況** → 系統 DEGRADED

這個「最差狀態上卷」的策略確保了：**任何一台設備出問題，系統級的健康狀態都會反映出來**。運維人員看到系統 DEGRADED 時，可以展開子節點找到具體是哪台設備出了問題。

### HealthCheckable 協定

任何類別都可以透過實作 `health()` 方法來支援健康檢查：

```python
from csp_lib.core.health import HealthCheckable, HealthReport, HealthStatus

@runtime_checkable
class HealthCheckable(Protocol):
    def health(self) -> HealthReport: ...
```

因為使用了 `@runtime_checkable` 的 `Protocol`，你不需要顯式繼承——只要你的類別有 `health()` 方法且回傳 `HealthReport`，它就自動滿足 `HealthCheckable` 協定。

---

## 告警管理

csp_lib 的告警系統是一個完整的狀態機，涵蓋告警定義、評估、狀態管理和遲滯處理。

### AlarmDefinition：定義告警

```python
from csp_lib.equipment.alarm import AlarmDefinition, AlarmLevel, HysteresisConfig

# 溫度過高告警
over_temp = AlarmDefinition(
    code="OVER_TEMP",
    name="溫度過高",
    level=AlarmLevel.ALARM,        # INFO / WARNING / ALARM
    hysteresis=HysteresisConfig(
        activate_threshold=3,       # 連續 3 次超標才觸發
        clear_threshold=5,          # 連續 5 次正常才解除
    ),
    description="電池模組溫度超過上限",
)
```

三個告警等級的語義：

| 等級 | 說明 | 系統反應 |
|------|------|---------|
| `INFO` | 資訊性告警 | 僅記錄，不影響運行 |
| `WARNING` | 警告告警 | 記錄 + 通知，不影響控制 |
| `ALARM` | 重大告警 | 記錄 + 通知 + 可能觸發保護動作 |

### HysteresisConfig：告警防抖

遲滯（Hysteresis）是工業告警系統中最重要的概念之一。想像一個溫度感測器的讀數在 79.8 到 80.2 之間波動，而告警閾值是 80 度。沒有遲滯的話，告警會不斷觸發和解除——這就是所謂的「告警抖動」（alarm chattering）。

```
時間軸：  t1   t2   t3   t4   t5   t6   t7   t8
溫度：   79.5 80.1 79.9 80.3 80.2 80.5 79.8 79.5
                                              │
無遲滯：  -    ON   OFF  ON   ON   ON   OFF  OFF  ← 抖動！
                                              │
遲滯(3/5): -    -    -    ON   ON   ON   ON   ON   ← 穩定
```

`activate_threshold=3` 表示溫度必須**連續** 3 次超標才會觸發告警；`clear_threshold=5` 表示溫度必須**連續** 5 次回到正常範圍才會解除告警。解除閾值通常設得比觸發閾值更高，因為誤解除比誤觸發的風險更大。

### AlarmEvaluator：告警評估器

csp_lib 提供了多種告警評估器：

```python
from csp_lib.equipment.alarm import (
    ThresholdAlarmEvaluator,   # 閾值比較
    BitMaskAlarmEvaluator,     # 位元遮罩
    TableAlarmEvaluator,       # 查表
    Operator,
    ThresholdCondition,
)

# 閾值告警：溫度 > 80
temp_evaluator = ThresholdAlarmEvaluator(
    alarm=over_temp,
    condition=ThresholdCondition(
        operator=Operator.GREATER_THAN,
        threshold=80.0,
    ),
)

# 位元遮罩告警：狀態字的第 3 位為 1 時觸發
fault_evaluator = BitMaskAlarmEvaluator(
    alarm=AlarmDefinition(code="PCS_FAULT", name="PCS 故障", level=AlarmLevel.ALARM),
    bit_mask=0x0008,  # 第 3 位
)

# 查表告警：特定故障碼 → 告警
table_evaluator = TableAlarmEvaluator(
    alarm_table={
        1: AlarmDefinition(code="E001", name="過壓", level=AlarmLevel.ALARM),
        2: AlarmDefinition(code="E002", name="欠壓", level=AlarmLevel.WARNING),
        3: AlarmDefinition(code="E003", name="過流", level=AlarmLevel.ALARM),
    },
)
```

### AlarmStateManager：狀態管理

`AlarmStateManager` 維護每個告警的狀態機，處理遲滯邏輯和狀態轉換：

```python
from csp_lib.equipment.alarm import AlarmStateManager, AlarmEvent, AlarmEventType

state_manager = AlarmStateManager()

# 每次讀取新值後，評估告警並更新狀態
events: list[AlarmEvent] = state_manager.evaluate(evaluators, current_values)

for event in events:
    if event.event_type == AlarmEventType.TRIGGERED:
        print(f"告警觸發: {event.alarm.name}")
    elif event.event_type == AlarmEventType.CLEARED:
        print(f"告警解除: {event.alarm.name}")
```

---

## 告警持久化與歷史

告警發生後需要記錄下來，用於事後分析、合規報告和趨勢追蹤。csp_lib 的 `AlarmPersistenceManager`（Layer 5）負責這件事。

```python
from csp_lib.manager import (
    AlarmPersistenceManager,
    AlarmPersistenceConfig,
    AlarmRecord,
    AlarmRepository,
    MongoAlarmRepository,
)

# 使用 MongoDB 作為儲存後端
repo = MongoAlarmRepository(db=mongo_db, collection_name="alarms")

config = AlarmPersistenceConfig(
    # 配置項目...
)

alarm_manager = AlarmPersistenceManager(config=config, repository=repo)
```

`AlarmPersistenceManager` 訂閱設備的告警事件，自動將告警記錄寫入 MongoDB。每筆 `AlarmRecord` 包含：

| 欄位 | 說明 |
|------|------|
| `device_id` | 觸發告警的設備 ID |
| `alarm_code` | 告警代碼 |
| `alarm_name` | 告警名稱 |
| `level` | 告警等級 |
| `status` | 當前狀態（觸發/解除） |
| `triggered_at` | 觸發時間 |
| `cleared_at` | 解除時間（若已解除） |
| `duration` | 持續時間 |

如果你的專案不使用 MongoDB，可以實作自己的 `AlarmRepository` 介面來對接其他儲存後端（PostgreSQL、InfluxDB 等）。

---

## 通知系統

告警記錄只是第一步。在工業場域，**關鍵告警必須即時通知到人**——否則值班工程師可能渾然不知系統已經出了問題。

### NotificationChannel：通知通道抽象

```python
from csp_lib.notification import NotificationChannel, Notification, NotificationEvent

class NotificationChannel:
    """通知通道抽象基礎類別"""

    async def send(self, notification: Notification) -> None:
        """發送通知"""
        ...
```

你需要實作自己的 `NotificationChannel`。常見的選擇包括 LINE Notify、Telegram Bot、Email、SMS 等。

### NotificationDispatcher：即時分發

```python
from csp_lib.notification import NotificationDispatcher

# 建立分發器，註冊多個通道
dispatcher = NotificationDispatcher()
dispatcher.add_channel(line_channel)
dispatcher.add_channel(telegram_channel)

# 發送通知（會同時發送到所有已註冊的通道）
await dispatcher.send(notification)
```

### NotificationBatcher：批次通知

在工業場域，最令人頭痛的問題之一是**告警風暴**（alarm flood）。當系統出現嚴重問題時，可能在幾秒內產生數十個甚至數百個告警。如果每個告警都發一條通知，值班工程師的手機會被淹沒，反而無法辨識真正重要的訊息。

`NotificationBatcher` 解決了這個問題：

```python
from csp_lib.notification import NotificationBatcher, BatchNotificationConfig

batcher_config = BatchNotificationConfig(
    # 批次通知配置...
)

batcher = NotificationBatcher(config=batcher_config, channels=[line_channel])
```

`NotificationBatcher` 的運作方式：
1. **防抖（Debounce）**：在指定的時間窗口內，合併重複的告警通知
2. **去重**：相同的告警不會重複通知
3. **分組**：將同一時間窗口內的多個告警合併成一條摘要通知

### 整合到 AlarmPersistenceManager

`NotificationDispatcher` 和 `NotificationBatcher` 都實作了 `NotificationSender` 協定。你可以將它們傳入 `AlarmPersistenceManager`，告警觸發時會自動發送通知：

```python
alarm_manager = AlarmPersistenceManager(
    config=alarm_config,
    repository=alarm_repo,
    notification_sender=batcher,  # 或 dispatcher
)
```

### 非告警事件通知

除了告警，你可能還想通知一些重要的系統事件——例如模式切換、系統啟動/停止、HA 主備切換等：

```python
from csp_lib.notification import EventCategory, EventNotification

# 定義事件通知
event = EventNotification(
    category=EventCategory.SYSTEM,
    title="模式切換",
    message="系統模式從 PQ 切換為 QV",
)
```

---

## 集中式日誌

csp_lib 使用 [loguru](https://github.com/Delgan/loguru) 作為日誌框架，並提供了模組級的日誌控制。

### get_logger：模組級 Logger

```python
from csp_lib.core import get_logger

logger = get_logger("csp_lib.integration.system_controller")
logger.info("SystemController started.")
logger.warning("Device alarm activated: pcs_1")
logger.error("Communication timeout: bms_1")
```

每個模組都應該使用自己的 logger 名稱。這樣你可以在不修改程式碼的情況下，透過 `set_level()` 動態調整各模組的日誌等級：

```python
from csp_lib.core import set_level

# 只對 mongo 模組開啟 DEBUG（其他模組維持 INFO）
set_level("DEBUG", module="csp_lib.mongo")

# 全域設定為 WARNING（減少日誌量）
set_level("WARNING")
```

### 模組級等級匹配

`set_level()` 支援前綴匹配。例如設定 `csp_lib.integration` 的等級，會同時影響 `csp_lib.integration.system_controller`、`csp_lib.integration.registry` 等所有子模組。

匹配規則採用「最長前綴優先」：

```python
set_level("WARNING", module="csp_lib")                    # 全域
set_level("DEBUG", module="csp_lib.integration")           # integration 子樹
set_level("INFO", module="csp_lib.integration.registry")   # registry 特例

# 結果：
# csp_lib.modbus → WARNING（匹配 csp_lib）
# csp_lib.integration.system_controller → DEBUG（匹配 csp_lib.integration）
# csp_lib.integration.registry → INFO（匹配精確路徑）
```

### 日誌格式

預設的日誌格式清楚顯示時間、等級和來源模組：

```
2026-03-10 14:30:25 | INFO     | csp_lib.integration.system_controller | SystemController started.
2026-03-10 14:30:26 | WARNING  | csp_lib.equipment.device | Communication retry: pcs_1 (attempt 2/3)
2026-03-10 14:30:27 | ERROR    | csp_lib.equipment.device | Communication timeout: pcs_1
```

### 自訂日誌格式

```python
from csp_lib.core import configure_logging

configure_logging(
    level="INFO",
    format_string=(
        "{time:YYYY-MM-DD HH:mm:ss.SSS} | "
        "{level: <8} | "
        "{extra[module]} | "
        "{message}"
    ),
)
```

### 生產環境建議

1. **日誌檔案輪替**：使用 loguru 的 `add()` 方法將日誌輸出到檔案，搭配自動輪替
2. **結構化日誌**：考慮輸出 JSON 格式的日誌，方便後續用 ELK 或 Loki 收集分析
3. **敏感資料過濾**：確保日誌中不包含密碼、token 等敏感資訊
4. **控制日誌量**：生產環境建議使用 `INFO` 等級，僅在除錯時才開啟 `DEBUG`

---

## 重點回顧

1. **可觀測性三支柱**在工業系統中的側重點不同：Metrics 關注設備健康和通訊品質，Logs 關注控制決策和告警歷史，Events 取代了 Traces 的位置。

2. **SystemMetricsCollector** 收集 CPU、RAM、磁碟、網路指標；**SystemMonitor** 整合資源監控和模組健康檢查，支援閾值告警。

3. **階層式健康檢查**透過 `HealthReport` 的遞迴結構，從系統到設備到通訊鏈路，提供完整的健康狀態樹。狀態聚合採用「最差狀態上卷」策略。

4. **告警系統**包含定義（`AlarmDefinition`）、評估（`AlarmEvaluator`）、狀態管理（`AlarmStateManager`）三個環節。遲滯配置（`HysteresisConfig`）是避免告警抖動的關鍵。

5. **通知系統**提供即時分發（`NotificationDispatcher`）和批次合併（`NotificationBatcher`）兩種模式，有效對抗告警風暴。

6. **loguru 日誌**提供模組級等級控制，支援前綴匹配和最長前綴優先的匹配規則。

---

## 下篇預告

理論講得差不多了，是時候動手了。下一篇是本系列的壓軸——我們將從零開始，用 csp_lib 的所有層級打造一座 **1MW 儲能系統的 EMS（能源管理系統）**。從設備範本定義、DeviceRegistry 配置、SystemController 組裝，到 HA 部署和監控告警，走完工業控制專案的全生命週期。

> [下一篇：實戰案例：打造一座 1MW 儲能系統的 EMS >>>](./28-ems-case-study.md)
