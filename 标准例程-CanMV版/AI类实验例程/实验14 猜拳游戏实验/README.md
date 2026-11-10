# 实验14 猜拳游戏实验

> 和开发板玩石头剪刀布！程序识别你出的 fist（石头）、yeah（剪刀）、five（布），随机出拳并按"三局两胜"判定输赢，还支持"玩家稳赢/必输"的娱乐模式。难度：高级。

## 1. 例程简介

本例程在手势识别的基础上做了一个完整的**互动小游戏**：复用实验11 的两级模型（`hand_det.kmodel` 手掌检测 + `handkp_det.kmodel` 关键点与几何手势判别），加上游戏状态管理（回合计数、胜负计分、结果展示）和三张 400x400 的石头/剪刀/布贴图（bin 文件）。`guess_mode` 参数决定玩法：0=玩家稳赢、1=玩家必输、>=2 的值（默认 3）=多局制公平对战（三局两胜，按 `counts_guess == guess_mode` 结算）。你将学到：AI 识别结果如何驱动一个带规则的应用层逻辑、随机数出拳、贴图显示与游戏流程控制。难度为**高级**（AI 部分与实验11 相同，复杂在应用层状态机），预期用途：互动展示、教学演示、手势游戏开发模板。

## 2. 硬件连接与原理

用到的硬件：

- **CNK230DF 核心板（K230D）**：RISC-V 双核 + 6TOPS KPU；
- **MIPI CSI 摄像头**：`Sensor(width=1280, height=960)`；送 AI 分辨率 `rgb888p_size=[1024,768]`；
- **2.4 寸 MIPI LCD**：`display_mode="lcd"`、`display_size=[640,480]`，显示贴图、回合与胜负文字；
- **TF 卡（/sdcard）**：存放两个 kmodel 与三张手势贴图 bin 文件。

**游戏原理（通俗版）**：AI 只负责"看懂你出了什么"——手掌检测找到手，关键点模型给出 21 个点，几何规则（`hk_gesture`，与实验11 完全相同）判出 fist/yeah/five。游戏逻辑全在 `draw_result()` 里：

- **公平模式（guess_mode=3）**：开发板用 `randint(1,10000) % 3` 随机生成 0/1/2，映射到 `LIBRARY = ["fist","yeah","five"]` 得到自己的出拳；按石头胜剪刀、剪刀胜布、布胜石头的规则给 `player_win`/`k230_win` 记分；打满 `guess_mode` 局后比较总分显示"你赢了！/你输了！/平局"，然后清零重来。
- **每局节奏控制**：出拳被识别后 `set_stop_id=False` 锁定本局，并 `sleep_ms(2000)` 停顿展示结果；**手移出画面**（`len(dets)==0`）才复位 `set_stop_id=True` 允许下一局——所以官方玩法是"先保持镜头无手，再把手移入出拳"（见文件头注释）。
- **娱乐模式 0/1**：不做随机，直接按你的出拳贴出"必输/必赢"的图：mode 0 时你出石头它贴剪刀图，你出布它贴石头图（你稳赢）；mode 1 反之（你必输）。

> [!WARNING]
> - 摄像头/LCD 的 MIPI 排线必须**断电插拔**，插反通电会烧坏器件；
> - 供电不足会导致推理时重启，请使用足功率 Type-C 电源；
> - AI 推理发热明显，**属正常现象**；
> - 游戏结果展示阶段有 `time.sleep_ms(2000)` 阻塞，期间画面暂停属正常设计，不是死机。

## 3. 相关前置知识

- **几何手势判别（hk_gesture）**：与实验11 相同的 9 种手势角度规则，本游戏只用 fist（五指全弯=石头）、yeah（食指+中指伸=剪刀）、five（五指全伸=布）三种。
- **关键点 21 点模型 / 向量夹角 / 阈值 65°/53°/49°**：见实验10、11。
- **状态机与标志位**：`counts_guess`（当前局数，-1 表示未开始）、`player_win`/`k230_win`（比分）、`set_stop_id`（本局是否已出拳锁定）、`sleep_end`（下一帧是否需要停顿 2 秒）——四个变量构成游戏流程控制。
- **随机数**：`from random import randint`，`randint(1,10000) % 3` 产生 0~2 的随机出拳。
- **ARGB 贴图 bin 文件**：`five.bin / fist.bin / shear.bin` 是 400x400x4 字节的裸 ARGB 图像数据，`read_file()` 用 `np.fromfile` 读入并 `reshape((400,400,4))`，再赋值给画布数组左上角 400x400 区域显示。
- **crop / ROI / 锚框 / 置信度 / NMS**：两级模型推理相关概念，同实验8~13。
- **帧率（FPS）**：`ScopedTiming("total",1)` 打印每帧耗时；游戏停顿帧不计入。

## 4. 软件代码逻辑说明

### 4.1 框架导入

```python
from libs.PipeLine import PipeLine, ScopedTiming
from libs.AIBase import AIBase
from libs.AI2D import Ai2d
from random import randint
import nncase_runtime as nn
import ulab.numpy as np
import aicube
```

### 4.2 AI 部分：HandDetApp 与 HandKPClassApp

两个类与实验11 **完全同构**：

- `HandDetApp`：pad（114 灰边）+ resize 到 512x512，`aicube.anchorbasedet_post_process` 后处理，`labels=["hand"]`；
- `HandKPClassApp`：crop（1.26 倍外扩）+ resize 到 256x256，后处理返回屏幕坐标关键点数组和 `hk_gesture()` 判出的手势名。

### 4.3 游戏类 FingerGuess 的初始化

```python
self.guess_mode = guess_mode
# 石头剪刀布的贴图array
self.five_image = self.read_file("/sdcard/examples/utils/five.bin")
self.fist_image = self.read_file("/sdcard/examples/utils/fist.bin")
self.shear_image = self.read_file("/sdcard/examples/utils/shear.bin")
self.counts_guess = -1      # 猜拳次数 计数
self.player_win = 0         # 玩家 赢次计数
self.k230_win = 0           # k230 赢次计数
self.sleep_end = False      # 是否 停顿
self.set_stop_id = True     # 是否 暂停猜拳
self.LIBRARY = ["fist","yeah","five"]    # 猜拳 石头剪刀布 三种方案
```

### 4.4 run()：每帧识别

```python
def run(self, input_np):
    det_boxes = self.hand_det.run(input_np)     # 先进行手掌检测
    boxes = []
    gesture_res = []
    for det_box in det_boxes:
        ...   # 三条"太小/贴边"过滤规则（同实验8）
        self.hand_kp.config_preprocess(det_box)
        results_show, gesture = self.hand_kp.run(input_np)
        boxes.append(det_box)
        gesture_res.append(gesture)
    return boxes, gesture_res
```

### 4.5 draw_result()：游戏逻辑核心

分支一：两只手以上入镜 → 显示"请保证只有一只手入镜！"。

分支二：`guess_mode == 0`（玩家稳赢）/`== 1`（玩家必输）——直接按你的手势贴对应的"输/赢"图：

```python
if (self.guess_mode == 0):
    if (gesture_res[0] == "fist"):
        draw_img_np[:400,:400,:] = self.shear_image   # 你出石头，它出剪刀
    elif (gesture_res[0] == "five"):
        draw_img_np[:400,:400,:] = self.fist_image    # 你出布，它出石头
    elif (gesture_res[0] == "yeah"):
        draw_img_np[:400,:400,:] = self.five_image    # 你出剪刀，它出布
```

分支三：公平对战模式（`guess_mode >= 2`）的关键流程：

```python
if (self.sleep_end):                 # 上一局刚结束：停顿2秒展示结果
    time.sleep_ms(2000)
    self.sleep_end = False
if (len(dets) == 0):                 # 画面无手：允许下一次出拳
    self.set_stop_id = True
    return
if (self.counts_guess == self.guess_mode):   # 打满局数：结算
    if (self.k230_win > self.player_win):
        draw_img.draw_string_advanced(..., "你输了！", ...)
    elif (self.k230_win < self.player_win):
        draw_img.draw_string_advanced(..., "你赢了！", ...)
    else:
        draw_img.draw_string_advanced(..., "平局", ...)
    self.counts_guess = -1           # 清零重开
    self.player_win = 0
    self.k230_win = 0
    self.sleep_end = True
else:
    if (self.set_stop_id):           # 等待玩家出拳
        if (... 手势是 fist/yeah/five 之一):
            k230_guess = randint(1,10000) % 3        # 开发板随机出拳
            if (gesture_res[0] == "fist" and self.LIBRARY[k230_guess] == "yeah"):
                self.player_win += 1                 # 石头砸剪刀，玩家胜
            elif (gesture_res[0] == "fist" and self.LIBRARY[k230_guess] == "five"):
                self.k230_win += 1                   # 布包石头，板胜
            ...                                      # yeah/five 同理共6条胜负规则
            # 贴上开发板出拳的图片
            if (self.LIBRARY[k230_guess] == "fist"):
                draw_img_np[:400,:400,:] = self.fist_image
            ...
            self.counts_guess += 1
            draw_img.draw_string_advanced(..., "第" + str(self.counts_guess) + "回合", ...)
            self.set_stop_id = False                 # 锁定本局，等手移出画面
            self.sleep_end = True
```

### 4.6 主流程

```python
hand_det_kmodel_path = "/sdcard/examples/kmodel/hand_det.kmodel"
hand_kp_kmodel_path = "/sdcard/examples/kmodel/handkp_det.kmodel"
rgb888p_size = [1024, 768]
hand_det_input_size = [512, 512]
hand_kp_input_size = [256, 256]
confidence_threshold = 0.2
nms_threshold = 0.5
labels = ["hand"]
guess_mode = 3        # 猜拳模式  0 玩家稳赢 ， 1 玩家必输 ， n >= 2 多局多胜

sensor = Sensor(width=1280, height=960)
pl = PipeLine(rgb888p_size=rgb888p_size, display_size=display_size, display_mode=display_mode)
pl.create(sensor=sensor)
hkc = FingerGuess(..., guess_mode=guess_mode, rgb888p_size=rgb888p_size, display_size=display_size)
while True:
    os.exitpoint()
    with ScopedTiming("total",1):
        img = pl.get_frame()
        det_boxes, gesture_res = hkc.run(img)
        hkc.draw_result(pl, det_boxes, gesture_res)
        pl.show_image()
        gc.collect()
```

### 4.7 可调参数一览

| 参数 | 位置 | 影响 |
| --- | --- | --- |
| `guess_mode = 3` | `__main__` | 0 稳赢 / 1 必输 / >=2 公平多局制（值为总局数） |
| `time.sleep_ms(2000)` | `draw_result()` | 每局结果展示停顿时长 |
| `confidence_threshold=0.2` / `nms_threshold=0.5` | `__main__` | 手掌检测灵敏度；手势判错多时可微调 |
| `thr_angle` 等三个角度阈值 | `hk_gesture()` | 手势判别松紧，同实验11 |
| `draw_img_np[:400,:400,:]` | 贴图处 | 贴图显示区域（左上角 400x400），可自行改位置 |

## 5. 运行效果

- **屏幕画面**：
  - 开局无手/手势无效时显示"游戏开始 第一回合"提示；
  - 出拳被识别后，屏幕左上角 400x400 区域显示**开发板出的拳**（石头/剪刀/布贴图），中央显示"第N回合"；停顿约 2 秒；
  - 手移出画面后可出下一拳；三局打完显示"你赢了！/你输了！/平局"，约 2 秒后自动重开；
  - 两只手同时入镜显示"请保证只有一只手入镜！"。
- **串口打印**：每帧耗时；取消 `print(det_boxes, gesture_res)` 注释可看到手势名。
- **成功现象**：出 fist 被判石头、yeah 被判剪刀、five 被判布，比分与胜负结算正确。
- **失败现象排查**：
  - 贴图/模型加载失败 → 核对两个 kmodel 与 three 个 bin 贴图路径；
  - 出拳一直不被识别 → 手势要标准（fist 握紧、yeah 两指分开伸直、five 五指张开）、手背正对镜头、只伸一只手；
  - 出拳后没反应 → 本局已锁定，需要**把手移出画面再伸回来**才能出下一拳；
  - 回合数不增加 → 手势被判成了其他 6 种（gun/love/one/six/three/thumbUp）或 None，重新摆标准手势。

## 6. 代码使用说明

1. **硬件准备**：断电插好摄像头与 LCD 排线，插入 TF 卡，Type-C 上电。
2. **固件要求**：正点原子 **CanMV K230D 固件**（内置 libs、aicube、nncase_runtime 等），固件与 CanMV IDE 版本匹配。
3. **运行方式**：CanMV IDE 打开本目录 `main.py` 运行。**按文件头注释的官方玩法**：先让镜头前保持无手掌，然后将一只手移入镜头出石头/剪刀/布；每局结束后手移出画面再进入出下一拳。
4. **模型与资源文件需求**（全部从正点原子资料拷贝到 TF 卡）：
   - `/sdcard/examples/kmodel/hand_det.kmodel` —— 手掌检测模型；
   - `/sdcard/examples/kmodel/handkp_det.kmodel` —— 手掌关键点模型；
   - `/sdcard/examples/utils/five.bin`、`fist.bin`、`shear.bin` —— 布/石头/剪刀三张 400x400 ARGB 贴图（尺寸与 `read_file` 中 `reshape((400,400,4))` 绑定）；
   - `anchors_path` 变量定义了但**未实际使用**，`prior_data_320.bin` 非必需。
5. **依赖库**：`libs.PipeLine / libs.AIBase / libs.AI2D`（板载 `/sdcard/app/libs/`）、固件内置 `aicube`、`ulab.numpy`、`image`、`media.sensor`、`nncase_runtime`、`random`；不依赖本例程文件夹内其他文件。

## 7. 拓展思考

1. **入门级**：把 `guess_mode` 改成 5（五局三胜制）体验多局模式；再改成 0 和 1，验证"稳赢/必输"贴图逻辑，理解 mode 0/1 分支为什么不做随机。
2. **进阶级**：给游戏加声光反馈——玩家胜时点亮蓝色 LED（GPIO59）并让蜂鸣器短鸣，板胜时亮红灯（GPIO61）；在 `counts_guess == guess_mode` 结算分支里加 GPIO 控制即可（参考基础实验例程）。
3. **挑战级**：把 `time.sleep_ms(2000)` 的阻塞式停顿改成非阻塞计时（记录 `time.time_ns()`，每帧判断是否到时），停顿期间画面继续刷新；进一步用 KEY0 按键切换 `guess_mode`、KEY1 重置比分，做成带完整交互的街机小游戏；还可通过 UART 把每局结果发给 PC 记录战绩。

## 8. 常见问题

**Q1：kmodel 或贴图 bin 加载失败怎么办？**
A：确认 TF 卡上有 `/sdcard/examples/kmodel/hand_det.kmodel`、`handkp_det.kmodel` 以及 `/sdcard/examples/utils/five.bin`、`fist.bin`、`shear.bin`。bin 贴图必须是配套资料中的 400x400x4 字节文件（640000 字节），自己替换图片需先转成同尺寸 ARGB 裸数据。

**Q2：出了拳但屏幕没反应/不记分？**
A：最常见原因是**上一局还没解锁**：出拳被识别后 `set_stop_id=False`，必须把手完全移出画面（`len(dets)==0`）才会复位。其次检查手势是否被判为 fist/yeah/five 三者之一——判成 None 或其他手势不记分，中央提示会停留在"第N回合"。

**Q3：手势经常被误判（比如 five 判成 None）？**
A：几何规则对角度敏感：① 手背正对摄像头；② 五指动作做到位（布要五指完全伸直分开、石头要握紧）；③ 光照充足；④ 仍不行可微调 `hk_gesture` 中的阈值（如 thr_angle 从 65 降到 62）。详细调法见实验11 README。

**Q4：结算后为什么画面卡住 2 秒？**
A：正常设计。`sleep_end=True` 时下一帧执行 `time.sleep_ms(2000)` 让你看清胜负结果，之后自动清零重开。若觉得太长可改小该值（注意它是阻塞的，期间不刷新画面）。

**Q5：帧率与内存表现正常吗？**
A：每帧跑手掌检测 + 关键点两个模型（1024x768 送 AI），十几 FPS 属正常。贴图数组常驻内存（3x400x400x4≈1.9MB），请保留循环里的 `gc.collect()`；若出现 MemoryError，复位开发板后直接运行本脚本。
