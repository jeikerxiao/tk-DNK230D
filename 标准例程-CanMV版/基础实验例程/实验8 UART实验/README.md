# 实验8 UART实验

> 按 KEY0/KEY1 分别从串口1、串口2 发送一句话，两个串口互收互打，第一次玩"通信"。难度：初级

## 1. 例程简介

本例程同时初始化开发板引出的 **UART1 和 UART2 两路串口**（均为 115200 波特率、8 数据位、无校验、1 停止位）：按 KEY0 时 UART1 发送字符串 `"From UART1!"`，按 KEY1 时 UART2 发送 `"From UART2!"`；主循环同时读取两路串口收到的数据并打印到 IDE 终端。你将学会 **UART 串口通信的引脚映射、参数配置、收发 API**，这是开发板与传感器模块、其他单片机、PC 通信的必备技能。难度为**初级**。

## 2. 硬件连接与原理

### 用到的引脚（与代码一致）

| 外设 | 引脚 | 代码中的对象 |
| --- | --- | --- |
| UART1 发送 | Pin40 → UART1_TXD | `uart1` |
| UART1 接收 | Pin41 → UART1_RXD | `uart1` |
| UART2 发送 | Pin44 → UART2_TXD | `uart2` |
| UART2 接收 | Pin45 → UART2_RXD | `uart2` |
| KEY0 按键 | GPIO34（上拉输入，按下读 0） | `key0` |
| KEY1 按键 | GPIO35（上拉输入，按下读 0） | `key1` |

### 接线方法（重点）

UART 通信必须"**你的发接我的收**"（TX 对 RX 交叉相连），两设备还要**共地**。本实验在同一块板上做 UART1 ↔ UART2 自发自收，用两根杜邦线这样接：

```text
Pin40 (UART1_TXD) ──── Pin45 (UART2_RXD)
Pin44 (UART2_TXD) ──── Pin41 (UART1_RXD)
（同一块板，GND 天然共地，无需第三根线）
```

接好后：按 KEY0，UART1 发出的数据经 Pin45 进入 UART2，终端打印 `UART2 get data: From UART1!`；按 KEY1 则相反，打印 `UART1 get data: From UART2!`。

### 硬件原理（通俗版）

**UART（通用异步收发器）** 是最经典的串行通信方式：只用两根信号线（TX 发送、RX 接收），把字节数据按位串行传输。"异步"指没有时钟线，双方必须**事先约定相同的速度（波特率）和格式（数据位/校验位/停止位）**，接收方按约定去采样。

打个比方：两人隔着墙按事先约好的语速互相念摩斯电码，只要语速（波特率）一致、暗号格式（8N1）一致，就能听懂；一方语速不对，收到的全是乱码。

### 新手风险提醒

> [!WARNING]
> - 接线时**必须先断开 USB 供电**，插好杜邦线后再上电，避免误把线插到电源引脚造成短路。
> - TX 一定要接对方的 RX，**TX 接 TX 收不到任何数据**（这是新手第一大坑）。
> - 开发板扩展引脚为 **3.3V 电平**，不要直接连接 5V 电平的老式串口设备（如某些 RS232 模块），否则可能损坏芯片；连接 PC 需经 USB 转 TTL 模块并确认其为 3.3V。

## 3. 相关前置知识

- **波特率（baudrate）**：每秒传输的位数。115200 表示约每秒 115200 位。通信双方必须一致，否则乱码。
- **8N1 格式**：8 个数据位（bits=8）、无校验位（No parity）、1 个停止位，是最通用的串口格式，代码中对应 `UART.EIGHTBITS`、`UART.PARITY_NONE`、`UART.STOPBITS_ONE`。
- **TXD / RXD**：发送线 / 接收线的缩写。
- **FPIOA 功能映射**：串口功能同样要先映射，把 Pin40/41/44/45 分别映射为 `UART1_TXD/RXD`、`UART2_TXD/RXD`。
- **bytes 与 decode()**：串口收到的数据是字节串（bytes），`data.decode()` 把它转换成 Python 字符串才能正常显示文字。
- **按键消抖**：延时 20ms 二次确认 + 等待松手（同前面实验）。

## 4. 软件代码逻辑说明

流程：**FPIOA 映射 → 构造按键与两路 UART → 主循环：按键发送 + 持续接收打印**。

### 第 1 步：FPIOA 功能映射

```python
fpioa = FPIOA()
fpioa.set_function(34, FPIOA.GPIO34)
fpioa.set_function(35, FPIOA.GPIO35)
fpioa.set_function(40, FPIOA.UART1_TXD)
fpioa.set_function(41, FPIOA.UART1_RXD)
fpioa.set_function(44, FPIOA.UART2_TXD)
fpioa.set_function(45, FPIOA.UART2_RXD)
```

两个按键映射为 GPIO；4 个引脚分别映射成两路串口的收、发线。

### 第 2 步：构造对象

```python
key0 = Pin(34, Pin.IN, pull=Pin.PULL_UP, drive=7)
key1 = Pin(35, Pin.IN, pull=Pin.PULL_UP, drive=7)
uart1 = UART(UART.UART1, baudrate=115200, bits=UART.EIGHTBITS, parity=UART.PARITY_NONE, stop=UART.STOPBITS_ONE)
uart2 = UART(UART.UART2, baudrate=115200, bits=UART.EIGHTBITS, parity=UART.PARITY_NONE, stop=UART.STOPBITS_ONE)
```

`UART()` 参数逐项：

| 参数 | 值 | 含义 |
| --- | --- | --- |
| 第 1 个 | `UART.UART1` / `UART.UART2` | 串口设备编号 |
| `baudrate` | `115200` | 波特率 |
| `bits` | `UART.EIGHTBITS` | 8 个数据位 |
| `parity` | `UART.PARITY_NONE` | 无校验位 |
| `stop` | `UART.STOPBITS_ONE` | 1 个停止位 |

### 第 3 步：主循环——按键发送

```python
while True:
    if key0.value() == 0:
        time.sleep_ms(20)               # 消抖
        if key0.value() == 0:
            uart1.write("From UART1!")  # UART发送数据
            while key0.value() == 0:
                pass                    # 等待松手
    elif key1.value() == 0:
        time.sleep_ms(20)
        if key1.value() == 0:
            uart2.write("From UART2!")
            while key1.value() == 0:
                pass
```

按 KEY0 → `uart1.write()` 从 UART1 发出字符串；按 KEY1 → UART2 发出。`write()` 可以发送字符串或 bytes。

### 第 4 步：主循环——持续接收

```python
    # UART接收数据
    data = uart1.read(128)
    if data != None:
        print("UART1 get data:", data.decode())

    # UART接收数据
    data = uart2.read(128)
    if data != None:
        print("UART2 get data:", data.decode())
```

`read(128)`：尝试从接收缓冲区**最多读 128 字节**；没收到数据时返回 `None`，收到了则返回 bytes，经 `decode()` 转成字符串打印。注意接收代码在循环里**每圈都执行**（不在 `else` 分支内），所以无论是否按键，串口数据都会被及时取走。

### 关键 API 小结

| API | 作用 |
| --- | --- |
| `UART(id, baudrate, bits, parity, stop)` | 创建并初始化串口 |
| `uart.write(str/bytes)` | 发送数据，返回发送字节数 |
| `uart.read(n)` | 最多读 n 字节；无数据返回 `None` |
| `bytes.decode()` | 字节串转字符串 |

### 可调参数

- `baudrate=115200`：两路串口必须保持一致；接外部设备时要与对方一致（常见 9600、115200）。
- `read(128)` 的 128：单次读取上限，超过此长度的数据会留在缓冲区下一圈再读。
- 发送内容 `"From UART1!"`：可改成任意字符串；发送中文需两端编码一致。

## 5. 运行效果

- **成功现象**（已按第 2 节交叉接线）：
  - 按 KEY0：终端打印 `UART2 get data: From UART1!`（UART1 发的被 UART2 收到）；
  - 按 KEY1：终端打印 `UART1 get data: From UART2!`；
  - 不按键时无任何打印。
- **失败现象排查**：
  1. 按键后无打印 → 最常见原因是 TX/RX 没交叉接，或杜邦线没插牢；对照第 2 节接线图检查。
  2. 打印乱码 → 两端波特率/格式不一致（本实验同一程序内两路参数相同，一般不会）。
  3. `UART1 get data` 和 `UART2 get data` 同时出现同一条 → 你把一根线同时接到了两个 RX 上，改回交叉接法。
  4. 报错找不到 `UART` → 固件不是 CanMV K230D 固件。

## 6. 代码使用说明

1. **断电**后用杜邦线完成 Pin40↔Pin45、Pin44↔Pin41 的交叉连接。
2. 用 **CanMV IDE** 打开本目录下的 `main.py`，USB Type-C 连接开发板与电脑，点击**连接**。
3. 点击**运行**，依次按 KEY0、KEY1，观察终端打印。
4. 固件要求：**CanMV K230D 固件**，版本与 IDE 匹配。
5. **不需要 TF 卡、不需要模型文件**。

> [!NOTE]
> 也可以不接线，改用一台 **USB 转 TTL 模块**连接 UART1（Pin40/Pin41）与电脑，在电脑的串口助手（115200-8N1）里发数据，同样能看到 `UART1 get data: ...` 打印；按 KEY0 时串口助手会收到 `From UART1!`。此时注意模块必须设为 3.3V 电平，且 TTL 模块的 TX 接板子的 RX、RX 接 TX，GND 与板子共地。

## 7. 拓展思考

1. **回环小助手**：把代码改成"UART1 收到什么就原样发回什么"（`uart1.write(data)`），做一个回显器，再用电脑串口助手验证。
2. **命令控制**：约定 UART1 收到字符 `"1"` 点亮红灯、`"0"` 熄灭红灯（结合[实验1 跑马灯实验](../实验1%20跑马灯实验/README.md)的 GPIO 输出），实现"串口遥控灯"。
3. **双板通信**：两块 DNK230D 板子各运行本程序，UART1_TXD/RXD 与对方交叉相连并**共地**，一块板按键、另一块板亮灯，体验真正的设备间通信。

## 8. 常见问题

**Q1：为什么 `uart1.read(128)` 经常返回 `None`？**
A：`read()` 是非阻塞的：缓冲区里有数据就立刻返回，没有就返回 `None`，不会等待。所以代码用 `if data != None` 判断后再处理。主循环转得很快（每圈几毫秒），不会漏数据。

**Q2：按 KEY0 后打印的是"UART2 get data"，是不是错了？**
A：没错，恰恰说明通信成功。KEY0 让 **UART1 发送**，数据经过 Pin40→Pin45 的连线进入 **UART2 的接收端**，所以由 UART2 打印。发送者和接收者是两路不同的串口。

**Q3：`UART.EIGHTBITS`、`UART.PARITY_NONE` 这些常量能不能省略？**
A：8N1 通常就是默认值，可以省略写成 `UART(UART.UART1, 115200)`。但例程把它们写全，是为了提醒你：与外部设备通信前，务必确认双方的波特率和数据格式完全一致。

**Q4：能同时用 UART0 吗？我想接三根串口。**
A：UART0（调试串口）通常被系统日志/调试终端占用，不建议在应用代码中重新映射使用。应用通信请使用本例的 UART1、UART2 这类引出的串口。

**Q5：发送中文会乱码吗？**
A：`write()` 默认按 UTF-8 编码发送中文，接收端也按 UTF-8 `decode()` 就不会乱码。如果对方是 PC 串口助手，需要把助手设为 UTF-8 显示；若对方只认 GBK 等编码，就要手动 `write("中文".encode('gbk'))`。
