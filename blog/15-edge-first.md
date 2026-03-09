# Edge-First 架構：為什麼控制邏輯要放在邊緣

> **Part 4 — 控制迴路篇 | Article 15**
>
> 系列：從零打造工業控制協定框架：給軟體工程師的 CSP Library 實戰系列

---

## 前言

如果你正在開發後端服務，「把所有邏輯放上雲端」大概是你最直覺的選擇。API Gateway、微服務、容器編排——這些都是雲原生架構的成熟工具。然而，當控制對象從 API 請求變成一座儲能系統的功率變換器（PCS）時，雲端架構的基本假設就開始崩塌了。

這篇文章將帶你理解為什麼工業控制必須採用 Edge-First 架構，以及 csp_lib 如何在設計層面實現這個理念。

---

## 1. Cloud vs Edge：兩種截然不同的思維

在典型的 SaaS 架構中，我們的思考模型大致是這樣的：

```
使用者 → CDN → Load Balancer → API Server → Database
```

請求的延遲大約在 50ms ~ 200ms，偶爾一兩次超時可以重試，使用者最多看到 loading 轉圈。這在網頁應用中完全可以接受。

但在工業控制領域，架構變成：

```
電網 → 感測器/電表 → 控制器 → PCS/BMS → 電池
```

這條鏈路上的每個環節都有物理意義。電網頻率每秒都在變化，功率指令必須在毫秒到秒級送達設備。如果你的控制迴路需要繞道雲端：

```
感測器 → Edge Gateway → Internet → Cloud Server → Internet → Edge Gateway → PCS
```

光是網路往返就可能超過 100ms，加上雲端服務的處理延遲、排隊時間、可能的 SSL 握手——一個控制週期可能要花上 200ms ~ 500ms，甚至在網路抖動時超過 1 秒。

### 延遲為什麼是災難

讓我們用一個具體的情境來說明。台灣電力系統的自動頻率控制（AFC）要求儲能系統在頻率偏移時**數秒內**響應。假設電網頻率從 60.00Hz 突然掉到 59.75Hz：

| 架構 | 響應時間 | 結果 |
|------|---------|------|
| Edge-First | ~1 秒 | 正常放電補償，頻率回穩 |
| Cloud-Based | ~0.5-2 秒（正常） | 勉強及格，但不穩定 |
| Cloud-Based | >3 秒（網路抖動） | 違反 AFC 服務規範，面臨罰款 |
| Cloud-Based | 斷線 | 完全失控，安全風險 |

在功率控制場景中，100ms 的延遲不是「使用者體驗不好」，而是可能導致逆送電力到電網、觸發保護跳脫、甚至造成設備損壞。

---

## 2. 為什麼延遲這麼重要：100ms 的物理意義

軟體工程師習慣用「P99 延遲」來衡量系統性能，但在工業控制中，**每一次**的延遲都必須在可接受範圍內。原因是物理系統不等人。

### 2.1 電網是即時系統

電力系統的基本物理規則是：**發電量必須等於用電量**。當兩者不平衡時，頻率立刻改變。在台灣 60Hz 的電網中：

- 用電 > 發電 → 頻率下降（發電機轉速降低）
- 發電 > 用電 → 頻率上升（發電機轉速加快）

這個變化是**瞬時**的，不是分鐘級，是毫秒級。

### 2.2 功率控制的迴路時間要求

不同的控制模式有不同的時間敏感度：

| 控制模式 | 典型週期 | 延遲容忍度 |
|---------|---------|-----------|
| AFC（頻率調節） | 1 秒 | < 200ms |
| Volt-VAR（電壓調節） | 1 秒 | < 500ms |
| PQ 定功率 | 1 秒 | < 1 秒 |
| 排程切換 | 分鐘級 | 數秒 |

如果你的控制器在收到頻率信號後需要 500ms 才能發出功率指令，那麼 1 秒的控制週期中有一半時間是在等待——系統只能以一半的效率運作。

### 2.3 逆送保護是生死問題

在表後型儲能系統中，如果放電功率超過場域負載，多餘的電力會「逆送」回電網。在台灣法規下，未經許可的逆送是違法的，而且可能觸發配電設備的保護跳脫，造成區域停電。

逆送保護的邏輯很簡單：

```
p_target <= meter_power + threshold
```

但這個判斷必須在**本地**做。如果你需要把電表讀數傳到雲端、計算後再傳回來，這中間的任何延遲都可能讓保護失效。

---

## 3. Edge Computing 在能源管理的實踐

理解了延遲的危險性之後，讓我們看看 Edge-First 架構在實務中怎麼運作。

### 3.1 本地決策：控制迴路在邊緣

Edge-First 的核心理念是：**所有即時控制決策都在邊緣設備上完成**。不需要網路，不需要雲端，邊緣設備本身就是一個完整的控制器。

```
┌─────────────────────────────────────────┐
│              Edge Controller             │
│                                         │
│  感測器讀取 → 策略計算 → 保護檢查 → 設備寫入 │
│                                         │
│  ┌─────────┐  ┌──────────┐  ┌─────────┐ │
│  │ Modbus  │  │ Strategy │  │ Modbus  │ │
│  │  Read   │→ │ Execute  │→ │  Write  │ │
│  └─────────┘  └──────────┘  └─────────┘ │
│       ↑                          ↓      │
│  ┌─────────┐              ┌──────────┐  │
│  │  PCS    │              │   PCS    │  │
│  │  BMS    │              │  (寫入)   │  │
│  │  電表   │              └──────────┘  │
│  └─────────┘                            │
└─────────────────────────────────────────┘
```

邊緣控制器直接透過 Modbus TCP/RTU 與現場設備通訊。從讀取感測值到發出控制指令，整個迴路在本地完成。典型的迴路時間是數十毫秒到數百毫秒——取決於 Modbus 通訊速度和策略計算複雜度。

### 3.2 雲端負責監控與分析

那雲端做什麼？雲端負責那些**不需要即時響應**的工作：

- **監控儀表板**：呈現歷史趨勢、即時狀態（延遲幾秒完全可以接受）
- **資料分析**：電量統計、效率分析、異常偵測
- **排程下發**：隔日排程、月度計畫（分鐘級延遲無影響）
- **模型更新**：機器學習模型的訓練與下發
- **告警通知**：推播、Email、LINE 通知

### 3.3 混合式架構

實際部署中，邊緣和雲端是協作關係：

```
┌─────────────┐         ┌─────────────────┐
│   Cloud     │         │  Edge Controller │
│             │  排程    │                 │
│  Dashboard ─┼────────→│  ModeManager    │
│  Analytics  │  配置    │  StrategyExec   │
│  ML Models ─┼────────→│  ProtectionGuard│
│             │         │                 │
│  MongoDB   ←┼─────────┤  DataUpload     │
│  時序資料   │  資料上傳 │  (async, batch) │
└─────────────┘         └─────────────────┘
        ↑                       │
        │                       ↓
   網路可斷               Modbus 直連設備
   延遲可高               延遲必須低
   最終一致               即時一致
```

關鍵原則是：**即使雲端完全斷線，邊緣控制器也能獨立運作**。雲端的排程可以預先下載到本地，資料上傳可以先緩存再批次傳送。

---

## 4. csp_lib 作為邊緣控制框架

csp_lib 從設計之初就是一個 Edge-First 框架。讓我們看看它的控制迴路是怎麼運作的。

### 4.1 完整的控制迴路在本地運行

csp_lib 的控制迴路由四個核心元件組成，全部在同一個 Python 程序中執行：

```
ContextBuilder.build() → StrategyContext
     ↓
StrategyExecutor (strategy chosen by ModeManager)
     ↓
Command → ProtectionGuard.apply() → protected Command
     ↓
CommandRouter.route() → device writes
```

這四步在原始碼中對應 `SystemController` 的內部流程：

```python
# 步驟 1: 建構上下文（從設備讀取最新值）
def _build_context(self) -> StrategyContext:
    context = self._context_builder.build()
    has_alarm = any(dev.is_protected for dev in self._registry.all_devices)
    context.extra[self._config.system_alarm_key] = has_alarm
    return context

# 步驟 2: 策略執行（由 StrategyExecutor 在迴圈中呼叫）
# StrategyExecutor 從 context_provider 取得上下文後執行策略
command = self._strategy.execute(context)

# 步驟 3 & 4: 保護 + 路由（在 _on_command 回呼中）
async def _on_command(self, command: Command) -> None:
    # 保護鏈
    result = self._protection_guard.apply(command, context)
    protected_command = result.protected_command

    # 評估事件驅動 overrides
    await self._evaluate_event_overrides(context)

    # 路由到設備
    await self._command_router.route(protected_command)
```

### 4.2 零雲端依賴的即時控制

讓我們用一個完整的例子來展示。假設我們要建立一個 AFC（自動頻率控制）系統：

```python
from csp_lib.integration import DeviceRegistry, ContextMapping
from csp_lib.integration.schema import AggregateFunc, CommandMapping
from csp_lib.integration.system_controller import SystemController, SystemControllerConfig
from csp_lib.controller.core import SystemBase
from csp_lib.controller.strategies.fp_strategy import FPStrategy, FPConfig
from csp_lib.controller.system import ModePriority
from csp_lib.controller.system.protection import (
    SOCProtection, SOCProtectionConfig
)

# 1. 建立設備 Registry
registry = DeviceRegistry()
registry.register(pcs_device, traits=["pcs"], metadata={"rated_p": 500.0})
registry.register(bms_device, traits=["bms"])
registry.register(meter_device, traits=["meter"])

# 2. 配置 SystemController
config = SystemControllerConfig(
    context_mappings=[
        # 從 BMS 讀取 SOC
        ContextMapping(point_name="soc", context_field="soc", trait="bms"),
        # 從電表讀取頻率
        ContextMapping(
            point_name="frequency", context_field="extra.frequency", trait="meter"
        ),
    ],
    command_mappings=[
        # P 指令寫入 PCS
        CommandMapping(command_field="p_target", point_name="p_set", trait="pcs"),
    ],
    system_base=SystemBase(p_base=500.0, q_base=500.0),
    protection_rules=[
        SOCProtection(SOCProtectionConfig(soc_high=95, soc_low=5)),
    ],
)

# 3. 建立 SystemController（純本地，不需要任何網路連線）
controller = SystemController(registry, config)

# 4. 註冊 AFC 策略
fp_config = FPConfig(f_base=60.0, f1=-0.5, f6=0.5)
controller.register_mode("afc", FPStrategy(fp_config), ModePriority.SCHEDULE)
await controller.set_base_mode("afc")

# 5. 啟動（整個控制迴路在本地運行）
async with controller:
    await asyncio.Event().wait()  # 持續運行直到外部中斷
```

注意這段程式碼中**沒有任何 URL、API key、雲端連線**。整個系統只依賴本地的 Modbus 連線。

### 4.3 雲端連線是可選的

csp_lib 的雲端整合放在獨立的模組中，作為可選的附加功能：

```python
# 這些都是可選的，控制迴路不依賴它們
from csp_lib.mongo import MongoUploader      # 可選：資料上傳
from csp_lib.redis import RedisAdapter       # 可選：遠端指令接收
from csp_lib.notification import Notifier    # 可選：告警推播
```

即使你啟用了這些模組，它們也是以「最終一致」的方式運作：

- 資料上傳失敗？緩存到本地，稍後重試
- Redis 斷線？本地的 ModeManager 繼續運作
- 推播服務不可用？控制迴路完全不受影響

這就是 Edge-First 的核心精神：**雲端增強，但不依賴**。

---

## 5. 控制迴路架構深入解析

讓我們更深入地看 csp_lib 的控制迴路架構。

### 5.1 ContextBuilder：感知世界

`ContextBuilder` 負責從設備收集最新的感測值，建構成策略需要的上下文：

```python
# ContextBuilder 從 Registry 中的設備讀取 latest_values
# 然後透過 ContextMapping 映射到 StrategyContext
context = StrategyContext(
    last_command=Command(),      # 上一次的輸出指令
    soc=85.0,                    # 從 BMS 讀取的 SOC
    system_base=SystemBase(p_base=500.0, q_base=500.0),
    current_time=datetime.now(timezone.utc),
    extra={
        "frequency": 59.82,      # 從電表讀取的頻率
        "voltage": 378.5,        # 從電表讀取的電壓
        "meter_power": 120.0,    # 電表功率
        "system_alarm": False,   # 系統告警旗標
    },
)
```

這一步完全是本地操作——直接從記憶體中的 `latest_values` 讀取，沒有任何 I/O 延遲。設備的 `latest_values` 則由 `ReadScheduler` 在背景透過 Modbus 定期更新。

### 5.2 StrategyExecutor：做出決策

`StrategyExecutor` 按照設定的執行模式（PERIODIC / TRIGGERED / HYBRID）定期呼叫當前策略的 `execute()` 方法：

```python
class StrategyExecutor:
    async def _execute_strategy(self) -> Command:
        # 取得上下文
        base_context = self._context_provider()
        context = dataclasses.replace(
            base_context,
            last_command=self._last_command,
            current_time=datetime.now(timezone.utc),
        )

        # 執行策略（純 CPU 計算，通常 < 1ms）
        command = self._strategy.execute(context)
        self._last_command = command

        # 回呼（觸發保護和路由）
        if self._on_command is not None:
            await self._on_command(command)

        return command
```

### 5.3 ProtectionGuard：安全防線

保護鏈是控制迴路中最關鍵的環節。它確保即使策略計算出錯，也不會發出危險的指令：

```python
class ProtectionGuard:
    def apply(self, command: Command, context: StrategyContext) -> ProtectionResult:
        current = command
        triggered = []

        for rule in self._rules:
            try:
                current = rule.evaluate(current, context)
                if rule.is_triggered:
                    triggered.append(rule.name)
            except Exception:
                # 保護規則自身出錯 → fail-safe: P=0, Q=0
                current = Command(p_target=0.0, q_target=0.0)
                triggered.append(f"{rule.name}(fail-safe)")

        return ProtectionResult(
            original_command=command,
            protected_command=current,
            triggered_rules=triggered,
        )
```

注意 `fail-safe` 設計：如果保護規則本身拋出例外，系統會立即輸出零功率——寧可停機也不冒險。這是工業控制中的基本原則。

### 5.4 CommandRouter：執行動作

最後，`CommandRouter` 將保護後的指令寫入設備：

```python
class CommandRouter:
    async def route(self, command: Command) -> None:
        for mapping in self._mappings:
            value = getattr(command, mapping.command_field, None)
            if value is None:
                continue

            if mapping.device_id is not None:
                await self._write_single(mapping.device_id, mapping.point_name, value)
            else:
                # trait 模式：廣播寫入所有同 trait 的 responsive 設備
                await self._write_trait_broadcast(
                    mapping.trait, mapping.point_name, value
                )
```

整個 `read → compute → protect → write` 迴路在本地完成，典型延遲不超過數百毫秒。

---

## 6. Edge-First 的實際效益

### 6.1 容錯性：斷網不停機

Edge-First 最大的優勢是容錯性。在實際部署中，我們遇過各種網路故障：

- 光纖被施工挖斷（停機數天）
- ISP 路由異常（間歇性斷線數小時）
- VPN 隧道因為金鑰過期斷開
- DNS 解析失敗

在 Edge-First 架構下，這些事件的影響是：

- 監控儀表板暫時看不到最新資料 → 可以接受
- 排程更新延遲 → 繼續用上次下載的排程
- 告警通知延遲 → 本地控制器仍然會自動停機保護
- **控制迴路完全不受影響** → 這才是最重要的

### 6.2 低延遲：亞秒級控制

在 Edge-First 架構中，控制迴路的延遲組成：

| 步驟 | 典型延遲 |
|------|---------|
| Modbus 讀取（背景） | 50-200ms（不在關鍵路徑上） |
| ContextBuilder.build() | < 1ms（記憶體讀取） |
| Strategy.execute() | < 1ms（CPU 計算） |
| ProtectionGuard.apply() | < 1ms（CPU 計算） |
| Modbus 寫入 | 20-50ms |
| **總計** | **~50ms**（不含背景讀取） |

與雲端方案的 200ms+ 相比，Edge-First 架構快了一個數量級。

### 6.3 資料主權：敏感數據不離場

在工業場景中，設備運行數據可能涉及商業機密（發電效率、調度策略、客戶用電模式等）。Edge-First 架構天然地將敏感數據保留在現場：

- 控制決策在本地完成，不需要上傳即時數據
- 可以選擇性地上傳統計數據而非原始數據
- 符合資料在地化的法規要求

---

## 7. 什麼時候該用雲端

Edge-First 不代表完全不用雲端。以下場景中，雲端仍然是更好的選擇：

### 7.1 分析與報表

歷史數據的聚合分析需要大量儲存和計算資源，這是雲端的強項：

- 月度電量統計
- 效率趨勢分析
- 異常模式識別
- 合規報表生成

### 7.2 車隊管理

當你管理數十甚至數百個站點時，需要一個集中的管理平台：

- 統一監控所有站點狀態
- 比較不同站點的效能
- 批次下發配置更新
- 集中式告警管理

### 7.3 機器學習模型更新

ML 模型的訓練需要大量歷史數據和計算資源：

1. 邊緣設備上傳歷史數據到雲端
2. 雲端訓練新模型
3. 新模型下發到邊緣設備
4. 邊緣設備用新模型做本地決策

注意關鍵點：**推理（inference）在邊緣執行**，只有訓練在雲端。

### 7.4 遠端操控

操作員可能需要從遠端更改控制模式或參數：

```python
# 雲端下發的指令透過 Redis/MQTT 傳到邊緣
# 邊緣控制器收到後更新本地狀態
await controller.push_override("bypass")  # 進入維護模式
await controller.pop_override("bypass")   # 恢復正常控制
```

但即使遠端操控的通道斷開，本地的控制迴路仍然以最後設定的模式繼續運作。

---

## 8. 重點回顧

1. **工業控制的延遲容忍度極低**。100ms 的網路延遲在功率控制場景中可能造成嚴重後果，雲端迴路無法滿足即時控制需求。

2. **Edge-First 不是反對雲端**，而是讓邊緣設備具備獨立運作的能力。雲端負責監控、分析、管理——這些不需要即時響應的工作。

3. **csp_lib 的控制迴路完全在本地運行**。`ContextBuilder → StrategyExecutor → ProtectionGuard → CommandRouter` 這條鏈路不依賴任何外部服務。

4. **Fail-safe 設計是 Edge-First 的基石**。保護規則出錯時輸出零功率、設備離線時跳過寫入、網路斷開時繼續用最後設定——這些都是邊緣控制器必須具備的能力。

5. **雲端連線是增值服務**。MongoDB 資料上傳、Redis 遠端指令、推播通知——這些都是可選的，而且設計為「斷線容忍」。

---

## 下篇預告

理解了 Edge-First 架構之後，下一篇我們將深入 csp_lib 的控制策略抽象設計。你會看到 GoF 的 Strategy Pattern 如何在能源管理場景中發揮威力——從簡單的 PQ 定功率到複雜的 AFC 頻率調節，所有策略都透過一個統一的抽象介面實現。
