# 反應的藝術：Observer × Chain of Responsibility × State

> **從零打造工業控制協定框架：給軟體工程師的 CSP Library 實戰系列**
>
> Part 0.5 — 設計模式活用篇 | Article 00g
>
> [<<< 上一篇：資料的旅程：Adapter × Pipeline × Builder](./00f-patterns-data-flow.md) | [下一篇：組裝的智慧：Factory × Registry × Mediator × Facade >>>](./00h-patterns-assembly.md)

---

## 目錄

1. [開場：工業系統為什麼需要「反應式」設計](#開場工業系統為什麼需要反應式設計)
2. [Observer Pattern — 事件驅動的設備監控](#observer-pattern--事件驅動的設備監控)
3. [Chain of Responsibility — 告警的層層過濾](#chain-of-responsibility--告警的層層過濾)
4. [State Pattern — 叢集選舉的狀態機](#state-pattern--叢集選舉的狀態機)
5. [Circuit Breaker — 通訊故障的自我保護](#circuit-breaker--通訊故障的自我保護)
6. [四個模式如何形成一個反應式系統](#四個模式如何形成一個反應式系統)
7. [重點回顧](#重點回顧)
8. [下篇預告](#下篇預告)

---

## 開場：工業系統為什麼需要「反應式」設計

想像你是一座儲能電站的調度員。你眼前有三十台設備的監控螢幕，電池的 SOC 不斷跳動、電表的功率隨負載波動、PCS 偶爾因為過溫觸發告警。你不可能每秒鐘逐一巡視三十台設備——你需要的是：當重要的事情發生時，系統主動通知你。

這就是「反應式設計」的核心思想：**不要輪詢，要訂閱；不要主動查，要被動收。**

在工業控制場域，反應式設計不只是效能優化，更是安全需求。一個 SOC 超限告警如果延遲 3 秒才被處理，可能造成電池過充；一條 Modbus 連線如果持續失敗卻不斷重試，可能拖垮整條 RS-485 匯流排上的所有設備。

本篇介紹四個在 csp_lib 中協同運作的反應式模式：

- **Observer**：設備狀態變化時主動通知所有關注者
- **Chain of Responsibility**：功率指令通過一連串保護規則的逐層過濾
- **State**：叢集選舉中節點角色的狀態機轉換
- **Circuit Breaker**：通訊故障時的自動熔斷與恢復

它們不是各自為政，而是共同構成了一個「感知→判斷→行動→自我保護」的反應式閉環。

---

## Observer Pattern — 事件驅動的設備監控

### 日常比喻

你訂閱了一個 YouTube 頻道。從此，每當頻道發布新影片，YouTube 就會推播通知給你——你不需要每天手動打開頻道頁面檢查。一個頻道可以有千萬個訂閱者，發布一支新影片就能同時通知所有人。

這就是 Observer 模式：**發布者不需要知道訂閱者是誰，訂閱者只需要表達「我對這件事有興趣」。**

### 問題場景

假設你有一台 PCS 設備，你需要在以下情況做出反應：

- 數值變化時 → 更新 Dashboard
- 告警觸發時 → 發送 LINE 通知
- 斷線時 → 標記設備不可用

你的第一直覺可能是：在讀取循環裡直接呼叫這些邏輯。

### 直覺解法的問題

```python
# 反面教材：讀取循環裡塞滿各種回呼
async def read_loop(device):
    while True:
        values = await device.read()

        # 更新 Dashboard（為什麼讀取循環要知道 Dashboard？）
        await dashboard.update(values)

        # 檢查告警並發通知（讀取循環變成告警系統？）
        if values["soc"] > 95:
            await line_notify.send("SOC 過高！")

        # 寫入 MongoDB（讀取循環還負責儲存？）
        await mongo.insert(values)

        # 未來還要加 Redis 快取、WebSocket 推送、統計計算...
        # 這個函式會無限膨脹
```

問題很明顯：讀取循環承擔了太多不屬於它的職責，而且每加一個新需求就要修改核心程式碼。

### 模式登場：Observer

Observer 模式將「事件的發生」和「事件的處理」完全解耦。發布者（Subject）只負責發射事件，訂閱者（Observer）各自註冊自己感興趣的事件。

### csp_lib 真實程式碼

csp_lib 的 `DeviceEventEmitter`（位於 `csp_lib/equipment/device/events.py`）是 Observer 模式的完整實現。先看事件的資料結構：

```python
# csp_lib/equipment/device/events.py

# 事件名稱常數 — 避免字串打錯字
EVENT_CONNECTED = "connected"
EVENT_DISCONNECTED = "disconnected"
EVENT_VALUE_CHANGE = "value_change"
EVENT_ALARM_TRIGGERED = "alarm_triggered"
EVENT_ALARM_CLEARED = "alarm_cleared"
EVENT_WRITE_COMPLETE = "write_complete"
EVENT_READ_ERROR = "read_error"


@dataclass(frozen=True)
class ValueChangePayload:
    """值變化事件資料"""
    device_id: str
    point_name: str
    old_value: Any
    new_value: Any
    timestamp: datetime = field(default_factory=lambda: datetime.now(timezone.utc))
```

注意 Payload 是 frozen dataclass——事件一旦發出就不可修改，避免某個 handler 修改了 payload 影響到後續 handler。

再看事件發射器的核心：

```python
# csp_lib/equipment/device/events.py

AsyncHandler = Callable[[Any], Awaitable[None]]

class DeviceEventEmitter:
    def __init__(self, max_queue_size: int = 10000) -> None:
        self._handlers: dict[str, list[AsyncHandler]] = {}
        self._queue: asyncio.Queue[tuple[str, Any]] = asyncio.Queue(maxsize=max_queue_size)
        self._worker_task: asyncio.Task[None] | None = None
        self._running = False

    def on(self, event: str, handler: AsyncHandler) -> Callable[[], None]:
        """註冊事件處理器，回傳取消訂閱函式"""
        if event not in self._handlers:
            self._handlers[event] = []
        self._handlers[event].append(handler)

        def cancel() -> None:
            if event in self._handlers and handler in self._handlers[event]:
                self._handlers[event].remove(handler)
        return cancel

    def emit(self, event: str, payload: Any = None) -> None:
        """發射事件（非阻塞）"""
        if not self._handlers.get(event):
            return  # 沒有監聽器就不入隊
        try:
            self._queue.put_nowait((event, payload))
        except asyncio.QueueFull:
            logger.warning("事件佇列已滿，丟棄事件: event=%s", event)
```

幾個精妙的設計值得注意：

1. **`on()` 回傳取消函式**：訂閱者不需要保留 emitter 的引用就能取消訂閱，這讓生命週期管理更簡單。
2. **非阻塞入隊**：`emit()` 使用 `put_nowait()`，絕不阻塞讀取循環。事件由背景 worker 處理。
3. **無監聽器則跳過**：`if not self._handlers.get(event): return` 這行避免了無謂的佇列操作。
4. **佇列滿時丟棄而非阻塞**：工業系統寧可丟幾個事件，也不能讓讀取循環卡住。

背景 worker 的處理邏輯：

```python
    async def _worker(self) -> None:
        """事件處理 worker"""
        while self._running:
            try:
                event, payload = await asyncio.wait_for(self._queue.get(), timeout=1.0)
                await self._process_event(event, payload)
                self._queue.task_done()
            except asyncio.TimeoutError:
                continue
            except asyncio.CancelledError:
                break

    async def _process_event(self, event: str, payload: Any) -> None:
        """順序執行 handlers，避免並行造成資源競爭"""
        handlers = self._handlers.get(event, [])
        for handler in handlers:
            try:
                await handler(payload)
            except Exception:
                logger.opt(exception=True).warning(
                    "事件處理失敗: event={}, payload={}", event, repr(payload)
                )
```

handler 是**順序執行**而非並行——這是刻意的設計。在工業系統中，事件處理的順序很重要：你不希望「告警觸發」和「告警解除」的通知順序被打亂。

### 如果不用 Observer

```python
# 反面教材：沒有 Observer 的世界
class AsyncModbusDevice:
    def __init__(self, dashboard, notifier, mongo, redis, websocket, ...):
        # 設備類別知道所有下游系統
        self._dashboard = dashboard
        self._notifier = notifier
        self._mongo = mongo
        # ... 每加一個消費者就要改建構子

    async def _on_value_change(self, point, old, new):
        await self._dashboard.update(point, new)   # 硬耦合
        await self._notifier.check(point, new)      # 硬耦合
        await self._mongo.insert(point, new)         # 硬耦合
        # 測試時需要 mock 所有這些依賴
```

### 何時不該用

- **只有一個消費者**：如果某個事件永遠只有一個處理器，直接回呼比事件系統更簡單。
- **需要嚴格的處理保證**：Observer 的佇列滿時會丟棄事件。如果每個事件都必須被處理（如金融交易），應該用 message queue 而非記憶體內的事件系統。
- **效能敏感的熱路徑**：佇列入隊有微小開銷。如果是每秒百萬次的高頻操作，考慮直接回呼。

### 練習題

在 csp_lib 中找到以下 Observer 應用：
1. `AsyncModbusDevice` 是如何在讀取循環中呼叫 `emit()` 發射 `value_change` 事件的？
2. `AlarmPersistenceManager` 是如何訂閱 `alarm_triggered` 事件來持久化告警記錄的？

---

## Chain of Responsibility — 告警的層層過濾

### 日常比喻

你去醫院急診。護理師先做初步分流（量體溫、測血壓），如果沒問題就讓你去普通門診；如果發現血壓異常，轉給內科醫師；如果醫師判斷需要手術，再轉給外科。每一關都可能解決問題，也可能把你傳遞給下一關。

這就是 Chain of Responsibility：**請求沿著一條鏈傳遞，每個節點決定自己要不要處理或修改這個請求。**

### 問題場景

你的控制策略計算出了一個功率指令：`P = -50kW`（充電 50kW）。但在下發到設備之前，你需要做多項安全檢查：

1. SOC 已經 96%，不能再充電
2. 電表讀到逆送功率，放電量不能超過負載
3. 系統告警旗標被設定，應該全部停機

你的第一直覺可能是一個巨大的 if-else：

```python
# 反面教材：所有保護邏輯擠在一起
def apply_protection(command, context):
    if context.soc >= 95 and command.p < 0:
        command.p = 0  # SOC 保護
    if context.soc <= 5 and command.p > 0:
        command.p = 0  # SOC 保護
    if context.extra.get("meter_power") is not None:
        max_p = context.extra["meter_power"]
        if command.p > max_p:
            command.p = max_p  # 逆送保護
    if context.extra.get("system_alarm"):
        command.p = 0  # 告警保護
        command.q = 0
    return command
```

### 直覺解法的問題

- 每個案場的保護規則組合不同——有的不需要逆送保護，有的需要額外的頻率保護
- 新增一條規則需要修改這個函式，違反開放封閉原則
- 無法追蹤「哪條規則修改了命令」——調試噩夢
- 規則之間可能有順序依賴，但全部攤平在一個函式裡看不出來

### 模式登場：Chain of Responsibility

每條保護規則是鏈上的一個節點。命令從鏈頭進入，每個節點可以修改命令再傳給下一個，也可以選擇不介入直接放行。

### csp_lib 真實程式碼

在 `csp_lib/controller/system/protection.py` 中，保護鏈由三部分組成：抽象規則、具體規則、鏈管理器。

首先是抽象規則：

```python
# csp_lib/controller/system/protection.py

class ProtectionRule(ABC):
    """保護規則抽象基礎類別"""

    @property
    @abstractmethod
    def name(self) -> str:
        """規則名稱"""

    @abstractmethod
    def evaluate(self, command: Command, context: StrategyContext) -> Command:
        """評估並可能修改命令"""

    @property
    @abstractmethod
    def is_triggered(self) -> bool:
        """該規則是否處於觸發狀態（診斷用）"""
```

介面很精簡：輸入一個 Command，輸出一個 Command（可能是原始的，也可能是修改過的）。`is_triggered` 屬性是診斷用的——讓你知道這條規則是否介入了。

SOC 保護的實現展示了規則的深度：

```python
class SOCProtection(ProtectionRule):
    """
    SOC 保護
    P > 0 = 放電，P < 0 = 充電
    - SOC >= soc_high: 禁止充電 → clamp P >= 0
    - SOC <= soc_low: 禁止放電 → clamp P <= 0
    - 警戒區漸進限制: ratio = 距離上下限的比例
    """

    def evaluate(self, command: Command, context: StrategyContext) -> Command:
        soc = context.soc
        if soc is None:
            self._is_triggered = False
            return command  # 沒有 SOC 資訊時不介入

        cfg = self._config
        p = command.p_target

        # SOC 過高：禁止充電
        if soc >= cfg.soc_high:
            if p < 0:
                self._is_triggered = True
                return command.with_p(0.0)
            self._is_triggered = False
            return command

        # 高側警戒區：漸進限制充電
        warning_high = cfg.soc_high - cfg.warning_band
        if soc >= warning_high and cfg.warning_band > 0:
            if p < 0:
                ratio = (cfg.soc_high - soc) / cfg.warning_band
                limited_p = p * ratio
                self._is_triggered = True
                return command.with_p(limited_p)
        # ...（低側保護對稱省略）
```

注意「漸進限制」的設計：SOC 在 90-95% 之間不是硬切到零，而是隨著 SOC 接近上限逐步降低充電功率。這種「軟著陸」在工業控制中非常重要——硬切會造成功率震盪。

`command.with_p()` 回傳一個新的 Command 而非修改原始的——**不可變性**確保了鏈中的每個規則看到的都是前一個規則的輸出，不會有奇怪的副作用。

逆送保護是另一條獨立的規則：

```python
class ReversePowerProtection(ProtectionRule):
    """表後逆送保護：約束 p_target <= meter_power + threshold"""

    def evaluate(self, command: Command, context: StrategyContext) -> Command:
        meter_power = context.extra.get(self._meter_power_key)
        if meter_power is None:
            self._is_triggered = False
            return command

        p = command.p_target
        if p < 0:  # 充電不受逆送保護限制
            self._is_triggered = False
            return command

        max_discharge = meter_power + self._threshold
        if max_discharge < 0:
            max_discharge = 0.0

        if p > max_discharge:
            self._is_triggered = True
            return command.with_p(max_discharge)

        self._is_triggered = False
        return command
```

最後是鏈管理器 `ProtectionGuard`：

```python
class ProtectionGuard:
    """保護鏈：鏈式套用所有保護規則，追蹤觸發狀態"""

    def apply(self, command: Command, context: StrategyContext) -> ProtectionResult:
        original = command
        current = command
        triggered: list[str] = []

        for rule in self._rules:
            try:
                current = rule.evaluate(current, context)
                if rule.is_triggered:
                    triggered.append(rule.name)
            except Exception:
                logger.exception(f"Protection rule '{rule.name}' failed, "
                                 "applying fail-safe (P=0, Q=0)")
                current = Command(p_target=0.0, q_target=0.0)
                triggered.append(f"{rule.name}(fail-safe)")

        result = ProtectionResult(
            original_command=original,
            protected_command=current,
            triggered_rules=triggered,
        )
        return result
```

這段程式碼有一個極其重要的設計：**fail-safe**。如果任何規則拋出例外，不是跳過它繼續執行——而是立即將命令設為 `P=0, Q=0`。在工業安全的世界裡，「出錯就停」永遠比「出錯就繼續」安全。

`ProtectionResult` 紀錄了原始命令、保護後命令和觸發的規則列表。這讓調試變得容易：你可以在日誌中看到「SOC 保護和逆送保護同時觸發，P 從 50kW 被限制到 12kW」。

### 如果不用 Chain of Responsibility

```python
# 反面教材：單一函式包含所有保護邏輯
def apply_all_protections(command, context, site_config):
    if site_config.enable_soc_protection:
        # 20 行 SOC 邏輯
        ...
    if site_config.enable_reverse_power:
        # 15 行逆送邏輯
        ...
    if site_config.enable_alarm_protection:
        # 5 行告警邏輯
        ...
    if site_config.enable_frequency_protection:  # 新需求！
        # 又加 20 行...
        ...
    # 這個函式會長到 200 行，而且無法追蹤是哪段邏輯改了命令
    return command
```

### 何時不該用

- **規則數量極少且固定**：如果永遠只有 2 條規則，不會增加，直接寫一個函式更清楚。
- **規則之間有強依賴**：如果規則 B 的結果取決於規則 A 是否觸發（不只是 A 修改後的命令），Chain of Responsibility 的線性結構可能不夠表達——考慮用決策樹。
- **效能極度敏感**：每條規則是一次虛擬呼叫。如果是微秒級的控制循環，用直接計算比物件導向的鏈更快。

### 練習題

1. 在 `csp_lib/controller/system/protection.py` 中，`SystemAlarmProtection` 是如何強制覆蓋命令為 `P=0, Q=0` 的？
2. `ProtectionGuard.add_rule()` 和 `remove_rule()` 如何實現規則的動態增減？

---

## State Pattern — 叢集選舉的狀態機

### 日常比喻

紅綠燈有三個狀態：紅、黃、綠。每個狀態下的行為不同——綠燈時車輛通行，紅燈時車輛停止。而且狀態之間的轉換有嚴格規則：綠→黃→紅→綠，你不能從綠燈直接跳到紅燈（否則追撞連連）。

State 模式就是：**物件的行為隨著內部狀態改變，而且狀態轉換有明確的規則。**

### 問題場景

你的儲能系統採用主備架構，需要 leader election。節點有四個角色：

- CANDIDATE：正在競選
- LEADER：當選為主節點，負責控制
- FOLLOWER：追隨者，備援待命
- STOPPED：已停止

不同角色下的行為完全不同。你的第一直覺：

```python
# 反面教材：到處都是 if-else
class ElectionManager:
    def __init__(self):
        self.role = "stopped"

    async def handle_heartbeat(self):
        if self.role == "leader":
            await self.renew_lease()
        elif self.role == "follower":
            await self.check_leader()
        elif self.role == "candidate":
            await self.try_campaign()
        # stopped 時什麼都不做

    async def handle_stop(self):
        if self.role == "leader":
            await self.resign()
            self.role = "stopped"
        elif self.role == "follower":
            self.role = "stopped"
        elif self.role == "candidate":
            self.role = "stopped"
```

### 直覺解法的問題

- 每個方法都有一組 if-else，隨著狀態和行為增加，分支會指數爆炸
- 狀態轉換邏輯散落各處，難以一眼看出「LEADER 可以轉換到哪些狀態」
- 新增一個狀態需要修改所有方法

### 模式登場：State

將狀態定義為列舉，在物件內部維護狀態機，每次狀態轉換都是明確的、可追蹤的事件。

### csp_lib 真實程式碼

`csp_lib/cluster/election.py` 中的 `LeaderElector` 是一個完整的狀態機實現：

```python
# csp_lib/cluster/election.py

class ElectionState(enum.Enum):
    """選舉狀態"""
    CANDIDATE = "candidate"
    LEADER = "leader"
    FOLLOWER = "follower"
    STOPPED = "stopped"


class LeaderElector(AsyncLifecycleMixin):
    def __init__(
        self,
        config: ClusterConfig,
        on_elected: Callable[[], Awaitable[None]] | None = None,
        on_demoted: Callable[[], Awaitable[None]] | None = None,
    ) -> None:
        self._state = ElectionState.STOPPED
        self._on_elected = on_elected
        self._on_demoted = on_demoted
        # ...

    @property
    def is_leader(self) -> bool:
        return self._state == ElectionState.LEADER

    @property
    def state(self) -> ElectionState:
        return self._state
```

狀態轉換圖（用程式碼表達）：

```
STOPPED → CANDIDATE  (啟動時)
CANDIDATE → LEADER   (競選成功)
CANDIDATE → FOLLOWER (競選失敗，發現現有 leader)
LEADER → FOLLOWER    (被降級：lease 續約失敗)
LEADER → STOPPED     (主動停止)
FOLLOWER → CANDIDATE (leader 消失，重新競選)
FOLLOWER → STOPPED   (主動停止)
```

啟動時的狀態轉換：

```python
    async def _on_start(self) -> None:
        """啟動選舉流程"""
        self._stop_event.clear()
        self._state = ElectionState.CANDIDATE  # STOPPED → CANDIDATE
        self._client = self._create_etcd_client()
        self._campaign_task = asyncio.create_task(self._campaign_loop())
```

競選的核心邏輯展示了 CANDIDATE 到 LEADER 或 FOLLOWER 的分叉：

```python
    async def _try_campaign(self) -> None:
        """嘗試一次競選"""
        # Grant lease，嘗試以 transaction 取得 election key
        success = await self._client.txn_put_if_not_exists(
            election_key, value, lease_id
        )

        if success:
            # CANDIDATE → LEADER
            self._state = ElectionState.LEADER
            self._current_leader_id = instance_id
            logger.info(f"Elected as leader: {instance_id}")

            if self._on_elected is not None:
                await self._on_elected()  # 通知上層：我當選了

            self._keepalive_task = asyncio.create_task(self._keepalive_loop(lease_id))
            self._watch_task = asyncio.create_task(self._watch_leader_key())
            await self._stop_event.wait()
        else:
            # CANDIDATE → FOLLOWER
            self._state = ElectionState.FOLLOWER
            current_value = await self._client.get(election_key)
            if current_value is not None:
                self._current_leader_id = current_value.split("@")[0]
            logger.info(f"Following leader: {self._current_leader_id}")
            await self._wait_for_leader_loss()
```

降級處理（LEADER → FOLLOWER）：

```python
    async def _handle_demotion(self) -> None:
        """處理從 leader 降級"""
        if self._state != ElectionState.LEADER:
            return  # 防禦性檢查：只有 LEADER 可以被降級

        self._state = ElectionState.FOLLOWER  # LEADER → FOLLOWER
        self._current_leader_id = None
        logger.warning("Demoted from leader")

        if self._on_demoted is not None:
            await self._on_demoted()  # 通知上層：我被降級了
```

注意 `_handle_demotion` 開頭的防禦性檢查——`if self._state != ElectionState.LEADER: return`。這是 State 模式中很重要的一點：**非法的狀態轉換應該被拒絕或忽略**，而不是靜默地執行。

keepalive 失敗觸發自我降級的邏輯（Self-fencing）：

```python
    async def _keepalive_loop(self, lease_id: int) -> None:
        """持續更新 lease TTL"""
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
                    await self._handle_demotion()  # 觸發 LEADER → FOLLOWER
                    return
```

Self-fencing 是分散式系統的關鍵安全機制：如果我無法證明自己還是 leader（lease 續約失敗），我必須主動放棄，避免「腦裂」——兩個節點都認為自己是 leader。

### 如果不用 State 模式

```python
# 反面教材：boolean 旗標地獄
class ElectionManager:
    def __init__(self):
        self.is_leader = False
        self.is_follower = False
        self.is_candidate = False
        self.is_stopped = True  # 四個 bool 能表達 16 種狀態組合，但合法的只有 4 種

    async def campaign(self):
        if self.is_stopped:
            self.is_stopped = False
            self.is_candidate = True
            # 忘記設 is_leader = False 和 is_follower = False 了嗎？
```

用多個 boolean 表達互斥狀態是常見的 bug 來源。一個 Enum 天然保證了互斥性。

### 何時不該用

- **狀態只有兩個**：開/關、啟用/禁用——用一個 boolean 就好，不需要 Enum + State Machine。
- **狀態數量會動態增長**：如果狀態不是設計時就能確定的（例如使用者自定義的 workflow），考慮用通用的狀態機框架而非 hardcode 的 Enum。
- **沒有狀態相關的行為差異**：如果所有狀態下的行為都一樣，只是需要記錄「目前在哪」，一個 Enum 屬性就夠了，不需要完整的 State Pattern。

### 練習題

1. `LeaderElector._on_stop()` 中，停止時如何根據當前狀態決定是否需要 resign？
2. 在 Follower 狀態下，`_wait_for_leader_loss()` 是如何偵測 leader 消失並轉回 CANDIDATE 的？

---

## Circuit Breaker — 通訊故障的自我保護

### 日常比喻

你家的電箱裡有個「無熔絲開關」（NFB）。平常電流正常時，開關是閉合的，電流自由流通。但如果某個電器短路造成電流異常飆高，NFB 會自動跳脫（斷開），切斷電路保護整棟房子。等你找到故障的電器、修好之後，手動把 NFB 推回去，電路恢復。

Circuit Breaker 模式就是軟體版的 NFB：**連續失敗太多次就停止嘗試，冷卻一段時間後再試一次。**

### 問題場景

你的系統通過 Modbus TCP 與一台 PCS 通訊。某天，PCS 因為韌體 bug 停止回應，但 TCP 連線還在。每次讀取都要等 5 秒的超時才會失敗。你的讀取循環每秒一次，結果：

- 佇列裡堆積了幾百個請求
- 其他設備的讀取也被拖慢（共用同一條 RS-485 匯流排）
- 整個系統的延遲從 100ms 飆升到 30 秒

### 模式登場：Circuit Breaker

```
CLOSED → 連續失敗達閾值 → OPEN
OPEN → 冷卻時間過後 → HALF_OPEN
HALF_OPEN → 成功 → CLOSED
HALF_OPEN → 失敗 → OPEN
```

三個狀態：

- **CLOSED**（正常）：請求正常通過，記錄失敗次數
- **OPEN**（斷路）：拒絕所有請求，直接失敗，不浪費時間等超時
- **HALF_OPEN**（試探）：允許一個請求通過，成功則恢復，失敗則繼續斷路

### csp_lib 真實程式碼

`csp_lib/core/resilience.py` 中的 `CircuitBreaker` 是通用實現：

```python
# csp_lib/core/resilience.py

class CircuitState(Enum):
    """斷路器狀態"""
    CLOSED = "closed"
    OPEN = "open"
    HALF_OPEN = "half_open"


class CircuitBreaker:
    def __init__(self, threshold: int, cooldown: float) -> None:
        self._threshold = threshold
        self._cooldown = cooldown
        self._state = CircuitState.CLOSED
        self._failure_count = 0
        self._last_failure_time: float = 0.0

    @property
    def state(self) -> CircuitState:
        """取得目前狀態 (含自動 OPEN → HALF_OPEN 轉換)"""
        if self._state == CircuitState.OPEN:
            if time.monotonic() - self._last_failure_time >= self._cooldown:
                self._state = CircuitState.HALF_OPEN
        return self._state

    def record_success(self) -> None:
        """記錄成功：重置斷路器"""
        self._failure_count = 0
        self._state = CircuitState.CLOSED

    def record_failure(self) -> None:
        """記錄失敗：累計失敗次數，達閾值時開啟斷路器"""
        self._failure_count += 1
        self._last_failure_time = time.monotonic()
        if self._failure_count >= self._threshold:
            self._state = CircuitState.OPEN

    def allows_request(self) -> bool:
        """是否允許請求通過"""
        return self.state != CircuitState.OPEN
```

注意 `state` 屬性的巧妙設計：讀取狀態時**自動**檢查冷卻時間是否已過，如果是就轉為 HALF_OPEN。這意味著你不需要一個 timer 或背景任務來驅動狀態轉換——惰性計算（lazy evaluation）就夠了。

在 Modbus 請求佇列中，每個設備（unit_id）有自己的斷路器：

```python
# csp_lib/modbus/clients/queue.py

class ModbusRequestQueue:
    def _get_circuit_breaker(self, unit_id: int) -> UnitCircuitBreaker:
        """取得或建立指定 unit_id 的斷路器"""
        if unit_id not in self._circuit_breakers:
            self._circuit_breakers[unit_id] = UnitCircuitBreaker(
                threshold=self._config.circuit_breaker_threshold,
                cooldown=self._config.circuit_breaker_cooldown,
            )
        return self._circuit_breakers[unit_id]

    async def submit(self, unit_id, priority, coroutine_factory, timeout=None):
        """提交請求到佇列"""
        cb = self._get_circuit_breaker(unit_id)
        if not cb.allows_request():
            raise ModbusCircuitBreakerError(unit_id)  # 斷路器開啟，直接拒絕
        # ...正常入隊

    async def _worker(self) -> None:
        """背景 worker"""
        while self._running or self._total_size > 0:
            request = await self._dequeue()
            # ...
            cb = self._get_circuit_breaker(request.unit_id)
            if not cb.allows_request():
                request.future.set_exception(ModbusCircuitBreakerError(request.unit_id))
                continue

            try:
                result = await coro
                cb.record_success()  # 成功 → 重置
                request.future.set_result(result)
            except Exception as exc:
                cb.record_failure()  # 失敗 → 累計
                request.future.set_exception(exc)
```

這個設計的關鍵是 **per-unit-id 的斷路器**。一台設備掛了，只有它的斷路器跳脫，不影響同一條匯流排上的其他設備。這就像你家某個房間的 NFB 跳脫，不會讓整棟樓停電。

還有一個細節：`submit()` 在入隊前檢查一次斷路器，`_worker()` 在執行前又檢查一次。為什麼？因為請求在佇列裡等待的期間，斷路器可能從 CLOSED 變成 OPEN（其他請求連續失敗觸發的）。雙重檢查確保不會浪費時間執行注定失敗的請求。

### 如果不用 Circuit Breaker

```python
# 反面教材：不斷重試失敗的設備
async def read_loop(client, unit_id):
    while True:
        try:
            result = await asyncio.wait_for(
                client.read_holding_registers(0, 10, unit_id=unit_id),
                timeout=5.0  # 每次等 5 秒
            )
        except asyncio.TimeoutError:
            logger.error(f"Unit {unit_id} timeout, retrying...")
            # 繼續重試，每次都浪費 5 秒
            # 如果有 10 台設備共用佇列，所有人都被拖慢
            continue
```

### 何時不該用

- **失敗成本很低**：如果每次失敗只是返回一個 null，不涉及超時等待，Circuit Breaker 的額外複雜度不值得。
- **失敗是常態**：某些場景下大部分請求都會失敗（例如探測性掃描），Circuit Breaker 會永遠處於 OPEN 狀態。
- **需要每次都嘗試**：如果業務要求「即使可能失敗也必須嘗試」（例如急救指令），Circuit Breaker 不適用。

### 練習題

1. `RetryPolicy`（同在 `csp_lib/core/resilience.py`）是如何計算指數退避延遲的？
2. 在 `ModbusRequestQueue` 中，round-robin 排程是如何跳過 Circuit Breaker 為 OPEN 的 unit_id 的？

---

## 四個模式如何形成一個反應式系統

讓我們用一個真實場景串聯這四個模式：

**場景**：BMS 回報 SOC = 96%，同時 PCS_3 的 Modbus 連線已經連續失敗 5 次。

1. **Observer** 發射 `value_change` 事件，SOC 從 94% 變為 96%
2. 控制策略計算出充電指令 `P = -30kW`
3. **Chain of Responsibility** 的 `ProtectionGuard` 介入：
   - `SOCProtection` 發現 SOC = 96% >= 95%，將 P 從 -30kW 鉗位到 0
   - 後續規則看到 P = 0，不再修改
4. **Circuit Breaker** 在 PCS_3 的佇列中已經觸發 OPEN，CommandRouter 寫入 PCS_3 時直接跳過
5. **State** 模式中，如果這台機器是叢集環境，LeaderElector 確保只有 LEADER 節點在執行控制邏輯

```
  設備讀取循環
       │
       ▼
  [Observer] ──emit(value_change)──→ Dashboard / DB / 通知
       │
       ▼
  控制策略計算 Command
       │
       ▼
  [Chain of Responsibility] ProtectionGuard
       │  SOCProtection → ReversePowerProtection → SystemAlarmProtection
       ▼
  保護後的 Command
       │
       ▼
  CommandRouter.route()
       │
       ├──→ PCS_1: 正常寫入
       ├──→ PCS_2: 正常寫入
       └──→ PCS_3: [Circuit Breaker] OPEN → 跳過

  ── 以上所有邏輯只在 [State] = LEADER 時執行 ──
```

這四個模式各自解決一個問題，組合起來形成了一個完整的反應式控制系統：

| 模式 | 解決的問題 | 位置 |
|------|-----------|------|
| Observer | 事件的發布與訂閱解耦 | `equipment/device/events.py` |
| Chain of Responsibility | 保護規則的可組合、可追蹤 | `controller/system/protection.py` |
| State | 角色轉換的狀態機管理 | `cluster/election.py` |
| Circuit Breaker | 故障隔離與自動恢復 | `core/resilience.py` |

---

## 重點回顧

| 模式 | 一句話 | csp_lib 應用 | 關鍵好處 |
|------|--------|-------------|---------|
| **Observer** | 發生事就通知我 | `DeviceEventEmitter` | 解耦發布者與訂閱者 |
| **Chain of Responsibility** | 一關一關過 | `ProtectionGuard` + `ProtectionRule` | 規則可組合、fail-safe |
| **State** | 不同狀態不同行為 | `LeaderElector` + `ElectionState` | 狀態轉換明確可追蹤 |
| **Circuit Breaker** | 壞了就先停 | `CircuitBreaker` + `UnitCircuitBreaker` | 故障隔離、自動恢復 |

設計模式不是為了讓程式碼看起來「很有設計」。在工業控制場域，這四個模式分別守護了系統的四個關鍵品質：

- Observer → **可擴展性**（新功能不改核心）
- Chain of Responsibility → **安全性**（保護規則可審計）
- State → **一致性**（狀態轉換不會出錯）
- Circuit Breaker → **韌性**（局部故障不擴散）

---

## 下篇預告

反應式模式讓系統能「感知」和「自我保護」。但一個完整的工業系統還需要被「組裝」起來：幾十台設備如何被標準化地建立？如何被統一管理？如何讓使用者面對一個簡潔的介面而非複雜的內部結構？

下一篇，我們將探討 **Factory × Registry × Mediator × Facade** 這組「組裝模式」，看看 csp_lib 如何用工廠生產設備、用註冊表管理設備、用中介者協調控制、用門面簡化介面。

[下一篇：組裝的智慧：Factory × Registry × Mediator × Facade >>>](./00h-patterns-assembly.md)
