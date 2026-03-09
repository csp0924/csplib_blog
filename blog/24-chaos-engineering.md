# 混沌工程在工業系統：主動測試你的容錯能力

> **從零打造工業控制協定框架：給軟體工程師的 CSP Library 實戰系列**
> Part 5 — 高可用篇 | Article 24

---

## 前言：你的 failover 真的有用嗎？

過去三篇文章，我們一磚一瓦地砌起了高可用架構：MongoDB Replica Set 確保資料不遺失，etcd leader election 實現控制器自動接管，Keepalived + VRRP 處理網路層的 VIP 切換。一切看起來滴水不漏。

但讓我問你一個問題：**你最後一次驗證 failover 是什麼時候？**

如果答案是「部署那天測過」或「從來沒測過」，那你的高可用架構可能只是一個精心繪製的建築藍圖——它在紙上完美無缺，但你不知道它在真正的地震中是否能站得住。

Netflix 在 2011 年創造了「混沌工程」這個詞，並建造了著名的 Chaos Monkey——一隻在生產環境中隨機殺掉伺服器的「猴子」。他們的哲學很簡單：**與其等待故障找上門，不如主動製造故障，在可控的條件下測試系統的韌性**。

這個理念在工業控制系統中同樣適用，甚至更加重要。因為工業系統的故障後果遠比一個 Web 服務掛掉嚴重得多。

---

## 為什麼工業系統需要混沌測試？

### 你無法等待真正的故障

在 Web 服務的世界裡，一個服務每年可能會遇到數十次各種等級的故障。每次故障都是一次學習機會——你可以從中改善系統。但在工業控制系統中，某些故障可能幾年才發生一次。

想像一下：你的儲能系統部署了三年，從來沒有發生過控制器主機故障。然後在某個雷雨天的夏日午後，主機硬體損壞了。這時候你才發現：

- 備機的 etcd 配置三個月前更新後忘了同步。
- Keepalived 的健康檢查腳本有個 bug，在特定條件下會返回錯誤的退出碼。
- 備機上的 MongoDB 連線字串還是指向開發環境的。

這不是假設，而是真實發生過的事故。

### 法規要求災難復原測試

越來越多的產業法規要求定期進行災難復原（DR）測試。例如：

- ISO 27001 要求定期測試業務連續性計劃
- IEC 62443（工業控制系統安全）要求驗證系統在異常條件下的行為
- 某些電力公司的併聯技術規範要求提供 failover 測試報告

混沌工程為你提供了一個系統化的方法來滿足這些要求。

### 工業系統的特殊挑戰

與 Web 服務不同，工業系統的混沌測試有幾個特殊考量：

1. **不能真的破壞設備**：你可以隨意殺掉一個 Web 伺服器，但你不能隨意關閉一台正在運轉的 PCS。
2. **安全邊界不可逾越**：測試不能導致 SOC 超出安全範圍、不能導致逆功率輸出。
3. **需要在維護窗口進行**：某些測試需要在設備離線或低負載時進行。
4. **物理層故障更常見**：網路線被老鼠咬斷、接頭氧化、RS-485 匯流排碰撞——這些不是玩笑。

---

## csp_lib 的內建韌性特徵

在設計混沌實驗之前，讓我們先盤點 csp_lib 提供了哪些內建的容錯機制。這些機制就是我們要驗證的目標。

### 設備斷線偵測與自動隔離

`DeviceConfig` 的 `disconnect_threshold` 參數定義了設備被判定為「不回應」的連續失敗次數：

```python
from csp_lib.equipment.device import DeviceConfig

config = DeviceConfig(
    device_id="pcs_01",
    unit_id=1,
    read_interval=1.0,            # 每秒讀取一次
    disconnect_threshold=5,        # 連續 5 次讀取失敗 → 判定為斷線
    reconnect_interval=5.0,        # 斷線後每 5 秒嘗試重連
    max_concurrent_reads=1,        # 最大並行讀取數
)
```

當一台設備連續 5 次 Modbus 讀取失敗，`AsyncModbusDevice` 會：

1. 發射 `disconnected` 事件，攜帶 `DisconnectPayload`（包含 `device_id`、`reason`、`consecutive_failures`）
2. 將設備標記為不回應（`is_responsive = False`）
3. 進入重連循環，每隔 `reconnect_interval` 秒嘗試恢復通訊
4. 重連成功後發射 `connected` 事件，恢復正常讀取

這整個流程都是事件驅動的。`AlarmPersistenceManager` 會自動將斷線/重連事件記錄到 MongoDB。`CommandRouter` 在寫入指令前會檢查 `is_responsive` 標誌，避免向已斷線的設備送出指令。

### 斷路器模式

csp_lib 的 `CircuitBreaker` 提供了通用的斷路器實作：

```python
from csp_lib.core.resilience import CircuitBreaker, CircuitState

# 建立斷路器：連續 5 次失敗後開啟，冷卻 30 秒
breaker = CircuitBreaker(threshold=5, cooldown=30.0)

# 在通訊前檢查斷路器狀態
if breaker.allows_request():
    try:
        result = await modbus_client.read_registers(...)
        breaker.record_success()  # 成功 → 重置失敗計數
    except Exception:
        breaker.record_failure()  # 失敗 → 累計計數
else:
    # 斷路器開啟中，跳過本次請求
    # 等待冷卻時間後自動轉為 HALF_OPEN
    pass
```

斷路器的三態轉換邏輯：

```
CLOSED (正常) ── 連續失敗達閾值 ──→ OPEN (斷路)
   ▲                                    │
   │                                    │ 冷卻時間到
   │                                    ▼
   └──── 成功 ──── HALF_OPEN (半開) ◄──┘
                        │
                        │ 失敗
                        ▼
                      OPEN (斷路)
```

配合重試策略，你可以實現指數退避：

```python
from csp_lib.core.resilience import RetryPolicy

policy = RetryPolicy(
    max_retries=3,
    base_delay=1.0,
    exponential_base=2.0,
)

# 第 0 次重試：延遲 1.0 秒
# 第 1 次重試：延遲 2.0 秒
# 第 2 次重試：延遲 4.0 秒
for attempt in range(policy.max_retries):
    delay = policy.get_delay(attempt)
    await asyncio.sleep(delay)
```

### SOC 保護：安全邊界守護

`SOCProtection` 是控制迴路的最後一道防線，確保電池的充放電不會超出安全範圍：

```python
from csp_lib.controller.system import SOCProtection, SOCProtectionConfig

protection = SOCProtection(SOCProtectionConfig(
    soc_high=95.0,      # SOC >= 95% 時禁止充電
    soc_low=5.0,        # SOC <= 5% 時禁止放電
    warning_band=5.0,   # 接近上下限時漸進限制
))
```

保護邏輯的行為如下（假設 P > 0 為放電，P < 0 為充電）：

| SOC 範圍 | P > 0（放電） | P < 0（充電） |
|---------|-------------|-------------|
| SOC >= 95% | 允許 | **強制 P=0** |
| 90% <= SOC < 95% | 允許 | **漸進限制**（ratio = (95-SOC)/5） |
| 10% < SOC < 90% | 允許 | 允許 |
| 5% < SOC <= 10% | **漸進限制**（ratio = (SOC-5)/5） | 允許 |
| SOC <= 5% | **強制 P=0** | 允許 |

這個設計的精妙之處在於「警戒區漸進限制」。不是到了邊界才突然斷電，而是在接近邊界時逐漸降低功率，避免功率階躍對電網的衝擊。

### 保護鏈：ProtectionGuard

多個保護規則可以組成保護鏈（Protection Chain），由 `ProtectionGuard` 統一管理：

```python
from csp_lib.controller.system import (
    ProtectionGuard,
    SOCProtection,
    SOCProtectionConfig,
    ReversePowerProtection,
    SystemAlarmProtection,
)

guard = ProtectionGuard(rules=[
    SOCProtection(SOCProtectionConfig(soc_high=95.0, soc_low=5.0)),
    ReversePowerProtection(threshold=0.0),  # 不允許逆送
    SystemAlarmProtection(),                # 系統告警時強制停機
])

# 策略計算出命令後，經過保護鏈過濾
result = guard.apply(command, context)

if result.was_modified:
    print(f"Protection triggered: {result.triggered_rules}")
    print(f"Original: P={result.original_command.p_target}")
    print(f"Protected: P={result.protected_command.p_target}")
```

`ProtectionGuard` 的一個關鍵設計是**失效安全**（fail-safe）：如果任何保護規則本身拋出異常，它會強制命令為 P=0, Q=0：

```python
for rule in self._rules:
    try:
        current = rule.evaluate(current, context)
    except Exception:
        logger.exception(f"Protection rule '{rule.name}' failed, "
                        f"applying fail-safe (P=0, Q=0)")
        current = Command(p_target=0.0, q_target=0.0)
        triggered.append(f"{rule.name}(fail-safe)")
```

這就是所謂的「保護機制失效時，系統傾向於安全狀態」——在工業控制的語境中，什麼都不做（P=0）永遠比做錯事安全。

---

## 監控模組：混沌實驗的觀測窗口

進行混沌實驗時，你需要一個可靠的觀測手段來判斷系統是否「正確地容錯了」。csp_lib 的 `monitor` 模組提供了完整的觀測能力。

### SystemMetricsCollector

```python
from csp_lib.monitor import SystemMetricsCollector, MonitorConfig, MetricThresholds

config = MonitorConfig(
    interval_seconds=5.0,
    thresholds=MetricThresholds(
        cpu_percent=90.0,    # CPU 超過 90% 告警
        ram_percent=85.0,    # RAM 超過 85% 告警
        disk_percent=95.0,   # 磁碟超過 95% 告警
    ),
    enable_cpu=True,
    enable_ram=True,
    enable_disk=True,
    enable_network=True,
    disk_paths=("/",),
)

collector = SystemMetricsCollector(config)

# 收集一次指標
metrics = collector.collect()

print(f"CPU: {metrics.cpu_percent}%")
print(f"RAM: {metrics.ram_used_mb:.0f} / {metrics.ram_total_mb:.0f} MB ({metrics.ram_percent}%)")
print(f"Disk: {metrics.disk_usage}")
print(f"Network: send={metrics.net_send_rate:.0f} B/s, recv={metrics.net_recv_rate:.0f} B/s")

# 網路介面級別的指標
for name, iface in metrics.interface_metrics.items():
    print(f"  {name}: send={iface.send_rate:.0f} B/s, recv={iface.recv_rate:.0f} B/s")
```

在混沌實驗中，`SystemMetricsCollector` 可以幫你回答：

- CPU 是否因為 failover 而短暫飆高？
- 記憶體是否因為重連風暴而洩漏？
- 網路流量是否因為 ARP 風暴而異常增大？

### ModuleHealthCollector

```python
from csp_lib.monitor import ModuleHealthCollector

health = ModuleHealthCollector()

# 註冊 HealthCheckable 元件
health.register_module("controller", system_controller)
health.register_module("device_manager", device_manager)

# 註冊自訂檢查
health.register_check("mongodb", lambda: mongo_health_check())
health.register_check("redis", lambda: redis_health_check())
health.register_check("keepalived", lambda: keepalived_health_check())

# 收集健康快照
snapshot = health.collect()
print(f"Overall: {snapshot.overall_status.value}")
# Overall: healthy / degraded / unhealthy

for module in snapshot.modules:
    print(f"  [{module.status.value}] {module.name}: {module.message}")
```

---

## 混沌實驗食譜

以下是四個專門為 csp_lib 部署設計的混沌實驗，從簡單到複雜。每個實驗都包含：前置條件、實驗步驟、預期行為、觀察指標。

### Recipe 1：殺掉 Leader 節點 — 驗證控制權自動接管

**假設**：當 Leader 節點的程式崩潰時，Follower 應在 `lease_ttl` 秒內接管控制迴路，設備應在 VIP 切換後自動重連。

**前置條件**：
- 兩個節點都正常運行，Node A 是 Leader
- 至少 3 台設備已連線並正常讀取
- MongoDB 和 Redis 可從兩個節點存取

**實驗步驟**：

```bash
# Step 1: 確認目前狀態
curl http://node-a:8080/cluster/state
# 應返回: {"election_state": "leader", "instance_id": "node_a"}

curl http://node-b:8080/cluster/state
# 應返回: {"election_state": "follower", "instance_id": "node_b"}

# Step 2: 記錄實驗開始時間
EXPERIMENT_START=$(date -u +"%Y-%m-%dT%H:%M:%SZ")

# Step 3: 殺掉 Node A 的 EMS 程式
ssh node-a "kill -9 \$(pgrep -f 'python.*ems_controller')"

# Step 4: 觀察 Node B 的狀態變化
watch -n 1 'curl -s http://node-b:8080/cluster/state'
# 預期：約 10 秒後顯示 {"election_state": "leader"}

# Step 5: 確認設備是否重連
curl http://node-b:8080/devices
# 預期：所有設備顯示為 connected

# Step 6: 確認控制迴路恢復
curl http://node-b:8080/strategy/status
# 預期：策略正在執行
```

**預期行為**：
1. Node A 程式被殺後 ~3 秒，Keepalived 的健康檢查失敗，VIP 遷移到 Node B
2. Node A 的 etcd lease 在 ~10 秒後過期
3. Node B 的 `LeaderElector` 競選成功，`on_elected` 回呼被觸發
4. Node B 啟動控制迴路，設備透過 VIP 連線到 Node B
5. AlarmPersistenceManager 記錄了 Node A 設備的短暫斷線事件

**觀察指標**：
- failover 總時間（從程式被殺到 Node B 控制迴路啟動）
- 設備斷線持續時間
- 是否有任何指令在 failover 期間遺失
- MongoDB 中是否有完整的告警記錄

### Recipe 2：斷開 PCS 設備通訊 — 驗證優雅降級

**假設**：當個別設備通訊失敗時，系統應將該設備標記為不回應，並繼續控制其他設備。被隔離的設備恢復後應自動重新納入控制。

**前置條件**：
- 系統正常運行，5 台 PCS 都已連線
- 控制策略正在執行（如 PQ 定功率策略）

**實驗步驟**：

```bash
# Step 1: 模擬網路分區——用 iptables 封鎖特定設備的通訊
ssh controller-node "iptables -A OUTPUT -d 192.168.1.101 -j DROP"
# 這會讓 pcs_01 的所有 Modbus 請求超時

# Step 2: 等待 disconnect_threshold（預設 5 秒）
sleep 10

# Step 3: 確認設備狀態
curl http://controller:8080/devices/pcs_01
# 預期：{"is_responsive": false, "consecutive_failures": 5}

# Step 4: 確認其他設備不受影響
curl http://controller:8080/devices/pcs_02
# 預期：{"is_responsive": true, "consecutive_failures": 0}

# Step 5: 確認告警記錄
curl http://controller:8080/alarms?device_id=pcs_01&status=active
# 預期：有一筆 DISCONNECT 告警

# Step 6: 恢復通訊
ssh controller-node "iptables -D OUTPUT -d 192.168.1.101 -j DROP"

# Step 7: 等待重連
sleep 10

# Step 8: 確認設備恢復
curl http://controller:8080/devices/pcs_01
# 預期：{"is_responsive": true, "consecutive_failures": 0}

# Step 9: 確認告警已解除
curl http://controller:8080/alarms?device_id=pcs_01&status=active
# 預期：空列表（告警已解除）
```

**預期行為**（對應 csp_lib 的內部流程）：

```
t=0s     iptables 封鎖 pcs_01 流量
t=1s     AsyncModbusDevice: 讀取 pcs_01 超時 (consecutive_failures=1)
t=2s     AsyncModbusDevice: 讀取 pcs_01 超時 (consecutive_failures=2)
         ...
t=5s     AsyncModbusDevice: consecutive_failures=5 >= disconnect_threshold
         → 發射 disconnected 事件
         → AlarmPersistenceManager: 寫入 DISCONNECT 告警
         → DataUploadManager: 寫入空值記錄（保留圖表結構）
         → is_responsive = False
         → CommandRouter: 不再向 pcs_01 送出指令
t=5s+    控制策略繼續在 pcs_02~pcs_05 上執行（降級運行）
         ...
t=恢復    iptables 規則移除
t=恢復+5  AsyncModbusDevice: 重連嘗試成功
         → 發射 connected 事件
         → AlarmPersistenceManager: 解除 DISCONNECT 告警
         → is_responsive = True
         → CommandRouter: 恢復對 pcs_01 的指令
```

**觀察指標**：
- 從通訊中斷到設備被標記為不回應的時間
- 其他設備在此期間是否受到影響
- 功率分配是否自動調整（5 台 → 4 台）
- 設備恢復後是否自動重新納入控制

### Recipe 3：斷開 Redis 連線 — 驗證本地緩衝

**假設**：當 Redis 不可用時，系統應切換到本地緩衝模式，不中斷控制迴路。Redis 恢復後應自動重新同步。

**前置條件**：
- 系統正常運行，Redis 用於狀態同步和命令接收
- `ClusterStatePublisher` 正在定期發布狀態

**實驗步驟**：

```bash
# Step 1: 記錄目前 Redis 狀態
redis-cli -h redis-server ping
# 預期：PONG

# Step 2: 停止 Redis
ssh redis-server "systemctl stop redis"

# Step 3: 觀察系統行為（等待 30 秒）
sleep 30

# Step 4: 確認控制迴路仍在運行
curl http://controller:8080/strategy/status
# 預期：策略仍在執行

# Step 5: 確認健康狀態
curl http://controller:8080/health
# 預期：overall_status = "degraded"（不是 "unhealthy"）
# Redis 模組應顯示為 unhealthy，但整體仍為 degraded

# Step 6: 恢復 Redis
ssh redis-server "systemctl start redis"

# Step 7: 確認自動恢復
sleep 10
curl http://controller:8080/health
# 預期：overall_status = "healthy"
```

**預期行為**：
1. Redis 斷開後，`RedisCommandAdapter` 的訂閱會中斷（無法接收外部指令）
2. `ClusterStatePublisher` 的發布操作會失敗並記錄警告
3. 控制迴路本身不依賴 Redis，繼續正常運作
4. 系統健康狀態降級為 `DEGRADED`，但不是 `UNHEALTHY`
5. Redis 恢復後，各元件自動重新連線

**觀察指標**：
- 控制迴路是否中斷
- 外部指令（透過 Redis Pub/Sub）在 Redis 斷開期間是否被正確緩衝或拒絕
- Redis 恢復後狀態是否自動同步

### Recipe 4：模擬 SOC 邊界 — 驗證保護機制觸發

**假設**：當 SOC 接近極限值時，`SOCProtection` 應正確限制充放電功率，保護電池安全。

**前置條件**：
- PCS 正在執行充電策略（P < 0）
- 電池 SOC 目前約 85%

**實驗步驟**：

這個實驗不需要真的讓 SOC 到 95%——我們可以在測試環境中直接操控 `StrategyContext` 的 SOC 值：

```python
import asyncio
from csp_lib.controller.system import (
    SOCProtection,
    SOCProtectionConfig,
    ProtectionGuard,
    ReversePowerProtection,
    SystemAlarmProtection,
)
from csp_lib.controller.core import Command, StrategyContext

async def chaos_test_soc_protection():
    """混沌測試：SOC 保護機制驗證"""

    # 建立保護鏈
    guard = ProtectionGuard(rules=[
        SOCProtection(SOCProtectionConfig(
            soc_high=95.0,
            soc_low=5.0,
            warning_band=5.0,
        )),
        ReversePowerProtection(threshold=0.0),
        SystemAlarmProtection(),
    ])

    # 模擬策略輸出：充電 -500 kW
    charge_command = Command(p_target=-500.0, q_target=0.0)

    # 場景 1: SOC 正常範圍
    context_normal = StrategyContext(soc=70.0, extra={})
    result = guard.apply(charge_command, context_normal)
    assert not result.was_modified, "SOC=70% 不應觸發保護"
    print(f"[PASS] SOC=70%: P={result.protected_command.p_target} (no change)")

    # 場景 2: SOC 進入高側警戒區 (90-95%)
    context_warning = StrategyContext(soc=92.0, extra={})
    result = guard.apply(charge_command, context_warning)
    assert result.was_modified, "SOC=92% 應觸發漸進限制"
    # ratio = (95 - 92) / 5 = 0.6, P = -500 * 0.6 = -300
    expected_p = -500.0 * (95.0 - 92.0) / 5.0  # -300.0
    assert abs(result.protected_command.p_target - expected_p) < 0.1
    print(f"[PASS] SOC=92%: P={result.protected_command.p_target} (limited to {expected_p})")

    # 場景 3: SOC 達到上限
    context_high = StrategyContext(soc=96.0, extra={})
    result = guard.apply(charge_command, context_high)
    assert result.protected_command.p_target == 0.0, "SOC=96% 應禁止充電"
    print(f"[PASS] SOC=96%: P={result.protected_command.p_target} (charging blocked)")

    # 場景 4: 放電指令在 SOC 高時應不受限
    discharge_command = Command(p_target=500.0, q_target=0.0)
    result = guard.apply(discharge_command, context_high)
    assert not result.was_modified, "SOC=96% 時放電不應受限"
    print(f"[PASS] SOC=96% discharge: P={result.protected_command.p_target} (no change)")

    # 場景 5: SOC 極低
    context_low = StrategyContext(soc=3.0, extra={})
    result = guard.apply(discharge_command, context_low)
    assert result.protected_command.p_target == 0.0, "SOC=3% 應禁止放電"
    print(f"[PASS] SOC=3%: P={result.protected_command.p_target} (discharging blocked)")

    # 場景 6: 系統告警觸發——強制 P=0, Q=0
    context_alarm = StrategyContext(soc=50.0, extra={"system_alarm": True})
    result = guard.apply(charge_command, context_alarm)
    assert result.protected_command.p_target == 0.0
    assert result.protected_command.q_target == 0.0
    assert "system_alarm_protection" in result.triggered_rules
    print(f"[PASS] System alarm: P=0, Q=0 (forced stop)")

    print("\nAll SOC protection chaos tests passed!")

asyncio.run(chaos_test_soc_protection())
```

這個實驗驗證了保護鏈在各種邊界條件下的行為。特別重要的是場景 6——`SystemAlarmProtection` 作為最後一道防線，在系統告警時強制停止所有功率輸出。

---

## 通知系統：確保警報送達

混沌實驗中觸發的告警需要通知到運維人員。csp_lib 的 `notification` 模組提供了多通道通知機制：

```python
from csp_lib.notification import (
    NotificationDispatcher,
    NotificationChannel,
    Notification,
    NotificationEvent,
    NotificationBatcher,
    BatchNotificationConfig,
)
from csp_lib.equipment.alarm import AlarmLevel

# 實作 LINE 通知通道
class LineChannel(NotificationChannel):
    @property
    def name(self) -> str:
        return "line"

    async def send(self, notification: Notification) -> None:
        # 呼叫 LINE Messaging API
        message = f"[{notification.level.name}] {notification.title}\n{notification.body}"
        await self._line_api.push_message(self._group_id, message)


# 實作 Webhook 通知通道
class WebhookChannel(NotificationChannel):
    @property
    def name(self) -> str:
        return "webhook"

    async def send(self, notification: Notification) -> None:
        payload = {
            "title": notification.title,
            "body": notification.body,
            "level": notification.level.value,
            "device_id": notification.device_id,
            "alarm_key": notification.alarm_key,
            "event": notification.event.value,
            "occurred_at": notification.occurred_at.isoformat(),
        }
        async with aiohttp.ClientSession() as session:
            await session.post(self._webhook_url, json=payload)


# 建立多通道分發器
dispatcher = NotificationDispatcher([
    LineChannel(api_key="...", group_id="..."),
    WebhookChannel(url="https://hooks.slack.com/..."),
])

# 個別通道失敗不影響其他通道
await dispatcher.dispatch(Notification(
    title="[ALARM] pcs_01 設備斷線",
    body="Modbus TCP 連線超時，連續失敗 5 次",
    level=AlarmLevel.WARNING,
    device_id="pcs_01",
    alarm_key="pcs_01:disconnect:DISCONNECT",
    event=NotificationEvent.TRIGGERED,
    occurred_at=datetime.now(timezone.utc),
))
```

`NotificationDispatcher` 的關鍵設計是**扇出式分發，個別失敗不阻塞**：

```python
async def dispatch(self, notification: Notification) -> None:
    for channel in self._channels:
        try:
            await channel.send(notification)
        except Exception:
            logger.warning(f"通知通道 '{channel.name}' 發送失敗", exc_info=True)
```

即使 LINE API 當機了，Webhook 通知仍然會被發送。這在混沌測試中特別重要——你不希望因為通知系統本身的故障而錯過實驗期間的告警。

### 與 AlarmPersistenceManager 整合

在前面的文章中，我們看到 `AlarmPersistenceManager` 可以注入一個 `NotificationSender`。`NotificationDispatcher` 實作了 `NotificationSender` Protocol，所以可以直接注入：

```python
from csp_lib.manager.alarm import AlarmPersistenceManager, MongoAlarmRepository

alarm_manager = AlarmPersistenceManager(
    repository=MongoAlarmRepository(db),
    dispatcher=dispatcher,  # 注入通知分發器
)

alarm_manager.subscribe(pcs_device)

# 當 pcs_device 斷線時：
# 1. AlarmPersistenceManager 寫入告警記錄到 MongoDB
# 2. 同時透過 dispatcher 發送通知到 LINE 和 Webhook
# 3. 當設備恢復時，自動發送 RESOLVED 通知
```

對於非告警類型的事件通知（如系統啟動、策略切換），可以使用 `EventNotification`：

```python
from csp_lib.notification import EventNotification, EventCategory

event = EventNotification(
    title="系統啟動完成",
    body="所有設備已連線，控制策略已啟動",
    category=EventCategory.SYSTEM,
    source="system_controller",
    immediate=True,  # 繞過批次佇列，立即發送
)
```

---

## 建構混沌測試框架

將以上的實驗系統化，你可以建構一個可重複執行的混沌測試框架：

```python
import asyncio
import time
from dataclasses import dataclass, field
from datetime import datetime, timezone
from enum import Enum
from typing import Callable, Awaitable


class ExperimentResult(Enum):
    PASS = "pass"
    FAIL = "fail"
    SKIP = "skip"


@dataclass
class ChaosExperiment:
    """混沌實驗定義"""
    name: str
    description: str
    inject_fault: Callable[[], Awaitable[None]]   # 注入故障
    verify: Callable[[], Awaitable[bool]]          # 驗證系統行為
    recover: Callable[[], Awaitable[None]]         # 恢復故障
    timeout: float = 60.0                          # 最大等待時間


@dataclass
class ExperimentReport:
    """實驗結果報告"""
    experiment: str
    result: ExperimentResult
    duration: float
    details: dict = field(default_factory=dict)
    timestamp: datetime = field(
        default_factory=lambda: datetime.now(timezone.utc)
    )


async def run_experiment(experiment: ChaosExperiment) -> ExperimentReport:
    """執行單一混沌實驗"""
    start = time.monotonic()

    try:
        # 1. 注入故障
        print(f"[INJECT] {experiment.name}: {experiment.description}")
        await experiment.inject_fault()

        # 2. 等待並驗證
        print(f"[VERIFY] Waiting for system response...")
        deadline = time.monotonic() + experiment.timeout
        passed = False

        while time.monotonic() < deadline:
            if await experiment.verify():
                passed = True
                break
            await asyncio.sleep(2.0)

        duration = time.monotonic() - start

        if passed:
            print(f"[PASS] {experiment.name} ({duration:.1f}s)")
            result = ExperimentResult.PASS
        else:
            print(f"[FAIL] {experiment.name} (timeout after {duration:.1f}s)")
            result = ExperimentResult.FAIL

    except Exception as e:
        duration = time.monotonic() - start
        print(f"[ERROR] {experiment.name}: {e}")
        result = ExperimentResult.FAIL
    finally:
        # 3. 恢復故障（無論成功或失敗）
        print(f"[RECOVER] Restoring normal state...")
        try:
            await experiment.recover()
        except Exception as e:
            print(f"[WARN] Recovery failed: {e}")

    return ExperimentReport(
        experiment=experiment.name,
        result=result,
        duration=duration,
    )


# 定義實驗套件
async def chaos_suite():
    experiments = [
        ChaosExperiment(
            name="device_disconnect",
            description="Block traffic to pcs_01",
            inject_fault=lambda: block_device("192.168.1.101"),
            verify=lambda: check_device_isolated("pcs_01"),
            recover=lambda: unblock_device("192.168.1.101"),
            timeout=30.0,
        ),
        ChaosExperiment(
            name="leader_crash",
            description="Kill leader process",
            inject_fault=lambda: kill_leader_process(),
            verify=lambda: check_follower_promoted(),
            recover=lambda: restart_leader_process(),
            timeout=30.0,
        ),
        ChaosExperiment(
            name="redis_outage",
            description="Stop Redis server",
            inject_fault=lambda: stop_redis(),
            verify=lambda: check_control_loop_running(),
            recover=lambda: start_redis(),
            timeout=30.0,
        ),
    ]

    reports = []
    for exp in experiments:
        report = await run_experiment(exp)
        reports.append(report)
        # 實驗之間等待系統穩定
        await asyncio.sleep(10)

    # 輸出摘要
    print("\n" + "=" * 60)
    print("Chaos Test Summary")
    print("=" * 60)
    for r in reports:
        status = "PASS" if r.result == ExperimentResult.PASS else "FAIL"
        print(f"  [{status}] {r.experiment} ({r.duration:.1f}s)")

    passed = sum(1 for r in reports if r.result == ExperimentResult.PASS)
    total = len(reports)
    print(f"\nResult: {passed}/{total} passed")
```

### 實驗的安全守則

在工業環境中執行混沌測試，必須遵守以下守則：

1. **永遠有人值守**：不要無人看管地執行混沌實驗。
2. **從非生產環境開始**：先在測試台或 staging 環境驗證。
3. **維持逃生通道**：確保你可以隨時手動恢復系統。
4. **限制爆炸半徑**：一次只注入一個故障，不要同時搞壞多個元件。
5. **在維護窗口執行**：選擇設備低負載或離線的時段。
6. **記錄一切**：實驗前後的系統快照、所有的操作步驟、完整的日誌。

### 漸進式混沌路線圖

不要一開始就嘗試殺掉生產環境的 Leader 節點。建議按以下路線漸進：

```
Level 0: 單元測試層
├── SOCProtection 邊界測試
├── CircuitBreaker 狀態轉換測試
└── BatchQueue 容量溢出測試

Level 1: 整合測試層
├── 模擬設備通訊超時
├── 模擬 MongoDB 寫入失敗
└── 模擬 Redis 連線中斷

Level 2: Staging 環境
├── 殺掉 Leader 程式，驗證 failover
├── 封鎖設備網路流量，驗證降級
└── 停止 Redis/MongoDB，驗證韌性

Level 3: 生產環境（維護窗口）
├── 有計劃的 Leader 切換演練
├── 單一設備通訊中斷測試
└── 備援節點接管全量測試
```

---

## 關鍵要點

1. **混沌工程不是破壞——是有紀律的實驗**。每次實驗都有明確的假設、可觀測的指標、和可逆的恢復步驟。

2. **csp_lib 的多層保護設計天然支持容錯驗證**：
   - `DeviceConfig.disconnect_threshold` → 設備隔離
   - `CircuitBreaker` → 通訊斷路器
   - `SOCProtection` + `ProtectionGuard` → 安全邊界守護
   - `HealthCheckable` → 健康狀態觀測

3. **監控是混沌實驗的眼睛**——`SystemMetricsCollector` 和 `ModuleHealthCollector` 提供了系統行為的即時觀測能力。

4. **通知系統確保告警送達**——`NotificationDispatcher` 的扇出設計保證單一通道失敗不影響其他通道。

5. **漸進式推進**——從單元測試開始，逐步升級到整合測試、staging、最終在生產環境的維護窗口中進行。

6. **工業系統的混沌測試有安全底線**——保護機制的 fail-safe 設計（`P=0, Q=0`）確保即使實驗失控，系統也會傾向安全狀態。

---

## 下一篇預告

高可用篇到此告一段落。我們從 MongoDB Replica Set 的資料持久化，到 Keepalived + VRRP 的網路層 failover，再到今天的混沌工程驗證，完整地走過了工業控制系統高可用架構的設計、實作與驗證。下一個篇章，我們將進入效能優化的領域——當設備數量從 5 台成長到 50 台、500 台時，csp_lib 如何保持控制迴路的即時性。
