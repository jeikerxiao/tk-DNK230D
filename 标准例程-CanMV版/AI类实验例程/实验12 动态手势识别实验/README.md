# 实验12 动态手势识别实验

> 综合三个模型 + 一个状态机：先用静态手势（five/yeah 指向上下左右）"下指令"，再用手势时序模型识别挥手动作，最后在屏幕上显示"向上/向下/向左/向右/中间"的箭头与文字。难度：高级。

## 1. 例程简介

本例程是手势类实验的集大成者，串联**三个模型**：① `hand_det.kmodel` 手掌检测；② `handkp_det.kmodel` 手掌 21 关键点（+实验11 的几何手势规则）；③ `gesture.kmodel` **动态手势识别模型**（带记忆结构的时序模型，识别挥手等连续动作）。程序用一个状态机管理交互流程：待机状态（TRIGGER）下检测你比出的静态手势方向（five/yeah 手势的指尖朝向决定上/下/左/右），进入对应方向状态后启动动态手势模型，识别到"向该方向挥手"或"五指捏合"即在屏幕上画出箭头和文字。你将学到：多模型调度、时序模型的多输入张量与状态传递、滑动平均 logits、状态机交互设计。难度为**高级**，预期用途：隔空翻页、无接触手势遥控、智能座舱交互原型。

## 2. 硬件连接与原理

用到的硬件：

- **CNK230DF 核心板（K230D）**：RISC-V 双核 + 6TOPS KPU，同时驻留三个模型，对内存和算力要求最高；
- **MIPI CSI 摄像头**：`Sensor(width=1280, height=960)`；送 AI 分辨率 `rgb888p_size=[1024,768]`；
- **2.4 寸 MIPI LCD**：`display_mode="lcd"`、`display_size=[640,480]`；
- **TF 卡（/sdcard）**：存放三个 kmodel 模型和四张方向指示贴图（shang/xia/zuo/you.bin，ARGB 原始图像数据）。

**动态手势识别原理（通俗版）**：挥手是一个**过程**，单帧图片看不出来。`gesture.kmodel` 是一个带"记忆"的网络（类似 RNN/3D 卷积结构）：它有 11 个输入张量——第 1 个是当前帧 224x224 的图像，后 10 个是**上一帧推理时产生的中间特征**（形状如 [1,3,56,56]、[1,8,14,14] 等，见代码 `gesture_kmodel_input_shape`）。每帧推理完，程序把输出的中间特征回填为下一帧的输入，网络的"记忆"就这样一帧帧传递下去，从而感知运动方向。模型输出 27 类动作分数，程序关注其中几类：向上/下/左/右挥、五指捏合等。

**为什么用滑动平均**：单帧分数噪声大，代码把最近若干帧的输出 logits 累加平均（`his_logit` 列表），平均分数超过阈值（如 0.7 且至少 2 帧，或 0.3 且至少 4 帧）才认定动作成立，避免误触发。

> [!WARNING]
> - 摄像头/LCD 的 MIPI 排线必须**断电插拔**，插反通电会烧坏器件；
> - 三个模型 + 时序缓存对内存压力大，**供电不足极易重启**，请务必使用足功率电源；
> - 本例程 KPU 和 CPU 都高负载，**发热明显强于其他实验，属正常现象**；
> - 运行中不要拔 TF 卡（贴图与模型都从卡上读取）。

## 3. 相关前置知识

- **静态手势 vs 动态手势**：静态看一帧的形状（实验9/11），动态看连续多帧的运动（本实验）。
- **时序模型 / 多输入张量**：模型除了图像输入，还有"上一帧的内部状态"输入。代码中 `self.input_tensors` 列表保存全部 11 个输入张量；`run()` 里推理完把 `outputs[1:]` 回填给 `input_tensors[1:]`，实现帧间状态传递。
- **logits 与滑动平均**：logits 是模型输出的原始类别分数。`his_logit.append(results[0])` 后 `avg_logit = sum(...)` 累加多帧，分数越高、帧数越多越可信。
- **状态机**：程序用 `cur_state` 在 TRIGGER（待机）、UP、DOWN、LEFT、RIGHT 几个状态间切换；`pre_state`/`draw_state` 记录历史与显示状态。每个状态只接受特定输入，超时 2 秒自动回待机。
- **几何手势规则（hk_gesture）**：与实验11 完全相同的 9 种静态手势角度判别（fist/five/gun/love/one/six/three/thumbUp/yeah），本实验只用到 five 和 yeah。
- **指尖朝向角**：待机状态下取"手腕→小指指尖方向点"的向量（`hk_results[24]-hk_results[0]` 等），计算它与水平方向的夹角，把 360° 分成四段：70~110° 为上、110~225° 为右、225~315° 为下、其余为左。
- **均值/方差归一化**：动态手势模型预处理要把像素除以 255 再减均值除方差：`mean=[0.485,0.456,0.406]`、`std=[0.229,0.224,0.225]`（ImageNet 标准值），代码在 `preprocess()` 中完成。
- **两个 Ai2d 对象**：Ai2d 硬件预处理有固定顺序 crop→shift→resize/affine→pad；本模型需要"先 resize 到 256 再中心 crop 到 224"，顺序不符合单对象限制，所以代码建了 `ai2d_resize` 和 `ai2d_crop` 两个对象分两步做。
- **锚框 / 置信度 / NMS / crop**：前两级同实验8~11。
- **ARGB 贴图 bin 文件**：`shang.bin` 等是把 ARGB8888 图像逐像素存成的裸数据文件，用 `np.fromfile` 读入后 `reshape((高,宽,4))`，直接赋值给画布数组的左上角区域显示方向图标。

## 4. 软件代码逻辑说明

### 4.1 框架导入

```python
from libs.PipeLine import PipeLine, ScopedTiming
from libs.AIBase import AIBase
from libs.AI2D import Ai2d
import nncase_runtime as nn
import ulab.numpy as np
import aicube
```

### 4.2 前两级：HandDetApp 与 HandKPClassApp

与实验10/11 完全同构：512x512 检测（pad 114 灰边）+ 256x256 关键点（crop 1.26 倍），`hk_gesture()` 规则判 9 种静态手势，本实验用到其中的 **five** 和 **yeah**。

### 4.3 第三级：DynamicGestureApp（时序模型）

构造时定义 11 个输入张量的形状与预处理参数：

```python
self.gesture_kmodel_input_shape = [[1, 3, 224, 224],   # 当前帧图像
                                   [1,3,56,56], [1,4,28,28], [1,4,28,28],
                                   [1,8,14,14], [1,8,14,14], [1,8,14,14],
                                   [1,12,14,14], [1,12,14,14],
                                   [1,20,7,7], [1,20,7,7]]   # 后10个是上一帧的中间特征
self.resize_shape = 256
self.mean_values = np.array([0.485, 0.456, 0.406]).reshape((3,1,1))
self.std_values = np.array([0.229, 0.224, 0.225]).reshape((3,1,1))
```

重写的 `preprocess()`（先 resize 后 crop，再归一化）：

```python
def preprocess(self, input_np):
    resize_tensor = self.ai2d_resize.run(input_np)
    crop_output_tensor = self.ai2d_crop.run(resize_tensor.to_numpy())
    ai2d_output = crop_output_tensor.to_numpy()
    self.first_data[0] = ai2d_output[0].copy()
    self.first_data[0] = (self.first_data[0]*1.0/255 - self.mean_values)/self.std_values
    self.input_tensors[0] = nn.from_numpy(self.first_data)
```

重写的 `run()` 实现状态传递：

```python
def run(self, input_np, his_logit, history):
    self.preprocess(input_np)
    outputs = self.inference(self.input_tensors)
    for i in range(1, self.get_kmodel_outputs_num()):
        self.input_tensors[i] = nn.from_numpy(outputs[i])   # 输出回填为下一帧输入
    return self.postprocess(outputs, his_logit, history)
```

`postprocess()` 累加平均 logits、取最大类别，并经 `gesture_process_output()` 结合 `history`（最近 20 帧结果）做**防跳变过滤**：一批"不可信类别 id"（如 0/3/4/6/7/8/9/14/19~24）会被替换为上一帧的结果；且要求连续两帧一致才接受新类别。

### 4.4 状态机调度：DynamicGesture.run()

```python
def run(self, input_np):
    if self.cur_state == self.TRIGGER:
        det_boxes = self.hand_det.run(input_np)      # 待机：手掌检测
        for det_box in det_boxes:
            ...  # 边缘框过滤（三条规则同实验8）
            self.hand_kp.config_preprocess(det_box)
            hk_results, gesture_str = self.hand_kp.run(input_np)   # 关键点+静态手势
        return boxes, gesture_res
    else:
        idx, avg_logit = self.dg.run(input_np, self.his_logit, self.history)  # 激活：动态手势
        return idx, avg_logit
```

### 4.5 draw_result()：方向判定 + 结果确认 + 绘制

**待机状态**：若静态手势是 five 或 yeah，计算手腕到指尖的朝向角并划分四方向，满足稳定条件后贴方向图标（如 `draw_img_np[:self.bin_height,:self.bin_width,:] = self.shang_argb`）并把 `cur_state` 切到 UP/RIGHT/DOWN/LEFT，同时记录 `m_start` 时刻。

**方向状态**（以 UP 为例）：

```python
if ((idx==15) or (idx==10)):            # 模型判为向某方向挥动的类别id
    self.vec_flag.clear()
    if (((avg_logit[idx] >= 0.7) and (len(self.his_logit) >= 2)) or ((avg_logit[idx] >= 0.3) and (len(self.his_logit) >= 4))):
        self.s_start = time.time_ns()
        self.cur_state = self.TRIGGER   # 确认成功，回待机
        self.draw_state = self.DOWN     # 记录要显示的结果
        self.history = [2]
    self.pre_state = self.UP
else:
    self.his_logit.clear()              # 类别不符则清空滑动窗口
```

另有超时保护：方向状态停留超过 2000ms 未确认则自动回 TRIGGER。

**结果展示**（1 秒内）：

```python
if (self.draw_state == self.UP):
    draw_img.draw_arrow(1068,330,1068,130, (255,170,190,230), thickness=13)
    draw_img.draw_string_advanced(self.display_size[0]//2-50, self.display_size[1]//2-50, 32, "向上")
elif (self.draw_state == self.MIDDLE):
    draw_img.draw_circle(320,240,100, (255,170,190,230), thickness=2, fill=True)
    draw_img.draw_string_advanced(..., 32, "中间")
```

向上/右/下/左分别画对应箭头 + 中文文字，"五指捏合"画屏幕中央实心圆 + "中间"。

> [!NOTE]
> 箭头坐标（如 1068,330 → 1068,130）是按更大显示分辨率写死的，在 640x480 的 LCD 上箭头可能画在可视区外，**以中央的中文文字为准**判断识别结果；文字坐标用 `display_size` 计算，任何分辨率都在屏幕中央附近。

### 4.6 主流程

```python
hand_det_kmodel_path = "/sdcard/examples/kmodel/hand_det.kmodel"
hand_kp_kmodel_path = "/sdcard/examples/kmodel/handkp_det.kmodel"
gesture_kmodel_path = "/sdcard/examples/kmodel/gesture.kmodel"
rgb888p_size = [1024, 768]
hand_det_input_size = [512, 512]
hand_kp_input_size = [256, 256]
gesture_input_size = [224, 224]
confidence_threshold = 0.2
nms_threshold = 0.5
labels = ["hand"]
anchors = [26,27, 53,52, 75,71, 80,99, 106,82, 99,134, 140,113, 161,172, 245,276]

sensor = Sensor(width=1280, height=960)
pl = PipeLine(rgb888p_size=rgb888p_size, display_size=display_size, display_mode=display_mode)
pl.create(sensor=sensor)
dg = DynamicGesture(hand_det_kmodel_path, hand_kp_kmodel_path, gesture_kmodel_path, ...)
while True:
    os.exitpoint()
    with ScopedTiming("total",1):
        img = pl.get_frame()
        output1, output2 = dg.run(img)
        dg.draw_result(pl, output1, output2)
        pl.show_image()
        gc.collect()
```

### 4.7 可调参数一览

| 参数 | 位置 | 影响 |
| --- | --- | --- |
| 触发手势 `gesture == "five"` / `"yeah"` | `draw_result()` | 可改成其他静态手势作为"激活暗号" |
| 方向角度分段 70/110/225/315 | `draw_result()` | 调整四个方向的判定区间 |
| `avg_logit[idx] >= 0.7 / 0.3` 及帧数条件 | 各方向分支 | 确认灵敏度：调低更易触发但误报多 |
| `elapsed_time > 2000` | `draw_result()` | 方向状态超时时间（毫秒） |
| `elapsed_ms_show < 1000` | `draw_result()` | 结果文字显示时长（毫秒） |
| `max_hist_len = 20` | `DynamicGestureApp` / `DynamicGesture` | 历史窗口长度，影响防跳变强度 |

## 5. 运行效果

- **屏幕画面**：
  1. 待机：无图标；比出 five 或 yeah 手势并让指尖朝上/下/左/右，屏幕**左上角**出现对应方向贴图（shang/xia/zuo/you），进入该方向的"待确认"状态；
  2. 在该状态下向对应方向挥手，屏幕中央显示"向上/向下/向左/向右"文字（并画箭头）；五指做捏合动作则显示实心圆 + "中间"；结果显示约 1 秒后消失，回到待机；
  3. 方向状态 2 秒内没完成动作则自动退回待机。
- **串口打印**：每帧耗时；取消 `print(output1, output2)` 注释可看到当前类别 id 和累计 logits。
- **成功现象**：指尖朝上比 five → 左上角出现"上"贴图 → 向上挥手 → 中央显示"向上"。整套流程 1~2 秒内完成。
- **失败现象排查**：
  - 贴图/模型加载失败 → 核对三个 kmodel 和四个 bin 贴图路径；
  - 进不了方向状态 → 静态手势必须是 five 或 yeah 且指尖朝向明显（角度落在分段边界会不稳定）；
  - 挥手不触发 → 挥动幅度要大、速度适中；确认阈值可调低；
  - 类别乱跳 → 正常会被 `gesture_process_output` 的 history 过滤，若仍乱跳说明光线/背景变化剧烈。

## 6. 代码使用说明

1. **硬件准备**：断电插好摄像头与 LCD 排线，插入 TF 卡，**足功率** Type-C 上电。
2. **固件要求**：正点原子 **CanMV K230D 固件**（内置 libs、aicube、nncase_runtime 等），固件与 CanMV IDE 版本匹配；三模型常驻内存较大，建议复位后直接运行本脚本，不要与其他大内存脚本连续切换。
3. **运行方式**：CanMV IDE 连接开发板 → 打开本目录 `main.py` → 运行 → 按"比手势定向 → 挥手确认"的两步流程操作。
4. **模型与资源文件需求**（全部从正点原子资料拷贝到 TF 卡）：
   - `/sdcard/examples/kmodel/hand_det.kmodel` —— 手掌检测模型；
   - `/sdcard/examples/kmodel/handkp_det.kmodel` —— 手掌关键点模型；
   - `/sdcard/examples/kmodel/gesture.kmodel` —— 动态手势（时序）模型；
   - `/sdcard/examples/utils/shang.bin`、`xia.bin`、`zuo.bin`、`you.bin` —— 四方向 ARGB 贴图（尺寸分别为 150x216 或 216x150，代码中按 `bin_width=150`、`bin_height=216` reshape）。
5. **依赖库**：`libs.PipeLine / libs.AIBase / libs.AI2D`（板载 `/sdcard/app/libs/`）、固件内置 `aicube`、`ulab.numpy`、`image`、`media.sensor`、`nncase_runtime`；不依赖本例程文件夹内其他文件。

## 7. 拓展思考

1. **入门级**：把四个方向的结果文字换成英文或加上图标颜色变化；把结果显示时长从 1000ms 改成 2000ms，观察交互体验差异。
2. **进阶级**：做**隔空翻页器**——识别到"向左/向右"后通过 UART 发送键盘翻页指令字符给 PC，或用板载 LED/蜂鸣器给出不同反馈（左转蓝灯短鸣、右转红灯短鸣），实现手势遥控。
3. **挑战级**：简化交互——跳过"静态手势定向"步骤，直接让动态手势模型全时运行并显示其 27 类中置信度最高的动作名，对比两种交互逻辑的稳定性与实时性；或把 `his_logit` 的滑动平均换成带遗忘因子的指数平滑（`avg = 0.8*avg + 0.2*new`），调参对比触发准确率。

## 8. 常见问题

**Q1：运行报内存错误（MemoryError）或频繁重启？**
A：本实验同时加载三个模型 + 11 个时序输入张量 + 贴图数据，内存占用是所有例程中最大的一档。对策：① 确认供电充足（弱供电下大内存分配最容易触发重启）；② 复位开发板后直接运行本脚本，避免多次运行残留内存；③ 保持主循环里的 `gc.collect()`。

**Q2：比了 five 手势但左上角不出方向贴图？**
A：检查三点：① 静态手势必须被 `hk_gesture` 判为 five 或 yeah（手指伸直到位、手背正对镜头）；② 指尖朝向要明显落在某个 90° 分段内，斜 45° 方向会落边界；③ 手掌检测框要先通过"太小/贴边"过滤——手别贴屏幕边缘。

**Q3：挥手了却不显示结果？**
A：① 挥动方向要与当前状态一致（UP 状态只认 idx 15/10 这两类输出）；② 挥动要干脆，慢速移动分数上不去；③ 确认条件较严格（平均分 ≥0.7 且 ≥2 帧，或 ≥0.3 且 ≥4 帧），调试时可先调低到 0.3/2 帧观察；④ 注意 2 秒超时，进入方向状态后要尽快挥手。

**Q4：帧率明显低于其他实验，正常吗？**
A：正常。三模型串联 + 时序状态传递 + 1024x768 送 AI 分辨率，计算量最大。待机阶段只跑前两级；进入方向状态后每帧还要跑 gesture.kmodel。若需提速可把 `rgb888p_size` 降到 [640,480]。

**Q5：kmodel 或 bin 文件加载失败？**
A：确认 TF 卡上 7 个文件齐全：`/sdcard/examples/kmodel/` 下的 `hand_det.kmodel`、`handkp_det.kmodel`、`gesture.kmodel`，以及 `/sdcard/examples/utils/` 下的 `shang.bin`、`xia.bin`、`zuo.bin`、`you.bin`。bin 贴图必须来自正点原子配套资料（其像素尺寸与代码中 150/216 的 reshape 参数绑定，换成别的尺寸会直接报错）。
