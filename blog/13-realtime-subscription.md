# 即時訂閱系統：從設備事件到前端推送

> **從零打造工業控制協定框架：給軟體工程師的 CSP Library 實戰系列**
> Part 3 — 資料管線篇 | Article 13

---

## 前言

在前兩篇中，我們建立了資料管線的兩個端點：Redis 負責即時狀態同步，MongoDB 負責歷史資料持久化。但這兩者都是「後端到後端」的資料流。真正讓系統活起來的，是**事件驅動架構**——設備的每一次讀取、每一個告警、每一次連線狀態變化，都應該像漣漪一樣擴散到所有關心它的訂閱者。

本篇深入 csp_lib 的事件系統核心，從底層的 `DeviceEventEmitter` 設計哲學，到上層的 WebSocket 即時推送，完整呈現一個工業級的事件管線。

---

## 1. 事件驅動架構概觀

csp_lib 的事件系統分為三層：

```
┌────────────────────────────────────────────┐
│  Layer 3: 傳輸層 (Delivery)                 │
│  WebSocket → 瀏覽器                         │
│  Redis Pub/Sub → 跨程序                     │
│  MongoDB → 持久化                           │
├────────────────────────────────────────────┤
│  Layer 2: 管理層 (Management)               │
│  StateSyncManager → Redis                  │
│  DataUploadManager → MongoDB               │
│  StatisticsManager → 統計計算               │
│  EventBridge (GUI) → WebSocket             │
├────────────────────────────────────────────┤
│  Layer 1: 設備層 (Source)                   │
│  DeviceEventEmitter → 非同步事件佇列         │
│  AsyncModbusDevice → 事件產生者              │
└────────────────────────────────────────────┘
```

事件從設備層產生，經過管理層的多個訂閱者處理，最終到達傳輸層的各個目的地。每一層都是解耦的——設備不知道誰在監聽，管理層不關心事件如何被傳送。

---

## 2. 設備事件類型

csp_lib 定義了 11 種設備事件，涵蓋設備生命週期的所有關鍵時刻：

```python
# csp_lib/equipment/device/events.py

EVENT_CONNECTED = "connected"           # 連線成功
EVENT_DISCONNECTED = "disconnected"     # 斷線
EVENT_READ_COMPLETE = "read_complete"   # 讀取完成
EVENT_READ_ERROR = "read_error"         # 讀取錯誤
EVENT_VALUE_CHANGE = "value_change"     # 值變化
EVENT_ALARM_TRIGGERED = "alarm_triggered"  # 告警觸發
EVENT_ALARM_CLEARED = "alarm_cleared"      # 告警解除
EVENT_WRITE_COMPLETE = "write_complete"    # 寫入完成
EVENT_WRITE_ERROR = "write_error"          # 寫入錯誤
EVENT_RECONFIGURED = "reconfigured"        # 設備重新配置
EVENT_RESTARTED = "restarted"              # 設備重啟
EVENT_POINT_TOGGLED = "point_toggled"      # 點位啟用/停用
```

每種事件都有對應的 Payload dataclass，攜帶結構化的事件資料：

### 2.1 ValueChangePayload

最常用的事件——設備點位值發生變化：

```python
@dataclass(frozen=True)
class ValueChangePayload:
    device_id: str
    point_name: str
    old_value: Any
    new_value: Any
    timestamp: datetime = field(
        default_factory=lambda: datetime.now(timezone.utc)
    )
```

`frozen=True` 確保 Payload 在事件處理過程中不會被意外修改——這在多個 handler 共享同一個 Payload 時很重要。

### 2.2 ReadCompletePayload

每次讀取循環完成時觸發，攜帶所有點位的最新值：

```python
@dataclass(frozen=True)
class ReadCompletePayload:
    device_id: str
    values: dict[str, Any]    # 所有點位的最新值
    duration_ms: float         # 本次讀取耗時（毫秒）
    timestamp: datetime = field(
        default_factory=lambda: datetime.now(timezone.utc)
    )
```

這是 `DataUploadManager` 和 `StatisticsManager` 的主要資料來源。

### 2.3 DeviceAlarmPayload

告警事件，攜帶完整的告警資訊：

```python
@dataclass(frozen=True)
class DeviceAlarmPayload:
    device_id: str
    alarm_event: AlarmEvent    # 包含告警定義和觸發/清除狀態
    timestamp: datetime = field(
        default_factory=lambda: datetime.now(timezone.utc)
    )
```

### 2.4 DisconnectPayload

斷線事件，包含斷線原因和連續失敗次數：

```python
@dataclass(frozen=True)
class DisconnectPayload:
    device_id: str
    reason: str                # 斷線原因描述
    consecutive_failures: int  # 連續失敗次數
    timestamp: datetime = field(
        default_factory=lambda: datetime.now(timezone.utc)
    )
```

### 2.5 其他 Payload

```python
@dataclass(frozen=True)
class ConnectedPayload:
    device_id: str
    timestamp: datetime

@dataclass(frozen=True)
class ReadErrorPayload:
    device_id: str
    error: str
    consecutive_failures: int
    timestamp: datetime

@dataclass(frozen=True)
class WriteCompletePayload:
    device_id: str
    point_name: str
    value: Any
    timestamp: datetime

@dataclass(frozen=True)
class WriteErrorPayload:
    device_id: str
    point_name: str
    value: Any
    error: str
    timestamp: datetime

@dataclass(frozen=True)
class ReconfiguredPayload:
    device_id: str
    changed_sections: tuple[str, ...]  # 哪些部分被重新配置
    timestamp: datetime

@dataclass(frozen=True)
class PointToggledPayload:
    device_id: str
    point_name: str
    enabled: bool
    timestamp: datetime
```

---

## 3. DeviceEventEmitter：非阻塞事件引擎

`DeviceEventEmitter` 是 csp_lib 事件系統的核心。它的設計目標是：**事件發射不能阻塞設備的讀取循環**。

### 3.1 為什麼需要非阻塞？

考慮這個場景：PCS 設備每 500ms 讀取一次。如果 `value_change` 事件的 handler 需要 200ms（例如寫入 Redis），而一次讀取產生 5 個值變化事件，那麼事件處理就要花 1 秒——直接超過讀取間隔。

```
不好的設計：同步處理
read_loop: [讀取 500ms][處理事件 1000ms][讀取 500ms][處理事件...]
                        ^^^^^^^^^^^^
                        阻塞了讀取循環！
```

### 3.2 Queue-based 架構

`DeviceEventEmitter` 使用 `asyncio.Queue` 將事件產生和處理解耦：

```python
class DeviceEventEmitter:
    def __init__(self, max_queue_size: int = 10000) -> None:
        self._handlers: dict[str, list[AsyncHandler]] = {}
        self._queue: asyncio.Queue[tuple[str, Any]] = asyncio.Queue(
            maxsize=max_queue_size
        )
        self._worker_task: asyncio.Task[None] | None = None
        self._running = False
```

事件流程：

```
發射端（讀取循環）              消費端（Worker）
     │                              │
     │  emit("value_change", p)     │
     │ ────────> Queue ────────>    │ handler_1(p)
     │  (非阻塞 put_nowait)         │ handler_2(p)
     │                              │ handler_3(p)
     │  繼續讀取...                  │
```

### 3.3 emit() vs emit_await()

csp_lib 提供了兩種發射方式：

**`emit()`** — 非阻塞，放入佇列即返回：

```python
def emit(self, event: str, payload: Any = None) -> None:
    # 沒有監聽器就不入隊——避免無謂的佇列操作
    if not self._handlers.get(event):
        return
    try:
        self._queue.put_nowait((event, payload))
    except asyncio.QueueFull:
        logger.warning("事件佇列已滿，丟棄事件: event=%s", event)
```

**`emit_await()`** — 阻塞，等待所有 handler 執行完畢：

```python
async def emit_await(self, event: str, payload: Any = None) -> None:
    await self._process_event(event, payload)
```

這兩種方式的使用場景不同：

| 方式 | 使用場景 | 範例 |
|------|---------|------|
| `emit()` | 高頻事件，允許延遲處理 | `value_change`, `read_complete` |
| `emit_await()` | 關鍵事件，必須立即處理 | `connected`, `disconnected`, `alarm_triggered` |

在 `AsyncModbusDevice` 的實際程式碼中，你可以看到這個區分：

```python
# 連線成功——必須等待處理完成（告知所有訂閱者）
await self._emitter.emit_await(
    EVENT_CONNECTED,
    ConnectedPayload(device_id=self._config.device_id)
)

# 值變化——放入佇列即返回（不影響讀取速度）
self._emitter.emit(
    EVENT_VALUE_CHANGE,
    ValueChangePayload(
        device_id=self._config.device_id,
        point_name=name,
        old_value=old_value,
        new_value=new_value,
    ),
)

# 讀取完成——放入佇列即返回
self._emitter.emit(
    EVENT_READ_COMPLETE,
    ReadCompletePayload(
        device_id=self._config.device_id,
        values=values,
        duration_ms=duration_ms,
    ),
)
```

### 3.4 Worker 處理循環

Worker 是一個持續運行的 asyncio Task，從佇列中取出事件並依序執行 handler：

```python
async def _worker(self) -> None:
    while self._running:
        try:
            event, payload = await asyncio.wait_for(
                self._queue.get(), timeout=1.0
            )
            await self._process_event(event, payload)
            self._queue.task_done()
        except asyncio.TimeoutError:
            continue  # 1 秒沒事件，繼續等待
        except asyncio.CancelledError:
            break
```

`_process_event()` 順序執行所有 handler，**不使用 `asyncio.gather()`**：

```python
async def _process_event(self, event: str, payload: Any) -> None:
    handlers = self._handlers.get(event, [])
    for handler in handlers:
        try:
            await handler(payload)
        except Exception:
            logger.opt(exception=True).warning(
                "事件處理失敗: event={}, payload={}", event, repr(payload)
            )
```

順序執行而非並行的原因是**避免資源競爭**——多個 handler 可能操作同一個 Redis client 或共享狀態，順序執行更安全。

### 3.5 訂閱與取消訂閱

```python
# 訂閱
async def on_value_change(payload: ValueChangePayload):
    print(f"[{payload.device_id}] {payload.point_name}: "
          f"{payload.old_value} -> {payload.new_value}")

cancel = device.on("value_change", on_value_change)

# 取消訂閱
cancel()  # 呼叫返回的 cancel 函數即可
```

`on()` 方法返回一個 cancel 函數，而不是要求呼叫者記住 handler 引用。這個設計讓取消訂閱變得簡單，也避免了 lambda handler 無法取消的問題：

```python
def on(self, event: str, handler: AsyncHandler) -> Callable[[], None]:
    if event not in self._handlers:
        self._handlers[event] = []
    self._handlers[event].append(handler)

    def cancel() -> None:
        if event in self._handlers and handler in self._handlers[event]:
            self._handlers[event].remove(handler)

    return cancel
```

### 3.6 優化：無監聽器不入隊

```python
def emit(self, event: str, payload: Any = None) -> None:
    if not self._handlers.get(event):
        return  # 沒人監聽，直接跳過
    # ...
```

這個小優化很重要——在沒有訂閱者的情況下，`value_change` 事件可能每秒產生數十個。如果全部放入佇列再由 worker 丟棄，是不必要的開銷。

---

## 4. 事件訂閱模式：DeviceEventSubscriber

csp_lib 的 Manager 層使用統一的基底類別 `DeviceEventSubscriber` 來管理事件訂閱：

```python
class DeviceEventSubscriber:
    def __init__(self) -> None:
        self._unsubscribes: dict[str, list[Callable[[], None]]] = {}

    def subscribe(self, device: AsyncModbusDevice) -> None:
        device_id = device.device_id
        if device_id in self._unsubscribes:
            return  # 防止重複訂閱
        self._unsubscribes[device_id] = self._register_events(device)

    def unsubscribe(self, device: AsyncModbusDevice) -> None:
        device_id = device.device_id
        if device_id not in self._unsubscribes:
            return
        for unsub in self._unsubscribes.pop(device_id):
            unsub()  # 呼叫每個 cancel 函數
        self._on_unsubscribe(device_id)

    def _register_events(self, device: AsyncModbusDevice) -> list[Callable[[], None]]:
        raise NotImplementedError  # 子類別實作
```

子類別只需覆寫 `_register_events()` 即可定義要訂閱的事件：

```python
class DataUploadManager(DeviceEventSubscriber):
    def _register_events(self, device: AsyncModbusDevice) -> list[Callable[[], None]]:
        return [
            device.on(EVENT_READ_COMPLETE, self._on_read_complete),
            device.on(EVENT_DISCONNECTED, self._on_disconnected),
        ]

class StateSyncManager(DeviceEventSubscriber):
    def _register_events(self, device: AsyncModbusDevice) -> list[Callable[[], None]]:
        return [
            device.on(EVENT_READ_COMPLETE, self._on_read_complete),
            device.on(EVENT_CONNECTED, self._on_connected),
            device.on(EVENT_DISCONNECTED, self._on_disconnected),
            device.on(EVENT_ALARM_TRIGGERED, self._on_alarm_triggered),
            device.on(EVENT_ALARM_CLEARED, self._on_alarm_cleared),
        ]
```

這個模式的優點：

1. **自動管理取消訂閱**：`unsubscribe()` 自動呼叫所有 cancel 函數，不會遺漏。
2. **防止重複訂閱**：同一個 device_id 只能訂閱一次。
3. **清理鉤子**：子類別可覆寫 `_on_unsubscribe()` 做額外清理。

---

## 5. EventBridge：跨設備事件聚合

單一設備的事件很有用，但有時候你需要的是**跨設備的聚合事件**——例如「所有 PCS 都上線」或「任一設備觸發保護告警」。

csp_lib 的 `EventBridge`（Equipment 層）提供了這個能力：

```python
from csp_lib.equipment.device.event_bridge import (
    EventBridge, AggregateCondition
)
from csp_lib.equipment.device.events import EVENT_CONNECTED

# 定義聚合條件：所有設備都連線時觸發 "system_ready"
bridge = EventBridge([
    AggregateCondition(
        source_event=EVENT_CONNECTED,
        target_event="system_ready",
        predicate=lambda payloads: len(payloads) >= 3,  # 至少 3 台設備
        debounce_seconds=2.0,  # 防抖動：2 秒內穩定才觸發
    ),
])

# 附加到設備
bridge.attach(devices)

# 監聽聚合事件
async def on_system_ready(payloads):
    device_ids = list(payloads.keys())
    print(f"系統就緒！所有設備已連線: {device_ids}")

bridge.on("system_ready", on_system_ready)
```

### 5.1 Edge Detection

`EventBridge` 實作了 edge detection——只有當 predicate 結果從 `False` 變為 `True` 時才觸發事件，避免重複通知：

```python
async def _debounce_check(self, cond: AggregateCondition, key: str) -> None:
    try:
        await asyncio.sleep(cond.debounce_seconds)
    except asyncio.CancelledError:
        return

    payloads = self._latest.get(cond.source_event, {})
    current_result = cond.predicate(payloads)
    last_result = self._last_result.get(key, False)

    # Edge detection：僅在 False → True 時觸發
    if current_result and not last_result:
        self._last_result[key] = True
        await self._emit(cond.target_event, payloads)
    elif not current_result and last_result:
        self._last_result[key] = False
```

### 5.2 Debounce 機制

在設備啟動時，多台設備可能在數秒內陸續上線。如果每台上線都檢查一次 predicate，可能在只有部分設備上線時就觸發「系統就緒」。Debounce 機制確保等待一段時間穩定後再判斷：

```python
def _make_handler(self, cond: AggregateCondition, device_id: str) -> AsyncHandler:
    async def handler(payload: Any) -> None:
        # 記錄最新狀態
        self._latest.setdefault(cond.source_event, {})[device_id] = payload

        # 取消既有的 debounce task
        key = f"{cond.source_event}:{cond.target_event}"
        existing = self._debounce_tasks.get(key)
        if existing is not None and not existing.done():
            existing.cancel()

        # 建立新的 debounce task（重新計時）
        self._debounce_tasks[key] = asyncio.create_task(
            self._debounce_check(cond, key)
        )

    return handler
```

---

## 6. 從設備事件到 WebSocket

csp_lib 的 GUI 層使用 FastAPI WebSocket 將設備事件推送到瀏覽器。這個管線由三個元件組成：

### 6.1 WebSocketManager

管理所有 WebSocket 連線，提供廣播功能：

```python
class WebSocketManager:
    def __init__(self) -> None:
        self._connections: set[WebSocket] = set()
        self._lock = asyncio.Lock()

    async def connect(self, ws: WebSocket) -> None:
        await ws.accept()
        async with self._lock:
            self._connections.add(ws)

    async def disconnect(self, ws: WebSocket) -> None:
        async with self._lock:
            self._connections.discard(ws)

    async def broadcast(self, message: dict[str, Any]) -> None:
        if not self._connections:
            return
        payload = json.dumps(message, default=str)
        dead: list[WebSocket] = []
        async with self._lock:
            for ws in self._connections:
                try:
                    await ws.send_text(payload)
                except Exception:
                    dead.append(ws)
            for ws in dead:
                self._connections.discard(ws)
```

注意 `broadcast()` 的錯誤處理：如果某個連線發送失敗，它會被標記為 dead 並移除，而不是讓整個廣播失敗。這個「fire and forget」模式對即時推送系統是必要的。

### 6.2 EventBridge（GUI 層）

GUI 層的 `EventBridge` 與 Equipment 層的同名但不同——它負責將設備事件轉換為 WebSocket JSON 訊息：

```python
class EventBridge:
    def __init__(
        self,
        system_controller: SystemController,
        ws_manager: WebSocketManager,
        snapshot_interval: float = 5.0,
    ) -> None:
        self._sc = system_controller
        self._ws = ws_manager
        self._snapshot_interval = snapshot_interval
        self._cancel_fns: list[Callable[[], None]] = []

    async def attach(self) -> None:
        for device in self._sc.registry.all_devices:
            self._attach_device(device)
        self._snapshot_task = asyncio.create_task(self._snapshot_loop())
```

每種設備事件被轉換為標準化的 JSON 格式：

```python
def _attach_device(self, device: Any) -> None:
    async def on_value_change(payload: ValueChangePayload) -> None:
        await self._ws.broadcast({
            "type": "value_change",
            "device_id": payload.device_id,
            "data": {
                "point_name": payload.point_name,
                "old_value": payload.old_value,
                "new_value": payload.new_value,
            },
            "timestamp": payload.timestamp.isoformat(),
        })

    async def on_alarm_triggered(payload: DeviceAlarmPayload) -> None:
        await self._ws.broadcast({
            "type": "alarm_triggered",
            "device_id": payload.device_id,
            "data": {
                "code": payload.alarm_event.alarm.code,
                "name": payload.alarm_event.alarm.name,
                "level": payload.alarm_event.alarm.level.name,
            },
            "timestamp": payload.timestamp.isoformat(),
        })

    # 註冊 6 種事件
    self._cancel_fns.append(device.on(EVENT_VALUE_CHANGE, on_value_change))
    self._cancel_fns.append(device.on(EVENT_ALARM_TRIGGERED, on_alarm_triggered))
    self._cancel_fns.append(device.on(EVENT_ALARM_CLEARED, on_alarm_cleared))
    self._cancel_fns.append(device.on(EVENT_CONNECTED, on_connected))
    self._cancel_fns.append(device.on(EVENT_DISCONNECTED, on_disconnected))
    self._cancel_fns.append(device.on(EVENT_READ_COMPLETE, on_read_complete))
```

### 6.3 Snapshot 機制

WebSocket 的一個經典問題：**新連線的客戶端如何取得初始狀態？**

EventBridge 透過定期快照解決：

```python
async def _snapshot_loop(self) -> None:
    while True:
        await asyncio.sleep(self._snapshot_interval)  # 預設 5 秒
        if self._ws.connection_count == 0:
            continue  # 沒有連線就跳過

        snapshot = self._build_snapshot()
        await self._ws.broadcast(snapshot)

def _build_snapshot(self) -> dict[str, Any]:
    devices = []
    for dev in self._sc.registry.all_devices:
        devices.append({
            "device_id": dev.device_id,
            "is_connected": dev.is_connected,
            "is_responsive": dev.is_responsive,
            "is_protected": dev.is_protected,
            "latest_values": dev.latest_values,
            "active_alarm_count": len(dev.active_alarms),
        })

    mm = self._sc.mode_manager
    effective = mm.effective_mode

    return {
        "type": "snapshot",
        "data": {
            "devices": devices,
            "mode": {
                "base_mode_names": mm.base_mode_names,
                "active_override_names": mm.active_override_names,
                "effective_mode": effective.name if effective else None,
            },
            "auto_stop_active": self._sc.auto_stop_active,
            "is_running": self._sc.is_running,
        },
        "timestamp": datetime.now(timezone.utc).isoformat(),
    }
```

這樣，新加入的客戶端最多等待 5 秒就能收到完整系統狀態，之後透過增量事件保持同步。

### 6.4 WebSocket 端點

```python
@router.websocket("/ws")
async def websocket_endpoint(ws: WebSocket) -> None:
    manager = get_ws_manager(ws)
    await manager.connect(ws)
    try:
        while True:
            await ws.receive_text()  # Keep alive
    except WebSocketDisconnect:
        pass
    finally:
        await manager.disconnect(ws)
```

---

## 7. Pub/Sub 模式：跨程序事件分發

除了 WebSocket（程序內 → 瀏覽器），csp_lib 也透過 Redis Pub/Sub 實現**跨程序事件分發**。

### 7.1 事件分發全景

```
┌──────────────┐    Redis Pub/Sub    ┌──────────────┐
│  控制節點 A   │ ──────────────────> │  控制節點 B   │
│  (Leader)    │                     │  (Follower)  │
│              │                     │              │
│ StateSyncMgr │ → channel:device:*  │              │
│ ClusterPub   │ → channel:cluster:* │ ClusterSub   │
│              │                     │              │
│ CmdAdapter   │ ← channel:commands  │              │
└──────────────┘                     └──────────────┘
        │                                    │
        │ WebSocket                          │ WebSocket
        ▼                                    ▼
  ┌──────────┐                         ┌──────────┐
  │ 瀏覽器 A  │                         │ 瀏覽器 B  │
  └──────────┘                         └──────────┘
```

### 7.2 Channel 命名規範

csp_lib 使用一致的 channel 命名規範：

```
channel:device:{device_id}:data      設備資料更新
channel:device:{device_id}:status    設備連線狀態
channel:device:{device_id}:alarm     設備告警事件
channel:cluster:{ns}:leader_change   Leader 變更
channel:commands:write               指令接收
channel:commands:result              指令結果
```

這個命名規範讓你可以用 `PSUBSCRIBE` 做模式訂閱：

```python
# 訂閱所有設備的資料更新
pubsub = redis_client.pubsub()
await pubsub.psubscribe("channel:device:*:data")

# 訂閱所有設備的告警
await pubsub.psubscribe("channel:device:*:alarm")
```

### 7.3 訊息格式標準化

所有 Pub/Sub 訊息都是 JSON，包含 `timestamp` 欄位：

```json
// 資料更新
{
    "timestamp": "2026-03-10T08:00:00.123Z",
    "values": {"active_power": 50.2, "soc": 85.3}
}

// 連線狀態
{
    "online": true,
    "timestamp": "2026-03-10T08:00:00.123Z"
}

// 告警事件
{
    "type": "triggered",
    "alarm": {
        "code": "OVER_VOLTAGE",
        "name": "過電壓",
        "level": "WARNING",
        "description": "DC 電壓超過上限"
    },
    "timestamp": "2026-03-10T08:00:00.123Z"
}
```

---

## 8. 背壓與流控

### 8.1 佇列滿了怎麼辦？

`DeviceEventEmitter` 的佇列有上限（預設 10,000 個事件）。當佇列滿了，新事件會被丟棄：

```python
def emit(self, event: str, payload: Any = None) -> None:
    try:
        self._queue.put_nowait((event, payload))
    except asyncio.QueueFull:
        logger.warning("事件佇列已滿，丟棄事件: event=%s", event)
```

這是一個**有意識的設計決策**：在工業系統中，丟棄一個 `value_change` 事件遠比阻塞讀取循環安全。下一次讀取會產生新的事件，補上丟失的資料。

### 8.2 WebSocket 慢消費者

如果某個瀏覽器客戶端處理速度跟不上（例如網路慢），`broadcast()` 的 `send_text()` 可能阻塞。csp_lib 的處理方式是：

1. 如果 `send_text()` 拋出例外，標記該連線為 dead。
2. 廣播完成後清理 dead 連線。
3. 不會因為一個慢客戶端影響其他客戶端。

```python
async def broadcast(self, message: dict[str, Any]) -> None:
    payload = json.dumps(message, default=str)
    dead: list[WebSocket] = []
    async with self._lock:
        for ws in self._connections:
            try:
                await ws.send_text(payload)
            except Exception:
                dead.append(ws)  # 標記為 dead
        for ws in dead:
            self._connections.discard(ws)  # 清理
```

### 8.3 Redis Pub/Sub 的特性

Redis Pub/Sub 是「fire and forget」的——如果沒有訂閱者在線，訊息會直接丟棄。這意味著：

- **不適合做可靠事件傳遞**。如果你需要確保每個事件都被處理，應該使用 Redis Streams 或 Message Queue。
- **適合做即時通知**。csp_lib 用 Pub/Sub 做的是「通知」，真正的狀態資料存在 Redis Hash 中。

這也是為什麼 `StateSyncManager` 同時做了雙層通知（參見 Article 11）——Hash 保證狀態可查，Pub/Sub 提供即時通知。

---

## 9. 生命週期管理

### 9.1 Emitter 的啟動與停止

`DeviceEventEmitter` 需要明確啟動和停止：

```python
# 啟動 worker
await emitter.start()

# 停止 worker，處理剩餘事件後關閉
await emitter.stop()
```

`stop()` 會先取消 worker task，然後同步處理佇列中剩餘的事件：

```python
async def stop(self) -> None:
    if not self._running:
        return
    self._running = False

    if self._worker_task:
        self._worker_task.cancel()
        try:
            await self._worker_task
        except asyncio.CancelledError:
            pass

    # 處理剩餘事件——確保不會遺漏
    while not self._queue.empty():
        try:
            event, payload = self._queue.get_nowait()
            await self._process_event(event, payload)
        except asyncio.QueueEmpty:
            break
```

### 9.2 AsyncModbusDevice 的事件生命週期

```python
# connect() 時啟動 emitter
async def connect(self) -> None:
    await self._client.connect()
    await self._emitter.start()  # 啟動事件 worker
    self._client_connected = True
    await self._emitter.emit_await(EVENT_CONNECTED, ...)

# disconnect() 時停止 emitter
async def disconnect(self) -> None:
    await self.stop()  # 停止讀取循環
    await self._client.disconnect()
    await self._emitter.emit_await(EVENT_DISCONNECTED, ...)
    await self._emitter.stop()  # 停止事件 worker
```

注意順序：斷線事件在 `emitter.stop()` 之前發送，確保所有訂閱者都能收到。

---

## 10. 重點回顧

| 面向 | 設計決策 | 原因 |
|------|---------|------|
| Queue-based 發射 | `asyncio.Queue` + Worker | 不阻塞讀取循環 |
| emit vs emit_await | 高頻非阻塞 / 關鍵等待 | 平衡效能與正確性 |
| Frozen Payload | `@dataclass(frozen=True)` | 多 handler 共享安全 |
| Cancel 函數 | `on()` 返回 `Callable` | 簡化取消訂閱 |
| 無監聽跳過 | emit 前檢查 handlers | 避免無謂的佇列操作 |
| Snapshot 機制 | 定期廣播完整狀態 | 解決新連線初始狀態問題 |
| Dead 連線清理 | broadcast 中自動移除 | 慢客戶端不影響其他人 |
| 雙層通知 | Redis Hash + Pub/Sub | 狀態可查 + 即時通知 |

---

## 下篇預告

事件系統把資料從設備推送到各個角落，但最終使用者看到的是**圖表**和**數字**。下一篇我們將探討 csp_lib 的 REST API 設計、統計引擎的能耗計算，以及如何建構一個工業級的資料視覺化後端——讓數以千計的數據點化為有意義的資訊。

> **下一篇：** [Article 14 — 資料視覺化與 API 設計：讓工業數據說話](14-data-visualization.md)
