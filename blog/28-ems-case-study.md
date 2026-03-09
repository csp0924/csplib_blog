# 實戰案例：打造一座 1MW 儲能系統的 EMS

> **從零打造工業控制協定框架：給軟體工程師的 CSP Library 實戰系列**
>
> Part 6 — 系統整合篇 | Article 28（系列完結篇）
>
> [<<< 上一篇：監控與告警](./27-monitoring.md)

---

## 目錄

1. [專案概述](#專案概述)
2. [系統架構設計](#系統架構設計)
3. [Step 1：定義設備範本](#step-1定義設備範本)
4. [Step 2：建立設備實例與 Registry](#step-2建立設備實例與-registry)
5. [Step 3：配置 SystemController](#step-3配置-systemcontroller)
6. [Step 4：註冊控制策略](#step-4註冊控制策略)
7. [Step 5：加入高可用叢集](#step-5加入高可用叢集)
8. [Step 6：加入監控與通知](#step-6加入監控與通知)
9. [Step 7：部署到生產環境](#step-7部署到生產環境)
10. [實戰經驗與踩坑記錄](#實戰經驗與踩坑記錄)
11. [效能數據](#效能數據)
12. [系列回顧與總結](#系列回顧與總結)

---

## 專案概述

讓筆者假設你剛接手了一個儲能系統的 EMS（Energy Management System）專案。以下是案場的基本規格：

### 硬體配置

| 設備 | 數量 | 規格 | 通訊方式 |
|------|------|------|---------|
| PCS（功率調節系統） | 2 台 | 各 500kW | Modbus TCP |
| BMS（電池管理系統） | 1 套 | 2MWh | Modbus TCP |
| 電力表計 | 1 台 | 關口表 | Modbus TCP |

**系統總容量**：1MW / 2MWh（2 台 500kW PCS 並聯）

### 功能需求

1. **PQ 定功率控制**：根據排程或手動指令，輸出指定的有功/無功功率
2. **QV 電壓無功控制**：根據電網電壓自動調節無功功率
3. **SOC 保護**：電池 SOC 低於 5% 或高於 95% 時限制充放電
4. **功率分配**：總功率依額定容量比例分配到 2 台 PCS
5. **設備告警管理**：告警偵測、記錄、通知
6. **高可用部署**：雙機熱備，主機故障時自動切換
7. **Web 監控介面**：即時查看系統狀態

### 對應 csp_lib 的架構層級

```
┌──────────────────────────────────────────────────────────────┐
│ Layer 8  GUI (FastAPI)  +  Monitor  +  Notification         │
├──────────────────────────────────────────────────────────────┤
│ Layer 7  MongoDB (告警/資料) + Redis (狀態同步/快取)          │
├──────────────────────────────────────────────────────────────┤
│ Layer 6  SystemController + DeviceRegistry                   │
│          ContextBuilder + CommandRouter + ProportionalDist.   │
├──────────────────────────────────────────────────────────────┤
│ Layer 5  AlarmPersistenceManager + DataUploadManager          │
├──────────────────────────────────────────────────────────────┤
│ Layer 4  PQModeStrategy + QVStrategy + SOCProtection         │
│          ModeManager + ProtectionGuard                        │
├──────────────────────────────────────────────────────────────┤
│ Layer 3  PCS Template + BMS Template + Meter Template         │
│          AsyncModbusDevice × 4                                │
├──────────────────────────────────────────────────────────────┤
│ Layer 2  PymodbusTcpClient × 4                                │
├──────────────────────────────────────────────────────────────┤
│ Layer 1  get_logger + AsyncLifecycleMixin + errors            │
└──────────────────────────────────────────────────────────────┘
```

現在，讓我們一步一步地把這個系統建構起來。

---

## 系統架構設計

在寫任何程式碼之前，先畫清楚資料流的方向：

```
         ┌─────────┐
         │  BMS    │──── SOC, Voltage, Temp, Alarms
         └────┬────┘
              │ Modbus TCP
              ▼
┌─────────────────────────────────────┐
│         SystemController            │
│                                     │
│  ContextBuilder ──► StrategyContext  │
│      (SOC, V, F, P_grid...)        │
│              │                      │
│              ▼                      │
│  ModeManager ──► Active Strategy    │
│              │                      │
│              ▼                      │
│  Strategy.execute(ctx) ──► Command  │
│              │                      │
│              ▼                      │
│  ProtectionGuard ──► Protected Cmd  │
│              │                      │
│              ▼                      │
│  PowerDistributor ──► Per-Device    │
│              │                      │
│              ▼                      │
│  CommandRouter ──► Device Writes    │
└────────┬──────────┬─────────────────┘
         │          │
         ▼          ▼
    ┌─────────┐ ┌─────────┐
    │  PCS 1  │ │  PCS 2  │
    │ 500kW   │ │ 500kW   │
    └─────────┘ └─────────┘

         ┌─────────┐
         │ Meter   │──── Grid Power, Voltage, Frequency
         └─────────┘
```

---

## Step 1：定義設備範本

設備範本（`EquipmentTemplate`）是可重用的設備模型定義。你定義一次 PCS 的所有點位、告警和轉換管線，就能用它建立多台 PCS 實例。

### PCS 範本

```python
from csp_lib.equipment.template import EquipmentTemplate
from csp_lib.equipment.core import (
    ReadPoint, WritePoint, PointMetadata,
    pipeline, ScaleTransform, RoundTransform, EnumMapTransform, ClampTransform,
)
from csp_lib.equipment.alarm import (
    AlarmDefinition, AlarmLevel, HysteresisConfig,
    ThresholdAlarmEvaluator, BitMaskAlarmEvaluator,
    ThresholdCondition, Operator,
)
from csp_lib.modbus import Float32, UInt16, Int16

pcs_template = EquipmentTemplate(
    model="PCS-500KW",
    description="500kW PCS 功率調節系統",

    # ---- 每次都要讀的點位（即時數據）----
    always_points=(
        ReadPoint(
            name="active_power",
            address=5000,
            data_type=Float32(),
            pipeline=pipeline(RoundTransform(decimals=1)),
            metadata=PointMetadata(unit="kW", description="即時有功功率"),
        ),
        ReadPoint(
            name="reactive_power",
            address=5002,
            data_type=Float32(),
            pipeline=pipeline(RoundTransform(decimals=1)),
            metadata=PointMetadata(unit="kVar", description="即時無功功率"),
        ),
        ReadPoint(
            name="status",
            address=5004,
            data_type=UInt16(),
            pipeline=pipeline(EnumMapTransform(
                mapping={0: "STOP", 1: "RUN", 2: "FAULT", 3: "STANDBY"},
                default="UNKNOWN",
            )),
            metadata=PointMetadata(description="運行狀態"),
        ),
        ReadPoint(
            name="fault_code",
            address=5005,
            data_type=UInt16(),
            metadata=PointMetadata(description="故障碼"),
        ),
    ),

    # ---- 輪詢點位（非即時，輪流讀取）----
    rotating_points=(
        (
            ReadPoint(
                name="igbt_temp",
                address=5010,
                data_type=Int16(),
                pipeline=pipeline(ScaleTransform(magnitude=0.1)),
                metadata=PointMetadata(unit="°C", description="IGBT 溫度"),
            ),
            ReadPoint(
                name="ambient_temp",
                address=5011,
                data_type=Int16(),
                pipeline=pipeline(ScaleTransform(magnitude=0.1)),
                metadata=PointMetadata(unit="°C", description="環境溫度"),
            ),
        ),
        (
            ReadPoint(
                name="daily_charge",
                address=5020,
                data_type=Float32(),
                pipeline=pipeline(RoundTransform(decimals=2)),
                metadata=PointMetadata(unit="kWh", description="日充電量"),
            ),
            ReadPoint(
                name="daily_discharge",
                address=5022,
                data_type=Float32(),
                pipeline=pipeline(RoundTransform(decimals=2)),
                metadata=PointMetadata(unit="kWh", description="日放電量"),
            ),
        ),
    ),

    # ---- 寫入點位 ----
    write_points=(
        WritePoint(
            name="p_set",
            address=6000,
            data_type=Float32(),
            metadata=PointMetadata(unit="kW", description="有功功率設定"),
        ),
        WritePoint(
            name="q_set",
            address=6002,
            data_type=Float32(),
            metadata=PointMetadata(unit="kVar", description="無功功率設定"),
        ),
        WritePoint(
            name="heartbeat",
            address=6004,
            data_type=UInt16(),
            metadata=PointMetadata(description="心跳看門狗"),
        ),
    ),

    # ---- 告警評估器 ----
    alarm_evaluators=(
        ThresholdAlarmEvaluator(
            alarm=AlarmDefinition(
                code="PCS_OVER_TEMP",
                name="IGBT 溫度過高",
                level=AlarmLevel.ALARM,
                hysteresis=HysteresisConfig(activate_threshold=3, clear_threshold=5),
            ),
            point_name="igbt_temp",
            condition=ThresholdCondition(operator=Operator.GREATER_THAN, threshold=85.0),
        ),
        BitMaskAlarmEvaluator(
            alarm=AlarmDefinition(
                code="PCS_FAULT",
                name="PCS 故障",
                level=AlarmLevel.ALARM,
            ),
            point_name="fault_code",
            bit_mask=0xFFFF,  # 任何非零值都是故障
        ),
    ),
)
```

讓我逐段解釋這個範本的設計思路。

**always_points**：這些是控制迴路每一輪都必須讀取的點位——即時功率、狀態、故障碼。控制決策依賴這些資料的即時性，不能容忍輪詢延遲。

**rotating_points**：溫度和累計電量不需要每秒讀取，但仍然需要定期監控。`rotating_points` 讓這些點位分組輪詢——第一輪讀溫度，第二輪讀電量——減少每輪的 Modbus 請求數量，提升通訊效率。

**write_points**：PCS 需要接收有功（P）和無功（Q）功率指令，以及心跳信號。心跳是一個看門狗機制——如果 PCS 超過一定時間沒收到心跳，它會自動進入安全模式停機。

**alarm_evaluators**：告警評估器定義了「什麼條件算告警」。IGBT 溫度超過 85 度且連續 3 次確認才觸發，避免瞬間干擾造成的誤報。

### BMS 範本

```python
bms_template = EquipmentTemplate(
    model="BMS-2MWH",
    description="2MWh 電池管理系統",
    always_points=(
        ReadPoint(
            name="soc",
            address=3000,
            data_type=UInt16(),
            pipeline=pipeline(ScaleTransform(magnitude=0.1)),
            metadata=PointMetadata(unit="%", description="電池 SOC"),
        ),
        ReadPoint(
            name="total_voltage",
            address=3002,
            data_type=Float32(),
            pipeline=pipeline(RoundTransform(decimals=1)),
            metadata=PointMetadata(unit="V", description="總電壓"),
        ),
        ReadPoint(
            name="total_current",
            address=3004,
            data_type=Float32(),
            pipeline=pipeline(RoundTransform(decimals=1)),
            metadata=PointMetadata(unit="A", description="總電流"),
        ),
        ReadPoint(
            name="max_cell_temp",
            address=3006,
            data_type=Int16(),
            pipeline=pipeline(ScaleTransform(magnitude=0.1)),
            metadata=PointMetadata(unit="°C", description="最高電芯溫度"),
        ),
        ReadPoint(
            name="alarm_word",
            address=3008,
            data_type=UInt16(),
            metadata=PointMetadata(description="告警字"),
        ),
    ),
    alarm_evaluators=(
        BitMaskAlarmEvaluator(
            alarm=AlarmDefinition(code="BMS_OVER_TEMP", name="電芯溫度過高", level=AlarmLevel.ALARM),
            point_name="alarm_word",
            bit_mask=0x0001,
        ),
        BitMaskAlarmEvaluator(
            alarm=AlarmDefinition(code="BMS_OVER_VOLTAGE", name="電芯過壓", level=AlarmLevel.ALARM),
            point_name="alarm_word",
            bit_mask=0x0002,
        ),
        BitMaskAlarmEvaluator(
            alarm=AlarmDefinition(code="BMS_UNDER_VOLTAGE", name="電芯欠壓", level=AlarmLevel.ALARM),
            point_name="alarm_word",
            bit_mask=0x0004,
        ),
    ),
)
```

### Meter 範本

```python
meter_template = EquipmentTemplate(
    model="METER-3P",
    description="三相電力表計",
    always_points=(
        ReadPoint(
            name="grid_power",
            address=1000,
            data_type=Float32(),
            pipeline=pipeline(RoundTransform(decimals=1)),
            metadata=PointMetadata(unit="kW", description="電網功率"),
        ),
        ReadPoint(
            name="voltage",
            address=1002,
            data_type=Float32(),
            pipeline=pipeline(RoundTransform(decimals=1)),
            metadata=PointMetadata(unit="V", description="電壓"),
        ),
        ReadPoint(
            name="frequency",
            address=1004,
            data_type=Float32(),
            pipeline=pipeline(RoundTransform(decimals=2)),
            metadata=PointMetadata(unit="Hz", description="頻率"),
        ),
        ReadPoint(
            name="power_factor",
            address=1006,
            data_type=Float32(),
            pipeline=pipeline(RoundTransform(decimals=3)),
            metadata=PointMetadata(description="功率因數"),
        ),
    ),
)
```

---

## Step 2：建立設備實例與 Registry

有了範本，接下來用 `DeviceFactory` 建立實際的設備實例，並註冊到 `DeviceRegistry`。

```python
from csp_lib.modbus import PymodbusTcpClient, ModbusTcpConfig
from csp_lib.equipment.template import DeviceFactory
from csp_lib.integration import DeviceRegistry

# ---- 建立 Modbus 客戶端 ----
pcs1_client = PymodbusTcpClient(ModbusTcpConfig(host="192.168.1.10", port=502, unit_id=1))
pcs2_client = PymodbusTcpClient(ModbusTcpConfig(host="192.168.1.11", port=502, unit_id=1))
bms_client  = PymodbusTcpClient(ModbusTcpConfig(host="192.168.1.20", port=502, unit_id=1))
meter_client = PymodbusTcpClient(ModbusTcpConfig(host="192.168.1.30", port=502, unit_id=1))

# ---- 使用 DeviceFactory 建立設備實例 ----
factory = DeviceFactory()

pcs_1 = factory.create(device_id="pcs_1", template=pcs_template, client=pcs1_client)
pcs_2 = factory.create(device_id="pcs_2", template=pcs_template, client=pcs2_client)
bms   = factory.create(device_id="bms_1", template=bms_template, client=bms_client)
meter = factory.create(device_id="meter_1", template=meter_template, client=meter_client)

# ---- 註冊到 DeviceRegistry ----
registry = DeviceRegistry()

# 每台設備帶上 traits（角色標籤）和 metadata（靜態參數）
registry.register(pcs_1, traits=["pcs"], metadata={"rated_p": 500.0})
registry.register(pcs_2, traits=["pcs"], metadata={"rated_p": 500.0})
registry.register(bms,   traits=["bms"])
registry.register(meter, traits=["meter"])
```

讓我強調幾個設計要點。

**traits 是角色標籤**，不是設備型號。你用 `"pcs"` 標籤來表示「這台設備扮演 PCS 的角色」。後續的映射配置都用 trait 來引用設備群組——這樣當你新增第三台 PCS 時，只需要多一行 `registry.register()`，所有的控制邏輯和映射配置都不需要改。

**metadata 是靜態參數**。`rated_p: 500.0` 告訴 `ProportionalDistributor`：這台 PCS 的額定功率是 500kW。當系統要輸出 800kW 時，分配器會按 500:500 的比例分給兩台 PCS，各 400kW。

---

## Step 3：配置 SystemController

`SystemControllerConfig` 是整個系統的核心配置。它定義了「設備的哪些值要餵給策略」以及「策略的輸出要寫到設備的哪裡」。

```python
from csp_lib.integration import (
    SystemController, SystemControllerConfig,
    ContextMapping, CommandMapping, HeartbeatMapping,
    ProportionalDistributor, HeartbeatMode,
)
from csp_lib.controller import SystemBase, SOCProtection, SOCProtectionConfig

config = SystemControllerConfig(
    # ---- 設備值 → 策略上下文 ----
    context_mappings=[
        # BMS 的 SOC → 策略的 soc 欄位（核心輸入）
        ContextMapping(point_name="soc", context_field="soc", trait="bms"),

        # 電表的電壓 → 策略的 extra.voltage（QV 策略需要）
        ContextMapping(point_name="voltage", context_field="extra.voltage", trait="meter"),

        # 電表的頻率 → 策略的 extra.frequency（FP 策略需要）
        ContextMapping(point_name="frequency", context_field="extra.frequency", trait="meter"),

        # 電表的電網功率 → 策略的 extra.grid_power（監控用）
        ContextMapping(point_name="grid_power", context_field="extra.grid_power", trait="meter"),
    ],

    # ---- 策略輸出 → 設備寫入 ----
    command_mappings=[
        # 策略的 p_target → 所有 PCS 的 p_set（由 distributor 分配）
        CommandMapping(command_field="p_target", point_name="p_set", trait="pcs"),

        # 策略的 q_target → 所有 PCS 的 q_set
        CommandMapping(command_field="q_target", point_name="q_set", trait="pcs"),
    ],

    # ---- 系統基準值 ----
    system_base=SystemBase(p_base=1000.0, q_base=500.0),

    # ---- 保護規則 ----
    protection_rules=[
        SOCProtection(SOCProtectionConfig(
            soc_high=95.0,  # SOC > 95% 時禁止充電（P > 0）
            soc_low=5.0,    # SOC < 5% 時禁止放電（P < 0）
        )),
    ],

    # ---- 心跳映射 ----
    heartbeat_mappings=[
        HeartbeatMapping(
            point_name="heartbeat",
            trait="pcs",
            mode=HeartbeatMode.TOGGLE,  # 交替 0/1
        ),
    ],
    heartbeat_interval=1.0,  # 每秒發送一次

    # ---- 功率分配器 ----
    power_distributor=ProportionalDistributor(rated_key="rated_p"),

    # ---- 告警模式 ----
    auto_stop_on_alarm=True,  # 系統告警時自動停機
)

# 建立 SystemController
controller = SystemController(registry, config)
```

這段配置是整個 EMS 的「接線圖」。讓我逐項解析：

### context_mappings：從哪裡讀

`ContextMapping(point_name="soc", context_field="soc", trait="bms")` 的意思是：

1. 從 DeviceRegistry 中找到所有 trait 為 `"bms"` 的設備
2. 從這些設備的 `latest_values` 中取出 `"soc"` 的值
3. 將這個值填入 `StrategyContext` 的 `soc` 欄位

如果有多台 BMS，會使用 `aggregate` 參數指定的聚合函式（預設 `AVERAGE`）來合併。

`context_field` 的 `"extra.voltage"` 語法表示嵌套路徑——值會被放到 `context.extra["voltage"]` 中。這讓你可以傳遞任意額外資料給策略，而不需要修改 `StrategyContext` 的定義。

### command_mappings：寫到哪裡

`CommandMapping(command_field="p_target", point_name="p_set", trait="pcs")` 的意思是：

1. 從策略輸出的 `Command` 中取出 `p_target` 值
2. 寫入所有 trait 為 `"pcs"` 的設備的 `"p_set"` 點位

因為我們設定了 `power_distributor=ProportionalDistributor(rated_key="rated_p")`，所以不是把同一個值寫給所有 PCS，而是先按 `metadata["rated_p"]` 的比例分配後再寫入。

### SOCProtection：安全閥門

`SOCProtection` 是控制迴路中的安全閥門。當電池 SOC 低於 5% 時，即使策略輸出的命令是放電 500kW，保護鏈也會將 P 值鉗位到 0——不允許繼續放電。同理，SOC 高於 95% 時禁止充電。

這個保護機制在 `ProtectionGuard.apply()` 中執行，位於策略計算之後、設備寫入之前。它是最後一道防線。

---

## Step 4：註冊控制策略

有了 SystemController，接下來註冊控制策略。每個策略都有一個優先權等級——高優先權的 override 模式會暫時取代低優先權的基礎模式。

```python
from csp_lib.controller import (
    PQModeStrategy, PQModeConfig,
    QVStrategy, QVConfig,
    BypassStrategy,
    ModePriority,
)

# ---- 註冊模式 ----

# PQ 定功率模式（排程優先權）
controller.register_mode(
    "pq",
    PQModeStrategy(PQModeConfig(p=800.0, q=0.0)),
    ModePriority.SCHEDULE,
    "定功率 800kW 放電",
)

# QV 電壓無功控制（排程優先權）
controller.register_mode(
    "qv",
    QVStrategy(QVConfig(
        nominal_voltage=380.0,   # 標稱電壓
        # QV 曲線會根據電壓偏差自動計算無功輸出
    )),
    ModePriority.SCHEDULE,
    "電壓無功控制",
)

# 旁路模式（手動優先權，比排程更高）
controller.register_mode(
    "bypass",
    BypassStrategy(),
    ModePriority.MANUAL,
    "旁路模式 - 控制器不輸出任何命令",
)

# ---- 設定初始模式 ----
await controller.set_base_mode("pq")
```

### 模式優先權機制

`ModePriority` 定義了模式的優先權層級。當多個模式同時存在時，高優先權的模式會覆蓋低優先權的模式：

```
PROTECTION (最高)  ← 系統告警自動停機
MANUAL             ← 運維人員手動切換
SCHEDULE           ← 排程自動切換
BASE (最低)        ← 預設模式
```

這個設計讓你可以在不同場景下優雅地切換控制策略：

```python
# 正常運行：PQ 模式在控制
# → 收到手動指令切換到旁路
await controller.push_override("bypass")
# → 現在 bypass 生效（MANUAL > SCHEDULE）

# → 解除旁路，回到 PQ
await controller.pop_override("bypass")
# → 回到 PQ 模式

# → 系統告警觸發
# → auto_stop_on_alarm=True，自動推入 StopStrategy
# → 停機模式生效（PROTECTION > 所有）
# → 告警解除後自動恢復之前的模式
```

### 多基礎模式共存

csp_lib 支援多個基礎模式同時存在。例如同時啟用 PQ 和 QV，由 `CascadingStrategy` 自動組合它們的輸出：

```python
# 同時啟用 PQ 和 QV
await controller.add_base_mode("pq")
await controller.add_base_mode("qv")

# 設定系統視在功率上限（避免 P+Q 超容量）
# 需在 SystemControllerConfig 中設定 capacity_kva
```

---

## Step 5：加入高可用叢集

單節點部署在生產環境中是不可接受的。任何硬體故障——硬碟損壞、記憶體錯誤、甚至是作業系統更新重啟——都會導致 EMS 停擺。在儲能系統中，EMS 停擺意味著電池系統失去控制，可能需要手動切換到本地控制模式。

csp_lib 的 `cluster` 模組提供了基於 etcd 的 leader election，實現雙機熱備。

```python
from csp_lib.cluster import (
    ClusterConfig, EtcdConfig,
    ClusterController, LeaderElector,
    ClusterStatePublisher, ClusterStateSubscriber,
)

# ---- 叢集配置 ----
cluster_config = ClusterConfig(
    node_id="node-a",           # 本節點 ID
    etcd=EtcdConfig(
        endpoints=["http://192.168.1.100:2379"],
    ),
    lease_ttl=10,               # etcd lease 存活秒數
    heartbeat_interval=3,       # leader 心跳間隔
)

# ---- 建立叢集控制器 ----
cluster = ClusterController(
    config=cluster_config,
    system_controller=controller,
    registry=registry,
)
```

### Leader Election 工作原理

```
Node A                          etcd                          Node B
  │                              │                              │
  ├──── Create lease (TTL=10s) ──►│                              │
  │     Grant key "/ems/leader"  │                              │
  │◄──── Lease granted ──────────│                              │
  │     (I am leader!)           │                              │
  │                              │                              │
  ├──── Keepalive (every 3s) ───►│                              │
  │                              │◄──── Watch "/ems/leader" ────┤
  │                              │      (Standby watching...)   │
  │                              │                              │
  │  ╳ Node A crashes            │                              │
  │                              │                              │
  │                    Lease TTL │                              │
  │                    expires   │                              │
  │                     (10s)    │                              │
  │                              │──── Key deleted ────────────►│
  │                              │                              │
  │                              │◄──── Create lease ───────────┤
  │                              │      Grant key               │
  │                              │──── Lease granted ──────────►│
  │                              │      (I am leader!)          │
```

Node A 透過 etcd 的 lease 機制取得 leader 身份。它每 3 秒發送一次 keepalive 來延續 lease。如果 Node A 故障，lease 在 10 秒後過期，Node B 偵測到 key 被刪除，立即嘗試取得 leader 身份。

從故障到切換完成的最大延遲是 lease TTL（10 秒）。在這段時間內，沒有節點在控制設備——設備端的看門狗會偵測到心跳停止，進入安全模式（通常是維持當前輸出或緩慢降載）。

### 狀態同步

兩個節點之間需要同步設備狀態，讓 standby 節點在接手時能快速恢復：

```python
# Leader 節點：發布狀態到 Redis
publisher = ClusterStatePublisher(redis_client=redis, registry=registry)

# Standby 節點：訂閱狀態更新
subscriber = ClusterStateSubscriber(redis_client=redis)
```

---

## Step 6：加入監控與通知

### 系統監控

```python
from csp_lib.monitor import SystemMonitor, MonitorConfig, MetricThresholds

monitor_config = MonitorConfig(
    collect_interval=5.0,
    thresholds=MetricThresholds(
        cpu_warning=80.0,
        cpu_critical=95.0,
        memory_warning=80.0,
        memory_critical=95.0,
        disk_warning=85.0,
        disk_critical=95.0,
    ),
)

monitor = SystemMonitor(monitor_config)
```

### 告警持久化

```python
from csp_lib.manager import (
    AlarmPersistenceManager,
    AlarmPersistenceConfig,
    MongoAlarmRepository,
)

alarm_repo = MongoAlarmRepository(db=mongo_db, collection_name="alarms")
alarm_manager = AlarmPersistenceManager(
    config=AlarmPersistenceConfig(),
    repository=alarm_repo,
)
```

### 通知通道

```python
from csp_lib.notification import (
    NotificationBatcher,
    NotificationChannel,
    Notification,
    BatchNotificationConfig,
)

# 實作一個 LINE Notify 通道（示例）
class LineNotifyChannel(NotificationChannel):
    def __init__(self, token: str):
        self._token = token

    async def send(self, notification: Notification) -> None:
        import httpx
        async with httpx.AsyncClient() as client:
            await client.post(
                "https://notify-api.line.me/api/notify",
                headers={"Authorization": f"Bearer {self._token}"},
                data={"message": f"\n{notification.title}\n{notification.message}"},
            )

# 建立批次通知器
batcher = NotificationBatcher(
    config=BatchNotificationConfig(),
    channels=[LineNotifyChannel(token="your-line-token")],
)
```

---

## Step 7：部署到生產環境

### 主程式入口

把所有元件組裝在一起：

```python
import asyncio
import signal
from csp_lib.core import configure_logging

async def main():
    configure_logging(level="INFO")

    # Step 1-2: 建立設備和 registry（如前述）
    # Step 3: 建立 SystemController（如前述）
    # Step 4: 註冊策略（如前述）

    # 建立停止事件
    stop_event = asyncio.Event()

    # 註冊信號處理
    loop = asyncio.get_running_loop()
    for sig in (signal.SIGINT, signal.SIGTERM):
        loop.add_signal_handler(sig, stop_event.set)

    # 啟動所有元件
    async with pcs_1, pcs_2, bms, meter:       # Layer 2-3: 設備連線
        async with controller:                    # Layer 4-6: 控制器
            async with monitor:                   # Layer 8: 監控
                async with batcher:               # Layer 8: 通知
                    await stop_event.wait()        # 等待停止信號

    print("EMS shutdown complete.")

if __name__ == "__main__":
    asyncio.run(main())
```

注意 `async with` 的巢狀順序——底層元件先啟動，頂層元件後啟動；停止時反向執行。這保證了：
1. 控制器啟動前，所有設備已經連線就緒
2. 控制器停止後，設備連線才會關閉
3. 即使發生例外，所有元件的 `_on_stop()` 都會被呼叫

### Docker Compose 配置

```yaml
version: "3.8"

services:
  ems:
    image: your-org/ems:latest
    network_mode: host           # 需要存取 Modbus 設備
    restart: always
    environment:
      - TZ=Asia/Taipei
      - MONGO_URI=mongodb://mongo:27017/ems
      - REDIS_URL=redis://redis:6379
    depends_on:
      - mongo
      - redis

  mongo:
    image: mongo:7
    volumes:
      - /opt/ems/data/mongo:/data/db
    restart: always

  redis:
    image: redis:7-alpine
    volumes:
      - /opt/ems/data/redis:/data
    restart: always

  etcd:
    image: quay.io/coreos/etcd:v3.5
    command: >
      etcd
      --name etcd-node
      --listen-client-urls http://0.0.0.0:2379
      --advertise-client-urls http://etcd:2379
    restart: always
```

---

## 實戰經驗與踩坑記錄

在真實的工業專案中，筆者累積了一些書本上不會教的經驗。

### 踩坑一：Modbus 通訊超時不是 Bug，是常態

**現象**：開發環境一切正常，部署到現場後 Modbus 讀取偶爾超時。

**原因**：工業現場的網路環境遠比辦公室複雜。Modbus TCP 走的可能是一個被多台設備共用的工業交換機，背景流量、電磁干擾都可能影響延遲。

**解法**：
1. 使用 `CircuitBreaker` 保護通訊——連續 N 次超時後暫停讀取，避免堵塞
2. 適當調高超時時間（從開發環境的 1 秒調到 3 秒）
3. `AsyncModbusDevice` 內建的 `is_responsive` 屬性會在通訊異常時自動設為 `False`，ContextBuilder 會使用 `default` 值，控制迴路不會因為單台設備的通訊問題而崩潰

### 踩坑二：心跳寫入不能停

**現象**：切換到旁路模式時，PCS 收不到心跳，自動進入安全模式停機。

**原因**：旁路模式（`BypassStrategy`）的 `suppress_heartbeat` 被設為 `True`，導致心跳服務暫停。但 PCS 的看門狗不區分「控制器故意停」和「控制器故障」。

**解法**：csp_lib 的 `Strategy` 基礎類別有一個 `suppress_heartbeat` 屬性。`BypassStrategy` 預設為 `True`（旁路模式不發心跳）。如果你的設備需要持續收到心跳，可以覆寫這個行為：

```python
class SafeBypassStrategy(BypassStrategy):
    @property
    def suppress_heartbeat(self) -> bool:
        return False  # 即使旁路也繼續發心跳
```

### 踩坑三：告警遲滯值要現場調

**現象**：溫度告警在夏天不斷觸發和解除（抖動），值班人員被通知轟炸。

**原因**：開發時設定的 `HysteresisConfig(activate_threshold=1, clear_threshold=1)` 太敏感。現場設備的溫度感測器有 +-0.5 度的波動。

**解法**：根據現場實測數據調整遲滯：

```python
HysteresisConfig(
    activate_threshold=5,   # 連續 5 次超標才觸發（5 秒）
    clear_threshold=10,     # 連續 10 次正常才解除（10 秒）
)
```

告警解除的閾值應該明顯高於觸發閾值——因為誤解除（以為問題解決了但其實沒有）比誤觸發更危險。

### 踩坑四：功率分配要考慮離線設備

**現象**：一台 PCS 通訊斷線時，另一台 PCS 沒有自動承擔更多功率。

**原因**：`ProportionalDistributor` 只分配給 `is_responsive=True` 且 `is_protected=False` 的設備。當 PCS_1 離線時，它確實被排除了——但 `Command` 中的 `p_target` 仍是以整個系統為基礎計算的。

**解法**：這其實是正確行為。`SystemController` 在 `_build_device_snapshots()` 中會過濾掉非 responsive 和被保護的設備。如果原來的命令是 800kW，兩台各分 400kW，現在只剩一台，那這台就會收到 800kW——前提是它的額定容量足夠。如果超過額定容量，PCS 自身的保護機制會限制實際輸出。

### 踩坑五：etcd Leader Election 的 Split Brain

**現象**：極端情況下，兩個節點都認為自己是 leader，同時向設備發送命令。

**原因**：網路分區（network partition）——兩個節點都還活著，但無法和 etcd 通訊。

**解法**：
1. etcd 部署至少 3 個節點，避免單點故障
2. 在 `SystemController` 層面加入設備端的互斥寫入驗證
3. 設備端的看門狗機制是最終防線——即使兩個控制器同時寫入，設備只會以最後收到的值為準

---

## 效能數據

以下是在典型工業 PC（Intel i5, 8GB RAM, Ubuntu 22.04）上的實測數據：

| 指標 | 純 Python | Cython 編譯 |
|------|-----------|------------|
| 控制迴路延遲（4 設備） | ~50ms | ~30ms |
| 單次 Modbus TCP 讀取 | ~15ms | ~15ms（I/O 瓶頸） |
| ContextBuilder.build() | ~0.8ms | ~0.3ms |
| ProtectionGuard.apply() | ~0.1ms | ~0.05ms |
| 記憶體使用（穩態） | ~80MB | ~75MB |
| CPU 使用率（1Hz 控制迴路） | ~3% | ~2% |

幾個觀察：

1. **Modbus 通訊是瓶頸**。不論是否使用 Cython，單次 Modbus TCP 讀取的延遲約 15ms，這取決於網路和設備回應速度，Python 程式碼的執行時間佔比很小。

2. **Cython 的效能提升在計算密集的部分最明顯**。ContextBuilder 和 ProtectionGuard 的計算速度提升了 2-3 倍。

3. **記憶體使用非常節制**。即使連接 4 台設備、運行完整的控制迴路 + 監控 + 告警，穩態記憶體使用也只有 80MB 左右。

4. **CPU 使用率極低**。1Hz 的控制迴路在空閒時幾乎不消耗 CPU。即使是性能較弱的嵌入式設備也能輕鬆運行。

---

## 系列回顧與總結

歷經 28 篇文章，我們從最基礎的「REST API vs Modbus」開始，一路走到了完整的 1MW 儲能系統 EMS。讓我用一張表回顧整個系列的知識地圖：

| Part | 主題 | 關鍵收穫 |
|------|------|---------|
| Part 1 觀念轉換篇 | REST → Modbus 概念映射 | 暫存器模型、二進制編碼、持久連線 |
| Part 2 通訊基礎篇 | Modbus 資料類型與客戶端 | Float32/UInt16、TCP/RTU 客戶端、編解碼 |
| Part 3 設備建模篇 | 點位、轉換、告警 | ReadPoint/WritePoint、Pipeline、AlarmDefinition |
| Part 4 控制策略篇 | 策略模式與保護鏈 | Strategy/Command、ModeManager、ProtectionGuard |
| Part 5 管理層篇 | 資料持久化與排程 | AlarmPersistenceManager、DeviceManager |
| Part 6 系統整合篇 | 架構設計與實戰部署 | SystemController、HA、監控、EMS 案例 |

### 軟體工程師進入工業領域的三個關鍵心態

**第一，擁抱不完美的通訊**。在 Web 開發中，API 呼叫失敗通常意味著有 Bug 要修。在工業場域，通訊超時是正常現象。你的系統必須在「設備有時候不回應」的前提下設計——這就是為什麼 csp_lib 有 CircuitBreaker、`is_responsive` 過濾、default 值這些機制。

**第二，安全永遠優先於功能**。在 Web 應用中，你可以先上線再修 Bug。在工業控制中，一個 Bug 可能導致設備損壞或人員傷亡。ProtectionGuard 存在的意義不是讓系統更「聰明」，而是確保在最壞的情況下，系統的行為是安全的。

**第三，配置比程式碼重要**。在 csp_lib 的設計哲學中，新增一台設備不需要寫任何程式碼——只需要在 `DeviceRegistry` 中多一行註冊、在 `EquipmentTemplate` 中定義範本。映射驅動的配置模式讓系統的擴展性和維護性遠高於手寫膠水程式碼。

### csp_lib 的設計哲學總結

回顧整個框架的設計，有幾個一以貫之的原則：

1. **分層隔離**：八層架構，依賴只向下，每層獨立可測試
2. **不可變配置**：Frozen dataclass 確保配置在運行時不被意外修改
3. **協定驅動**：Protocol 而非繼承，結構型子型別而非名義型子型別
4. **事件驅動**：設備值變化透過事件傳播，上層不需要輪詢
5. **映射驅動**：ContextMapping 和 CommandMapping 取代手寫膠水程式碼
6. **按需組裝**：Optional dependencies 讓你只安裝需要的元件

### 寫在最後

工業控制軟體是一個正在快速演化的領域。十年前，EMS 還是用 C++ 和 OPC UA 寫的閉源系統。今天，Python 的生態系統已經成熟到可以處理從 Modbus 通訊到雲端整合的全鏈路。

csp_lib 不是要取代那些經過數十年驗證的工業控制平台，而是要降低軟體工程師進入這個領域的門檻。如果這個系列讓你覺得「工業控制其實也不過如此」，那它就達到了目的。

當然，「不過如此」的前提是你有一個好的抽象層幫你處理底層的複雜性。而這正是 csp_lib 存在的意義。

感謝你一路讀到這裡。期待在真實的儲能案場中，看到你用 csp_lib 打造的系統。
