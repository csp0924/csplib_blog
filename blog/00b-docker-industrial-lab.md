# 用 Docker 搭建你的工業開發實驗室

> **從零打造工業控制協定框架：給軟體工程師的 CSP Library 實戰系列**
>
> Part 0 — 預備知識篇 | Article 00B
>
> [<<< 上一篇：你需要知道的 Python 非同步基礎](./00a-python-async-fundamentals.md) | [下一篇：網路基礎：TCP、序列通訊與封包的概念 >>>](./00c-networking-fundamentals.md)

---

## 目錄

1. [為什麼不直接連真實設備開發？](#為什麼不直接連真實設備開發)
2. [開發實驗室架構總覽](#開發實驗室架構總覽)
3. [Docker 與 Docker Compose 基礎回顧](#docker-與-docker-compose-基礎回顧)
4. [完整 docker-compose.yml 詳解](#完整-docker-composeyml-詳解)
5. [一鍵啟動與驗證](#一鍵啟動與驗證)
6. [常見問題排除](#常見問題排除)
7. [重點回顧](#重點回顧)
8. [下篇預告](#下篇預告)

---

## 為什麼不直接連真實設備開發？

如果你是做 Web 開發的，你的「開發環境」大概是：一台筆電、一個本地 PostgreSQL、一個 Redis，再加上 `npm run dev`。你隨時可以重啟、隨時可以清資料庫、出了 bug 最多網頁當掉。

工業開發完全不是這樣。

**設備昂貴**。一台功率調節系統（PCS）動輒數百萬台幣，一組鋰電池模組幾十萬。你不可能在每個開發者桌上放一台。

**設備危險**。PCS 內部有數百伏特的直流電壓、電池模組的短路能量可以瞬間熔化金屬。你的程式如果對設備發送了錯誤的控制指令，後果可能是設備損壞，甚至人員受傷。

**設備不方便**。案場通常在偏遠地區——屋頂、郊區變電站、離岸風場。你不能每次改一行程式碼就坐兩小時的車去案場測試。而且案場的設備是 24 小時運行的，你不能因為開發測試就把它停下來。

**所以我們需要模擬**。一個好的模擬開發環境應該：

- 能模擬真實設備的通訊行為（Modbus TCP 回應）
- 能提供完整的基礎設施（資料庫、快取、協調服務）
- 一個指令就能啟動或重置
- 在任何開發者的電腦上都能跑
- 不需要任何特殊硬體

Docker 正是最適合的工具。

---

## 開發實驗室架構總覽

我們的開發實驗室由四個服務組成，各自扮演不同的角色：

```
┌─────────────────────────────────────────────────────────────────┐
│                    Developer Machine (Host)                      │
│                                                                 │
│   ┌──────────────────────────────────────────────────────────┐  │
│   │                  Docker Network: csp-lab                 │  │
│   │                                                          │  │
│   │  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐   │  │
│   │  │   Modbus      │  │              │  │              │   │  │
│   │  │   Simulator   │  │    Redis 7   │  │  MongoDB 7   │   │  │
│   │  │              │  │              │  │              │   │  │
│   │  │  Port: 5020   │  │  Port: 6379  │  │  Port: 27017 │   │  │
│   │  │              │  │              │  │              │   │  │
│   │  │  模擬 PCS,    │  │  即時數據快取  │  │  時序資料儲存  │   │  │
│   │  │  BMS, 電表    │  │  告警狀態     │  │  歷史記錄     │   │  │
│   │  └──────────────┘  └──────────────┘  └──────────────┘   │  │
│   │                                                          │  │
│   │  ┌──────────────┐                                        │  │
│   │  │              │                                        │  │
│   │  │   etcd 3.5   │                                        │  │
│   │  │              │                                        │  │
│   │  │  Port: 2379  │                                        │  │
│   │  │              │                                        │  │
│   │  │  叢集協調     │                                        │  │
│   │  │  Leader 選舉  │                                        │  │
│   │  └──────────────┘                                        │  │
│   │                                                          │  │
│   └──────────────────────────────────────────────────────────┘  │
│                                                                 │
│   ┌──────────────┐                                              │
│   │  你的 Python  │  ← 從 Host 連入 Docker 服務                  │
│   │  開發程式碼    │    localhost:5020 / 6379 / 27017 / 2379      │
│   └──────────────┘                                              │
└─────────────────────────────────────────────────────────────────┘
```

每個服務的角色：

| 服務 | 角色 | 為什麼需要它 |
|------|------|-------------|
| **Modbus Simulator** | 模擬工業設備 | 提供 Modbus TCP 介面，模擬 PCS、BMS 的暫存器讀寫 |
| **Redis 7** | 即時資料快取 | 儲存設備的最新狀態、告警旗標，支援 Pub/Sub 事件通知 |
| **MongoDB 7** | 時序資料儲存 | 儲存設備歷史數據、告警記錄、操作日誌 |
| **etcd 3.5** | 叢集協調 | 多節點部署時的 Leader 選舉、分散式鎖、設定同步 |

你的 Python 程式碼跑在 Host 上（不在 Docker 裡），透過 `localhost` 的對應 port 連入各個服務。這樣你可以用慣用的 IDE、偵錯器、virtualenv，同時享有完整的基礎設施。

---

## Docker 與 Docker Compose 基礎回顧

如果你已經熟悉 Docker，可以跳過這一節。以下是快速回顧。

### 四個核心概念

**Image（映像檔）**：一個唯讀的應用程式打包。包含了執行環境、程式碼、設定檔。你可以把它想成一張光碟——它是靜態的，不會改變。

```bash
# 從 Docker Hub 下載 Redis 7 的映像檔
docker pull redis:7-alpine
```

**Container（容器）**：從 Image 啟動的一個執行實例。就像從光碟安裝到電腦後跑起來的程式。每個 Container 有自己獨立的檔案系統、網路、行程空間。

```bash
# 從 Redis 映像檔啟動一個容器
docker run -d --name my-redis -p 6379:6379 redis:7-alpine
```

**Volume（卷宗）**：持久化儲存。Container 被刪除時，裡面的資料會消失。如果你想保留資料（例如 MongoDB 的資料庫檔案），就需要掛載 Volume。

```bash
# 建立一個 Volume 並掛載到容器
docker run -d --name my-mongo \
  -v mongo-data:/data/db \
  mongo:7
```

**Network（網路）**：Docker 容器之間的通訊管道。同一個 Network 裡的容器可以用服務名稱互相連線（例如 `redis:6379`），不需要知道 IP 位址。

```bash
# 建立一個自訂網路
docker network create csp-lab
```

### Docker Compose：一鍵管理多個容器

手動逐一啟動四個容器很繁瑣。Docker Compose 讓你用一個 YAML 檔案定義所有服務，一個指令全部啟動：

```bash
# 啟動所有服務（背景執行）
docker compose up -d

# 查看所有服務狀態
docker compose ps

# 查看某個服務的日誌
docker compose logs -f redis

# 停止並移除所有服務
docker compose down

# 停止並移除所有服務 + 刪除 Volume（完全重置）
docker compose down -v
```

---

## 完整 docker-compose.yml 詳解

以下是我們開發實驗室的完整設定。建議你在專案根目錄建立一個 `docker-compose.dev.yml`：

```yaml
# docker-compose.dev.yml
# CSP Library 開發實驗室
# 用法: docker compose -f docker-compose.dev.yml up -d

# ============================================================
# 版本說明：
#   - Docker Compose V2 不再需要 version 欄位
#   - 所有服務共用 csp-lab 網路
#   - 所有有狀態服務掛載 named volume 持久化
# ============================================================

services:

  # ----------------------------------------------------------
  # Modbus TCP 模擬器
  #
  # 使用 oitc/modbus-server 模擬器，提供標準的 Modbus TCP 介面。
  # 預設建立 Holding Registers (FC03/FC06/FC16) 供讀寫。
  #
  # 你的 Python 程式碼透過 localhost:5020 連入，
  # 就像連接真實的 PCS 或 BMS 一樣。
  # ----------------------------------------------------------
  modbus-simulator:
    image: oitc/modbus-server:latest
    container_name: csp-modbus-sim
    ports:
      - "5020:502"           # Host 5020 → Container 502 (Modbus TCP 標準 port)
    environment:
      - SERVER_HOST=0.0.0.0  # 監聽所有介面
    networks:
      - csp-lab
    healthcheck:
      # 檢查 port 502 是否有在監聽
      test: ["CMD-SHELL", "nc -z localhost 502 || exit 1"]
      interval: 10s
      timeout: 5s
      retries: 3
      start_period: 5s
    restart: unless-stopped

  # ----------------------------------------------------------
  # Redis 7
  #
  # 用途：
  #   1. 即時設備數據快取（最後一筆讀數）
  #   2. 告警狀態旗標
  #   3. Pub/Sub 事件通知（值變更、告警觸發）
  #   4. 分散式鎖（多行程寫入保護）
  #
  # 使用 Alpine 版本以減少映像大小（~30MB vs ~130MB）。
  # 設定 maxmemory 為 256MB，超過時使用 allkeys-lru 淘汰策略。
  # ----------------------------------------------------------
  redis:
    image: redis:7-alpine
    container_name: csp-redis
    ports:
      - "6379:6379"
    command: >
      redis-server
      --maxmemory 256mb
      --maxmemory-policy allkeys-lru
      --save 60 1000
      --loglevel warning
    volumes:
      - redis-data:/data     # 持久化 RDB 快照
    networks:
      - csp-lab
    healthcheck:
      test: ["CMD", "redis-cli", "ping"]
      interval: 10s
      timeout: 5s
      retries: 3
      start_period: 5s
    restart: unless-stopped

  # ----------------------------------------------------------
  # MongoDB 7
  #
  # 用途：
  #   1. 設備歷史數據儲存（時序集合）
  #   2. 告警記錄持久化
  #   3. 設備設定與中繼資料
  #   4. 操作日誌（審計追蹤）
  #
  # 開發環境不啟用認證（方便偵錯），生產環境務必設定帳密。
  # 預設建立 csp_dev 資料庫。
  # ----------------------------------------------------------
  mongodb:
    image: mongo:7
    container_name: csp-mongodb
    ports:
      - "27017:27017"
    environment:
      # 開發環境不設密碼，生產環境請務必設定
      - MONGO_INITDB_DATABASE=csp_dev
    volumes:
      - mongo-data:/data/db           # 資料庫檔案
      - mongo-config:/data/configdb   # 設定檔案
    networks:
      - csp-lab
    healthcheck:
      test: ["CMD", "mongosh", "--eval", "db.adminCommand('ping')"]
      interval: 10s
      timeout: 5s
      retries: 3
      start_period: 10s
    restart: unless-stopped

  # ----------------------------------------------------------
  # etcd 3.5
  #
  # 用途：
  #   1. 多節點部署的 Leader 選舉
  #   2. 分散式設定管理（設備 map、策略參數）
  #   3. 服務發現（哪些節點在線）
  #   4. 分散式鎖（跨節點互斥操作）
  #
  # 開發環境用單節點模式。生產環境建議至少 3 節點。
  # ----------------------------------------------------------
  etcd:
    image: quay.io/coreos/etcd:v3.5.17
    container_name: csp-etcd
    ports:
      - "2379:2379"          # Client API
      - "2380:2380"          # Peer（叢集間通訊，開發用不到，但預留）
    environment:
      - ETCD_NAME=csp-dev-node
      - ETCD_DATA_DIR=/etcd-data
      - ETCD_LISTEN_CLIENT_URLS=http://0.0.0.0:2379
      - ETCD_ADVERTISE_CLIENT_URLS=http://etcd:2379
      - ETCD_LISTEN_PEER_URLS=http://0.0.0.0:2380
      - ETCD_INITIAL_ADVERTISE_PEER_URLS=http://etcd:2380
      - ETCD_INITIAL_CLUSTER=csp-dev-node=http://etcd:2380
      - ETCD_INITIAL_CLUSTER_TOKEN=csp-dev-token
      - ETCD_INITIAL_CLUSTER_STATE=new
      # 開發環境不啟用認證
      - ETCD_AUTH_TOKEN=simple
    volumes:
      - etcd-data:/etcd-data
    networks:
      - csp-lab
    healthcheck:
      test: ["CMD", "etcdctl", "endpoint", "health", "--endpoints=http://localhost:2379"]
      interval: 10s
      timeout: 5s
      retries: 3
      start_period: 5s
    restart: unless-stopped

# ----------------------------------------------------------
# Named Volumes
# Docker 會自動管理這些卷宗的存放位置
# 使用 `docker compose down -v` 可以刪除所有卷宗（完全重置）
# ----------------------------------------------------------
volumes:
  redis-data:
    driver: local
  mongo-data:
    driver: local
  mongo-config:
    driver: local
  etcd-data:
    driver: local

# ----------------------------------------------------------
# Network
# 所有服務共用一個 bridge 網路，容器間可用服務名稱通訊
# ----------------------------------------------------------
networks:
  csp-lab:
    driver: bridge
```

### 設定要點說明

**為什麼 Modbus 用 port 5020 而不是 502？**

Modbus TCP 的標準 port 是 502，但在 Linux/macOS 上，1024 以下的 port 需要 root 權限。用 5020 避免了這個問題，你的 Python 程式碼連到 `localhost:5020` 就好，Docker 會把流量轉到容器內的 502。

**Redis 的 maxmemory 設定**

工業數據的特點是「最新的最重要」。`allkeys-lru` 策略表示：記憶體滿了就淘汰最久沒被存取的 key。這很適合即時數據快取——你只需要設備的最新狀態，30 秒前的數據可以被淘汰。

**MongoDB 不設密碼**

開發環境的便利性優先。但請注意：**絕對不要把這個 compose 檔用在生產環境**。生產環境必須設定 `MONGO_INITDB_ROOT_USERNAME` 和 `MONGO_INITDB_ROOT_PASSWORD`。

**etcd 單節點模式**

etcd 設計上是 3 或 5 節點的叢集，但開發時一個節點就夠了。我們透過 `ETCD_INITIAL_CLUSTER` 只定義一個節點來啟用單節點模式。

---

## 一鍵啟動與驗證

### 啟動所有服務

```bash
# 在專案根目錄執行
docker compose -f docker-compose.dev.yml up -d
```

第一次啟動會下載映像檔，可能需要幾分鐘。後續啟動只需要幾秒。

```
[+] Running 5/5
 ✔ Network csp-lab           Created
 ✔ Container csp-modbus-sim  Started
 ✔ Container csp-redis       Started
 ✔ Container csp-mongodb     Started
 ✔ Container csp-etcd        Started
```

### 檢查服務狀態

```bash
docker compose -f docker-compose.dev.yml ps
```

所有服務的 STATUS 應該顯示 `Up` 和 `(healthy)`：

```
NAME              IMAGE                           STATUS                   PORTS
csp-etcd          quay.io/coreos/etcd:v3.5.17     Up 30s (healthy)         0.0.0.0:2379-2380->2379-2380/tcp
csp-modbus-sim    oitc/modbus-server:latest        Up 30s (healthy)         0.0.0.0:5020->502/tcp
csp-mongodb       mongo:7                          Up 30s (healthy)         0.0.0.0:27017->27017/tcp
csp-redis         redis:7-alpine                   Up 30s (healthy)         0.0.0.0:6379->6379/tcp
```

### 驗證各個服務

#### 驗證 Modbus Simulator

用 Python 快速測試 Modbus 連線：

```python
"""test_modbus_connection.py"""
import asyncio
from pymodbus.client import AsyncModbusTcpClient


async def main():
    client = AsyncModbusTcpClient("localhost", port=5020)
    await client.connect()

    if client.connected:
        print("Modbus 連線成功！")

        # 讀取 Holding Registers（位址 0，讀 10 個）
        result = await client.read_holding_registers(address=0, count=10, slave=1)
        if not result.isError():
            print(f"暫存器 0-9: {result.registers}")
        else:
            print(f"讀取錯誤: {result}")

        # 寫入一個值
        await client.write_register(address=0, value=12345, slave=1)
        print("寫入位址 0 = 12345")

        # 驗證寫入
        result = await client.read_holding_registers(address=0, count=1, slave=1)
        print(f"讀回位址 0: {result.registers[0]}")

        client.close()
    else:
        print("Modbus 連線失敗")


asyncio.run(main())
```

預期輸出：

```
Modbus 連線成功！
暫存器 0-9: [0, 0, 0, 0, 0, 0, 0, 0, 0, 0]
寫入位址 0 = 12345
讀回位址 0: 12345
```

#### 驗證 Redis

```bash
# 用 redis-cli 連入
docker exec -it csp-redis redis-cli

# 在 redis-cli 裡測試
127.0.0.1:6379> SET device:pcs_01:power 50.5
OK
127.0.0.1:6379> GET device:pcs_01:power
"50.5"
127.0.0.1:6379> PING
PONG
127.0.0.1:6379> exit
```

或者用 Python：

```python
"""test_redis_connection.py"""
import redis

r = redis.Redis(host="localhost", port=6379, decode_responses=True)
r.set("device:pcs_01:power", "50.5")
value = r.get("device:pcs_01:power")
print(f"Redis 測試: power = {value}")
# Redis 測試: power = 50.5
```

#### 驗證 MongoDB

```bash
# 用 mongosh 連入
docker exec -it csp-mongodb mongosh csp_dev

# 在 mongosh 裡測試
csp_dev> db.device_readings.insertOne({
...   device_id: "PCS_01",
...   timestamp: new Date(),
...   power: 50.5,
...   soc: 85.2
... })
{ acknowledged: true, insertedId: ObjectId("...") }

csp_dev> db.device_readings.find()
[
  {
    _id: ObjectId("..."),
    device_id: 'PCS_01',
    timestamp: ISODate("..."),
    power: 50.5,
    soc: 85.2
  }
]

csp_dev> exit
```

或者用 Python：

```python
"""test_mongo_connection.py"""
from pymongo import MongoClient
from datetime import datetime, timezone

client = MongoClient("mongodb://localhost:27017/")
db = client["csp_dev"]

db.device_readings.insert_one({
    "device_id": "PCS_01",
    "timestamp": datetime.now(timezone.utc),
    "power": 50.5,
    "soc": 85.2,
})

doc = db.device_readings.find_one({"device_id": "PCS_01"})
print(f"MongoDB 測試: {doc['device_id']} power={doc['power']}")
# MongoDB 測試: PCS_01 power=50.5
```

#### 驗證 etcd

```bash
# 用 etcdctl 測試
docker exec -it csp-etcd etcdctl put /csp/config/poll_interval "1000"
# OK

docker exec -it csp-etcd etcdctl get /csp/config/poll_interval
# /csp/config/poll_interval
# 1000

# 檢查叢集健康狀態
docker exec -it csp-etcd etcdctl endpoint health
# 127.0.0.1:2379 is healthy: successfully committed proposal: took = 1.234ms
```

### 一鍵驗證腳本

你可以把以上驗證整合成一個腳本：

```python
"""verify_lab.py - 驗證所有開發實驗室服務"""
import asyncio
import sys


async def check_modbus() -> bool:
    try:
        from pymodbus.client import AsyncModbusTcpClient
        client = AsyncModbusTcpClient("localhost", port=5020)
        await client.connect()
        if client.connected:
            result = await client.read_holding_registers(address=0, count=1, slave=1)
            client.close()
            return not result.isError()
        return False
    except Exception as e:
        print(f"  Modbus 錯誤: {e}")
        return False


def check_redis() -> bool:
    try:
        import redis
        r = redis.Redis(host="localhost", port=6379, socket_timeout=3)
        return r.ping()
    except Exception as e:
        print(f"  Redis 錯誤: {e}")
        return False


def check_mongodb() -> bool:
    try:
        from pymongo import MongoClient
        client = MongoClient("mongodb://localhost:27017/", serverSelectionTimeoutMS=3000)
        client.admin.command("ping")
        return True
    except Exception as e:
        print(f"  MongoDB 錯誤: {e}")
        return False


def check_etcd() -> bool:
    try:
        import httpx
        resp = httpx.get("http://localhost:2379/health", timeout=3)
        return resp.status_code == 200
    except Exception as e:
        print(f"  etcd 錯誤: {e}")
        return False


async def main():
    print("CSP 開發實驗室 - 服務驗證\n")

    checks = {
        "Modbus Simulator (port 5020)": await check_modbus(),
        "Redis 7          (port 6379)": check_redis(),
        "MongoDB 7        (port 27017)": check_mongodb(),
        "etcd 3.5         (port 2379)": check_etcd(),
    }

    all_ok = True
    for name, ok in checks.items():
        status = "OK" if ok else "FAIL"
        symbol = "[+]" if ok else "[-]"
        print(f"  {symbol} {name} ... {status}")
        if not ok:
            all_ok = False

    print()
    if all_ok:
        print("所有服務正常運作！可以開始開發了。")
    else:
        print("部分服務異常，請檢查 docker compose logs。")
        sys.exit(1)


asyncio.run(main())
```

---

## 常見問題排除

### Port 衝突

**症狀**：`docker compose up` 時出現 `Bind for 0.0.0.0:6379 failed: port is already allocated`。

**原因**：你的 Host 上已經有其他程式佔用了該 port。常見的情況是你本機已安裝了 Redis 或 MongoDB。

**解法**：

方法一：停止佔用 port 的程式

```bash
# Linux/macOS: 查看誰佔用了 6379
lsof -i :6379

# Windows: 查看誰佔用了 6379
netstat -ano | findstr :6379
```

方法二：修改 docker-compose.yml 的 port mapping

```yaml
redis:
  ports:
    - "16379:6379"  # 改用 16379
```

然後你的 Python 程式碼也要改連 `localhost:16379`。

### Volume 權限問題（Linux）

**症狀**：MongoDB 或 etcd 啟動失敗，日誌顯示 `Permission denied`。

**原因**：容器內的行程以非 root 使用者運行，但 volume 目錄的權限不允許寫入。

**解法**：

```bash
# 刪除舊 volume 重新建立
docker compose -f docker-compose.dev.yml down -v
docker compose -f docker-compose.dev.yml up -d
```

如果問題持續：

```bash
# 手動調整權限（MongoDB 用 uid 999）
docker run --rm -v mongo-data:/data/db busybox chown -R 999:999 /data/db
```

### Apple Silicon（M1/M2/M3）注意事項

**症狀**：某些映像檔拉取時警告 `WARNING: The requested image's platform (linux/amd64) does not match the detected host platform (linux/arm64/v8)`。

**原因**：部分映像檔尚未提供 ARM64 原生版本，Docker 需要透過 Rosetta 2 模擬執行。

**解法**：

大部分官方映像（Redis、MongoDB、etcd）都已支援 ARM64，不需要特殊處理。如果遇到 Modbus 模擬器不支援 ARM64 的情況：

```yaml
modbus-simulator:
  image: oitc/modbus-server:latest
  platform: linux/amd64  # 強制使用 x86 模擬
```

效能會稍微降低，但對開發環境而言完全可以接受。

### 容器啟動後立刻退出

**症狀**：`docker compose ps` 顯示某個服務狀態為 `Exited`。

**解法**：

```bash
# 查看該服務的日誌
docker compose -f docker-compose.dev.yml logs modbus-simulator

# 常見原因：
# 1. 映像檔不對：檢查 image 名稱和 tag
# 2. 設定錯誤：檢查 environment 變數
# 3. 資源不足：Docker Desktop 的記憶體設定至少給 4GB
```

### 服務之間無法通訊

**症狀**：從一個容器 ping 另一個容器失敗。

**解法**：確認所有服務都在同一個 network 裡。

```bash
# 檢查網路
docker network inspect csp-lab

# 應該能看到所有四個容器都在 Containers 清單裡
```

### 如何完全重置環境

當你想從頭開始（清除所有資料）：

```bash
# 停止服務 + 刪除容器 + 刪除 volume + 刪除網路
docker compose -f docker-compose.dev.yml down -v

# 重新啟動
docker compose -f docker-compose.dev.yml up -d
```

這是 Docker 開發環境最大的優勢之一：**隨時可以回到乾淨狀態**。試想如果你在真實設備上操作，要「重置環境」可能意味著重刷韌體、重新校準感測器、甚至重新配置整個電力系統。

---

## 重點回顧

經過這篇文章，讓我們整理幾個核心重點：

1. **不要用真實設備開發**。設備昂貴、危險、不方便。Docker 模擬環境讓你在筆電上就能進行完整的工業通訊開發。

2. **開發實驗室包含四個服務**：Modbus 模擬器（模擬設備）、Redis（即時快取）、MongoDB（歷史儲存）、etcd（叢集協調）。各自對應工業系統的不同需求。

3. **Docker Compose 是一鍵管理工具**。一個 YAML 檔案定義所有服務，`docker compose up -d` 啟動、`docker compose down -v` 完全重置。

4. **每個服務都有 healthcheck**。不要只看容器有沒有跑起來，healthcheck 會驗證服務是否真正可用。

5. **Port mapping 讓 Host 能連入容器**。你的 Python 程式碼連 `localhost:5020` 就等於連 Modbus 模擬器的 502 port。

6. **Volume 確保資料不會因容器重啟而消失**。但開發時你可以用 `down -v` 隨時清除重來。

7. **Apple Silicon 使用者不用擔心**。主流映像都已支援 ARM64，少數不支援的可以透過 `platform: linux/amd64` 模擬執行。

如果你只記住一件事，那就是：**`docker compose up -d` 是你每天開發的第一條指令**。它幫你在幾秒鐘內啟動一個完整的工業模擬環境，讓你專注在程式碼上，而不是基礎設施。

---

## 下篇預告

開發環境準備好了，但在真正開始寫工業通訊程式之前，我們還需要理解一些網路基礎知識。工業協定的通訊不是 HTTP——它可能是 TCP 的長連線、RS-485 的串列通訊、甚至是原始的二進制封包。

在下一篇文章中，我們將探討：

- TCP 連線的三次握手與持久連線
- RS-485 串列通訊的基本概念
- 封包結構：PDU、ADU 的差異
- 為什麼 Modbus RTU 和 Modbus TCP 的封包格式不同
- 常見的通訊問題：半雙工衝突、封包黏連、超時重試

理解這些基礎後，你在後續閱讀 Modbus 相關文章時，就不會對「ADU」、「CRC 校驗」、「串列匯流排仲裁」這些詞彙感到陌生。

[下一篇：網路基礎：TCP、序列通訊與封包的概念 >>>](./00c-networking-fundamentals.md)

---

> 本文為「從零打造工業控制協定框架：給軟體工程師的 CSP Library 實戰系列」的 Part 0 預備知識篇 Article 00B。
> 完整系列文章請參閱系列目錄。
