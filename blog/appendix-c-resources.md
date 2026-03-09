# 附錄 C：延伸學習資源

> **從零打造工業控制協定框架：給軟體工程師的 CSP Library 實戰系列**
> 附錄 | Appendix C

---

CSP Library 的設計融合了工業通訊協定、電力系統控制、非同步程式設計等多個領域的知識。這篇附錄整理了值得深入學習的資源，幫助你從「會用框架」進階到「理解為什麼這樣設計」。

資源分為七大類：官方規範、書籍、線上學習、工具軟體、社群、CSP Library 專案文件、以及術語表。每個資源都附有簡短說明和建議的學習順序。

---

## 1. 官方規範與標準文件

工業控制通訊的核心是各種國際標準。即使你不需要逐字閱讀，瞭解這些標準的存在和範圍，能幫助你理解 CSP Library 為什麼做出特定的設計決策。

### Modbus 協定

- **Modbus Application Protocol Specification V1.1b3**
  - 來源：[modbus.org](https://modbus.org/specs.php)
  - 這是 Modbus 協定的根本文件。CSP Library 的 `csp_lib/modbus/` 模組直接實作了這份規範中定義的功能碼、資料模型和例外處理。
  - **建議閱讀**：Chapter 4（Data Model）和 Chapter 6（Function Codes）是最實用的部分，大約 20 頁。

- **Modbus Messaging on TCP/IP Implementation Guide V1.0b**
  - 來源：[modbus.org](https://modbus.org/specs.php)
  - 定義了 Modbus TCP 的 MBAP Header 結構。如果你需要用 Wireshark 分析封包，這份文件是必讀的。

- **Modbus over Serial Line Specification V1.02**
  - 來源：[modbus.org](https://modbus.org/specs.php)
  - RTU 和 ASCII 模式的串口通訊規範。CSP Library 的 RTU 客戶端依據此規範實作。

### IEC 61850 系列

- **IEC 61850: Communication Networks and Systems for Power Utility Automation**
  - 國際電工委員會（IEC）發布的變電站通訊標準。
  - 雖然 CSP Library 目前主要支援 Modbus，但架構設計預留了 IEC 61850 整合的空間。理解 IEC 61850 的資料模型（Logical Node、Data Object、Data Attribute）能幫助你理解為什麼設備抽象層設計成現在的樣子。
  - **注意**：IEC 標準需要付費購買。建議先閱讀免費的概述文件或教科書中的摘要。

### IEC 60870-5-104

- **IEC 60870-5-104: Telecontrol Equipment and Systems**
  - 常稱為「IEC 104」，是電力系統遠端遙控的通訊協定。
  - 在台灣的電力系統中，台電與民間儲能系統之間的通訊通常採用此協定。
  - 理解 IEC 104 的「問答式」和「主動上報」兩種通訊模式，能幫助你設計更好的設備事件系統。

### MQTT 與 Sparkplug B

- **MQTT V5.0 Specification**
  - 來源：[docs.oasis-open.org](https://docs.oasis-open.org/mqtt/mqtt/v5.0/mqtt-v5.0.html)
  - 輕量級的發布/訂閱訊息協定。在工業物聯網（IIoT）場景中，MQTT 常用於邊緣設備到雲端的資料傳輸。

- **Sparkplug B Specification**
  - 來源：[sparkplug.eclipse.org](https://sparkplug.eclipse.org/)
  - 建構在 MQTT 之上的工業資料交換標準。定義了統一的主題結構和 payload 格式，解決了原生 MQTT 缺乏資料語義的問題。
  - 如果你的系統需要與 SCADA 平台整合，Sparkplug B 是值得研究的方向。

### OPC UA

- **OPC Unified Architecture Specification**
  - 來源：[opcfoundation.org](https://opcfoundation.org/developer-tools/documents/)
  - 工業通訊的「大一統」標準，提供跨平台、安全的資料交換框架。
  - OPC UA 的資訊模型非常豐富，支援複雜的設備描述和服務導向架構。
  - **免費資源**：OPC Foundation 提供部分規範的免費下載（需註冊）。

---

## 2. 推薦書籍

### 工業通訊與自動化

- **"Industrial Communication Technology Handbook, 2nd Edition"** - Richard Zurawski (CRC Press)
  - 工業通訊的百科全書級參考書。涵蓋 Modbus、PROFINET、EtherCAT、CAN、Wireless HART 等所有主流工業協定。
  - **建議**：不需要從頭讀到尾，當作參考書使用。需要瞭解某個協定時翻閱對應章節。

- **"Modbus: The Evergreen Field Protocol"** - Moishe Garfinkel
  - 專注於 Modbus 協定的實務指南。用大量圖表解釋位元組順序、功能碼、例外處理等概念。
  - 適合需要快速上手 Modbus 的軟體工程師。

### 電力系統與儲能

- **"Smart Grid: Technology and Applications"** - Janaka Ekanayake et al. (Wiley)
  - 智慧電網的入門教科書。解釋了分散式能源、需量反應、儲能系統在電網中的角色。
  - 讀完後你會理解為什麼 CSP Library 需要 PQ 策略、QV 策略這些控制模式。

- **"Electric Energy Storage Systems: Flexibility Options for Smart Grids"** - Matthias Sterner, Ingo Stadler
  - 深入介紹各種儲能技術（鋰電池、液流電池、飛輪、抽蓄水力）及其在電力系統中的應用。
  - 幫助你理解 BMS（電池管理系統）資料的物理含義。

- **"Power System Analysis and Design, 6th Edition"** - J. Duncan Glover et al.
  - 電力系統分析的經典教科書。如果你想理解 kW、kVar、功率因數、頻率調節這些概念背後的物理原理，這本是標準參考。
  - **建議**：Chapter 1-3（基本概念）和 Chapter 12（Power System Controls）最相關。

### Python 與軟體工程

- **"Python Concurrency with asyncio"** - Matthew Fowler (Manning)
  - 深入 Python asyncio 的實務指南。從 event loop 的運作原理，到 task 管理、錯誤處理、效能最佳化。
  - 這本書的內容直接對應 CSP Library 中大量使用的非同步模式。

- **"Architecture Patterns with Python"** - Harry Percival, Bob Gregory (O'Reilly)
  - Python 中的領域驅動設計（DDD）和整潔架構。Repository Pattern、Unit of Work、Event-Driven Architecture。
  - CSP Library 的分層架構和事件驅動設計受到這些模式的啟發。

- **"Python for DevOps"** - Noah Gift et al. (O'Reilly)
  - 涵蓋 Python 在自動化部署、容器化、CI/CD 中的應用。
  - 對於部署 CSP Library 到工業現場環境很有幫助。

---

## 3. 線上課程與教學

### 工業自動化基礎

- **RealPars YouTube 頻道** ([youtube.com/@RealPars](https://www.youtube.com/@RealPars))
  - 免費的工業自動化教學影片。PLC 程式設計、Modbus 通訊、SCADA 系統等主題都有清晰的動畫講解。
  - **推薦播放清單**："What is Modbus?"、"What is SCADA?"。

- **Udemy: Industrial Automation from Scratch**
  - 從零開始的工業自動化入門課程。涵蓋 PLC、HMI、SCADA 的基本概念。

### 電力系統入門

- **MIT OpenCourseWare: Introduction to Electric Power Systems**
  - 免費的電力系統入門課程。講義和錄影都可以線上取得。
  - 不需要電機工程背景也能理解基本概念。

- **台灣電力公司 - 電力知識專區**
  - 中文的電力系統基礎知識，對理解台灣電力市場和併網要求特別有用。

### Python AsyncIO 深入學習

- **Python 官方文件 - asyncio** ([docs.python.org/3/library/asyncio.html](https://docs.python.org/3/library/asyncio.html))
  - 最權威的 asyncio 參考文件。特別注意 "Developing with asyncio" 章節中的除錯技巧。

- **"Async IO in Python: A Complete Walkthrough"** - Real Python
  - 來源：[realpython.com](https://realpython.com/async-io-python/)
  - 非常好的 asyncio 實務教學。從 coroutine 的基本概念到生產環境的最佳實踐。

- **Lynn Root: "Advanced asyncio: Solving Real-world Production Problems"**
  - PyCon 演講錄影。討論 asyncio 在生產環境中的實際問題和解決方案，包括 graceful shutdown、signal handling 等。

---

## 4. 開發與除錯工具

### 通訊分析

| 工具 | 用途 | 取得方式 |
|------|------|----------|
| **Wireshark** | 網路封包分析，內建 Modbus 協定解析器 | [wireshark.org](https://www.wireshark.org/) (免費) |
| **ModRSsim2** | Windows Modbus Slave 模擬器，支援 TCP/RTU | [sourceforge.net/projects/modrssim2](https://sourceforge.net/projects/modrssim2/) (免費) |
| **diagslave** | Linux/Windows Modbus Slave 命令列工具 | [modbusdriver.com](https://www.modbusdriver.com/diagslave.html) (免費) |
| **QModMaster** | 跨平台 Modbus Master 測試工具，有 GUI | [github.com/ed-chemnitz/qmodmaster](https://github.com/ed-chemnitz/qmodmaster) (免費) |
| **Simply Modbus** | Windows Modbus Master/Slave 圖形工具 | [simplymodbus.ca](https://www.simplymodbus.ca/) (付費，有試用) |

**使用建議**：開發初期，先用 ModRSsim2 或 diagslave 建立模擬設備，確認你的程式邏輯正確後，再連接真實設備。這樣可以避免在除錯程式邏輯時同時處理硬體問題。

### 訊息與資料

| 工具 | 用途 | 取得方式 |
|------|------|----------|
| **MQTT Explorer** | MQTT broker 視覺化瀏覽和除錯 | [mqtt-explorer.com](http://mqtt-explorer.com/) (免費) |
| **MongoDB Compass** | MongoDB 圖形化管理工具 | [mongodb.com/products/compass](https://www.mongodb.com/products/compass) (免費) |
| **Redis Insight** | Redis 圖形化管理和監控 | [redis.com/redis-enterprise/redis-insight](https://redis.com/redis-enterprise/redis-insight/) (免費) |
| **mongosh** | MongoDB 互動式命令列 Shell | 隨 MongoDB 安裝 (免費) |

### Python 開發

| 工具 | 用途 | 取得方式 |
|------|------|----------|
| **uv** | 快速的 Python 套件管理器 | [astral.sh/uv](https://docs.astral.sh/uv/) (免費) |
| **Ruff** | 極速 Python Linter + Formatter | [astral.sh/ruff](https://docs.astral.sh/ruff/) (免費) |
| **mypy** | Python 靜態型別檢查 | [mypy-lang.org](https://mypy-lang.org/) (免費) |
| **pytest** | Python 測試框架 | [pytest.org](https://pytest.org/) (免費) |

---

## 5. 社群與論壇

### 工業物聯網

- **Industrial IoT / ICS 相關 Reddit 社群**
  - r/PLC - PLC 程式設計與工業自動化討論
  - r/SCADA - SCADA 系統相關討論
  - 這些社群以英文為主，但有很多來自實際工廠現場的經驗分享。

- **控制工程中文社群**
  - 中國的工控論壇和微信公眾號有大量中文技術討論。搜尋「Modbus 開發」、「儲能 EMS」等關鍵字可以找到很多實務經驗。

### Python 社群

- **Python Discord** ([discord.gg/python](https://discord.gg/python))
  - 活躍的 Python 社群。`#async` 頻道專門討論 asyncio 相關問題。

- **Pythonista Cafe** / **PyCon Taiwan**
  - 台灣的 Python 社群。PyCon Taiwan 每年的演講錄影都可以在 YouTube 找到。

### 儲能產業

- **台灣儲能系統產業推動聯盟（TESSA）**
  - 台灣儲能產業的資訊交流平台。

- **Energy Storage Association (ESA)**
  - 國際儲能產業協會。提供技術白皮書和市場報告。

---

## 6. CSP Library 專案文件

### 專案內文件

| 文件 | 位置 | 說明 |
|------|------|------|
| API Reference | `docs/` 目錄 | 各模組的 API 文件 |
| CHANGELOG | `CHANGELOG.md` | 版本更新記錄 |
| README | `README.md` | 專案總覽和快速開始 |
| Architecture | `CLAUDE.md` | 架構層級和模組依賴說明 |
| Examples | `examples/` 目錄 | 使用範例程式碼 |

### 原始碼導讀建議

如果你想深入理解 CSP Library 的實作，建議按照以下順序閱讀原始碼：

1. **`csp_lib/core/errors.py`** - 例外階層，瞭解錯誤處理的設計
2. **`csp_lib/core/lifecycle.py`** - `AsyncLifecycleMixin`，整個框架的生命週期基礎
3. **`csp_lib/modbus/enums.py`** - Modbus 列舉定義，建立通訊協定的基本概念
4. **`csp_lib/equipment/device.py`** - `AsyncModbusDevice`，核心設備抽象
5. **`csp_lib/controller/protocol.py`** - Strategy Protocol，理解控制策略的介面設計
6. **`csp_lib/integration/schema.py`** - Frozen dataclass 配置模式

每個檔案都有清楚的中文和英文註解，搭配型別標注，應該不難理解。

---

## 7. 術語表

工業控制和電力系統領域有大量的專業術語和縮寫。以下是使用 CSP Library 時最常遇到的術語。

### 設備與系統

| 縮寫 | 全稱 | 說明 |
|------|------|------|
| **PCS** | Power Conversion System | 功率轉換系統。將電池的直流電轉換為交流電（或反向），是儲能系統的核心設備。 |
| **BMS** | Battery Management System | 電池管理系統。監控電池的電壓、電流、溫度、SOC 等，並執行保護功能。 |
| **EMS** | Energy Management System | 能源管理系統。負責整體能源調度策略，是 CSP Library 要實現的上層控制軟體。 |
| **SCADA** | Supervisory Control and Data Acquisition | 監控與資料擷取系統。工業控制的人機介面和資料收集平台。 |
| **DCS** | Distributed Control System | 分散式控制系統。用於大型工廠的過程控制。 |
| **PLC** | Programmable Logic Controller | 可程式邏輯控制器。工業自動化的基本控制單元。 |
| **RTU** | Remote Terminal Unit | 遠端終端設備。在遠端站點收集資料並轉發到中控中心。 |
| **HMI** | Human-Machine Interface | 人機介面。操作人員與控制系統互動的觸控螢幕或軟體介面。 |

### 電力參數

| 縮寫/符號 | 全稱 | 說明 |
|-----------|------|------|
| **SOC** | State of Charge | 電池荷電狀態（0-100%）。類似手機電量百分比。 |
| **SOH** | State of Health | 電池健康狀態。反映電池容量衰減程度。 |
| **P** | Active Power | 有功功率，單位 kW。實際做功的功率。 |
| **Q** | Reactive Power | 無功功率，單位 kVar。不做功但維持電網穩定的功率。 |
| **S** | Apparent Power | 視在功率，單位 kVA。S = sqrt(P^2 + Q^2)。 |
| **PF** | Power Factor | 功率因數。PF = P/S，理想值為 1.0。 |
| **V** | Voltage | 電壓，單位 V（伏特）。 |
| **I** | Current | 電流，單位 A（安培）。 |
| **f** | Frequency | 頻率，單位 Hz。台灣電網標準為 60Hz。 |

### 通訊協定

| 縮寫 | 全稱 | 說明 |
|------|------|------|
| **Modbus RTU** | Modbus Remote Terminal Unit | 基於串口（RS-485/RS-232）的二進位通訊協定。 |
| **Modbus TCP** | Modbus over TCP/IP | 基於乙太網路的 Modbus 通訊。將 Modbus 幀封裝在 TCP 封包中。 |
| **IEC 61850** | - | 變電站通訊標準。使用 MMS（Manufacturing Message Specification）作為傳輸協定。 |
| **IEC 104** | IEC 60870-5-104 | 電力系統遠端遙控協定。基於 TCP/IP。 |
| **DNP3** | Distributed Network Protocol 3 | 北美電力系統常用的通訊協定。 |
| **MQTT** | Message Queuing Telemetry Transport | 輕量級發布/訂閱訊息協定。常用於 IIoT 場景。 |
| **OPC UA** | Open Platform Communications Unified Architecture | 工業通訊的統一架構標準。跨平台、安全、支援複雜資料模型。 |
| **CAN** | Controller Area Network | 控制器區域網路。常用於車用和小型嵌入式系統。CSP Library 提供 `csp_lib[can]` 支援。 |

### 控制模式

| 術語 | 說明 |
|------|------|
| **PQ 控制** | 指定有功功率（P）和無功功率（Q）的控制模式。最基本的儲能控制方式。 |
| **QV 控制** | 根據電壓（V）調節無功功率（Q）的控制模式。用於電壓調節。 |
| **FP 控制** | 根據頻率（F）調節有功功率（P）的控制模式。用於頻率調節（AFC）。 |
| **Island 模式** | 孤島模式。儲能系統獨立供電，不與電網連接。 |
| **Grid-following** | 併網跟隨模式。儲能系統跟隨電網的電壓和頻率運行。 |
| **Grid-forming** | 併網建立模式。儲能系統主動建立電壓和頻率參考。 |
| **AFC** | Automatic Frequency Control，自動頻率控制。台灣電力輔助服務市場的核心產品之一。 |

---

## 學習路線建議

根據你的背景和目標，以下是三條建議的學習路線：

### 路線 A：純軟體工程師，想快速上手

1. 閱讀 [附錄 A](appendix-a-dev-environment.md) 建置環境
2. 跑通 `examples/` 中的基本範例
3. 閱讀 `csp_lib/core/` 原始碼，理解框架基礎
4. 閱讀 Real Python 的 asyncio 教學
5. 使用 ModRSsim2 模擬設備，練習 Modbus 讀寫

### 路線 B：有 IoT 經驗，想深入電力領域

1. 閱讀 Modbus Application Protocol 規範
2. 閱讀《Smart Grid: Technology and Applications》Chapter 1-5
3. 理解 PCS/BMS/EMS 的角色和資料流
4. 研究 CSP Library 的 `controller/strategies/` 原始碼
5. 閱讀 IEC 61850 概述文件

### 路線 C：想成為 CSP Library 貢獻者

1. 完整閱讀 `CLAUDE.md` 中的架構說明
2. 按照第 6 節的原始碼導讀順序閱讀核心模組
3. 執行完整測試套件，確保環境正確
4. 閱讀 `CHANGELOG.md` 瞭解專案演進
5. 從修復 Issues 或補充測試開始貢獻

---

## 小結

工業控制軟體開發是一個跨領域的工作。你不需要成為電力系統專家才能寫出好的控制程式碼，但理解基本概念能幫助你做出更好的設計決策。

這份資源清單不是要你全部讀完，而是提供一張地圖。當你在開發過程中遇到不熟悉的概念時，知道去哪裡找答案，比什麼都重要。

祝你在工業控制軟體開發的旅程中一切順利。

---

**系列導航**

- [附錄 A：開發環境建置指南](appendix-a-dev-environment.md)
- [附錄 B：常見陷阱與除錯技巧](appendix-b-common-pitfalls.md)
