# 部署策略：從開發環境到工業現場

> **從零打造工業控制協定框架：給軟體工程師的 CSP Library 實戰系列**
>
> Part 6 — 系統整合篇 | Article 26
>
> [<<< 上一篇：系統架構總覽](./25-architecture-overview.md) | [下一篇：監控與告警 >>>](./27-monitoring.md)

---

## 目錄

1. [開發環境建置](#開發環境建置)
2. [測試管線](#測試管線)
3. [生產構建：Cython 編譯](#生產構建cython-編譯)
4. [CI/CD 管線：GitHub Actions](#cicd-管線github-actions)
5. [部署拓撲](#部署拓撲)
6. [Docker 部署考量](#docker-部署考量)
7. [配置管理](#配置管理)
8. [重點回顧](#重點回顧)
9. [下篇預告](#下篇預告)

---

## 開發環境建置

### 工具鏈選擇

csp_lib 使用 Python 3.13+ 作為基礎運行環境，搭配 [uv](https://docs.astral.sh/uv/) 作為套件管理器。如果你還在用 `pip` + `venv`，強烈建議你試試 uv——它的安裝速度是 pip 的 10-100 倍，而且內建了虛擬環境管理。

```bash
# 安裝 uv（如果還沒裝的話）
curl -LsSf https://astral.sh/uv/install.sh | sh  # macOS/Linux
# 或 Windows:
powershell -ExecutionPolicy ByPass -c "irm https://astral.sh/uv/install.ps1 | iex"
```

### 第一次設定

```bash
# 克隆專案
git clone https://github.com/your-org/csp_lib.git
cd csp_lib

# 安裝所有依賴（包括開發工具 + 所有可選依賴）
uv sync --all-groups --all-extras
```

`--all-groups` 安裝開發依賴組（pytest、ruff、mypy 等），`--all-extras` 安裝所有可選功能（modbus、mongo、redis、gui 等）。

### 開發模式安裝

在開發階段，你希望修改程式碼後立即生效，不需要每次都重新安裝。使用 editable install：

```bash
# 跳過 Cython 編譯（開發不需要）
SKIP_CYTHON=1 pip install -e .
```

`SKIP_CYTHON=1` 這個環境變數很關鍵。csp_lib 的 `setup.py` 在正常安裝時會嘗試使用 Cython 將 `.py` 檔案編譯成 C 擴展模組（`.so` 或 `.pyd`），這在開發階段會大幅拖慢安裝速度。設定這個環境變數後，安裝過程會跳過 Cython 編譯，直接使用純 Python 原始碼。

### 編輯器設定

csp_lib 使用 Ruff 作為 linter 和 formatter。建議在你的 VS Code 中安裝 Ruff 擴充套件，並在 `.vscode/settings.json` 中加入：

```json
{
    "editor.formatOnSave": true,
    "[python]": {
        "editor.defaultFormatter": "charliermarsh.ruff",
        "editor.codeActionsOnSave": {
            "source.fixAll.ruff": "explicit",
            "source.organizeImports.ruff": "explicit"
        }
    }
}
```

專案的程式碼風格規範定義在 `pyproject.toml` 中：

| 規則 | 設定值 |
|------|--------|
| 行寬 | 120 字元 |
| 引號 | 雙引號 |
| Lint 規則 | E, W, F, I (isort), B (flake8-bugbear) |
| 目標版本 | Python 3.13 |

---

## 測試管線

csp_lib 的測試管線包含四個步驟，建議按順序執行：

### Step 1：程式碼格式檢查

```bash
# 檢查格式（不修改檔案）
uv run ruff format --check .

# 自動修正格式
uv run ruff format .
```

### Step 2：靜態分析

```bash
# Lint 檢查
uv run ruff check .

# Lint + 自動修正可安全修正的問題
uv run ruff check --fix .
```

### Step 3：型別檢查

```bash
uv run mypy csp_lib/
```

csp_lib 附帶 `py.typed` 標記檔案，表示它是一個 PEP 561 相容的型別套件。這意味著使用 csp_lib 的下游專案也能獲得完整的型別提示。

### Step 4：單元測試

```bash
# 執行所有測試
uv run pytest tests/ -v

# 執行特定測試檔案
uv run pytest tests/equipment/test_core_point.py

# 依名稱模式過濾
uv run pytest -k "test_scale_transform"

# 附帶覆蓋率報告
uv run pytest tests/ -v --cov=csp_lib --cov-report=xml
```

非同步測試使用 `@pytest.mark.asyncio` 裝飾器：

```python
import pytest

@pytest.mark.asyncio
async def test_device_read():
    async with mock_device() as device:
        values = device.latest_values
        assert "soc" in values
```

### Pre-commit Hooks

專案配置了 pre-commit hooks，會在每次 `git commit` 前自動執行 `ruff --fix` 和 `ruff format`。首次設定：

```bash
uv run pre-commit install
```

之後每次 commit 都會自動觸發格式化和 lint 修正，確保提交到倉庫的程式碼永遠符合規範。

---

## 生產構建：Cython 編譯

在開發環境中，我們跳過了 Cython 編譯。但在生產部署時，Cython 編譯能帶來兩個重要的好處：

1. **效能提升**：Python 程式碼編譯成 C 擴展模組後，執行速度可提升 2-10 倍（取決於程式碼特性）
2. **原始碼保護**：編譯後的 `.so`/`.pyd` 檔案是二進制格式，無法直接閱讀原始碼

### 構建流程

```bash
# 建置 Cython wheel
python build_wheel.py

# 清理建置產物
python build_wheel.py clean
```

`build_wheel.py` 會自動完成以下步驟：
1. 掃描 `csp_lib/` 下所有 `.py` 檔案
2. 使用 Cython 將 `.py` 轉譯為 `.c` 檔案
3. 使用系統的 C 編譯器將 `.c` 編譯為 `.so`（Linux）或 `.pyd`（Windows）
4. 使用 mypy 的 stubgen 產生 `.pyi` 型別存根檔案
5. 打包為 wheel 檔案

構建完成後，你會在 `dist/` 目錄下看到類似這樣的檔案：

```
dist/
  csp0924_lib-0.4.0-cp313-cp313-win_amd64.whl     # Windows
  csp0924_lib-0.4.0-cp313-cp313-linux_x86_64.whl   # Linux
```

注意檔名中的 `cp313`——這表示這個 wheel 是針對 Python 3.13 編譯的，無法在其他 Python 版本上使用。這是 C 擴展模組的限制。

---

## CI/CD 管線：GitHub Actions

csp_lib 的 CI/CD 管線定義在 `.github/workflows/build-wheels.yml`，根據觸發條件執行不同的任務：

### PR 觸發：驗證品質

當你對 `main` 分支發起 Pull Request 時，管線會執行：

```
[Lint] → [Test (Ubuntu)] + [Test (Windows)]
```

兩個平台的測試並行執行，都使用 `SKIP_CYTHON=1` 來避免不必要的編譯。測試通過是合併 PR 的必要條件。

### Tag 觸發：發布流程

當你推送 `v*` 格式的 tag 時（例如 `v0.4.0`），管線執行完整的發布流程：

```
[Lint] → [Test (Ubuntu + Windows)]
       → [Build Wheel (Windows)]
       → [Build Wheel (manylinux)]
       → [Publish to PyPI]
       → [Upload to GitHub Release]
```

Linux 的 wheel 需要額外的 `auditwheel repair` 步驟，將 `linux_x86_64` 標記轉換為 `manylinux_2_28_x86_64`，確保在各種 Linux 發行版上都能安裝。

### 發布步驟

```bash
# 1. 更新版本號（在 csp_lib/__init__.py 中）
# 2. 更新 CHANGELOG.md
# 3. 提交並推送
git add .
git commit -m "Release v0.4.0"
git push

# 4. 打 tag 觸發發布
git tag v0.4.0
git push origin v0.4.0
```

Tag 推送後，GitHub Actions 會自動完成構建、測試、發布的全部流程。

---

## 部署拓撲

根據案場的規模和可靠性需求，csp_lib 支援三種部署拓撲。

### 拓撲一：單節點部署

```
┌──────────────────────────────────┐
│         Industrial PC            │
│                                  │
│  ┌────────────────────────────┐  │
│  │     SystemController       │  │
│  │  ┌──────┐ ┌────────────┐  │  │
│  │  │Modes │ │ Protection │  │  │
│  │  └──────┘ └────────────┘  │  │
│  │  ┌──────────────────────┐ │  │
│  │  │   DeviceRegistry     │ │  │
│  │  └──────────────────────┘ │  │
│  └────────────────────────────┘  │
│                                  │
│  ┌──────────┐  ┌──────────────┐  │
│  │ MongoDB  │  │  Redis       │  │
│  └──────────┘  └──────────────┘  │
│                                  │
│  ┌──────────┐  ┌──────────────┐  │
│  │ Monitor  │  │  GUI (API)   │  │
│  └──────────┘  └──────────────┘  │
└────────────┬─────────────────────┘
             │ Modbus TCP / RS-485
    ┌────────┼────────┐
    │        │        │
  [PCS]    [BMS]   [Meter]
```

**適用場景**：小型案場（單一儲能系統）、概念驗證、開發測試

**優點**：架構簡單、部署容易、維護成本低
**缺點**：單點故障、主機故障時系統完全停擺

### 拓撲二：HA 雙機熱備

```
┌────────────────────┐    ┌────────────────────┐
│   Node A (Active)  │    │  Node B (Standby)  │
│                    │    │                    │
│  SystemController  │    │  SystemController  │
│  ClusterController ├────┤  ClusterController │
│  LeaderElector     │    │  LeaderElector     │
│                    │    │                    │
└────────┬───────────┘    └────────┬───────────┘
         │                         │
         │    ┌──────────┐         │
         └────┤  etcd    ├─────────┘
              └──────────┘
              ┌──────────┐
              │  Redis   │  (狀態同步)
              └──────────┘
         │
    ┌────┼────────┐
    │    │        │
  [PCS] [BMS]  [Meter]
```

**適用場景**：中型案場、SLA 要求較高的項目

csp_lib 的 `cluster` 模組提供了基於 etcd 的 leader election。兩個節點同時運行，但只有 leader 節點會執行控制迴路。當 leader 故障時，standby 節點在數秒內接手：

```python
from csp_lib.cluster import ClusterConfig, EtcdConfig, ClusterController

cluster_config = ClusterConfig(
    node_id="node-a",
    etcd=EtcdConfig(endpoints=["http://etcd:2379"]),
    lease_ttl=10,       # etcd lease 存活時間（秒）
    heartbeat_interval=3,  # 心跳間隔
)
```

### 拓撲三：分散式多站點

```
┌──────────────┐     ┌──────────────┐     ┌──────────────┐
│   Site A     │     │   Site B     │     │  Central     │
│              │     │              │     │              │
│  Controller  │────►│  Controller  │────►│ Aggregator   │
│  Local PCS   │     │  Local PCS   │     │ Dashboard    │
└──────────────┘     └──────────────┘     └──────────────┘
```

**適用場景**：多案場管理、區域電力調度

csp_lib 的 `integration.distributed` 模組提供了 `DistributedController`，支援跨站點的命令路由和狀態同步。每個站點運行獨立的 `SystemController`，中央節點透過 `RemoteCommandRouter` 協調全局策略。

---

## Docker 部署考量

在工業場域使用 Docker 部署時，有幾個關鍵的差異需要注意。

### 網路存取

Modbus TCP 設備通常在特定的子網路中。容器需要能夠存取這些設備：

```yaml
# docker-compose.yml
services:
  ems:
    image: your-org/ems:latest
    network_mode: host  # 最簡單的方式：使用主機網路
    # 或者使用 macvlan 網路：
    # networks:
    #   modbus_net:
    #     ipv4_address: 192.168.1.100
```

`network_mode: host` 是最簡單的做法，讓容器直接使用主機的網路堆疊。如果你需要更精細的網路隔離，可以使用 Docker 的 macvlan 驅動程式，將容器直接暴露到 Modbus 設備所在的 VLAN 中。

### RS-485 串列埠

如果使用 Modbus RTU（RS-485），需要將主機的串列埠裝置映射到容器中：

```yaml
services:
  ems:
    image: your-org/ems:latest
    devices:
      - /dev/ttyUSB0:/dev/ttyUSB0  # RS-485 轉接器
    privileged: true  # 某些系統需要特權模式才能存取串列埠
```

### 持久化資料

MongoDB 和 Redis 的資料需要持久化到主機磁碟：

```yaml
services:
  mongodb:
    image: mongo:7
    volumes:
      - /opt/ems/data/mongo:/data/db

  redis:
    image: redis:7-alpine
    volumes:
      - /opt/ems/data/redis:/data
```

### 時區設定

工業控制系統對時間非常敏感。確保容器使用正確的時區：

```yaml
services:
  ems:
    image: your-org/ems:latest
    environment:
      - TZ=Asia/Taipei
    volumes:
      - /etc/localtime:/etc/localtime:ro
```

### 自動重啟

工業控制系統需要 24/7 運行。設定自動重啟策略：

```yaml
services:
  ems:
    restart: always
    # 或者更精細的控制：
    # restart: on-failure
    # deploy:
    #   restart_policy:
    #     condition: on-failure
    #     max_attempts: 5
    #     delay: 10s
```

---

## 配置管理

### Frozen Dataclass 配置模式

csp_lib 的配置物件都是不可變的 frozen dataclass。這在部署中意味著：**配置在建立後就不會改變**。如果需要修改配置，你必須重新建立配置物件並重啟相關的服務。

```python
from csp_lib.integration import SystemControllerConfig, ContextMapping

# 從字典建立配置（通常來自 JSON/YAML 檔案）
config_dict = {
    "context_mappings": [
        {"point_name": "soc", "context_field": "soc", "trait": "bms"},
    ],
    "command_mappings": [
        {"command_field": "p_target", "point_name": "p_set", "trait": "pcs"},
    ],
}

# 手動轉換（因為巢狀的 dataclass 需要逐層建構）
config = SystemControllerConfig(
    context_mappings=[
        ContextMapping(**m) for m in config_dict["context_mappings"]
    ],
    command_mappings=[
        CommandMapping(**m) for m in config_dict["command_mappings"]
    ],
)
```

### 環境特定配置

建議使用分層配置的方式管理不同環境：

```
config/
  base.yaml       # 基礎配置（設備點位定義、映射關係）
  dev.yaml         # 開發環境覆蓋（模擬設備位址）
  staging.yaml     # 測試環境覆蓋
  production.yaml  # 生產環境覆蓋（真實設備位址）
```

```python
import yaml

def load_config(env: str) -> dict:
    """載入分層配置"""
    with open("config/base.yaml") as f:
        config = yaml.safe_load(f)

    env_file = f"config/{env}.yaml"
    with open(env_file) as f:
        overrides = yaml.safe_load(f)

    # 合併覆蓋
    config.update(overrides)
    return config
```

### Modbus 連線配置

不同環境的設備位址通常不同。使用環境變數或配置檔案來管理：

```python
from csp_lib.modbus import ModbusTcpConfig

# 開發環境：連到模擬器
dev_config = ModbusTcpConfig(host="127.0.0.1", port=5020, unit_id=1)

# 生產環境：連到真實設備
prod_config = ModbusTcpConfig(host="192.168.1.10", port=502, unit_id=1)
```

### 機密管理

工業環境中可能需要管理 MongoDB 密碼、Redis 密碼、etcd 認證等機密資訊。建議的做法：

1. **開發環境**：使用 `.env` 檔案（加入 `.gitignore`）
2. **生產環境**：使用 Docker secrets 或 Kubernetes secrets
3. **永遠不要**把密碼寫在程式碼或配置檔案中

```yaml
# docker-compose.yml
services:
  ems:
    environment:
      - MONGO_URI=mongodb://user:${MONGO_PASSWORD}@mongo:27017/ems
    secrets:
      - mongo_password

secrets:
  mongo_password:
    file: ./secrets/mongo_password.txt
```

---

## 重點回顧

1. **開發環境**使用 `uv sync --all-groups --all-extras` 一鍵安裝所有依賴，`SKIP_CYTHON=1` 跳過不必要的 Cython 編譯。

2. **測試管線**四步驟：格式檢查 → 靜態分析 → 型別檢查 → 單元測試。Pre-commit hooks 自動維持程式碼品質。

3. **生產構建**使用 `python build_wheel.py` 編譯 Cython wheel，獲得效能提升和原始碼保護。

4. **CI/CD** 區分 PR（只測試）和 Tag（測試 + 構建 + 發布）兩種流程，支援 Windows 和 manylinux 雙平台。

5. **部署拓撲**從單節點到 HA 雙機到分散式多站點，按需選擇。

6. **Docker 部署**需要特別注意網路存取（host networking 或 macvlan）、串列埠映射、時區設定。

7. **配置管理**使用 frozen dataclass + 分層 YAML 配置，環境特定資訊透過覆蓋檔案管理。

---

## 下篇預告

部署上線之後，接下來最重要的事情是：**你怎麼知道系統運作正常？** 下一篇我們將深入 csp_lib 的監控與告警架構——從系統資源監控、階層式健康檢查、告警狀態管理到多通道通知，建立工業級的可觀測性。

> [下一篇：監控與告警：建立工業級的可觀測性 >>>](./27-monitoring.md)
