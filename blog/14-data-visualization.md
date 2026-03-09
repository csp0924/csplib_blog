# 資料視覺化與 API 設計：讓工業數據說話

> **從零打造工業控制協定框架：給軟體工程師的 CSP Library 實戰系列**
> Part 3 — 資料管線篇 | Article 14

---

## 前言

在前三篇中，我們完成了工業資料管線的核心基礎設施：Redis 負責即時同步、MongoDB 負責持久化、事件系統負責串接一切。但對操作員和管理者來說，他們不關心你的 `MongoBatchUploader` 多高效、你的 `DeviceEventEmitter` 多優雅——他們需要的是一個**清晰的儀表板**，能在一眼之間看出：系統是否正常、功率曲線有無異常、告警是否需要處理。

本篇是 Part 3 資料管線篇的最後一章。我們將探討 csp_lib 的 FastAPI 層如何將工業數據轉化為結構化的 API，以及統計引擎如何進行能耗計算。這是「從 Modbus 暫存器到瀏覽器畫面」這條完整管線的最後一哩路。

---

## 1. 挑戰：讓數千個數據點產生意義

一個中型 EMS 系統可能有這些原始資料：

| 來源 | 數量 | 更新頻率 |
|------|------|---------|
| PCS 點位 | 5 台 x 40 點位 = 200 | 每秒 |
| 電表點位 | 3 台 x 20 點位 = 60 | 每秒 |
| 環境感測器 | 2 台 x 10 點位 = 20 | 每 5 秒 |
| 告警狀態 | ~50 種定義 | 事件觸發 |
| 控制狀態 | 模式、保護、命令 | 事件觸發 |
| **合計** | **~280 個數據點** | **~280/秒** |

把這 280 個原始數據點直接丟到前端是沒有意義的。API 層的職責是**聚合、過濾、結構化**——讓前端拿到的是「可以直接畫圖的資料」。

---

## 2. csp_lib 的 FastAPI 應用架構

### 2.1 App Factory 模式

csp_lib 使用 Factory 模式建立 FastAPI 應用，將 `SystemController` 作為核心依賴注入：

```python
from csp_lib.gui.app import create_app
from csp_lib.gui.config import GUIConfig

# SystemController 是整個系統的 Facade
app = create_app(
    system_controller=system_controller,
    config=GUIConfig(
        host="0.0.0.0",
        port=8080,
        cors_origins=["*"],
        snapshot_interval=5.0,
    ),
)
```

`create_app()` 內部完成所有初始化：

```python
def create_app(
    system_controller: SystemController,
    config: GUIConfig | None = None,
) -> FastAPI:
    config = config or GUIConfig()

    @asynccontextmanager
    async def lifespan(app: FastAPI) -> AsyncIterator[None]:
        # 啟動 EventBridge：設備事件 → WebSocket
        bridge = EventBridge(
            system_controller=app.state.system_controller,
            ws_manager=app.state.ws_manager,
            snapshot_interval=config.snapshot_interval,
        )
        app.state.event_bridge = bridge
        await bridge.attach()
        try:
            yield
        finally:
            await bridge.detach()

    app = FastAPI(title="CSP Control Panel", version="1.0.0", lifespan=lifespan)

    # 狀態存儲
    app.state.system_controller = system_controller
    app.state.ws_manager = WebSocketManager()

    # CORS 中介軟體
    app.add_middleware(
        CORSMiddleware,
        allow_origins=config.cors_origins,
        allow_credentials=True,
        allow_methods=["*"],
        allow_headers=["*"],
    )

    # 掛載 API 路由
    app.include_router(devices_router, prefix="/api")
    app.include_router(alarms_router, prefix="/api")
    app.include_router(commands_router, prefix="/api")
    app.include_router(modes_router, prefix="/api")
    app.include_router(health_router, prefix="/api")
    app.include_router(config_io_router, prefix="/api")
    app.include_router(ws_router)  # WebSocket 不需要 /api 前綴

    return app
```

### 2.2 依賴注入

csp_lib 使用 FastAPI 的 `Depends` 機制從 `app.state` 中提取依賴：

```python
# csp_lib/gui/dependencies.py

def get_system_controller(request: Request) -> SystemController:
    return request.app.state.system_controller

def get_registry(request: Request) -> DeviceRegistry:
    return request.app.state.system_controller.registry

def get_mode_manager(request: Request) -> ModeManager:
    return request.app.state.system_controller.mode_manager

def get_ws_manager(request: Request) -> WebSocketManager:
    return request.app.state.ws_manager
```

這些 dependency function 讓 Router 層不需要直接存取 `app.state`，保持了層級邊界的清潔。

---

## 3. API 端點設計

### 3.1 設備狀態 API

**列出所有設備**

```
GET /api/devices
```

```python
@router.get("/devices")
def list_devices(
    registry: DeviceRegistry = Depends(get_registry)
) -> list[dict[str, Any]]:
    return [_serialize_device(dev, registry) for dev in registry.all_devices]
```

回應範例：

```json
[
    {
        "device_id": "pcs_1",
        "is_connected": true,
        "is_responsive": true,
        "is_protected": false,
        "config": {
            "device_id": "pcs_1",
            "unit_id": 1,
            "address_offset": 0,
            "read_interval": 1.0,
            "reconnect_interval": 5.0,
            "disconnect_threshold": 5
        },
        "traits": ["pcs", "inverter"],
        "active_alarm_count": 0
    }
]
```

**按 Trait 查詢設備**

```
GET /api/devices/by-trait/pcs
```

```python
@router.get("/devices/by-trait/{trait}")
def get_devices_by_trait(
    trait: str,
    registry: DeviceRegistry = Depends(get_registry),
) -> list[dict[str, Any]]:
    return [
        _serialize_device(dev, registry)
        for dev in registry.get_devices_by_trait(trait)
    ]
```

Trait 是 csp_lib 的設備分類機制——例如 `"pcs"` 代表所有儲能變流器，`"meter"` 代表所有電表。

**取得設備詳情（含最新值和告警）**

```
GET /api/devices/pcs_1
```

```python
@router.get("/devices/{device_id}")
def get_device(
    device_id: str,
    registry: DeviceRegistry = Depends(get_registry),
) -> dict[str, Any]:
    device = registry.get_device(device_id)
    if device is None:
        raise HTTPException(status_code=404, detail=f"Device '{device_id}' not found")

    info = _serialize_device(device, registry)
    info["latest_values"] = device.latest_values
    info["active_alarms"] = [_serialize_alarm_state(a) for a in device.active_alarms]
    return info
```

回應範例：

```json
{
    "device_id": "pcs_1",
    "is_connected": true,
    "is_responsive": true,
    "is_protected": false,
    "latest_values": {
        "active_power": 50.2,
        "reactive_power": 10.1,
        "soc": 85.3,
        "dc_voltage": 720.5,
        "grid_frequency": 59.98
    },
    "active_alarms": [
        {
            "code": "HIGH_TEMP",
            "name": "高溫警告",
            "level": "WARNING",
            "is_active": true,
            "activated_at": "2026-03-10T07:45:00Z",
            "duration": 900.5
        }
    ]
}
```

**僅取得最新點位值**

```
GET /api/devices/pcs_1/values
```

```python
@router.get("/devices/{device_id}/values")
def get_device_values(
    device_id: str,
    registry: DeviceRegistry = Depends(get_registry),
) -> dict[str, Any]:
    device = registry.get_device(device_id)
    if device is None:
        raise HTTPException(status_code=404, detail=f"Device '{device_id}' not found")
    return device.latest_values
```

這是前端儀表板最常呼叫的端點——輕量、快速，只回傳數值。

### 3.2 告警管理 API

**列出所有告警**

```
GET /api/alarms
```

```python
@router.get("/alarms")
def list_all_alarms(
    registry: DeviceRegistry = Depends(get_registry)
) -> list[dict[str, Any]]:
    alarms = []
    for device in registry.all_devices:
        for alarm in device.active_alarms:
            alarms.append(_serialize_alarm(device.device_id, alarm))
    return alarms
```

回應範例：

```json
[
    {
        "device_id": "pcs_1",
        "code": "HIGH_TEMP",
        "name": "高溫警告",
        "level": "WARNING",
        "is_active": true,
        "activated_at": "2026-03-10T07:45:00Z",
        "duration": 900.5
    },
    {
        "device_id": "pcs_3",
        "code": "COMM_FAULT",
        "name": "通訊故障",
        "level": "CRITICAL",
        "is_active": true,
        "activated_at": "2026-03-10T08:00:00Z",
        "duration": 60.2
    }
]
```

**清除告警**

```
POST /api/alarms/pcs_1/HIGH_TEMP/clear
```

```python
@router.post("/alarms/{device_id}/{code}/clear")
async def clear_alarm(
    device_id: str,
    code: str,
    registry: DeviceRegistry = Depends(get_registry),
) -> dict[str, str]:
    device = registry.get_device(device_id)
    if device is None:
        raise HTTPException(status_code=404, detail=f"Device '{device_id}' not found")
    await device.clear_alarm(code)
    return {"status": "ok"}
```

### 3.3 指令與寫入 API

**寫入設備點位**

```
POST /api/devices/pcs_1/write
Content-Type: application/json

{"point_name": "active_power_sp", "value": 100, "verify": false}
```

```python
class WriteRequest(BaseModel):
    point_name: str
    value: Any
    verify: bool = False

@router.post("/devices/{device_id}/write")
async def write_to_device(
    device_id: str,
    body: WriteRequest,
    registry: DeviceRegistry = Depends(get_registry),
) -> dict[str, Any]:
    device = registry.get_device(device_id)
    if device is None:
        raise HTTPException(status_code=404, detail=f"Device '{device_id}' not found")

    result = await device.write(body.point_name, body.value, verify=body.verify)
    return {
        "status": result.status.name,
        "point_name": result.point_name,
        "value": result.value,
        "error_message": result.error_message,
    }
```

**列出可寫入的點位**

```
GET /api/devices/pcs_1/write-points
```

```python
@router.get("/devices/{device_id}/write-points")
def list_write_points(
    device_id: str,
    registry: DeviceRegistry = Depends(get_registry),
) -> list[str]:
    device = registry.get_device(device_id)
    if device is None:
        raise HTTPException(status_code=404, detail=f"Device '{device_id}' not found")
    return sorted(device._write_points.keys())
```

**手動觸發策略執行**

```
POST /api/executor/trigger
```

```python
@router.post("/executor/trigger")
def trigger_executor(
    sc: SystemController = Depends(get_system_controller),
) -> dict[str, str]:
    sc.trigger()
    return {"status": "triggered"}
```

### 3.4 模式管理 API

**查看模式狀態**

```
GET /api/modes
```

```python
@router.get("/modes")
def get_modes(
    sc: SystemController = Depends(get_system_controller)
) -> dict[str, Any]:
    mm = sc.mode_manager
    modes_info = []
    for name, mode_def in mm.registered_modes.items():
        if name.startswith("__"):
            continue
        modes_info.append({
            "name": mode_def.name,
            "priority": mode_def.priority,
            "description": mode_def.description,
            "strategy_type": type(mode_def.strategy).__name__,
        })

    effective = mm.effective_mode
    return {
        "registered_modes": modes_info,
        "base_mode_names": mm.base_mode_names,
        "active_override_names": mm.active_override_names,
        "effective_mode": effective.name if effective else None,
    }
```

回應範例：

```json
{
    "registered_modes": [
        {
            "name": "pq_dispatch",
            "priority": 10,
            "description": "PQ 調度模式",
            "strategy_type": "PQStrategy"
        },
        {
            "name": "island_mode",
            "priority": 50,
            "description": "孤島運行模式",
            "strategy_type": "IslandStrategy"
        }
    ],
    "base_mode_names": ["pq_dispatch"],
    "active_override_names": [],
    "effective_mode": "pq_dispatch"
}
```

**切換模式**

```python
# 設定基礎模式（替換所有現有基礎模式）
POST /api/modes/base
{"name": "pq_dispatch"}

# 新增基礎模式（多基礎共存）
POST /api/modes/base/add
{"name": "frequency_regulation"}

# 推入 Override 模式
POST /api/modes/override/push
{"name": "emergency_mode"}

# 移除 Override 模式
POST /api/modes/override/pop
{"name": "emergency_mode"}
```

**動態更新策略配置**

```
PUT /api/modes/pq_dispatch/config
Content-Type: application/json

{"p_target": 100.0, "q_target": 0.0}
```

```python
@router.put("/modes/{name}/config")
def update_mode_config(
    name: str,
    body: ModeConfigRequest,
    sc: SystemController = Depends(get_system_controller),
) -> dict[str, str]:
    mm = sc.mode_manager
    modes = mm.registered_modes
    if name not in modes:
        raise HTTPException(status_code=404, detail=f"Mode '{name}' not found")

    strategy = modes[name].strategy
    if not hasattr(strategy, "update_config"):
        raise HTTPException(
            status_code=400,
            detail="Strategy does not support config update"
        )

    strategy.update_config(body.config)
    return {"status": "ok"}
```

### 3.5 保護狀態 API

```
GET /api/protection
```

```python
@router.get("/protection")
def get_protection_status(
    sc: SystemController = Depends(get_system_controller),
) -> dict[str, Any]:
    result = sc.protection_status
    if result is None:
        return {"status": "no_data"}

    return {
        "was_modified": result.was_modified,
        "triggered_rules": result.triggered_rules,
        "original_command": {
            "p_target": result.original_command.p_target,
            "q_target": result.original_command.q_target,
        },
        "protected_command": {
            "p_target": result.protected_command.p_target,
            "q_target": result.protected_command.q_target,
        },
    }
```

回應範例：

```json
{
    "was_modified": true,
    "triggered_rules": ["soc_lower_limit", "ramp_rate"],
    "original_command": {"p_target": 200.0, "q_target": 0.0},
    "protected_command": {"p_target": 150.0, "q_target": 0.0}
}
```

### 3.6 系統健康 API

```
GET /api/health
```

```python
@router.get("/health")
def get_health(
    sc: SystemController = Depends(get_system_controller)
) -> dict[str, Any]:
    report = sc.health()
    return _serialize_health(report)

def _serialize_health(report: HealthReport) -> dict[str, Any]:
    return {
        "status": report.status.value,
        "component": report.component,
        "message": report.message,
        "details": report.details,
        "children": [_serialize_health(c) for c in report.children],
    }
```

回應範例：

```json
{
    "status": "degraded",
    "component": "system",
    "message": null,
    "details": {},
    "children": [
        {
            "status": "healthy",
            "component": "device:pcs_1",
            "details": {
                "connected": true,
                "responsive": true,
                "protected": false,
                "active_alarms": 0
            },
            "children": []
        },
        {
            "status": "unhealthy",
            "component": "device:pcs_3",
            "details": {
                "connected": false,
                "responsive": false,
                "protected": false,
                "active_alarms": 1
            },
            "children": []
        }
    ]
}
```

---

## 4. 即時儀表板：WebSocket 事件串流

REST API 適合「請求-回應」模式，但儀表板需要**即時更新**。csp_lib 透過 WebSocket 提供六種即時事件：

### 4.1 WebSocket 訊息格式

所有 WebSocket 訊息遵循統一格式：

```json
{
    "type": "事件類型",
    "device_id": "設備 ID（如適用）",
    "data": { ... },
    "timestamp": "ISO 8601"
}
```

### 4.2 事件類型與資料結構

**value_change** — 點位值變化

```json
{
    "type": "value_change",
    "device_id": "pcs_1",
    "data": {
        "point_name": "active_power",
        "old_value": 49.8,
        "new_value": 50.2
    },
    "timestamp": "2026-03-10T08:00:00.123Z"
}
```

**read_complete** — 讀取完成（含所有值）

```json
{
    "type": "read_complete",
    "device_id": "pcs_1",
    "data": {
        "values": {
            "active_power": 50.2,
            "reactive_power": 10.1,
            "soc": 85.3
        },
        "duration_ms": 45.2
    },
    "timestamp": "2026-03-10T08:00:00.123Z"
}
```

**alarm_triggered / alarm_cleared** — 告警變化

```json
{
    "type": "alarm_triggered",
    "device_id": "pcs_1",
    "data": {
        "code": "HIGH_TEMP",
        "name": "高溫警告",
        "level": "WARNING"
    },
    "timestamp": "2026-03-10T08:00:00.123Z"
}
```

**connected / disconnected** — 連線狀態

```json
{
    "type": "disconnected",
    "device_id": "pcs_3",
    "data": {
        "reason": "連續 5 次讀取失敗",
        "consecutive_failures": 5
    },
    "timestamp": "2026-03-10T08:00:00.123Z"
}
```

**snapshot** — 完整系統快照（定期推送）

```json
{
    "type": "snapshot",
    "data": {
        "devices": [
            {
                "device_id": "pcs_1",
                "is_connected": true,
                "is_responsive": true,
                "is_protected": false,
                "latest_values": {"active_power": 50.2, "soc": 85.3},
                "active_alarm_count": 0
            }
        ],
        "mode": {
            "base_mode_names": ["pq_dispatch"],
            "active_override_names": [],
            "effective_mode": "pq_dispatch"
        },
        "auto_stop_active": false,
        "is_running": true
    },
    "timestamp": "2026-03-10T08:00:00.123Z"
}
```

### 4.3 前端連接範例

```javascript
const ws = new WebSocket("ws://localhost:8080/ws");

ws.onmessage = (event) => {
    const msg = JSON.parse(event.data);

    switch (msg.type) {
        case "value_change":
            updatePointValue(msg.device_id, msg.data);
            break;
        case "alarm_triggered":
            showAlarmNotification(msg.device_id, msg.data);
            break;
        case "snapshot":
            refreshDashboard(msg.data);
            break;
        case "read_complete":
            updateDevicePanel(msg.device_id, msg.data.values);
            break;
    }
};
```

---

## 5. 統計引擎：能耗計算

csp_lib 的統計模組提供工業級的能耗計算，核心是 `StatisticsEngine` 和 `DeviceEnergyTracker`。

### 5.1 配置定義

```python
from csp_lib.statistics import (
    StatisticsConfig,
    MetricDefinition,
    PowerSumDefinition,
    DeviceMeterType,
)

config = StatisticsConfig(
    metrics=[
        # PCS：瞬時型，讀取 active_power (kW)，梯形積分算 kWh
        MetricDefinition(
            device_id="pcs_1",
            meter_type=DeviceMeterType.INSTANTANEOUS,
            point_name="active_power",
        ),
        MetricDefinition(
            device_id="pcs_2",
            meter_type=DeviceMeterType.INSTANTANEOUS,
            point_name="active_power",
        ),
        # 電表：累計型，讀取 kwh_total，差值計算區間能耗
        MetricDefinition(
            device_id="meter_1",
            meter_type=DeviceMeterType.CUMULATIVE,
            point_name="kwh_total",
        ),
    ],
    power_sums=[
        # 跨設備功率加總：所有 PCS 的 active_power 總和
        PowerSumDefinition(
            name="p_total_pcs",
            trait="pcs",
            point_name="active_power",
        ),
    ],
    intervals_minutes=(15, 30, 60),  # 15/30/60 分鐘統計
    collection_name="statistics",
)
```

### 5.2 兩種電表類型

**累計型 (CUMULATIVE)**：電表回報的是累計 kWh。區間能耗 = 末值 - 首值。

```
時間   08:00  08:05  08:10  08:15
讀數   1000   1003   1007   1012

15 分鐘能耗 = 1012 - 1000 = 12 kWh
```

**瞬時型 (INSTANTANEOUS)**：設備回報的是瞬時功率 kW。使用梯形積分計算能耗。

```
時間   08:00  08:05  08:10  08:15
功率   50kW   52kW   48kW   51kW

kWh ≈ (50+52)/2 × 5/60 + (52+48)/2 × 5/60 + (48+51)/2 × 5/60
    = 4.25 + 4.17 + 4.13 = 12.55 kWh
```

`IntervalAccumulator` 透過 `DeviceMeterType` 自動選擇計算方式：

```python
class IntervalAccumulator:
    def _accumulate(self, value: float, timestamp: datetime) -> None:
        self._sample_count += 1
        if self._meter_type == DeviceMeterType.CUMULATIVE:
            self._last_value = value  # 只記錄最新值
        else:
            # 梯形積分
            if self._prev_timestamp is not None and self._prev_value is not None:
                dt_hours = (timestamp - self._prev_timestamp).total_seconds() / 3600.0
                self._kwh_accumulated += (self._prev_value + value) / 2.0 * dt_hours
            self._prev_value = value
            self._prev_timestamp = timestamp

    def _finalize(self, boundary: datetime) -> IntervalRecord:
        if self._meter_type == DeviceMeterType.CUMULATIVE:
            kwh = (self._last_value or 0.0) - (self._first_value or 0.0)
        else:
            kwh = self._kwh_accumulated

        return IntervalRecord(
            device_id=self._device_id,
            interval_minutes=self._interval_minutes,
            period_start=self._period_start,
            period_end=boundary,
            kwh=kwh,
            sample_count=self._sample_count,
            meter_type=self._meter_type.value,
        )
```

### 5.3 時鐘對齊

區間邊界對齊到整點時刻：15 分鐘區間對齊到 :00/:15/:30/:45，60 分鐘區間對齊到整點。

```python
def _floor_timestamp(self, ts: datetime) -> datetime:
    floored_minute = (ts.minute // self._interval_minutes) * self._interval_minutes
    return ts.replace(minute=floored_minute, second=0, microsecond=0)
```

例如，收到 08:17:23 的讀數時，15 分鐘區間會歸入 08:15:00 ~ 08:30:00。

### 5.4 多區間追蹤

`DeviceEnergyTracker` 同時維護多個區間的累積器：

```python
class DeviceEnergyTracker:
    def __init__(
        self,
        device_id: str,
        intervals: tuple[int, ...],    # 例如 (15, 30, 60)
        meter_type: DeviceMeterType,
    ) -> None:
        self._accumulators = [
            IntervalAccumulator(device_id, interval, meter_type)
            for interval in intervals
        ]

    def feed(self, value: float, timestamp: datetime) -> list[IntervalRecord]:
        records = []
        for acc in self._accumulators:
            record = acc.feed(value, timestamp)
            if record is not None:
                records.append(record)
        return records
```

### 5.5 功率加總

`StatisticsEngine` 追蹤跨設備的即時功率加總：

```python
class StatisticsEngine:
    def process_read(
        self, device_id: str, values: dict[str, Any], timestamp: datetime
    ) -> list[IntervalRecord]:
        records = []

        # 能耗追蹤
        point_name = self._metric_point_map.get(device_id)
        if point_name and point_name in values:
            tracker = self._trackers[device_id]
            records.extend(tracker.feed(values[point_name], timestamp))

        # 功率加總追蹤
        for sum_name, ps_point in self._device_power_sums.get(device_id, []):
            if ps_point in values:
                self._power_device_values[sum_name][device_id] = float(values[ps_point])

        return records

    def get_power_sum(self, name: str) -> float:
        """取得即時功率加總"""
        return sum(self._power_device_values.get(name, {}).values())

    def get_all_power_sums(self) -> dict[str, float]:
        """取得所有功率加總"""
        return {name: self.get_power_sum(name) for name in self._power_device_values}
```

### 5.6 StatisticsManager：事件驅動

```python
from csp_lib.statistics import StatisticsManager, StatisticsConfig

# 建立統計管理器
stats_manager = StatisticsManager(
    config=config,
    uploader=uploader,      # MongoBatchUploader
    registry=registry,      # DeviceRegistry（解析 trait → device_ids）
)

# 訂閱設備
for device in devices:
    stats_manager.subscribe(device)

# 查詢即時功率加總
total_pcs_power = stats_manager.engine.get_power_sum("p_total_pcs")
all_sums = stats_manager.engine.get_all_power_sums()
```

`StatisticsManager` 訂閱設備的 `read_complete` 事件，自動驅動統計引擎並將完成的區間記錄上傳到 MongoDB：

```python
async def _on_read_complete(self, payload: ReadCompletePayload) -> None:
    records = self._engine.process_read(
        payload.device_id, payload.values, payload.timestamp
    )

    # 上傳能耗記錄
    for record in records:
        document = {
            "type": "energy",
            "device_id": record.device_id,
            "interval_minutes": record.interval_minutes,
            "period_start": record.period_start,
            "period_end": record.period_end,
            "kwh": record.kwh,
            "sample_count": record.sample_count,
            "meter_type": record.meter_type,
            "timestamp": datetime.now(timezone.utc),
        }
        await self._uploader.enqueue(self._collection_name, document)

    # 上傳功率加總記錄
    if records and self._config.power_sums:
        seen = set()
        for record in records:
            if record.interval_minutes not in seen:
                seen.add(record.interval_minutes)
                power_records = self._engine.build_power_sum_records(
                    record.interval_minutes,
                    record.period_start,
                    record.period_end,
                )
                for pr in power_records:
                    document = {
                        "type": "power_sum",
                        "name": pr.name,
                        "interval_minutes": pr.interval_minutes,
                        "period_start": pr.period_start,
                        "period_end": pr.period_end,
                        "total_power": pr.total_power,
                        "device_count": pr.device_count,
                        "timestamp": datetime.now(timezone.utc),
                    }
                    await self._uploader.enqueue(self._collection_name, document)
```

---

## 6. 關鍵指標與視覺化建議

### 6.1 即時監控面板

| 指標 | 資料來源 | 更新方式 |
|------|---------|---------|
| PCS 輸出功率 | WebSocket `read_complete` | 即時 |
| SOC 曲線 | WebSocket `value_change` | 即時 |
| 設備連線狀態 | WebSocket `snapshot` | 5 秒 |
| 活躍告警列表 | WebSocket `alarm_*` + REST API | 即時 + 輪詢 |
| PCS 功率加總 | REST API 查詢 StatisticsEngine | 輪詢 |

### 6.2 歷史趨勢

| 指標 | 資料來源 | 查詢方式 |
|------|---------|---------|
| 24H 功率曲線 | MongoDB `device_data` | Aggregation Pipeline |
| 每日能耗 | MongoDB `statistics` (type=energy) | $match + $group |
| 月度功率加總 | MongoDB `statistics` (type=power_sum) | $match + $sort |
| 告警頻率 | MongoDB 告警記錄 | $group by code |

### 6.3 系統健康總覽

```
┌─────────────────────────────────────────┐
│         系統健康狀態: DEGRADED           │
│                                         │
│  ● pcs_1: HEALTHY    ● meter_1: HEALTHY │
│  ● pcs_2: HEALTHY    ● meter_2: HEALTHY │
│  ● pcs_3: UNHEALTHY  ● env_1: HEALTHY   │
│                                         │
│  模式: pq_dispatch                       │
│  保護: soc_lower_limit (已觸發)          │
│  自動停機: 未啟用                         │
└─────────────────────────────────────────┘
```

---

## 7. 最佳實踐

### 7.1 即時 vs 歷史資料分離

```
即時資料:
  - Redis Hash (device:*:state) → 最新值查詢
  - WebSocket → 前端即時更新
  - REST API /devices/{id}/values → Polling fallback

歷史資料:
  - MongoDB device_data → 原始讀數
  - MongoDB statistics → 區間統計
  - REST API 查詢 MongoDB → 趨勢圖表
```

### 7.2 告警通知閾值

不是所有告警都需要立即通知。csp_lib 的告警有等級（Level）區分：

| Level | 處理方式 | 通知方式 |
|-------|---------|---------|
| INFO | 記錄 | 僅儀表板顯示 |
| WARNING | 記錄 + 標註 | WebSocket + 儀表板高亮 |
| CRITICAL | 記錄 + 保護動作 | WebSocket + 推播通知 + 聲音警報 |
| EMERGENCY | 立即停機 | 全通道通知 |

### 7.3 Downsampling 策略

| 時間範圍 | 建議粒度 | 資料來源 |
|---------|---------|---------|
| 即時 (~1 分鐘) | 原始（1 秒） | WebSocket |
| 短期 (1-24 小時) | 5-15 秒 | MongoDB 原始資料 |
| 中期 (1-7 天) | 1-5 分鐘 | MongoDB + Aggregation |
| 長期 (1-12 月) | 15-60 分鐘 | MongoDB statistics |
| 歷史 (1+ 年) | 每日 | MongoDB + Pre-computed |

---

## 8. 完整架構回顧

```
                        ┌──────────┐
                        │  瀏覽器   │
                        │          │
                        │  REST    │ ← GET /api/devices
                        │  WS      │ ← ws://host/ws
                        └────┬─────┘
                             │
                    ┌────────┴────────┐
                    │   FastAPI App    │
                    │                  │
                    │  devices router  │
                    │  alarms router   │
                    │  commands router │
                    │  modes router    │
                    │  health router   │
                    │  ws router       │
                    │  EventBridge     │
                    └────────┬────────┘
                             │ Depends()
                    ┌────────┴────────┐
                    │ SystemController │
                    │                  │
                    │  DeviceRegistry  │
                    │  ModeManager     │
                    │  ProtectionGuard │
                    └────────┬────────┘
                             │
              ┌──────────────┼──────────────┐
              │              │              │
     ┌────────┴──┐  ┌───────┴───┐  ┌───────┴───┐
     │StateSyncMgr│  │DataUpload │  │Statistics │
     │  → Redis   │  │  → Mongo  │  │  → Mongo  │
     └────────────┘  └───────────┘  └───────────┘
              │              │              │
     ┌────────┴──────────────┴──────────────┴──┐
     │           AsyncModbusDevice              │
     │        DeviceEventEmitter               │
     │                                          │
     │  emit: value_change, read_complete,     │
     │        alarm_triggered, connected, ...   │
     └──────────────────────────────────────────┘
```

---

## 9. 重點回顧

| 面向 | 設計決策 | 原因 |
|------|---------|------|
| App Factory | `create_app()` 接受 SystemController | 測試友善、配置靈活 |
| 依賴注入 | `Depends(get_registry)` | 層級解耦，Router 不直接存取 state |
| 即時 + 輪詢 | WebSocket 事件 + REST Polling | 即時更新 + 可靠的初始狀態 |
| Snapshot | 定期 5 秒全狀態廣播 | 新客戶端快速同步 |
| 統計引擎 | 累計型 + 瞬時型雙模式 | 適應不同設備特性 |
| 時鐘對齊 | 區間對齊到整點 | 統計結果一致性 |
| 多區間追蹤 | 15/30/60 分鐘同步計算 | 一次讀取產生多種粒度 |

---

## Part 3 資料管線篇總結

回顧 Part 3 的四篇文章，我們建立了完整的工業資料管線：

| 篇章 | 主題 | 核心元件 |
|------|------|---------|
| Article 11 | Redis 即時資料匯流排 | RedisClient, StateSyncManager, ClusterStatePublisher |
| Article 12 | MongoDB 時序資料持久化 | MongoBatchUploader, DataUploadManager |
| Article 13 | 事件訂閱系統 | DeviceEventEmitter, EventBridge, WebSocketManager |
| Article 14 | API 與視覺化 | FastAPI REST, WebSocket, StatisticsEngine |

資料流的完整路徑：

```
Modbus 暫存器 → AsyncModbusDevice → DeviceEventEmitter
    → StateSyncManager → Redis（即時）
    → DataUploadManager → MongoBatchUploader → MongoDB（歷史）
    → StatisticsManager → StatisticsEngine → MongoDB（統計）
    → EventBridge (GUI) → WebSocketManager → 瀏覽器
    → REST API → 瀏覽器
```

---

## 下篇預告

Part 3 完成了資料從設備到前端的完整管線。Part 4 將進入**叢集與高可用**的世界——當你的 EMS 需要 7x24 不間斷運行時，如何實現 Leader 選舉、故障轉移、以及多節點協調？我們將從 etcd 選舉機制開始，逐步建構一個工業級的高可用架構。

> **下一篇：** Part 4 — 叢集與高可用篇（敬請期待）
