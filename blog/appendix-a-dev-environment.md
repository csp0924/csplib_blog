# 附錄 A：開發環境建置指南

> **從零打造工業控制協定框架：給軟體工程師的 CSP Library 實戰系列**
> 附錄 | Appendix A

---

在正式開始使用 CSP Library 之前，你需要一個配置正確的開發環境。這篇附錄會帶你從零開始，完成 Python 環境安裝、專案建置、測試執行，到 IDE 設定。即使你過去沒有接觸過工業控制相關的 Python 專案，跟著這篇做完，就能順利進入後續的實戰章節。

---

## 1. 前置需求

### Python 3.13+

CSP Library 要求 Python 3.13 以上版本。這是因為專案大量使用了 `type` 語法、`Self` 回傳型別、以及 `__future__.annotations` 等現代 Python 特性。

確認你的 Python 版本：

```bash
python --version
# Python 3.13.x
```

如果你的系統預設版本較低，建議使用 [pyenv](https://github.com/pyenv/pyenv)（Linux/macOS）或從 [python.org](https://www.python.org/downloads/) 下載安裝（Windows）。

> **Windows 使用者注意**：建議從 Microsoft Store 安裝 Python，或使用官方安裝程式並勾選「Add Python to PATH」。避免使用 Anaconda 環境，因為它可能與 `uv` 產生衝突。

### uv 套件管理器

CSP Library 使用 [uv](https://docs.astral.sh/uv/) 作為套件管理工具。`uv` 是 Rust 實作的 Python 套件管理器，速度遠快於 `pip`，並且原生支援 `pyproject.toml` 中的 dependency groups。

安裝 `uv`：

```bash
# macOS / Linux
curl -LsSf https://astral.sh/uv/install.sh | sh

# Windows (PowerShell)
powershell -ExecutionPolicy ByPass -c "irm https://astral.sh/uv/install.ps1 | iex"

# 或透過 pip
pip install uv
```

驗證安裝：

```bash
uv --version
# uv 0.6.x
```

### Git

確保你已安裝 Git 2.x 以上版本：

```bash
git --version
# git version 2.x.x
```

---

## 2. 取得並安裝 CSP Library

### 方式一：開發模式（推薦）

如果你要閱讀原始碼、執行測試、或貢獻程式碼，請使用開發模式：

```bash
# 1. Clone 專案
git clone https://github.com/your-org/csp_lib.git
cd csp_lib

# 2. 安裝所有依賴（包含開發工具和所有 extras）
uv sync --all-groups --all-extras
```

`uv sync` 會自動建立虛擬環境（`.venv/` 目錄），並安裝以下內容：

- **核心依賴**：`loguru`（結構化日誌）
- **選用功能**：`pymodbus`、`motor`、`redis`、`python-can`、`psutil`、`etcetra`、`fastapi` 等
- **開發工具**：`pytest`、`pytest-asyncio`、`ruff`、`mypy`、`pre-commit` 等

### 方式二：按需安裝

如果你只是在自己的專案中使用 CSP Library，可以根據需要安裝特定功能：

```bash
# 只安裝 Modbus 通訊功能
pip install csp0924_lib[modbus]

# 安裝 MongoDB + Redis 儲存功能
pip install csp0924_lib[mongo,redis]

# 安裝 GUI 管理介面
pip install csp0924_lib[gui]

# 安裝全部功能
pip install csp0924_lib[all]
```

可用的 extras 包括：

| Extra | 安裝的套件 | 用途 |
|-------|-----------|------|
| `modbus` | pymodbus >= 3.0 | Modbus TCP/RTU 通訊 |
| `mongo` | motor >= 3.0 | MongoDB 非同步存取 |
| `redis` | redis >= 5.0 | Redis 快取與串流 |
| `can` | python-can >= 4.0 | CAN Bus 通訊 |
| `monitor` | psutil >= 5.9 | 系統監控 |
| `cluster` | etcetra >= 0.1 | 高可用叢集（etcd） |
| `gui` | fastapi, uvicorn, pyyaml | Web 管理介面 |
| `all` | 以上全部 | 完整安裝 |

---

## 3. 開發工作流程

### 執行測試

CSP Library 使用 pytest 作為測試框架，非同步測試透過 `pytest-asyncio` 搭配 `@pytest.mark.asyncio` 裝飾器：

```bash
# 執行所有測試
uv run pytest tests/ -v

# 執行特定測試檔案
uv run pytest tests/equipment/test_core_point.py

# 以關鍵字篩選測試
uv run pytest -k "test_scale_transform"

# 執行並產出覆蓋率報告
uv run pytest tests/ --cov=csp_lib --cov-report=html
```

測試目錄結構與原始碼鏡像對應：

```
tests/
├── core/              # Layer 1 測試
├── modbus/            # Layer 2 測試
├── equipment/         # Layer 3 測試
├── controller/        # Layer 4 測試
├── manager/           # Layer 5 測試
├── integration/       # Layer 6 測試
├── mongo/             # Layer 7 測試
├── redis/             # Layer 7 測試
├── gui/               # Layer 8 測試
└── ...
```

### 程式碼品質工具

```bash
# 靜態檢查（Ruff）
uv run ruff check .

# 自動修復可修正的問題
uv run ruff check --fix .

# 格式化程式碼
uv run ruff format .

# 型別檢查
uv run mypy csp_lib/
```

專案的 Ruff 設定（定義在 `pyproject.toml`）：

- **行長度**：120 字元
- **引號風格**：雙引號
- **規則集**：E（pycodestyle errors）、W（warnings）、F（pyflakes）、I（isort）、B（flake8-bugbear）
- **目標版本**：Python 3.13

### Pre-commit Hooks

專案配置了 pre-commit hooks，在每次 `git commit` 時自動執行 `ruff --fix` 和 `ruff format`：

```bash
# 安裝 pre-commit hooks（首次 clone 後執行一次）
uv run pre-commit install

# 手動觸發（對所有檔案執行）
uv run pre-commit run --all-files
```

這意味著你提交的程式碼會自動保持一致的風格，不用擔心忘記格式化。

---

## 4. 專案結構總覽

CSP Library 採用 8 層架構，每一層只能依賴更低的層級：

```
csp_lib/
├── __init__.py         # 版本資訊、頂層匯出
├── core/               # Layer 1：日誌、生命週期、錯誤定義、健康檢查
│   ├── __init__.py     #   get_logger, configure_logging
│   ├── lifecycle.py    #   AsyncLifecycleMixin
│   ├── errors.py       #   DeviceError 例外階層
│   ├── health.py       #   HealthCheckable Protocol
│   └── resilience.py   #   CircuitBreaker, RetryPolicy
├── modbus/             # Layer 2：Modbus 協定層
│   ├── enums.py        #   ByteOrder, RegisterOrder, FunctionCode
│   ├── types.py        #   資料型別定義
│   ├── codec.py        #   編解碼器
│   └── client/         #   TCP/RTU/Shared 非同步客戶端
├── equipment/          # Layer 3：設備抽象層
│   ├── core/           #   Point 定義、Transform 管線
│   ├── device.py       #   AsyncModbusDevice 核心類別
│   ├── alarm/          #   告警定義與評估
│   ├── processing/     #   解碼與聚合處理
│   ├── transport/      #   讀取分組與排程
│   ├── simulation/     #   設備模擬器
│   └── template/       #   設備範本與工廠
├── controller/         # Layer 4：控制策略層
│   ├── protocol.py     #   Strategy Protocol 定義
│   ├── strategies/     #   PQ/QV/FP/Island 等策略
│   ├── executor.py     #   StrategyExecutor
│   └── mode.py         #   ModeManager, ProtectionGuard
├── manager/            # Layer 5：管理層
│   ├── device.py       #   DeviceManager
│   ├── alarm.py        #   AlarmPersistenceManager
│   ├── upload.py       #   DataUploadManager
│   └── unified.py      #   UnifiedDeviceManager
├── integration/        # Layer 6：整合層
│   ├── schema.py       #   Frozen dataclass 配置
│   ├── registry.py     #   DeviceRegistry
│   ├── context.py      #   ContextBuilder
│   ├── router.py       #   CommandRouter
│   └── controller.py   #   SystemController
├── mongo/              # Layer 7：MongoDB 儲存
├── redis/              # Layer 7：Redis 儲存
├── cluster/            # Layer 8：高可用叢集
├── monitor/            # Layer 8：系統監控
├── notification/       # Layer 8：告警通知
├── modbus_server/      # Layer 8：Modbus Server
├── gui/                # Layer 8：FastAPI Web 介面
└── statistics/         # Layer 8：能源統計
```

**關鍵原則**：低層不得引用高層。例如 `csp_lib/modbus/` 絕不會 import `csp_lib/equipment/` 的任何東西。這確保了模組之間的解耦，讓你可以單獨使用底層模組而不需拉入整個框架。

---

## 5. 建置 Cython 生產用 Wheel

CSP Library 支援透過 Cython 編譯為二進位 wheel，用於生產環境部署。這能提供原始碼保護和一定程度的效能提升：

```bash
# 建置最佳化的 wheel
python build_wheel.py

# 清理建置產物
python build_wheel.py clean
```

如果你不需要 Cython 編譯（例如開發階段），可以跳過：

```bash
# 可編輯安裝（不經過 Cython）
SKIP_CYTHON=1 pip install -e .
```

CI/CD 流程中，測試階段使用 `SKIP_CYTHON=1` 以加速，而正式發布時才執行完整的 Cython 編譯。

---

## 6. IDE 設定建議

### VS Code

推薦安裝以下擴充套件：

- **Python** (ms-python.python)：基本 Python 支援
- **Pylance** (ms-python.vscode-pylance)：型別檢查與智慧提示
- **Ruff** (charliermarsh.ruff)：即時 lint 和格式化

建議的 `.vscode/settings.json`：

```json
{
    "python.defaultInterpreterPath": ".venv/bin/python",
    "[python]": {
        "editor.defaultFormatter": "charliermarsh.ruff",
        "editor.formatOnSave": true,
        "editor.codeActionsOnSave": {
            "source.fixAll.ruff": "explicit",
            "source.organizeImports.ruff": "explicit"
        }
    },
    "python.analysis.typeCheckingMode": "standard",
    "python.testing.pytestEnabled": true,
    "python.testing.pytestArgs": ["tests/", "-v"],
    "ruff.lint.args": ["--config=pyproject.toml"],
    "editor.rulers": [120]
}
```

這樣設定後，每次存檔都會自動格式化、排序 import、並顯示 lint 警告。

### PyCharm

1. **設定 Interpreter**：File > Settings > Project > Python Interpreter，選擇 `.venv` 中的 Python
2. **啟用 pytest**：File > Settings > Tools > Python Integrated Tools，將 Default test runner 設為 `pytest`
3. **安裝 Ruff 外掛**：Marketplace 搜尋 "Ruff" 並安裝
4. **設定行長度**：File > Settings > Editor > Code Style > Python，將 Hard wrap at 設為 120
5. **Mark `csp_lib` 為 Sources Root**：右鍵點擊 `csp_lib` 資料夾 > Mark Directory as > Sources Root

### 共通建議

無論使用哪個 IDE，建議開啟以下功能：

- **自動 import 排序**：專案使用 isort 規則（透過 Ruff）
- **顯示行號**和 **120 字元標尺**
- **啟用型別提示**：CSP Library 有完整的型別標注
- **設定 `.venv` 為 Python 解析器**：確保 IDE 能正確解析所有依賴

---

## 7. 執行範例程式

專案的 `examples/` 目錄包含完整的使用範例。典型的執行方式：

```bash
# 基本 Modbus 通訊範例
uv run python examples/basic_modbus.py

# 設備管理範例
uv run python examples/device_management.py

# 控制策略範例
uv run python examples/strategy_demo.py
```

> **注意**：部分範例需要實際的 Modbus 設備或模擬器。如果你手邊沒有真實設備，可以使用 ModRSsim2（Windows）或 diagslave（Linux）作為 Modbus 模擬器。關於模擬器的介紹，請參考[附錄 C：延伸學習資源](appendix-c-resources.md)。

---

## 8. Docker 開發環境

如果你偏好容器化的開發環境，以下是參考的 `Dockerfile` 和 `docker-compose.yml`：

### Dockerfile

```dockerfile
FROM python:3.13-slim

# 安裝系統依賴
RUN apt-get update && apt-get install -y --no-install-recommends \
    build-essential \
    git \
    && rm -rf /var/lib/apt/lists/*

# 安裝 uv
RUN pip install uv

WORKDIR /app

# 複製專案檔案
COPY pyproject.toml uv.lock* ./
RUN uv sync --all-groups --all-extras

COPY . .

# 預設執行測試
CMD ["uv", "run", "pytest", "tests/", "-v"]
```

### docker-compose.yml

當你的開發需要 MongoDB 和 Redis 時，可以使用 Docker Compose 一鍵啟動整個環境：

```yaml
version: "3.8"

services:
  app:
    build: .
    volumes:
      - .:/app
    depends_on:
      - mongo
      - redis
    environment:
      - MONGO_URI=mongodb://mongo:27017/csp_dev
      - REDIS_URL=redis://redis:6379/0

  mongo:
    image: mongo:7
    ports:
      - "27017:27017"
    volumes:
      - mongo_data:/data/db

  redis:
    image: redis:7-alpine
    ports:
      - "6379:6379"
    volumes:
      - redis_data:/data

  modbus-sim:
    image: oitc/modbus-server:latest
    ports:
      - "5020:5020"

volumes:
  mongo_data:
  redis_data:
```

啟動開發環境：

```bash
# 啟動所有服務
docker compose up -d

# 進入應用容器執行測試
docker compose exec app uv run pytest tests/ -v

# 查看日誌
docker compose logs -f app

# 停止所有服務
docker compose down
```

---

## 常見安裝問題排解

| 問題 | 原因 | 解決方式 |
|------|------|----------|
| `uv sync` 失敗，找不到 Python 3.13 | 系統 Python 版本太低 | 安裝 Python 3.13+ 或使用 `uv python install 3.13` |
| `ModuleNotFoundError: pymodbus` | 未安裝 modbus extra | `uv sync --all-extras` 或 `pip install csp0924_lib[modbus]` |
| Cython 編譯失敗 | 缺少 C 編譯器 | Windows: 安裝 Visual Studio Build Tools；Linux: `apt install build-essential` |
| `pre-commit` hook 失敗 | 程式碼風格不符合規範 | 執行 `uv run ruff check --fix . && uv run ruff format .` |
| Windows 上 `SKIP_CYTHON=1` 無效 | 環境變數語法不同 | 使用 `set SKIP_CYTHON=1 && pip install -e .`（CMD）或 `$env:SKIP_CYTHON=1; pip install -e .`（PowerShell） |
| pytest 找不到 async 測試 | 缺少 pytest-asyncio | `uv sync --all-groups` 安裝開發依賴 |

---

## 小結

完成以上步驟後，你應該擁有：

1. Python 3.13+ 和 `uv` 套件管理器
2. 完整的 CSP Library 開發環境（所有依賴已安裝）
3. 能夠執行測試、lint、格式化的工作流程
4. 配置好的 IDE 環境
5. （選用）Docker Compose 開發環境

環境就緒後，你可以回到系列文的第一篇，跟著實際的程式碼範例開始學習。如果在建置過程中遇到問題，建議先查看上方的「常見安裝問題排解」表格，或到專案的 GitHub Issues 頁面搜尋類似問題。

---

**系列導航**

- [附錄 B：常見陷阱與除錯技巧](appendix-b-common-pitfalls.md)
- [附錄 C：延伸學習資源](appendix-c-resources.md)
