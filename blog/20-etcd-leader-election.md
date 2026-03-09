# etcd Leader Election：分散式控制器的主從切換

> **從零打造工業控制協定框架：給軟體工程師的 CSP Library 實戰系列**
> Part 5 — 高可用篇 | Article 20

上一篇我們談了工業系統高可用性的全貌。這一篇我們深入到最核心的技術問題：**當你有多個控制器節點時，誰來決定哪一個是 Leader？**

這個問題在分散式系統中被稱為 Leader Election，它是所有叢集高可用方案的基石。選錯了技術，你的 HA 就是紙上談兵。

---

## 1. 為什麼選 etcd？

在實作 leader election 之前，我們先比較幾個候選方案：

| 方案 | 一致性模型 | Leader Election 支援 | 運維複雜度 | 適合場景 |
|------|-----------|---------------------|-----------|---------|
| **etcd** | 強一致性 (Raft) | 原生 lease + txn | 中等 | K8s 生態、小規模叢集 |
| ZooKeeper | 強一致性 (ZAB) | 原生 ephemeral node | 高 | 大數據生態 (Kafka, HBase) |
| Redis | 最終一致性 / Redlock | 需自行實作 | 低 | 快取、Pub/Sub |
| Consul | 強一致性 (Raft) | 原生 session + KV | 中等 | 服務發現、多 DC |

csp_lib 選擇 etcd 的理由：

1. **強一致性是剛需**：工業控制不能容忍「兩個 leader 同時存在一小段時間」。etcd 基於 Raft 協定，保證 linearizable reads，不會出現 stale read 導致的 split-brain。

2. **Lease 機制天生適合 leader election**：etcd 的 lease 有明確的 TTL 語意——lease 到期，key 自動刪除。不需要額外的 session 管理或心跳協定。

3. **Transaction 支援原子比較-交換**：`txn_put_if_not_exists` 是一個原子操作，保證只有一個節點能搶到 election key。

4. **輕量部署**：etcd 是單一二進位檔，不需要 JVM，資源佔用遠低於 ZooKeeper。對於工業邊緣計算場景（通常是低規格工業電腦），這是重要考量。

5. **Kubernetes 生態**：如果你的工業控制系統部署在 K8s 上，etcd 已經在那裡了。

為什麼不用 Redis？雖然 csp_lib 已經用 Redis 做狀態同步，但 Redis 的一致性模型不適合 leader election。Redlock 演算法在時鐘偏移的情況下有已知的安全問題（Martin Kleppmann 在 2016 年的分析），而工業電腦的硬體時鐘品質通常不太可靠。

---

## 2. etcd Lease-Based Election 演算法

在進入 csp_lib 的程式碼之前，讓我們先理解 etcd lease-based election 的原理。

### 核心流程

```
┌──────────────────────────────────────────────────────┐
│                    etcd Cluster                      │
│                                                      │
│   election_key: "/csp/cluster/election"              │
│                                                      │
└───────────────────────┬──────────────────────────────┘
                        │
    ┌───────────────────┼───────────────────┐
    │                   │                   │
┌───┴────┐         ┌───┴────┐         ┌───┴────┐
│ Node A │         │ Node B │         │ Node C │
│        │         │        │         │        │
│ Step 1 │         │ Step 1 │         │ Step 1 │
│ Grant  │         │ Grant  │         │ Grant  │
│ Lease  │         │ Lease  │         │ Lease  │
│ TTL=10 │         │ TTL=10 │         │ TTL=10 │
│        │         │        │         │        │
│ Step 2 │         │ Step 2 │         │ Step 2 │
│ TXN:   │         │ TXN:   │         │ TXN:   │
│ IF key │         │ IF key │         │ IF key │
│ NOT    │         │ NOT    │         │ NOT    │
│ EXISTS │         │ EXISTS │         │ EXISTS │
│ → PUT  │         │ → PUT  │         │ → PUT  │
│        │         │        │         │        │
│ WIN! ✓ │         │ FAIL ✗ │         │ FAIL ✗ │
│ LEADER │         │FOLLOWER│         │FOLLOWER│
└────────┘         └────────┘         └────────┘
```

### 四個步驟

**Step 1: Grant Lease**

向 etcd 申請一個有 TTL 的 lease。Lease 是 etcd 的核心概念——它是一個有期限的「租約」，key 可以綁定到 lease 上，lease 到期時 key 自動被刪除。

```
LeaseGrant(TTL=10) → lease_id=12345
```

**Step 2: Atomic Transaction**

使用 etcd 的 transaction API，嘗試原子地寫入 election key：

```
IF key("/csp/cluster/election") NOT EXISTS:
    PUT("/csp/cluster/election", "node-a@host1", lease=12345)
```

這個操作是原子的。即使三個節點同時嘗試，只有一個會成功。

**Step 3a: 成功 → LEADER**

搶到 election key 的節點成為 Leader。它需要持續續租（keepalive），防止 lease 到期：

```
每 TTL/3 秒 → LeaseKeepAlive(lease_id=12345)
```

為什麼是 TTL/3？這給了 3 次續租機會。即使 1 次失敗，還有 2 次機會在 lease 到期前成功續租。

**Step 3b: 失敗 → FOLLOWER**

沒有搶到的節點成為 Follower。它會 watch election key，等待 key 被刪除（即 leader 的 lease 到期或主動 resign），然後重新競選。

---

## 3. csp_lib 的 LeaderElector 實作

現在讓我們看 csp_lib 是如何把上述演算法包裝成可用的元件的。

### 3.1 狀態機

`LeaderElector` 是一個四狀態的狀態機：

```python
class ElectionState(enum.Enum):
    CANDIDATE = "candidate"    # 剛啟動，準備競選
    LEADER = "leader"          # 已當選為 leader
    FOLLOWER = "follower"      # 競選失敗，追隨 leader
    STOPPED = "stopped"        # 已停止
```

狀態轉換圖：

```
STOPPED ──(start)──→ CANDIDATE ──(txn success)──→ LEADER
                         │                           │
                         │                    (keepalive fail ×N)
                         │                    (resign)
                         │                           │
                         └──(txn fail)──→ FOLLOWER ←─┘
                                             │
                                      (leader key 消失)
                                             │
                                         CANDIDATE ──→ ...
```

### 3.2 配置

`LeaderElector` 透過 `ClusterConfig` 配置：

```python
from csp_lib.cluster import ClusterConfig, EtcdConfig

config = ClusterConfig(
    instance_id="node-1",                       # 唯一實例 ID
    etcd=EtcdConfig(
        endpoints=["etcd1:2379", "etcd2:2379", "etcd3:2379"],
        username="root",                         # 可選認證
        password="secret",
        ca_cert="/etc/ssl/etcd-ca.crt",          # 可選 TLS
    ),
    election_key="/csp/cluster/election",        # etcd 選舉 key
    lease_ttl=10,                                # Lease TTL 秒數
    max_keepalive_failures=3,                    # 續租失敗上限
    campaign_retry_delay=2.0,                    # 競選失敗重試間隔
)
```

每個參數都有其工程考量：

- **`lease_ttl=10`**：太短會增加網路抖動導致誤判的風險；太長會延長 failover 時間。10 秒在工業場景中是常見的折衷。
- **`max_keepalive_failures=3`**：搭配 TTL/3 的續租間隔，意味著 leader 最多有 ~10 秒的時間嘗試續租。
- **`campaign_retry_delay=2.0`**：競選失敗後不立即重試，避免對 etcd 產生風暴。

### 3.3 啟動流程

```python
from csp_lib.cluster import LeaderElector

elector = LeaderElector(
    config=config,
    on_elected=handle_elected,
    on_demoted=handle_demoted,
)

# 方式一：手動管理
await elector.start()
# ... 做其他事 ...
await elector.stop()

# 方式二：async context manager（推薦）
async with elector:
    await asyncio.Event().wait()  # 永久運行
```

`_on_start` 做兩件事：初始化 etcd client，然後啟動 campaign loop：

```python
async def _on_start(self) -> None:
    self._stop_event.clear()
    self._state = ElectionState.CANDIDATE

    self._client = self._create_etcd_client()

    logger.info(f"LeaderElector started: instance={self._config.instance_id}")
    self._campaign_task = asyncio.create_task(self._campaign_loop())
```

注意 `_campaign_loop` 是一個 `asyncio.Task`，不會阻塞 `start()` 的返回。這意味著 `start()` 返回後，選舉可能還在進行中。caller 應該透過 callback 或輪詢 `elector.is_leader` 來得知結果。

### 3.4 Campaign Loop

Campaign loop 是 `LeaderElector` 的核心迴圈。它會持續嘗試競選，直到被停止：

```python
async def _campaign_loop(self) -> None:
    """持續嘗試競選 leader"""
    while not self._stop_event.is_set():
        try:
            await self._try_campaign()
        except asyncio.CancelledError:
            return
        except Exception:
            logger.exception(
                f"Campaign failed, retrying in {self._config.campaign_retry_delay}s"
            )
            try:
                await asyncio.wait_for(
                    self._stop_event.wait(),
                    timeout=self._config.campaign_retry_delay,
                )
                return  # stopped
            except asyncio.TimeoutError:
                pass
```

這裡有一個精巧的模式：用 `asyncio.wait_for(self._stop_event.wait(), timeout=delay)` 來實現「可中斷的 sleep」。如果在等待期間 `stop()` 被呼叫（設定了 `_stop_event`），等待會立即結束，避免延遲關閉。

### 3.5 競選嘗試

`_try_campaign` 是單次競選的完整流程：

```python
async def _try_campaign(self) -> None:
    if self._client is None:
        return

    # 1. Grant lease
    self._lease = await self._client.lease_grant(self._config.lease_ttl)
    lease_id = self._lease

    # 2. 組裝 value（包含 hostname，方便診斷）
    election_key = self._config.election_key
    instance_id = self._config.instance_id
    hostname = socket.gethostname()
    value = f"{instance_id}@{hostname}"

    # 3. 原子 compare-and-swap
    success = await self._client.txn_put_if_not_exists(
        election_key, value, lease_id
    )

    if success:
        # === 我是 LEADER ===
        self._state = ElectionState.LEADER
        self._current_leader_id = instance_id
        logger.info(f"Elected as leader: {instance_id}")

        # 觸發 on_elected callback
        if self._on_elected is not None:
            await self._on_elected()

        # 啟動 keepalive 和 watch
        self._keepalive_task = asyncio.create_task(
            self._keepalive_loop(lease_id)
        )
        self._watch_task = asyncio.create_task(
            self._watch_leader_key()
        )

        # 阻塞直到被 demoted 或 stopped
        await self._stop_event.wait()
    else:
        # === 我是 FOLLOWER ===
        # 讀取目前的 leader
        current_value = await self._client.get(election_key)
        if current_value is not None:
            self._current_leader_id = (
                current_value.split("@")[0]
                if "@" in current_value
                else current_value
            )
        self._state = ElectionState.FOLLOWER
        logger.info(f"Following leader: {self._current_leader_id}")

        # 撤銷自己的 lease（不需要了）
        try:
            await self._client.lease_revoke(lease_id)
        except Exception:
            pass
        self._lease = None

        # 等待 leader 消失
        await self._wait_for_leader_loss()
```

幾個值得注意的設計：

1. **value 包含 hostname**：`node-1@worker-03` 格式，方便在 etcd 中直接看到 leader 是哪台機器，不需要另外查詢。

2. **失敗時主動撤銷 lease**：follower 不需要持有 lease，提前釋放可以避免 etcd 的 lease 數量膨脹。

3. **Leader 路徑結尾是 `await self._stop_event.wait()`**：這意味著 `_try_campaign` 只有在 leader 被 demoted 或 stopped 時才會返回，返回後 `_campaign_loop` 會重新嘗試競選。

---

## 4. Keepalive Loop：續租與自保

Leader 當選後，最重要的工作就是持續續租。如果 lease 到期，election key 會被自動刪除，follower 就會嘗試搶佔。

```python
async def _keepalive_loop(self, lease_id: int) -> None:
    interval = max(self._config.lease_ttl / 3, 1.0)
    consecutive_failures = 0
    max_failures = self._config.max_keepalive_failures

    while not self._stop_event.is_set():
        # 可中斷的等待
        try:
            await asyncio.wait_for(
                self._stop_event.wait(), timeout=interval
            )
            return  # stopped
        except asyncio.TimeoutError:
            pass

        # 嘗試續租
        try:
            await self._client.lease_keepalive(lease_id)
            consecutive_failures = 0
        except asyncio.CancelledError:
            return
        except Exception:
            consecutive_failures += 1
            logger.warning(
                f"Lease keepalive failed ({consecutive_failures}/{max_failures})"
            )
            if consecutive_failures >= max_failures:
                logger.error(
                    "Lease keepalive failed too many times, self-fencing"
                )
                await self._handle_demotion()
                return
```

### 時間線分析

假設 `lease_ttl=10`，`max_keepalive_failures=3`：

```
t=0s   : 當選 leader，lease 剩餘 10s
t=3.3s : 第 1 次 keepalive → 成功 → lease 重置為 10s
t=6.6s : 第 2 次 keepalive → 失敗 → consecutive_failures=1
t=9.9s : 第 3 次 keepalive → 失敗 → consecutive_failures=2
t=10s  : lease 到期！etcd 自動刪除 election key
t=13.2s: 第 4 次 keepalive → 失敗 → consecutive_failures=3 → SELF-FENCE
```

注意兩個安全網：

1. **etcd 側**：即使 leader 程式碼有 bug 沒有 self-fence，lease 到期後 etcd 會自動刪除 key，follower 可以接管。
2. **Leader 側**：`max_keepalive_failures` 讓 leader 主動放棄，比等待 lease 自然到期更快。

### 為什麼不用 etcd Watch API？

你可能注意到 `_wait_for_leader_loss` 使用的是輪詢（polling）而不是 etcd 的 watch API：

```python
async def _wait_for_leader_loss(self) -> None:
    election_key = self._config.election_key
    poll_interval = max(self._config.lease_ttl / 2, 1.0)

    while not self._stop_event.is_set():
        try:
            await asyncio.wait_for(
                self._stop_event.wait(), timeout=poll_interval
            )
            return
        except asyncio.TimeoutError:
            pass

        try:
            value = await self._client.get(election_key)
            if value is None:
                logger.info("Leader key disappeared, re-campaigning")
                self._state = ElectionState.CANDIDATE
                return
        except Exception:
            logger.exception("Error polling leader key")
```

這是一個刻意的工程取捨。Watch API 的延遲更低，但也更複雜（需要處理 watch connection 斷線、revision compaction 等）。在工業場景中，failover 時間的差異（poll 間隔 ~5s vs watch ~瞬間）通常可以接受，而簡單可靠的實作更重要。

---

## 5. Self-Fencing：防止 Split-Brain 的最後防線

Self-fencing 是 csp_lib leader election 中最重要的安全機制。讓我們仔細分析它為什麼有效。

### 問題場景

```
t=0   : Node A 是 leader，正常運作
t=1   : Node A ↔ etcd 之間發生網路分區
t=1-10: Node A 無法續租，但它不知道 lease 是否已到期
t=10  : etcd 上的 lease 到期，key 被刪除
t=11  : Node B 成功競選為新 leader
```

如果 Node A 在 t=1 到 t=10 之間繼續發送命令，而 Node B 在 t=11 也開始發送命令，就可能出現短暫的 split-brain。

### Self-Fencing 的解法

```python
async def _handle_demotion(self) -> None:
    if self._state != ElectionState.LEADER:
        return

    self._state = ElectionState.FOLLOWER
    self._current_leader_id = None
    logger.warning("Demoted from leader")

    if self._on_demoted is not None:
        await self._on_demoted()
```

在 `ClusterController` 中，`on_demoted` callback 會立即停止控制迴路：

```python
async def _handle_demoted(self) -> None:
    logger.info("Demoting to follower...")

    # 1. 停止狀態發佈
    if self._publisher is not None:
        await self._publisher.stop()
        self._publisher = None

    # 2. 切換 executor 到 follower 模式（不再下發命令）
    self._enter_follower_mode()

    # 3. 停止設備管理器（斷開 Modbus 連線）
    await self._unified_manager.stop()
```

關鍵在 `_enter_follower_mode()`：

```python
def _enter_follower_mode(self) -> None:
    executor = self._system_controller.executor
    if self._virtual_builder is not None:
        executor.set_context_provider(self._virtual_builder.build)
    executor.set_on_command(self._noop_command_handler)
    logger.debug("Executor switched to follower mode (virtual context, no-op command).")

@staticmethod
async def _noop_command_handler(command) -> None:
    """Follower 的 no-op 命令處理器"""
    pass
```

**命令處理器被替換為 no-op**。即使策略還在執行，產生的命令也不會被發送到任何設備。這是一個非常乾淨的 fencing 實作——不需要在每個寫入操作前檢查 leader 狀態，只要在源頭把水龍頭關掉就好了。

### 時間窗口分析

在最壞的情況下，self-fencing 的時間窗口是多少？

```
keepalive 間隔 = lease_ttl / 3
self-fence 延遲 = keepalive 間隔 × max_keepalive_failures

以 lease_ttl=10, max_keepalive_failures=3:
self-fence 延遲 ≈ 3.3 × 3 ≈ 10 秒
```

而 follower 偵測到 leader 消失的時間：

```
follower poll 間隔 = lease_ttl / 2 = 5 秒
最壞情況：lease 剛好在 poll 之後到期 → 再等 5 秒

follower 偵測延遲 ≈ 5 + lease_ttl = 15 秒
```

所以 self-fencing（~10 秒）會在 follower 接管（~15 秒）之前完成，避免了 split-brain。

---

## 6. Callback Hooks：與應用邏輯整合

`LeaderElector` 透過兩個 callback 與應用邏輯整合：

```python
async def on_elected():
    """成為 leader 後的處理"""
    logger.info("I am now the leader!")

    # 啟動控制迴路
    await system_controller.start()

    # 啟動資料上傳
    await data_upload_manager.start()

    # 通知運維
    await notify_ops("Node promoted to leader")

async def on_demoted():
    """失去 leader 後的處理"""
    logger.warning("I am no longer the leader!")

    # 停止控制迴路
    await system_controller.stop()

    # 停止資料上傳
    await data_upload_manager.stop()

    # 通知運維
    await notify_ops("Node demoted to follower")

elector = LeaderElector(
    config=config,
    on_elected=on_elected,
    on_demoted=on_demoted,
)
```

在 `ClusterController` 中，這些 callback 被用來進行完整的角色切換。但你也可以直接使用 `LeaderElector`，搭配自己的邏輯。

### 主動 Resign

有時候你需要主動讓出 leader：例如在維護時、或在偵測到自身異常時：

```python
# 主動讓出 leader
await elector.resign()

# resign 會撤銷 lease，etcd 上的 election key 立即被刪除
# follower 會立即偵測到 leader 消失，開始競選
```

`resign` 的實作很直接——撤銷 lease：

```python
async def resign(self) -> None:
    """主動辭去 leader 角色"""
    if self._state != ElectionState.LEADER:
        return
    await self._resign_internal()

async def _resign_internal(self) -> None:
    if self._lease is not None and self._client is not None:
        try:
            await self._client.lease_revoke(self._lease)
            logger.info("Lease revoked (resigned).")
        except Exception:
            logger.exception("Failed to revoke lease during resign")
        self._lease = None
```

---

## 7. 測試 Leader Election

測試分散式選舉很有挑戰性，但 `LeaderElector` 的設計讓它可以被單元測試。關鍵是 `_create_etcd_client` 可以被覆寫：

```python
import asyncio
import pytest
from unittest.mock import AsyncMock, MagicMock
from csp_lib.cluster import LeaderElector, ElectionState, ClusterConfig, EtcdConfig


class MockEtcdClient:
    """模擬 etcd 客戶端"""

    def __init__(self):
        self._store: dict[str, str] = {}
        self._leases: dict[int, float] = {}
        self._next_lease_id = 1

    async def lease_grant(self, ttl: int) -> int:
        lease_id = self._next_lease_id
        self._next_lease_id += 1
        self._leases[lease_id] = ttl
        return lease_id

    async def txn_put_if_not_exists(self, key: str, value: str, lease_id: int) -> bool:
        if key not in self._store:
            self._store[key] = value
            return True
        return False

    async def get(self, key: str) -> str | None:
        return self._store.get(key)

    async def lease_keepalive(self, lease_id: int) -> None:
        if lease_id not in self._leases:
            raise RuntimeError("Lease not found")

    async def lease_revoke(self, lease_id: int) -> None:
        self._leases.pop(lease_id, None)

    async def close(self) -> None:
        pass


class TestableElector(LeaderElector):
    """可測試版本的 LeaderElector"""

    def __init__(self, config, mock_client, **kwargs):
        super().__init__(config, **kwargs)
        self._mock_client = mock_client

    def _create_etcd_client(self):
        return self._mock_client


@pytest.mark.asyncio
async def test_single_node_becomes_leader():
    """單一節點應該直接成為 leader"""
    config = ClusterConfig(instance_id="node-1", lease_ttl=10)
    mock_client = MockEtcdClient()
    elected = asyncio.Event()

    async def on_elected():
        elected.set()

    elector = TestableElector(
        config=config,
        mock_client=mock_client,
        on_elected=on_elected,
    )

    await elector.start()

    # 等待選舉完成
    await asyncio.wait_for(elected.wait(), timeout=2.0)

    assert elector.is_leader
    assert elector.state == ElectionState.LEADER
    assert elector.current_leader_id == "node-1"

    await elector.stop()


@pytest.mark.asyncio
async def test_second_node_becomes_follower():
    """第二個節點應該成為 follower"""
    mock_client = MockEtcdClient()

    config_a = ClusterConfig(instance_id="node-a", lease_ttl=10)
    config_b = ClusterConfig(instance_id="node-b", lease_ttl=10)

    elected_a = asyncio.Event()
    elector_a = TestableElector(
        config=config_a, mock_client=mock_client, on_elected=lambda: elected_a.set(),
    )

    await elector_a.start()
    await asyncio.wait_for(elected_a.wait(), timeout=2.0)
    assert elector_a.is_leader

    # Node B 加入 —— 應該成為 follower
    elector_b = TestableElector(config=config_b, mock_client=mock_client)
    await elector_b.start()
    await asyncio.sleep(0.5)

    assert not elector_b.is_leader
    assert elector_b.state == ElectionState.FOLLOWER
    assert elector_b.current_leader_id == "node-a"

    await elector_a.stop()
    await elector_b.stop()
```

### 測試 Failover

```python
@pytest.mark.asyncio
async def test_failover_on_leader_resign():
    """Leader resign 後，follower 應該接管"""
    mock_client = MockEtcdClient()

    config_a = ClusterConfig(instance_id="node-a", lease_ttl=10)
    config_b = ClusterConfig(instance_id="node-b", lease_ttl=10)

    elected_a = asyncio.Event()
    elector_a = TestableElector(
        config=config_a, mock_client=mock_client, on_elected=lambda: elected_a.set(),
    )
    await elector_a.start()
    await asyncio.wait_for(elected_a.wait(), timeout=2.0)

    elected_b = asyncio.Event()
    elector_b = TestableElector(
        config=config_b, mock_client=mock_client, on_elected=lambda: elected_b.set(),
    )
    await elector_b.start()
    await asyncio.sleep(0.5)
    assert elector_b.state == ElectionState.FOLLOWER

    # Leader A resign
    await elector_a.resign()
    # 模擬 etcd 刪除 key
    mock_client._store.pop("/csp/cluster/election", None)

    # 等待 B 接管
    await asyncio.wait_for(elected_b.wait(), timeout=15.0)
    assert elector_b.is_leader

    await elector_a.stop()
    await elector_b.stop()
```

---

## 8. 生產環境注意事項

### etcd 叢集部署

etcd 本身也需要高可用。生產環境建議：

- **3 或 5 個 etcd 節點**：Raft 需要多數派存活，3 節點容忍 1 節點故障，5 節點容忍 2 節點故障
- **分散部署**：不要把所有 etcd 節點放在同一台機器或同一個機櫃
- **SSD 磁碟**：etcd 對磁碟延遲很敏感，務必使用 SSD
- **定期備份**：`etcdctl snapshot save`

### 監控

在生產環境中，你需要監控以下指標：

```python
# ClusterController 提供的健康資訊
health = cluster.health()
# {
#     "role": "leader",
#     "instance_id": "node-1",
#     "is_leader": True,
#     "leader_id": "node-1",
#     "unified_manager_running": True,
#     "system_controller_running": True,
# }
```

建議監控的警報：

- **Leader 切換**：每次 leader 變化都應該觸發告警
- **Follower 數量為 0**：只有一個節點在運作，沒有備援
- **keepalive 失敗次數**：接近 `max_keepalive_failures` 時預警
- **Failover 頻率**：頻繁切換可能表示 etcd 或網路不穩定

---

## 9. 重點回顧

1. **etcd 適合工業場景**：強一致性 + 原生 lease + 輕量部署
2. **Lease-based election 的核心**：Grant Lease → Atomic TXN → Keepalive Loop
3. **TTL/3 續租間隔**：給 3 次機會在 lease 到期前成功續租
4. **Self-fencing 是安全網**：leader 無法續租時主動放棄，比等 lease 自然到期更快
5. **no-op command handler**：follower 模式下在源頭切斷命令，而非在每個寫入點檢查
6. **可測試設計**：透過覆寫 `_create_etcd_client` 注入 mock

在下一篇文章中，我們將看到 Leader 和 Follower 之間如何透過 Redis 同步狀態，以及 `VirtualContextBuilder` 如何讓 Follower 隨時準備好接管。

---

> **下一篇**：[Article 21 — Redis Streams 作為資料平面：Leader-Follower 狀態同步](./21-redis-data-plane.md)
