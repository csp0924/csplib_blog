# 你需要知道的 Python 非同步基礎

> **從零打造工業控制協定框架：給軟體工程師的 CSP Library 實戰系列**
>
> Part 0 — 預備知識篇 | Article 00A
>
> 上一篇：（系列首篇） | [下一篇：用 Docker 搭建你的工業開發實驗室 >>>](./00b-docker-industrial-lab.md)

---

## 目錄

1. [為什麼工業通訊需要非同步？](#為什麼工業通訊需要非同步)
2. [Event Loop 心智模型：餐廳裡的服務生](#event-loop-心智模型餐廳裡的服務生)
3. [Coroutine vs Thread vs Process](#coroutine-vs-thread-vs-process)
4. [await 的本質：讓出控制權](#await-的本質讓出控制權)
5. [asyncio.gather 與 create_task 的差異](#asynciogather-與-create_task-的差異)
6. [async with 與 async for](#async-with-與-async-for)
7. [常見陷阱：阻塞事件迴圈](#常見陷阱阻塞事件迴圈)
8. [實作練習：同時讀取 3 台虛擬設備](#實作練習同時讀取-3-台虛擬設備)
9. [重點回顧](#重點回顧)
10. [下篇預告](#下篇預告)

---

## 為什麼工業通訊需要非同步？

想像你是一位駐場工程師，負責監控一座儲能案場。案場裡有 10 台功率調節系統（PCS）、20 組電池管理系統（BMS）、5 個電表，以及若干溫濕度感測器。你的程式需要每秒鐘讀取所有設備的狀態。

如果用傳統的同步方式，程式碼看起來會是這樣：

```python
import time
import socket

def read_device(host: str, port: int) -> bytes:
    """讀取一台設備，約耗時 50ms"""
    sock = socket.create_connection((host, port), timeout=1.0)
    sock.sendall(b"\x00\x01\x00\x00\x00\x06\x01\x03\x00\x00\x00\x0a")
    data = sock.recv(1024)
    sock.close()
    return data

def main():
    devices = [
        ("192.168.1.101", 502),
        ("192.168.1.102", 502),
        ("192.168.1.103", 502),
        # ... 假設共 35 台設備
    ]

    while True:
        start = time.time()
        for host, port in devices:
            data = read_device(host, port)
            process(data)
        elapsed = time.time() - start
        print(f"一輪讀取耗時: {elapsed:.2f}s")
        time.sleep(max(0, 1.0 - elapsed))
```

每台設備的通訊延遲約 50 毫秒。35 台設備依序讀取，一輪就需要 `35 x 50ms = 1750ms`——將近兩秒。你的一秒輪詢目標，在第一天就破滅了。

問題出在哪裡？**大部分時間，你的程式都在等待**。發送請求後，CPU 閒著等網路回應；收到回應後，處理完又去等下一台設備。這就像你在餐廳一桌一桌點餐——走到 A 桌、等客人翻完菜單、寫好訂單，再走到 B 桌重複一次。在等待的過程中，你什麼事都沒做。

非同步程式設計的核心理念就是：**在等待 I/O 的時候，去做別的事**。

用非同步的方式改寫：

```python
import asyncio

async def read_device(host: str, port: int) -> bytes:
    """非同步讀取一台設備"""
    reader, writer = await asyncio.open_connection(host, port)
    writer.write(b"\x00\x01\x00\x00\x00\x06\x01\x03\x00\x00\x00\x0a")
    await writer.drain()
    data = await reader.read(1024)
    writer.close()
    await writer.wait_closed()
    return data

async def main():
    devices = [
        ("192.168.1.101", 502),
        ("192.168.1.102", 502),
        ("192.168.1.103", 502),
        # ... 共 35 台設備
    ]

    while True:
        start = asyncio.get_event_loop().time()
        results = await asyncio.gather(
            *(read_device(host, port) for host, port in devices)
        )
        for data in results:
            process(data)
        elapsed = asyncio.get_event_loop().time() - start
        print(f"一輪讀取耗時: {elapsed:.2f}s")
        await asyncio.sleep(max(0, 1.0 - elapsed))
```

所有設備同時發送請求、同時等待回應。一輪讀取的時間取決於最慢的那台設備（約 50-100ms），而不是所有設備延遲的總和。35 台設備，從 1750ms 降到 100ms 以內。

這就是工業通訊需要非同步的根本原因：**設備數量多、每次通訊都有 I/O 等待、而你的時間預算很緊**。

---

## Event Loop 心智模型：餐廳裡的服務生

理解 `asyncio` 最重要的是建立正確的心智模型。我們用一個餐廳來比喻。

### 同步模型：一桌一服務

```
服務生 → A桌（等客人看菜單...5分鐘...點完了）
       → B桌（等客人看菜單...5分鐘...點完了）
       → C桌（等客人看菜單...5分鐘...點完了）
總耗時：15 分鐘
```

服務生走到 A 桌，客人還在看菜單，服務生就站在旁邊等。等 A 桌點完，才去 B 桌。這就是同步 I/O：程式停在 `recv()` 等資料回來，什麼都不做。

### 非同步模型：一個服務生服務多桌

```
服務生 → A桌（「菜單在這，看好叫我」）
       → B桌（「菜單在這，看好叫我」）
       → C桌（「菜單在這，看好叫我」）
       → A桌舉手了！去收單
       → C桌舉手了！去收單
       → B桌舉手了！去收單
總耗時：約 5 分鐘（取決於最慢的那桌）
```

服務生不會傻等。他把菜單發給每桌，然後巡視——誰準備好了就去服務誰。這就是 Event Loop 的本質：**一個執行緒、輪流處理已經準備好的任務**。

在 Python 的 `asyncio` 中：

- **服務生** = Event Loop（事件迴圈），只有一個
- **客人看菜單** = I/O 等待（網路請求、檔案讀寫）
- **舉手叫服務生** = I/O 完成，事件迴圈收到通知
- **收單、上菜** = 執行 `await` 之後的程式碼

```python
import asyncio

async def serve_table(table: str):
    print(f"[{table}] 發菜單")
    await asyncio.sleep(2)  # 模擬客人看菜單（I/O 等待）
    print(f"[{table}] 客人點完了，收單！")

async def main():
    # 服務生同時照顧三桌
    await asyncio.gather(
        serve_table("A桌"),
        serve_table("B桌"),
        serve_table("C桌"),
    )

asyncio.run(main())
```

輸出：

```
[A桌] 發菜單
[B桌] 發菜單
[C桌] 發菜單
[A桌] 客人點完了，收單！
[B桌] 客人點完了，收單！
[C桌] 客人點完了，收單！
```

三桌幾乎同時點完。如果用同步的 `time.sleep(2)`，需要 6 秒鐘才能服務完三桌。

**關鍵認知**：Event Loop 是單執行緒的。它不是同時做三件事，而是在三件事之間快速切換。就像那個服務生，他一次只能跟一桌說話，但因為他不會傻等，所以效率很高。

---

## Coroutine vs Thread vs Process

在 Python 中，有三種並行處理的方式。它們各有適用場景：

| 特性 | Coroutine（協程） | Thread（執行緒） | Process（行程） |
|------|------------------|-----------------|----------------|
| **建立成本** | 極低（~幾 KB） | 中等（~8 MB stack） | 高（整個 Python 直譯器複製） |
| **切換成本** | 極低（使用者空間） | 中（作業系統核心排程） | 高（行程上下文切換） |
| **數量上限** | 數萬到數十萬 | 數百到數千 | 數十到數百 |
| **GIL 影響** | 無（單執行緒） | 有（CPU 密集任務無法並行） | 無（各自有獨立 GIL） |
| **適合場景** | I/O 密集（網路、檔案） | I/O 密集 + 需與同步庫整合 | CPU 密集（運算、影像處理） |
| **共享狀態** | 安全（單執行緒，無 race condition） | 不安全（需要 Lock） | 不直接共享（需 IPC） |
| **排程方式** | 合作式（程式主動讓出） | 搶佔式（OS 強制切換） | 搶佔式（OS 強制切換） |
| **Python API** | `asyncio` | `threading` | `multiprocessing` |

對工業通訊而言，**Coroutine 幾乎是最佳選擇**：

1. **I/O 密集**：與設備的通訊幾乎都是網路 I/O，CPU 運算佔比極低。
2. **大量併發**：一個案場可能有幾十台設備、上百個點位，需要同時管理大量連線。Coroutine 輕量到可以為每個連線建立一個。
3. **沒有 race condition**：工業控制中，共享狀態（如設備連線、告警列表）非常多。用 Thread 寫容易出 bug，用 Coroutine 天然安全。
4. **可預測的執行順序**：合作式排程意味著你知道程式碼在哪裡會切換，不會在意想不到的地方被中斷。

---

## await 的本質：讓出控制權

`await` 是 Python 非同步程式設計中最核心的關鍵字。但它到底在做什麼？

**`await` 的本質是：暫停當前的 coroutine，把控制權還給 Event Loop，等到某個條件滿足後再繼續。**

讓我們用步驟拆解：

```python
async def fetch_data():
    print("1. 開始讀取")           # 立即執行
    data = await read_network()    # 暫停，讓出控制權
    print("2. 讀取完成")           # 等 read_network 完成後才執行
    return data
```

當執行到 `await read_network()` 時：

1. `fetch_data` 被**掛起**（suspended），它的執行狀態被保存
2. 控制權回到 Event Loop
3. Event Loop 可以去執行其他準備好的 coroutine
4. 當 `read_network()` 的 I/O 完成，Event Loop 收到通知
5. Event Loop **恢復** `fetch_data` 的執行，從第 4 行繼續

重點在於：**`await` 不是阻塞**。阻塞是「我在這裡等，誰也別想用我的執行緒」；`await` 是「我先讓開，你們先忙，等我的事情好了再叫我」。

你可以用一個簡單的實驗來驗證：

```python
import asyncio

async def task_a():
    print("[A] 開始工作")
    await asyncio.sleep(1)
    print("[A] 完成工作")

async def task_b():
    print("[B] 開始工作")
    await asyncio.sleep(1)
    print("[B] 完成工作")

async def main():
    import time
    start = time.time()
    await asyncio.gather(task_a(), task_b())
    print(f"總耗時: {time.time() - start:.2f}s")

asyncio.run(main())
```

輸出：

```
[A] 開始工作
[B] 開始工作
[A] 完成工作
[B] 完成工作
總耗時: 1.00s
```

兩個各需 1 秒的任務，加起來只花 1 秒。因為在 `await asyncio.sleep(1)` 的瞬間，coroutine 讓出了控制權，Event Loop 就去跑另一個 coroutine。

### 沒有 await 的 coroutine 是什麼？

一個重要的規則：**如果你的 async 函式裡面沒有任何 `await`，它就不會讓出控制權**。

```python
async def bad_coroutine():
    # 沒有 await，永遠不會讓出控制權
    total = 0
    for i in range(10_000_000):
        total += i
    return total
```

這個 coroutine 在執行期間會「霸佔」Event Loop，其他所有 coroutine 都得等它算完。這就像那個服務生走到 A 桌後，開始幫客人念完整本菜單——其他桌再怎麼舉手也沒用。

---

## asyncio.gather 與 create_task 的差異

這是初學者最常搞混的兩個 API。它們都能「同時」執行多個 coroutine，但語意和控制力不同。

### asyncio.gather：一次等待多個結果

`gather` 像是跟 Event Loop 說：「把這些任務都跑起來，全部完成後把結果一起給我。」

```python
import asyncio

async def read_sensor(sensor_id: int) -> dict:
    await asyncio.sleep(0.5)  # 模擬 I/O
    return {"sensor_id": sensor_id, "value": sensor_id * 10.5}

async def main():
    # 同時讀取三個感測器，等全部完成
    results = await asyncio.gather(
        read_sensor(1),
        read_sensor(2),
        read_sensor(3),
    )
    print(results)
    # [{'sensor_id': 1, 'value': 10.5},
    #  {'sensor_id': 2, 'value': 21.0},
    #  {'sensor_id': 3, 'value': 31.5}]

asyncio.run(main())
```

`gather` 的特點：
- 傳入多個 coroutine，回傳一個結果列表
- 結果的順序與傳入的順序一致（不是完成的順序）
- 預設情況下，如果其中一個 coroutine 拋出例外，`gather` 會立刻拋出該例外
- 可以用 `return_exceptions=True` 讓例外也作為結果回傳

```python
async def main():
    results = await asyncio.gather(
        read_sensor(1),
        failing_sensor(),    # 這個會拋出例外
        read_sensor(3),
        return_exceptions=True,  # 例外不會中斷其他任務
    )
    for r in results:
        if isinstance(r, Exception):
            print(f"失敗: {r}")
        else:
            print(f"成功: {r}")
```

### asyncio.create_task：啟動一個背景任務

`create_task` 像是跟 Event Loop 說：「幫我把這個任務跑起來，我先去做別的事，晚點再來看結果。」

```python
import asyncio

async def background_monitor():
    """背景任務：每秒列印心跳"""
    while True:
        print("[monitor] 心跳")
        await asyncio.sleep(1)

async def do_work():
    """前景任務：做一些工作"""
    print("[work] 開始工作")
    await asyncio.sleep(3)
    print("[work] 工作完成")

async def main():
    # 啟動背景監控任務
    monitor_task = asyncio.create_task(background_monitor())

    # 做前景工作
    await do_work()

    # 取消背景任務
    monitor_task.cancel()
    try:
        await monitor_task
    except asyncio.CancelledError:
        print("[monitor] 已停止")

asyncio.run(main())
```

輸出：

```
[work] 開始工作
[monitor] 心跳
[monitor] 心跳
[monitor] 心跳
[work] 工作完成
[monitor] 已停止
```

`create_task` 的特點：
- 立即回傳一個 `Task` 物件，不會等待 coroutine 完成
- 你可以在稍後 `await` 這個 Task 來取得結果
- 可以用 `task.cancel()` 來取消任務
- Task 一建立就開始排入 Event Loop 等待執行

### 什麼時候用哪個？

| 場景 | 建議使用 | 原因 |
|------|---------|------|
| 同時讀取多台設備，需要所有結果 | `gather` | 語意清楚：「全部完成後一起給我」 |
| 啟動背景輪詢任務 | `create_task` | 任務需要持續運行，不能等它完成 |
| 動態增減任務 | `create_task` | 可以隨時建立或取消 |
| 錯誤處理需精細控制 | `create_task` | 可以對每個 Task 獨立 try/except |

---

## async with 與 async for

Python 的 `with` 語句是管理資源的利器（檔案、連線、鎖）。`async with` 把同樣的概念帶到了非同步世界。

### async with：非同步的上下文管理器

為什麼設備連線要用 `async with`？因為工業設備的連線建立和斷開通常涉及 I/O 操作：

```python
import asyncio

class DeviceConnection:
    def __init__(self, host: str, port: int):
        self.host = host
        self.port = port
        self.reader = None
        self.writer = None

    async def __aenter__(self):
        """連線建立（非同步 I/O）"""
        self.reader, self.writer = await asyncio.open_connection(
            self.host, self.port
        )
        print(f"已連線到 {self.host}:{self.port}")
        return self

    async def __aexit__(self, exc_type, exc_val, exc_tb):
        """連線關閉（非同步 I/O）"""
        if self.writer:
            self.writer.close()
            await self.writer.wait_closed()
        print(f"已斷開 {self.host}:{self.port}")
        return False  # 不吞掉例外

    async def read(self, n: int) -> bytes:
        return await self.reader.read(n)
```

使用時：

```python
async def main():
    async with DeviceConnection("192.168.1.100", 502) as device:
        data = await device.read(1024)
        # ... 處理資料
    # 離開 async with 區塊後，連線自動關閉
    # 即使中間發生例外，也保證連線會被關閉
```

`async with` 的好處和同步的 `with` 一模一樣——**保證資源一定會被釋放**。在工業場景中，這至關重要：如果你忘了關閉 Modbus 連線，設備可能拒絕新連線（很多設備只允許一個同時連線）。

### async for：非同步的迭代器

`async for` 用於遍歷非同步產生的資料流。在工業場景中，你可能需要持續接收設備的即時資料：

```python
import asyncio

async def device_data_stream(device_id: str):
    """模擬設備即時資料流"""
    import random
    for i in range(5):
        await asyncio.sleep(0.5)  # 模擬等待新資料
        yield {
            "device_id": device_id,
            "timestamp": i,
            "power": round(random.uniform(40, 60), 1),
        }

async def main():
    async for reading in device_data_stream("PCS_01"):
        print(f"收到資料: {reading}")

asyncio.run(main())
```

輸出：

```
收到資料: {'device_id': 'PCS_01', 'timestamp': 0, 'power': 52.3}
收到資料: {'device_id': 'PCS_01', 'timestamp': 1, 'power': 45.7}
收到資料: {'device_id': 'PCS_01', 'timestamp': 2, 'power': 58.1}
...
```

`async for` 背後使用的是 `async def ... yield`（非同步產生器）。每次 `yield` 時，coroutine 讓出控制權，等到消費者需要下一筆資料時才繼續執行。

---

## 常見陷阱：阻塞事件迴圈

非同步程式設計最常見的 bug，就是不小心**阻塞了 Event Loop**。記住：Event Loop 只有一個執行緒。如果你在 coroutine 裡做了任何同步的 I/O 或耗時運算，整個 Event Loop 都會卡住。

### 陷阱一：time.sleep vs asyncio.sleep

```python
import asyncio
import time

async def bad_example():
    print("開始等待")
    time.sleep(3)  # 阻塞！整個 Event Loop 凍結 3 秒
    print("等待結束")

async def good_example():
    print("開始等待")
    await asyncio.sleep(3)  # 非阻塞，讓出控制權
    print("等待結束")
```

`time.sleep(3)` 會讓整個執行緒停下來。Event Loop 也在這個執行緒上，所以它也停了。所有其他 coroutine 都會被凍結 3 秒。

`asyncio.sleep(3)` 只是告訴 Event Loop：「3 秒後叫醒我」，然後立刻讓出控制權。其他 coroutine 可以照常運行。

### 陷阱二：同步 I/O 混入 async

```python
import asyncio

async def bad_file_read():
    # 同步檔案讀取，會阻塞 Event Loop
    with open("big_file.csv") as f:
        data = f.read()  # 如果檔案很大，Event Loop 會卡住
    return data

async def good_file_read():
    # 用 asyncio 的執行緒池來做同步 I/O
    loop = asyncio.get_event_loop()
    data = await loop.run_in_executor(None, read_file_sync)
    return data

def read_file_sync():
    with open("big_file.csv") as f:
        return f.read()
```

`run_in_executor` 是你的救命稻草。它把同步操作丟到一個執行緒池裡執行，不會阻塞 Event Loop。

### 陷阱三：CPU 密集運算

```python
import asyncio

async def bad_computation():
    # CPU 密集運算，霸佔 Event Loop
    total = sum(i * i for i in range(10_000_000))
    return total

async def good_computation():
    # 丟到行程池裡算
    loop = asyncio.get_event_loop()
    from concurrent.futures import ProcessPoolExecutor
    with ProcessPoolExecutor() as pool:
        total = await loop.run_in_executor(
            pool,
            lambda: sum(i * i for i in range(10_000_000))
        )
    return total
```

### 如何偵測阻塞？

Python 3.13 之後，你可以啟用 asyncio 的慢回呼偵測：

```python
import asyncio
import logging

logging.basicConfig(level=logging.DEBUG)

async def main():
    loop = asyncio.get_event_loop()
    loop.slow_callback_duration = 0.1  # 超過 100ms 就警告
    # ... 你的程式碼

asyncio.run(main(), debug=True)
```

啟用 debug 模式後，如果某個 callback 執行超過 `slow_callback_duration`，asyncio 會在日誌中發出警告。這對找出「哪裡阻塞了 Event Loop」非常有用。

---

## 實作練習：同時讀取 3 台虛擬設備

讓我們把學到的概念串起來，寫一個完整的練習。這個程式模擬同時讀取三台設備的場景。

```python
"""
練習：用 asyncio 同時讀取 3 台虛擬設備

執行方式：python async_device_reader.py
"""
import asyncio
import random
import time


class VirtualDevice:
    """虛擬設備，模擬 Modbus 設備的讀取行為"""

    def __init__(self, device_id: str, latency: float):
        self.device_id = device_id
        self.latency = latency  # 模擬通訊延遲（秒）
        self._connected = False

    async def connect(self):
        """模擬建立連線"""
        await asyncio.sleep(0.1)  # 連線需要一點時間
        self._connected = True
        print(f"[{self.device_id}] 連線成功")

    async def disconnect(self):
        """模擬斷開連線"""
        self._connected = False
        print(f"[{self.device_id}] 已斷開")

    async def read_power(self) -> float:
        """模擬讀取功率值"""
        if not self._connected:
            raise ConnectionError(f"{self.device_id} 未連線")
        await asyncio.sleep(self.latency)  # 模擬 I/O 延遲
        return round(random.uniform(10.0, 100.0), 1)

    async def read_temperature(self) -> float:
        """模擬讀取溫度值"""
        if not self._connected:
            raise ConnectionError(f"{self.device_id} 未連線")
        await asyncio.sleep(self.latency)
        return round(random.uniform(20.0, 45.0), 1)

    # 實作 async context manager
    async def __aenter__(self):
        await self.connect()
        return self

    async def __aexit__(self, exc_type, exc_val, exc_tb):
        await self.disconnect()
        return False


async def read_all_data(device: VirtualDevice) -> dict:
    """讀取一台設備的所有資料"""
    power = await device.read_power()
    temperature = await device.read_temperature()
    return {
        "device_id": device.device_id,
        "power_kw": power,
        "temperature_c": temperature,
    }


async def polling_loop(devices: list[VirtualDevice], interval: float = 1.0):
    """輪詢迴圈：每隔 interval 秒讀取所有設備"""
    for cycle in range(3):  # 跑 3 輪
        start = time.time()

        # 同時讀取所有設備
        results = await asyncio.gather(
            *(read_all_data(d) for d in devices),
            return_exceptions=True,
        )

        elapsed = time.time() - start
        print(f"\n--- 第 {cycle + 1} 輪 (耗時 {elapsed:.3f}s) ---")

        for result in results:
            if isinstance(result, Exception):
                print(f"  錯誤: {result}")
            else:
                print(
                    f"  {result['device_id']}: "
                    f"功率={result['power_kw']}kW, "
                    f"溫度={result['temperature_c']}°C"
                )

        # 等待到下一個輪詢週期
        wait_time = max(0, interval - elapsed)
        if wait_time > 0:
            await asyncio.sleep(wait_time)


async def main():
    print("=== 非同步設備讀取練習 ===\n")

    # 建立三台虛擬設備，各有不同延遲
    pcs = VirtualDevice("PCS_01", latency=0.05)
    bms = VirtualDevice("BMS_01", latency=0.08)
    meter = VirtualDevice("METER_01", latency=0.03)

    # 用 async with 管理連線生命週期
    async with pcs, bms, meter:
        # 開始輪詢
        await polling_loop([pcs, bms, meter], interval=1.0)

    print("\n所有設備已斷開，程式結束。")


# ----------- 比較：同步版本 -----------

def sync_main():
    """同步版本，用來對比耗時"""
    print("=== 同步設備讀取（對照組）===\n")
    start = time.time()

    for device_id, latency in [("PCS_01", 0.05), ("BMS_01", 0.08), ("METER_01", 0.03)]:
        time.sleep(0.1)  # 連線
        time.sleep(latency)  # 讀功率
        time.sleep(latency)  # 讀溫度
        print(f"  {device_id}: 讀取完成")

    print(f"\n同步總耗時: {time.time() - start:.3f}s")


if __name__ == "__main__":
    # 先跑同步版本
    sync_main()
    print()

    # 再跑非同步版本
    asyncio.run(main())
```

執行後你會看到類似這樣的輸出：

```
=== 同步設備讀取（對照組）===

  PCS_01: 讀取完成
  BMS_01: 讀取完成
  METER_01: 讀取完成

同步總耗時: 0.620s

=== 非同步設備讀取練習 ===

[PCS_01] 連線成功
[BMS_01] 連線成功
[METER_01] 連線成功

--- 第 1 輪 (耗時 0.082s) ---
  PCS_01: 功率=72.3kW, 溫度=31.2°C
  BMS_01: 功率=45.8kW, 溫度=28.5°C
  METER_01: 功率=88.1kW, 溫度=25.7°C

--- 第 2 輪 (耗時 0.081s) ---
  ...

[PCS_01] 已斷開
[BMS_01] 已斷開
[METER_01] 已斷開

所有設備已斷開，程式結束。
```

重點觀察：

1. **同步版本**需要 0.62 秒（所有延遲的總和）
2. **非同步版本**每輪只需要約 0.08 秒（最慢那台設備的延遲 x 2 次讀取）
3. `async with` 確保了即使出錯，連線也會被正確關閉
4. `asyncio.gather` 配合 `return_exceptions=True`，即使某台設備讀取失敗，其他設備不受影響

---

## 重點回顧

經過這篇文章，讓我們整理幾個核心概念：

1. **工業通訊是典型的 I/O 密集場景**。設備多、每次通訊有網路延遲、時間預算緊，非同步程式設計是天然的解決方案。

2. **Event Loop 是單執行緒的排程器**。它不是同時做多件事，而是在多個任務之間快速切換——像餐廳裡高效率的服務生。

3. **`await` 的本質是讓出控制權**。它不是阻塞，而是告訴 Event Loop：「我在等 I/O，你先去忙別的，好了再叫我。」

4. **Coroutine 比 Thread 更適合工業 I/O**。更輕量、沒有 race condition、建立和切換成本極低。

5. **`asyncio.gather` 等全部完成、`create_task` 立刻回傳**。前者適合「同時讀取多台設備」，後者適合「啟動背景任務」。

6. **`async with` 保證資源一定釋放**。對工業設備連線來說，忘記關連線可能導致設備拒絕後續連線。

7. **最大的陷阱是阻塞 Event Loop**。`time.sleep`、同步檔案 I/O、CPU 密集運算——這些都會讓所有 coroutine 停擺。用 `asyncio.sleep`、`run_in_executor` 來解決。

如果你只記住一件事，那就是：**`await` 就是讓出控制權。理解了這一點，你就理解了 Python 非同步程式設計的核心。**

---

## 下篇預告

現在你已經掌握了 Python 非同步程式設計的基礎。但要開始實際的工業通訊開發，你還需要一個模擬真實設備的開發環境——畢竟你不太可能把一台價值百萬的 PCS 搬到辦公室來寫程式。

在下一篇文章中，我們將用 Docker 搭建一個完整的工業開發實驗室，包含：

- Modbus TCP 模擬器（模擬真實設備）
- Redis（即時資料快取）
- MongoDB（時序資料儲存）
- etcd（叢集協調）

只要一個 `docker compose up`，你就擁有了一個可以離線開發、隨時重置的工業模擬環境。

[下一篇：用 Docker 搭建你的工業開發實驗室 >>>](./00b-docker-industrial-lab.md)

---

> 本文為「從零打造工業控制協定框架：給軟體工程師的 CSP Library 實戰系列」的 Part 0 預備知識篇 Article 00A。
> 完整系列文章請參閱系列目錄。
