# 实验16 视频播放器实验

> 在 LCD 上播放 MP4 视频，KEY0 暂停、KEY1 继续，播完自动退出，学会用 Player 播放器接口。难度：中级

## 1. 例程简介

本例程使用 CanMV 固件的 **Player 播放器接口**，从 `/data/VIDEO/test.mp4` 加载 MP4 视频，在 2.4 寸 MIPI LCD 上自动播放（同时回传 IDE 预览）：**按 KEY0 暂停、按 KEY1 继续**；视频播放到结尾时，播放器抛出 **EOF（播放结束）事件**，回调函数把状态标志清零，程序自动停止播放并退出。你将学会**视频解码播放、事件回调机制、按键控制播放状态**。难度为**中级**，是多媒体应用的综合实验。

> [!NOTE]
> 代码头部注释明确说明：**当前版本仅支持 MP4** 格式。

## 2. 硬件连接与原理

| 外设 | 引脚/资源 | 代码中的体现 |
| --- | --- | --- |
| 2.4 寸 MIPI LCD（ST7701） | MIPI DSI | `Player(Display.ST7701, display_to_ide=True)` |
| KEY0 按键（暂停） | GPIO34（上拉输入，按下读 0） | `key0` |
| KEY1 按键（继续） | GPIO35（上拉输入，按下读 0） | `key1` |
| 存储分区 | 视频文件 `/data/VIDEO/test.mp4` | `play_mp4_test("/data/VIDEO/test.mp4")` |
| USB 串口 | 打印播放状态 | `print("play pause")` 等 |

屏幕出厂已连接，无需接线；但**需要事先把视频文件放到指定路径**（见第 6 节）。

### 硬件原理（通俗版）

**视频 = 一连串压缩的图片（帧）+ 音轨**。MP4 是一种"容器"格式，里面的画面通常用 H.264/H.265 等算法压缩过——相邻帧大部分内容相似，只存"变化部分"，体积可缩小几十倍。播放时**解码器**要把每帧还原成完整图像，这是很重的计算。

K230D 芯片内置**硬件视频解码单元（VPU）**，解码工作由它完成，CPU 只需要"喂文件、管流程"。CanMV 的 `Player` 类封装了整条链路：**读文件 → 硬件解码 → 送 LCD 显示**，所以在 MicroPython 里播放视频也能流畅。

**事件回调**：播放器工作在自己的流程里，"播完了"这种事件没法靠主循环询问，而是你登记一个回调函数（`set_event_callback`），事件发生时播放器自动调用它——本例回调收到 `K_PLAYER_EVENT_EOF`（End Of File）后把 `start_play` 置 False，主循环随即退出、停止播放。这与[实验5 Timer实验](../实验5%20Timer实验/README.md)的定时器回调是同一种思想。

### 新手风险提醒

> [!WARNING]
> - `/data/VIDEO/test.mp4` **必须提前放好**，文件不存在时 `player.load()` 会抛异常。
> - `/data` 分区容量有限，视频文件较大时建议改存 TF 卡（`/sdcard/`）并同步修改代码路径。
> - 只支持 MP4 封装；直接把 `.avi`、`.mkv` 改名成 `.mp4` 是**没用的**，编码格式不对照样播不了。

## 3. 相关前置知识

- **MP4 容器格式**：常见的视频封装格式，内部是压缩编码的视频帧（当前固件版本仅支持 MP4）。
- **硬件解码（VPU）**：芯片里专门做视频解压的硬件模块，不占 CPU，播放流畅的关键。
- **帧与帧率**：视频每秒播放的张数（如 25/30fps），Player 内部按文件信息自动处理。
- **事件回调（event callback）**：登记一个函数，事件发生时被自动调用；本例处理"播放结束"事件。
- **`K_PLAYER_EVENT_EOF`**：固件定义的常量，表示视频已播放到文件末尾。
- **`global` 关键字**：回调函数内修改函数外定义的全局变量 `start_play` 前必须声明（同实验 5 的 `count`）。
- **暂停/继续（pause/resume）**：暂停时解码停止、画面定格在当前帧；继续时从暂停位置接着播，而不是从头播。

## 4. 软件代码逻辑说明

流程：**FPIOA 映射按键 → 定义事件回调 → 创建 Player 并加载文件 → 启动播放 → 循环响应按键/等待结束 → 停止播放退出**。

### 第 1 步：按键初始化

```python
fpioa = FPIOA()
fpioa.set_function(34, FPIOA.GPIO34)
fpioa.set_function(35, FPIOA.GPIO35)

key0 = Pin(34, Pin.IN, pull=Pin.PULL_UP, drive=7)
key1 = Pin(35, Pin.IN, pull=Pin.PULL_UP, drive=7)
```

KEY0/KEY1 上拉输入，按下读 0，与前几个实验一致。

### 第 2 步：全局状态与事件回调

```python
start_play = False
def player_event(event,data):
    global start_play
    if(event == K_PLAYER_EVENT_EOF):
        start_play = False
```

- `start_play` 是"正在播放"标志。
- `player_event(event, data)` 是登记给播放器的回调：播放器每有事件就调用它；当收到 `K_PLAYER_EVENT_EOF`（播完）时，把 `start_play` 置 False，主循环条件不再成立，播放流程收尾退出。

### 第 3 步：创建播放器并加载视频

```python
def play_mp4_test(filename):
    global start_play

    # 使用LCD作为输出显示
    player = Player(Display.ST7701, display_to_ide=True)
    player.load(filename)
    player.set_event_callback(player_event)
    player.start()
    start_play = True
```

- `Player(Display.ST7701, display_to_ide=True)`：创建播放器，输出到 ST7701 LCD，且 `display_to_ide=True` 同时把画面回传 CanMV IDE 预览。
- `player.load(filename)`：加载视频文件（读取封装信息、初始化解码器）。
- `player.set_event_callback(player_event)`：登记事件回调。
- `player.start()`：开始播放；随后置 `start_play = True`。

### 第 4 步：播放期间的按键控制循环

```python
    try:
        while(start_play):
            # 读取按键状态，并做相应的按键解释
            if key0.value() == 0:
                player.pause()
                print("play pause")
            if key1.value() == 0:
                player.resume()
                print("play resume")
            time.sleep(0.1)
            os.exitpoint()
    except KeyboardInterrupt as e:
        print("user stop: ", e)
    except BaseException as e:
        import sys
        sys.print_exception(e)
```

- 循环条件是 `while(start_play)`：一旦 EOF 回调把它清零，循环自然结束。
- KEY0 → `player.pause()` 暂停并打印；KEY1 → `player.resume()` 继续并打印。
- `time.sleep(0.1)`：每 100ms 轮询一次按键，播放本身不受影响（解码由硬件异步进行）。
- `os.exitpoint()`：响应 IDE 停止；`sys.print_exception(e)` 打印完整异常堆栈便于排错。

### 第 5 步：停止播放

```python
    player.stop() #停止播放
    print("play over")

if __name__ == "__main__":
    os.exitpoint(os.EXITPOINT_ENABLE)
    play_mp4_test("/data/VIDEO/test.mp4")
```

循环退出后调用 `player.stop()` 彻底停止并释放播放器，打印 `play over`。入口先开启中断检查点，再以固定路径启动播放。

### 关键 API 小结

| API | 作用 |
| --- | --- |
| `Player(Display.ST7701, display_to_ide=True)` | 创建播放器，指定显示输出与 IDE 回传 |
| `player.load(路径)` | 加载视频文件 |
| `player.set_event_callback(函数)` | 登记播放事件回调 |
| `player.start()` | 开始播放 |
| `player.pause() / player.resume()` | 暂停 / 继续 |
| `player.stop()` | 停止播放并释放 |
| `K_PLAYER_EVENT_EOF` | "播放到文件末尾"事件常量 |

### 可调参数

- 视频路径 `"/data/VIDEO/test.mp4"`：换成你自己的 MP4 文件路径（放 TF 卡就写 `/sdcard/xxx.mp4`）。
- `display_to_ide=True`：不需要 IDE 预览可改 `False` 省 USB 带宽。
- 轮询间隔 `time.sleep(0.1)`：改小按键响应更灵敏。
- 播放模式：本例播完即退；想循环播放可在 EOF 回调里重新 `load + start`（见拓展思考 2）。

## 5. 运行效果

- **成功现象**：
  1. 运行后 LCD（以及 IDE 帧缓冲区）立即开始播放视频画面；
  2. 按 KEY0：画面定格，终端打印 `play pause`；
  3. 按 KEY1：从定格处继续播放，打印 `play resume`；
  4. 视频播完（EOF 事件触发）：自动停止，终端打印 `play over`，程序正常结束。
- **失败现象排查**：
  1. `load` 阶段抛异常、无画面 → `/data/VIDEO/test.mp4` 不存在或路径写错；按第 6 节先放好文件。
  2. 有声音提示但黑屏 → LCD 排线问题，断电重插（参考[实验10 LCD实验](../实验10%20LCD实验/README.md)）。
  3. 画面卡顿/花屏 → 视频码率过高超出解码能力，换低分辨率、低码率的 MP4 再试。
  4. 按键无反应 → GPIO34/35 映射遗漏；注意暂停中再按 KEY0 会重复打印 `play pause`，属正常。
  5. 其他格式（avi/mkv/flv）不播放 → 当前版本仅支持 MP4，请先转格式。

## 6. 代码使用说明

1. **准备视频文件**（关键步骤）：把一段 MP4 视频放到开发板 `/data/VIDEO/test.mp4`。可先运行下面的准备代码（IDE 交互终端或临时脚本）：

   ```python
   import os
   try:
       os.mkdir("/data/VIDEO")
   except Exception:
       pass
   ```

   然后把视频拷贝进该目录（通过 IDE 的文件传输功能，或把文件放 TF 卡后修改代码路径为 `/sdcard/VIDEO/test.mp4` 并确保目录存在）。
2. 用 **CanMV IDE** 打开本目录 `main.py`，USB Type-C 连接开发板，点击**连接**、**运行**。
3. 固件要求：**CanMV K230D 固件**（含 Player 解码组件），版本与 IDE 匹配。
4. **不需要模型文件；需要按上述路径准备好 MP4 视频文件**（代码实际读取 `/data/VIDEO/test.mp4`）。

> [!NOTE]
> 视频规格建议：分辨率不超过 LCD 的 640×480（更高分辨率会缩放、更吃解码能力）、常见 H.264 编码的 MP4，兼容性最好。

## 7. 拓展思考

1. **播放列表**：把多个视频路径放进列表，播完一个（EOF 后）自动 `load` 下一个，做一个简易连播播放器。
2. **循环播放**：修改 EOF 回调逻辑，收到结束事件后重新 `player.load(filename)` 并 `player.start()`，让视频无限循环（电子相册场景）。
3. **完整遥控**：增加 KEY2 或屏幕触摸按钮（参考[实验11 触摸屏实验](../实验11%20触摸屏实验/README.md)），实现"上一段/下一段/停止"，并在屏幕上用 OSD 叠加显示当前播放状态。

## 8. 常见问题

**Q1：运行报错，提示打不开文件？**
A：九成是 `/data/VIDEO/test.mp4` 不存在。确认：① `/data/VIDEO` 目录已创建；② 文件名大小写完全一致（`test.mp4` ≠ `TEST.MP4`）；③ 文件确实拷贝完成（可在终端 `import uos; uos.listdir("/data/VIDEO")` 验证）。

**Q2：为什么代码里没有初始化摄像头或 MediaManager？**
A：Player 是独立封装的多媒体组件，`Player(...)` 创建时内部自行完成解码器和显示输出的初始化，不需要像摄像头实验那样手动 `MediaManager.init()`。这正是高级接口"开箱即用"的体现。

**Q3：视频播完了程序为什么不循环播放？**
A：本例的设计是"播完即止"：EOF 事件回调把 `start_play` 置 False，主循环退出，`player.stop()` 收尾。要循环播放需自行在 EOF 后重新 load/start（见拓展思考 2）。

**Q4：暂停后画面会一直定格吗？会烧屏吗？**
A：会一直定格在你按 KEY0 的那一帧，按 KEY1 才继续。LCD（液晶屏）不存在 OLED 那种烧屏问题，短时间定格无碍。

**Q5：能把别的格式转成 MP4 吗？怎么转？**
A：可以。在电脑上用 ffmpeg 等工具转换，例如：`ffmpeg -i input.avi -c:v h264 -vf scale=640:480 output.mp4`，把分辨率缩到 640×480、编码选 H.264，与 LCD 和解码能力最匹配。转好后按第 6 节放入开发板。
