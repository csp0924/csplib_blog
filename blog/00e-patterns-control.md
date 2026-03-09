# 控制的骨架：Strategy × Template Method × Command

> **從零打造工業控制協定框架：給軟體工程師的 CSP Library 實戰系列**
>
> Part 0.5 — 設計模式活用篇 | Article 00e
>
> [上一篇：<<< 給電機/機械工程師的軟體速成](./00d-software-crash-course-for-engineers.md) | [下一篇：資料的旅程：Adapter × Pipeline × Builder >>>](./00f-patterns-data-flow.md)

---

## 目錄

1. [為什麼控制系統特別需要設計模式？](#為什麼控制系統特別需要設計模式)
2. [Strategy Pattern — 控制策略的熱插拔](#strategy-pattern--控制策略的熱插拔)
3. [Template Method — 生命週期的固定骨架](#template-method--生命週期的固定骨架)
4. [Command Pattern — 把控制指令變成物件](#command-pattern--把控制指令變成物件)
5. [Value Object — 不可變的設定](#value-object--不可變的設定)
6. [四個模式如何協作](#四個模式如何協作)
7. [重點回顧](#重點回顧)
8. [下篇預告](#下篇預告)

---

## 為什麼控制系統特別需要設計模式？

寫一般的 Web 應用程式，如果架構不好，最差的結果大概是 response time 慢一點、程式碼難維護一點。但在工業控制領域，架構不好的後果可能是：

- **儲能系統在該停止充電時繼續充電**，鋰電池熱失控
- **PCS 收到矛盾的功率指令**，保護跳脫導致整個案場停機
- **切換控制模式時出現空窗期**，電網頻率偏移觸發低頻卸載

這不是危言聳聽。工業控制系統有三個特性，讓設計模式從「錦上添花」變成「不可或缺」：

**模式多變**：同一套儲能系統可能需要 PQ 定功率、QV 電壓調節、離網模式、排程模式、緊急停機等十幾種控制策略，而且要能在運行中即時切換。

**安全關鍵**：每一條指令都直接操控物理設備。不像 Web API 可以 retry，一個錯誤的功率指令送出去就是送出去了。

**要可測試**：你不可能每次改程式都搬一套 500kW 的 PCS 到辦公室來測試。控制邏輯必須能脫離硬體獨立驗證。

接下來我們要看四個設計模式，它們在 csp_lib 中聯手解決了上述問題。這不是教科書上的理論範例，而是真正運行在儲能案場的程式碼。

---

## Strategy Pattern — 控制策略的熱插拔

### 日常比喻

想像你是一間咖啡店的老闆。早上人潮多，你用「快速出杯」策略——只做美式和拿鐵，30 秒一杯。下午人少了，切換到「精品手沖」策略——單品豆、手沖壺、4 分鐘一杯。晚上打烊前，切換到「清倉」策略——今日特價、買一送一。

重點是：**泡咖啡的人（執行器）不變，變的是策略**。你不需要早班、午班、晚班各請一個咖啡師，只需要一個咖啡師按照不同策略執行就好。

### 問題場景

假設你要為一套儲能系統實作多種控制模式。你的第一直覺可能是：

```python
# 直覺解法：用 if-else 處理所有模式
class GridController:
    def __init__(self):
        self.mode = "pq"

    def execute(self):
        if self.mode == "pq":
            p = self.config["p"]
            q = self.config["q"]
            self.send_command(p, q)
        elif self.mode == "qv":
            voltage = self.read_voltage()
            q = self.calculate_droop(voltage)
            self.send_command(self.last_p, q)
        elif self.mode == "island":
            self.open_acb()
            # ... 離網邏輯
        elif self.mode == "bypass":
            pass  # 什麼都不做
        elif self.mode == "stop":
            self.send_command(0, 0)
        # ... 再加 10 種模式？
```

這段程式碼有什麼問題？

1. **新增模式要改核心檔案**：每加一種模式，就要在 if-else 裡面塞一段，GridController 會越來越肥。
2. **無法獨立測試**：想測試 QV 策略的下垂計算？你得先 mock 整個 GridController。
3. **切換模式不安全**：if-else 裡面沒有「啟用」和「停用」的概念。離網模式需要在切出時等待同步信號再搭接斷路器，這種生命週期邏輯塞在 if-else 裡會變成一團亂麻。

### 模式登場

Strategy Pattern 的核心思想是：**把每種演算法封裝成獨立的類別，讓它們可以互相替換**。

在 csp_lib 中，所有控制策略都繼承自 `Strategy` 抽象基礎類別：

```python
# csp_lib/controller/core/strategy.py

class Strategy(ABC):
    """
    策略抽象基礎類別

    所有策略必須繼承此類別並實作：
    - execution_config: 回傳執行配置
    - execute(): 執行策略邏輯並回傳 Command
    """

    @property
    @abstractmethod
    def execution_config(self) -> ExecutionConfig:
        """回傳策略的執行配置"""
        pass

    @abstractmethod
    def execute(self, context: StrategyContext) -> Command:
        """執行策略邏輯"""
        pass

    async def on_activate(self) -> None:
        """策略啟用時呼叫 (可選覆寫)"""
        pass

    async def on_deactivate(self) -> None:
        """策略停用時呼叫 (可選覆寫)"""
        pass
```

注意幾個關鍵設計：

- `execute()` 接收 `StrategyContext`（唯讀的外部狀態），回傳 `Command`（不可變的指令物件）。策略本身不直接操控設備，只負責「算」。
- `on_activate()` 和 `on_deactivate()` 是可選的生命週期掛鉤，策略可以在啟用/停用時執行特殊動作。
- `execution_config` 告訴執行器這個策略該怎麼被呼叫（每秒一次？等觸發？）。

### csp_lib 真實程式碼

最簡單的具體策略——PQ 定功率模式：

```python
# csp_lib/controller/strategies/pq_strategy.py

class PQModeStrategy(Strategy):
    """根據配置輸出固定的 P/Q 值"""

    def __init__(self, config: Optional[PQModeConfig] = None):
        self._config = config or PQModeConfig()

    @property
    def execution_config(self) -> ExecutionConfig:
        return ExecutionConfig(mode=ExecutionMode.PERIODIC, interval_seconds=1)

    def execute(self, context: StrategyContext) -> Command:
        return Command(p_target=self._config.p, q_target=self._config.q)
```

再看一個稍微複雜的——QV 電壓無功控制，根據電壓偏差計算無功功率：

```python
# csp_lib/controller/strategies/qv_strategy.py

class QVStrategy(Strategy):
    """根據系統電壓偏差，透過下垂控制計算無功功率輸出"""

    def __init__(self, config: Optional[QVConfig] = None) -> None:
        self._config = config or QVConfig()

    @property
    def execution_config(self) -> ExecutionConfig:
        return ExecutionConfig(mode=ExecutionMode.PERIODIC, interval_seconds=1)

    def execute(self, context: StrategyContext) -> Command:
        voltage = context.extra.get("voltage")
        if voltage is None:
            return context.last_command  # 無資料時維持上一次命令

        q_ratio = self._calculate_q_ratio(voltage)

        if context.system_base is not None:
            q_kvar = context.percent_to_kvar(q_ratio * 100)
        else:
            q_kvar = q_ratio

        return Command(p_target=context.last_command.p_target, q_target=q_kvar)
```

還有最有趣的——離網模式，它充分利用了 `on_activate` / `on_deactivate` 生命週期：

```python
# csp_lib/controller/strategies/island_strategy.py

class IslandModeStrategy(Strategy):
    """離網模式：啟用時切離 ACB，停用時等待同步後搭接"""

    def __init__(self, relay: RelayProtocol, config: Optional[IslandModeConfig] = None):
        self._relay = relay
        self._config = config or IslandModeConfig()

    @property
    def execution_config(self) -> ExecutionConfig:
        return ExecutionConfig(mode=ExecutionMode.TRIGGERED, interval_seconds=1)

    def execute(self, context: StrategyContext) -> Command:
        return context.last_command  # 不主動發送功率命令

    async def on_activate(self) -> None:
        """策略啟用: 切離 ACB，進入離網模式"""
        await self._relay.set_open()

    async def on_deactivate(self) -> None:
        """策略停用: 等待 sync_ok 後搭接 ACB，返回併網模式"""
        timeout = self._config.sync_timeout
        elapsed = 0.0
        while elapsed < timeout:
            if self._relay.sync_ok:
                break
            await asyncio.sleep(0.5)
            elapsed += 0.5
        if self._relay.sync_ok:
            await self._relay.set_close()
```

而在 `StrategyExecutor` 中，切換策略時會自動呼叫生命週期方法：

```python
# csp_lib/controller/executor/strategy_executor.py (節錄)

async def set_strategy(self, strategy: Optional[Strategy]) -> None:
    if self._strategy is not None:
        await self._strategy.on_deactivate()

    self._strategy = strategy

    if self._strategy is not None:
        await self._strategy.on_activate()
```

這意味著從離網模式切換到 PQ 模式時，系統會自動完成：切離 ACB → 等待同步 → 搭接 ACB → 開始發送 PQ 命令。全部自動，沒有任何 if-else。

### 反面教材

如果不用 Strategy Pattern，想要處理離網模式的切換邏輯，你得這樣寫：

```python
# 反面教材：不用 Strategy 的話
class GridController:
    async def switch_mode(self, new_mode: str):
        # 離開舊模式的清理
        if self.mode == "island":
            # 等待同步...
            while not self.relay.sync_ok:
                await asyncio.sleep(0.5)
            await self.relay.set_close()
        elif self.mode == "some_other_mode":
            # 其他清理邏輯...
            pass

        self.mode = new_mode

        # 進入新模式的初始化
        if new_mode == "island":
            await self.relay.set_open()
        elif new_mode == "some_other_mode":
            # 其他初始化邏輯...
            pass
```

每增加一個有生命週期的模式，`switch_mode` 就要多兩段 if-else（進入和離開）。如果你有 10 種模式，互相切換的組合就是 10 × 10 = 100 種路徑。這是維護噩夢。

### 何時不該用

如果你的系統只有兩三種控制模式，而且永遠不會增加，用 Strategy Pattern 反而是殺雞用牛刀。一個簡單的 if-else 配上清楚的註解就夠了。Strategy Pattern 的價值在於**策略數量會增長**的場景。

### 練習題

翻閱 `csp_lib/controller/strategies/` 目錄，你會找到 `StopStrategy`、`BypassStrategy`、`FPStrategy`、`ScheduleStrategy` 等更多具體策略。觀察它們各自的 `execution_config` 設定——有的用 `PERIODIC`，有的用 `TRIGGERED`，有的用 `HYBRID`。思考一下：為什麼不同策略需要不同的執行模式？

---

## Template Method — 生命週期的固定骨架

### 日常比喻

去餐廳吃飯的流程永遠是：入座 → 點餐 → 上菜 → 吃飯 → 結帳 → 離開。這個**骨架**不會變，但每一步的具體內容因餐廳而異。高級法餐的「點餐」是侍酒師配酒；路邊小吃攤的「點餐」是對老闆喊一碗滷肉飯。

Template Method 就是定義好「流程骨架」，讓子類別填入「具體步驟」。

### 問題場景

在 csp_lib 中，有大量元件需要管理 async 生命週期：Modbus 連線、設備管理器、資料上傳服務、控制器、Redis 連線池......每一個都需要「啟動」和「停止」的概念，而且都要支援 `async with` 語法。

你的第一直覺可能是每個類別各寫各的：

```python
# 直覺解法：每個類別自己實作
class DeviceManager:
    async def __aenter__(self):
        await self.connect_all_devices()
        return self

    async def __aexit__(self, *args):
        await self.disconnect_all_devices()

class DataUploader:
    async def __aenter__(self):
        await self.init_db_connection()
        return self

    async def __aexit__(self, *args):
        await self.close_db_connection()

class AlarmManager:
    async def __aenter__(self):
        await self.load_alarm_rules()
        return self

    async def __aexit__(self, *args):
        await self.flush_pending_alarms()
```

問題是什麼？

1. **重複的樣板碼**：每個類別都要寫 `__aenter__` 和 `__aexit__`，格式幾乎一模一樣。
2. **容易忘記**：某個工程師新增了一個元件，忘記實作 `__aexit__`，資源洩漏了。
3. **沒有統一介面**：想寫一個通用的「健康檢查」或「優雅關閉」機制時，沒有統一的 `start()` / `stop()` 可以呼叫。

### 模式登場

csp_lib 的 `AsyncLifecycleMixin` 就是一個教科書級的 Template Method 實作：

```python
# csp_lib/core/lifecycle.py

class AsyncLifecycleMixin:
    """
    Async 生命週期 Mixin

    子類別只需覆寫 _on_start() 與 _on_stop() 即可。
    """

    async def start(self) -> None:
        """啟動服務"""
        await self._on_start()

    async def stop(self) -> None:
        """停止服務"""
        await self._on_stop()

    async def _on_start(self) -> None:
        """子類別覆寫此方法以實作啟動邏輯"""

    async def _on_stop(self) -> None:
        """子類別覆寫此方法以實作停止邏輯"""

    async def __aenter__(self) -> Self:
        await self.start()
        return self

    async def __aexit__(self, *args: Any) -> None:
        await self.stop()
```

骨架很清楚：

- **固定的部分**（Template）：`start()` 呼叫 `_on_start()`，`stop()` 呼叫 `_on_stop()`，`__aenter__` / `__aexit__` 分別委託給 `start()` / `stop()`。
- **可變的部分**（Hook）：`_on_start()` 和 `_on_stop()` 留給子類別覆寫。

注意前綴底線 `_on_start` 的命名慣例。這清楚地告訴使用者：「你應該覆寫這個，但不要直接呼叫它。外部應該呼叫 `start()`。」

### 使用範例

所有繼承 `AsyncLifecycleMixin` 的元件，使用方式完全一致：

```python
class MyService(AsyncLifecycleMixin):
    async def _on_start(self) -> None:
        self._connection = await create_connection()

    async def _on_stop(self) -> None:
        await self._connection.close()

# 使用時：
async with MyService() as svc:
    await svc.do_work()
# _on_stop() 自動被呼叫，即使 do_work() 拋出例外
```

### 反面教材

```python
# 反面教材：沒有統一骨架
class ServiceA:
    async def initialize(self):  # 方法名不一樣
        ...
    async def cleanup(self):     # 方法名不一樣
        # 忘記實作 __aexit__

class ServiceB:
    async def open(self):        # 又一個不同的名字
        ...
    async def shutdown(self):
        ...

# 想要統一管理？
services = [ServiceA(), ServiceB()]
for svc in services:
    await svc.???()  # start? initialize? open? 每個都不一樣
```

### 何時不該用

如果你的類別只有一兩個，而且生命週期邏輯非常簡單（例如就是打開/關閉一個檔案），直接實作 `__aenter__` / `__aexit__` 反而更直觀。Template Method 的價值在於**多個類別共享相同的流程骨架**。

### 練習題

在 csp_lib 中搜尋 `AsyncLifecycleMixin` 的使用者。你會在 `DeviceManager`、`AlarmPersistenceManager`、`DataUploadManager`、`UnifiedDeviceManager` 等多個管理器中看到它。觀察這些子類別的 `_on_start()` 都做了什麼——有的啟動排程任務，有的初始化連線，有的載入設定。骨架相同，血肉各異。

---

## Command Pattern — 把控制指令變成物件

### 日常比喻

你去餐廳點餐時，不是直接衝進廚房跟廚師說「給我一份牛排七分熟」。你是把需求寫在**點菜單**上，服務生拿著這張單子交給廚房。這張點菜單就是一個 Command 物件——它把「要做什麼」從「誰來做」和「誰要求的」中解耦了。

### 問題場景

控制策略算出了結果，要發送給 PCS。你的第一直覺可能是：

```python
# 直覺解法：策略直接呼叫設備方法
class PQModeStrategy:
    def __init__(self, pcs):
        self.pcs = pcs  # 直接持有設備引用

    def execute(self):
        self.pcs.set_active_power(self.config.p)
        self.pcs.set_reactive_power(self.config.q)
```

問題是什麼？

1. **策略綁死了設備**：想測試 PQ 策略？你得 mock 整個 PCS 物件。
2. **無法記錄歷史**：指令發出去就發出去了，你不知道上一次發了什麼。
3. **策略無法組合**：如果我想讓 QV 策略只改 Q 值、P 值保持不變，策略之間怎麼溝通？

### 模式登場

csp_lib 的 `Command` 是一個不可變的 frozen dataclass，封裝了策略的輸出：

```python
# csp_lib/controller/core/command.py

@dataclass(frozen=True)
class Command:
    """
    策略輸出命令 (不可變)

    Attributes:
        p_target: 有功功率目標值 (kW)
        q_target: 無功功率目標值 (kVar)
    """

    p_target: float = 0.0
    q_target: float = 0.0

    def with_p(self, p: float) -> Command:
        """建立新 Command，替換 P 值"""
        return dataclasses.replace(self, p_target=p)

    def with_q(self, q: float) -> Command:
        """建立新 Command，替換 Q 值"""
        return dataclasses.replace(self, q_target=q)
```

幾個精妙的設計：

**`frozen=True`**：Command 一旦建立就不能修改。這在多執行緒或 async 環境中至關重要——你不用擔心某個 coroutine 偷偷改了 Command 的值。

**`with_p()` / `with_q()`**：因為是 frozen 的，想「修改」就必須建立新物件。`with_p()` 使用 `dataclasses.replace()` 建立一個只改了 P 值的新 Command。這是函數式程式設計中 immutable update 的常見手法。

**預設值 0.0**：空的 `Command()` 代表「什麼都不做」，這是安全的預設值。

### 看看策略怎麼使用 Command

回頭看各種策略的 `execute()` 方法：

```python
# PQ 策略：直接建立新 Command
def execute(self, context: StrategyContext) -> Command:
    return Command(p_target=self._config.p, q_target=self._config.q)

# Stop 策略：建立零功率 Command
def execute(self, context: StrategyContext) -> Command:
    return Command(p_target=0.0, q_target=0.0)

# Bypass 策略：返回上一次的 Command（維持現狀）
def execute(self, context: StrategyContext) -> Command:
    return context.last_command

# QV 策略：只改 Q 值，P 保持不變
def execute(self, context: StrategyContext) -> Command:
    # ... 計算 q_kvar ...
    return Command(p_target=context.last_command.p_target, q_target=q_kvar)
```

注意 QV 策略的做法：它從 `context.last_command` 取得上一次的 P 值，只替換 Q 值。這就是 Command Pattern 的威力——**指令是資料，可以讀取、組合、傳遞**。

而 `StrategyExecutor` 在執行策略後，會把 Command 儲存起來供下次使用：

```python
# csp_lib/controller/executor/strategy_executor.py (節錄)

async def _execute_strategy(self) -> Command:
    base_context = self._context_provider()
    context = dataclasses.replace(
        base_context,
        last_command=self._last_command,  # 注入上一次的 Command
        current_time=datetime.now(timezone.utc)
    )

    command = self._strategy.execute(context)
    self._last_command = command  # 儲存供下次使用

    if self._on_command is not None:
        await self._on_command(command)  # 通知外部處理

    return command
```

### 反面教材

```python
# 反面教材：策略直接操控設備
class QVStrategy:
    def __init__(self, pcs):
        self.pcs = pcs

    def execute(self, voltage):
        q = self.calculate_droop(voltage)
        self.pcs.set_reactive_power(q)
        # 問題1: P 值呢？忘了設了嗎？還是故意不設？語意不明確
        # 問題2: 想知道上一次發了什麼 Q？self.pcs 沒有 last_q 屬性
        # 問題3: 想寫單元測試？得 mock self.pcs 和它的所有方法
```

### 何時不該用

如果你的系統只有一個輸出通道、一種指令格式，而且不需要記錄歷史，直接呼叫方法更簡單。Command Pattern 的價值在於：**指令需要被記錄、被傳遞、被組合、或被延遲執行**。

### 練習題

注意 `Command` 的 `__str__` 方法回傳 `Command(P=100.0kW, Q=50.0kVar)` 這樣的格式。想一想：為什麼要自訂 `__str__`？在什麼場景下這個格式化輸出特別有用？（提示：想想 log 檔案。）

---

## Value Object — 不可變的設定

### 日常比喻

你拿到一張電影票，上面印著「3/15 14:00 第5廳 F排12號」。這些資訊一旦印上去就不會變——你不能拿修正液把座位號改成 F排1號。如果你要換座位，只能去櫃台重新出一張新票。

Value Object 就是這樣的概念：一旦建立就不可修改，要「修改」就建立新的。

### 問題場景

控制策略需要各種設定參數。你的第一直覺可能是用普通的 dict 或可變的 dataclass：

```python
# 直覺解法：用 dict 存設定
config = {"p": 100, "q": 50}

# 某個函式悄悄改了設定
def some_function(cfg):
    cfg["p"] = 0  # 直覺上只是「讀取」設定，但其實改了

# 原始的 config 也被改了！
print(config["p"])  # 0，不是 100
```

或者：

```python
# 直覺解法：可變的 dataclass
@dataclass
class Config:
    p: float = 0.0
    q: float = 0.0

config = Config(p=100, q=50)
# 在多執行緒環境中，某個 thread 改了 config.p
# 另一個 thread 讀到了改到一半的值
```

### 模式登場

csp_lib 中，所有設定物件都使用 `@dataclass(frozen=True)`：

```python
# csp_lib/controller/core/command.py

@dataclass(frozen=True)
class SystemBase(ConfigMixin):
    """
    系統基準值，用於百分比與絕對值轉換

    Usage:
        p_kw = p_percent * system_base.p_base / 100
        q_kvar = q_percent * system_base.q_base / 100
    """

    p_base: float = 0.0
    q_base: float = 0.0
```

```python
# csp_lib/controller/core/execution.py

@dataclass(frozen=True)
class ExecutionConfig:
    """
    策略執行配置

    Attributes:
        mode: 執行模式
        interval_seconds: 週期秒數
    """

    mode: ExecutionMode
    interval_seconds: int = 1

    def __post_init__(self):
        if self.mode != ExecutionMode.TRIGGERED and self.interval_seconds <= 0:
            raise ValueError("interval_seconds must be positive for PERIODIC/HYBRID mode")
```

注意 `ExecutionConfig` 的 `__post_init__`：frozen dataclass 在建立時就驗證資料。如果你傳入無效的 `interval_seconds`，馬上就會得到例外。這比事後才發現「怎麼週期變成 0 秒了」好太多。

再看 `ConfigMixin` 提供的便利方法：

```python
# csp_lib/controller/core/command.py (節錄)

class ConfigMixin:
    """Config 類別的 Mixin，提供統一的 from_dict 方法"""

    @classmethod
    def from_dict(cls: Type[T], data: dict) -> T:
        """
        從字典建立 Config 實例

        自動過濾不存在於 dataclass 欄位的 key。
        支援 camelCase 到 snake_case 的轉換。
        """
        field_names = {f.name for f in dataclasses.fields(cls)}
        filtered_data = {}
        for key, value in data.items():
            if key in field_names:
                filtered_data[key] = value
            else:
                snake_key = _camel_to_snake(key)
                if snake_key in field_names:
                    filtered_data[snake_key] = value
        return cls(**filtered_data)
```

`from_dict` 方法支援從 JSON（通常是 camelCase）直接建立不可變的 Config 物件，並且自動過濾多餘的 key。這在接收 API 請求或讀取設定檔時非常實用。

### 反面教材

```python
# 反面教材：可變的設定
class PQConfig:
    def __init__(self):
        self.p = 0.0
        self.q = 0.0

config = PQConfig()
config.p = 100

# 策略拿到 config 的引用
strategy = PQStrategy(config)

# 某處修改了 config（可能是 API handler、可能是另一個 thread）
config.p = -999  # 策略下次執行就會送出 -999kW！

# 更糟：沒有驗證
config.p = "hello"  # 直到 execute() 才會爆炸
```

### 何時不該用

如果物件確實需要頻繁更新狀態（例如即時感測器數據），強制 frozen 會造成大量物件建立的開銷。Value Object 適合用在**設定、命令、映射規格**這類建立後不需要修改的資料。

### 練習題

找到 `csp_lib/integration/schema.py`，觀察 `ContextMapping`、`CommandMapping`、`HeartbeatMapping` 這些映射物件。它們全部都是 `@dataclass(frozen=True)`。想一想：為什麼映射規格特別適合用 Value Object？（提示：一個映射在系統運行期間應該永遠不變。）

---

## 四個模式如何協作

現在讓我們把四個模式串起來，走訪一次完整的控制流程。

假設場景：儲能系統正在 PQ 模式下以 P=200kW 運行，操作員透過 API 下達「切換到 QV 模式」的指令。

**第 1 步：Value Object 定義設定**

```python
qv_config = QVConfig(nominal_voltage=380, v_set=100, droop=5)
# frozen=True：這份設定在策略運行期間不可能被意外修改
```

**第 2 步：Strategy 封裝控制邏輯**

```python
qv_strategy = QVStrategy(config=qv_config)
# 具體策略繼承自 Strategy ABC，實作了 execute() 和 execution_config
```

**第 3 步：切換策略觸發 Template Method 生命週期**

```python
await executor.set_strategy(qv_strategy)
# 1. 呼叫舊策略 (PQ) 的 on_deactivate()
# 2. 呼叫新策略 (QV) 的 on_activate()
# 這個流程是 StrategyExecutor 的固定骨架，不需要 if-else
```

**第 4 步：執行器按週期呼叫 Strategy.execute()**

```python
# StrategyExecutor 的 run() 迴圈
context = context_provider()  # 取得即時設備狀態
context = replace(context, last_command=last_command)  # 注入上次的 Command

command = qv_strategy.execute(context)
# QV 策略從 context.extra["voltage"] 讀取電壓
# 計算無功功率，回傳新的 Command
```

**第 5 步：Command 被傳遞給下游**

```python
# command 是 frozen 的 Command 物件
# 可以安全地傳給 log、傳給 MongoDB、傳給 PCS
# 不用擔心任何人在傳遞過程中偷改它
await on_command(command)  # 回呼函式將 Command 路由到設備
```

整個流程中：
- **Strategy Pattern** 讓我們不用改任何核心程式碼就能切換控制邏輯
- **Template Method** 確保生命週期轉換的流程一致且安全
- **Command Pattern** 讓策略的輸出變成可傳遞、可記錄的物件
- **Value Object** 確保設定和命令在整個流程中不被意外修改

這四個模式缺一個都會出問題：沒有 Strategy，你會陷入 if-else 地獄；沒有 Template Method，生命週期管理會變成每個類別各自為政；沒有 Command，策略和設備之間會緊耦合；沒有 Value Object，多執行緒環境下的共享狀態會成為 bug 溫床。

---

## 重點回顧

| 模式 | 解決什麼問題 | csp_lib 中的角色 | 關鍵程式碼 |
|------|-------------|-----------------|-----------|
| **Strategy** | 多種演算法的熱插拔 | 控制策略的統一介面 | `Strategy` ABC + 各種具體策略 |
| **Template Method** | 固定流程 + 可變步驟 | Async 生命週期管理 | `AsyncLifecycleMixin` |
| **Command** | 將請求封裝為物件 | 策略輸出的不可變指令 | `Command` frozen dataclass |
| **Value Object** | 防止共享狀態被修改 | 設定、命令、映射規格 | `frozen=True` + `ConfigMixin` |

記住一個原則：**設計模式不是為了讓程式碼看起來高級，而是為了讓系統在面對變化時仍然可控**。在工業控制這個「變化 = 風險」的領域，這一點格外重要。

---

## 下篇預告

控制策略算出了 Command，但這個 Command 要怎麼送到設備？設備回傳的 raw register 又要怎麼變成策略能理解的「電壓」和「SOC」？

下一篇 [**資料的旅程：Adapter × Pipeline × Builder**](./00f-patterns-data-flow.md)，我們會跟著一筆資料走完從 Modbus 暫存器到 StrategyContext 的完整旅程，看看 Adapter、Pipeline、Builder 這三個模式如何讓異質資料在系統中順暢流動。
