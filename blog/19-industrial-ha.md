# 工業系統的高可用性：不只是多開幾台機器

> **從零打造工業控制協定框架：給軟體工程師的 CSP Library 實戰系列**
> Part 5 — 高可用篇 | Article 19

在前幾篇文章中，我們已經建立了設備通訊、控制策略、以及完整的系統整合。到了這個階段，自然會有人問：「如果控制器掛了怎麼辦？」

如果你來自 Web 後端的世界，你可能會說：「不就是加一台備援嗎？」但在工業控制的領域，高可用性（High Availability, HA）的意義和複雜度，遠超你的想像。

---

## 1. Web 的 HA 和工業的 HA：代價完全不同

先讓我們比較兩個場景：

### Web 服務掛了

```
用戶打開網頁 → 502 Bad Gateway → 用戶罵一句 → 重新整理 → 正常了
```

最糟的情況是什麼？用戶抱怨、訂單延遲、SLA 被扣款。不開心，但沒有人受傷。

### 工業控制系統掛了

```
控制器離線 → 儲能系統失去調度
→ 電池持續充電超過 SOC 上限
→ 電池過熱 → 熱失控 → 火災
```

或是另一個場景：

```
控制器離線 → PCS 失去功率指令
→ 大量逆送功率灌回電網
→ 觸發電力公司保護 → 整條饋線跳脫
→ 周邊工廠停電
```

這不是危言聳聽。工業系統的 HA 不是「體驗優化」，而是「安全要求」。控制系統離線的每一秒，都可能造成設備損壞、人員安全風險、甚至法規違規。

| 面向 | Web 服務 | 工業控制系統 |
|------|---------|------------|
| 停機影響 | 用戶體驗下降 | 設備損壞、安全風險 |
| 容忍停機時間 | 秒到分鐘級 | 毫秒到秒級 |
| 故障模式 | Fail-fast, 回傳錯誤 | Fail-safe, 進入保護模式 |
| 資料一致性 | 最終一致性 OK | 命令衝突 → 物理損壞 |
| 恢復策略 | 重啟、重試 | 先確認安全，再恢復控制 |

理解了這個根本差異後，讓我們來看工業系統中到底有哪些故障類型。

---

## 2. 工業系統的故障分類

### 2.1 設備通訊故障

最常見的故障類型。Modbus 設備可能因為以下原因失去通訊：

- **物理層故障**：RS-485 線路斷裂、接線鬆動、電磁干擾
- **設備故障**：PCS / BMS 韌體當機、看門狗重啟
- **網路問題**：交換器故障、IP 衝突、TCP 連線超時

在 csp_lib 中，設備層級用 `disconnect_threshold` 追蹤連續通訊失敗：

```python
from csp_lib.equipment.device import AsyncModbusDevice, DeviceConfig

config = DeviceConfig(
    device_id="pcs_01",
    unit_id=1,
    read_interval=1.0,
    disconnect_threshold=5,      # 連續 5 次讀取失敗 → 視為無回應
    reconnect_interval=5.0,      # 無回應後每 5 秒嘗試重連
)
```

當連續失敗次數達到 `disconnect_threshold`，設備的 `is_responsive` 屬性會變為 `False`：

```python
# 設備狀態判斷
device.is_connected     # Socket 層級：TCP 連線是否存在
device.is_responsive    # 通訊層級：設備是否有回應資料
device.is_unreachable   # is_connected and not is_responsive
device.is_healthy       # is_connected and is_responsive and not is_protected
```

這裡有一個重要的設計選擇：**連線中斷不代表立即放棄**。`is_connected` 和 `is_responsive` 是兩個獨立的狀態。TCP 連線可能還在，但設備可能已經不回應了（例如韌體卡死）。反過來，TCP 斷線後設備本身可能完全正常，只是網路有問題。

### 2.2 控制器節點故障

控制器本身也可能掛掉：

- **程式崩潰**：未處理的例外、記憶體不足
- **作業系統故障**：OS 當機、磁碟滿了
- **硬體故障**：工業電腦斷電、主板故障

這是最嚴重的故障，因為整個控制迴路都停了。

### 2.3 資料庫不可用

MongoDB 或 Redis 掛了，影響：

- **MongoDB 不可用**：歷史資料無法寫入、設定無法讀取
- **Redis 不可用**：即時狀態無法同步、叢集節點間無法通訊

但這類故障通常不應該影響核心控制迴路。設備讀取和命令下發不依賴資料庫。

### 2.4 網路分區（Network Partition）

最棘手的故障類型。控制器和設備之間的網路斷了，但雙方都還在運作。這會導致：

- 控制器以為設備掛了，但設備其實正常
- 如果有備援控制器，兩邊都可能嘗試控制同一台設備

---

## 3. csp_lib 的容錯設計：三層防禦

csp_lib 的容錯不是靠單一機制，而是在三個層級分別建立防線。

### 3.1 設備層級：斷路器與狀態追蹤

csp_lib 的 `CircuitBreaker` 是一個通用的斷路器模式實作：

```python
from csp_lib.core.resilience import CircuitBreaker, CircuitState

# 建立斷路器：5 次失敗後開啟，30 秒冷卻
breaker = CircuitBreaker(threshold=5, cooldown=30.0)

# 讀取設備前檢查
if breaker.allows_request():
    try:
        values = await device.read_all()
        breaker.record_success()
    except CommunicationError:
        breaker.record_failure()
        # 如果已經達到閾值，斷路器會進入 OPEN 狀態
else:
    # 斷路器開啟中，跳過此設備
    pass
```

斷路器有三種狀態：

```
CLOSED ──(連續失敗達閾值)──→ OPEN ──(冷卻時間到)──→ HALF_OPEN
  ↑                                                      │
  └──────────────(成功)──────────────────────────────────┘
                                                         │
                              OPEN ←──────(失敗)─────────┘
```

在 `AsyncModbusDevice` 內部，這個邏輯已經內建了。設備的 `should_attempt_read` 屬性會根據連續失敗次數和重連間隔，自動決定是否該嘗試讀取：

```python
@property
def should_attempt_read(self) -> bool:
    """無回應的設備在超過 reconnect_interval 後才回傳 True"""
    if self._device_responsive:
        return True
    if self._last_failure_time is None:
        return True
    return (time.monotonic() - self._last_failure_time) >= self._config.reconnect_interval
```

這個設計的好處是：**一台設備掛掉不會拖慢整個讀取迴圈**。無回應的設備會被暫時跳過，避免 timeout 阻塞其他設備的讀取。

### 3.2 控制器層級：ProtectionGuard 安全規則

即使控制迴路還在運作，下發的命令也必須經過安全檢查。這就是 `ProtectionGuard` 的工作：

```python
from csp_lib.controller.system.protection import (
    ProtectionGuard,
    SOCProtection,
    SOCProtectionConfig,
    ReversePowerProtection,
    SystemAlarmProtection,
)

# 建立保護鏈
guard = ProtectionGuard(rules=[
    SOCProtection(SOCProtectionConfig(
        soc_high=95.0,      # SOC >= 95% → 禁止充電
        soc_low=5.0,        # SOC <= 5%  → 禁止放電
        warning_band=5.0,   # 警戒區寬度 5%，漸進限制
    )),
    ReversePowerProtection(threshold=0.0),   # 不允許逆送
    SystemAlarmProtection(),                  # 系統告警 → 強制停機
])
```

保護鏈的設計哲學是「鏈式套用、最嚴格的規則贏」：

```python
result = guard.apply(command, context)
# result.original_command    → 策略產生的原始命令
# result.protected_command   → 保護後的命令
# result.triggered_rules     → 哪些規則被觸發了
# result.was_modified        → 命令是否被修改
```

有一個非常重要的安全設計：**如果任何保護規則執行時拋出例外，會自動套用 fail-safe（P=0, Q=0）**：

```python
for rule in self._rules:
    try:
        current = rule.evaluate(current, context)
        if rule.is_triggered:
            triggered.append(rule.name)
    except Exception:
        logger.exception(f"Protection rule '{rule.name}' failed, applying fail-safe (P=0, Q=0)")
        current = Command(p_target=0.0, q_target=0.0)
        triggered.append(f"{rule.name}(fail-safe)")
```

這就是工業控制系統和 Web 的根本不同：**Web 的 catch-all 通常是回傳 500 錯誤，工業系統的 catch-all 是停止輸出**。不確定的時候，什麼都不做是最安全的選擇。

### 3.3 系統層級：ClusterController 多節點協調

當單一節點的容錯不夠用時，我們需要多節點。這就是 `ClusterController` 的角色：

```python
from csp_lib.cluster import ClusterController, ClusterConfig, EtcdConfig

cluster_config = ClusterConfig(
    instance_id="node-1",
    etcd=EtcdConfig(endpoints=["etcd1:2379", "etcd2:2379", "etcd3:2379"]),
    namespace="production",
    lease_ttl=10,                     # Leader 租約 10 秒
    failover_grace_period=2.0,        # 接任後等待 2 秒
    max_keepalive_failures=3,         # 續租失敗 3 次 → 自動降級
    device_ids=["pcs_01", "bms_01"],  # 需同步的設備
)

cluster = ClusterController(
    config=cluster_config,
    system_controller=sys_ctrl,
    unified_manager=unified_mgr,
    redis_client=redis,
    on_promoted=on_promoted_callback,
    on_demoted=on_demoted_callback,
)

async with cluster:
    # ClusterController 會自動處理 leader election
    # 以 follower 模式啟動，等待成為 leader
    await asyncio.Event().wait()
```

`ClusterController` 的核心設計是**角色切換**：

- **Leader**：完整管線 — 連接 Modbus 設備、執行保護評估、下發命令、同步狀態到 Redis
- **Follower**：虛擬管線 — 從 Redis 讀取設備狀態、策略 dry-run、不連接設備

這個角色切換是動態的，透過 etcd leader election 來決定。

---

## 4. Split-Brain：工業控制系統的噩夢

Split-brain 是分散式系統中最危險的故障情境。在工業控制中，它的後果尤其嚴重。

### 什麼是 Split-Brain？

想像這個場景：

```
       ┌─── Node A (Leader) ───┐
       │  "我是 Leader"         │
       │  → 送出 P=100kW 充電  │
       └────────────────────────┘
                    ╳  ← 網路分區
       ┌─── Node B (Follower) ──┐
       │  "Leader 消失了！"      │
       │  "我升級為 Leader"      │
       │  → 送出 P=-50kW 放電   │
       └────────────────────────┘
```

兩個節點同時認為自己是 Leader，同時對同一台 PCS 發送矛盾的功率命令。結果：PCS 在充電和放電之間快速切換，電池承受極大的電流衝擊。

### 為什麼簡單的 Active-Standby 不夠？

許多人的第一反應是：「用心跳偵測，主機掛了備機接手就好。」但問題是：

1. **心跳超時不等於主機掛了**：可能只是網路延遲
2. **接手的時機很難判斷**：太快 → 可能造成 split-brain；太慢 → 停機時間太長
3. **沒有仲裁者**：兩個節點自己無法判斷對方是否真的掛了

### Lease-Based Leadership with Fencing

csp_lib 採用的是基於 etcd lease 的選舉機制，搭配 self-fencing：

```python
# 選舉的核心邏輯
async def _try_campaign(self) -> None:
    # 1. 向 etcd 申請一個有 TTL 的 lease
    self._lease = await self._client.lease_grant(self._config.lease_ttl)

    # 2. 原子操作：如果 election key 不存在 → 寫入我的 ID
    success = await self._client.txn_put_if_not_exists(
        election_key, value, lease_id
    )

    if success:
        # 我是 leader，開始 keepalive 續租
        self._state = ElectionState.LEADER
        self._keepalive_task = asyncio.create_task(
            self._keepalive_loop(lease_id)
        )
    else:
        # 別人先搶到了，我當 follower
        self._state = ElectionState.FOLLOWER
        await self._wait_for_leader_loss()
```

關鍵的 self-fencing 機制：**如果 leader 無法續租，它會主動放棄領導權**：

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
            logger.warning(f"Lease keepalive failed ({consecutive_failures}/{max_failures})")
            if consecutive_failures >= max_failures:
                logger.error("Lease keepalive failed too many times, self-fencing")
                await self._handle_demotion()
                return
```

這個設計的核心思想是：**如果我不確定我還是 leader，我就不應該繼續發送命令**。在工業系統中，「不確定就停」永遠比「不確定就繼續」安全。

---

## 5. 優雅降級：不是全有就是全無

工業系統不能像 Web 一樣，一個服務掛了就回傳 503。我們需要的是**優雅降級**（Graceful Degradation）——在部分元件故障時，系統以降低的能力繼續運作。

### 5.1 設備層級降級

當某台設備無回應時，控制器不應該停止整個控制迴路，而是跳過這台設備：

```python
# CommandRouter 的實際行為（概念）
for device_id, (point_name, value) in command_writes.items():
    device = registry.get_device(device_id)

    if not device.is_responsive:
        logger.warning(f"Skipping unresponsive device: {device_id}")
        continue

    await device.write(point_name, value)
```

在 `ContextBuilder` 中，如果某個設備讀不到值，會使用 `ContextMapping` 中定義的 `default`：

```python
from csp_lib.integration.schema import ContextMapping

mappings = [
    ContextMapping(
        device_id="bms_01",
        point_name="soc",
        context_field="soc",
        default=50.0,         # 讀不到 SOC 時，假設 50%
    ),
    ContextMapping(
        device_id="meter_01",
        point_name="active_power",
        context_field="extra.meter_power",
        default=0.0,          # 讀不到電表時，假設 0
    ),
]
```

### 5.2 資料庫層級降級

MongoDB 掛了？資料寫入可以先緩衝在本地。Redis 掛了？叢集狀態同步暫停，但 Leader 的控制迴路繼續運作。

csp_lib 的 `ClusterStatePublisher` 在發佈失敗時不會停止控制迴路：

```python
async def _publish_loop(self) -> None:
    interval = self._config.state_publish_interval
    while not self._stop_event.is_set():
        try:
            await self._publish_all()
        except asyncio.CancelledError:
            return
        except Exception:
            # 發佈失敗只記錄 log，不中斷控制迴路
            logger.exception("Failed to publish cluster state")

        # 等待下一次發佈
        try:
            await asyncio.wait_for(self._stop_event.wait(), timeout=interval)
            return
        except asyncio.TimeoutError:
            pass
```

### 5.3 網路分區降級

當控制器與 etcd 之間網路分區時，leader 的 keepalive 會失敗。根據 `max_keepalive_failures` 設定，leader 會在可控的時間內自動降級：

```
TTL = 10 秒
keepalive 間隔 = TTL / 3 ≈ 3.3 秒
max_keepalive_failures = 3

最遲在 3.3 × 3 ≈ 10 秒後，leader 會自動 self-fence
```

降級後，系統進入「安全模式」：停止發送命令，等待網路恢復。

---

## 6. 故障恢復的正確姿勢

故障恢復不是簡單的「重新啟動」。在工業系統中，恢復有嚴格的順序：

### Step 1：確認安全狀態

在恢復控制之前，必須確認所有設備的當前狀態。如果設備在故障期間進入了保護模式，直接下發命令可能造成衝擊。

### Step 2：Grace Period

csp_lib 在 `ClusterController` 的 promotion 流程中，有一個 `failover_grace_period`：

```python
async def _handle_elected(self) -> None:
    # 1. 啟動設備管理器，連接設備
    await self._unified_manager.start()

    # 2. 等待 grace period，讓設備產生新資料
    await asyncio.sleep(self._config.failover_grace_period)

    # 3. 確認資料穩定後，切換到 live 模式
    self._enter_leader_mode()

    # 4. 啟動狀態發佈
    publisher = ClusterStatePublisher(...)
    await publisher.start()
```

### Step 3：漸進恢復

不要一口氣恢復所有設備的控制。先恢復監控（只讀），確認資料正確後，再恢復寫入控制。

---

## 7. 重點回顧

1. **工業 HA 的代價不同**：Web 掛了是體驗問題，工業掛了是安全問題
2. **故障類型多樣**：設備通訊、控制器節點、資料庫、網路分區，各有不同的應對策略
3. **三層防禦**：設備層（斷路器、狀態追蹤）、控制器層（保護規則）、系統層（叢集協調）
4. **Split-brain 必須防範**：lease-based election + self-fencing 是工業場景的標準做法
5. **優雅降級而非全面停止**：部分故障時系統以降低能力繼續運作
6. **恢復要謹慎**：先確認安全，等待 grace period，再漸進恢復

在下一篇文章中，我們將深入 etcd leader election 的實作細節，看看 csp_lib 的 `LeaderElector` 是如何實現可靠的主從切換的。

---

> **下一篇**：[Article 20 — etcd Leader Election：分散式控制器的主從切換](./20-etcd-leader-election.md)
