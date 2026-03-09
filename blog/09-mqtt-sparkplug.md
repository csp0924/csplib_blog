# MQTT 與 Sparkplug B：工業物聯網的雲端橋樑

> **Part 2 — 協定轉接層 | Article 09**
> 系列：從零打造工業控制協定框架：給軟體工程師的 CSP Library 實戰系列

---

## 前言

上一篇我們聊了 IEC 61850——一個為電力系統量身打造的重量級標準。它有完整的語義模型、毫秒級的保護通訊，但也有著陡峭的學習曲線和有限的開源生態。

這一篇，我們把視角轉向後端工程師最熟悉的領域——**MQTT**。如果你寫過微服務，你一定用過（或至少聽過）MQTT 或類似的 pub/sub 訊息系統。MQTT 輕量、簡單、無所不在。但當你把它用在工業控制場景時，你會發現「太簡單」也是一種問題。

**Sparkplug B** 就是為了解決這個問題而生的——它在 MQTT 之上建立了一套工業級的語義規範，讓原本鬆散的 MQTT 訊息變成結構化的設備資料流。

---

## 1. MQTT 基礎：後端工程師的熟悉領域

MQTT（Message Queuing Telemetry Transport）是一個輕量級的 publish/subscribe 訊息協定。如果你用過 Redis Pub/Sub、RabbitMQ 或 Kafka，MQTT 的概念對你來說完全不陌生。

### 核心概念

```
┌──────────┐                         ┌──────────┐
│ Publisher │ ── publish ──→          │Subscriber│
│ (設備)    │               ┌──────┐  │ (雲端)    │
│          │               │Broker│  │          │
│          │               │(中介) │  │          │
└──────────┘               └──┬───┘  └──────────┘
                              │
                        Topic 路由
```

| 概念 | 說明 | 類比 |
|------|------|------|
| Broker | 訊息中介伺服器 | Redis Server / RabbitMQ |
| Topic | 訊息主題（層級式路徑）| Channel / Queue |
| Publish | 發布訊息 | `PUBLISH channel message` |
| Subscribe | 訂閱主題 | `SUBSCRIBE channel` |
| QoS | 服務品質等級（0/1/2）| At-most/At-least/Exactly-once |
| Retain | 保留最後一則訊息 | 類似 cache |
| Last Will | 連線中斷時自動發布的訊息 | Health check 的被動版 |

### 為什麼 MQTT 適合 IoT

1. **極小的通訊開銷**：最小 header 只有 2 bytes（對比 HTTP 的數百 bytes header）
2. **支援不穩定網路**：內建重連、QoS、session 機制
3. **雙向通訊**：設備可以發布資料，也可以訂閱控制命令
4. **Topic 萬用字元**：`factory/+/temperature` 訂閱所有設備的溫度

### Python MQTT 範例

```python
import asyncio
import json

import aiomqtt  # pip install aiomqtt

async def main():
    async with aiomqtt.Client("broker.example.com") as client:
        # 發布設備資料
        payload = json.dumps({
            "device_id": "pcs_01",
            "active_power": 50.0,
            "soc": 78.5,
            "timestamp": "2026-03-10T08:30:00Z",
        })
        await client.publish("site/devices/pcs_01/data", payload)

        # 訂閱控制命令
        await client.subscribe("site/devices/pcs_01/command")
        async for message in client.messages:
            cmd = json.loads(message.payload)
            print(f"收到命令: {cmd}")

asyncio.run(main())
```

看起來很簡單對吧？對於後端工程師來說，這跟寫任何其他的 pub/sub 程式碼沒什麼差別。

但問題就在這個「沒什麼差別」。

---

## 2. 為什麼「純 MQTT」不夠用於工業場景

當你把 MQTT 用在正式的工業控制系統中，你會很快遇到以下問題：

### 2.1 沒有標準的 Payload 格式

```python
# 團隊 A 的做法
{"device": "pcs_01", "power": 50.0, "unit": "kW"}

# 團隊 B 的做法
{"id": "PCS-001", "active_power_kw": 50, "timestamp": 1678234567}

# 團隊 C 的做法
{"dev_id": "pcs01", "values": [{"name": "P", "val": 50000, "scale": 0.001}]}

# 第三方廠商的做法
"PCS01,P=50.0,Q=10.5,SOC=78.5"  # CSV 字串...
```

每個人都用 MQTT，但 payload 格式完全不統一。你的上層應用需要為每個來源寫不同的 parser。

### 2.2 沒有設備生命週期管理

MQTT 的 Last Will 機制可以偵測連線中斷，但它缺乏完整的設備生命週期概念：

- 設備上線時，**應該**發布什麼？
- 設備的點位清單在哪裡定義？
- 設備的元資料（型號、韌體版本）怎麼傳遞？
- 如何區分「設備正常下線」和「設備異常斷線」？

### 2.3 沒有狀態管理

```python
# 情境：你的 Dashboard 服務重啟了
# 問題：所有設備的最新狀態在哪裡？

# 方案 1：用 MQTT Retain → 但只保留最後一則訊息，缺少完整快照
# 方案 2：主動查詢所有設備 → 但 MQTT 是 pub/sub，不是 request/response
# 方案 3：另外維護狀態資料庫 → 那 MQTT 的角色是什麼？
```

### 2.4 沒有指標 (Metric) 型別系統

MQTT payload 是 byte array，沒有內建的型別系統。你得自己決定：

- `50` 是整數還是浮點數？
- `true` 是布林值還是字串？
- 時間戳記用什麼格式？
- 列表型的資料（如多組電池電壓）怎麼表示？

這些問題在小型專案中可以「約定好就行」，但在跨團隊、跨廠商的大型系統中，缺乏標準就意味著持續的整合成本。

---

## 3. Sparkplug B 規範

Sparkplug B 是由 Eclipse Foundation 維護的開放規範（現為 Eclipse Sparkplug），專門解決「MQTT 用於工業 IoT」的標準化問題。它在 MQTT 3.1.1 / 5.0 之上定義了：

1. **標準化的 Topic 命名空間**
2. **設備生命週期（Birth/Death）**
3. **結構化的指標型別**
4. **狀態管理機制**

### 3.1 Topic 命名空間

Sparkplug B 定義了嚴格的 Topic 結構：

```
spBv1.0/{group_id}/{message_type}/{edge_node_id}/{device_id}
```

| 欄位 | 說明 | 範例 |
|------|------|------|
| `spBv1.0` | 協定版本前綴 | 固定值 |
| `group_id` | 群組 ID（案場/廠區）| `taoyuan_ess` |
| `message_type` | 訊息類型 | `NBIRTH`, `NDEATH`, `DDATA` 等 |
| `edge_node_id` | 邊緣節點 ID | `gateway_01` |
| `device_id` | 設備 ID（可選）| `pcs_01` |

### 訊息類型

| 類型 | 全名 | 方向 | 說明 |
|------|------|------|------|
| `NBIRTH` | Node Birth | Edge → Broker | 邊緣節點上線宣告 |
| `NDEATH` | Node Death | Edge → Broker | 邊緣節點離線宣告 |
| `DBIRTH` | Device Birth | Edge → Broker | 設備上線宣告 |
| `DDEATH` | Device Death | Edge → Broker | 設備離線宣告 |
| `NDATA` | Node Data | Edge → Broker | 邊緣節點資料更新 |
| `DDATA` | Device Data | Edge → Broker | 設備資料更新 |
| `NCMD` | Node Command | App → Edge | 對邊緣節點下命令 |
| `DCMD` | Device Command | App → Edge | 對設備下命令 |
| `STATE` | State | App → Broker | 應用程式狀態 |

### Topic 範例

```
# PCS 設備上線
spBv1.0/taoyuan_ess/DBIRTH/gateway_01/pcs_01

# PCS 資料更新
spBv1.0/taoyuan_ess/DDATA/gateway_01/pcs_01

# 對 PCS 下達控制命令
spBv1.0/taoyuan_ess/DCMD/gateway_01/pcs_01

# 邊緣節點意外斷線（由 Broker 的 Last Will 自動發布）
spBv1.0/taoyuan_ess/NDEATH/gateway_01
```

### 3.2 Birth/Death 證書

Sparkplug B 的 Birth Certificate 是設備上線時發布的完整自描述訊息：

```python
# DBIRTH payload（概念性表示）
dbirth_payload = {
    "timestamp": 1678234567000,  # 毫秒時間戳
    "metrics": [
        {
            "name": "active_power",
            "alias": 1,              # 後續 DDATA 可用 alias 代替完整名稱
            "datatype": "Float",
            "value": 0.0,
            "properties": {
                "unit": "kW",
                "description": "Active Power Output",
            },
        },
        {
            "name": "reactive_power",
            "alias": 2,
            "datatype": "Float",
            "value": 0.0,
            "properties": {
                "unit": "kVar",
            },
        },
        {
            "name": "soc",
            "alias": 3,
            "datatype": "Float",
            "value": 50.0,
            "properties": {
                "unit": "%",
                "min": 0.0,
                "max": 100.0,
            },
        },
        {
            "name": "operating_mode",
            "alias": 4,
            "datatype": "UInt16",
            "value": 0,
            "properties": {
                "value_map": {0: "STOP", 1: "RUN", 2: "FAULT"},
            },
        },
    ],
    "seq": 0,
}
```

DBIRTH 的精髓在於：
1. **自描述**：收到 DBIRTH 的應用程式不需要預先知道設備有哪些點位
2. **Alias 機制**：後續的 DDATA 只需傳 alias（數字），節省頻寬
3. **初始值**：所有指標的初始值都包含在 Birth 中

Death Certificate 則簡單得多——它通常設定為 MQTT 的 Last Will 訊息，當邊緣節點意外斷線時由 Broker 自動發布：

```python
# NDEATH — 透過 MQTT Last Will 機制
ndeath_payload = {
    "timestamp": 1678234567000,
    "metrics": [
        {
            "name": "bdSeq",     # Birth/Death 序號
            "datatype": "UInt64",
            "value": 42,
        }
    ],
}
```

### 3.3 指標型別與模板

Sparkplug B 支援完整的型別系統：

| 型別 | 說明 |
|------|------|
| `Int8` / `Int16` / `Int32` / `Int64` | 有號整數 |
| `UInt8` / `UInt16` / `UInt32` / `UInt64` | 無號整數 |
| `Float` / `Double` | 浮點數 |
| `Boolean` | 布林值 |
| `String` | 字串 |
| `DateTime` | 日期時間 |
| `Bytes` | 位元組陣列 |
| `Template` | 模板（結構化型別）|

**Template** 是特別值得注意的功能。它允許你定義結構化的複合型別：

```python
# Sparkplug B Template 定義
battery_module_template = {
    "name": "BatteryModule",
    "version": "1.0",
    "metrics": [
        {"name": "voltage", "datatype": "Float", "value": 0.0},
        {"name": "current", "datatype": "Float", "value": 0.0},
        {"name": "temperature", "datatype": "Float", "value": 25.0},
        {"name": "soh", "datatype": "Float", "value": 100.0},
    ],
}

# 使用 Template 定義多個電池模組
# 每個模組都遵循 BatteryModule 的結構
# metrics: [
#     {"name": "modules/1", "datatype": "Template", "template_ref": "BatteryModule"},
#     {"name": "modules/2", "datatype": "Template", "template_ref": "BatteryModule"},
#     ...
# ]
```

---

## 4. 與 csp_lib 設備模型的對比

讓我們把 Sparkplug B 的概念與 csp_lib 的設備模型做一個系統性的對比。

### 4.1 NBIRTH ≈ device.start() + register()

當 csp_lib 的設備啟動時，`AsyncModbusDevice` 會經歷一連串的初始化步驟：建立連線、開始讀取循環、發出 `connected` 事件。這跟 Sparkplug B 的 NBIRTH/DBIRTH 在概念上是對應的。

```python
# csp_lib 的事件系統天然映射 Sparkplug B 生命週期
from csp_lib.equipment.device.events import (
    EVENT_CONNECTED,
    EVENT_DISCONNECTED,
    EVENT_VALUE_CHANGE,
    ConnectedPayload,
    DisconnectPayload,
    ValueChangePayload,
)

# 設備連線事件 ≈ Sparkplug B DBIRTH
async def on_connected(payload: ConnectedPayload):
    """設備連線成功 — 類似 Sparkplug B 的 DBIRTH"""
    print(f"設備 {payload.device_id} 上線 at {payload.timestamp}")
    # 在 Sparkplug B 中，這裡會發布 DBIRTH 訊息
    # 包含所有 metrics 的定義和初始值

# 設備斷線事件 ≈ Sparkplug B DDEATH
async def on_disconnected(payload: DisconnectPayload):
    """設備斷線 — 類似 Sparkplug B 的 DDEATH"""
    print(f"設備 {payload.device_id} 離線: {payload.reason}")
    # 在 Sparkplug B 中，這裡會發布 DDEATH 訊息

# 註冊事件處理器
device.on(EVENT_CONNECTED, on_connected)       # ≈ DBIRTH
device.on(EVENT_DISCONNECTED, on_disconnected) # ≈ DDEATH
```

### 4.2 NDEATH ≈ device.stop() + unregister()

Sparkplug B 的 NDEATH 通常是透過 MQTT 的 Last Will 機制實現的——當邊緣節點意外斷線時，Broker 自動發布 NDEATH。csp_lib 的 `DisconnectPayload` 提供了類似的語義，包含斷線原因和連續失敗次數：

```python
# csp_lib 的 DisconnectPayload 包含豐富的上下文
# 這些資訊可以映射到 Sparkplug B 的 NDEATH 指標

async def handle_disconnect(payload: DisconnectPayload):
    device_id = payload.device_id
    reason = payload.reason
    failures = payload.consecutive_failures

    # 根據斷線原因決定 Sparkplug B 的處理方式
    if failures > 5:
        # 持續失敗 — 設備可能有硬體問題
        # 發布 DDEATH + 告警
        pass
    else:
        # 暫時性斷線 — 可能只是網路抖動
        # 等待自動重連
        pass
```

### 4.3 DDATA ≈ device.latest_values / value_change 事件

Sparkplug B 的 DDATA 是設備的即時資料更新。在 csp_lib 中，這對應到兩個機制：

```python
# 方式 1：Pull 模式 — 直接讀取最新值
# 類似於 Sparkplug B 中定期發布 DDATA
latest = device.latest_values
# → {"active_power": 50.0, "soc": 78.5, "voltage": 380.2, ...}

# 方式 2：Push 模式 — 值變化事件
# 類似於 Sparkplug B 中只在值變化時發布 DDATA (Report by Exception)
async def on_value_change(payload: ValueChangePayload):
    """值變化事件 — 類似 Sparkplug B 的 DDATA (RBE 模式)"""
    print(
        f"設備 {payload.device_id}: "
        f"{payload.point_name} 從 {payload.old_value} → {payload.new_value}"
    )
    # 在 Sparkplug B 中，這裡只需發送變化的 metric
    # 使用 alias 而非完整名稱，節省頻寬

device.on(EVENT_VALUE_CHANGE, on_value_change)
```

### 概念對應表

| Sparkplug B | csp_lib | 說明 |
|-------------|---------|------|
| Edge Node | `SimulationServer` / Gateway 程式 | 邊緣節點 |
| Device | `AsyncModbusDevice` | 設備抽象 |
| NBIRTH | Gateway 啟動 + 註冊所有設備 | 節點上線 |
| DBIRTH | `EVENT_CONNECTED` + `EquipmentTemplate` | 設備上線 + 點位定義 |
| NDEATH | Gateway 斷線（Last Will）| 節點離線 |
| DDEATH | `EVENT_DISCONNECTED` | 設備離線 |
| DDATA | `EVENT_VALUE_CHANGE` / `latest_values` | 資料更新 |
| DCMD | `device.write()` | 設備控制 |
| Metric | `ReadPoint` / `WritePoint` | 資料點位 |
| Template | `EquipmentTemplate` | 設備模型定義 |
| Alias | `ReadPoint.name`（字串 alias）| 點位識別 |

---

## 5. 何時使用 MQTT / Sparkplug B

### 5.1 雲端連接

當你需要將案場資料上傳到雲端平台時，MQTT 是最自然的選擇：

```
案場 (edge)                     雲端 (cloud)
┌─────────────┐                ┌─────────────────┐
│  csp_lib    │   MQTT/TLS    │  Cloud Platform  │
│  Gateway    │ ─────────────→│  (AWS IoT Core,  │
│             │               │   Azure IoT Hub, │
│  PCS ─┐    │               │   GCP IoT...)    │
│  電表 ─┤    │               │                  │
│  逆變器┘    │               │  Dashboard       │
└─────────────┘               │  Analytics       │
                               │  ML Pipeline     │
                               └─────────────────┘
```

MQTT 在這個場景的優勢：
- **穿透防火牆**：MQTT 走 TCP 443（TLS），不需要開放額外的入站埠
- **低頻寬**：適合 4G/LTE 行動網路
- **內建重連**：網路不穩定時自動恢復
- **雙向通訊**：雲端可以下達控制命令

### 5.2 多案場聚合

當你管理多個案場時，Sparkplug B 的 Topic 結構天然支援多站點架構：

```
# 桃園案場
spBv1.0/taoyuan_ess/DDATA/gw_01/pcs_01
spBv1.0/taoyuan_ess/DDATA/gw_01/pcs_02
spBv1.0/taoyuan_ess/DDATA/gw_01/meter_01

# 彰化案場
spBv1.0/changhua_ess/DDATA/gw_01/pcs_01
spBv1.0/changhua_ess/DDATA/gw_01/pcs_02

# 中央監控
# 訂閱所有案場：spBv1.0/+/DDATA/#
# 訂閱特定案場：spBv1.0/taoyuan_ess/DDATA/#
# 訂閱特定設備類型（需自訂 Topic 結構）
```

### 5.3 Modbus 無法觸及之處

有些場景 Modbus 力有未逮：

| 場景 | 為什麼 Modbus 不行 | MQTT 的解法 |
|------|-------------------|-------------|
| 遠端案場 | Modbus TCP 需要 VPN/專線 | MQTT 走公網 TLS |
| 行動設備 | Modbus 不支援行動網路 | MQTT 原生支援 |
| 多重消費者 | Modbus 是 1:1 通訊 | MQTT 是 1:N 廣播 |
| 歷史資料回溯 | Modbus 只有即時值 | MQTT + 時序資料庫 |

### 決策矩陣

```
使用 Modbus 直連：
  ✅ 案場內部設備通訊
  ✅ 即時控制迴路（< 100ms）
  ✅ 簡單架構、少量設備

使用 MQTT（純 MQTT）：
  ✅ 小型 IoT 專案
  ✅ 團隊內部系統
  ✅ Payload 格式可自訂

使用 Sparkplug B：
  ✅ 多案場、多廠商整合
  ✅ 需要標準化的設備生命週期
  ✅ 需要自描述的資料模型
  ✅ 與第三方 SCADA 平台對接
```

---

## 6. 整合模式：csp_lib 設備 → MQTT 發布者

讓我們設計一個將 csp_lib 設備資料透過 MQTT/Sparkplug B 發布到雲端的整合架構。

### 6.1 事件驅動發布

csp_lib 的 `DeviceEventEmitter` 提供了完美的事件驅動發布基礎。我們可以訂閱設備事件，自動轉換為 MQTT 訊息：

```python
"""
csp_lib 設備 → MQTT Sparkplug B 發布器（概念性範例）

將 csp_lib 的設備事件映射為 Sparkplug B 訊息。
"""

import asyncio
import json
import time
from dataclasses import dataclass
from typing import Any

from csp_lib.equipment.device.events import (
    EVENT_CONNECTED,
    EVENT_DISCONNECTED,
    EVENT_VALUE_CHANGE,
    ConnectedPayload,
    DisconnectPayload,
    ValueChangePayload,
)


@dataclass(frozen=True)
class MqttPublisherConfig:
    """MQTT 發布器配置"""

    broker_host: str = "broker.example.com"
    broker_port: int = 8883  # TLS
    group_id: str = "my_site"
    edge_node_id: str = "gateway_01"
    publish_interval: float = 5.0  # 定期發布間隔（秒）


class SparkplugBPublisher:
    """
    將 csp_lib 設備事件轉換為 Sparkplug B MQTT 訊息

    這是一個概念性的實作，展示 csp_lib 事件系統
    如何自然映射到 Sparkplug B 的生命週期。
    """

    def __init__(self, config: MqttPublisherConfig):
        self._config = config
        self._metric_aliases: dict[str, dict[str, int]] = {}  # device_id → {name: alias}
        self._alias_counter = 0

    def _topic(self, message_type: str, device_id: str | None = None) -> str:
        """建構 Sparkplug B Topic"""
        base = f"spBv1.0/{self._config.group_id}/{message_type}/{self._config.edge_node_id}"
        if device_id:
            return f"{base}/{device_id}"
        return base

    def build_dbirth_payload(
        self,
        device_id: str,
        template_points: list[dict[str, Any]],
        initial_values: dict[str, Any],
    ) -> dict[str, Any]:
        """
        建構 DBIRTH payload

        從 csp_lib 的 EquipmentTemplate 資訊建構 Sparkplug B 的 Birth Certificate。
        """
        metrics = []
        self._metric_aliases[device_id] = {}

        for point_info in template_points:
            self._alias_counter += 1
            alias = self._alias_counter
            name = point_info["name"]

            self._metric_aliases[device_id][name] = alias

            metric = {
                "name": name,
                "alias": alias,
                "timestamp": int(time.time() * 1000),
                "datatype": point_info.get("datatype", "Float"),
                "value": initial_values.get(name, 0),
            }

            # 附加 properties（來自 PointMetadata）
            if "unit" in point_info:
                metric["properties"] = {"unit": point_info["unit"]}

            metrics.append(metric)

        return {
            "timestamp": int(time.time() * 1000),
            "metrics": metrics,
            "seq": 0,
        }

    def build_ddata_payload(
        self,
        device_id: str,
        changed_values: dict[str, Any],
    ) -> dict[str, Any]:
        """
        建構 DDATA payload

        只包含變化的 metrics，使用 alias 代替完整名稱。
        """
        metrics = []
        aliases = self._metric_aliases.get(device_id, {})

        for name, value in changed_values.items():
            alias = aliases.get(name)
            if alias is None:
                continue  # 未知的 metric，跳過

            metrics.append({
                "alias": alias,
                "timestamp": int(time.time() * 1000),
                "value": value,
            })

        return {
            "timestamp": int(time.time() * 1000),
            "metrics": metrics,
        }

    def build_ddeath_payload(self, device_id: str) -> dict[str, Any]:
        """建構 DDEATH payload"""
        return {
            "timestamp": int(time.time() * 1000),
        }


# ---- 整合 csp_lib 設備事件 ----

def bind_device_to_publisher(device, publisher: SparkplugBPublisher):
    """
    將 csp_lib 設備的事件綁定到 Sparkplug B 發布器

    這個函式展示了 csp_lib 事件系統如何驅動 MQTT 發布。
    """

    async def on_connected(payload: ConnectedPayload):
        """設備上線 → 發布 DBIRTH"""
        topic = publisher._topic("DBIRTH", payload.device_id)

        # 從 device 的 template 取得點位資訊
        # 實務中會從 EquipmentTemplate 提取
        dbirth = publisher.build_dbirth_payload(
            device_id=payload.device_id,
            template_points=[
                {"name": "active_power", "datatype": "Float", "unit": "kW"},
                {"name": "reactive_power", "datatype": "Float", "unit": "kVar"},
                {"name": "soc", "datatype": "Float", "unit": "%"},
            ],
            initial_values=device.latest_values,
        )
        print(f"PUBLISH {topic}: {json.dumps(dbirth, indent=2)}")
        # await mqtt_client.publish(topic, encode_sparkplug(dbirth))

    async def on_disconnected(payload: DisconnectPayload):
        """設備離線 → 發布 DDEATH"""
        topic = publisher._topic("DDEATH", payload.device_id)
        ddeath = publisher.build_ddeath_payload(payload.device_id)
        print(f"PUBLISH {topic}: {json.dumps(ddeath)}")

    async def on_value_change(payload: ValueChangePayload):
        """值變化 → 發布 DDATA"""
        topic = publisher._topic("DDATA", payload.device_id)
        ddata = publisher.build_ddata_payload(
            device_id=payload.device_id,
            changed_values={payload.point_name: payload.new_value},
        )
        print(f"PUBLISH {topic}: {json.dumps(ddata)}")

    # 綁定事件
    device.on(EVENT_CONNECTED, on_connected)
    device.on(EVENT_DISCONNECTED, on_disconnected)
    device.on(EVENT_VALUE_CHANGE, on_value_change)
```

### 6.2 定期批次發布

除了事件驅動的逐筆發布，實務中更常見的是定期批次發布所有設備資料：

```python
async def periodic_publish_loop(
    devices: list,
    publisher: SparkplugBPublisher,
    interval: float = 5.0,
):
    """
    定期批次發布所有設備資料

    類似 csp_lib 的 DataUploadManager 概念，
    但輸出目標從 MongoDB 改為 MQTT。
    """
    while True:
        for device in devices:
            if not device.is_responsive:
                continue

            values = device.latest_values
            if not values:
                continue

            topic = publisher._topic("DDATA", device.device_id)
            ddata = publisher.build_ddata_payload(
                device_id=device.device_id,
                changed_values=values,  # 全量發布
            )
            # await mqtt_client.publish(topic, encode_sparkplug(ddata))

        await asyncio.sleep(interval)
```

### 6.3 命令接收（DCMD → device.write）

反方向的整合——從雲端接收控制命令，轉換為 csp_lib 的設備寫入：

```python
async def command_listener(
    devices_by_id: dict,
    publisher: SparkplugBPublisher,
):
    """
    監聽 Sparkplug B DCMD 訊息，轉換為 csp_lib device.write()

    Topic: spBv1.0/{group_id}/DCMD/{edge_node_id}/{device_id}
    """
    topic_pattern = publisher._topic("DCMD", "+")  # 萬用字元

    # await mqtt_client.subscribe(topic_pattern)
    # async for message in mqtt_client.messages:
    #     device_id = extract_device_id(message.topic)
    #     payload = decode_sparkplug(message.payload)
    #
    #     device = devices_by_id.get(device_id)
    #     if device is None:
    #         continue
    #
    #     for metric in payload["metrics"]:
    #         point_name = resolve_alias(metric.get("alias"))
    #         value = metric["value"]
    #         await device.write(point_name, value)
```

---

## 7. 重點整理與下篇預告

### 本篇重點

1. **MQTT 對後端工程師來說很熟悉**，但純 MQTT 缺乏工業所需的標準化：沒有統一的 payload 格式、沒有設備生命週期管理、沒有狀態管理機制

2. **Sparkplug B 在 MQTT 之上建立工業語義**：
   - 標準化的 Topic 命名空間：`spBv1.0/{group}/{type}/{node}/{device}`
   - Birth/Death 證書：設備自描述上下線機制
   - 結構化的 Metric 型別系統
   - Alias 機制節省傳輸頻寬

3. **csp_lib 的事件系統天然映射 Sparkplug B 生命週期**：
   - `EVENT_CONNECTED` → DBIRTH
   - `EVENT_DISCONNECTED` → DDEATH
   - `EVENT_VALUE_CHANGE` → DDATA
   - `device.write()` ← DCMD

4. **MQTT/Sparkplug B 的適用場景**：雲端連接、多案場聚合、遠端監控——那些 Modbus 無法直接觸及的場景

5. **與 Modbus 是互補而非替代**：案場內部用 Modbus 直連設備，對外用 MQTT 上傳雲端，是最常見的混合架構

### 給後端工程師的行動建議

- 如果你的系統需要雲端上傳：先從簡單的 JSON over MQTT 開始，當規模擴大再考慮 Sparkplug B
- 如果你需要跨廠商整合：Sparkplug B 的 Birth Certificate 機制可以大幅減少整合工作
- 如果你正在用 csp_lib：思考如何利用事件系統建立 MQTT 橋接層，而不是在每個地方都直接呼叫 MQTT client

### 下篇預告

下一篇文章我們將回到實務——**沒有設備也能開發：測試驅動的協定開發**。我們會深入探討 csp_lib 的測試策略：如何用 Mock 模擬設備、如何用 `modbus_server` 模組建立完整的模擬環境、以及如何在 CI/CD 中運行不需要真實硬體的整合測試。對於開發工業控制軟體來說，「沒有設備」是常態而非例外，掌握測試技術是生產力的關鍵。

---

> **系列導航**
> - 上一篇：Article 08 — IEC 61850：智慧電網的語義模型
> - **本篇：Article 09 — MQTT 與 Sparkplug B：工業物聯網的雲端橋樑**
> - 下一篇：Article 10 — 沒有設備也能開發：測試驅動的協定開發
