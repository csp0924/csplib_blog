# Keepalived 與 VRRP：網路層的高可用切換

> **從零打造工業控制協定框架：給軟體工程師的 CSP Library 實戰系列**
> Part 5 — 高可用篇 | Article 23

---

## 前言：應用層 HA 夠了嗎？

上一篇文章，我們建立了 MongoDB Replica Set 來保障資料的持久化。在這個系列的前幾篇中，我們也透過 etcd leader election（csp_lib 的 `LeaderElector`）實現了控制器層的主備切換——當 Leader 節點的程式當機時，Follower 會在幾秒內接管控制迴路。

聽起來很完美，但讓我們思考一個問題：現場的 PCS、電錶、逆變器，它們是怎麼連上控制器的？

答案是：**TCP 連線到一個固定的 IP 位址**。

Modbus TCP 設備在配置時會指定控制器的 IP（例如 `192.168.1.100`），然後持續與這個 IP 建立 TCP 連線。如果 Leader 節點的網路介面故障——注意，不是程式當機，是網路卡壞了或網路線被拔了——會發生什麼？

1. etcd leader election 偵測到 lease 過期，Follower 節點升格為新 Leader。
2. 新 Leader 啟動控制迴路......但設備還在嘗試連線到舊 IP。
3. 如果新 Leader 的 IP 是 `192.168.1.101`，設備根本不知道要改連到這個新 IP。

這就是**網路層 HA** 的問題。應用層的 failover 和網路層的 failover 是兩個獨立的議題，都必須解決，系統才能真正做到高可用。

---

## VRRP 協定：Virtual Router Redundancy Protocol

### 什麼是 VRRP？

VRRP（Virtual Router Redundancy Protocol，虛擬路由器冗餘協定）是 RFC 5798 定義的標準協定。它的核心概念非常簡單：**多台機器共享一個虛擬 IP 位址（VIP），任何時刻只有一台機器「擁有」這個 VIP**。

在工業控制的場景中，設備連線的目標不再是某台特定伺服器的物理 IP，而是一個 VIP。誰擁有這個 VIP，誰就是設備的通訊對象。

### VRRP 的運作機制

VRRP 定義了兩個角色：

- **Master**：目前擁有 VIP 的節點。Master 定期發送 Advertisement 封包（預設每秒一次），告訴 Backup 節點「我還活著」。
- **Backup**：監聽 Advertisement 封包的節點。如果連續超過 `3 x Advertisement 間隔 + 偏移` 都沒收到 Advertisement，就判定 Master 已經失效，並接管 VIP。

接管 VIP 的過程如下：

```
時間線：
──────────────────────────────────────────────────────────
t=0s    Master 正常發送 Advertisement
t=1s    Master 正常發送 Advertisement
t=2s    Master 網卡故障！停止發送
t=3s    Backup 等待中...
t=4s    Backup 等待中...（累積 2 秒未收到）
t=5s    Backup 判定 Master 失效！
        ├── 將 VIP 綁定到自己的網卡
        ├── 發送 Gratuitous ARP（更新交換機的 MAC 表）
        └── 開始以 Master 身份發送 Advertisement
t=5.1s  交換機更新轉發表，所有發往 VIP 的封包轉向新 Master
t=5.2s  設備的 TCP 連線超時，自動重連到 VIP（現在指向新 Master）
```

### Gratuitous ARP：快速切換的關鍵

VIP 切換的速度取決於 **Gratuitous ARP**（免費 ARP）。當 Backup 接管 VIP 時，它會廣播一個 ARP 回應封包，告訴同一網段的所有設備：「VIP `192.168.1.100` 的 MAC 位址現在是 `AA:BB:CC:DD:EE:FF`（我的 MAC）」。

交換機收到這個封包後，會立即更新它的 MAC 位址轉發表，把所有發往 VIP 的封包轉向新的 Master。這個過程通常在毫秒級完成。

對於 Modbus TCP 設備來說，TCP 連線會因為超時而斷開，然後設備會自動重新連線到 VIP——此時 VIP 已經指向新的 Master 了。整個過程對設備來說是透明的：它始終連線到同一個 IP，只是背後的伺服器換了一台。

---

## Keepalived：VRRP 的 Linux 實作

### 什麼是 Keepalived？

Keepalived 是 VRRP 在 Linux 上最成熟的實作。它以 daemon 的形式運行，負責：

1. 管理 VIP 的綁定與釋放
2. 發送和接收 VRRP Advertisement
3. 執行健康檢查腳本，判斷是否需要主動降級

### 基本配置

以下是一個針對 csp_lib 部署的 Keepalived 配置範例：

**Node A（優先權較高，預設為 Master）：**

```conf
# /etc/keepalived/keepalived.conf (Node A)

global_defs {
    router_id EMS_NODE_A
    script_security SCRIPT
    enable_script_security
}

# 健康檢查腳本
vrrp_script chk_ems {
    script "/usr/local/bin/check_ems_health.sh"
    interval 2        # 每 2 秒檢查一次
    weight -20         # 健康檢查失敗時降低 20 點優先權
    fall 3             # 連續失敗 3 次才判定為不健康
    rise 2             # 連續成功 2 次才判定為恢復
}

vrrp_instance VI_EMS {
    state MASTER                   # 初始角色
    interface eth0                 # 綁定的網路介面
    virtual_router_id 51           # VRRP 群組 ID（兩台必須相同）
    priority 100                   # 優先權（數字越大越優先）
    advert_int 1                   # Advertisement 間隔（秒）

    authentication {
        auth_type PASS
        auth_pass EMS_SECRET_2026  # 認證密碼（兩台必須相同）
    }

    virtual_ipaddress {
        192.168.1.100/24           # VIP
    }

    track_script {
        chk_ems                    # 追蹤健康檢查腳本
    }

    # 狀態變更時執行通知腳本
    notify_master "/usr/local/bin/ems_notify.sh MASTER"
    notify_backup "/usr/local/bin/ems_notify.sh BACKUP"
    notify_fault  "/usr/local/bin/ems_notify.sh FAULT"
}
```

**Node B（優先權較低，預設為 Backup）：**

```conf
# /etc/keepalived/keepalived.conf (Node B)

global_defs {
    router_id EMS_NODE_B
    script_security SCRIPT
    enable_script_security
}

vrrp_script chk_ems {
    script "/usr/local/bin/check_ems_health.sh"
    interval 2
    weight -20
    fall 3
    rise 2
}

vrrp_instance VI_EMS {
    state BACKUP                   # 初始角色
    interface eth0
    virtual_router_id 51           # 必須與 Node A 相同
    priority 90                    # 低於 Node A 的 100
    advert_int 1

    authentication {
        auth_type PASS
        auth_pass EMS_SECRET_2026  # 必須與 Node A 相同
    }

    virtual_ipaddress {
        192.168.1.100/24           # 相同的 VIP
    }

    track_script {
        chk_ems
    }

    notify_master "/usr/local/bin/ems_notify.sh MASTER"
    notify_backup "/usr/local/bin/ems_notify.sh BACKUP"
    notify_fault  "/usr/local/bin/ems_notify.sh FAULT"
}
```

### 健康檢查腳本

Keepalived 的 `vrrp_script` 機制允許你執行自定義的健康檢查。如果腳本返回非零退出碼，Keepalived 會根據 `weight` 參數調整該節點的優先權。

以下是一個結合 csp_lib 健康檢查的腳本：

```bash
#!/bin/bash
# /usr/local/bin/check_ems_health.sh
#
# 檢查 EMS 控制器的健康狀態：
# 1. EMS 程式是否在執行
# 2. HTTP 健康端點是否回應正常
# 3. etcd 連線是否正常

# 檢查 EMS 程式是否存在
if ! pgrep -f "python.*ems_controller" > /dev/null; then
    echo "EMS process not running"
    exit 1
fi

# 檢查健康端點
HEALTH_RESPONSE=$(curl -s -o /dev/null -w "%{http_code}" \
    --max-time 3 \
    http://localhost:8080/health)

if [ "$HEALTH_RESPONSE" != "200" ]; then
    echo "Health endpoint returned $HEALTH_RESPONSE"
    exit 1
fi

# 檢查健康狀態是否為 healthy 或 degraded（degraded 仍可運作）
HEALTH_STATUS=$(curl -s --max-time 3 http://localhost:8080/health | \
    python3 -c "import sys,json; print(json.load(sys.stdin).get('status',''))")

if [ "$HEALTH_STATUS" = "unhealthy" ]; then
    echo "System is unhealthy"
    exit 1
fi

exit 0
```

這個腳本利用了 csp_lib 的健康檢查機制。在 csp_lib 中，`HealthCheckable` 是一個 Protocol，所有核心元件都實作了它：

```python
from csp_lib.core.health import HealthCheckable, HealthStatus, HealthReport

# HealthReport 的結構
report = HealthReport(
    status=HealthStatus.HEALTHY,       # HEALTHY | DEGRADED | UNHEALTHY
    component="system_controller",
    message="All systems operational",
    details={
        "active_devices": 5,
        "connected_devices": 5,
        "active_alarms": 0,
    },
    children=[                         # 子元件的健康報告
        HealthReport(
            status=HealthStatus.HEALTHY,
            component="mongodb",
            message="Connected to replica set rs0",
        ),
        HealthReport(
            status=HealthStatus.HEALTHY,
            component="redis",
            message="Connected",
        ),
    ],
)
```

`HealthStatus` 有三個等級：

| 狀態 | 含義 | Keepalived 動作 |
|------|------|----------------|
| `HEALTHY` | 一切正常 | 維持正常優先權 |
| `DEGRADED` | 部分功能降級，但仍可運作 | 維持正常優先權 |
| `UNHEALTHY` | 系統無法正常運作 | 降低優先權，觸發 VIP 遷移 |

### 健康檢查的遲滯設計

注意配置中的 `fall 3` 和 `rise 2`：

- `fall 3`：連續 3 次檢查失敗才判定為不健康。這避免了瞬時的網路波動導致不必要的 VIP 切換。
- `rise 2`：連續 2 次檢查成功才判定為恢復。這避免了系統尚未完全啟動就搶回 VIP。

這與 csp_lib 的 `MonitorConfig` 中的遲滯設計理念一致——`hysteresis_activate` 和 `hysteresis_clear` 參數控制系統告警的觸發和解除門檻：

```python
from csp_lib.monitor import MonitorConfig, MetricThresholds

monitor_config = MonitorConfig(
    interval_seconds=5.0,
    thresholds=MetricThresholds(
        cpu_percent=90.0,
        ram_percent=85.0,
        disk_percent=95.0,
    ),
    hysteresis_activate=3,  # 連續 3 次超閾值才觸發告警
    hysteresis_clear=3,     # 連續 3 次低於閾值才解除告警
)
```

---

## 為什麼需要 etcd AND Keepalived？

這是最容易讓人困惑的地方：既然 etcd leader election 已經可以選出 Leader，為什麼還需要 Keepalived？它們不是在做同一件事嗎？

答案是否定的。它們解決的是不同層次的問題：

### 兩層 failover 的角色劃分

```
┌─────────────────────────────────────────────────────────────┐
│                      應用層 (Layer 7)                        │
│                                                             │
│  etcd Leader Election                                       │
│  ├── 職責：決定「誰執行控制迴路」                               │
│  ├── 機制：lease-based election（LeaderElector）              │
│  ├── 粒度：應用程式層級                                       │
│  └── 切換時間：lease_ttl 到期（預設 10 秒）                    │
│                                                             │
├─────────────────────────────────────────────────────────────┤
│                      網路層 (Layer 3)                        │
│                                                             │
│  Keepalived + VRRP                                          │
│  ├── 職責：決定「誰擁有 VIP（設備連線的目標 IP）」              │
│  ├── 機制：VRRP Advertisement + Gratuitous ARP               │
│  ├── 粒度：網路介面層級                                       │
│  └── 切換時間：3 x advert_int + 偏移（約 3-4 秒）            │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

### 完整的架構圖

```
                    VIP: 192.168.1.100
                         │
                    ┌────┴────┐
                    │  VRRP   │
                    └────┬────┘
          ┌──────────────┼──────────────┐
          │              │              │
     Node A              │         Node B
     192.168.1.10        │         192.168.1.11
     ┌───────────┐       │         ┌───────────┐
     │ Keepalived│       │         │ Keepalived│
     │ (MASTER)  │  Advertisement  │ (BACKUP)  │
     │ 擁有 VIP  │ ─────────────→  │ 監聽中    │
     └─────┬─────┘                 └─────┬─────┘
           │                             │
     ┌─────┴─────┐                 ┌─────┴─────┐
     │   etcd    │                 │   etcd    │
     │  LEADER   │                 │ FOLLOWER  │
     │ 執行控制   │                 │  待命中   │
     └─────┬─────┘                 └───────────┘
           │
     ┌─────┴─────────────────────────────┐
     │         Control Loop              │
     │  StrategyExecutor                 │
     │  CommandRouter                    │
     │  ProtectionGuard                  │
     └──────────────────────────────────┘
           │
     ┌─────┴─────┐
     │  Devices   │
     │  PCS x 5   │
     │  Meter x 2 │
     └───────────┘
```

### 兩者必須協調一致

理想狀態下，etcd 的 Leader 和 Keepalived 的 Master 應該是**同一台機器**。如果不一致——例如 Node A 是 etcd Leader 但 Node B 是 Keepalived Master——設備會連線到 Node B，但控制邏輯在 Node A 上運行，指令無法到達設備。

協調方式有兩種：

**方式一：Keepalived 追蹤 etcd 狀態（推薦）**

讓 Keepalived 的健康檢查腳本包含 etcd leader 狀態判斷：

```bash
#!/bin/bash
# /usr/local/bin/check_ems_health.sh

# ... 基本健康檢查 ...

# 檢查 etcd election 狀態
# 如果本機是 etcd leader，優先權不變
# 如果本機是 etcd follower，降低優先權讓 VIP 漂移到 leader 節點
ELECTION_STATE=$(curl -s --max-time 3 http://localhost:8080/cluster/state | \
    python3 -c "import sys,json; print(json.load(sys.stdin).get('election_state',''))")

if [ "$ELECTION_STATE" = "follower" ]; then
    echo "Not etcd leader, yielding VIP"
    exit 1  # 觸發 weight 降低，讓 VIP 漂移
fi

exit 0
```

**方式二：etcd Leader 啟動時主動搶 VIP**

在 `LeaderElector` 的 `on_elected` 回呼中，調整 Keepalived 的優先權：

```python
from csp_lib.cluster import ClusterConfig, EtcdConfig, LeaderElector

async def on_elected():
    """被選為 Leader 時執行"""
    # 提升 Keepalived 優先權，搶回 VIP
    import subprocess
    subprocess.run(
        ["keepalived", "--reload"],
        check=False,
    )
    logger.info("Elected as leader, notifying Keepalived")

async def on_demoted():
    """從 Leader 降級時執行"""
    logger.warning("Demoted from leader, stopping control loop")

config = ClusterConfig(
    instance_id="node_a",
    etcd=EtcdConfig(endpoints=["etcd1:2379", "etcd2:2379", "etcd3:2379"]),
    election_key="/csp/cluster/election",
    lease_ttl=10,
)

elector = LeaderElector(
    config=config,
    on_elected=on_elected,
    on_demoted=on_demoted,
)

async with elector:
    # elector 運行中，自動參與選舉
    await asyncio.Event().wait()
```

`LeaderElector` 的選舉演算法基於 etcd lease：

1. Grant 一個 TTL 為 10 秒的 lease
2. 嘗試以 transaction 寫入 election key：`IF key NOT EXISTS THEN PUT(key, instance_id, lease)`
3. 成功 → 成為 LEADER，啟動 keepalive loop 持續更新 lease
4. 失敗 → 成為 FOLLOWER，watch key 等待 Leader 離開後重新競選

Keepalive loop 的細節值得注意——它每隔 `lease_ttl / 3` 秒更新一次 lease，並追蹤連續失敗次數。當連續失敗超過 `max_keepalive_failures` 次時，會主動 self-fence（自我隔離）：

```python
async def _keepalive_loop(self, lease_id: int) -> None:
    interval = max(self._config.lease_ttl / 3, 1.0)
    consecutive_failures = 0
    max_failures = self._config.max_keepalive_failures

    while not self._stop_event.is_set():
        try:
            await self._client.lease_keepalive(lease_id)
            consecutive_failures = 0
        except Exception:
            consecutive_failures += 1
            if consecutive_failures >= max_failures:
                logger.error("Lease keepalive failed too many times, self-fencing")
                await self._handle_demotion()
                return
```

Self-fencing 是工業系統 HA 的關鍵概念：當一個節點無法確認自己是否仍是 Leader 時，它必須**主動放棄 Leader 身份**，而不是繼續假設自己是 Leader。在控制系統中，兩個節點同時認為自己是 Leader（split-brain）並同時下達衝突的控制指令，可能導致災難性的後果。

---

## 失效場景分析

讓我們分析幾種常見的失效場景，看看雙層 HA 是如何協同工作的：

### 場景一：Node A 應用程式當機

```
事件：Node A 的 Python 程式崩潰
─────────────────────────────────────────
t=0s    Node A 程式崩潰
t=2s    Keepalived 健康檢查失敗 (chk_ems 發現程式不存在)
t=6s    Keepalived fall=3 觸發，Node A 優先權降至 80
        Node B (priority=90) 接管 VIP
        Node B 發送 Gratuitous ARP
t=10s   etcd lease 過期（lease_ttl=10s）
        Node B 的 LeaderElector 競選成功，成為 etcd Leader
        on_elected() 啟動控制迴路
t=10.2s 設備 TCP 重連到 VIP（現在指向 Node B）
t=11s   系統完全恢復，Node B 同時是 etcd Leader + Keepalived Master
```

### 場景二：Node A 網路故障

```
事件：Node A 的 eth0 網路斷線
─────────────────────────────────────────
t=0s    Node A 網路斷線
        Node A 無法發送 VRRP Advertisement
        Node A 無法更新 etcd lease（etcd 也連不上）
t=3s    Node B 沒收到 Advertisement，接管 VIP
t=10s   etcd lease 過期，Node B 競選成功
t=10s   Node A 偵測到 keepalive 失敗，self-fence（自動降級為 Follower）
t=11s   系統完全恢復
```

### 場景三：Node A 整台機器當機

```
事件：Node A 硬體故障或斷電
─────────────────────────────────────────
t=0s    Node A 完全失聯
t=3s    Node B 接管 VIP（VRRP 超時）
t=10s   Node B 成為 etcd Leader（lease 過期）
t=11s   系統完全恢復
```

在所有場景中，完全恢復的時間上限是 `max(3 x advert_int, lease_ttl)` 再加上設備 TCP 重連的時間（通常 1-2 秒）。以預設配置來說，約 12 秒。

---

## 工業網路的部署考量

### 網路拓撲

工業環境的網路與辦公室或資料中心有顯著差異：

```
┌─────────────────────────────────────────────────────┐
│                   控制中心網路                        │
│                   10.0.0.0/24                        │
│                                                     │
│    Node A (10.0.0.10) ──── Node B (10.0.0.11)       │
│                     VIP: 10.0.0.100                  │
│                          │                           │
│                    ┌─────┴─────┐                     │
│                    │ L2 Switch │                     │
│                    └─────┬─────┘                     │
│                          │                           │
├──────────────────────────┼───────────────────────────┤
│                   現場設備網路                        │
│                 192.168.1.0/24                        │
│                          │                           │
│     ┌────────────────────┼────────────────┐          │
│     │                    │                │          │
│  PCS #1              PCS #2           Meter #1      │
│  192.168.1.101       192.168.1.102    192.168.1.201 │
│                                                     │
│     所有設備都連線到 VIP: 192.168.1.100              │
└─────────────────────────────────────────────────────┘
```

注意：VIP 必須與設備在同一個 Layer 2 網段。如果控制器和設備之間有 Layer 3 路由器，Gratuitous ARP 無法穿越路由器邊界，VIP 切換會失效。在這種情況下，你需要在路由器上配置額外的方案（如 BGP failover）。

### 多 VIP 部署

某些大型場站可能需要多個 VIP——例如一個用於 Modbus TCP 設備通訊，另一個用於 Web 管理介面：

```conf
vrrp_instance VI_MODBUS {
    state MASTER
    interface eth0
    virtual_router_id 51
    priority 100
    virtual_ipaddress {
        192.168.1.100/24    # 設備通訊 VIP
    }
}

vrrp_instance VI_WEB {
    state MASTER
    interface eth1
    virtual_router_id 52
    priority 100
    virtual_ipaddress {
        10.0.0.100/24       # Web 管理 VIP
    }
}
```

### 防火牆配置

VRRP 使用 IP 協定號 112（不是 TCP 或 UDP），你需要在防火牆上放行：

```bash
# iptables
iptables -A INPUT -p vrrp -j ACCEPT

# 或者 firewalld
firewall-cmd --add-protocol=vrrp --permanent
firewall-cmd --reload

# 同時放行 VRRP 的組播位址
iptables -A INPUT -d 224.0.0.18 -j ACCEPT
```

### 與 systemd 整合

在生產環境中，Keepalived 應該由 systemd 管理，確保開機自動啟動和異常重啟：

```ini
# /etc/systemd/system/keepalived.service (通常已隨套件安裝)
[Unit]
Description=Keepalived VRRP daemon
After=network-online.target
Wants=network-online.target

[Service]
Type=notify
ExecStart=/usr/sbin/keepalived --dont-fork
ExecReload=/bin/kill -HUP $MAINPID
Restart=always
RestartSec=3

[Install]
WantedBy=multi-user.target
```

---

## 監控 Keepalived 的狀態

在 csp_lib 的監控體系中，你可以透過 `SystemMetricsCollector` 收集系統指標，並搭配 `ModuleHealthCollector` 追蹤 Keepalived 的狀態：

```python
from csp_lib.monitor import (
    SystemMetricsCollector,
    ModuleHealthCollector,
    MonitorConfig,
    MetricThresholds,
)
from csp_lib.core.health import HealthReport, HealthStatus

config = MonitorConfig(
    interval_seconds=5.0,
    thresholds=MetricThresholds(
        cpu_percent=90.0,
        ram_percent=85.0,
        disk_percent=95.0,
    ),
    enable_cpu=True,
    enable_ram=True,
    enable_disk=True,
    enable_network=True,
    disk_paths=("/",),
)

# 系統指標收集
sys_collector = SystemMetricsCollector(config)
metrics = sys_collector.collect()
print(f"CPU: {metrics.cpu_percent}%")
print(f"RAM: {metrics.ram_percent}%")
print(f"Network send rate: {metrics.net_send_rate} bytes/s")
# 也可以取得每個網路介面的流量：
for iface_name, iface_metrics in metrics.interface_metrics.items():
    print(f"  {iface_name}: send={iface_metrics.send_rate} B/s, recv={iface_metrics.recv_rate} B/s")

# 模組健康收集
health_collector = ModuleHealthCollector()

# 註冊 Keepalived 健康檢查
def check_keepalived() -> HealthReport:
    import subprocess
    result = subprocess.run(
        ["systemctl", "is-active", "keepalived"],
        capture_output=True, text=True,
    )
    if result.stdout.strip() == "active":
        return HealthReport(
            status=HealthStatus.HEALTHY,
            component="keepalived",
            message="Keepalived is running",
        )
    return HealthReport(
        status=HealthStatus.UNHEALTHY,
        component="keepalived",
        message=f"Keepalived is {result.stdout.strip()}",
    )

health_collector.register_check("keepalived", check_keepalived)

# 收集整體健康快照
snapshot = health_collector.collect()
print(f"Overall: {snapshot.overall_status.value}")
for module in snapshot.modules:
    print(f"  {module.name}: {module.status.value} - {module.message}")
```

`ModuleHealthCollector` 的 `_compute_overall` 方法實現了一個簡潔的聚合邏輯：任何一個模組 UNHEALTHY 就整體 UNHEALTHY，任何一個 DEGRADED 就整體 DEGRADED，全部 HEALTHY 才整體 HEALTHY。

---

## 完整的高可用架構回顧

到目前為止，我們已經建立了一個三層的高可用架構：

```
Layer 3 — 網路層 HA（本篇）
├── Keepalived + VRRP
├── VIP 自動遷移
└── 切換時間：~3 秒

Layer 7 — 應用層 HA（前篇）
├── etcd Leader Election（LeaderElector）
├── 控制迴路自動接管
└── 切換時間：~10 秒（lease TTL）

資料層 HA（上篇）
├── MongoDB Replica Set
├── 自動主從切換
└── 切換時間：~10-12 秒
```

三層同時運作，才能提供端到端的高可用保障。

---

## 關鍵要點

1. **應用層 HA 和網路層 HA 是獨立的問題**——etcd 解決「誰執行控制邏輯」，Keepalived 解決「設備連到誰」。
2. **VRRP 透過虛擬 IP 實現透明的網路切換**——設備始終連線到同一個 VIP，不需要知道背後的伺服器是哪一台。
3. **Gratuitous ARP 是快速切換的關鍵**——它讓交換機在毫秒內更新轉發表。
4. **健康檢查要有遲滯設計**——`fall` 和 `rise` 參數避免了瞬時波動導致的不必要切換。
5. **etcd Leader 和 Keepalived Master 必須協調一致**——透過健康檢查腳本將兩者綁定。
6. **Self-fencing 是工業系統的必備機制**——寧可暫時停止服務，也不能出現 split-brain。

---

## 下一篇預告

我們已經建立了完整的高可用架構：資料層有 MongoDB Replica Set，應用層有 etcd Leader Election，網路層有 Keepalived + VRRP。但這一切真的能在故障時正確運作嗎？下一篇文章，我們將引入混沌工程的方法論，主動在系統中注入故障，驗證我們的容錯機制是否真的如預期般運作。
