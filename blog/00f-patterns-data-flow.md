# 資料的旅程：Adapter × Pipeline × Builder

> **從零打造工業控制協定框架：給軟體工程師的 CSP Library 實戰系列**
>
> Part 0.5 — 設計模式活用篇 | Article 00f
>
> [上一篇：<<< 控制的骨架：Strategy × Template Method × Command](./00e-patterns-control.md) | [下一篇：反應的藝術：Observer × Chain of Responsibility × State >>>](./00g-patterns-reactive.md)

---

## 目錄

1. [工業資料的旅程](#工業資料的旅程)
2. [Adapter Pattern — 讓異質資料說同一種語言](#adapter-pattern--讓異質資料說同一種語言)
3. [Pipeline Pattern — 資料處理的流水線](#pipeline-pattern--資料處理的流水線)
4. [Builder Pattern — 複雜物件的分步建構](#builder-pattern--複雜物件的分步建構)
5. [Configuration Object — 不可變的設定檔](#configuration-object--不可變的設定檔)
6. [完整資料流走訪](#完整資料流走訪)
7. [重點回顧](#重點回顧)
8. [下篇預告](#下篇預告)

---

## 工業資料的旅程

上一篇我們看了控制策略如何算出 `Command(p_target=200, q_target=50)`。但如果你往上追問一步：策略用來計算的「電壓 380V」和「SOC 85%」是從哪來的？

答案是：從 Modbus 暫存器的原始 bytes。

一筆工業資料從設備到控制策略的旅程長這樣：

```
Modbus Register (raw: 3800)
    ↓  ScaleTransform(magnitude=0.1)
物理量 (380.0V)
    ↓  RoundTransform(decimals=1)
清潔資料 (380.0)
    ↓  ContextMapping(context_field="extra.voltage")
StrategyContext.extra["voltage"] = 380.0
    ↓  Strategy.execute(context)
Command(p_target=..., q_target=...)
```

這條路徑上的每一個箭頭，都對應一個設計模式。接下來我們逐一拆解。

---

## Adapter Pattern — 讓異質資料說同一種語言

### 日常比喻

你出國旅行，帶了一堆台灣的電器。到了英國，插座是三孔方頭的；到了日本，電壓是 100V。你需要各種**轉接頭**——不是改變電器本身，也不是改變牆上的插座，而是在中間加一個轉換層。

Adapter Pattern 就是這個轉接頭：讓原本介面不相容的東西能夠合作。

### 問題場景

工業設備回傳的原始資料五花八門。同樣是「溫度」：

- 設備 A 回傳整數 `250`，實際溫度 = 250 × 0.1 - 40 = **-15.0°C**
- 設備 B 回傳整數 `3500`，實際溫度 = 3500 × 0.01 = **35.0°C**
- 設備 C 回傳 `0` / `1` / `2` 代表 `停機` / `運行` / `故障`

你的第一直覺可能是在讀取資料的地方寫轉換：

```python
# 直覺解法：到處散落的轉換邏輯
class DeviceReader:
    def read_temperature_a(self):
        raw = self.read_register(100)
        return raw * 0.1 - 40  # 散落在讀取邏輯中

    def read_temperature_b(self):
        raw = self.read_register(200)
        return raw * 0.01  # 另一種倍率

    def read_status(self):
        raw = self.read_register(300)
        if raw == 0:
            return "STOP"
        elif raw == 1:
            return "RUN"
        elif raw == 2:
            return "FAULT"
        else:
            return "UNKNOWN"
```

問題是什麼？

1. **轉換邏輯散落各處**：倍率和偏移量硬編碼在讀取函式裡，改一個設備的倍率要翻遍整個程式碼。
2. **無法重用**：設備 A 和 B 都是縮放轉換，邏輯幾乎一樣但寫了兩次。
3. **無法組合**：如果先要縮放、再要四捨五入、再要限制值域呢？巢狀函式呼叫會變成義大利麵。

### 模式登場

csp_lib 定義了 `TransformStep` Protocol 作為所有轉換的統一介面：

```python
# csp_lib/equipment/core/transform.py

class TransformStep(Protocol):
    """轉換步驟介面"""

    def apply(self, value: Any) -> Any:
        """套用轉換"""
        ...
```

只有一個方法 `apply()`：輸入一個值，輸出一個值。簡單到不能再簡單。

然後，每一種轉換都是一個獨立的 frozen dataclass，實作了 `apply()`：

```python
# csp_lib/equipment/core/transform.py

@dataclass(frozen=True)
class ScaleTransform:
    """
    縮放轉換
    計算: result = value * magnitude + offset
    """

    magnitude: float = 1.0
    offset: float = 0.0

    def apply(self, value: Any) -> float:
        if not isinstance(value, (int, float)):
            raise TypeError(f"ScaleTransform 需要數值，收到: {type(value).__name__}")
        return float(value) * self.magnitude + self.offset
```

```python
@dataclass(frozen=True)
class RoundTransform:
    """四捨五入轉換"""

    decimals: int = 2

    def apply(self, value: Any) -> float:
        if not isinstance(value, (int, float)):
            raise TypeError(f"RoundTransform 需要數值，收到: {type(value).__name__}")
        return round(float(value), self.decimals)
```

```python
@dataclass(frozen=True)
class EnumMapTransform:
    """數值 → 枚舉映射轉換"""

    mapping: dict[int, str]
    default: str = "UNKNOWN"

    def apply(self, value: Any) -> str:
        if not isinstance(value, int):
            try:
                value = int(value)
            except (TypeError, ValueError):
                return self.default + " (" + str(value) + ")"
        return self.mapping.get(value, self.default)
```

```python
@dataclass(frozen=True)
class ClampTransform:
    """值域限制轉換"""

    min_value: float | None = None
    max_value: float | None = None

    def apply(self, value: Any) -> float:
        if not isinstance(value, (int, float)):
            raise TypeError(f"ClampTransform 需要數值，收到: {type(value).__name__}")
        result = float(value)
        if self.min_value is not None:
            result = max(result, self.min_value)
        if self.max_value is not None:
            result = min(result, self.max_value)
        return result
```

每一個 Transform 都是一個「轉接頭」——它不改變來源（Modbus 暫存器），也不改變目的地（StrategyContext），只負責把格式不合的資料轉成統一的物理量。

csp_lib 甚至提供了一些領域專用的 Adapter，例如功率因數解碼：

```python
@dataclass(frozen=True)
class PowerFactorTransform:
    """
    功率因數解碼轉換（Schneider PM5350 專用）

    PM5350 使用特殊編碼表示功率因數與相位：
        - Q1 (0° ~ 90°):   0 < x < 1   → PF = x, lagging
        - Q2 (90° ~ 180°): -2 < x < -1 → PF = -2 - x, leading
        - Q4 (270° ~ 360°): 1 < x < 2  → PF = 2 - x, leading
    """

    include_status: bool = False

    def apply(self, value: Any) -> float | dict[str, Any]:
        reg_val = float(value)
        if reg_val > 1:
            pf_val = 2 - reg_val
            status = "leading"
        elif reg_val < -1:
            pf_val = -2 - reg_val
            status = "leading"
        elif abs(reg_val) == 1:
            pf_val = reg_val
            status = "unity"
        else:
            pf_val = reg_val
            status = "lagging"

        if self.include_status:
            return {"pf": pf_val, "status": status}
        return pf_val
```

這段程式碼封裝了 Schneider PM5350 電表的功率因數編碼規則。如果沒有 Adapter Pattern，這些供應商專用的解碼邏輯會散落在各個讀取函式中，更換電表型號時改到崩潰。有了 `PowerFactorTransform`，你只需要在設備設定中指定使用哪個 Transform，就完成了。

還有專門處理位元欄位的 Adapter：

```python
@dataclass(frozen=True)
class BitExtractTransform:
    """從整數值中提取指定範圍的位元"""

    bit_offset: int
    bit_length: int = 1

    @property
    def mask(self) -> int:
        return (1 << self.bit_length) - 1

    def apply(self, value: Any) -> int | bool:
        if not isinstance(value, int):
            raise TypeError(f"BitExtractTransform 需要整數，收到: {type(value).__name__}")
        result = (value >> self.bit_offset) & self.mask
        return bool(result) if self.bit_length == 1 else result
```

以及一次提取多個位元欄位的 `MultiFieldExtractTransform`：

```python
@dataclass(frozen=True)
class MultiFieldExtractTransform:
    """從單一整數值中提取多個命名的位元欄位"""

    fields: tuple[tuple[str, int, int], ...]

    def apply(self, value: Any) -> dict[str, int | bool]:
        result: dict[str, int | bool] = {}
        for name, offset, length in self.fields:
            mask = (1 << length) - 1
            extracted = (value >> offset) & mask
            result[name] = bool(extracted) if length == 1 else extracted
        return result
```

工業設備經常把多個狀態塞在同一個暫存器的不同 bit 裡面。例如一個 16-bit 的「系統狀態字」，bit 0 是運行中、bit 1 是故障、bit 8-11 是運行模式。`MultiFieldExtractTransform` 讓你用宣告式的方式描述這些位元欄位，而不是寫一堆 `& 0x01`、`>> 8` 的位元運算。

### 反面教材

```python
# 反面教材：把轉換邏輯寫死在設備類別裡
class PCS:
    def get_status(self):
        raw = self.read_register(100)
        # Bit 0: running
        running = bool(raw & 0x01)
        # Bit 1: fault
        fault = bool(raw & 0x02)
        # Bit 8-11: mode
        mode = (raw >> 8) & 0x0F
        return {"running": running, "fault": fault, "mode": mode}

    def get_power(self):
        raw = self.read_register(200)
        return raw * 0.1  # 倍率硬編碼

# 問題：換了一款 PCS，倍率不一樣？bit 位置不一樣？
# 你得複製整個類別，改掉散落各處的魔術數字
```

### 何時不該用

如果你的系統只接一台設備，而且永遠不會換型號，把轉換邏輯寫在讀取函式裡也無妨。Adapter Pattern 的價值在於**需要對接多種格式不同的資料來源**。在工業場域，光一個案場可能就有三五種不同品牌的設備，每種的暫存器格式都不一樣——這正是 Adapter 大顯身手的場景。

### 練習題

`csp_lib/equipment/core/transform.py` 中還有 `InverseTransform` 和 `BoolTransform`。`InverseTransform` 的計算是 `(value - offset) / magnitude`，剛好是 `ScaleTransform` 的反向。想一想：為什麼需要反向轉換？（提示：讀取時要從 raw → 物理量，寫入時要從物理量 → raw。）

---

## Pipeline Pattern — 資料處理的流水線

### 日常比喻

想像一條工廠的組裝流水線。原材料從一端進去，經過第一站切割、第二站研磨、第三站烤漆、第四站品檢，最後成品從另一端出來。每一站只負責自己的工序，不需要知道前面做了什麼、後面要做什麼。

Pipeline Pattern 就是把多個處理步驟串聯起來，前一步的輸出就是後一步的輸入。

### 問題場景

上一節我們看到了各種 Transform。但現實中，一筆資料往往需要**連續套用多個轉換**。例如溫度資料：

1. 先縮放：`raw * 0.1 - 40`
2. 再四捨五入到小數一位
3. 最後限制在 -40 ~ 85 度的合理範圍

你的第一直覺可能是巢狀呼叫：

```python
# 直覺解法：巢狀函式呼叫
result = clamp(round(scale(raw, 0.1, -40), 1), -40, 85)
# 從裡到外讀，反直覺
# 如果再多幾層，可讀性會變成災難
```

或者連續賦值：

```python
# 稍好：連續賦值
v1 = raw * 0.1 - 40
v2 = round(v1, 1)
v3 = max(-40, min(85, v2))
# 問題：這些邏輯散落在讀取函式裡
# 每個點位都要寫一遍類似的程式碼
```

### 模式登場

csp_lib 的 `ProcessingPipeline` 把多個 `TransformStep` 串聯成一條流水線：

```python
# csp_lib/equipment/core/pipeline.py

@dataclass(frozen=True)
class ProcessingPipeline:
    """
    資料處理管線

    將多個 TransformStep 串聯執行，依序處理資料。
    """

    steps: tuple[TransformStep, ...]

    def process(self, raw_value: Any) -> Any:
        """執行管線處理"""
        value = raw_value
        for step in self.steps:
            value = step.apply(value)
        return value

    def __len__(self) -> int:
        return len(self.steps)

    def __bool__(self) -> bool:
        return len(self.steps) > 0
```

核心邏輯只有三行：初始值 → for 迴圈逐步套用 → 回傳結果。

搭配便捷建構函式：

```python
# csp_lib/equipment/core/pipeline.py

def pipeline(*steps: TransformStep) -> ProcessingPipeline:
    """便捷建構函數"""
    return ProcessingPipeline(steps=steps)
```

使用起來非常直觀：

```python
from csp_lib.equipment import pipeline, ScaleTransform, RoundTransform, ClampTransform

# 溫度處理管線：縮放 → 四捨五入 → 值域限制
temp_pipeline = pipeline(
    ScaleTransform(magnitude=0.1, offset=-40),
    RoundTransform(decimals=1),
    ClampTransform(min_value=-40, max_value=85),
)

result = temp_pipeline.process(250)
# Step 1: 250 * 0.1 - 40 = -15.0
# Step 2: round(-15.0, 1) = -15.0
# Step 3: clamp(-40, -15.0, 85) = -15.0
# result = -15.0
```

幾個設計亮點：

**`frozen=True`**：Pipeline 本身也是不可變的。一旦建立，步驟不會被修改。這意味著多個點位可以共享同一個 Pipeline 物件。

**`tuple[TransformStep, ...]`**：步驟用 tuple 而非 list 存放，進一步強化不可變性。

**`__bool__`**：空的 Pipeline 會被判定為 False，方便做條件檢查（`if pipeline: pipeline.process(raw)`）。

**Pipeline 是宣告式的**：你描述的是「要做什麼」（縮放、四捨五入、限制），而不是「怎麼做」。讀程式碼的人一看就知道這個點位的轉換流程。

### 反面教材

```python
# 反面教材：每個點位各寫各的轉換
class DeviceReader:
    def read_temperature(self):
        raw = self.read_register(100)
        v = raw * 0.1 - 40
        v = round(v, 1)
        v = max(-40, min(85, v))
        return v

    def read_power(self):
        raw = self.read_register(200)
        v = raw * 0.1
        v = round(v, 2)
        v = max(0, min(500, v))  # 不同的範圍
        return v

    def read_voltage(self):
        raw = self.read_register(300)
        v = raw * 0.01
        v = round(v, 1)
        return v  # 忘了加 clamp？

    # 50 個點位 × 每個都手寫轉換 = 維護噩夢
```

### 何時不該用

如果轉換步驟只有一步（例如只需要乘個倍率），直接用單個 `ScaleTransform` 就好，不需要包一層 Pipeline。Pipeline 的價值在於**需要串聯兩個以上的處理步驟**。

### 練習題

試著為以下場景設計 Pipeline：一個暫存器存的是 PCS 的運行狀態字 (16-bit)，你需要先用 `BitExtractTransform(bit_offset=8, bit_length=4)` 提取 bit 8-11 的模式值，然後用 `EnumMapTransform(mapping={0: "STANDBY", 1: "CHARGE", 2: "DISCHARGE"})` 映射成字串。寫出這個 Pipeline 的建構程式碼。

---

## Builder Pattern — 複雜物件的分步建構

### 日常比喻

你去速食店點餐。如果只是點一杯美式咖啡，直接說就好。但如果你要點一個「客製套餐」：大薯、不要洋蔥的牛肉堡、可樂去冰、加一份蘋果派、再多一包番茄醬——你需要一個「組合過程」，一步步把各個元件湊成一份完整的訂單。

Builder Pattern 就是為「建構過程很複雜」的物件提供一個分步組裝的介面。

### 問題場景

控制策略需要一個 `StrategyContext`，裡面包含 SOC、電壓、功率、上一次的命令、系統基準值等等。這些資料來自不同的設備：

- SOC 來自 BMS
- 電壓來自電表
- 功率也來自電表，但可能是多台電表的聚合值
- 系統基準值來自設定檔

你的第一直覺可能是手動一個一個塞：

```python
# 直覺解法：手動建構 context
def build_context():
    ctx = StrategyContext()

    # 讀 BMS 的 SOC
    bms = get_device("bms_01")
    if bms and bms.is_responsive:
        ctx.soc = bms.latest_values.get("soc")

    # 讀電表的電壓
    meter = get_device("meter_01")
    if meter and meter.is_responsive:
        ctx.extra["voltage"] = meter.latest_values.get("voltage")

    # 讀多台 PCS 的功率並加總
    pcs_list = get_devices_by_trait("pcs")
    powers = []
    for pcs in pcs_list:
        if pcs.is_responsive:
            p = pcs.latest_values.get("active_power")
            if p is not None:
                powers.append(p)
    if powers:
        ctx.extra["total_power"] = sum(powers)

    return ctx
```

問題是什麼？

1. **硬編碼**：設備 ID、點位名稱、聚合邏輯全部寫死在函式裡。換個案場就要重寫。
2. **不可配置**：如果客戶 A 的 SOC 來自 BMS，客戶 B 的 SOC 來自 PCS 內建的 BMS 模組，你得寫兩個版本的 `build_context()`。
3. **難以測試**：要測試 context 建構邏輯，得 mock 所有設備。

### 模式登場

csp_lib 的 `ContextBuilder` 將「哪個設備的哪個點位映射到 context 的哪個欄位」抽離成可配置的 `ContextMapping`，然後用 Builder 的方式組裝 `StrategyContext`：

```python
# csp_lib/integration/context_builder.py

class ContextBuilder:
    """
    設備值 → StrategyContext 建構器

    透過 ContextMapping 列表，從 DeviceRegistry 中的設備讀取點位值，
    映射並聚合後填入 StrategyContext。
    """

    def __init__(
        self,
        registry: DeviceRegistry,
        mappings: list[ContextMapping],
        system_base: SystemBase | None = None,
        capability_mappings: list[CapabilityContextMapping] | None = None,
    ) -> None:
        self._registry = registry
        self._mappings = mappings
        self._system_base = system_base
        self._capability_mappings = capability_mappings or []

    def build(self) -> StrategyContext:
        """建構 StrategyContext"""
        ctx = StrategyContext(
            last_command=Command(),
            system_base=self._system_base,
        )

        for mapping in self._mappings:
            value = self._resolve_value(mapping)
            self._set_context_field(ctx, mapping.context_field, value)

        for cap_mapping in self._capability_mappings:
            value = self._resolve_capability_value(cap_mapping)
            self._set_context_field(ctx, cap_mapping.context_field, value)

        return ctx
```

`build()` 方法的流程清晰：建立空 context → 遍歷所有映射 → 解析每個映射的值 → 填入 context。

映射的解析支援兩種模式：

```python
def _resolve_value(self, mapping: ContextMapping) -> Any:
    """解析單一映射的值"""
    if mapping.device_id is not None:
        raw = self._read_single_device(mapping)     # 單一設備模式
    else:
        raw = self._read_trait_aggregate(mapping)     # 多設備聚合模式

    if raw is None:
        return mapping.default

    if mapping.transform is not None:
        try:
            raw = mapping.transform(raw)
        except Exception:
            logger.warning(f"Transform failed for mapping '{mapping.context_field}', using default.")
            return mapping.default

    return raw
```

而多設備聚合模式支援內建聚合函式和自訂聚合：

```python
def _read_trait_aggregate(self, mapping: ContextMapping) -> Any:
    """trait 模式：收集所有 responsive 設備的值並聚合"""
    devices = self._registry.get_responsive_devices_by_trait(mapping.trait)
    if not devices:
        return None

    values = []
    for device in devices:
        v = device.latest_values.get(mapping.point_name)
        if v is not None:
            values.append(v)

    if not values:
        return None

    if mapping.custom_aggregate is not None:
        try:
            return mapping.custom_aggregate(values)
        except Exception:
            return None

    return apply_builtin_aggregate(mapping.aggregate, values)
```

使用時只需要宣告映射關係：

```python
# 用宣告式的方式定義映射
builder = ContextBuilder(
    registry=device_registry,
    mappings=[
        # SOC 來自 BMS
        ContextMapping(
            point_name="soc",
            context_field="soc",
            device_id="bms_01",
            default=50.0,
        ),
        # 電壓來自電表
        ContextMapping(
            point_name="voltage",
            context_field="extra.voltage",
            device_id="meter_01",
        ),
        # 總功率 = 所有 PCS 的功率加總
        ContextMapping(
            point_name="active_power",
            context_field="extra.total_power",
            trait="pcs",
            aggregate=AggregateFunc.SUM,
            default=0.0,
        ),
    ],
    system_base=SystemBase(p_base=500, q_base=250),
)

# 每次需要 context 時呼叫 build()
context = builder.build()
```

注意 `_set_context_field` 支援點號路徑，可以把值塞進 `extra` 字典：

```python
@staticmethod
def _set_context_field(ctx: StrategyContext, field: str, value: Any) -> None:
    """支援點號路徑："soc" → ctx.soc, "extra.xxx" → ctx.extra["xxx"]"""
    if field.startswith("extra."):
        key = field[len("extra."):]
        ctx.extra[key] = value
    else:
        setattr(ctx, field, value)
```

而且 `ContextBuilder.build` 的簽名 `Callable[[], StrategyContext]` 剛好符合 `StrategyExecutor` 的 `context_provider` 參數：

```python
# StrategyExecutor 接受一個 context_provider
executor = StrategyExecutor(
    context_provider=builder.build,  # 直接傳入 build 方法
    on_command=send_command,
)
```

這是一個精妙的設計——Builder 的 `build()` 既是建構方法，又是符合 Executor 要求的 callback 介面。不需要額外的 adapter 或 wrapper。

### 反面教材

```python
# 反面教材：每個案場寫一個專用的 context builder
def build_context_site_a():
    """台南案場專用"""
    ctx = StrategyContext()
    ctx.soc = read_device("bms_01", "soc")
    ctx.extra["voltage"] = read_device("meter_01", "v_ab")
    # ... 50 行硬編碼
    return ctx

def build_context_site_b():
    """彰化案場專用"""
    ctx = StrategyContext()
    ctx.soc = read_device("bms_main", "battery_soc")  # 不同的設備 ID
    ctx.extra["voltage"] = read_device("pm5350_01", "voltage_ln")  # 不同的點位名
    # ... 又 50 行硬編碼
    return ctx

# 問題：兩個函式 90% 的結構一樣，只是設備 ID 和點位名不同
# 第三個案場？再複製一份。第十個案場？炸了。
```

### 何時不該用

如果你的 context 只需要一兩個欄位，而且不會因案場而異，直接手動建構就好。Builder Pattern 的價值在於**物件的建構過程複雜且需要可配置**。在 csp_lib 的場景中，一個中型案場可能有 20+ 個映射，聚合邏輯各不相同，Builder 才能真正發揮價值。

### 練習題

`ContextBuilder` 還支援 `CapabilityContextMapping`——不指定具體的點位名稱，而是用 capability（能力）和 slot（插槽）來解析。思考一下：這種設計的好處是什麼？（提示：如果兩台不同品牌的 PCS 都有「讀取 SOC」的能力，但暫存器位址和點位名稱不同......）

---

## Configuration Object — 不可變的設定檔

### 日常比喻

你去銀行開戶時填了一張申請表。這張表格有各種欄位：姓名、身分證號、聯絡電話、開戶類型......但不是每個欄位都是必填的。填完之後，這份文件就是你的帳戶設定——銀行不會在你不知道的情況下偷改你的聯絡電話。

Configuration Object 就是這樣的「申請表」：結構化、有驗證、建立後不可變。

### 問題場景

`ContextBuilder` 需要知道「哪個設備的哪個點位映射到哪個 context 欄位」。你的第一直覺可能是用 dict：

```python
# 直覺解法：用 dict 表示映射
mapping = {
    "device_id": "bms_01",
    "point": "soc",
    "target": "soc",
    "default": 50.0,
}

# 問題：
# - 打錯 key 名稱？ mapping["device_id"] vs mapping["deviceId"]
# - 忘記必填欄位？ 直到 runtime 才發現
# - 同時設了 device_id 和 trait？ 語意衝突，沒人驗證
```

### 模式登場

csp_lib 的 `ContextMapping` 是一個 frozen dataclass，帶有建構時驗證：

```python
# csp_lib/integration/schema.py

@dataclass(frozen=True)
class ContextMapping:
    """
    設備點位 → StrategyContext 欄位映射

    device_id 模式用於單一設備讀取；trait 模式用於多設備聚合。
    兩者必須恰好設定其一。
    """

    point_name: str
    context_field: str
    device_id: str | None = None
    trait: str | None = None
    aggregate: AggregateFunc = AggregateFunc.AVERAGE
    custom_aggregate: Callable[[list[Any]], Any] | None = None
    default: Any = None
    transform: Callable[[Any], Any] | None = None

    def __post_init__(self) -> None:
        _validate_device_or_trait(self.device_id, self.trait)
```

驗證函式確保 `device_id` 和 `trait` 恰好設定其一：

```python
def _validate_device_or_trait(device_id: str | None, trait: str | None) -> None:
    if device_id is not None and trait is not None:
        raise ValueError("Cannot set both device_id and trait; choose exactly one.")
    if device_id is None and trait is None:
        raise ValueError("Must set either device_id or trait; neither was provided.")
```

這個驗證在物件建立時就執行。如果你寫了：

```python
# 這會在建立時立即拋出 ValueError
bad = ContextMapping(
    point_name="soc",
    context_field="soc",
    device_id="bms_01",
    trait="bms",  # 不能同時設定 device_id 和 trait！
)
```

錯誤在「寫程式碼的時候」就被抓到，而不是在「凌晨三點系統跑到這段程式碼」的時候才爆炸。

同樣的模式也用在 `CommandMapping`、`HeartbeatMapping`、`DataFeedMapping` 等其他映射物件上：

```python
@dataclass(frozen=True)
class CommandMapping:
    """Command 欄位 → 設備寫入映射"""

    command_field: str
    point_name: str
    device_id: str | None = None
    trait: str | None = None
    transform: Callable[[float], Any] | None = None

    def __post_init__(self) -> None:
        _validate_device_or_trait(self.device_id, self.trait)


@dataclass(frozen=True)
class HeartbeatMapping:
    """心跳寫入映射"""

    point_name: str
    device_id: str | None = None
    trait: str | None = None
    mode: HeartbeatMode = HeartbeatMode.TOGGLE
    constant_value: int = 1
    increment_max: int = 65535

    def __post_init__(self) -> None:
        _validate_device_or_trait(self.device_id, self.trait)
```

注意每一個映射都共享相同的 `_validate_device_or_trait` 驗證邏輯。這是因為 `device_id` / `trait` 的互斥規則是跨映射類型的通用約束。

還有更進階的 `CapabilityContextMapping`，增加了 capability slot 的驗證：

```python
@dataclass(frozen=True)
class CapabilityContextMapping:
    """Capability-driven context mapping"""

    capability: Capability
    slot: str
    context_field: str
    device_id: str | None = None
    trait: str | None = None
    aggregate: AggregateFunc = AggregateFunc.AVERAGE
    custom_aggregate: Callable[[list[Any]], Any] | None = None
    default: Any = None
    transform: Callable[[Any], Any] | None = None

    def __post_init__(self) -> None:
        if self.device_id is not None and self.trait is not None:
            raise ValueError("Cannot set both device_id and trait.")
        if self.slot not in self.capability.read_slots:
            raise ValueError(
                f"Slot '{self.slot}' not in capability '{self.capability.name}' "
                f"read_slots: {self.capability.read_slots}"
            )
```

這裡多了一層驗證：`slot` 必須存在於 `capability` 的 `read_slots` 中。如果你寫了一個 capability 沒有的 slot，建立時就會報錯。這種「提早失敗」的設計哲學，在工業控制中尤其重要——你絕對不想在設備已經開始運行之後，才發現某個映射設定是無效的。

### 反面教材

```python
# 反面教材：用字典和字串表示設定
config = {
    "mappings": [
        {"device": "bms_01", "point": "soc", "target": "soc"},
        {"device": "meter", "point": "voltage", "target": "extra.voltage"},
    ]
}

# 問題 1: 打錯 key 不會有任何提示
config["mappings"][0]["divice"]  # KeyError 在 runtime 才爆

# 問題 2: 沒有型別提示，IDE 無法自動補全

# 問題 3: 可以隨時修改
config["mappings"][0]["point"] = "wrong_name"  # 靜默修改，沒人知道

# 問題 4: 沒有驗證
config["mappings"].append({"device": "x", "trait": "y", "point": "z", "target": "w"})
# device 和 trait 同時存在？沒人檢查
```

### 何時不該用

如果設定只有兩三個欄位，而且來源固定（例如環境變數），用 dict 或 NamedTuple 就夠了。Configuration Object 的價值在於**設定結構複雜、有交叉驗證規則、且需要在多處傳遞**。

### 練習題

觀察 `HeartbeatMapping` 的 `mode` 欄位，它使用 `HeartbeatMode` 枚舉（`TOGGLE`、`INCREMENT`、`CONSTANT`）。想一想：為什麼用 Enum 而不是字串？（提示：如果有人寫了 `mode="togle"`......）

---

## 完整資料流走訪

現在讓我們把四個模式串起來，走訪一筆資料從 Modbus 暫存器到控制策略的完整旅程。

**場景**：一台 PCS 的暫存器 5004 存著 SOC 值（raw: 850，倍率 0.1，代表 85.0%）。

### Step 1：Adapter 轉換原始值

設備讀取到 raw = 850，經過 `ScaleTransform` 轉換為物理量。

```python
# 設備點位定義中配置的 transform
soc_transform = ScaleTransform(magnitude=0.1)
# 套用：850 * 0.1 = 85.0
```

### Step 2：Pipeline 串聯多步處理

如果需要多步處理（例如加上四捨五入和值域限制）：

```python
soc_pipeline = pipeline(
    ScaleTransform(magnitude=0.1),
    RoundTransform(decimals=1),
    ClampTransform(min_value=0, max_value=100),
)
# 850 → 85.0 → 85.0 → 85.0
```

### Step 3：Configuration Object 定義映射

用 `ContextMapping` 宣告這個點位映射到 context 的哪個欄位：

```python
mapping = ContextMapping(
    point_name="soc",
    context_field="soc",
    device_id="pcs_01",
    default=50.0,
    transform=lambda v: v * 0.1,  # 也可以在 mapping 層做簡單轉換
)
```

### Step 4：Builder 組裝 StrategyContext

`ContextBuilder.build()` 遍歷所有映射，從 DeviceRegistry 讀取值，套用 transform，填入 context：

```python
builder = ContextBuilder(
    registry=registry,
    mappings=[
        ContextMapping(point_name="soc", context_field="soc", device_id="pcs_01"),
        ContextMapping(point_name="voltage", context_field="extra.voltage", device_id="meter_01"),
        ContextMapping(point_name="active_power", context_field="extra.total_power",
                       trait="pcs", aggregate=AggregateFunc.SUM),
    ],
    system_base=SystemBase(p_base=500, q_base=250),
)

context = builder.build()
# context.soc = 85.0
# context.extra["voltage"] = 380.0
# context.extra["total_power"] = 450.0
# context.system_base = SystemBase(p_base=500, q_base=250)
```

### Step 5：策略使用 Context 計算 Command

```python
# QV 策略從 context 讀取電壓，計算無功功率
command = qv_strategy.execute(context)
# command = Command(p_target=200.0, q_target=35.5)
```

整個流程中：
- **Adapter（Transform）** 將 raw register 轉為物理量
- **Pipeline** 串聯多步轉換
- **Configuration Object（ContextMapping）** 宣告式定義映射關係
- **Builder（ContextBuilder）** 根據映射組裝完整的 StrategyContext

四個模式環環相扣，把「從 Modbus 暫存器到控制決策」這個複雜的過程拆成了清晰的層次。每一層都可以獨立測試、獨立替換、獨立配置。

---

## 重點回顧

| 模式 | 解決什麼問題 | csp_lib 中的角色 | 關鍵程式碼 |
|------|-------------|-----------------|-----------|
| **Adapter** | 異質介面的統一 | 原始暫存器值 → 物理量 | `TransformStep` Protocol + 各種 Transform |
| **Pipeline** | 多步處理的串聯 | 可組合的資料處理流水線 | `ProcessingPipeline` + `pipeline()` |
| **Builder** | 複雜物件的分步建構 | 設備值 → StrategyContext | `ContextBuilder.build()` |
| **Configuration Object** | 結構化、可驗證的設定 | 映射規格的不可變宣告 | `ContextMapping` 等 frozen dataclass |

一個重要的觀察：這四個模式的共通點是 **`frozen=True`**。Transform 是 frozen 的、Pipeline 是 frozen 的、ContextMapping 是 frozen 的。在工業控制系統中，「不可變」不是教條，而是安全保障——你不會想在系統運行中突然發現某個映射的目標欄位被改了。

---

## 下篇預告

到目前為止，我們看的都是「資料流進來、處理、送出去」的單向流程。但真實的工業系統是**反應式**的：設備狀態改變時要發通知、告警觸發時要通知所有訂閱者、系統狀態機要在不同狀態間轉換。

下一篇 [**反應的藝術：Observer × Chain of Responsibility × State**](./00g-patterns-reactive.md)，我們會看 csp_lib 如何用事件驅動的方式處理設備告警、狀態變化和連鎖反應。
