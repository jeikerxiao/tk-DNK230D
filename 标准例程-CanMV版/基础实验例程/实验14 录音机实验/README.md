# 实验14 录音机实验

> 用板载麦克风录制 15 秒双声道 WAV 音频，保存到 TF 卡，学会音频采集全流程。难度：中级

## 1. 例程简介

本例程使用 CanMV 固件的 **PyAudio 音频接口**，从开发板**板载麦克风**采集音频：以 44100Hz 采样率、16 位精度、双声道录制 **15 秒**，再通过 `wave` 模块把数据封装成标准 WAV 文件保存到 `/sdcard/app/output.wav`。你将学会**音频采集流（stream）的概念、采样率/声道/位深三要素、WAV 文件封装、资源释放**。难度为**中级**，是语音识别、声音报警等项目的数据入口。

## 2. 硬件连接与原理

| 外设 | 资源 | 代码中的体现 |
| --- | --- | --- |
| 板载麦克风 | 芯片音频输入（固件内部驱动，无需引脚映射） | `stream = p.open(..., input=True, ...)` |
| TF 卡 / 存储 | 挂载点 `/sdcard/`，保存路径 `/sdcard/app/output.wav` | `record_audio('/sdcard/app/output.wav', 15)` |
| USB 串口 | 打印开始/结束提示 | `print("audio sample start")` |

**无需任何接线**，麦克风是板上资源；TF 卡用于保存录音文件。

### 硬件原理（通俗版）

**麦克风**把声音（空气振动）转换成连续变化的电压，**ADC（模数转换）** 再按固定节奏把这个电压"拍照"成一串数字——每秒拍多少次就是**采样率**（本例 44100Hz，即 CD 音质标准，每秒 44100 个样本）；每个样本用几位二进制保存就是**位深/采样精度**（本例 16 位，即 `paInt16`）；同时用几个麦克风记录就是**声道数**（本例 2 个声道，双声道/立体声）。

一秒录音的数据量 = 44100 样本 × 2 字节 × 2 声道 ≈ 176KB，15 秒约 2.6MB，TF 卡轻松容纳。

**WAV 文件**是最简单的音频文件格式：44 字节的"文件头"（写明采样率、声道数、位深）+ 原始样本数据。任何播放器都能直接播放，不需要解码算法。

采集时数据以 **chunk（块）** 为单位流动：代码里 `CHUNK = int(44100/25) = 1764`，即每块 1764 个样本，相当于 1/25 秒（40ms）的音频；录 15 秒就是循环读 `25 × 15 = 375` 块。

### 新手风险提醒

> [!WARNING]
> - **保存目录必须已存在**：代码写入 `/sdcard/app/output.wav`，如果 `/sdcard/app/` 目录不存在会抛异常。请先建好目录（见第 6 节）。
> - 录音过程中不要断电或拔卡，否则 WAV 文件不完整无法播放。
> - 插拔 TF 卡必须断电操作。

## 3. 相关前置知识

- **采样率（RATE）**：每秒采集的样本数，44100Hz 可完整记录人耳可听范围（约 20Hz~20kHz）的声音。
- **采样精度（FORMAT=paInt16）**：每个样本量化成 16 位有符号整数（-32768~32767），数值越大声音越细腻。
- **声道（CHANNELS=2）**：双声道数据交错存放。
- **音频流（stream）**：一条持续产生音频数据的"管道"，`stream.read()` 每次取出一块（chunk）数据。
- **chunk（数据块）**：音频采集的最小搬运单位。块越小延迟越低、块越大越省 CPU；本例取 1/25 秒。
- **WAV 封装**：把原始样本数据加上描述头存成 `.wav` 文件。
- **字节串拼接**：`b''.join(frames)` 把几百块 bytes 数据首尾相连成一整段，`b''` 前缀表示"空的字节串"。
- **MediaManager / os.exitpoint**：媒体内存管理与 IDE 中断检查点（同[实验10 LCD实验](../实验10%20LCD实验/README.md)）。

## 4. 软件代码逻辑说明

流程：**导入模块 → 初始化音频 → 打开输入流 → 循环读块 → 封装 WAV 保存 → 释放资源**。

### 第 1 步：导入模块与退出检查

```python
import os
from media.media import *   #导入media模块，用于初始化vb buffer
from media.pyaudio import * #导入pyaudio模块，用于采集和播放音频
import media.wave as wave   #导入wav模块，用于保存和加载wav音频文件

def exit_check():
    try:
        os.exitpoint()
    except KeyboardInterrupt as e:
        print("user stop: ", e)
        return True
    return False
```

三个模块都是 CanMV 固件原生自带：`media.pyaudio` 提供音频采集/播放，`media.wave` 提供 WAV 文件读写。`exit_check()` 封装了"检查 IDE 是否点了停止"，返回 True 表示用户要求中断。

### 第 2 步：录音函数——参数设置与初始化

```python
def record_audio(filename, duration):
    CHUNK = int(44100/25)  #设置音频chunk值
    FORMAT = paInt16       #设置采样精度
    CHANNELS = 2           #设置声道数
    RATE = 44100           #设置采样率
    try:
        p = PyAudio()
        p.initialize(CHUNK)    #初始化PyAudio对象
        MediaManager.init()    #vb buffer初始化

        #创建音频输入流
        stream = p.open(format=FORMAT,
                        channels=CHANNELS,
                        rate=RATE,
                        input=True,
                        frames_per_buffer=CHUNK)
```

- `PyAudio()` 创建音频对象，`initialize(CHUNK)` 按块大小初始化。
- `p.open(...)` 打开**输入流**：`input=True` 表示录音方向（播放则是 `output=True`），格式/声道/采样率与上面常量一致，`frames_per_buffer=CHUNK` 指定每块的样本数。

### 第 3 步：循环采集数据块

```python
        frames = []
        #采集音频数据并存入列表
        for i in range(0, int(RATE / CHUNK * duration)):
            data = stream.read()
            frames.append(data)
            if exit_check():
                break
```

循环次数 = `RATE / CHUNK × duration` = `44100 / 1764 × 15` = `25 × 15` = **375 次**，每次 `stream.read()` 读一块（1/25 秒）音频存入列表 `frames`；期间若用户在 IDE 点停止，`exit_check()` 返回 True 提前跳出。

### 第 4 步：封装成 WAV 文件

```python
        #将列表中的数据保存到wav文件中
        wf = wave.open(filename, 'w') #创建wav 文件
        wf.set_channels(CHANNELS) #设置wav 声道数
        wf.set_sampwidth(p.get_sample_size(FORMAT))  #设置wav 采样精度
        wf.set_framerate(RATE)  #设置wav 采样率
        wf.write_frames(b''.join(frames)) #存储wav音频数据
        wf.close() #关闭wav文件
```

`wave.open(路径, 'w')` 创建 WAV 文件，依次写入声道数、采样位宽（`p.get_sample_size(paInt16)` 返回 2 字节）、采样率这三项"文件头信息"，然后把所有块拼接成的完整音频数据写入，最后关闭文件。**头信息与数据必须匹配**，否则播放器会变速或出噪声。

### 第 5 步：释放资源

```python
    except BaseException as e:
            print(f"Exception {e}")
    finally:
        stream.stop_stream() #停止采集音频数据
        stream.close()#关闭音频输入流
        p.terminate()#释放音频对象
        MediaManager.deinit() #释放vb buffer
```

`finally` 保证无论如何都执行：停流 → 关流 → 释放 PyAudio → 释放媒体内存。

### 第 6 步：程序入口

```python
if __name__ == "__main__":
    os.exitpoint(os.EXITPOINT_ENABLE)
    print("audio sample start")
    record_audio('/sdcard/app/output.wav', 15)  #录制wav文件
    print("audio sample done")
```

开启中断检查点 → 打印开始 → 录 15 秒保存到 `/sdcard/app/output.wav` → 打印结束。

### 关键 API 小结

| API | 作用 |
| --- | --- |
| `PyAudio()` / `p.initialize(CHUNK)` | 创建并初始化音频子系统 |
| `p.open(format, channels, rate, input=True, frames_per_buffer)` | 打开音频输入流 |
| `stream.read()` | 读取一个 chunk 的音频数据 |
| `wave.open(path, 'w')` | 创建 WAV 文件写入器 |
| `wf.set_channels/set_sampwidth/set_framerate/write_frames/close` | 写文件头与数据并关闭 |
| `stream.stop_stream()/close()`、`p.terminate()` | 逐级释放音频资源 |

### 可调参数

- `duration=15`：录音秒数（入口函数第二个参数），改 5 就录 5 秒。
- `CHANNELS = 2`：改成 1 为单声道，文件减半。
- `RATE = 44100`：改成 16000（语音识别常用）文件更小；注意 CHUNK 与循环次数的计算要同步自洽。
- 保存路径 `/sdcard/app/output.wav`：可改文件名/目录（目录必须存在）。

## 5. 运行效果

- **成功现象**：
  1. 终端打印 `audio sample start`；
  2. 15 秒内对着板载麦克风说话/放音乐；
  3. 到时后打印 `audio sample done`；
  4. 把 TF 卡（或经 IDE 文件管理）取出 `/sdcard/app/output.wav`，用电脑播放器打开，能听到刚才录的 15 秒声音（约 2.6MB，双声道 44.1kHz）。
- **失败现象排查**：
  1. 报 `OSError`/异常且没生成文件 → `/sdcard/app/` 目录不存在，先创建（见第 6 节）。
  2. 文件生成了但没声音/全是杂音 → 录音时离麦克风太远，或 `set_channels` 等头信息与实际数据不匹配（改动过参数）。
  3. 播放速度不对（变快/变慢）→ `set_framerate` 写入的采样率与采集时不一致。
  4. 中途想停止 → 在 IDE 点停止，程序会在当前块结束时退出（文件可能未写入，属正常）。

## 6. 代码使用说明

1. **准备存储目录**：程序写入 `/sdcard/app/output.wav`，请确保 `/sdcard/app/` 目录存在。最简单的方法是在 IDE 交互终端执行：

   ```python
   import uos
   uos.mkdir("/sdcard/app")   # 若已存在会报错，忽略即可
   ```

   （目录操作详见[实验13 文件读写实验](../实验13%20文件读写实验/README.md)。）
2. 断电插入 TF 卡（FAT32 格式），上电后用 **CanMV IDE** 打开本目录 `main.py`，连接开发板。
3. 点击**运行**，打印 `audio sample start` 后开始对麦克风说话，15 秒后自动结束。
4. 固件要求：**CanMV K230D 固件**（含 pyaudio/wave 模块），版本与 IDE 匹配。
5. **需要 TF 卡或可用的 /sdcard 挂载分区；不需要模型文件。**

> [!NOTE]
> 录音文件在 TF 卡上，取出方法：停止程序并断电后拔卡用读卡器读；或保留卡在板上，用文件操作代码把内容读出。

## 7. 拓展思考

1. **按键控制录音**：结合[实验3 按键输入实验](../实验3%20按键输入实验/README.md)，改成"按住 KEY0 才录音、松开即停并保存"，做一个真正的按键录音笔（提示：把时长参数改成动态判断）。
2. **播放录音**：PyAudio 同样支持输出流（`output=True`），研究固件文档用 `wave.open(path, 'r')` 读出刚才的文件并从扬声器/耳机播放，凑齐"录 + 放"闭环。
3. **分段录音**：修改文件名带序号（`output_1.wav`、`output_2.wav`……），每按一次 KEY2 录一段 5 秒，做连续语音记录器；配合 `uos.listdir` 检查已录段数。

## 8. 常见问题

**Q1：为什么 `CHUNK = int(44100/25)` 等于 1764？这个 25 是什么？**
A：25 表示每秒把音频切成 25 块，每块 = 44100 ÷ 25 = 1764 个样本，时长 40ms。块大小是延迟和 CPU 开销的折中：块小响应快但中断频繁，块大省 CPU 但延迟高。录音场景 40ms 一块很合适。

**Q2：运行直接抛异常，文件也没生成，怎么办？**
A：先检查 `/sdcard/app/` 目录是否存在——`wave.open('/sdcard/app/output.wav', 'w')` **不会自动创建目录**。按第 6 节方法先 `uos.mkdir("/sdcard/app")`。其次确认 TF 卡已挂载（可先跑一下[实验13 文件读写实验](../实验13%20文件读写实验/README.md)的 `uos.listdir("/sdcard/")` 验证）。

**Q3：录出来的声音很小？**
A：板载麦克风灵敏度和距离有限，请靠近（10~30cm）正常音量说话。另外确认录的是正确时段——程序打印 `audio sample start` 后就要开始出声，15 秒一到立即结束。

**Q4：`frames` 列表存了 375 块数据，会不会内存不够？**
A：每块 1764 样本 × 2 字节 × 2 声道 ≈ 7KB，375 块约 2.6MB，K230D 的内存足够。但如果把 `duration` 改成几百秒，就可能内存不足——长录音应改成**边录边写**文件，而不是全部攒在列表里。

**Q5：双声道数据是左右分开存的吗？**
A：不是。WAV 中双声道数据是**交错存放**的：左1、右1、左2、右2……`wave` 模块按 `set_channels(2)` 自动处理这种排布，播放器也会正确还原为立体声。
