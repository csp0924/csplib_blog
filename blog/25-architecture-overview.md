# 系統架構總覽：八層式架構的設計哲學

> **從零打造工業控制協定框架：給軟體工程師的 CSP Library 實戰系列**
>
> Part 6 — 系統整合篇 | Article 25
>
> [<<< 上一篇：Article 24](./24-prev.md) | [下一篇：部署策略 >>>](./26-deployment.md)

---

## 目錄

1. [為什麼工業系統需要分層架構？](#為什麼工業系統需要分層架構)
2. [八層式架構全貌](#八層式架構全貌)
3. [逐層解析：責任與關鍵類別](#逐層解析責任與關鍵類別)
4. [依賴方向規則：鐵律中的鐵律](#依賴方向規則鐵律中的鐵律)
5. [橫跨全局的設計模式](#橫跨全局的設計模式)
6. [可選依賴設計：按需組裝](#可選依賴設計按需組裝)
7. [資料流端到端追蹤](#資料流端到端追蹤)
8. [重點回顧](#重點回顧)
9. [下篇預告](#下篇預告)

---

## 為什麼工業系統需要分層架構？

如果你曾經開發過中大型的後端系統，你大概熟悉 MVC、Clean Architecture、Hexagonal Architecture 這些分層模式。你知道分層的目的是「關注點分離」——讓每一層只做一件事，讓變化被隔離在最小的範圍內。

但工業控制系統的分層需求，比一般的 Web 應用更加嚴格。讓我用三個理由來說明。

### 理由一：硬體多樣性是常態

一個儲能案場可能同時存在三到五種不同廠牌的設備。PCS（功率調節系統）用的是 Modbus TCP，BMS（電池管理系統）走 RS-485 串列通訊，電表又是另一家的 Modbus TCP。每台設備的暫存器表完全不同，資料格式各有各的「方言」。

如果你把通訊協定的細節和商業邏輯混在一起，每換一家設備供應商，你就得改一次控制邏輯。這在工業界是致命的——因為設備供應商的變更頻率遠高於你的想像。

### 理由二：安全性要求非同小可

工業控制系統（ICS）不像 Web 應用，最壞的情況只是資料庫被清空。一個失控的功率命令可能讓價值千萬的電池組過充爆炸，一個未經保護的控制迴路可能讓電力系統頻率失穩。

分層架構讓你在控制命令到達設備之前，必須經過層層防線——保護鏈（ProtectionGuard）、模式管理器（ModeManager）、告警系統。每一層都是一道安全閘門。

### 理由三：測試必須可以不接設備

在 Web 開發中，你可以用 Docker 啟動一個 PostgreSQL 來跑整合測試。但你不可能在 CI 環境中插一台真正的 PCS 來測試。分層架構讓你可以在高層（策略層、管理層）用 mock 物件替代底層的設備通訊，確保控制邏輯的正確性不依賴於實體設備的存在。

---

## 八層式架構全貌

csp_lib 採用八層式架構，由下而上堆疊。每一層只能依賴它下方的層級，絕對不允許反向引用。

```
┌─────────────────────────────────────────────────────────────┐
│  Layer 8  Additional                                        │
│           cluster, monitor, notification,                   │
│           modbus_server, gui, statistics                    │
├─────────────────────────────────────────────────────────────┤
│  Layer 7  Storage                                           │
│           mongo, redis                                      │
├─────────────────────────────────────────────────────────────┤
│  Layer 6  Integration                                       │
│           DeviceRegistry, ContextBuilder,                   │
│           CommandRouter, SystemController                   │
├─────────────────────────────────────────────────────────────┤
│  Layer 5  Manager                                           │
│           DeviceManager, AlarmPersistenceManager,           │
│           DataUploadManager, UnifiedDeviceManager           │
├─────────────────────────────────────────────────────────────┤
│  Layer 4  Controller                                        │
│           Strategies, StrategyExecutor,                     │
│           ModeManager, ProtectionGuard                      │
├─────────────────────────────────────────────────────────────┤
│  Layer 3  Equipment                                         │
│           AsyncModbusDevice, Points, Transforms,            │
│           Alarms, ReadScheduler                             │
├─────────────────────────────────────────────────────────────┤
│  Layer 2  Modbus                                            │
│           Data types, async clients (TCP/RTU/Shared),       │
│           codec, exceptions                                 │
├─────────────────────────────────────────────────────────────┤
│  Layer 1  Core                                              │
│           get_logger, AsyncLifecycleMixin,                  │
│           errors, HealthCheckable, CircuitBreaker           │
└─────────────────────────────────────────────────────────────┘

         ↑ 依賴方向：只能向下依賴，絕不向上
```

這張圖是理解整個 csp_lib 的地圖。在你繼續閱讀後面的整合篇文章時，你可以隨時回來對照每個元件的位置。

---

## 逐層解析：責任與關鍵類別

### Layer 1：Core — 一切的根基

**路徑：** `csp_lib/core/`
**依賴：** 無（僅依賴 Python 標準庫和 loguru）
**被依賴：** 所有其他層

Core 層提供的是全系統共用的基礎設施——日誌、生命週期管理、錯誤定義、健康檢查。它不包含任何業務邏輯，也不知道 Modbus 是什麼。

```python
from csp_lib.core import (
    get_logger,            # 模組級 logger 工廠
    AsyncLifecycleMixin,   # async with 生命週期 Mixin
    HealthCheckable,       # 健康檢查協定
    HealthStatus,          # HEALTHY / DEGRADED / UNHEALTHY
    HealthReport,          # 階層式健康報告
    CircuitBreaker,        # 斷路器（連續失敗保護）
    RetryPolicy,           # 重試策略
)

# 錯誤層次結構
from csp_lib.core.errors import (
    DeviceError,           # 設備層基礎例外
    DeviceConnectionError, # 連線失敗
    CommunicationError,    # 讀寫逾時/解碼錯誤
    AlarmError,            # 告警觸發
    ConfigurationError,    # 配置無效
)
```

Core 層最重要的設計決策是 `AsyncLifecycleMixin`。這個 Mixin 要求所有需要啟動/停止的元件都遵循統一的生命週期協定：

```python
class AsyncLifecycleMixin:
    async def start(self) -> None: ...
    async def stop(self) -> None: ...
    async def __aenter__(self) -> Self: ...
    async def __aexit__(self, *args) -> None: ...
```

子類只要覆寫 `_on_start()` 和 `_on_stop()`，就能同時獲得 `start()/stop()` 方法和 `async with` 語法支援。這個模式在後續每一層都會反覆出現。

### Layer 2：Modbus — 通訊基礎

**路徑：** `csp_lib/modbus/`
**依賴：** Core
**被依賴：** Equipment

Modbus 層負責所有 Modbus 協定相關的低階操作——資料類型定義、位元組順序、編解碼、非同步客戶端封裝。

```python
from csp_lib.modbus import (
    # 資料類型
    Int16, UInt16, Int32, UInt32,
    Float32, Float64, ModbusString,
    DynamicInt, DynamicUInt,

    # 客戶端
    PymodbusTcpClient,        # TCP 非同步客戶端
    PymodbusRtuClient,        # RTU 非同步客戶端
    SharedPymodbusTcpClient,  # 多設備共用 TCP 連線

    # 編解碼
    ModbusCodec,

    # 配置
    ModbusTcpConfig,
    ModbusRtuConfig,
    ByteOrder, RegisterOrder, FunctionCode,
)
```

這一層的價值在於：**它讓上層完全不需要知道 Modbus 的二進制格式**。當你在 Equipment 層定義一個 `Float32` 類型的點位時，Modbus 層會自動處理 2 個暫存器的讀取、位元組順序的轉換、浮點數的解碼。

### Layer 3：Equipment — 設備建模

**路徑：** `csp_lib/equipment/`
**依賴：** Core, Modbus
**被依賴：** Controller, Manager, Integration

Equipment 層是整個架構中最豐富的一層。它把「Modbus 設備」從一堆暫存器位址，抽象成有名稱、有型別、有告警、有事件的 Python 物件。

```python
from csp_lib.equipment.core import (
    ReadPoint, WritePoint,          # 讀寫點位定義
    pipeline, ScaleTransform,       # 轉換管線
    RangeValidator,                 # 值域驗證
)
from csp_lib.equipment.alarm import (
    AlarmDefinition, AlarmLevel,    # 告警定義
    HysteresisConfig,              # 遲滯（防抖）
    AlarmStateManager,             # 狀態機
)
from csp_lib.equipment.device import AsyncModbusDevice  # 核心設備類別
from csp_lib.equipment.template import EquipmentTemplate  # 可重用範本
```

`AsyncModbusDevice` 是這一層的核心。它是一個事件驅動的非同步物件，能夠：
- 定期讀取所有點位（ReadScheduler）
- 偵測值變化並發射事件（`value_change`、`alarm_triggered`）
- 管理告警狀態（含遲滯防抖）
- 報告自身健康狀態

### Layer 4：Controller — 控制策略

**路徑：** `csp_lib/controller/`
**依賴：** Core, Equipment
**被依賴：** Manager, Integration

Controller 層定義了控制策略的抽象框架。這是工業控制的靈魂——決定「現在應該輸出多少功率」的邏輯都在這裡。

```python
from csp_lib.controller import (
    # 核心抽象
    Strategy,             # 策略基礎類別
    Command,              # 策略輸出（P, Q）
    StrategyContext,       # 執行上下文（SOC, 電壓, 頻率...）
    StrategyExecutor,      # 定時執行策略

    # 系統管理
    ModeManager,           # 多模式優先權管理
    ModePriority,          # 模式優先權
    ProtectionGuard,       # 保護鏈
    SOCProtection,         # SOC 上下限保護

    # 內建策略
    PQModeStrategy,        # 定功率模式
    QVStrategy,            # 電壓無功控制
    FPStrategy,            # 頻率功率控制
    IslandModeStrategy,    # 孤島模式
    BypassStrategy,        # 旁路模式
    StopStrategy,          # 停機策略
)
```

所有策略都繼承自 `Strategy` 抽象類別，只需實作兩個方法：`execution_config`（定義執行頻率）和 `execute(context) -> Command`（計算輸出功率）。

### Layer 5：Manager — 業務管理

**路徑：** `csp_lib/manager/`
**依賴：** Core, Equipment, Controller, Storage
**被依賴：** Integration

Manager 層處理「非即時」的管理任務——告警記錄的持久化、設備資料的批量上傳、設備群組管理、排程策略管理。

```python
from csp_lib.manager import (
    DeviceManager,              # 設備讀取管理
    AlarmPersistenceManager,    # 告警持久化
    DataUploadManager,          # 資料上傳
    UnifiedDeviceManager,       # 統一設備管理器
    WriteCommandManager,        # 寫入指令管理
    ScheduleService,            # 排程策略管理
    StateSyncManager,           # 狀態同步
)
```

### Layer 6：Integration — 系統整合

**路徑：** `csp_lib/integration/`
**依賴：** Core, Equipment, Controller, Manager
**被依賴：** Additional

Integration 層是整個系統的黏合劑。它的任務是：**讓你不需要寫膠水程式碼**。

```python
from csp_lib.integration import (
    DeviceRegistry,        # Trait-based 設備查詢索引
    ContextBuilder,        # 設備值 → StrategyContext 映射
    CommandRouter,         # Command → 設備寫入路由
    SystemController,      # 頂層控制器（核心入口）
    SystemControllerConfig,

    # 映射定義（宣告式配置）
    ContextMapping,        # 設備點位 → Context 欄位
    CommandMapping,        # Command 欄位 → 設備寫入
    HeartbeatMapping,      # 心跳寫入

    # 功率分配
    ProportionalDistributor,  # 按額定功率比例分配
    EqualDistributor,         # 均分
    SOCBalancingDistributor,  # SOC 平衡分配
)
```

`SystemController` 是整個框架的頂層入口。它組合了 ModeManager、ProtectionGuard、ContextBuilder、CommandRouter，把所有元件串成一個完整的控制迴路。

### Layer 7：Storage — 持久化

**路徑：** `csp_lib/mongo/`, `csp_lib/redis/`
**依賴：** Core
**被依賴：** Manager, Additional

Storage 層提供 MongoDB 和 Redis 的封裝。注意它的位置——它只依賴 Core 層，而被 Manager 層引用。這意味著儲存邏輯和業務邏輯是解耦的，你可以實作自己的 Repository 替換預設實作。

### Layer 8：Additional — 附加功能

**路徑：** `csp_lib/cluster/`, `csp_lib/monitor/`, `csp_lib/notification/`, `csp_lib/modbus_server/`, `csp_lib/gui/`, `csp_lib/statistics/`
**依賴：** 依模組而定
**被依賴：** 無

Additional 層是可選的附加功能。每個子模組都是獨立的：

- **cluster**：透過 etcd leader election 實現高可用
- **monitor**：系統資源監控（CPU、RAM、磁碟、網路）
- **notification**：多通道告警通知（可擴充）
- **modbus_server**：Modbus TCP 伺服器（暴露資料給上位系統）
- **gui**：FastAPI Web 介面
- **statistics**：統計分析

---

## 依賴方向規則：鐵律中的鐵律

> **Lower layers MUST NOT import upper layers.**

這是 csp_lib 架構中最重要的一條規則。讓我用一個具體的例子說明為什麼：

```python
# --- 正確 ---
# csp_lib/integration/context_builder.py (Layer 6)
from csp_lib.equipment.device import AsyncModbusDevice  # Layer 3 ← 向下依賴

# --- 錯誤 ---
# csp_lib/modbus/clients.py (Layer 2)
from csp_lib.equipment.core import ReadPoint  # Layer 3 ← 向上依賴！禁止！
```

為什麼如此嚴格？因為在工業系統中，底層通訊的穩定性是一切的基礎。如果 Modbus 層引用了 Equipment 層的某個類別，那麼 Equipment 層的任何變更都可能影響 Modbus 通訊——這在即時控制系統中是不可接受的風險。

在 csp_lib 的程式碼中，你會看到大量使用 `TYPE_CHECKING` 來避免運行時的循環引用：

```python
from __future__ import annotations
from typing import TYPE_CHECKING

if TYPE_CHECKING:
    from csp_lib.equipment.device import AsyncModbusDevice  # 只在型別檢查時引用
```

這個模式讓靜態型別檢查器（mypy）能驗證型別正確性，同時不會在運行時引入向上依賴。

### 依賴關係速查表

| 模組 | 路徑 | 可依賴 | 被依賴 |
|------|------|--------|--------|
| Core | `csp_lib/core/` | （無） | 全部 |
| Modbus | `csp_lib/modbus/` | Core | Equipment |
| Equipment | `csp_lib/equipment/` | Core, Modbus | Controller, Manager, Integration |
| Controller | `csp_lib/controller/` | Core, Equipment | Manager, Integration |
| Manager | `csp_lib/manager/` | Core, Equipment, Controller, Storage | Integration |
| Integration | `csp_lib/integration/` | Core, Equipment, Controller, Manager | Additional |
| Storage | `csp_lib/mongo/`, `csp_lib/redis/` | Core | Manager, Additional |
| Additional | `csp_lib/cluster/`, `csp_lib/monitor/`... | 依模組而定 | （無） |

---

## 橫跨全局的設計模式

csp_lib 在整個程式碼庫中反覆使用幾個關鍵的設計模式。理解它們，你就掌握了閱讀任何模組程式碼的金鑰匙。

### 模式一：Frozen Dataclass 配置

幾乎所有的配置物件都使用 `@dataclass(frozen=True)` 來定義。不可變性（immutability）在工業控制中是一個安全特性——你不希望某段程式碼在控制迴路運行中悄悄修改了配置。

```python
@dataclass(frozen=True)
class ContextMapping:
    """設備點位 → StrategyContext 欄位映射"""
    point_name: str
    context_field: str
    device_id: str | None = None
    trait: str | None = None
    aggregate: AggregateFunc = AggregateFunc.AVERAGE
    default: Any = None
```

搭配 `ConfigMixin`，這些配置可以直接從字典建立，並自動過濾多餘的 key、轉換 camelCase 到 snake_case：

```python
config = PQModeConfig.from_dict({"p": 100, "q": 50, "extraField": "ignored"})
```

### 模式二：AsyncLifecycleMixin 生命週期管理

每個需要啟動/停止的元件——從底層的 Modbus 客戶端，到頂層的 SystemController——都繼承 `AsyncLifecycleMixin`：

```python
class SystemController(AsyncLifecycleMixin):
    async def _on_start(self) -> None:
        # 啟動心跳服務、策略執行器...

    async def _on_stop(self) -> None:
        # 按序停止所有子元件...

# 使用者只需要：
async with SystemController(registry, config) as controller:
    await asyncio.Event().wait()  # 執行直到被中斷
```

這個模式的價值在於**資源回收的確定性**。Python 的垃圾收集器不保證析構函式的呼叫順序，但 `async with` 保證 `_on_stop()` 一定會被呼叫——即使發生例外。在工業場域，「沒有正確停止」可能意味著設備收不到心跳而進入安全模式。

### 模式三：Protocol-based 抽象（@runtime_checkable）

csp_lib 使用 Python 的 `Protocol` 而非 ABC（抽象基礎類別）來定義核心介面：

```python
@runtime_checkable
class HealthCheckable(Protocol):
    """健康檢查協定"""
    def health(self) -> HealthReport: ...

@runtime_checkable
class GridControllerProtocol(Protocol):
    """併網控制器協定"""
    def set_strategy(self, strategy: Strategy | None) -> None: ...
    async def start(self) -> None: ...
    async def stop(self) -> None: ...
```

Protocol 的優勢是**結構型子型別**（structural subtyping）。任何實作了 `health()` 方法的類別都自動滿足 `HealthCheckable`，不需要顯式繼承。這在整合第三方程式碼時特別有用。

### 模式四：事件驅動架構

`AsyncModbusDevice` 採用事件驅動設計。當點位值變化或告警觸發時，它會發射事件，訂閱者可以註冊回呼函式：

```python
device.on("value_change", callback)
device.on("alarm_triggered", alarm_callback)
```

這個模式讓上層不需要輪詢設備狀態。Manager 層的 `DataUploadManager` 監聽值變化事件來觸發資料上傳，`AlarmPersistenceManager` 監聽告警事件來記錄歷史。

### 模式五：映射驅動配置（零程式碼整合）

Integration 層最強大的設計是**映射驅動配置**。你不需要寫任何膠水程式碼來串接設備和控制器——只需要宣告映射關係：

```python
config = SystemControllerConfig(
    context_mappings=[
        # 從 BMS 讀 SOC → 注入策略的 soc 欄位
        ContextMapping(point_name="soc", context_field="soc", trait="bms"),
        # 從電表讀電壓 → 注入策略的 extra.voltage 欄位
        ContextMapping(point_name="voltage", context_field="extra.voltage", trait="meter"),
    ],
    command_mappings=[
        # 策略輸出的 p_target → 寫入所有 PCS 的 p_set 點位
        CommandMapping(command_field="p_target", point_name="p_set", trait="pcs"),
    ],
)
```

`ContextBuilder` 根據 `context_mappings` 自動從設備讀取值並組裝 `StrategyContext`；`CommandRouter` 根據 `command_mappings` 自動將 `Command` 路由到對應的設備寫入點位。新增設備或修改映射，完全不需要改程式碼。

---

## 可選依賴設計：按需組裝

csp_lib 使用 Python 的 optional dependencies 機制，讓你只安裝需要的部分：

```toml
# pyproject.toml
[project.optional-dependencies]
modbus = ["pymodbus>=3.0.0"]
mongo  = ["motor>=3.0.0"]
redis  = ["redis>=5.0.0"]
monitor = ["psutil>=5.9.0"]
cluster = ["etcetra>=0.1.0"]
gui = ["fastapi>=0.115.0", "uvicorn[standard]>=0.30.0", "pyyaml>=6.0"]
all = ["csp0924_lib[mongo,redis,modbus,can,monitor,cluster,gui]"]
```

安裝時按需選擇：

```bash
# 最小安裝：只有核心 + loguru
pip install csp0924_lib

# 開發環境：安裝所有功能
pip install csp0924_lib[all]

# 生產環境：只裝需要的
pip install csp0924_lib[modbus,mongo,redis]

# 需要 GUI 的場景
pip install csp0924_lib[modbus,gui]
```

為什麼這麼設計？因為工業現場的部署環境往往非常受限。一個嵌入式 Linux 設備可能只有 512MB 的磁碟空間，安裝完整的 FastAPI + uvicorn + MongoDB driver 是浪費。而在開發機器上，你希望所有功能都能用。

核心套件唯一的必要依賴只有 `loguru`——一個零配置的 Python 日誌框架。其餘所有外部依賴都是可選的。

---

## 資料流端到端追蹤

讓我們追蹤一筆資料從 Modbus 暫存器到最終的設備寫入，完整走過八層架構的每一站。

### 場景：BMS 回報 SOC 為 75%，策略決定輸出 500kW

```
Step 1 [Layer 2 - Modbus]
  PymodbusTcpClient 讀取 BMS 暫存器 5004
  → 原始值：[0x02, 0xEE] (UInt16 = 750)

Step 2 [Layer 3 - Equipment]
  ReadPoint 的 pipeline 對原始值做轉換
  → ScaleTransform(magnitude=0.1)
  → 750 × 0.1 = 75.0
  → AsyncModbusDevice.latest_values["soc"] = 75.0
  → 發射 value_change 事件

Step 3 [Layer 6 - Integration / ContextBuilder]
  ContextBuilder 根據 ContextMapping(point_name="soc",
      context_field="soc", trait="bms")
  → 從 DeviceRegistry 找到 trait="bms" 的設備
  → 讀取 latest_values["soc"] = 75.0
  → 注入 StrategyContext(soc=75.0)

Step 4 [Layer 4 - Controller / StrategyExecutor]
  StrategyExecutor 以 StrategyContext 呼叫 PQModeStrategy.execute()
  → PQModeStrategy 輸出 Command(p_target=500.0, q_target=0.0)

Step 5 [Layer 4 - Controller / ProtectionGuard]
  ProtectionGuard 檢查 SOCProtection
  → SOC 75% 在安全範圍內（5%~95%）
  → Command 不做修改

Step 6 [Layer 6 - Integration / CommandRouter]
  CommandRouter 根據 CommandMapping(command_field="p_target",
      point_name="p_set", trait="pcs")
  → 找到 trait="pcs" 的所有設備
  → PowerDistributor 按額定功率比例分配
  → PCS_1: 250kW, PCS_2: 250kW

Step 7 [Layer 3 - Equipment]
  WritePoint 的 pipeline 反向轉換
  → AsyncModbusDevice.write("p_set", 250.0)

Step 8 [Layer 2 - Modbus]
  PymodbusTcpClient 寫入 PCS 暫存器
  → Float32 編碼 → 寫入 2 個暫存器
```

整個流程從讀取到寫入，典型延遲在 50-200 毫秒之間。StrategyExecutor 以可配置的頻率（預設每秒一次）執行這個迴圈。

需要注意的是，**控制迴路是閉環的**。上一次的 `Command` 會作為 `StrategyContext.last_command` 傳入下一次的策略計算——這讓策略可以實作漸進式的功率爬坡（ramp rate）或 PID 控制。

---

## 重點回顧

1. **八層架構**是 csp_lib 的骨幹。由下而上：Core → Modbus → Equipment → Controller → Manager → Integration → Storage → Additional。

2. **依賴方向規則**是鐵律：下層絕不引用上層。這保證了底層通訊的穩定性和每一層的獨立可測試性。

3. **五大設計模式**貫穿全局：Frozen Dataclass 配置、AsyncLifecycleMixin 生命週期、Protocol-based 抽象、事件驅動架構、映射驅動配置。

4. **可選依賴**讓你在資源受限的環境中只安裝需要的元件，核心套件僅依賴 loguru。

5. **資料從暫存器到設備寫入**需要經過八個步驟，每一步都在對應的架構層級中處理，關注點完全分離。

---

## 下篇預告

理解了架構全貌之後，下一篇我們將進入實戰：**如何把 csp_lib 從開發環境部署到工業現場**。我們會涵蓋開發環境的設置、CI/CD 管線、Cython 生產構建、Docker 部署拓撲，以及工業場域特有的配置管理挑戰。

> [下一篇：部署策略：從開發環境到工業現場 >>>](./26-deployment.md)
