# 給電機/機械工程師的軟體速成

> **從零打造工業控制協定框架：給軟體工程師的 CSP Library 實戰系列**
>
> Part 0 — 預備知識篇 | Article 00D
>
> [上一篇：<<< 網路基礎：TCP、序列通訊與封包的概念](./00c-networking-fundamentals.md) | [下一篇：控制的骨架：Strategy × Template Method × Command >>>](./00e-patterns-control.md)

---

## 目錄

1. [為什麼電機/機械工程師需要會寫程式？](#為什麼電機機械工程師需要會寫程式)
2. [Git 基礎：版本控制的概念](#git-基礎版本控制的概念)
3. [Python 虛擬環境](#python-虛擬環境)
4. [Python 語法速覽](#python-語法速覽)
5. [VS Code 設定建議](#vs-code-設定建議)
6. [「Hello Modbus」第一個程式](#hello-modbus第一個程式)
7. [從 PLC 梯形圖到 Python：思維轉換](#從-plc-梯形圖到-python思維轉換)
8. [重點回顧](#重點回顧)
9. [下篇預告](#下篇預告)

---

## 為什麼電機/機械工程師需要會寫程式？

你可能花了四年學電路學、電力系統、控制理論、機構設計，現在有人跟你說：「你該學 Python。」你的第一反應可能是：「我又不是要當軟體工程師，為什麼要學這個？」

這是個合理的質疑，讓我用幾個產業趨勢來回答：

**1. 能源管理系統（EMS）正在改變電力產業**

過去，電力系統的調度靠 SCADA 畫面和人工判斷。現在，隨著再生能源和儲能系統的普及，EMS 需要即時地蒐集數百個測點的資料、執行最佳化演算法、自動下達控制指令。這些功能不可能只靠 PLC 的梯形圖完成——你需要能處理資料庫、網路通訊、演算法的程式語言。

**2. 智慧電網要求跨領域整合**

台電的電力交易平台、需量反應機制、自動頻率控制（AFC）——這些都需要軟體系統與電力設備的深度整合。理解電力系統的人很多，會寫程式的人也很多，但**同時理解兩邊的人很少**。這是你的競爭優勢。

**3. IoT 讓設備數據變得可程式化**

以前你用三用電表量一個訊號，記在筆記本上。現在，每台設備每秒產生數十個測點的資料，儲存在資料庫裡。你需要程式來分析這些資料——計算效率、預測壽命、偵測異常。Excel 可以處理小規模資料，但當你面對一年份、數百台設備的資料時，Python 是更實際的選擇。

**你不需要成為軟體工程師**，但你需要具備足夠的程式能力來：
- 讀取設備資料並做基本分析
- 理解開發團隊的程式碼，能有效溝通需求
- 撰寫簡單的自動化腳本，取代重複性的手動作業

這篇文章就是幫你跨過這個門檻。

---

## Git 基礎：版本控制的概念

### 用 AutoCAD 存檔來比喻

假設你在畫一張電路圖。你可能會這樣存檔：

```
電路圖_v1.dwg
電路圖_v2_修改電容值.dwg
電路圖_v3_老闆改的.dwg
電路圖_v3_老闆改的_最終版.dwg
電路圖_v3_老闆改的_最終版_真的最終.dwg
```

這個情境你一定不陌生。問題是：
- 不知道每個版本改了什麼
- 想回到兩週前的版本，找不到了
- 兩個人同時修改，不知道怎麼合併

**Git 就是解決這些問題的工具**。它幫你追蹤每一次修改的內容、時間、作者，讓你隨時回到任何一個歷史版本。

### 核心概念

```
Working Directory     →  Staging Area     →  Repository
（你正在編輯的檔案）      （準備要提交的變更）    （已保存的歷史紀錄）
        │                       │                    │
    git add ───────────>    git commit ──────────>   │
        │                       │                    │
        <──────────── git checkout ──────────────────│
```

### 五個必學指令

```bash
# 1. 初始化一個 Git 倉庫（在專案資料夾中執行一次）
git init

# 2. 把修改過的檔案加入暫存區（像是「我決定要存這些變更」）
git add main.py                   # 加入特定檔案
git add .                         # 加入所有變更（新手先別用這個）

# 3. 提交變更（像是「存檔」，附帶一段說明）
git commit -m "修改 PCS 功率上限為 500kW"

# 4. 建立分支（像是「複製一份來改，不影響原本的」）
git branch feature/alarm-system   # 建立分支
git checkout feature/alarm-system # 切換到該分支

# 5. 合併分支（像是「把改好的東西合回主線」）
git checkout main                 # 先切回主線
git merge feature/alarm-system    # 把功能分支合進來
```

### GitHub 基本操作

GitHub 是 Git 的雲端託管平台，就像 Google Drive 但專門給程式碼用。

```bash
# 把本地的倉庫推到 GitHub
git remote add origin https://github.com/你的帳號/你的專案.git
git push -u origin main

# 從 GitHub 下載別人的專案
git clone https://github.com/someone/some-project.git

# 把別人的最新修改拉下來
git pull origin main
```

**給工程師的建議**：一開始不用理解 Git 的所有功能。只要會 `add`、`commit`、`push`、`pull` 四個指令，就足以應付大部分場景。等到你需要「兩個人同時改同一個檔案」時，再學 `branch` 和 `merge` 也不遲。

---

## Python 虛擬環境

### 為什麼不直接裝在系統？

假設你有兩個專案：

- 專案 A 需要 `pymodbus==3.5.0`
- 專案 B 需要 `pymodbus==3.7.2`

如果你把套件都裝在系統的 Python 裡，兩個版本會打架。虛擬環境就是讓每個專案有自己獨立的套件空間，互不干擾。

用一個比喻：虛擬環境就像實驗室的不同工作台。每張工作台上有自己的工具組，你在 A 工作台用 3.5 版的示波器，不影響 B 工作台上 3.7 版的示波器。

### venv、uv、pip 的關係

```
venv   →  建立虛擬環境的工具（Python 內建）
pip    →  安裝套件的工具（Python 內建）
uv     →  更快的替代方案（同時取代 venv + pip）
```

**用 venv + pip（傳統方式）：**

```bash
# 建立虛擬環境
python -m venv .venv

# 啟用虛擬環境
# Windows:
.venv\Scripts\activate
# macOS/Linux:
source .venv/bin/activate

# 安裝套件
pip install pymodbus

# 離開虛擬環境
deactivate
```

**用 uv（推薦，更快更方便）：**

```bash
# 安裝 uv
pip install uv

# 建立虛擬環境 + 安裝套件（一步完成）
uv sync

# 在虛擬環境中執行指令（不需要手動 activate）
uv run python main.py
uv run pytest tests/
```

`uv` 是近年 Python 生態中最受歡迎的新工具之一。它的安裝速度比 `pip` 快 10-100 倍，而且可以直接管理虛擬環境，不需要手動 `activate`/`deactivate`。本系列的 csp_lib 專案就是使用 `uv` 來管理依賴。

---

## Python 語法速覽

如果你有 C 語言或 MATLAB 的經驗，學 Python 會非常快。以下是幾個重要語法特性，我會用你熟悉的概念來類比。

### Type Hints：像宣告變數型別

C 語言強制宣告型別，Python 原本不需要。但現代 Python 推薦用 type hints 來標注型別——它不會強制執行，但能讓工具幫你檢查錯誤。

```python
# C 語言
# float power = 50.5;
# int register_count = 10;

# Python（沒有 type hints）
power = 50.5
register_count = 10

# Python（有 type hints）— 推薦寫法
power: float = 50.5
register_count: int = 10

# 函式的 type hints
def calculate_power(voltage: float, current: float) -> float:
    """計算功率"""
    return voltage * current

# 複合型別
from typing import Optional

def read_register(address: int, timeout: float = 1.0) -> Optional[int]:
    """讀取暫存器，失敗回傳 None"""
    ...
```

### Dataclass：像 C 的 struct

在 C 語言中你用 `struct` 來定義結構化的資料。Python 的 `dataclass` 做同樣的事，但更方便。

```python
# C 語言
# struct Point {
#     char name[50];
#     int address;
#     float scale;
# };

# Python dataclass
from dataclasses import dataclass

@dataclass
class Point:
    name: str
    address: int
    scale: float

# 建立實例
p = Point(name="active_power", address=5000, scale=0.1)
print(p.name)      # "active_power"
print(p.address)   # 5000

# frozen=True 代表建立後不能修改（像 const struct）
@dataclass(frozen=True)
class Config:
    host: str
    port: int
    timeout: float = 1.0   # 預設值

config = Config(host="192.168.1.100", port=502)
# config.port = 503   # 這會報錯！frozen 的 dataclass 不可修改
```

### async def：像中斷處理（ISR）的概念

如果你寫過微控制器程式，你一定知道中斷（Interrupt）。當硬體事件發生時，CPU 暫停目前的工作，跳去執行 ISR（Interrupt Service Routine），執行完再回來。

Python 的 `async/await` 概念類似，但是是協作式的——程式主動告訴排程器「我現在在等 I/O，你去做別的事吧」。

```python
import asyncio

# 想像這是兩個設備的讀取任務
async def read_device_a():
    print("開始讀取設備 A...")
    await asyncio.sleep(1)  # 模擬等待設備回應（像等中斷）
    print("設備 A 回應：50.5 kW")
    return 50.5

async def read_device_b():
    print("開始讀取設備 B...")
    await asyncio.sleep(1.5)  # 設備 B 回應慢一點
    print("設備 B 回應：85.0%")
    return 85.0

async def main():
    # 同時發出兩個讀取請求（不需要等 A 讀完才讀 B）
    results = await asyncio.gather(
        read_device_a(),
        read_device_b(),
    )
    print(f"結果：{results}")  # [50.5, 85.0]
    # 總耗時約 1.5 秒（不是 2.5 秒！）

asyncio.run(main())
```

**關鍵概念：**
- `async def` 定義一個「可以被暫停的函式」
- `await` 是暫停點——「我現在在等，排程器去做別的事」
- `asyncio.gather()` 同時執行多個任務，等全部完成

這在工業通訊中非常實用：你可以同時對多台設備發出讀取請求，而不需要一台一台等。

### List Comprehension 和 Dict

```python
# 傳統寫法（像 C 的 for 迴圈）
addresses = []
for i in range(10):
    addresses.append(5000 + i * 2)

# List comprehension（Python 風格，一行搞定）
addresses = [5000 + i * 2 for i in range(10)]
# [5000, 5002, 5004, 5006, 5008, 5010, 5012, 5014, 5016, 5018]

# 加上條件過濾
even_addresses = [addr for addr in addresses if addr % 4 == 0]
# [5000, 5004, 5008, 5012, 5016]

# Dict（字典）— 類似 key-value 對照表
device_status = {
    0: "停機",
    1: "運行",
    2: "故障",
    3: "維護中",
}

code = 1
print(device_status[code])  # "運行"
print(device_status.get(99, "未知"))  # "未知"（找不到時的預設值）
```

---

## VS Code 設定建議

VS Code（Visual Studio Code）是目前最受歡迎的程式編輯器。以下是針對 Python 工業開發的推薦設定。

### 必裝 Extensions

在 VS Code 左側的 Extensions 面板中搜尋並安裝：

| Extension | 用途 |
|-----------|------|
| **Python** (Microsoft) | Python 語法支援、除錯、執行 |
| **Pylance** (Microsoft) | 智慧型別檢查、自動補全 |
| **Ruff** (Astral Software) | 程式碼格式化和 lint（取代 Black + Flake8） |

### 推薦設定

按 `Ctrl+Shift+P` → 輸入 `Preferences: Open Settings (JSON)` → 加入以下設定：

```json
{
    "python.defaultInterpreterPath": ".venv/Scripts/python",
    "[python]": {
        "editor.defaultFormatter": "charliermarsh.ruff",
        "editor.formatOnSave": true,
        "editor.codeActionsOnSave": {
            "source.fixAll.ruff": "explicit",
            "source.organizeImports.ruff": "explicit"
        }
    },
    "python.analysis.typeCheckingMode": "basic"
}
```

這些設定會：
- 自動使用虛擬環境中的 Python
- 存檔時自動格式化程式碼
- 開啟基本的型別檢查

### 實用快捷鍵

| 快捷鍵 | 功能 |
|--------|------|
| `Ctrl+`` ` | 開啟終端機 |
| `F5` | 啟動除錯 |
| `F12` | 跳到函式定義 |
| `Ctrl+Shift+F` | 全域搜尋 |
| `Ctrl+P` | 快速開啟檔案 |

---

## 「Hello Modbus」第一個程式

讓我們寫第一個真正有用的程式：用 Python 讀取一台 Modbus 設備的暫存器。

### 前置準備

```bash
# 建立專案資料夾
mkdir hello-modbus
cd hello-modbus

# 建立虛擬環境並安裝 pymodbus
python -m venv .venv
# Windows:
.venv\Scripts\activate
# macOS/Linux:
# source .venv/bin/activate

pip install pymodbus
```

### 啟動一個模擬設備（不需要真設備）

pymodbus 自帶模擬器，可以模擬一台 Modbus TCP 設備。建立 `simulator.py`：

```python
"""Modbus TCP 模擬器：模擬一台有基本測點的設備"""
import asyncio
import struct

from pymodbus.datastore import (
    ModbusSequentialDataBlock,
    ModbusSlaveContext,
    ModbusServerContext,
)
from pymodbus.server import StartAsyncTcpServer


def create_datastore():
    """建立模擬的暫存器資料"""
    # 建立一個有 100 個暫存器的資料區塊（位址從 0 開始）
    holding_registers = ModbusSequentialDataBlock(0, [0] * 100)

    # 在位址 0-1 寫入 Float32 的有功功率 (50.5 kW)
    power_bytes = struct.pack(">f", 50.5)
    reg_hi = int.from_bytes(power_bytes[0:2], "big")
    reg_lo = int.from_bytes(power_bytes[2:4], "big")
    holding_registers.setValues(1, [reg_hi, reg_lo])  # pymodbus 位址從 1 開始

    # 在位址 2 寫入 UInt16 的 SOC (850 → 85.0%)
    holding_registers.setValues(3, [850])

    # 在位址 3 寫入 UInt16 的設備狀態 (1 = 運行)
    holding_registers.setValues(4, [1])

    slave = ModbusSlaveContext(hr=holding_registers)
    return ModbusServerContext(slaves=slave, single=True)


async def main():
    context = create_datastore()
    print("Modbus TCP 模擬器啟動中... (port 5020)")
    print("暫存器內容：")
    print("  位址 0-1: 有功功率 = 50.5 kW (Float32)")
    print("  位址 2:   SOC = 850 (÷10 = 85.0%)")
    print("  位址 3:   狀態 = 1 (運行)")
    print("按 Ctrl+C 停止")

    await StartAsyncTcpServer(
        context=context,
        address=("127.0.0.1", 5020),
    )


if __name__ == "__main__":
    asyncio.run(main())
```

### 寫你的第一個 Modbus Client

建立 `read_device.py`：

```python
"""你的第一個 Modbus TCP 客戶端程式"""
import struct
from pymodbus.client import ModbusTcpClient


def main():
    # 1. 建立連線
    client = ModbusTcpClient("127.0.0.1", port=5020)
    connected = client.connect()
    print(f"連線狀態: {'成功' if connected else '失敗'}")

    if not connected:
        print("無法連線到設備，請確認模擬器是否啟動")
        return

    try:
        # 2. 讀取有功功率（位址 0，讀 2 個暫存器，因為 Float32 佔 2 個）
        result = client.read_holding_registers(address=0, count=2, slave=1)
        if not result.isError():
            # 將兩個 16-bit 暫存器組合成 Float32
            reg_hi = result.registers[0]
            reg_lo = result.registers[1]
            raw_bytes = struct.pack(">HH", reg_hi, reg_lo)
            power = struct.unpack(">f", raw_bytes)[0]
            print(f"有功功率: {power:.1f} kW")
        else:
            print(f"讀取功率失敗: {result}")

        # 3. 讀取 SOC（位址 2，讀 1 個暫存器）
        result = client.read_holding_registers(address=2, count=1, slave=1)
        if not result.isError():
            raw_soc = result.registers[0]
            soc = raw_soc * 0.1  # 倍率 0.1
            print(f"SOC: {soc:.1f}%")
        else:
            print(f"讀取 SOC 失敗: {result}")

        # 4. 讀取設備狀態（位址 3，讀 1 個暫存器）
        result = client.read_holding_registers(address=3, count=1, slave=1)
        if not result.isError():
            status_code = result.registers[0]
            status_map = {0: "停機", 1: "運行", 2: "故障"}
            status = status_map.get(status_code, "未知")
            print(f"設備狀態: {status} (code={status_code})")
        else:
            print(f"讀取狀態失敗: {result}")

    finally:
        # 5. 關閉連線
        client.close()
        print("連線已關閉")


if __name__ == "__main__":
    main()
```

### 執行方式

開兩個終端機：

```bash
# 終端機 1：啟動模擬器
python simulator.py

# 終端機 2：執行客戶端
python read_device.py
```

預期輸出：

```
連線狀態: 成功
有功功率: 50.5 kW
SOC: 85.0%
設備狀態: 運行 (code=1)
連線已關閉
```

恭喜！你剛剛完成了第一次 Modbus TCP 通訊。雖然只是讀取三個值，但這個程式展示了工業通訊的完整流程：建立連線 → 讀取暫存器 → 解碼原始數據 → 關閉連線。

注意到程式中 `struct.pack(">HH", ...)` 和 `struct.unpack(">f", ...)` 這段嗎？這就是上一篇文章提到的 Byte Ordering 問題的實際應用。`>` 代表 Big Endian，`H` 是 unsigned short（16 bits），`f` 是 float（32 bits）。

---

## 從 PLC 梯形圖到 Python：思維轉換

如果你有 PLC 程式設計的經驗，你習慣的是梯形圖（Ladder Diagram）的思維模式。從梯形圖轉換到 Python，需要適應幾個根本性的差異。

### 執行模式不同

**PLC 梯形圖：掃描式執行**

```
┌─────────────────────────────────────────┐
│ 掃描週期（通常 10-50ms）                   │
│                                          │
│  1. 讀取所有輸入 (Input Scan)              │
│  2. 從上到下執行所有梯級 (Logic Execution)  │
│  3. 更新所有輸出 (Output Update)           │
│  4. 回到步驟 1                            │
└─────────────────────────────────────────┘
```

PLC 的所有邏輯在每個掃描週期都會執行一次，就像一個永遠不停的 while 迴圈。所有的 I/O 在週期開始時統一讀取、週期結束時統一寫入。

**Python：事件驅動 / 循序執行**

```python
# Python 不會自動重複執行，你需要明確寫出迴圈
async def control_loop():
    while True:
        # 讀取
        power = await device.read("active_power")
        soc = await device.read("soc")

        # 邏輯判斷
        if soc < 10.0:
            await device.write("power_setpoint", 0)
            print("SOC 過低，停止放電")

        # 等待到下一個週期
        await asyncio.sleep(1.0)  # 1 秒一次
```

### 變數生命週期不同

**PLC**：所有變數在每個掃描週期都存在，不會被「回收」。即使你不用某個變數，它還是佔著記憶體。

**Python**：變數有作用域（scope）。函式結束後，局部變數就消失了。如果你需要在多次呼叫之間保持狀態，需要使用類別（class）或全域變數。

```python
class PowerController:
    def __init__(self):
        # 這些變數會在物件的整個生命週期中存在（像 PLC 的全域變數）
        self.last_power: float = 0.0
        self.alarm_active: bool = False

    def check_power(self, current_power: float) -> str:
        # 可以存取並修改 self 上的變數
        if current_power > 500.0 and not self.alarm_active:
            self.alarm_active = True
            return "ALARM"
        self.last_power = current_power
        return "OK"
```

### 錯誤處理不同

**PLC**：通常不會「當機」。即使某個梯級出錯，PLC 會繼續執行下一個梯級。最壞的情況是進入 FAULT 模式。

**Python**：一個未處理的例外（Exception）會終止你的程式。你必須明確地處理錯誤。

```python
# 不處理錯誤 — 設備斷線時程式會直接崩潰
power = client.read_holding_registers(address=0, count=2)

# 正確的做法 — 處理可能的錯誤
try:
    result = client.read_holding_registers(address=0, count=2)
    if result.isError():
        print(f"讀取失敗: {result}")
        power = None  # 用安全值代替
    else:
        power = decode_float32(result.registers)
except ConnectionError:
    print("設備斷線，嘗試重連...")
    reconnect()
```

### 概念對照表

| PLC 概念 | Python 對應 | 說明 |
|----------|-------------|------|
| 掃描週期 | `while True` + `sleep` | 需要自己實作迴圈 |
| 全域變數 | 類別屬性（`self.xxx`） | 用物件管理狀態 |
| 計時器（TON/TOF） | `asyncio.sleep()` / `time.monotonic()` | async 版更優雅 |
| 計數器（CTU/CTD） | `int` 變數 + 手動加減 | 沒有內建計數器 |
| 子程式（Subroutine） | 函式（`def`） | 基本上一樣 |
| 功能塊（FB） | 類別（`class`） | 封裝狀態和行為 |
| I/O 映射表 | 設備點位配置 | 對應暫存器位址 |

**一個重要的心態轉換**：PLC 的世界是「一切已知」的——你在設計階段就定義了所有 I/O、所有邏輯、所有計時器。Python 的世界更加動態——你可以在執行時期決定要讀哪些點位、要執行哪些策略。這種靈活性是 Python 的優勢，但也需要更多的紀律來確保系統的可靠性。

---

## 重點回顧

1. **電機/機械工程師學程式不是轉行**，而是增加你的跨域整合能力。EMS、智慧電網、IoT 都需要同時理解設備和軟體的人才。

2. **Git 是程式碼的版本控制工具**。記住四個指令就夠入門：`add`（標記變更）、`commit`（存檔）、`push`（上傳）、`pull`（下載）。

3. **Python 虛擬環境讓不同專案的套件互不干擾**。推薦使用 `uv` 來管理，它比傳統的 `venv` + `pip` 更快更方便。

4. **Python 語法對有 C/MATLAB 經驗的人很友善**：
   - Type hints ≈ 型別宣告
   - `dataclass` ≈ `struct`
   - `async/await` ≈ 中斷處理（但是協作式的）

5. **VS Code + Python + Pylance + Ruff** 是 Python 開發的黃金組合。開啟「存檔自動格式化」能省去很多格式煩惱。

6. **第一個 Modbus 程式只需要 30 行**。`pymodbus` 套件處理了連線管理和協定細節，你只需要知道要讀哪個位址、讀幾個暫存器、怎麼解碼。

7. **從 PLC 梯形圖到 Python 的最大差異**是執行模式（掃描式 vs 事件驅動）和錯誤處理（自動容錯 vs 需要明確處理）。理解這些差異，轉換就不難。

---

## 下篇預告

到目前為止，你已經具備了足夠的預備知識：網路基礎、開發工具、Python 語法、以及第一次 Modbus 通訊體驗。

下一篇我們將進入設計模式的領域——但不是教科書式的泛泛而談。我們會聚焦在工業控制系統中最常用的三個模式：

- **Strategy Pattern**：如何讓控制策略可以動態切換（PQ 模式 ↔ QV 模式）
- **Template Method Pattern**：如何定義控制流程的骨架，同時允許子類別自訂細節
- **Command Pattern**：如何把控制指令封裝成物件，支援排隊、重試、撤銷

這三個模式在 csp_lib 的 Controller 層有大量應用。理解它們之後，你看 csp_lib 的原始碼會有「原來如此」的豁然開朗感。

[下一篇：控制的骨架：Strategy × Template Method × Command >>>](./00e-patterns-control.md)

---

> 本文為「從零打造工業控制協定框架：給軟體工程師的 CSP Library 實戰系列」的第 00D 篇。
> 完整系列文章請參閱系列目錄。
