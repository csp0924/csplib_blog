# 時序資料庫實戰：MongoDB Time Series 與工業數據持久化

> **從零打造工業控制協定框架：給軟體工程師的 CSP Library 實戰系列**
> Part 3 — 資料管線篇 | Article 12

---

## 前言

上一篇我們用 Redis 解決了即時狀態同步的問題，但 Redis 是記憶體資料庫——它不適合儲存數週、數月的歷史資料。工業系統需要長期保存時序資料（Time Series Data），用於：

- 功率曲線回溯：操作員查看過去 24 小時的 PCS 輸出功率。
- 能耗統計：計算每日、每月的發電量與用電量。
- 故障分析：告警發生前後的設備數據，用於 Root Cause Analysis。
- 合規報告：電力調度單位要求的運行記錄。

本篇深入 csp_lib 的 MongoDB 整合層，涵蓋從連線配置到批次寫入、從事件驅動上傳到資料查詢模式的完整流程。

---

## 1. 為什麼時序資料在工業系統中至關重要

工業設備的讀取頻率通常是每秒 1 次或更快。以一個包含 10 台設備的 EMS 為例：

| 參數 | 數值 |
|------|------|
| 設備數量 | 10 |
| 每台點位數 | 30 |
| 讀取頻率 | 1 Hz |
| 每筆文件大小 | ~500 bytes |
| **每日資料量** | **~420 MB** |
| **每月資料量** | **~12.6 GB** |

這個量級對 MongoDB 來說很輕鬆，但如果不做適當的批次處理和降頻，仍然會造成不必要的 I/O 負擔。

### MongoDB Time Series Collections

MongoDB 從 5.0 開始支援 Time Series Collections，專為時序資料設計：

- **自動時間分區**：資料按時間範圍組織在底層的 bucket 中。
- **壓縮**：相鄰時間點的數據有高度相似性，壓縮率可達 5-10 倍。
- **高效查詢**：按時間範圍查詢時，引擎直接跳過不相關的 bucket。

```javascript
// MongoDB shell - 建立 Time Series Collection
db.createCollection("device_data", {
    timeseries: {
        timeField: "timestamp",
        metaField: "device_id",
        granularity: "seconds"
    },
    expireAfterSeconds: 2592000  // 30 天自動過期
})
```

在 csp_lib 中，建立 collection 是你的職責（通常在部署腳本中完成），而框架負責的是如何高效地把資料塞進去。

---

## 2. csp_lib 的 MongoDB 整合

### 2.1 連線配置

與 Redis 模組的設計哲學一致，`MongoConfig` 同樣使用 frozen dataclass：

```python
from csp_lib.mongo import MongoConfig, create_mongo_client

# 最簡配置
config = MongoConfig(host="localhost", port=27017)

# 生產配置：帶認證和 TLS
config = MongoConfig(
    host="mongo.example.com",
    port=27017,
    username="ems_writer",
    password="strong_password",
    auth_source="admin",
    tls=True,
    tls_ca_file="/etc/certs/ca.crt",
)

# 建立客戶端
client = create_mongo_client(config)
db = client["ems_database"]
```

### 2.2 Replica Set 模式

生產環境必須使用 Replica Set 確保資料不丟失：

```python
config = MongoConfig(
    replica_hosts=("rs1:27017", "rs2:27017", "rs3:27017"),
    replica_set="ems-rs",
    username="ems_writer",
    password="strong_password",
    tls=True,
    tls_cert_key_file="/etc/certs/client.pem",
    tls_ca_file="/etc/certs/ca.crt",
)

client = create_mongo_client(config)
```

`create_mongo_client()` 函數根據配置自動選擇連線模式，並設定正確的 `directConnection` 參數：

```python
def create_mongo_client(config: MongoConfig) -> AsyncIOMotorClient:
    if config.is_replica_set_mode:
        hosts = ",".join(config.replica_hosts)
        uri = f"mongodb://{hosts}"
        kwargs["replicaSet"] = config.replica_set
        kwargs["directConnection"] = False  # Replica Set 必須設為 False
    else:
        uri = f"mongodb://{config.host}:{config.port}"
        kwargs["directConnection"] = config.direct_connection
    # ... TLS, Auth, Timeout 配置
    return AsyncIOMotorClient(uri, **kwargs)
```

### 2.3 X.509 認證

在高安全要求的工業環境中，用戶名密碼認證可能不夠。csp_lib 支援 MongoDB 的 X.509 客戶端憑證認證：

```python
config = MongoConfig(
    host="mongo.example.com",
    port=27017,
    tls=True,
    tls_cert_key_file="/path/to/client.pem",  # 包含 cert + key
    tls_ca_file="/path/to/ca.crt",
    auth_mechanism="MONGODB-X509",
)
```

---

## 3. MongoBatchUploader：高效批次寫入

在工業系統中，**絕對不要每次設備讀取都立即寫入資料庫**。原因有二：

1. **I/O 效率**：`insert_many` 比 N 次 `insert_one` 快 10 倍以上。
2. **背壓管理**：如果資料庫暫時不可用，你需要緩衝機制避免資料丟失。

csp_lib 的 `MongoBatchUploader` 正是為此設計的。

### 3.1 架構拆解

`MongoBatchUploader` 由三個元件組成：

```
┌─────────────────────────────────────────────┐
│           MongoBatchUploader                 │
│                                              │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐  │
│  │BatchQueue│  │BatchQueue│  │BatchQueue│  │
│  │ coll_A   │  │ coll_B   │  │ coll_C   │  │
│  └─────┬────┘  └─────┬────┘  └─────┬────┘  │
│        └──────┬──────┘──────┬──────┘        │
│               ▼              ▼               │
│          MongoWriter                         │
│         (insert_many)                        │
└─────────────────────────────────────────────┘
```

- **BatchQueue**：每個 collection 一個 async-safe 佇列，使用 `asyncio.Lock` 保證協程安全。
- **MongoWriter**：負責執行 `insert_many`，回傳 `WriteResult`（成功/失敗）。
- **MongoBatchUploader**：協調多個 BatchQueue，排程定期 flush，處理重試邏輯。

### 3.2 使用方式

```python
from motor.motor_asyncio import AsyncIOMotorClient
from csp_lib.mongo import MongoBatchUploader, UploaderConfig

# 建立 MongoDB 連線
client = AsyncIOMotorClient("mongodb://localhost:27017")
db = client["ems"]

# 配置上傳器
config = UploaderConfig(
    flush_interval=5,           # 每 5 秒 flush 一次
    batch_size_threshold=100,   # 單一 collection 累積 100 筆觸發上傳
    max_queue_size=10000,       # 佇列上限（超過丟棄最舊資料）
    max_retry_count=3,          # 最大重試次數
)

# 建立並啟動上傳器
uploader = MongoBatchUploader(db, config=config)
uploader.start()

# 註冊 collection（可選，enqueue 時會自動註冊）
uploader.register_collection("device_data")

# 塞入資料
await uploader.enqueue("device_data", {
    "device_id": "pcs_1",
    "timestamp": datetime.now(timezone.utc),
    "active_power": 50.2,
    "reactive_power": 10.1,
    "soc": 85.3,
})

# 關閉時確保所有資料都已上傳
await uploader.stop()
```

### 3.3 Flush 機制

`MongoBatchUploader` 有兩種觸發 flush 的方式：

1. **閾值觸發**：某個 collection 的佇列累積達到 `batch_size_threshold`（預設 100 筆），立即上傳。
2. **定時觸發**：每 `flush_interval` 秒（預設 5 秒），不論累積多少筆都上傳。

```python
async def _flush_loop(self) -> None:
    while not self._stop_event.is_set():
        try:
            # 檢查達到閾值的 collection 並立即上傳
            for name, queue in list(self._queues.items()):
                if queue.size_sync() >= self._config.batch_size_threshold:
                    await self._flush_collection(name)

            # 等待下一次定時 flush 或收到停止信號
            try:
                await asyncio.wait_for(
                    self._stop_event.wait(),
                    timeout=self._config.flush_interval,
                )
                await self.flush_all()  # 停止前 flush 所有資料
                break
            except asyncio.TimeoutError:
                await self.flush_all()  # 定時 flush
        except Exception as e:
            logger.error(f"MongoBatchUploader: flush loop 錯誤: {e}")
            await asyncio.sleep(1)
```

### 3.4 BatchQueue：背壓與容量控制

`BatchQueue` 使用 `collections.deque` 實現，有上限保護：

```python
class BatchQueue:
    def __init__(self, collection_name: str, max_size: int = 10000) -> None:
        self.collection_name = collection_name
        self.max_size = max_size
        self._queue: deque[dict[str, Any]] = deque()
        self._lock = asyncio.Lock()

    async def enqueue(self, document: dict[str, Any]) -> bool:
        async with self._lock:
            if len(self._queue) >= self.max_size:
                self._queue.popleft()  # 丟棄最舊的資料
                return False  # 通知呼叫者有資料被丟棄
            self._queue.append(document)
            return True
```

當佇列滿了，**丟棄最舊的資料而不是阻塞生產者**。這個決策在工業系統中是正確的——寧可丟失幾筆歷史資料，也不能讓設備讀取循環卡住。

`size_sync()` 方法提供無鎖的快速大小檢查，用於 flush loop 的閾值判斷：

```python
def size_sync(self) -> int:
    """同步取得目前佇列大小（僅供檢查用，不保證精確）"""
    return len(self._queue)
```

### 3.5 重試機制

寫入失敗時，`MongoBatchUploader` 會將資料放回佇列前端重試，直到超過最大重試次數：

```python
async def _handle_write_failure(
    self, collection_name: str, documents: list[dict[str, Any]]
) -> None:
    self._retry_counts[collection_name] += 1
    retry_count = self._retry_counts[collection_name]

    if retry_count <= self._config.max_retry_count:
        logger.warning(
            f"'{collection_name}' 寫入失敗，"
            f"重試 {retry_count}/{self._config.max_retry_count}"
        )
        await self._queues[collection_name].restore(documents)
    else:
        logger.error(
            f"'{collection_name}' 超過最大重試次數，"
            f"丟棄 {len(documents)} 筆資料"
        )
        self._retry_counts[collection_name] = 0
```

`BatchQueue.restore()` 使用 `deque.appendleft()` 將資料放回佇列前端，確保重試順序正確：

```python
async def restore(self, documents: list[dict[str, Any]]) -> None:
    async with self._lock:
        for doc in reversed(documents):
            if len(self._queue) < self.max_size:
                self._queue.appendleft(doc)
            else:
                logger.warning("restore 時容量已滿，丟棄資料")
                break
```

### 3.6 MongoWriter：單一職責

`MongoWriter` 只做一件事：執行 `insert_many` 並回報結果。

```python
class MongoWriter:
    async def write_batch(
        self, collection_name: str, documents: list[dict[str, Any]]
    ) -> WriteResult:
        if not documents:
            return WriteResult(success=True, inserted_count=0)
        try:
            result = await self._db[collection_name].insert_many(documents)
            return WriteResult(
                success=True,
                inserted_count=len(result.inserted_ids),
            )
        except Exception as e:
            return WriteResult(
                success=False,
                error_message=f"寫入 '{collection_name}' 失敗: {e}",
            )
```

注意它不會拋出例外——所有錯誤都透過 `WriteResult` 回報。這讓上層的重試邏輯更容易實現。

---

## 4. DataUploadManager：從設備事件到儲存

`DataUploadManager` 是連接設備層和儲存層的橋樑。它訂閱設備的 `read_complete` 事件，自動將資料塞入 `MongoBatchUploader`。

### 4.1 基本用法

```python
from csp_lib.manager.data import DataUploadManager
from csp_lib.mongo import MongoBatchUploader

# 建立上傳器
uploader = MongoBatchUploader(db).start()

# 建立資料上傳管理器
data_manager = DataUploadManager(uploader)

# 訂閱設備（指定 collection 名稱）
data_manager.subscribe(pcs_device, collection_name="pcs_data")
data_manager.subscribe(meter_device, collection_name="meter_data")

# 之後設備每次讀取，資料自動進入上傳佇列
```

### 4.2 降頻儲存

設備可能每秒讀取一次，但你不一定需要每秒都存一筆。`save_interval` 參數讓你控制儲存頻率：

```python
# 設備每 1 秒讀取一次，但每 30 秒才存一筆至 MongoDB
data_manager.subscribe(
    pcs_device,
    collection_name="pcs_data",
    save_interval=30,
)
```

降頻的實現方式是記錄上次儲存的 monotonic 時間戳，只在間隔超過閾值時才上傳：

```python
async def _on_read_complete(self, payload: ReadCompletePayload) -> None:
    device_id = payload.device_id
    collection_name = self._device_collection.get(device_id)
    if not collection_name:
        return

    # 快取結構供斷線時使用（無論是否降頻都要更新）
    self._last_values[device_id] = payload.values

    # 降頻檢查
    interval = self._save_intervals.get(device_id)
    if interval is not None:
        now = time.monotonic()
        last_save = self._last_save_times.get(device_id)
        if last_save is not None and (now - last_save) < interval:
            return  # 尚未到達儲存間隔，跳過
        self._last_save_times[device_id] = now

    # 建立文件並上傳
    document = {
        "device_id": device_id,
        "timestamp": payload.timestamp,
        **payload.values,
    }
    await self._uploader.enqueue(collection_name, document)
```

### 4.3 斷線空值記錄

一個容易忽略的細節：**設備斷線時也需要寫入一筆記錄**。為什麼？

想像一個功率曲線圖：如果設備在 10:00 斷線，最後一筆資料是 50 kW，在 10:05 恢復後第一筆是 48 kW——如果不寫入斷線記錄，圖表會顯示一條從 50 到 48 的直線，而不是斷開。

`DataUploadManager` 在收到 `disconnected` 事件時，使用最後快取的值結構產生空值記錄：

```python
async def _on_disconnected(self, payload: DisconnectPayload) -> None:
    device_id = payload.device_id
    last_values = self._last_values.get(device_id)
    if last_values:
        # 保留結構，但所有葉節點設為 None
        null_values = {k: nullify_nested(v) for k, v in last_values.items()}
    else:
        return  # 無快取，無法產生空值記錄

    document = {
        "device_id": device_id,
        "timestamp": payload.timestamp,
        **null_values,
    }
    await self._uploader.enqueue(collection_name, document)
```

`nullify_nested()` 遞歸地將所有值替換為 None，但保留字典和列表的結構：

```python
def nullify_nested(value: Any) -> Any:
    if isinstance(value, dict):
        return {k: nullify_nested(v) for k, v in value.items()}
    elif isinstance(value, list):
        return [nullify_nested(item) for item in value]
    else:
        return None

# 範例
nullify_nested({"active_power": 50.2, "status": {"running": True, "mode": 2}})
# -> {"active_power": None, "status": {"running": None, "mode": None}}
```

---

## 5. 資料 Schema 設計

### 5.1 設備讀取資料

```json
{
    "device_id": "pcs_1",
    "timestamp": "2026-03-10T08:00:00.123Z",
    "active_power": 50.2,
    "reactive_power": 10.1,
    "soc": 85.3,
    "dc_voltage": 720.5,
    "grid_frequency": 59.98,
    "status": {
        "running": true,
        "fault": false,
        "mode": 2
    }
}
```

使用 `device_id` + `timestamp` 作為查詢的主要維度。在 Time Series Collection 中，`device_id` 是 `metaField`，`timestamp` 是 `timeField`。

### 5.2 能源統計記錄

csp_lib 的統計模組會產生另一類文件——區間能耗記錄：

```json
{
    "type": "energy",
    "device_id": "pcs_1",
    "interval_minutes": 15,
    "period_start": "2026-03-10T08:00:00Z",
    "period_end": "2026-03-10T08:15:00Z",
    "kwh": 12.35,
    "sample_count": 900,
    "meter_type": "instantaneous",
    "timestamp": "2026-03-10T08:15:01Z"
}
```

### 5.3 功率加總記錄

```json
{
    "type": "power_sum",
    "name": "p_total_pcs",
    "interval_minutes": 15,
    "period_start": "2026-03-10T08:00:00Z",
    "period_end": "2026-03-10T08:15:00Z",
    "total_power": 250.5,
    "device_count": 5,
    "timestamp": "2026-03-10T08:15:01Z"
}
```

### 5.4 索引建議

```javascript
// 設備資料查詢（按設備 + 時間範圍）
db.device_data.createIndex(
    { "device_id": 1, "timestamp": -1 }
)

// 統計記錄查詢（按類型 + 區間 + 時間）
db.statistics.createIndex(
    { "type": 1, "device_id": 1, "interval_minutes": 1, "period_start": -1 }
)

// 功率加總查詢
db.statistics.createIndex(
    { "type": 1, "name": 1, "period_start": -1 }
)
```

---

## 6. 查詢模式

### 6.1 最近 N 小時的功率曲線

```python
from datetime import datetime, timedelta, timezone

async def get_power_curve(
    db, device_id: str, hours: int = 24
) -> list[dict]:
    since = datetime.now(timezone.utc) - timedelta(hours=hours)
    cursor = db["device_data"].find(
        {
            "device_id": device_id,
            "timestamp": {"$gte": since},
        },
        {
            "_id": 0,
            "timestamp": 1,
            "active_power": 1,
            "reactive_power": 1,
        },
    ).sort("timestamp", 1)
    return await cursor.to_list(length=None)
```

### 6.2 每日能耗統計（Aggregation Pipeline）

```python
async def get_daily_energy(db, device_id: str, date: datetime) -> dict:
    start = date.replace(hour=0, minute=0, second=0, microsecond=0)
    end = start + timedelta(days=1)

    pipeline = [
        {
            "$match": {
                "type": "energy",
                "device_id": device_id,
                "interval_minutes": 15,
                "period_start": {"$gte": start, "$lt": end},
            }
        },
        {
            "$group": {
                "_id": "$device_id",
                "total_kwh": {"$sum": "$kwh"},
                "record_count": {"$sum": 1},
                "avg_samples_per_interval": {"$avg": "$sample_count"},
            }
        },
    ]

    results = await db["statistics"].aggregate(pipeline).to_list(length=1)
    return results[0] if results else {}
```

### 6.3 月度功率加總趨勢

```python
async def get_monthly_power_trend(
    db, name: str, year: int, month: int
) -> list[dict]:
    start = datetime(year, month, 1, tzinfo=timezone.utc)
    if month == 12:
        end = datetime(year + 1, 1, 1, tzinfo=timezone.utc)
    else:
        end = datetime(year, month + 1, 1, tzinfo=timezone.utc)

    pipeline = [
        {
            "$match": {
                "type": "power_sum",
                "name": name,
                "interval_minutes": 60,
                "period_start": {"$gte": start, "$lt": end},
            }
        },
        {
            "$project": {
                "_id": 0,
                "period_start": 1,
                "total_power": 1,
                "device_count": 1,
            }
        },
        {"$sort": {"period_start": 1}},
    ]
    return await db["statistics"].aggregate(pipeline).to_list(length=None)
```

### 6.4 Downsampling 查詢

對於長時間範圍（如一年），不可能回傳每秒的資料。可以用 `$bucket` 做 downsampling：

```python
async def get_downsampled_power(
    db, device_id: str, start: datetime, end: datetime,
    bucket_hours: int = 1
) -> list[dict]:
    bucket_ms = bucket_hours * 3600 * 1000

    pipeline = [
        {
            "$match": {
                "device_id": device_id,
                "timestamp": {"$gte": start, "$lte": end},
            }
        },
        {
            "$group": {
                "_id": {
                    "$subtract": [
                        {"$toLong": "$timestamp"},
                        {"$mod": [{"$toLong": "$timestamp"}, bucket_ms]},
                    ]
                },
                "avg_power": {"$avg": "$active_power"},
                "max_power": {"$max": "$active_power"},
                "min_power": {"$min": "$active_power"},
                "count": {"$sum": 1},
            }
        },
        {
            "$project": {
                "_id": 0,
                "bucket_start": {"$toDate": "$_id"},
                "avg_power": 1,
                "max_power": 1,
                "min_power": 1,
                "count": 1,
            }
        },
        {"$sort": {"bucket_start": 1}},
    ]
    return await db["device_data"].aggregate(pipeline).to_list(length=None)
```

---

## 7. 效能調校

### 7.1 批次大小選擇

`batch_size_threshold` 的選擇取決於你的延遲容忍度：

| batch_size | 平均延遲 | 吞吐量 | 適用場景 |
|-----------|---------|--------|---------|
| 10 | ~1s | 低 | 低流量，需要較即時 |
| 100（預設） | ~5s | 中 | 一般工業場景 |
| 500 | ~10s | 高 | 高流量，延遲不敏感 |

### 7.2 flush_interval 調整

`flush_interval` 是「最大延遲上限」——即使資料量未達 batch_size，也會在這個間隔內 flush：

```python
# 低延遲場景：每 2 秒 flush
config = UploaderConfig(flush_interval=2, batch_size_threshold=50)

# 高吞吐場景：每 10 秒 flush
config = UploaderConfig(flush_interval=10, batch_size_threshold=500)
```

### 7.3 佇列大小與背壓

`max_queue_size` 控制記憶體使用上限。以每筆 500 bytes 估算：

| max_queue_size | 記憶體估算 | 緩衝時間（10 設備/秒） |
|---------------|-----------|---------------------|
| 1,000 | ~500 KB | ~100 秒 |
| 10,000（預設） | ~5 MB | ~1,000 秒 |
| 100,000 | ~50 MB | ~10,000 秒 |

10,000 筆可以緩衝約 16 分鐘的資料——足夠應對大多數資料庫暫時不可用的情況。

### 7.4 Save Interval 策略

不同設備可能需要不同的儲存頻率：

```python
# PCS：每 5 秒存一筆（功率變化快，需要細粒度）
data_manager.subscribe(pcs_device, "pcs_data", save_interval=5)

# 電表：每 60 秒存一筆（累計型讀數，變化慢）
data_manager.subscribe(meter_device, "meter_data", save_interval=60)

# 環境感測器：每 300 秒存一筆（溫度、濕度變化緩慢）
data_manager.subscribe(env_sensor, "env_data", save_interval=300)
```

---

## 8. MongoDB Replica Set 高可用

在生產環境中，MongoDB Replica Set 提供：

1. **自動故障轉移**：Primary 故障時，Secondary 自動升格。
2. **讀取分流**：讀取操作可以導向 Secondary，減輕 Primary 負擔。
3. **資料持久性**：Write Concern `majority` 確保資料寫入多數節點。

```python
# Replica Set 配置
config = MongoConfig(
    replica_hosts=("mongo-1:27017", "mongo-2:27017", "mongo-3:27017"),
    replica_set="ems-rs",
    username="ems_writer",
    password="strong_password",
    server_selection_timeout_ms=5000,  # 5 秒內選不到節點就失敗
    connect_timeout_ms=3000,
    socket_timeout_ms=10000,
)
```

搭配 `MongoBatchUploader` 的重試機制，即使 Primary 發生故障轉移（通常 10-30 秒），緩衝佇列也能撐過這段時間，不會丟失資料。

---

## 9. 完整整合範例

```python
import asyncio
from datetime import datetime, timezone
from motor.motor_asyncio import AsyncIOMotorClient

from csp_lib.mongo import (
    MongoConfig,
    create_mongo_client,
    MongoBatchUploader,
    UploaderConfig,
)
from csp_lib.manager.data import DataUploadManager


async def main():
    # 1. 建立 MongoDB 連線
    mongo_config = MongoConfig(host="localhost", port=27017)
    mongo_client = create_mongo_client(mongo_config)
    db = mongo_client["ems"]

    # 2. 建立批次上傳器
    uploader_config = UploaderConfig(
        flush_interval=5,
        batch_size_threshold=100,
        max_queue_size=10000,
        max_retry_count=3,
    )
    uploader = MongoBatchUploader(db, config=uploader_config)
    uploader.start()

    # 3. 建立資料上傳管理器
    data_manager = DataUploadManager(uploader)

    # 4. 訂閱設備（假設 devices 已建立）
    for device in devices:
        data_manager.subscribe(
            device,
            collection_name="device_data",
            save_interval=5,
        )

    # 5. 設備運行中...資料自動上傳

    # 6. 關閉
    for device in devices:
        data_manager.unsubscribe(device)
    await uploader.stop()
```

---

## 10. 重點回顧

| 面向 | 設計決策 | 原因 |
|------|---------|------|
| 批次寫入 | Queue + 定時/閾值 flush | I/O 效率，避免逐筆寫入 |
| 背壓控制 | 固定大小 deque，滿時丟棄最舊 | 保護記憶體，不阻塞生產者 |
| 重試機制 | restore 到佇列前端，超過上限丟棄 | 平衡可靠性與系統穩定性 |
| 降頻儲存 | save_interval 參數 | 減少儲存量，按需調整 |
| 斷線記錄 | null 值文件保留結構 | 前端圖表正確顯示斷線區間 |
| 職責分離 | BatchQueue / MongoWriter / Uploader | 每個元件單一職責，容易測試 |

---

## 下篇預告

資料已經從設備流入 Redis（即時）和 MongoDB（持久），但還缺一環：**事件如何從設備傳遞到前端？** 下一篇我們將深入 csp_lib 的事件系統——從 `DeviceEventEmitter` 的 Queue 設計、到 `EventBridge` 的跨設備聚合、再到 WebSocket 推送，看看一個完整的「設備到瀏覽器」的即時訂閱管線是怎麼建起來的。

> **下一篇：** [Article 13 — 即時訂閱系統：從設備事件到前端推送](13-realtime-subscription.md)
