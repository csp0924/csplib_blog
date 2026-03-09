# 網路基礎：TCP、序列通訊與封包的概念

> **從零打造工業控制協定框架：給軟體工程師的 CSP Library 實戰系列**
>
> Part 0 — 預備知識篇 | Article 00C
>
> [上一篇：<<< 用 Docker 搭建你的工業開發實驗室](./00b-docker-industrial-lab.md) | [下一篇：給電機/機械工程師的軟體速成 >>>](./00d-software-crash-course-for-engineers.md)

---

## 目錄

1. [為什麼要懂網路底層？](#為什麼要懂網路底層)
2. [OSI 模型速覽：只看 L1 到 L4](#osi-模型速覽只看-l1-到-l4)
3. [TCP 三次握手](#tcp-三次握手)
4. [TCP 持久連線 vs HTTP 短連線](#tcp-持久連線-vs-http-短連線)
5. [TCP 粘包問題與工業協定的解法](#tcp-粘包問題與工業協定的解法)
6. [序列通訊基礎：RS-232 vs RS-485](#序列通訊基礎rs-232-vs-rs-485)
7. [Byte Ordering：Big Endian vs Little Endian](#byte-orderingbig-endian-vs-little-endian)
8. [Wireshark 抓包入門](#wireshark-抓包入門)
9. [Python socket 範例：TCP Echo Server/Client](#python-socket-範例tcp-echo-serverclient)
10. [重點回顧](#重點回顧)
11. [下篇預告](#下篇預告)

---

## 為什麼要懂網路底層？

你可能在想：「我平常用 `requests.get()` 就能呼叫 API，為什麼要去理解 TCP 三次握手這種底層細節？」

答案很簡單——**工業協定沒有 HTTP 這層幫你包裝**。

當你的程式透過 Modbus TCP 和一台 PCS（功率調節系統）溝通時，你操作的就是赤裸裸的 TCP socket。沒有 URL、沒有 HTTP header、沒有 Content-Type。你送出去的是一串精心排列的 bytes，收回來的也是一串 bytes。如果你不了解 TCP 的行為特性，你會遇到各種莫名其妙的問題：

- **為什麼我的程式跑一陣子就收到不完整的回應？**（TCP 粘包）
- **為什麼設備明明在線，但連線建立要等好幾秒？**（三次握手 + 超時重傳）
- **為什麼 RS-485 的兩台設備不能同時說話？**（半雙工通訊）
- **為什麼同一個數值，用不同廠牌的工具讀出來不一樣？**（Byte Order 問題）

這篇文章不會深入到封包的每一個 bit，但會給你足夠的底層知識，讓你在後續接觸 Modbus TCP 和 Modbus RTU 時不會感到茫然。

---

## OSI 模型速覽：只看 L1 到 L4

OSI（Open Systems Interconnection）模型把網路通訊分成七層。對工業協定來說，我們只需要關心前四層：

| 層級 | 名稱 | 工業場景對應 | 你會接觸到的東西 |
|------|------|-------------|----------------|
| **L4** 傳輸層 | Transport | TCP（Modbus TCP 用）| Port 502、Socket、連線管理 |
| **L3** 網路層 | Network | IP | 設備 IP 位址、子網路規劃 |
| **L2** 資料鏈結層 | Data Link | Ethernet / RS-485 幀 | MAC 位址、RS-485 的從站位址 |
| **L1** 實體層 | Physical | RJ-45 網路線 / RS-485 雙絞線 | 接線、佈線、接地 |

上面三層（L5 會話層、L6 表示層、L7 應用層）在 HTTP 的世界裡很重要，但在工業協定中，Modbus 直接坐在 L4（TCP）上面，中間沒有那些層。這也是為什麼 Modbus 如此簡單——它省略了所有「不必要」的抽象。

用一個具體的例子來理解：

```
你的 Python 程式
    ↓ 送出 Modbus 請求（應用層資料）
TCP（L4）：加上 port 號，確保可靠傳輸
    ↓
IP（L3）：加上來源/目的 IP 位址
    ↓
Ethernet（L2）：加上 MAC 位址
    ↓
實體線路（L1）：電訊號在網路線上傳輸
    ↓
到達 PCS 設備，反向拆封
```

**為什麼要知道這些？** 因為當通訊出問題時，你需要知道問題發生在哪一層。如果 `ping` 不通，問題在 L3（IP）或更下面。如果 `ping` 得到但 Modbus 連不上，問題在 L4（TCP port 可能被防火牆擋住）。如果連線建立了但資料不對，問題在應用層（Modbus 請求格式錯誤）。

---

## TCP 三次握手

TCP 在傳輸資料之前，必須先建立連線。這個過程叫做「三次握手」（Three-way Handshake），就像是兩個人打電話前的確認：

```
你的程式 (Client)                     PCS 設備 (Server)
    |                                      |
    |  -------- SYN (seq=100) -------->    |  ① 你好，我想建立連線
    |                                      |
    |  <--- SYN-ACK (seq=300,ack=101) --   |  ② 好的，我也準備好了
    |                                      |
    |  -------- ACK (ack=301) -------->    |  ③ 收到，開始通訊吧
    |                                      |
    |        === 連線已建立 ===              |
    |                                      |
    |  ---- Modbus 請求 (TCP 資料) --->    |  正式傳輸資料
    |  <--- Modbus 回應 (TCP 資料) ----    |
```

三個步驟的意義：

1. **SYN**（Synchronize）：Client 告訴 Server「我要建立連線」，並附上一個初始序號（seq=100）。
2. **SYN-ACK**：Server 回應「我收到了，我也準備好了」，同時回傳自己的序號（seq=300）和對 Client 的確認（ack=101，代表「我已收到你 100 號之前的所有資料」）。
3. **ACK**（Acknowledge）：Client 確認「我收到你的回應了」（ack=301）。

**這對工業通訊的影響是什麼？**

三次握手需要 1.5 個 RTT（Round-Trip Time）。在區域網路中，這大約是 1-3 毫秒，幾乎感覺不到。但如果你的程式每次讀取都重新建立連線，這個開銷就會累積。假設你每秒要讀取 10 個設備，每次都重新握手，光是連線建立就佔用了 10-30 毫秒——這在要求即時回應的控制迴路中是不可接受的。

這就引出了下一個重點：為什麼 Modbus TCP 使用持久連線。

---

## TCP 持久連線 vs HTTP 短連線

在 Web 開發中，你習慣了 HTTP 的「請求-回應」模式。雖然 HTTP/1.1 預設啟用 keep-alive，但概念上每個請求是獨立的，伺服器隨時可能關閉連線。

Modbus TCP 不同。**它假設連線一旦建立就會持續存在**，直到你主動關閉或網路斷開。

```
HTTP 模式（概念上）：
Client --[建立連線]--> Server
Client --[GET /api/data]--> Server
Client <--[200 OK + JSON]-- Server
Client --[關閉連線]--> Server     ← 每次都可能斷開

Modbus TCP 模式：
Client --[建立連線（三次握手）]--> PCS
Client --[讀取暫存器 5000]--> PCS
Client <--[回應：50.5 kW]-- PCS
Client --[讀取暫存器 5004]--> PCS   ← 同一條連線
Client <--[回應：85.0%]-- PCS      ← 持續使用
  ... （持續數小時、數天）
Client --[關閉連線]--> PCS          ← 只在程式結束時
```

為什麼要持久連線？

1. **省去反覆握手的開銷**：工業設備可能每 100ms 就要讀取一次，不可能每次都重新建立連線。
2. **設備資源有限**：很多工業設備的 TCP 連線數有上限（例如只支援 5 個同時連線），頻繁斷開重連可能導致連線耗盡。
3. **連線狀態即健康狀態**：如果 TCP 連線斷開了，你的程式應該立刻察覺並觸發告警，而不是等到下一次請求失敗才發現。

**這意味著你的程式需要處理以下問題：**

- **斷線偵測**：透過 TCP keepalive 或應用層心跳，及時發現連線中斷。
- **自動重連**：斷線後按照策略重連（例如指數退避：1s → 2s → 4s → 8s → ...）。
- **連線池管理**：當你同時控制多台設備時，需要管理多條 TCP 連線的生命週期。

---

## TCP 粘包問題與工業協定的解法

TCP 是一個「位元組串流」（byte stream）協定。這是什麼意思？它代表 TCP **不保留你的訊息邊界**。

假設你的程式連續發送了兩條 Modbus 請求：

```python
sock.send(b'\x00\x01\x00\x00\x00\x06\x01\x03\x00\x00\x00\x0a')  # 請求 1
sock.send(b'\x00\x02\x00\x00\x00\x06\x01\x03\x00\x10\x00\x05')  # 請求 2
```

在接收端，TCP 可能會把這兩條合併成一次 `recv()`，或者把第一條拆成兩半分兩次送到——這就是所謂的「**粘包**」和「**拆包**」。

```
你以為收到的：     [請求1完整][請求2完整]
實際可能收到的：   [請求1的前半][請求1的後半 + 請求2]
或者：            [請求1 + 請求2 的前半][請求2的後半]
```

**Modbus TCP 的解法：MBAP Header**

Modbus TCP 在每個訊息前面加上了一個 7 bytes 的 MBAP（Modbus Application Protocol）Header：

```
MBAP Header（7 bytes）:
┌──────────────┬──────────────┬──────────────┬─────────┐
│ Transaction  │ Protocol ID  │   Length     │ Unit ID │
│   ID (2B)    │    (2B)      │   (2B)      │  (1B)   │
└──────────────┴──────────────┴──────────────┴─────────┘
```

| 欄位 | 長度 | 說明 |
|------|------|------|
| Transaction ID | 2 bytes | 請求/回應配對用的序號 |
| Protocol ID | 2 bytes | 固定 0x0000（代表 Modbus） |
| **Length** | 2 bytes | **後面還有多少 bytes**（含 Unit ID） |
| Unit ID | 1 byte | 從站編號（通常 0x01） |

關鍵在 **Length 欄位**。接收端讀完前 6 bytes 後，就知道接下來還需要讀多少 bytes 才是一個完整的訊息。這就是「**長度前綴**」（length-prefixed）解法：

```python
# 簡化的 Modbus TCP 接收邏輯
def recv_modbus_response(sock):
    # 1. 先讀 MBAP header（6 bytes，不含 Unit ID）
    header = recv_exact(sock, 6)
    transaction_id = int.from_bytes(header[0:2], 'big')
    protocol_id = int.from_bytes(header[2:4], 'big')
    length = int.from_bytes(header[4:6], 'big')

    # 2. 根據 length 讀取剩餘的 payload（含 Unit ID）
    payload = recv_exact(sock, length)

    return header + payload

def recv_exact(sock, n):
    """確保讀取恰好 n 個 bytes"""
    data = b''
    while len(data) < n:
        chunk = sock.recv(n - len(data))
        if not chunk:
            raise ConnectionError("連線中斷")
        data += chunk
    return data
```

注意 `recv_exact()` 函式——它會持續讀取直到拿到指定數量的 bytes。這是處理 TCP 粘包/拆包的標準做法。如果你直接用一次 `sock.recv(1024)` 就假設收到完整訊息，在高流量或網路不穩的環境下一定會出錯。

---

## 序列通訊基礎：RS-232 vs RS-485

不是所有工業設備都用 TCP 網路連線。很多舊設備（或為了節省成本的場景）使用序列通訊（Serial Communication）。Modbus RTU 就是跑在序列介面上的 Modbus 協定。

### RS-232：一對一通訊

RS-232 是你可能在實驗室見過的 DB-9 接頭（九針序列埠）。

```
┌──────────┐    TX ──────────> RX    ┌──────────┐
│  電腦     │    RX <────────── TX    │  設備     │
│ (DTE)    │    GND ─────────── GND  │ (DCE)    │
└──────────┘                          └──────────┘
```

特點：
- **點對點**：一條線只能接一台設備
- **全雙工**：TX 和 RX 分開，可同時收發
- **距離短**：標準規格最長 15 公尺
- **電壓**：±3V 到 ±15V 的差分訊號

### RS-485：多設備匯流排

RS-485 是工業場景的主流，因為它支援多台設備共用同一條通訊線路（匯流排拓撲）。

```
                    RS-485 匯流排（雙絞線 A/B）
 ─────────┬────────────┬────────────┬────────────┬─────
          │            │            │            │
     ┌────┴───┐   ┌────┴───┐  ┌────┴───┐  ┌────┴───┐
     │ Master │   │ Slave  │  │ Slave  │  │ Slave  │
     │ (主站) │   │ ID=1   │  │ ID=2   │  │ ID=3   │
     └────────┘   └────────┘  └────────┘  └────────┘
```

特點：
- **多點**：一條匯流排可接最多 32 台設備（使用中繼器可擴展）
- **半雙工**：同一時間只能有一台設備在「說話」，其他必須「聽」
- **距離長**：最遠可達 1200 公尺
- **差分訊號**：A 線和 B 線的電壓差來表示 0 和 1，抗干擾能力強

**半雙工的實際意義：**

因為 RS-485 是半雙工，通訊模式是嚴格的「一問一答」：

```
時間 →
Master:  [請求 Slave 1] ─── 等待 ─── [請求 Slave 2] ─── 等待 ───
Slave 1: ─── 等待 ─── [回應] ─── 等待 ────────────────────────────
Slave 2: ─── 等待 ──────────────────── 等待 ─── [回應] ─── 等待 ─
```

Master 發出請求後，必須等待目標 Slave 回應（或超時），才能發下一個請求。這就是為什麼 Modbus RTU 的吞吐量比 Modbus TCP 低得多——TCP 上你可以用多條連線同時跟不同設備溝通，但 RS-485 匯流排上同一時間只允許一組問答。

**常見的序列通訊參數：**

```
baudrate=9600, bytesize=8, parity='N', stopbits=1
```

簡寫為 `9600 8N1`，意思是：
- **9600** bps：每秒傳輸 9600 bits
- **8** data bits：每個字元 8 bits
- **N**o parity：無奇偶校驗位
- **1** stop bit：1 個停止位

每傳輸一個 byte 實際要送 10 bits（1 start + 8 data + 1 stop），所以 9600 bps 的有效傳輸速率是 960 bytes/s。一個 Modbus RTU 回應大約 10-50 bytes，所以理論最低延遲約 10-50 毫秒——還不算處理時間和靜默間隔。

---

## Byte Ordering：Big Endian vs Little Endian

當一個數值超過 1 byte 時，就會面臨一個問題：**先傳高位還是先傳低位？**

以十六進位數 `0x1234` 為例（十進位 4660，佔 2 bytes）：

```
Big Endian（大端序）：高位在前
記憶體位址：  [低]  [高]
存放內容：    0x12  0x34
              ↑高位  ↑低位

Little Endian（小端序）：低位在前
記憶體位址：  [低]  [高]
存放內容：    0x34  0x12
              ↑低位  ↑高位
```

用 Python 的 `struct` 模組來驗證：

```python
import struct

value = 0x1234  # 十進位 4660

# Big Endian（大端序）
big = struct.pack('>H', value)    # '>' 代表 Big Endian, 'H' 代表 unsigned short (2 bytes)
print(f"Big Endian:    {big.hex()}")     # 輸出: 1234
print(f"Big Endian:    {list(big)}")     # 輸出: [18, 52]  即 [0x12, 0x34]

# Little Endian（小端序）
little = struct.pack('<H', value)  # '<' 代表 Little Endian
print(f"Little Endian: {little.hex()}")  # 輸出: 3412
print(f"Little Endian: {list(little)}")  # 輸出: [52, 18]  即 [0x34, 0x12]
```

**Modbus 標準使用 Big Endian**（也叫 Network Byte Order）。這很合理，因為網路協定的傳統就是 Big Endian。

但問題來了：**不是所有設備都遵守標準**。

當你遇到 32-bit 的數值（如 Float32），它橫跨 2 個 Modbus 暫存器（每個 16 bits），這時候就有四種可能的排列方式：

```python
import struct

value = 123.456  # 要存入的浮點數

# IEEE 754 Float32 的 bytes：42 F6 E9 79
raw = struct.pack('>f', value)
print(f"原始 bytes: {raw.hex()}")  # 42f6e979

# 四種可能的暫存器排列
# 假設 Register[0] = 前 2 bytes, Register[1] = 後 2 bytes

# 1. Big Endian（高字在前）：Reg[0]=0x42F6, Reg[1]=0xE979 — Modbus 標準
# 2. Little Endian（低字在前）：Reg[0]=0xE979, Reg[1]=0x42F6 — 有些設備用這種
# 3. Big Endian Byte Swap：Reg[0]=0xF642, Reg[1]=0x79E9 — 罕見但存在
# 4. Little Endian Byte Swap：Reg[0]=0x79E9, Reg[1]=0xF642 — 也有人用
```

這四種排列在不同廠牌的設備上都有可能遇到，沒有統一標準。**你只能查設備的通訊協議書，或者用已知值來試**。這也是為什麼工業通訊框架需要支援多種 byte order 配置。

---

## Wireshark 抓包入門

當 Modbus 通訊出問題時，最有效的除錯工具就是 Wireshark——一個免費的封包分析器。

### 安裝與啟動

從 [wireshark.org](https://www.wireshark.org/) 下載安裝後，選擇你的網路介面開始擷取封包。

### 過濾 Modbus TCP 封包

Modbus TCP 預設使用 port 502。在 Wireshark 的 filter 欄輸入：

```
tcp.port == 502
```

或者更精確地只看 Modbus 協定：

```
modbus
```

### 看懂一個 Modbus TCP 封包

假設你抓到一個「讀取保持暫存器」的請求，Wireshark 會解析出：

```
Modbus/TCP
    Transaction Identifier: 1
    Protocol Identifier: 0
    Length: 6
    Unit Identifier: 1
Modbus
    Function Code: Read Holding Registers (3)
    Reference Number: 5000        ← 起始位址
    Word Count: 2                 ← 讀取 2 個暫存器（Float32 需要 2 個）
```

對應的原始 bytes：

```
00 01  00 00  00 06  01  03  13 88  00 02
│      │      │      │   │   │      │
│      │      │      │   │   │      └─ Word Count: 2
│      │      │      │   │   └─ Reference: 5000 (0x1388)
│      │      │      │   └─ Function Code: 3 (讀保持暫存器)
│      │      │      └─ Unit ID: 1
│      │      └─ Length: 6
│      └─ Protocol ID: 0 (Modbus)
└─ Transaction ID: 1
```

### 常用的抓包技巧

1. **用 Transaction ID 配對請求和回應**：每個請求有一個 Transaction ID，回應會帶相同的 ID。
2. **觀察時間戳**：看兩個封包之間的時間差，可以判斷是設備回應慢還是你的程式處理慢。
3. **匯出為 pcap 檔案**：方便保存和分享給同事一起分析。
4. **追蹤 TCP Stream**：右鍵點選一個封包 → Follow → TCP Stream，可以看到整條連線的所有資料交換。

---

## Python socket 範例：TCP Echo Server/Client

在進入 Modbus 之前，先用一個最簡單的 TCP echo server/client 來感受 socket 程式設計。所謂 echo，就是 server 把收到的資料原封不動送回去。

### Echo Server

```python
import socket

def run_echo_server(host: str = "127.0.0.1", port: int = 9999):
    """最簡單的 TCP echo server"""
    # 建立 TCP socket
    server_sock = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
    server_sock.setsockopt(socket.SOL_SOCKET, socket.SO_REUSEADDR, 1)

    # 綁定位址並開始監聽
    server_sock.bind((host, port))
    server_sock.listen(1)
    print(f"Echo server 監聽中: {host}:{port}")

    while True:
        # 等待 client 連線（三次握手在這裡完成）
        client_sock, addr = server_sock.accept()
        print(f"Client 已連線: {addr}")

        try:
            while True:
                # 接收資料（最多 1024 bytes）
                data = client_sock.recv(1024)
                if not data:
                    # Client 關閉連線
                    break
                print(f"收到 {len(data)} bytes: {data}")
                # 原封不動送回去
                client_sock.sendall(data)
        except ConnectionResetError:
            print("Client 斷線")
        finally:
            client_sock.close()
            print(f"Client {addr} 已斷線")

if __name__ == "__main__":
    run_echo_server()
```

### Echo Client

```python
import socket

def run_echo_client(host: str = "127.0.0.1", port: int = 9999):
    """最簡單的 TCP echo client"""
    sock = socket.socket(socket.AF_INET, socket.SOCK_STREAM)

    # 建立連線（觸發三次握手）
    sock.connect((host, port))
    print(f"已連線到 {host}:{port}")

    try:
        messages = [b"Hello", b"Modbus", b"World"]
        for msg in messages:
            # 送出資料
            sock.sendall(msg)
            print(f"送出: {msg}")

            # 接收回應
            response = sock.recv(1024)
            print(f"收到: {response}")
            assert response == msg, "回應不符！"

        print("所有測試通過！")
    finally:
        sock.close()

if __name__ == "__main__":
    run_echo_client()
```

### 執行方式

開兩個終端機：

```bash
# 終端機 1：啟動 server
python echo_server.py

# 終端機 2：啟動 client
python echo_client.py
```

輸出：

```
# Server 端
Echo server 監聽中: 127.0.0.1:9999
Client 已連線: ('127.0.0.1', 54321)
收到 5 bytes: b'Hello'
收到 6 bytes: b'Modbus'
收到 5 bytes: b'World'
Client ('127.0.0.1', 54321) 已斷線

# Client 端
已連線到 127.0.0.1:9999
送出: b'Hello'
收到: b'Hello'
送出: b'Modbus'
收到: b'Modbus'
送出: b'World'
收到: b'World'
所有測試通過！
```

這個範例很簡單，但它展示了 TCP 程式設計的基本流程：`socket()` → `bind()` → `listen()` → `accept()` → `recv()`/`send()` → `close()`。Modbus TCP 的底層通訊就是基於完全一樣的模式，只是送出和接收的不是純文字，而是經過 MBAP header 包裝的二進制 Modbus 幀。

---

## 重點回顧

1. **工業協定直接跑在 TCP/Serial 上**，沒有 HTTP 這層包裝。理解 TCP 和序列通訊的特性是掌握工業協定的前提。

2. **OSI 模型的 L1~L4** 是工業通訊的核心層級。網路除錯時，從低層往高層排查：先確認實體連線（L1），再確認 IP 可達（L3），再確認 TCP port 開放（L4），最後檢查應用層資料。

3. **TCP 三次握手需要時間成本**。這就是為什麼 Modbus TCP 使用持久連線——避免反覆握手的開銷。

4. **TCP 是位元組串流，不保留訊息邊界**。Modbus TCP 透過 MBAP header 的 Length 欄位解決粘包問題，這是工業協定中「長度前綴」的經典應用。

5. **RS-485 是工業場域的序列通訊主流**。它支援多台設備共用匯流排，但因為半雙工特性，同一時間只能有一組問答進行。

6. **Byte Order 在工業通訊中是大坑**。Modbus 標準用 Big Endian，但跨暫存器的 32-bit 值有四種排列方式。務必查閱設備文件或用已知值驗證。

7. **Wireshark 是 Modbus 除錯的最佳工具**。用 `tcp.port == 502` 過濾，觀察 Transaction ID、Function Code 和時間戳來定位問題。

8. **Python socket 程式設計是 Modbus TCP 的基礎**。理解 `sendall()`/`recv()` 的行為，特別是 `recv()` 不保證一次收到完整訊息這個特性。

---

## 下篇預告

如果你是電機或機械工程背景，對 Python 和軟體開發還不太熟悉，下一篇是專門為你準備的。

我們會從「為什麼工程師需要寫程式」開始，快速帶過 Git 版本控制、Python 虛擬環境、型別提示、async/await 等概念。我們會用你熟悉的類比——AutoCAD 存檔比喻版本控制、struct 比喻 dataclass、ISR 比喻 async——讓這些軟體概念變得直覺。最後，你會寫出第一個用 Python 讀取 Modbus register 的程式。

[下一篇：給電機/機械工程師的軟體速成 >>>](./00d-software-crash-course-for-engineers.md)

---

> 本文為「從零打造工業控制協定框架：給軟體工程師的 CSP Library 實戰系列」的第 00C 篇。
> 完整系列文章請參閱系列目錄。
