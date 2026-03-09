# MongoDB Replica Set：工業資料的持久化保障

> **從零打造工業控制協定框架：給軟體工程師的 CSP Library 實戰系列**
> Part 5 — 高可用篇 | Article 22

---

## 前言：資料不只是資料，而是證據

在 Web 應用的世界裡，一筆訂單寫入失敗，最壞的情況是用戶重新操作一次。但在工業控制系統中，一筆遺失的告警記錄，可能讓你在事故調查時無法還原現場；一筆消失的指令記錄，可能讓你在合規稽核時無法自證清白。

工業資料的持久化不僅僅是「存下來」這麼簡單。它是法規遵循的基礎，是事故分析的線索，是責任歸屬的依據。這篇文章將帶你從資料持久化的「為什麼」出發，深入 MongoDB Replica Set 的架構原理，最終透過 csp_lib 的實際程式碼，展示如何在工業系統中建構一套可靠的資料持久化方案。

---

## 為什麼工業系統對資料持久性要求特別高？

### 法規遵循：有些資料必須留存數年

台灣的《電業法》要求再生能源發電業者保留發電數據至少五年。中國的《電力監控系統安全防護規定》要求操作日誌保留不少於六個月。歐盟的能源法規更是嚴格，某些數據的留存期限長達十年。

這意味著你的儲存方案不能只考慮「當下能不能寫進去」，還要考慮「三年後能不能讀出來」、「資料在這段期間有沒有被竄改」。單一磁碟故障就導致資料永久遺失，這是不可接受的。

### 稽核軌跡：誰在什麼時候送了什麼指令？

想像這個場景：下午 2:47 分，儲能系統突然以最大功率放電，導致電網頻率異常。事後調查時，你需要回答以下問題：

- 是誰下達了放電指令？是控制策略自動發出的，還是操作人員手動下達的？
- 指令的具體內容是什麼？目標功率多少？
- 指令下達前，系統的 SOC 是多少？是否已經觸發保護機制？

如果沒有完整的指令記錄，這些問題都無法回答。更糟的是，如果記錄在事故發生時恰好因為磁碟故障而遺失，你連為自己辯護的機會都沒有。

### 事故分析：故障前到底發生了什麼？

工業系統的故障往往不是瞬間發生的，而是一連串微小異常累積的結果。電池溫度在過去 48 小時內緩慢上升，Modbus 通訊在故障前 30 分鐘開始出現間歇性超時，某個 PCS 的效率在最近一週持續下降......

這些線索散落在不同的資料流中。只有當你把告警記錄、設備數據、操作指令放在一起交叉比對時，才能拼湊出完整的故事。這要求你的資料儲存方案不僅可靠，而且支援高效的歷史查詢。

---

## MongoDB Replica Set 基礎

為什麼選擇 MongoDB 而不是 PostgreSQL 或 InfluxDB？工業設備的數據結構是高度異質的——不同廠牌的 PCS 回傳的暫存器欄位完全不同，同一廠牌不同型號的設備點位定義也可能不一樣。MongoDB 的 schema-less 特性讓你能優雅地處理這種異質性，而不需要為每種設備設計不同的 table schema。

### Primary、Secondary 與 Arbiter

MongoDB Replica Set 由三種角色組成：

**Primary** 是唯一接受寫入操作的節點。所有寫入都先到 Primary，再透過 oplog（操作日誌）複製到 Secondary 節點。在工業場景中，Primary 就是你的「主存儲節點」。

**Secondary** 持續從 Primary 複製資料，保持數據同步。在 Primary 故障時，Replica Set 會自動選舉一個 Secondary 升格為新的 Primary。這個過程通常在 10-12 秒內完成。

**Arbiter** 是一個輕量級節點，不存儲任何數據，唯一的作用是在選舉時投票。當你只有兩個數據節點時，Arbiter 可以打破平票僵局，確保選舉能成功完成。

一個典型的工業部署架構如下：

```
┌─────────────┐     ┌─────────────┐     ┌──────────────┐
│  Primary    │────>│  Secondary  │     │   Arbiter    │
│  mongo1     │     │  mongo2     │     │   mongo3     │
│  (讀+寫)    │<────│  (唯讀備援)  │     │  (僅投票)     │
│  資料完整    │     │  資料完整    │     │  無資料      │
└─────────────┘     └─────────────┘     └──────────────┘
      ▲                    ▲                    ▲
      └────── oplog 複製 ───┘                    │
      └──────── election 投票 ──────────────────┘
```

### Write Concern：寫入保證等級

Write Concern 決定了「一筆寫入操作在什麼條件下被認為是成功的」。這是工業資料持久化最關鍵的配置：

| Write Concern | 含義 | 適用場景 |
|--------------|------|---------|
| `w: 1` | Primary 確認即成功 | 即時設備數據（可接受少量遺失） |
| `w: "majority"` | 多數節點確認 | 告警記錄、指令審計（不可遺失） |
| `w: 1, j: true` | Primary 寫入 journal 確認 | Primary 重啟後資料不遺失 |
| `w: "majority", j: true` | 多數節點寫入 journal | 最高保障，效能最低 |

在工業場景中，我們的建議是：

- **設備即時數據**（每秒幾十筆）：`w: 1`——即使 Primary 當機遺失幾秒數據，也不會影響系統安全。
- **告警記錄**：`w: "majority"`——告警是事故分析的關鍵證據，絕不能遺失。
- **操作指令記錄**：`w: "majority", j: true`——這是稽核的核心，需要最高等級的保障。

### Read Preference：讀取偏好

Read Preference 決定了讀取操作要從哪個節點執行：

- `primary`：只從 Primary 讀取。保證讀到最新數據，但 Primary 負載較高。
- `primaryPreferred`：優先 Primary，Primary 不可用時降級到 Secondary。
- `secondary`：只從 Secondary 讀取。分散 Primary 負載，但可能讀到稍微過時的數據。
- `secondaryPreferred`：優先 Secondary，適合歷史數據查詢。
- `nearest`：選擇網路延遲最低的節點。

工業場景的最佳實踐：

- **控制迴路讀取**（需要最新數據）：`primary`
- **儀表板與報表**（容許數秒延遲）：`secondaryPreferred`
- **歷史數據分析**：`secondary`

---

## csp_lib 的 MongoDB 配置

csp_lib 的 `MongoConfig` 設計為 frozen dataclass，同時支援 Standalone 和 Replica Set 兩種模式，並提供完整的 TLS 與認證支持：

```python
from csp_lib.mongo import MongoConfig, create_mongo_client

# === Standalone 模式（開發環境） ===
dev_config = MongoConfig(
    host="localhost",
    port=27017,
)

# === Replica Set 模式（生產環境） ===
prod_config = MongoConfig(
    replica_hosts=("mongo1:27017", "mongo2:27017", "mongo3:27017"),
    replica_set="rs0",
    username="ems_user",
    password="secure_password",
    auth_source="admin",
)

# === 帶 TLS 的 Replica Set（高安全環境） ===
secure_config = MongoConfig(
    replica_hosts=("mongo1:27017", "mongo2:27017", "mongo3:27017"),
    replica_set="rs0",
    tls=True,
    tls_cert_key_file="/path/to/client.pem",
    tls_ca_file="/path/to/ca.crt",
    auth_mechanism="MONGODB-X509",
)

# 根據配置自動建立客戶端
client = create_mongo_client(prod_config)
db = client["ems"]
```

`MongoConfig` 的幾個設計要點值得注意：

1. **模式自動判斷**：當同時提供 `replica_hosts` 和 `replica_set` 時，自動切換為 Replica Set 模式，`directConnection` 會被設為 `False`。
2. **驗證一致性**：`__post_init__` 確保 `replica_hosts` 和 `replica_set` 必須同時提供或同時省略，避免配置錯誤。
3. **超時配置**：`server_selection_timeout_ms`、`connect_timeout_ms`、`socket_timeout_ms` 三個獨立的超時參數，讓你可以針對工業網路的特性進行調校。

`create_mongo_client` 工廠函式的內部邏輯會根據 `is_replica_set_mode` 屬性自動選擇 URI 格式和連線參數，並在 Replica Set 模式下自動設定 `replicaSet` 和 `directConnection=False`：

```python
def create_mongo_client(config: MongoConfig) -> AsyncIOMotorClient:
    # Replica Set 模式：URI 包含所有節點
    if config.is_replica_set_mode:
        hosts = ",".join(config.replica_hosts)
        uri = f"mongodb://{hosts}"
    else:
        uri = f"mongodb://{config.host}:{config.port}"

    kwargs: dict = {}

    if config.is_replica_set_mode:
        kwargs["replicaSet"] = config.replica_set
        kwargs["directConnection"] = False
    else:
        kwargs["directConnection"] = config.direct_connection

    # ... Auth, TLS, Timeout 配置 ...

    client = AsyncIOMotorClient(uri, **kwargs)
    return client
```

注意這裡使用的是 `motor` 的 `AsyncIOMotorClient`，與 csp_lib 全框架 async-first 的設計哲學一致。所有的資料庫操作都是非同步的，不會阻塞控制迴路。

---

## 告警持久化：AlarmPersistenceManager

csp_lib 的告警持久化系統是一個教科書等級的觀察者模式實作。`AlarmPersistenceManager` 訂閱設備事件，自動將告警記錄寫入 MongoDB：

### 資料模型：AlarmRecord

```python
from csp_lib.manager.alarm import AlarmRecord, AlarmType, AlarmStatus

# 告警記錄的核心結構
record = AlarmRecord(
    alarm_key="pcs_01:disconnect:DISCONNECT",  # 業務唯一鍵
    device_id="pcs_01",
    alarm_type=AlarmType.DISCONNECT,            # 斷線 | 設備告警
    alarm_code="DISCONNECT",
    name="設備斷線",
    level=AlarmLevel.WARNING,
    description="Modbus TCP 連線超時",
    occurred_at=datetime.now(timezone.utc),
    resolved_at=None,                           # None = 進行中
    status=AlarmStatus.ACTIVE,                  # ACTIVE | RESOLVED
)

# 轉換為 MongoDB document
doc = record.to_document()
# {
#     "alarm_key": "pcs_01:disconnect:DISCONNECT",
#     "device_id": "pcs_01",
#     "alarm_type": "disconnect",
#     "alarm_code": "DISCONNECT",
#     "name": "設備斷線",
#     "level": "warning",
#     "status": "active",
#     "occurred_at": datetime(...),
#     "resolved_at": None,
#     ...
# }
```

`alarm_key` 的設計特別精巧：它由 `device_id:alarm_type:alarm_code` 組成，作為業務唯一鍵。同一設備的同一種告警在 `ACTIVE` 狀態下不會重複寫入——`MongoAlarmRepository.upsert()` 會先檢查是否已存在相同 key 且狀態為 ACTIVE 的記錄：

```python
async def upsert(self, record: AlarmRecord) -> tuple[str, bool]:
    existing = await self._collection.find_one({
        "alarm_key": record.alarm_key,
        "status": AlarmStatus.ACTIVE.value,
    })

    if existing:
        return existing["_id"], False  # 已存在，不重複寫入
    result = await self._collection.insert_one(record.to_document())
    return result.inserted_id, True     # 新告警，寫入成功
```

### 事件驅動的持久化流程

`AlarmPersistenceManager` 繼承自 `DeviceEventSubscriber`，訂閱四種設備事件：

```python
from csp_lib.manager.alarm import (
    AlarmPersistenceManager,
    MongoAlarmRepository,
    AlarmPersistenceConfig,
)
from csp_lib.mongo import MongoConfig, create_mongo_client

# 1. 建立 MongoDB 連線
config = MongoConfig(
    replica_hosts=("mongo1:27017", "mongo2:27017", "mongo3:27017"),
    replica_set="rs0",
)
client = create_mongo_client(config)
db = client["ems"]

# 2. 建立 Repository
alarm_repo = MongoAlarmRepository(db)
await alarm_repo.ensure_indexes()  # 啟動時建立索引

# 3. 建立持久化管理器
alarm_manager = AlarmPersistenceManager(
    repository=alarm_repo,
    config=AlarmPersistenceConfig(
        disconnect_code="DISCONNECT",
        disconnect_name="設備斷線",
    ),
)

# 4. 訂閱設備事件
alarm_manager.subscribe(pcs_device)
alarm_manager.subscribe(meter_device)
alarm_manager.subscribe(inverter_device)

# 從此刻起，以下事件會自動持久化：
# - disconnected → 寫入斷線告警 (AlarmType.DISCONNECT)
# - connected    → 解除斷線告警 (status → RESOLVED)
# - alarm_triggered → 寫入設備告警 (AlarmType.DEVICE_ALARM)
# - alarm_cleared   → 解除設備告警 (status → RESOLVED)
```

整個流程是事件驅動的：設備斷線時自動產生告警記錄，重新連線時自動更新 `resolved_at` 和 `status`。你不需要寫任何輪詢邏輯。

告警解除的邏輯同樣簡潔，`MongoAlarmRepository.resolve()` 使用原子性的 `update_one` 操作，確保在並行環境下不會出現競態條件：

```python
async def resolve(self, alarm_key: str, resolved_at: datetime) -> bool:
    result = await self._collection.update_one(
        {
            "alarm_key": alarm_key,
            "status": AlarmStatus.ACTIVE.value,
        },
        {
            "$set": {
                "status": AlarmStatus.RESOLVED.value,
                "resolved_at": resolved_at,
            }
        },
    )
    return result.modified_count == 1
```

### 查詢進行中的告警

運維面板經常需要顯示「目前有哪些設備在告警中」，Repository 提供了便捷的查詢方法：

```python
# 查詢所有進行中的告警
active_alarms = await alarm_repo.get_active_alarms()
for alarm in active_alarms:
    print(f"[{alarm.level.value}] {alarm.device_id}: {alarm.name}")

# 查詢特定設備的進行中告警
pcs_alarms = await alarm_repo.get_active_by_device("pcs_01")
```

---

## 指令審計軌跡：WriteCommandManager

每一筆寫入指令——無論來自控制策略的自動決策、操作人員的手動操作，還是外部系統的 API 呼叫——都會被完整記錄。

### 資料模型：WriteCommand 與 CommandRecord

```python
from csp_lib.manager.command import (
    WriteCommand,
    ActionCommand,
    CommandRecord,
    CommandSource,
    CommandStatus,
    WriteCommandManager,
    MongoCommandRepository,
)

# 點位寫入指令
write_cmd = WriteCommand(
    device_id="pcs_01",
    point_name="active_power_setpoint",
    value=500.0,                            # 500 kW
    source=CommandSource.REST_API,           # 來自 REST API
    source_info={
        "user_id": "operator_wang",
        "ip": "192.168.1.50",
        "session_id": "abc123",
    },
    verify=True,                            # 寫入後讀回驗證
)

# 動作指令（高階操作）
action_cmd = ActionCommand(
    device_id="generator_01",
    action="start",
    source=CommandSource.INTERNAL,           # 來自控制策略
    source_info={"strategy": "island_mode"},
)
```

`CommandSource` 枚舉記錄了指令的來源管道，這對稽核至關重要：

| CommandSource | 說明 | 典型場景 |
|--------------|------|---------|
| `REDIS_PUBSUB` | 透過 Redis Pub/Sub 接收 | 外部 SCADA 系統 |
| `GRPC` | 透過 gRPC 介面 | 微服務架構間呼叫 |
| `REST_API` | 透過 REST API | 操作員手動操作 |
| `INTERNAL` | 內部控制策略 | 自動控制邏輯 |

### 完整的指令生命週期

`WriteCommandManager` 實作了指令從接收到完成的完整生命週期追蹤：

```python
# 1. 建立 Repository
cmd_repo = MongoCommandRepository(db)
await cmd_repo.ensure_indexes()

# 2. 建立 Manager 並註冊設備
cmd_manager = WriteCommandManager(repository=cmd_repo)
cmd_manager.register_device(pcs_device)
cmd_manager.register_device(meter_device)

# 3. 執行指令
command = WriteCommand(
    device_id="pcs_01",
    point_name="active_power_setpoint",
    value=500.0,
    source=CommandSource.REST_API,
    source_info={"user_id": "operator_wang"},
)
result = await cmd_manager.execute(command)
```

執行過程中，`CommandRecord` 的狀態會按序更新：

```
PENDING → EXECUTING → SUCCESS / FAILED / DEVICE_NOT_FOUND
```

每個狀態轉換都會呼叫 `repository.update_status()` 寫入 MongoDB，並記錄對應的時間戳：

```python
# 內部流程：
# 1. 建立 PENDING 記錄 → repository.create(record)
# 2. 查找設備
# 3. 更新為 EXECUTING → repository.update_status(id, EXECUTING)
#    此時記錄 executed_at = datetime.now(timezone.utc)
# 4. 執行寫入
# 5. 更新為 SUCCESS/FAILED → repository.update_status(id, SUCCESS, result)
#    此時記錄 completed_at = datetime.now(timezone.utc)
```

最終，一筆完整的 `CommandRecord` 在 MongoDB 中看起來像這樣：

```json
{
    "command_id": "a1b2c3d4-e5f6-...",
    "device_id": "pcs_01",
    "point_name": "active_power_setpoint",
    "value": 500.0,
    "source": "rest_api",
    "source_info": {
        "user_id": "operator_wang",
        "ip": "192.168.1.50"
    },
    "status": "success",
    "result": {
        "status": "success",
        "point_name": "active_power_setpoint",
        "value": 500.0,
        "error_message": null
    },
    "created_at": "2026-03-10T06:30:00Z",
    "executed_at": "2026-03-10T06:30:00.012Z",
    "completed_at": "2026-03-10T06:30:00.145Z",
    "error_message": null
}
```

從這筆記錄，你可以精確地知道：誰（operator_wang）在什麼時候（06:30:00）透過什麼管道（REST API）對哪台設備（pcs_01）的哪個點位（active_power_setpoint）寫入了什麼值（500.0），以及執行結果如何（success）、耗時多久（133ms）。

---

## 批次上傳優化：MongoBatchUploader

設備數據的上傳量遠大於告警和指令記錄。一個管理 20 台 PCS 的系統，每秒可能產生 20 x 50 = 1000 筆數據點。如果每筆都獨立 `insert_one`，MongoDB 的寫入壓力和網路開銷會非常大。

csp_lib 的解決方案是 `MongoBatchUploader`——一個集合式的批次上傳器。

### 架構設計

```
                                    ┌─────────────────┐
設備 A ─── read_complete ──→        │   BatchQueue    │
                              ├────→│  "pcs_data"     │──→ MongoWriter.write_batch()
設備 B ─── read_complete ──→  │     └─────────────────┘         │
                              │     ┌─────────────────┐         ↓
設備 C ─── read_complete ──→  ├────→│   BatchQueue    │──→ MongoDB
                              │     │  "meter_data"   │     insert_many()
設備 D ─── read_complete ──→  │     └─────────────────┘
                              │
                         DataUploadManager
                     (事件驅動 → enqueue)
```

`MongoBatchUploader` 由三個元件組成：

1. **BatchQueue**：每個 collection 一個獨立的 async-safe 佇列。使用 `asyncio.Lock` 確保協程安全，支援容量上限保護。
2. **MongoWriter**：負責執行 `insert_many` 批次寫入，回報 `WriteResult`。
3. **MongoBatchUploader**：組合 queue 和 writer，管理定期 flush 和重試邏輯。

### 使用方式

```python
from csp_lib.mongo import MongoBatchUploader, UploaderConfig, create_mongo_client, MongoConfig

# 配置批次上傳器
uploader_config = UploaderConfig(
    flush_interval=5,            # 每 5 秒 flush 一次
    batch_size_threshold=100,    # 單一 collection 累積 100 筆後立即 flush
    max_queue_size=10000,        # 佇列上限 10000 筆（超過丟棄最舊）
    max_retry_count=3,           # 單批最多重試 3 次
)

# 建立上傳器
config = MongoConfig(
    replica_hosts=("mongo1:27017", "mongo2:27017", "mongo3:27017"),
    replica_set="rs0",
)
client = create_mongo_client(config)
db = client["ems"]

uploader = MongoBatchUploader(db, config=uploader_config)
uploader.start()  # 啟動背景 flush loop

# 資料自動入隊
await uploader.enqueue("pcs_data", {
    "device_id": "pcs_01",
    "timestamp": datetime.now(timezone.utc),
    "active_power": 487.3,
    "reactive_power": 12.1,
    "soc": 72.5,
})

# 結束時確保所有資料都已寫入
await uploader.stop()
```

### 失敗處理與重試

`MongoBatchUploader` 的重試邏輯是層級式的：

```python
async def _handle_write_failure(self, collection_name, documents):
    self._retry_counts[collection_name] += 1
    retry_count = self._retry_counts[collection_name]

    if retry_count <= self._config.max_retry_count:
        # 寫入失敗：放回佇列前端，下次 flush 重試
        await self._queues[collection_name].restore(documents)
    else:
        # 超過最大重試次數：丟棄資料，記錄 error log
        self._retry_counts[collection_name] = 0
```

`BatchQueue.restore()` 的設計特別巧妙：它把失敗的資料放回佇列的「前端」（使用 `appendleft`），確保下次 flush 時這些資料優先被處理。同時，如果佇列已滿，會停止 restore 避免新資料被擠出去：

```python
async def restore(self, documents: list[dict]) -> None:
    async with self._lock:
        for doc in reversed(documents):
            if len(self._queue) < self.max_size:
                self._queue.appendleft(doc)
            else:
                break  # 佇列已滿，停止放回
```

### 與 DataUploadManager 整合

在實際使用中，你不需要手動呼叫 `enqueue`。`DataUploadManager` 會訂閱設備的 `read_complete` 事件，自動將數據送入上傳器。它還支援降頻儲存——例如設備每秒讀取一次，但只需要每 30 秒存一筆到 MongoDB：

```python
from csp_lib.manager.data import DataUploadManager

data_manager = DataUploadManager(uploader)

# 訂閱設備，指定 collection 和儲存間隔
data_manager.subscribe(pcs_device, collection_name="pcs_data", save_interval=30)
data_manager.subscribe(meter_device, collection_name="meter_data", save_interval=5)

# 設備讀取時資料自動上傳
# 設備斷線時自動上傳空值記錄（保留結構，讓前端圖表能正確顯示斷線區間）
```

`save_interval` 的設計非常實用。設備通訊層仍然維持每秒讀取（確保控制迴路的即時性），但儲存層可以按需降頻，大幅減少 MongoDB 的寫入量。

---

## Schema 設計考量

### 時間序列集合用於設備數據

MongoDB 5.0 以後支援原生的時間序列集合（Time Series Collection），特別適合設備數據：

```javascript
// MongoDB shell
db.createCollection("pcs_data", {
    timeseries: {
        timeField: "timestamp",
        metaField: "device_id",
        granularity: "seconds"
    }
})
```

時間序列集合的優勢：
- 自動壓縮：相同 `device_id` 的連續資料會被自動合併儲存，磁碟用量可減少 80% 以上。
- 高效查詢：按時間範圍查詢的速度比普通集合快數倍。
- 自動 bucketing：MongoDB 內部按時間區間分桶，避免碎片化。

### 一般集合用於告警和指令

告警和指令記錄使用一般集合，因為它們需要頻繁的狀態更新（`update_one`）——時間序列集合不支援就地更新。

`MongoAlarmRepository` 和 `MongoCommandRepository` 都在啟動時呼叫 `ensure_indexes()` 建立必要的索引：

```python
# AlarmRepository 索引
await collection.create_indexes([
    IndexModel([("alarm_key", 1)]),                      # 唯一鍵查詢
    IndexModel([("device_id", 1)]),                       # 設備篩選
    IndexModel([("status", 1)]),                          # 狀態篩選
    IndexModel([("device_id", 1), ("status", 1)]),        # 複合查詢
    IndexModel([("occurred_at", -1)]),                    # 時間排序
])

# CommandRepository 索引
await collection.create_indexes([
    IndexModel([("command_id", 1)], unique=True),         # 唯一指令 ID
    IndexModel([("device_id", 1)]),                       # 設備篩選
    IndexModel([("status", 1)]),                          # 狀態篩選
    IndexModel([("created_at", -1)]),                     # 時間排序
])
```

### TTL 索引用於自動過期

對於不需要永久保留的數據（如即時設備讀數），可以使用 MongoDB 的 TTL 索引自動清除過期資料：

```javascript
// 設備數據保留 90 天
db.pcs_data.createIndex(
    { "timestamp": 1 },
    { expireAfterSeconds: 90 * 24 * 3600 }
)

// 已解除的告警保留 1 年
db.alarms.createIndex(
    { "resolved_at": 1 },
    { expireAfterSeconds: 365 * 24 * 3600, partialFilterExpression: { status: "resolved" } }
)
```

注意使用 `partialFilterExpression`：TTL 索引只對 `status: "resolved"` 的告警生效，進行中的告警（`ACTIVE`）永遠不會被自動刪除。

---

## 完整的持久化架構總覽

把以上所有元件組合在一起，一個完整的工業資料持久化架構如下：

```
AsyncModbusDevice (x N)
  │
  ├── read_complete 事件
  │     └── DataUploadManager
  │           └── MongoBatchUploader → MongoDB (pcs_data / meter_data)
  │                                    [時間序列集合, w:1]
  │
  ├── disconnected / connected 事件
  │     └── AlarmPersistenceManager
  │           └── MongoAlarmRepository → MongoDB (alarms)
  │                                      [一般集合, w:majority]
  │
  ├── alarm_triggered / alarm_cleared 事件
  │     └── AlarmPersistenceManager (同上)
  │
  └── (外部指令)
        └── WriteCommandManager
              └── MongoCommandRepository → MongoDB (commands)
                                            [一般集合, w:majority, j:true]
```

三種資料流，三種寫入策略，三種保留期限——各自匹配其業務重要性。

---

## 關鍵要點

1. **工業資料持久化不是可選的**——法規遵循、稽核軌跡、事故分析都依賴完整的歷史記錄。
2. **MongoDB Replica Set 提供自動容錯**——Primary 故障時自動選舉新 Primary，通常在 10-12 秒內完成。
3. **Write Concern 需要按資料重要性分級**——即時數據用 `w:1`，告警和指令用 `w:majority`。
4. **批次上傳是高頻數據的必備優化**——`MongoBatchUploader` 透過佇列 + 定期 flush + 重試，在可靠性和效能間取得平衡。
5. **事件驅動架構讓持久化邏輯與業務邏輯解耦**——設備只管發事件，持久化管理器自動處理後續儲存。

---

## 下一篇預告

資料安全地存入 MongoDB 了，但如果控制器所在的伺服器網路介面故障，現場設備就會失去與控制系統的連線。下一篇文章，我們將深入 Keepalived 與 VRRP 協定，了解如何在網路層實現高可用切換，確保虛擬 IP 在節點故障時能快速遷移到備援節點。
