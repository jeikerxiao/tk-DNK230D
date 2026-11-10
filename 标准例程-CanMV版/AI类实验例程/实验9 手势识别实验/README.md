# 实验9 手势识别实验

> 两级模型串联：先检测手掌位置，再把握掌区域裁剪出来送分类模型，识别出 gun（手枪手势）、yeah（剪刀手）、five（张开五指）或 other（其他），并在屏幕上标注结果。难度：中级。

## 1. 例程简介

本例程组合两个模型完成"看懂手势"：第一级 `hand_det.kmodel`（与实验8 相同的手掌检测模型）找出画面中所有手；第二级 `hand_reco.kmodel`（224x224 输入的**分类**模型）对每只手的裁剪区域做 4 分类，类别为 `["gun","other","yeah","five"]`。识别结果以文字+分数显示在手的检测框上方。你将学到：分类网络与检测网络的区别、softmax 概率归一化、crop（裁剪）预处理、ROI（感兴趣区域）两级推理架构。难度为**中级**，预期用途：手势控制设备、无接触交互、趣味演示。

## 2. 硬件连接与原理

用到的硬件：

- **CNK230DF 核心板（K230D）**：RISC-V 双核 + 6TOPS KPU，每帧可能连续跑 1+N 次模型（N 为手数）；
- **MIPI CSI 摄像头**：`Sensor(width=1280, height=960)`；送 AI 分辨率 `rgb888p_size=[1280,960]`；
- **2.4 寸 MIPI LCD**：`display_mode="lcd"`、`display_size=[640,480]`；
- **TF 卡（/sdcard）**：存放两个 kmodel 模型文件（锚框直接写在代码里）。

**两级识别原理（通俗版）**：直接对整幅画面做手势分类又慢又不准（背景干扰大）。所以程序分两步：第一步"手在哪"——检测模型给出每只手的框；第二步"手在做什么"——按框把这只手**裁剪（crop）**出来（代码里还向四周扩了 1.26 倍留余量），缩放到 224x224 送分类模型。分类模型输出的不是坐标而是一组分数（logits），经过 **softmax** 变成加起来等于 1 的概率，取概率最大的类别作为答案。

> [!WARNING]
> - 摄像头/LCD 的 MIPI 排线必须**断电插拔**，插反通电会烧坏器件；
> - 供电不足会导致双模型推理时死机重启，请使用足功率 Type-C 电源；
> - AI 推理发热明显，**属正常现象**；
> - 1280x960 的送 AI 分辨率数据量较大，帧率会低于低分辨率例程，请勿误以为死机。

## 3. 相关前置知识

- **分类模型 vs 检测模型**：检测回答"有什么、在哪里"（输出框）；分类只回答"这张图是哪一类"（输出各类别分数）。本实验第二级是分类模型，输入是一只手的裁剪图。
- **softmax**：把任意实数分数向量变成 0~1 且总和为 1 的概率向量。代码实现：

```python
def softmax(self, x):
    x -= np.max(x)                      # 减去最大值防止exp溢出
    x = np.exp(x) / np.sum(np.exp(x))
    return x
```

- **ROI（感兴趣区域）/ crop**：`ai2d.crop(x, y, w, h)` 让硬件直接抠出指定矩形区域，比软件拷贝快得多。
- **锚框 / 置信度 / IoU / NMS / strides**：第一级检测用，`confidence_threshold=0.2`、`nms_threshold=0.5`、`strides=[8,16,32]`、9 对手掌锚框写在代码里。
- **nms_option**：`True` 做类间 NMS、`False` 做类内 NMS（代码注释原文），本例用 False。
- **kmodel / KPU / 张量 / 推理**：两个模型均为 K230 专用 `.kmodel` 格式，由 KPU 加速。
- **pad 补边色 114**：第一级检测预处理补的是灰色 [114,114,114]（YOLOv5 惯用值），注意与实验5/7/8 的黑色补边不同，需与模型训练时一致。
- **帧率（FPS）**：`ScopedTiming("total",1)` 打印每帧耗时。

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

### 4.2 第一级：HandDetApp（手掌检测）

结构与实验8 的检测类基本相同，区别有二：

- pad 补边色为 `[114,114,114]`；
- `postprocess` 调 `aicube.anchorbasedet_post_process(...)` 时类别数直接写 `1`。

### 4.3 第二级：HandRecognitionApp（手势分类）

预处理——按检测框裁剪正方形 ROI：

```python
def config_preprocess(self, det, input_image_size=None):
    ai2d_input_size = input_image_size if input_image_size else self.rgb888p_size
    self.crop_params = self.get_crop_param(det)
    self.ai2d.crop(self.crop_params[0], self.crop_params[1], self.crop_params[2], self.crop_params[3])
    self.ai2d.resize(nn.interp_method.tf_bilinear, nn.interp_mode.half_pixel)
    self.ai2d.build([1,3,ai2d_input_size[1],ai2d_input_size[0]],[1,3,self.model_input_size[1],self.model_input_size[0]])
```

`get_crop_param()` 的要点：以检测框中心 `cx,cy` 为中心，取半长边 `length = max(w,h)/2` 的 **1.26 倍** 作为半边长（多留一圈背景，与分类模型训练时的裁剪习惯一致），并用 `max(0,...)`/`min(size-1,...)` 防止裁出画面外。

后处理——softmax + 取最大概率类别：

```python
def postprocess(self, results):
    result = results[0].reshape(results[0].shape[0]*results[0].shape[1])
    x_softmax = self.softmax(result)
    idx = np.argmax(x_softmax)
    text = " " + self.labels[idx] + ": " + str(round(x_softmax[idx],2))
    return text
```

输出直接拼成可显示的文字，如 `" yeah: 0.97"`。

### 4.4 调度类 HandRecognition

```python
def run(self, input_np):
    det_boxes = self.hand_det.run(input_np)      # 执行手掌检测
    hand_rec_res = []
    hand_det_res = []
    for det_box in det_boxes:
        x1, y1, x2, y2 = det_box[2], det_box[3], det_box[4], det_box[5]
        w, h = int(x2 - x1), int(y2 - y1)
        if (h < (0.1*self.rgb888p_size[1])):            # 丢弃不合理的框
            continue
        if (w < (0.25*self.rgb888p_size[0]) and ((x1 < (0.03*self.rgb888p_size[0])) or (x2 > (0.97*self.rgb888p_size[0])))):
            continue
        if (w < (0.15*self.rgb888p_size[0]) and ((x1 < (0.01*self.rgb888p_size[0])) or (x2 > (0.99*self.rgb888p_size[0])))):
            continue
        self.hand_rec.config_preprocess(det_box)        # 每个手掌重新配置crop
        text = self.hand_rec.run(input_np)              # 手势分类
        hand_det_res.append(det_box)
        hand_rec_res.append(text)
    return hand_det_res, hand_rec_res
```

三条 `if...continue` 与实验8 相同，是"太小/贴边"框过滤，避免对残缺的手做分类。

### 4.5 绘制

```python
pl.osd_img.draw_rectangle(x_det, y_det, w_det, h_det, color=(255, 0, 255, 0), thickness=2)
pl.osd_img.draw_string_advanced(x_det, y_det-50, 32, hand_rec_res[k], color=(255,0,255,0))
```

紫框 + 框上方的手势名与概率。

### 4.6 主流程

```python
hand_det_kmodel_path = "/sdcard/examples/kmodel/hand_det.kmodel"
hand_rec_kmodel_path = "/sdcard/examples/kmodel/hand_reco.kmodel"
anchors_path = "/sdcard/examples/utils/prior_data_320.bin"   # 本实验定义了该变量但实际未使用，锚框直接写在下方
rgb888p_size = [1280, 960]
hand_det_input_size = [512, 512]
hand_rec_input_size = [224, 224]
confidence_threshold = 0.2
nms_threshold = 0.5
labels = ["gun", "other", "yeah", "five"]
anchors = [26,27, 53,52, 75,71, 80,99, 106,82, 99,134, 140,113, 161,172, 245,276]

sensor = Sensor(width=1280, height=960)
pl = PipeLine(rgb888p_size=rgb888p_size, display_size=display_size, display_mode=display_mode)
pl.create(sensor=sensor)
hr = HandRecognition(...)
while True:
    os.exitpoint()
    with ScopedTiming("total",1):
        img = pl.get_frame()
        hand_det_res, hand_rec_res = hr.run(img)
        hr.draw_result(pl, hand_det_res, hand_rec_res)
        pl.show_image()
        gc.collect()
```

> [!NOTE]
> `labels` 的顺序 `["gun","other","yeah","five"]` 对应模型输出的类别 id 0~3，**不可调换**。gun=拇指+食指比枪、yeah=剪刀手（V 字）、five=五指张开，其余手势都会归入 other。

### 4.7 可调参数一览

| 参数 | 位置 | 影响 |
| --- | --- | --- |
| `confidence_threshold=0.2` / `nms_threshold=0.5` | `__main__` | 第一级检手灵敏度 |
| `ratio_num = 1.26*length` | `HandRecognitionApp.get_crop_param()` | 裁剪余量；改小裁剪更紧，但可能与训练不匹配导致识别率下降 |
| `rgb888p_size=[1280,960]` | `__main__` | 送 AI 分辨率，降低可提帧率 |
| `labels` | `__main__` | 类别显示名，与模型输出 id 绑定 |
| `hand_rec_input_size=[224,224]` | `__main__` | 与分类模型绑定，不可改 |

## 5. 运行效果

- **屏幕画面**：每只手被紫框框住，框上方显示如 ` yeah: 0.97`、` gun: 0.93`、` five: 0.99` 或 ` other: 0.88` 的文字；切换手势后 1 秒内文字更新。
- **串口打印**：每帧耗时；取消 `print(hand_det_res, hand_rec_res)` 注释可看到框数据和识别文字。
- **成功现象**：比出剪刀手稳定显示 yeah，张开五指显示 five，比枪显示 gun，其余显示 other，分数普遍 0.9 以上。
- **失败现象排查**：
  - 模型加载失败 → 核对两个 kmodel 路径；
  - 手势识别经常错 → 手离摄像头太近/太远导致裁剪不佳、光线太暗、手势不标准（如剪刀手两指没分开）；
  - 显示 other 但你比的是标准手势 → 角度问题，手背正对摄像头效果最好；
  - 检不到手 → 参考实验8 的排查方法（背景对比度、光照、阈值）。

## 6. 代码使用说明

1. **硬件准备**：断电插好摄像头与 LCD 排线，插入 TF 卡，Type-C 上电。
2. **固件要求**：正点原子 **CanMV K230D 固件**（内置 libs、aicube、nncase_runtime 等），固件与 CanMV IDE 版本匹配。
3. **运行方式**：CanMV IDE 连接开发板 → 打开本目录 `main.py` → 运行，手距离摄像头 20~50cm 比手势。
4. **模型文件需求**（从正点原子资料拷贝到 TF 卡）：
   - `/sdcard/examples/kmodel/hand_det.kmodel` —— 手掌检测模型；
   - `/sdcard/examples/kmodel/hand_reco.kmodel` —— 手势分类模型；
   - 代码中虽定义了 `anchors_path="/sdcard/examples/utils/prior_data_320.bin"`，但**本实验并未加载该文件**（锚框数组直接写在代码中），无需拷贝也可运行。
5. **依赖库**：`libs.PipeLine / libs.AIBase / libs.AI2D`（板载 `/sdcard/app/libs/`）、固件内置 `aicube`、`ulab.numpy`、`image`、`media.sensor`、`nncase_runtime`；不依赖本例程文件夹内其他文件。

## 7. 拓展思考

1. **入门级**：把识别结果联动 LED——yeah 亮蓝灯（GPIO59）、gun 亮红灯（GPIO61）、five 双灯齐亮；在 `run` 返回后根据 `hand_rec_res` 中的类别字符串写 GPIO 即可（参考基础实验例程）。
2. **进阶级**：做"手势遥控器"——识别到 five 时通过 UART 发送字符 'A'，yeah 发送 'B'，gun 发送 'C'，用另一个单片机/PC 接收执行不同动作。
3. **挑战级**：串联实验10（手掌关键点）：分类结果 + 关键点位置结合，比如"yeah 手势且食指尖位于屏幕左上区域"才触发操作，减少误触发；或参考实验14（猜拳游戏）把本实验的静态手势识别扩展成完整互动游戏。

## 8. 常见问题

**Q1：kmodel 加载失败怎么办？**
A：确认 `/sdcard/examples/kmodel/` 下同时有 `hand_det.kmodel` 和 `hand_reco.kmodel`，文件名大小写与代码一致，且来自正点原子配套资料。

**Q2：识别结果跳变、一会儿 yeah 一会儿 other？**
A：分类模型对裁剪区域敏感：① 保持手稳定、手背正对摄像头；② 光照均匀避免阴影；③ 手势要做"标准"（剪刀手两指分开、five 五指伸直）；④ 应用层可加"连续 3 帧同一结果才输出"的投票逻辑消抖。

**Q3：为什么只有 3 种手势能识别，其他都显示 other？**
A：模型训练时只定义了 gun/yeah/five 三个有效手势类，`other` 是"兜底类"（文件头注释写明"支持3种不同手势进行分类"）。想识别更多手势需要用更多类别的数据重新训练模型并转换 kmodel。

**Q4：帧率是否正常？双手入镜会更慢吗？**
A：正常。每帧 = 1 次检测 + 每只手 1 次分类，双手时分类耗时翻倍，帧率下降属预期。可把 `rgb888p_size` 降到 [640,480] 提速，检测精度略降。

**Q5：检测框有但分类文字是乱码/空白？**
A：文字由 `postprocess` 拼接（`labels[idx]` + 分数），若 `labels` 被改动过少或顺序变化，`idx` 可能越界。保持 `labels=["gun","other","yeah","five"]` 原样；若自行训练新模型，labels 必须与新模型的类别顺序严格对应。
