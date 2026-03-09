# Redis Streams 作為資料平面：Leader-Follower 狀態同步

> **從零打造工業控制協定框架：給軟體工程師的 CSP Library 實戰系列**
> Part 5 — 高可用篇 | Article 21

上一篇我們解決了「誰是 Leader」的問題。但光有選舉還不夠。想像一下：Leader 掛了，Follower 成功當選為新 Leader——然後呢？

新 Leader 剛啟動，它對設備的最新狀態一無所知。它不知道現在 SOC 是多少、功率輸出多少、哪些保護規則已經觸發。如果它盲目地開始控制迴路，可能會下達錯誤的命令。

這就是**資料平面（Data Plane）**的問題：Follower 需要持續同步 Leader 的運行狀態，才能在接管時無縫銜接。

---

## 1. 資料平面的需求

讓我們定義「Follower 需要知道什麼」：

| 資料類別 | 具體內容 | 為什麼需要 |
|---------|---------|-----------|
| 設備狀態 | 每台設備的最新讀取值 | 接管後立即有 context 可用 |
| 模式狀態 | 當前運行模式、override 列表 | 不中斷模式設定 |
| 保護狀態 | 觸發的保護規則 | 了解系統安全狀態 |
| 命令狀態 | 最後一次命令的 P/Q 值 | 平滑接管，避免功率突變 |
| Leader 身份 | 誰是 leader、何時上任 | 診斷和監控 |

這些資料有幾個特性：

1. **高頻更新**：設備值每秒讀取一次，模式和保護狀態隨控制迴圈更新
2. **容忍遺失**：丟失一筆中間狀態沒關係，只要有最新值就好
3. **需要持久化**：Follower 重啟後需要能讀到最新狀態，不能靠記憶
4. **多消費者**：未來可能有多個 Follower 同時讀取

---

## 2. 為什麼用 Redis（而非 Pub/Sub 或 Message Queue）

### Redis Pub/Sub 的問題

Redis Pub/Sub 看起來很直覺——Leader publish，Follower subscribe。但它有一個致命問題：**fire-and-forget**。

```
Leader 發佈 SOC=80% ──→ Follower 收到 ✓
Leader 發佈 SOC=79% ──→ Follower 斷線，沒收到 ✗
Leader 發佈 SOC=78% ──→ Follower 重連，但 79% 已經丟了
```

Follower 重連後，它錯過了中間的訊息。對於我們的場景，這不是問題（我們只需要最新值），但 Pub/Sub 連最新值都不保留——Follower 重連後什麼都沒有，要等到下一次 publish 才會收到資料。

### Message Queue（Kafka、RabbitMQ）

太重了。我們不需要訊息佇列的完整語意（順序保證、exactly-once delivery、consumer group offset tracking）。我們需要的只是「最新狀態的共享記憶體」。

### Redis Hash + TTL：剛好的選擇

csp_lib 的做法是用 Redis Hash 存放最新狀態，搭配 TTL 自動過期：

```
Redis Key                                    Type    TTL
────────────────────────────────────────────────────────────
cluster:production:leader                    String  30s
cluster:production:mode_state                Hash    30s
cluster:production:protection_state          Hash    30s
cluster:production:last_command              Hash    30s
cluster:production:auto_stop_active          String  30s
device:pcs_01:state                          Hash    30s
device:bms_01:state                          Hash    30s
```

這個設計的優點：

1. **Follower 任何時候加入都能讀到最新狀態**：不需要回溯歷史
2. **TTL 自動清理**：Leader 掛了，狀態會在 30 秒後自動消失，Follower 不會讀到陳舊資料
3. **原子操作**：Redis HSET/HGETALL 是原子的，不會讀到半更新的狀態
4. **簡單**：不需要額外的 consumer group 管理或 offset tracking

同時，csp_lib 也使用 Redis Pub/Sub 做即時通知（例如 leader 變更事件），但它只是「提醒」，不承載實際資料。實際資料永遠從 Hash 中讀取。

---

## 3. ClusterStatePublisher：Leader 的狀態廣播器

`ClusterStatePublisher` 是 Leader 端的核心元件，負責定期把系統狀態寫入 Redis。

### 架構位置

```
┌─────────────────────────────────────────────────────┐
│                   Leader Node                        │
│                                                      │
│  Devices ──→ ReadLoop ──→ latest_values              │
│                              │                       │
│  ModeManager ────────────────┤                       │
│  ProtectionGuard ────────────┤                       │
│  StrategyExecutor ───────────┤                       │
│                              ▼                       │
│                   ClusterStatePublisher               │
│                              │                       │
│                              ▼                       │
│                        Redis (Hash)                  │
└─────────────────────────────────────────────────────┘
```

### 初始化

Publisher 在 `ClusterController._handle_elected()` 中被建立和啟動：

```python
async def _handle_elected(self) -> None:
    # ... 前置步驟 ...

    executor = self._system_controller.executor
    self._publisher = ClusterStatePublisher(
        config=self._config,
        redis_client=self._redis,
        mode_manager=self._system_controller.mode_manager,
        protection_guard=self._system_controller.protection_guard,
        get_last_command=lambda: (
            executor.last_command.p_target,
            executor.last_command.q_target,
        ),
        get_auto_stop=lambda: self._system_controller.auto_stop_active,
    )
    await self._publisher.start()
```

注意 `get_last_command` 和 `get_auto_stop` 是 callable，不是靜態值。每次發佈都會即時取得最新資料。

### 發佈迴圈

Publisher 以固定間隔發佈所有狀態：

```python
async def _publish_loop(self) -> None:
    interval = self._config.state_publish_interval  # 預設 1.0 秒
    while not self._stop_event.is_set():
        try:
            await self._publish_all()
        except asyncio.CancelledError:
            return
        except Exception:
            logger.exception("Failed to publish cluster state")

        try:
            await asyncio.wait_for(
                self._stop_event.wait(), timeout=interval
            )
            return
        except asyncio.TimeoutError:
            pass
```

`_publish_all` 依序發佈五種狀態：

```python
async def _publish_all(self) -> None:
    await self._publish_leader_identity()
    await self._publish_mode_state()
    await self._publish_protection_state()
    await self._publish_last_command()
    await self._publish_auto_stop()
```

### 各狀態的資料結構

**Leader 身份**（String + TTL）：

```python
async def _publish_leader_identity(self) -> None:
    key = self._config.redis_key("leader")  # "cluster:production:leader"
    data = {
        "instance_id": self._config.instance_id,
        "elected_at": self._elected_at,
        "hostname": socket.gethostname(),
    }
    await self._redis.set(key, json.dumps(data), ex=self._config.state_ttl)
```

**模式狀態**（Hash）：

```python
async def _publish_mode_state(self) -> None:
    mm = self._mode_manager
    key = self._config.redis_key("mode_state")
    data = {
        "base_modes": json.dumps(mm.base_mode_names),
        "overrides": json.dumps(mm.active_override_names),
        "effective_mode": mm.effective_mode.name if mm.effective_mode else "",
    }
    await self._redis.hset(key, data)
    await self._redis.expire(key, self._config.state_ttl)
```

**保護狀態**（Hash）：

```python
async def _publish_protection_state(self) -> None:
    key = self._config.redis_key("protection_state")
    result = self._protection_guard.last_result
    data = {
        "triggered_rules": json.dumps(result.triggered_rules if result else []),
        "was_modified": json.dumps(result.was_modified if result else False),
    }
    await self._redis.hset(key, data)
    await self._redis.expire(key, self._config.state_ttl)
```

**最後命令**（Hash）：

```python
async def _publish_last_command(self) -> None:
    key = self._config.redis_key("last_command")
    p, q = self._get_last_command()
    data = {
        "p_target": json.dumps(p),
        "q_target": json.dumps(q),
        "timestamp": json.dumps(time.time()),
    }
    await self._redis.hset(key, data)
    await self._redis.expire(key, self._config.state_ttl)
```

### Redis Key 命名空間

所有 key 都帶有 namespace 隔離，避免多套系統共用同一個 Redis 時衝突：

```python
def redis_key(self, suffix: str) -> str:
    return f"cluster:{self.namespace}:{suffix}"
    # 例如: "cluster:production:leader"
    #       "cluster:staging:mode_state"

def redis_channel(self, suffix: str) -> str:
    return f"channel:cluster:{self.namespace}:{suffix}"
```

---

## 4. ClusterStateSubscriber：Follower 的狀態接收器

Follower 端用 `ClusterStateSubscriber` 定期從 Redis 讀取 Leader 發佈的狀態。

### 資料模型：ClusterSnapshot

Subscriber 把所有狀態匯總到一個 `ClusterSnapshot` 物件中：

```python
@dataclass
class ClusterSnapshot:
    """叢集狀態快照"""

    leader_id: str | None = None
    elected_at: float | None = None
    base_modes: list[str] = field(default_factory=list)
    override_names: list[str] = field(default_factory=list)
    effective_mode: str | None = None
    triggered_rules: list[str] = field(default_factory=list)
    protection_was_modified: bool = False
    p_target: float = 0.0
    q_target: float = 0.0
    command_timestamp: float | None = None
    auto_stop_active: bool = False
```

每次輪詢完成後，整個 snapshot 會被原子替換（不是逐欄位更新），確保消費者不會讀到不一致的狀態。

### 輪詢迴圈

```python
class ClusterStateSubscriber(AsyncLifecycleMixin):
    def __init__(self, config: ClusterConfig, redis_client: RedisClient) -> None:
        self._config = config
        self._redis = redis_client
        self._snapshot = ClusterSnapshot()
        self._device_states: dict[str, dict[str, Any]] = {}
        self._task: asyncio.Task | None = None
        self._stop_event = asyncio.Event()

    @property
    def snapshot(self) -> ClusterSnapshot:
        """目前叢集狀態快照"""
        return self._snapshot

    @property
    def device_states(self) -> dict[str, dict[str, Any]]:
        """設備狀態快取（device_id → latest_values dict）"""
        return self._device_states

    async def _poll_loop(self) -> None:
        interval = self._config.state_publish_interval
        while not self._stop_event.is_set():
            try:
                await self._poll_all()
            except asyncio.CancelledError:
                return
            except Exception:
                logger.exception("Failed to poll cluster state")

            try:
                await asyncio.wait_for(
                    self._stop_event.wait(), timeout=interval
                )
                return
            except asyncio.TimeoutError:
                pass
```

### 設備狀態同步

除了叢集狀態，Subscriber 也讀取個別設備的狀態：

```python
# 在 _poll_all 的最後
for device_id in self._config.device_ids:
    try:
        state = await self._redis.hgetall(f"device:{device_id}:state")
        if state:
            self._device_states[device_id] = state
    except Exception:
        logger.debug(f"Failed to read device state for {device_id}")
```

`device_ids` 是在 `ClusterConfig` 中配置的：

```python
config = ClusterConfig(
    instance_id="node-2",
    device_ids=["pcs_01", "bms_01", "meter_01"],
    # ...
)
```

這裡的設備狀態是由 Leader 端的 `StateSyncManager`（在 `UnifiedDeviceManager` 中）寫入 Redis 的，key 格式為 `device:{device_id}:state`。

---

## 5. VirtualContextBuilder：Follower 的「虛擬設備」

這是 csp_lib 叢集架構中最精巧的設計。

### 問題

在正常的 Leader 模式下，`ContextBuilder` 從設備的 `latest_values` 讀取資料來建構 `StrategyContext`：

```
Leader:
  device.latest_values["soc"] ──→ ContextBuilder ──→ StrategyContext.soc
```

但 Follower 沒有連接設備，它沒有 `latest_values`。如果我們讓 Follower 也連接設備，就違反了「同一時間只有一個控制器連接設備」的原則。

### 解法：VirtualContextBuilder

`VirtualContextBuilder` 用 Redis 中同步的設備狀態替代直接的設備讀取：

```
Follower:
  Redis device_states["pcs_01"]["soc"] ──→ VirtualContextBuilder ──→ StrategyContext.soc
```

對下游的 `StrategyExecutor` 來說，`VirtualContextBuilder.build()` 和 `ContextBuilder.build()` 的介面完全相同，都返回 `StrategyContext`：

```python
class VirtualContextBuilder:
    def __init__(
        self,
        subscriber: DeviceStateProvider,
        mappings: list[ContextMapping],
        system_base: SystemBase | None = None,
        trait_device_map: dict[str, list[str]] | None = None,
    ) -> None:
        self._subscriber = subscriber
        self._mappings = mappings
        self._system_base = system_base
        self._trait_device_map = trait_device_map or {}

    def build(self) -> StrategyContext:
        """建構 StrategyContext（與 ContextBuilder.build 相同簽名）"""
        ctx = StrategyContext(
            last_command=Command(),
            system_base=self._system_base,
        )

        for mapping in self._mappings:
            value = self._resolve_value(mapping)
            _set_context_field(ctx, mapping.context_field, value)

        return ctx
```

### DeviceStateProvider Protocol

`VirtualContextBuilder` 不直接依賴 `ClusterStateSubscriber`，而是依賴一個 Protocol：

```python
@runtime_checkable
class DeviceStateProvider(Protocol):
    """提供設備狀態資料的 Protocol"""

    @property
    def device_states(self) -> dict[str, dict[str, Any]]: ...
```

`ClusterStateSubscriber` 自然滿足這個 Protocol（它有 `device_states` 屬性）。這個設計讓 `VirtualContextBuilder` 可以與任何實作 `device_states` 的物件搭配使用，增加了可測試性和可擴展性。

### 值解析

`VirtualContextBuilder` 支援兩種模式，與 `ContextBuilder` 完全對稱：

**單一設備模式**（device_id 指定）：

```python
def _read_single_device(self, mapping: ContextMapping) -> Any:
    device_id = mapping.device_id
    if device_id is None:
        return None

    device_state = self._subscriber.device_states.get(device_id)
    if not device_state:
        return None

    return device_state.get(mapping.point_name)
```

**Trait 聚合模式**（跨多設備聚合）：

```python
def _read_trait_aggregate(self, mapping: ContextMapping) -> Any:
    trait = mapping.trait
    if trait is None:
        return None

    device_ids = self._trait_device_map.get(trait, [])
    if not device_ids:
        return None

    values = []
    for device_id in device_ids:
        device_state = self._subscriber.device_states.get(device_id)
        if device_state:
            v = device_state.get(mapping.point_name)
            if v is not None:
                values.append(v)

    if not values:
        return None

    if mapping.custom_aggregate is not None:
        return mapping.custom_aggregate(values)

    return apply_builtin_aggregate(mapping.aggregate, values)
```

`trait_device_map` 是在 `ClusterController._on_start` 中從 `DeviceRegistry` 建構的：

```python
def _build_trait_device_map(self) -> dict[str, list[str]]:
    trait_map: dict[str, list[str]] = {}
    sc = self._system_controller
    for mapping in sc.config.context_mappings:
        if mapping.trait is not None and mapping.trait not in trait_map:
            devices = sc.registry.get_devices_by_trait(mapping.trait)
            trait_map[mapping.trait] = [d.device_id for d in devices]
    return trait_map
```

---

## 6. 完整的 Failover 流程

現在讓我們把所有元件串起來，看完整的 failover 流程：

### 正常運行期

```
┌──────────── Leader (Node A) ─────────────────┐
│                                               │
│  AsyncModbusDevice                            │
│       │ read_loop (每秒)                      │
│       ▼                                       │
│  latest_values ──→ ContextBuilder ──→ Context │
│       │                │                      │
│       │          StrategyExecutor              │
│       │                │                      │
│       │           ProtectionGuard              │
│       │                │                      │
│       │           CommandRouter                │
│       │                │                      │
│       │          write to devices              │
│       │                                       │
│       ▼                                       │
│  StateSyncManager ──→ Redis (device:*:state)  │
│  ClusterStatePublisher ──→ Redis (cluster:*)  │
└───────────────────────────────────────────────┘

                    Redis
                      │
                      ▼

┌──────────── Follower (Node B) ────────────────┐
│                                                │
│  ClusterStateSubscriber                        │
│       │ poll_loop (每秒)                       │
│       ▼                                        │
│  device_states + snapshot                      │
│       │                                        │
│  VirtualContextBuilder ──→ Context             │
│       │                                        │
│  StrategyExecutor (dry-run)                    │
│       │                                        │
│  _noop_command_handler (不下發命令)             │
│                                                │
│  ★ 隨時準備接管                                │
└────────────────────────────────────────────────┘
```

### Failover 發生

```
Timeline:
─────────────────────────────────────────────────────────────
t=0   Node A (Leader) 節點故障
t=0   keepalive 開始失敗
t=3.3 第 1 次 keepalive 失敗
t=6.6 第 2 次 keepalive 失敗
t=9.9 第 3 次 keepalive 失敗 → self-fence
      (或 t=10 lease 到期，etcd 自動刪除 key)

t=10  Node A 的 election key 消失

t=10-15 Node B 的 _wait_for_leader_loss 偵測到 key 消失
        Node B 進入 CANDIDATE 狀態
        Node B 執行 _try_campaign → 成功
        Node B 成為 LEADER
        觸發 on_elected callback

t=15  ClusterController._handle_elected() 開始執行

t=15  Step 1: 啟動 UnifiedDeviceManager
      → 連接 Modbus 設備
      → 啟動讀取迴圈
      → 啟動 StateSyncManager

t=17  Step 2: 等待 failover_grace_period (2 秒)
      → 設備產生新的讀取值

t=17  Step 3: 切換 executor 到 leader 模式
      → context_provider: VirtualContextBuilder → ContextBuilder
      → on_command: noop → live command handler
      → 控制迴路恢復！

t=17  Step 4: 啟動 ClusterStatePublisher
      → 開始向 Redis 發佈狀態

t=17  Step 5: 同步模式狀態
      → 從 subscriber 的 snapshot 讀取之前 leader 的模式設定
      → 同步到 live ModeManager

t=17  ★ Failover 完成
      ★ 總停機時間: ~17 秒
─────────────────────────────────────────────────────────────
```

### 模式狀態同步

Failover 過程中有一個容易忽略的細節：模式狀態同步。

當 Follower 升格為 Leader 時，它需要延續之前 Leader 的模式設定。如果之前的 Leader 正在執行 PQ 策略加上一個事件 override，新 Leader 應該繼續執行相同的模式組合：

```python
async def _sync_mode_state_from_snapshot(self) -> None:
    if self._subscriber is None:
        return

    snap = self._subscriber.snapshot
    mm = self._system_controller.mode_manager

    if snap.base_modes:
        for mode_name in snap.base_modes:
            if mode_name in mm.registered_modes and mode_name not in mm.base_mode_names:
                try:
                    await mm.add_base_mode(mode_name)
                except (KeyError, ValueError):
                    pass
```

---

## 7. Executor 的角色切換機制

`ClusterController` 最巧妙的設計在於它如何切換 `StrategyExecutor` 的行為，而不需要停止和重新建立 executor：

```python
def _enter_follower_mode(self) -> None:
    """切換 executor 到 follower 模式"""
    executor = self._system_controller.executor
    # 替換 context 來源：從設備讀取 → 從 Redis 讀取
    if self._virtual_builder is not None:
        executor.set_context_provider(self._virtual_builder.build)
    # 替換命令處理器：真實寫入 → no-op
    executor.set_on_command(self._noop_command_handler)

def _enter_leader_mode(self) -> None:
    """切換 executor 到 leader 模式"""
    executor = self._system_controller.executor
    # 還原 context 來源：從設備讀取
    executor.set_context_provider(self._live_context_provider)
    # 還原命令處理器：真實寫入
    executor.set_on_command(self._live_on_command)
```

這個設計的優點：

1. **Executor 始終在運行**：策略持續執行，只是資料來源和命令目的地不同
2. **切換是原子的**：兩個 setter 呼叫之間沒有中間狀態
3. **Follower 的策略是 warm 的**：策略一直在處理資料，接管時不需要冷啟動

### 為什麼 Follower 要執行策略？

你可能會問：Follower 既然不下發命令，為什麼要跑策略？

答案是**預熱**。有些策略有內部狀態（例如 PID 控制器的積分項、移動平均的窗口）。如果 Follower 在接管前完全空閒，接管後策略的內部狀態是空的，輸出可能不穩定。

透過持續用 Redis 同步的資料驅動策略 dry-run，Follower 的策略內部狀態會逐漸收斂到和 Leader 相似的狀態。

---

## 8. Redis Sentinel：Redis 本身的 HA

到目前為止，我們用 etcd 做了控制器的 HA，但 Redis 呢？如果 Redis 掛了，整個狀態同步就斷了。

csp_lib 的 `RedisClient` 原生支援 Redis Sentinel 模式：

```python
from csp_lib.redis import RedisClient, RedisConfig

# Sentinel 模式配置
config = RedisConfig(
    sentinels=[
        ("sentinel-1", 26379),
        ("sentinel-2", 26379),
        ("sentinel-3", 26379),
    ],
    sentinel_master="mymaster",
    password="redis_password",
    sentinel_password="sentinel_password",
)

client = RedisClient.from_config(config)
await client.connect()
```

Redis Sentinel 的作用：

1. **監控**：持續檢查 Redis master 和 replica 是否正常
2. **通知**：master 故障時通知應用程式
3. **自動 failover**：master 掛了，自動將一個 replica 提升為新 master
4. **配置提供者**：客戶端從 Sentinel 查詢當前 master 的位址

在 `RedisClient.from_config` 中，Sentinel 模式的連線流程：

```python
async def _connect_sentinel(self) -> None:
    # 建立 Sentinel 連線
    self._sentinel = Sentinel(
        list(self._config.sentinels),
        sentinel_kwargs=sentinel_kwargs,
    )

    # 從 Sentinel 取得目前 master 的位址
    self._client = self._sentinel.master_for(
        self._config.sentinel_master,
        **master_kwargs,
    )

    await self._client.ping()
```

`sentinel.master_for()` 返回的是一個特殊的 Redis client，它會在 master 切換時自動重新連線到新的 master。

### 部署架構建議

完整的 HA 部署架構：

```
┌───────────────────────────────────────────────────┐
│                etcd Cluster (3 nodes)              │
│   etcd-1 ──── etcd-2 ──── etcd-3                 │
│       │           │           │                   │
│       └───── Raft 共識 ──────┘                    │
└───────────────────────────────────────────────────┘
        │                           │
        ▼                           ▼
┌───────────────┐           ┌───────────────┐
│   Node A      │           │   Node B      │
│   (Leader)    │           │   (Follower)  │
│               │           │               │
│ ClusterCtrl   │           │ ClusterCtrl   │
│ SystemCtrl    │           │ SystemCtrl    │
│ Modbus ↔ PCS  │           │ (dry-run)     │
└───────┬───────┘           └───────┬───────┘
        │                           │
        ▼                           ▼
┌───────────────────────────────────────────────────┐
│            Redis Sentinel Cluster                  │
│                                                    │
│  Sentinel-1  Sentinel-2  Sentinel-3               │
│      │           │           │                    │
│      ▼           ▼           ▼                    │
│   Master ←── Replica-1  Replica-2                 │
└───────────────────────────────────────────────────┘
```

### 生產環境的容錯矩陣

| 故障場景 | 影響 | 恢復方式 |
|---------|------|---------|
| 1 個 etcd 節點故障 | 無影響（Raft 多數派存活） | 自動 |
| Redis master 故障 | 短暫中斷狀態同步 | Sentinel 自動 failover |
| Leader Node 故障 | 控制迴路中斷 ~15-20 秒 | etcd election failover |
| Follower Node 故障 | 無影響（可用性降低） | 重啟即可 |
| etcd 全部故障 | 無法選舉新 leader | 現有 leader 繼續運作 |
| Redis 全部故障 | 狀態同步中斷 | leader 控制迴路不受影響 |
| 網路分區 | leader self-fence | 分區恢復後重新選舉 |

注意最後三行：**etcd 或 Redis 全部故障不會立即停止控制迴路**。Leader 會繼續運作（只是失去了備援能力），直到它自己也故障才需要 failover。

---

## 9. 完整使用範例

讓我們把所有元件組合起來，看一個完整的生產部署範例：

```python
import asyncio
from csp_lib.cluster import ClusterController, ClusterConfig, EtcdConfig
from csp_lib.redis import RedisClient, RedisConfig
from csp_lib.integration import SystemController
from csp_lib.manager import UnifiedDeviceManager
from csp_lib.core import get_logger

logger = get_logger("app")


async def main():
    # 1. Redis 配置（Sentinel 模式）
    redis_config = RedisConfig(
        sentinels=[
            ("sentinel-1", 26379),
            ("sentinel-2", 26379),
            ("sentinel-3", 26379),
        ],
        sentinel_master="mymaster",
        password="redis_password",
    )
    redis = RedisClient.from_config(redis_config)
    await redis.connect()

    # 2. 建立 SystemController 和 UnifiedDeviceManager
    #    （假設已經配置好設備、策略、映射）
    sys_ctrl = SystemController(config=sys_config)
    unified_mgr = UnifiedDeviceManager(config=mgr_config)

    # 3. Cluster 配置
    cluster_config = ClusterConfig(
        instance_id="node-1",
        etcd=EtcdConfig(
            endpoints=["etcd-1:2379", "etcd-2:2379", "etcd-3:2379"],
        ),
        namespace="production",
        lease_ttl=10,
        state_publish_interval=1.0,
        state_ttl=30,
        failover_grace_period=2.0,
        max_keepalive_failures=3,
        device_ids=["pcs_01", "bms_01", "meter_01"],
    )

    # 4. 建立 ClusterController
    async def on_promoted():
        logger.info("Promoted to leader! Control loop is active.")

    async def on_demoted():
        logger.warning("Demoted to follower. Control loop suspended.")

    cluster = ClusterController(
        config=cluster_config,
        system_controller=sys_ctrl,
        unified_manager=unified_mgr,
        redis_client=redis,
        on_promoted=on_promoted,
        on_demoted=on_demoted,
    )

    # 5. 啟動並永久運行
    try:
        async with cluster:
            logger.info(f"Cluster node started: {cluster_config.instance_id}")
            logger.info(f"Initial role: {cluster.role}")

            # 永久運行，直到被中斷
            await asyncio.Event().wait()
    finally:
        await redis.disconnect()


if __name__ == "__main__":
    asyncio.run(main())
```

部署時只需要在不同的機器上啟動相同的程式，設定不同的 `instance_id`：

```bash
# 機器 A
INSTANCE_ID=node-1 python main.py

# 機器 B
INSTANCE_ID=node-2 python main.py
```

第一個啟動的會成為 Leader，後面的自動成為 Follower。Leader 掛了，Follower 自動接管。就這麼簡單。

---

## 10. 重點回顧

1. **資料平面的核心需求**：Follower 需要持續同步 Leader 的設備狀態和系統狀態
2. **Redis Hash + TTL 是最佳選擇**：簡單、持久、自動過期、原子操作
3. **ClusterStatePublisher / Subscriber**：Leader 定期發佈、Follower 定期輪詢
4. **VirtualContextBuilder**：用 Redis 資料替代設備讀取，對策略層透明
5. **Executor 角色切換**：替換 context_provider 和 on_command，不需重建 executor
6. **Follower 持續 dry-run**：預熱策略內部狀態，接管時更平滑
7. **Redis Sentinel**：Redis 本身的 HA，完成容錯的最後一哩路
8. **Failover 總時間 ~15-20 秒**：從 Leader 掛掉到新 Leader 完全接管

這三篇高可用系列文章涵蓋了工業控制系統 HA 的完整圖景：從「為什麼需要 HA」到「如何選舉 Leader」再到「如何同步狀態實現無縫 failover」。csp_lib 的叢集架構不是學術論文中的理論模型，而是可以直接部署在生產環境的實作方案。

---

> **下一篇預告**：Part 6 將進入監控與可觀測性的世界——當你的控制系統在半夜三點出了問題，你怎麼知道發生了什麼？
