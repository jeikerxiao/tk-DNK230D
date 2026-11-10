# 实验19 关键词唤醒实验

> 本例程在正点原子 DNK230D（K230D BOX）开发板上运行关键词唤醒（KWS）语音模型，通过板载麦克风持续听音，检测到唤醒词"小楠小楠"后播放回复音频"我在"并在屏幕上显示提示。难度：高级。

## 1. 例程简介

关键词唤醒（Keyword Spotting，KWS）是智能音箱、语音助手的第一步：设备平时低功耗地"监听"，只有听到特定唤醒词才开始工作。本实验用 K230D 的 KPU 运行一个流式 KWS 模型，从板载麦克风采集 16kHz 音频，每 0.3 秒推理一次，判断这段时间内是否包含唤醒词"小楠小楠"。

- **知识点**：音频采集（PyAudio/PCM）、音频特征提取（`aidemo.kws_preprocess`）、流式推理与 cache 张量反馈、音频播放（wave/PyAudio 输出流）、功放使能 GPIO 控制、非 PipeLine 的 Display 直驱显示。
- **难度**：高级（涉及音频链路、双输入张量流式模型和 GPIO 控制，是本套 AI 实验中唯一不使用摄像头的例程）。
- **用途**：语音助手唤醒、口令触发控制、离线语音交互入口。

> [!NOTE]
> - 唤醒词为**"小楠小楠"**（模型固定，无法通过改代码更换唤醒词）。
> - 模型文件：`/sdcard/examples/kmodel/kws.kmodel`；回复音频：`/sdcard/examples/utils/wozai.wav`。
> - 代码头注释说明：**K230D-BOX 默认无扬声器**，可以通过显示器（LCD）唤醒现象来查看结果；若要听声音需外接扬声器。

## 2. 硬件连接与原理

### 2.1 硬件清单

| 硬件 | 说明 |
| --- | --- |
| CNK230DF 核心板 | 嘉楠 K230D 芯片（RISC-V 双核 + 6TOPS 算力 KPU） |
| DNK230D 底板 | 板载麦克风（录音输入）、功放电路 |
| 2.4 寸 MIPI LCD | 唤醒成功后显示 "Hello K230D-BOX" |
| 扬声器（选配） | K230D BOX 默认不带扬声器，想听到"我在"回复需外接 |
| GPIO63 | 功放使能控制引脚（代码中 `spk_sd`，置 0 使能） |
| TF 卡 | 存放固件、`kws.kmodel`、`wozai.wav`、`libs` 库 |

本实验**不使用摄像头**，核心外设是板载麦克风与音频输出电路。

> [!WARNING]
> - LCD 排线插反会烧坏接口，务必断电插接。
> - 外接扬声器时注意功放输出功率，不要接入阻抗过低的负载。
> - 麦克风对环境噪声敏感，实验请在相对安静环境下进行；供电不足会引起音频采集异常。

### 2.2 工作原理（通俗版）

1. **采音**：PyAudio 打开输入流，以 16000Hz 采样率、单声道、16bit 格式，每次读取 0.3 秒（4800 个采样点）的 PCM 原始音频。
2. **特征提取**：每 2 字节还原成一个采样值，`aidemo.kws_preprocess` 把 4800 个采样点变换成 (1,30,40) 的声学特征图（类似"声音的缩略图"，横轴时间、纵轴频率能量）。
3. **流式推理**：KWS 模型有两个输入——本帧声学特征 + 上一帧留下的 cache 张量 (1,256,105)；输出除了分类得分 logits 外，还输出**新的 cache**，代码把它存回 `self.cache_np`，作为下一帧的第二个输入。这样模型"记得"前文，能识别跨越 0.3 秒边界的完整词语。
4. **判决**：对 logits 取最大值 `max_p` 和对应类别 `idx`；当 `max_p > 0.5` 且 `idx == 1`（类别 1 = 含唤醒词）时判定唤醒成功。
5. **响应**：播放 `wozai.wav`（"我在"）到输出流（经 GPIO63 使能的功放），LCD 清屏后显示红色 "Hello K230D-BOX"，串口打印 `====Detected XiaonanXiaonan!====`；未唤醒时串口持续打印 `Deactivated!`。

## 3. 相关前置知识

- **PCM 音频**：麦克风输出的原始数字信号，每个采样点是一个 16bit 有符号整数（`struct.unpack("<h", ...)` 小端解析）。16000Hz 采样率即每秒记录 16000 个数值。
- **CHUNK / 帧长**：`CHUNK = int(0.3 * 16000) = 4800`，即每次读 0.3 秒音频，这就是模型的"滑窗"步长。
- **声学特征**：原始波形不能直接喂神经网络，需要先变换成时频特征（帧数 30 × 频带 40），由 `aidemo.kws_preprocess` 完成，特征提取器句柄由 `aidemo.kws_fp_create()` 创建。
- **流式模型与 cache**：KWS 是边听边判的流式网络。cache 张量保存网络内部的历史状态（如卷积/LSTM 记忆），每帧推理后必须把输出 cache 反馈给下一帧，否则模型"失忆"，无法识别完整词语。
- **logits 与阈值**：logits 是模型对"非唤醒/唤醒"两个类别的原始打分；`np.max`、`np.argmax` 找出最高分和对应类别，超过 `THRESH=0.5` 才算数，避免把普通说话误判成唤醒词。
- **wav 文件播放**：`media.wave` 打开 wav 文件，按 CHUNK 分块读出数据写入 PyAudio 输出流，即完成播放。
- **FPIOA / Pin**：K230D 的引脚功能可重映射，`fpioa.set_function(63, FPIOA.GPIO63)` 把物理引脚 63 配置为 GPIO63，再用 `Pin(63, Pin.OUT)` 控制功放的使能端。
- **Display 直驱显示**：本实验不用 PipeLine（无摄像头），直接 `Display.init(Display.ST7701, ...)` 初始化 LCD，用 `image.Image` 画布 + `Display.show_image(img)` 显示。

## 4. 软件代码逻辑说明

### 4.1 程序整体流程

```
配置 GPIO63（功放使能，置 0）→ 初始化 LCD（Display.ST7701, 640×480）
→ aidemo.kws_fp_create() 创建音频特征提取器
→ PyAudio 初始化，打开输入流（麦克风）与输出流（喇叭）
→ 创建 KWSApp 加载 kws.kmodel
→ 主循环：读 0.3s PCM → 特征提取 → KPU 推理（特征 + cache 双输入）
   → 判决 → 唤醒则播放 wozai.wav + LCD 显示；否则打印 Deactivated!
→ 退出：关音频流、Display.deinit、销毁特征器、kws.deinit()
```

### 4.2 功放使能与全局设置

```python
fpioa = FPIOA()
fpioa.set_function(63, FPIOA.GPIO63)   # 设置Pin63为GPIO63
spk_sd = Pin(63, Pin.OUT, pull=Pin.PULL_NONE, drive=7)
spk_sd.value(0)                        # 拉低使能功放

os.exitpoint(os.EXITPOINT_ENABLE)
nn.shrink_memory_pool()                # 收缩nncase内存池，给音频/显示留内存
```

### 4.3 参数配置（`__main__` 部分）

```python
kmodel_path = "/sdcard/examples/kmodel/kws.kmodel"
THRESH = 0.5                # 检测阈值
SAMPLE_RATE = 16000         # 采样率16000Hz
CHANNELS = 1                # 单声道
FORMAT = paInt16            # 16bit PCM 格式
CHUNK = int(0.3 * 16000)    # 每次读取0.3s音频（4800点）
reply_wav_file = "/sdcard/examples/utils/wozai.wav"  # 唤醒回复音频
```

### 4.4 音频与显示初始化

```python
img = image.Image(DISPLAY_WIDTH, DISPLAY_HEIGHT, image.RGB888)
Display.init(Display.ST7701, width=DISPLAY_WIDTH, height=DISPLAY_HEIGHT, to_ide=True)
fp = aidemo.kws_fp_create()               # 音频特征提取器
p = PyAudio()
p.initialize(CHUNK)
MediaManager.init()
input_stream = p.open(format=FORMAT,channels=CHANNELS,rate=SAMPLE_RATE,
                      input=True,frames_per_buffer=CHUNK)   # 录音流
output_stream = p.open(format=FORMAT,channels=CHANNELS,rate=SAMPLE_RATE,
                       output=True,frames_per_buffer=CHUNK) # 播放流
```

### 4.5 KWSApp 类：预处理（波形 → 特征 + cache 双张量）

```python
def preprocess(self,pcm_data):
    pcm_data_list=[]
    for i in range(0, len(pcm_data), 2):
        # 每两个字节组织成一个有符号整数（一次采样）
        int_pcm_data = struct.unpack("<h", pcm_data[i:i+2])[0]
        pcm_data_list.append(float(int_pcm_data))
    # pcm数据处理为模型输入的特征向量
    mp_feats = aidemo.kws_preprocess(fp, pcm_data_list)[0]
    mp_feats_np = np.array(mp_feats).reshape((1, 30, 40))
    audio_input_tensor = nn.from_numpy(mp_feats_np)   # 输入1：声学特征
    cache_input_tensor = nn.from_numpy(self.cache_np) # 输入2：历史cache
    return [audio_input_tensor,cache_input_tensor]
```

`self.cache_np` 初始为 `np.zeros((1, 256, 105), dtype=np.float)`。

### 4.6 KWSApp 类：后处理（判决 + cache 反馈）

```python
def postprocess(self, results):
    logits_np = results[0]
    self.cache_np = results[1]            # 输出cache留给下一帧
    max_logits = np.max(logits_np, axis=1)[0]
    max_p = np.max(max_logits)
    idx = np.argmax(max_logits)
    # 分数大于阈值且idx==1（包含唤醒词）→ 唤醒
    if max_p > self.threshold and idx == 1:
        return 1
    else:
        return 0
```

### 4.7 主循环：听音 → 判决 → 响应

```python
while True:
    os.exitpoint()
    with ScopedTiming("total",1):
        pcm_data=input_stream.read()   # 读0.3s音频
        res=kws.run(pcm_data)          # 特征提取+推理+判决
        if res:
            print("====Detected XiaonanXiaonan!====")
            wf = wave.open(reply_wav_file, "rb")
            wav_data = wf.read_frames(CHUNK)
            while wav_data:                     # 分块播放"我在"
                output_stream.write(wav_data)
                wav_data = wf.read_frames(CHUNK)
            time.sleep(1)                       # 播放缓冲
            wf.close()
            img.clear()
            time.sleep_ms(100)
            img.draw_string(0, 20, "Hello K230D-BOX\r\n", color=(255, 0, 0), scale=4)
            Display.show_image(img)             # LCD显示提示
        else:
            print("Deactivated!")
        gc.collect()
```

### 4.8 资源释放（finally）

依次执行：停/关两个音频流 → `p.terminate()` → `Display.deinit()` → `MediaManager.deinit()` → `aidemo.kws_fp_destroy(fp)` → `kws.deinit()`。

### 4.9 可调参数一览

| 参数 | 位置 | 作用 |
| --- | --- | --- |
| `THRESH=0.5` | `__main__` | 调大 → 更难误唤醒但可能唤不醒；调小 → 更灵敏但易误触发 |
| `CHUNK` | `__main__` | 0.3s 窗口长度，与模型匹配，不建议修改 |
| `reply_wav_file` | `__main__` | 换成自己的 16kHz/16bit/单声道 wav 即可换回复语 |
| `img.draw_string(...)` | 主循环 | 唤醒后屏幕文字内容/颜色/位置 |
| `time.sleep(1)` | 主循环 | 播放后的缓冲时间，也是防连触间隔 |

## 5. 运行效果

- **未唤醒时**：串口终端持续打印 `Deactivated!`（约每 0.3 秒一次），LCD 无提示。
- **唤醒成功时**：
  - 串口打印 `====Detected XiaonanXiaonan!====`；
  - 若接了扬声器（GPIO63 功放已使能），播放"我在"回复音；K230D BOX 默认无扬声器，听不到声音属正常；
  - LCD 显示红色放大文字 `Hello K230D-BOX`。
- **成功标志**：对板载麦克风清晰地说"小楠小楠"，1 秒内出现上述现象。
- **失败现象排查**：
  - 一直 `Deactivated!` → 提高音量、靠近麦克风、放慢语速；确认环境安静；可尝试把 `THRESH` 降到 0.3。
  - 频繁误唤醒 → 调大 `THRESH`。
  - 无声音但有串口/LCD 响应 → 板上无扬声器，属预期；需外接扬声器到功放输出。
  - 运行报错找不到文件 → 检查 TF 卡上 `examples/kmodel/kws.kmodel` 与 `examples/utils/wozai.wav`。

## 6. 代码使用说明

1. **固件准备**：烧录与 CanMV IDE 版本匹配的 **CanMV K230D 固件**（正点原子出厂固件即可）。
2. **拷贝模型与资源**：将正点原子资料中的以下文件拷贝到 TF 卡：
   - `/sdcard/examples/kmodel/kws.kmodel`（关键词唤醒模型）
   - `/sdcard/examples/utils/wozai.wav`（唤醒回复音频）
3. **依赖库确认**：`libs.AIBase`（本实验还 import 了 `libs.PipeLine`、`libs.AI2D` 但未使用摄像头链路）、`media.pyaudio`、`media.wave`、`aidemo`（`kws_fp_create/kws_preprocess/kws_fp_destroy`）、`nncase_runtime`、`ulab.numpy`，均为固件自带，无需安装第三方库。
4. **运行步骤**：
   - 打开 CanMV IDE → 连接开发板 → 打开本目录 `main.py` → 运行；
   - 打开串口终端观察输出；
   - 对板载麦克风说"小楠小楠"，观察串口打印、LCD 显示（和外接扬声器的声音）；
   - 停止运行，程序自动释放音频、显示与模型资源。

> [!WARNING]
> 本实验会长时间占用麦克风和音频输出流。停止程序后如果立刻重新运行报音频相关错误，可等待几秒或给开发板复位后再试。

## 7. 拓展思考

1. **唤醒后控制 LED/蜂鸣器**：在 `if res:` 分支中加入 `machine.Pin` 操作，唤醒成功时点亮板载双色 LED（红 GPIO61、蓝 GPIO59）或驱动蜂鸣器，做成"语音控灯"。
2. **自定义回复音频**：用音频工具把一段话导出为 16kHz、16bit、单声道 wav，替换 `wozai.wav`（或修改 `reply_wav_file` 路径），让开发板说出自己的应答语。
3. **唤醒次数统计**：增加一个计数器变量，每次唤醒 +1 并打印/显示在 LCD 上；进一步思考如何加入"冷却时间"防止一次呼叫被连续判定多次。

## 8. 常见问题

**Q1：为什么运行时听不到"我在"的声音？**
A：K230D-BOX 默认**没有扬声器**（代码注释已说明）。功放已由 GPIO63 使能，但需要外接扬声器才能发声；没有扬声器时请通过 LCD 显示和串口打印确认唤醒效果。

**Q2：能换成"你好小天"之类的其他唤醒词吗？**
A：不能通过改代码实现。唤醒词由 `kws.kmodel` 模型在训练时固定，更换唤醒词需要用对应数据集重新训练并转换模型。

**Q3：`cache_np` 是什么？删掉会怎样？**
A：它是流式模型的历史状态张量 (1,256,105)，模型每帧输出的 `results[1]` 会更新它并反馈给下一帧输入。删掉或每帧都清零，模型将无法感知 0.3 秒窗口之间的上下文，唤醒率会大幅下降。

**Q4：串口一直打印 Deactivated!，说唤醒词也没反应？**
A：逐项排查：① 是否对着**板载麦克风**说话（距离 10~30cm）；② 环境是否嘈杂；③ 语速是否正常、发音是否清晰（"小楠小楠"四字连读两遍）；④ 尝试把 `THRESH` 从 0.5 降到 0.3 测试。

**Q5：为什么这个实验不用 PipeLine 和摄像头？**
A：PipeLine 封装的是"摄像头→AI→显示"的图像链路。KWS 是纯音频任务，因此代码直接初始化 `Display` 显示结果、用 `PyAudio` 处理音频，`AIBase` 仅用于加载 kmodel 和调度推理。

**Q6：`nn.shrink_memory_pool()` 有什么作用？**
A：收缩 nncase（KPU 运行时）的内存池占用，为音频缓冲、LCD 画布等留出内存。KWS 模型小，不需要大内存池；若注释掉该行，可能因内存不足导致音频或显示初始化失败。
