# Redis 在工業場域的應用：從快取到即時資料匯流排

> **從零打造工業控制協定框架：給軟體工程師的 CSP Library 實戰系列**
> Part 3 — 資料管線篇 | Article 11

---

## 前言

提到 Redis，多數後端工程師第一個聯想到的是「Web 應用快取」：Session 存放、API 回應快取、排行榜 Sorted Set。但在工業控制系統（ICS）領域，Redis 扮演的角色遠不止於此。它是一條串接設備狀態、叢集協調、指令分發的**即時資料匯流排（Real-time Data Bus）**。

在本篇中，我們將深入 csp_lib 的 Redis 整合層，看看一個為工業場域設計的 Python 框架如何善用 Redis 的資料結構，在毫秒級延遲下完成設備狀態同步、跨節點指令分發、以及即時告警通知。

---

## 1. 為什麼工業系統需要 Redis？

在典型的能源管理系統（EMS）中，你會面對這些挑戰：

1. **即時性要求**：PCS（儲能變流器）每秒回報功率數據，操作員需要在儀表板上看到秒級更新。
2. **跨程序狀態共享**：Leader 節點計算控制策略、Follower 節點執行寫入，兩者需要共享設備狀態。
3. **指令分發**：遠端操作員透過 Web UI 下達指令，需要即時傳遞到控制程序。
4. **叢集協調**：多個控制節點之間需要同步模式狀態、保護規則、最後一次命令。

傳統做法是用 Message Queue（RabbitMQ、Kafka）或資料庫輪詢。但 Redis 以其低延遲（亞毫秒級）、豐富的資料結構、以及同時支援 Pub/Sub 和持久化的特性，成為工業場域的理想選擇。

### 工業場域 vs Web 快取的差異

| 面向 | Web 快取 | 工業資料匯流排 |
|------|----------|---------------|
| 資料生命週期 | 分鐘～小時 | 秒～十秒（TTL 驅動） |
| 一致性要求 | Eventually Consistent | 秒級一致（設備狀態） |
| 訂閱者數量 | 百～千 | 十～百（但延遲要求嚴格） |
| 資料結構 | 主要 String | Hash + Set + Pub/Sub 混合 |
| 失效策略 | 被動失效 | 主動 TTL + 心跳機制 |

---

## 2. Redis 資料結構在工業場域的對應

### 2.1 Hash：設備狀態快照

Redis Hash 天生適合表達「一台設備的所有點位值」。相較於把整個 JSON 塞進一個 String，Hash 允許你**原子性地更新單一欄位**，也能一次取回整台設備的完整狀態。

在 csp_lib 中，設備狀態的 Redis Key 結構如下：

```
device:{device_id}:state    → Hash（所有點位最新值）
device:{device_id}:online   → String（"1" 或 "0"）
device:{device_id}:alarms   → Set（活躍告警代碼）
```

這個設計的精妙之處在於——每種資料用最適合的結構：

- **Hash** 存放點位值，因為點位數量多（可能 50+ 個），且需要部分更新。
- **String** 存放連線狀態，因為只有一個布林值，且需要帶 TTL 做心跳。
- **Set** 存放活躍告警代碼，因為告警是無序的、不重複的，且需要快速新增/移除。

### 2.2 Pub/Sub：即時事件通知

Redis Pub/Sub 提供了輕量的發布/訂閱機制。在工業場域，它的角色是**事件通知層**——告訴訂閱者「有東西變了」，而不是傳遞完整資料。

csp_lib 定義了三類 channel：

```
channel:device:{device_id}:data    → 資料更新通知
channel:device:{device_id}:status  → 連線狀態變化
channel:device:{device_id}:alarm   → 告警事件
```

以及叢集級別的 channel：

```
channel:cluster:{namespace}:leader_change  → Leader 變更通知
channel:commands:write                      → 指令接收
channel:commands:result                     → 指令執行結果
```

### 2.3 String + TTL：心跳機制

在工業系統中，「設備是否在線」是一個至關重要的資訊。csp_lib 使用 String + TTL 組合實現心跳：

- 每次讀取完成，設定 `device:{device_id}:online` 為 `"1"`，並設定 TTL（例如 60 秒）。
- 如果設備停止回報，TTL 過期後 Key 自動消失，訂閱者可以判斷設備離線。

這比「設備斷線時主動通知」更可靠——因為程序可能崩潰而來不及發送斷線通知，但 TTL 機制不會受此影響。

### 2.4 Set：告警狀態追蹤

告警是無序的集合：一台設備可能同時觸發多個告警，也可能清除其中一個。Redis Set 的 `SADD` / `SREM` 操作完美對應「觸發告警」和「清除告警」的語意。

---

## 3. csp_lib 的 Redis 整合

### 3.1 連線配置

csp_lib 提供了 `RedisConfig` 和 `RedisClient` 兩個核心類別，封裝了連線管理的複雜性：

```python
from csp_lib.redis import RedisClient, RedisConfig, TLSConfig

# 最簡單的 Standalone 模式
config = RedisConfig(host="localhost", port=6379)

# 帶密碼與 TLS 的生產配置
config = RedisConfig(
    host="redis.example.com",
    port=6380,
    password="your_redis_password",
    tls_config=TLSConfig(ca_certs="/path/to/ca.crt"),
    socket_timeout=5.0,
    socket_connect_timeout=3.0,
    retry_on_timeout=True,
)

# 從 Config 建立客戶端（推薦方式）
client = RedisClient.from_config(config)
await client.connect()
```

`RedisConfig` 是一個 frozen dataclass，保證配置建立後不可變——這在工業系統中很重要，因為你不希望運行中的配置被意外修改：

```python
@dataclass(frozen=True, slots=True)
class RedisConfig:
    host: str = "localhost"
    port: int = 6379
    password: str | None = None
    sentinels: tuple[tuple[str, int], ...] | None = None
    sentinel_master: str | None = None
    sentinel_password: str | None = None
    tls_config: TLSConfig | None = None
    socket_timeout: float | None = None
    socket_connect_timeout: float | None = None
    retry_on_timeout: bool = False
```

### 3.2 Sentinel 高可用模式

在生產環境中，單點 Redis 是不可接受的。csp_lib 原生支援 Redis Sentinel 模式：

```python
config = RedisConfig(
    sentinels=(
        ("sentinel1", 26379),
        ("sentinel2", 26379),
        ("sentinel3", 26379),
    ),
    sentinel_master="mymaster",
    password="redis_password",
    sentinel_password="sentinel_password",
)

client = RedisClient.from_config(config)
await client.connect()  # 自動透過 Sentinel 取得 Master
```

`RedisClient` 的 `from_config()` 工廠方法會根據配置自動選擇 Standalone 或 Sentinel 模式。這個設計讓上層程式碼完全不需要關心連線模式的差異：

```python
@classmethod
def from_config(cls, config: "RedisConfig") -> "RedisClient":
    instance = cls.__new__(cls)
    instance._config = config
    instance._host = config.host
    instance._port = config.port
    # ... 複製所有配置
    return instance

async def connect(self) -> None:
    if self.is_sentinel_mode:
        await self._connect_sentinel()
    else:
        await self._connect_standalone()
```

### 3.3 TLS 支援

工業場域的網路環境往往比辦公室更複雜——設備可能跨越不同的網段，甚至透過 VPN 連接。TLS 加密是基本要求：

```python
# 單向 TLS（僅驗證伺服器）
tls = TLSConfig(ca_certs="/path/to/ca.crt")

# 雙向 TLS（mTLS）—— 工業場域常見
tls = TLSConfig(
    ca_certs="/path/to/ca.crt",
    certfile="/path/to/client.crt",
    keyfile="/path/to/client.key",
)

config = RedisConfig(
    host="redis.example.com",
    port=6380,
    tls_config=tls,
)
```

`TLSConfig` 提供了 `to_ssl_context()` 方法，將配置轉為 Python 標準的 `ssl.SSLContext`。這個封裝隱藏了 SSL 配置的繁瑣細節：

```python
def to_ssl_context(self) -> ssl.SSLContext:
    cert_reqs_map = {
        "required": ssl.CERT_REQUIRED,
        "optional": ssl.CERT_OPTIONAL,
        "none": ssl.CERT_NONE,
    }
    context = ssl.create_default_context(cafile=self.ca_certs)
    context.check_hostname = self.cert_reqs == "required"
    context.verify_mode = cert_reqs_map[self.cert_reqs]
    if self.certfile and self.keyfile:
        context.load_cert_chain(certfile=self.certfile, keyfile=self.keyfile)
    return context
```

---

## 4. 狀態同步：StateSyncManager

`StateSyncManager` 是 csp_lib 中 Redis 最核心的應用——它將設備事件自動同步到 Redis，實現跨程序的狀態共享。

### 4.1 訂閱設備事件

```python
from csp_lib.redis import RedisClient, RedisConfig
from csp_lib.manager.state import StateSyncManager, StateSyncConfig

# 建立 Redis 客戶端
redis_client = RedisClient.from_config(
    RedisConfig(host="localhost", port=6379)
)
await redis_client.connect()

# 建立狀態同步管理器
config = StateSyncConfig(state_ttl=60, online_ttl=60)
state_manager = StateSyncManager(redis_client, config=config)

# 訂閱設備——之後所有事件自動同步至 Redis
state_manager.subscribe(pcs_device)
state_manager.subscribe(meter_device)
```

### 4.2 事件到 Redis 的映射

`StateSyncManager` 訂閱了五種設備事件，每種事件對應不同的 Redis 操作：

**讀取完成 (read_complete)**：更新 Hash + 刷新 online 心跳 + 發布到 data channel

```python
async def _on_read_complete(self, payload: ReadCompletePayload) -> None:
    device_id = payload.device_id
    state_key = self._state_key(device_id)    # device:{id}:state
    online_key = self._online_key(device_id)  # device:{id}:online

    # 更新設備狀態 Hash + 設定 TTL
    await self._redis.hset(state_key, payload.values)
    await self._redis.expire(state_key, self._state_ttl)

    # 同時刷新 online 心跳
    await self._redis.set(online_key, "1", ex=self._online_ttl)

    # 發布到 Pub/Sub channel
    message = json.dumps({
        "timestamp": payload.timestamp.isoformat(),
        "values": payload.values,
    }, default=str)
    await self._redis.publish(self._data_channel(device_id), message)
```

**連線 / 斷線事件**：更新 online 狀態 + 發布到 status channel

```python
async def _on_connected(self, payload: ConnectedPayload) -> None:
    await self._redis.set(
        self._online_key(payload.device_id), "1",
        ex=self._online_ttl
    )
    message = json.dumps({"online": True, "timestamp": payload.timestamp.isoformat()})
    await self._redis.publish(self._status_channel(payload.device_id), message)

async def _on_disconnected(self, payload: DisconnectPayload) -> None:
    await self._redis.set(self._online_key(payload.device_id), "0")
    message = json.dumps({
        "online": False,
        "reason": payload.reason,
        "timestamp": payload.timestamp.isoformat(),
    })
    await self._redis.publish(self._status_channel(payload.device_id), message)
```

**告警事件**：更新 alarms Set + 發布到 alarm channel

```python
async def _on_alarm_triggered(self, payload: DeviceAlarmPayload) -> None:
    alarm = payload.alarm_event.alarm
    await self._redis.sadd(self._alarms_key(payload.device_id), alarm.code)
    message = json.dumps({
        "type": "triggered",
        "alarm": {
            "code": alarm.code,
            "name": alarm.name,
            "level": alarm.level.value,
            "description": alarm.description,
        },
        "timestamp": payload.timestamp.isoformat(),
    })
    await self._redis.publish(self._alarm_channel(payload.device_id), message)
```

### 4.3 設計亮點：雙層通知

注意 `StateSyncManager` 同時做了兩件事：

1. **更新 Redis 資料結構**（Hash / String / Set）：這是「狀態層」，任何程序隨時可以查詢。
2. **發布到 Pub/Sub channel**：這是「事件層」，只有正在監聽的程序才會收到。

這個雙層設計解決了一個常見問題：**新加入的訂閱者如何取得初始狀態？**

- 訂閱者先用 `HGETALL` 讀取完整狀態（狀態層）。
- 然後 `SUBSCRIBE` channel 接收增量更新（事件層）。

---

## 5. Redis 作為叢集協調存儲

在多節點部署中，csp_lib 使用 Redis 作為 Leader/Follower 之間的狀態同步媒介。

### 5.1 ClusterStatePublisher（Leader 端）

Leader 節點定期將控制狀態寫入 Redis：

```python
class ClusterStatePublisher(AsyncLifecycleMixin):
    async def _publish_all(self) -> None:
        await self._publish_leader_identity()    # 寫入 leader 身份
        await self._publish_mode_state()         # 寫入模式狀態
        await self._publish_protection_state()   # 寫入保護狀態
        await self._publish_last_command()        # 寫入最後一次命令
        await self._publish_auto_stop()           # 寫入自動停機狀態
```

Redis Key 使用命名空間隔離，避免多叢集衝突：

```python
# ClusterConfig 提供命名空間化的 Key 生成
def redis_key(self, suffix: str) -> str:
    return f"cluster:{self.namespace}:{suffix}"

def redis_channel(self, suffix: str) -> str:
    return f"channel:cluster:{self.namespace}:{suffix}"
```

例如，namespace 為 `"production"` 時：
- `cluster:production:leader` → Leader 身份
- `cluster:production:mode_state` → 模式狀態
- `cluster:production:last_command` → 最後一次命令

### 5.2 ClusterStateSubscriber（Follower 端）

Follower 節點定期輪詢 Redis，將狀態反序列化為 `ClusterSnapshot`：

```python
class ClusterStateSubscriber(AsyncLifecycleMixin):
    async def _poll_all(self) -> None:
        snap = ClusterSnapshot()

        # 讀取 leader 身份
        leader_raw = await self._redis.get(self._config.redis_key("leader"))
        if leader_raw is not None:
            leader_data = json.loads(leader_raw)
            snap.leader_id = leader_data.get("instance_id")

        # 讀取模式狀態
        mode_data = await self._redis.hgetall(self._config.redis_key("mode_state"))
        if mode_data:
            snap.base_modes = json.loads(mode_data.get("base_modes", "[]"))
            snap.effective_mode = mode_data.get("effective_mode") or None

        # 讀取設備狀態（利用 StateSyncManager 發佈的 key）
        for device_id in self._config.device_ids:
            state = await self._redis.hgetall(f"device:{device_id}:state")
            if state:
                self._device_states[device_id] = state

        self._snapshot = snap
```

### 5.3 指令分發：RedisCommandAdapter

`RedisCommandAdapter` 監聽 Redis Pub/Sub channel，接收遠端指令並轉發到本地設備：

```python
class RedisCommandAdapter(AsyncLifecycleMixin):
    async def _listen_loop(self) -> None:
        pubsub = self._redis.pubsub()
        await pubsub.subscribe(self._command_channel)
        try:
            while self._running:
                message = await pubsub.get_message(
                    ignore_subscribe_messages=True, timeout=1.0
                )
                if message and message["type"] == "message":
                    await self._handle_message(message["data"])
        finally:
            await pubsub.unsubscribe(self._command_channel)
            await pubsub.aclose()
```

支援三種指令類型：

```python
# 點位寫入
# PUBLISH channel:commands:write '{"device_id":"pcs_1","point_name":"active_power_sp","value":100}'

# 動作執行
# PUBLISH channel:commands:write '{"device_id":"generator_1","action":"start"}'

# 系統指令
# PUBLISH channel:commands:write '{"system_command":"emergency_stop"}'
```

執行完成後，結果發布到 result channel：

```python
async def _publish_result(self, result: CommandResult) -> None:
    message = json.dumps(result.to_dict())
    await self._redis.publish(self._result_channel, message)
```

---

## 6. 配置模式總覽

### 6.1 開發環境

```python
# 最小配置：本地 Redis，無加密
redis_config = RedisConfig(host="localhost", port=6379)
state_config = StateSyncConfig(state_ttl=30, online_ttl=30)
```

### 6.2 生產環境（Sentinel + TLS）

```python
redis_config = RedisConfig(
    sentinels=(
        ("sentinel-1.internal", 26379),
        ("sentinel-2.internal", 26379),
        ("sentinel-3.internal", 26379),
    ),
    sentinel_master="ems-master",
    password="strong_password_from_vault",
    sentinel_password="sentinel_password",
    tls_config=TLSConfig(
        ca_certs="/etc/certs/ca.crt",
        certfile="/etc/certs/client.crt",
        keyfile="/etc/certs/client.key",
    ),
    socket_timeout=5.0,
    socket_connect_timeout=3.0,
    retry_on_timeout=True,
)

state_config = StateSyncConfig(state_ttl=60, online_ttl=60)
```

### 6.3 叢集配置

```python
from csp_lib.cluster import ClusterConfig, EtcdConfig

cluster_config = ClusterConfig(
    instance_id="node-1",
    etcd=EtcdConfig(endpoints=["etcd-1:2379", "etcd-2:2379"]),
    namespace="production",
    state_publish_interval=1.0,  # Leader 每秒發佈狀態
    state_ttl=30,                # 狀態 TTL 30 秒
    device_ids=["pcs_1", "pcs_2", "meter_1"],
)
```

---

## 7. 效能考量

### 7.1 操作頻率估算

假設系統有 10 台設備，每台每秒讀取一次：

| 操作 | 次數/秒 | Redis 命令 |
|------|---------|-----------|
| 狀態更新 | 10 | HSET + EXPIRE + SET + PUBLISH = 40 |
| 連線心跳 | 已包含在上面 | - |
| 告警事件 | ~0.1（偶發） | SADD + PUBLISH = ~0.2 |
| 叢集同步 | 1 | ~10 |
| **總計** | | **~50 cmd/s** |

50 cmd/s 對 Redis 來說微不足道（Redis 單執行緒可達 100K+ cmd/s）。

### 7.2 連線池

`RedisClient` 基於 `redis.asyncio.Redis`，內建連線池。所有 csp_lib 的 Manager 共用同一個 `RedisClient` 實例，避免連線浪費：

```python
# 建立一個客戶端，多處共用
redis_client = RedisClient.from_config(config)
await redis_client.connect()

state_sync = StateSyncManager(redis_client)
# RedisCommandAdapter 也使用相同的底層 Redis 連線
```

### 7.3 TTL 策略

csp_lib 的 TTL 策略遵循「寧可過期也不留髒資料」的原則：

- `state_ttl`（預設 60 秒）：設備狀態 Hash 的 TTL。如果設備停止回報超過 60 秒，狀態自動清除。
- `online_ttl`（預設 60 秒）：連線狀態的 TTL。每次讀取都會刷新，過期即代表設備離線。
- 叢集 `state_ttl`（預設 30 秒）：叢集狀態的 TTL。Leader 每秒發佈，30 秒未更新代表 Leader 故障。

---

## 8. Context Manager 支援

`RedisClient` 支援 async context manager，確保連線正確釋放：

```python
async with RedisClient.from_config(config) as client:
    await client.hset("device:pcs_1:state", {"active_power": 50.0})
    state = await client.hgetall("device:pcs_1:state")
    print(state)  # {"active_power": 50.0}
# 離開 with 區塊時自動斷線
```

這個模式特別適合腳本式的操作——例如部署時的健康檢查：

```python
async def check_redis_health(config: RedisConfig) -> bool:
    try:
        async with RedisClient.from_config(config) as client:
            return client.is_connected
    except Exception:
        return False
```

---

## 9. 重點回顧

| 面向 | 設計決策 | 原因 |
|------|---------|------|
| 資料結構選擇 | Hash 存狀態、Set 存告警、String+TTL 做心跳 | 每種資料用最適合的結構 |
| 雙層通知 | 資料結構 + Pub/Sub 並行 | 新訂閱者能取得初始狀態 |
| Sentinel 支援 | 設定層面透明切換 | 生產環境高可用需求 |
| TLS 配置 | 完整的單向/雙向 TLS | 工業網路安全要求 |
| Frozen Config | `@dataclass(frozen=True, slots=True)` | 運行時配置不可變 |
| Context Manager | `async with` 支援 | 確保連線正確釋放 |

---

## 下篇預告

Redis 解決了「即時」的問題，但工業數據還需要**持久化**——設備的歷史功率曲線、每日發電量統計、月度報表。下一篇我們將進入 MongoDB Time Series 的世界，看看 csp_lib 如何用 `MongoBatchUploader` 實現高效的批次寫入，以及 `DataUploadManager` 如何將設備事件無縫接入儲存層。

> **下一篇：** [Article 12 — 時序資料庫實戰：MongoDB Time Series 與工業數據持久化](12-timeseries-storage.md)
