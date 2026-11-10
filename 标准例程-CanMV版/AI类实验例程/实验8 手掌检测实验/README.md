# 实验8 手掌检测实验

> 用手掌检测模型实时框出画面中的手，是后续手势识别、手掌关键点等实验（实验9~14）的第一步基础。难度：初级。

## 1. 例程简介

本例程运行 `hand_det.kmodel`（YOLOv5 结构、单类别 hand、输入 512x512），把摄像头画面中的手掌用紫色矩形框标出，并在框上方显示 `hand` 和置信度。代码结构与实验5（人体检测）几乎一致——同样的 anchor-based YOLO 流程、同样的 `aicube.anchorbasedet_post_process` 后处理，只是模型、锚框和分辨率不同，很适合作为"看懂一个、迁移一片"的练习。难度为**初级**，预期用途：手势交互入口、挥手感应开关、无接触操作设备。

## 2. 硬件连接与原理

用到的硬件：

- **CNK230DF 核心板（K230D）**：RISC-V 双核 + 6TOPS KPU；
- **MIPI CSI 摄像头**：`Sensor(width=1280, height=960)`；送 AI 分辨率 `rgb888p_size=[640,360]`；
- **2.4 寸 MIPI LCD**：`display_mode="lcd"`、`display_size=[640,480]`；
- **TF 卡（/sdcard）**：存放模型文件（锚框写在代码里，无需 bin 文件）。

**检测原理（通俗版）**：与实验5 相同，YOLO 网络对 512x512 输入图输出三个尺度的候选框（`strides=[8,16,32]`），每个候选框带"是手"的概率；后处理过滤低分框、NMS 去重后得到最终手掌框。本实验的锚框是针对"手"的形状统计出来的，比人体锚框更方正：

```python
anchors = [26,27, 53,52, 75,71, 80,99, 106,82, 99,134, 140,113, 161,172, 245,276]
```

> [!WARNING]
> - 摄像头/LCD 的 MIPI 排线必须**断电插拔**，插反通电会烧坏器件；
> - 请用足功率 Type-C 电源，供电不足会导致推理时死机重启；
> - AI 推理时核心板发热明显，**属正常现象**。

## 3. 相关前置知识

- **目标检测**：让程序回答"图里有什么、在哪里"，输出 = 类别 + 置信度 + 框坐标。
- **kmodel / KPU / 张量 / 推理**：`.kmodel` 是 K230 专用模型格式（本模型输入 512x512）；KPU 是 6TOPS 的 AI 加速单元；张量即多维数组；推理即"喂图出结果"。
- **锚框（anchor）**：预设的候选框标准尺寸，3 个尺度 x 3 种形状共 9 对宽高的数据。
- **置信度与阈值**：`confidence_threshold = 0.2`，低于它的候选框被丢弃。
- **IoU 与 NMS**：重叠度超过 `nms_threshold = 0.5` 的低分重复框被删除。
- **strides**：特征图相对输入图的缩小倍数 `[8,16,32]`，后处理据此把格子坐标还原成像素坐标。
- **pad + resize 预处理**：Ai2d 硬件等比缩放并补黑边到 512x512，防止手被拉伸变形。
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

框架同实验5：`PipeLine` 管采集显示、`AIBase` 管 KPU 推理、`Ai2d` 管硬件预处理、`aicube` 管通用 YOLO 后处理。

### 4.2 自定义类 HandDetectionApp

```python
class HandDetectionApp(AIBase):
    def __init__(self, kmodel_path, model_input_size, labels, anchors, confidence_threshold=0.2,
                 nms_threshold=0.5, nms_option=False, strides=[8,16,32], ...):
        ...
        self.ai2d = Ai2d(debug_mode)
        self.ai2d.set_ai2d_dtype(nn.ai2d_format.NCHW_FMT, nn.ai2d_format.NCHW_FMT, np.uint8, np.uint8)
```

参数与实验5 的 `PersonDetectionApp` 一一对应：`labels=["hand"]`、锚框列表、双阈值、三尺度步长。

### 4.3 预处理

```python
def config_preprocess(self, input_image_size=None):
    ai2d_input_size = input_image_size if input_image_size else self.rgb888p_size
    top, bottom, left, right = self.get_padding_param()
    self.ai2d.pad([0, 0, 0, 0, top, bottom, left, right], 0, [0, 0, 0])
    self.ai2d.resize(nn.interp_method.tf_bilinear, nn.interp_mode.half_pixel)
    self.ai2d.build([1,3,ai2d_input_size[1],ai2d_input_size[0]],[1,3,self.model_input_size[1],self.model_input_size[0]])
```

`get_padding_param()` 计算上下左右的补边量（本例 `right = int(round(dw + 0.1))`，与实验5 的 `dw - 0.1` 略有差别，是凑整像素的小技巧），图像等比缩放居中、四周补黑。

### 4.4 后处理

```python
def postprocess(self, results):
    dets = aicube.anchorbasedet_post_process(results[0], results[1], results[2],
        self.model_input_size, self.rgb888p_size, self.strides, len(self.labels),
        self.confidence_threshold, self.nms_threshold, self.anchors, self.nms_option)
    return dets
```

每个返回的 `det_box` 为 `[类别id, 置信度, x1, y1, x2, y2]`。

### 4.5 绘制（含边缘框过滤）

```python
x1, y1, x2, y2 = det_box[2], det_box[3], det_box[4], det_box[5]
...
# 过滤掉太小或者位置不合理的检测框
if (h < (0.1 * self.display_size[0])):
    continue
if (w < (0.25 * self.display_size[0]) and ((x1 < (0.03 * self.display_size[0])) or (x2 > (0.97 * self.display_size[0])))):
    continue
if (w < (0.15 * self.display_size[0]) and ((x1 < (0.01 * self.display_size[0])) or (x2 > (0.99 * self.display_size[0])))):
    continue
pl.osd_img.draw_rectangle(x1, y1, int(w), int(h), color=(255, 0, 255, 0), thickness=2)
pl.osd_img.draw_string_advanced(x1, y1-50, 32, " " + self.labels[det_box[0]] + " " + str(round(det_box[1], 2)), color=(255, 0, 255, 0))
```

与实验5 相同的三条边缘过滤规则（太小的框、又窄又贴边的框不画），框为紫色，文字形如 `hand 0.93`。

### 4.6 主流程

```python
kmodel_path = "/sdcard/examples/kmodel/hand_det.kmodel"
confidence_threshold = 0.2
nms_threshold = 0.5
rgb888p_size = [640, 360]
labels = ["hand"]

sensor = Sensor(width=1280, height=960)
pl = PipeLine(rgb888p_size=rgb888p_size, display_size=display_size, display_mode=display_mode)
pl.create(sensor=sensor)
hand_det = HandDetectionApp(kmodel_path, model_input_size=[512,512], labels=labels, anchors=anchors,
    confidence_threshold=confidence_threshold, nms_threshold=nms_threshold, nms_option=False,
    strides=[8,16,32], rgb888p_size=rgb888p_size, display_size=display_size, debug_mode=0)
hand_det.config_preprocess()
while True:
    os.exitpoint()
    with ScopedTiming("total",1):
        img = pl.get_frame()
        res = hand_det.run(img)
        hand_det.draw_result(pl, res)
        pl.show_image()
        gc.collect()
```

### 4.7 可调参数一览

| 参数 | 位置 | 影响 |
| --- | --- | --- |
| `confidence_threshold = 0.2` | `__main__` | 调低更灵敏（可能把手部局部误检成整只手），调高更严格 |
| `nms_threshold = 0.5` | `__main__` | 去重强度，双手交叉时调太低可能只剩一个框 |
| `rgb888p_size=[640,360]` | `__main__` | 送 AI 分辨率，加大可改善远处小手的检出，帧率下降 |
| `model_input_size=[512,512]` | 构造处 | 与 hand_det.kmodel 绑定，不可改 |
| 边缘过滤三个 `if` | `draw_result()` | 贴边的小手不出框时可放宽 |

## 5. 运行效果

- **屏幕画面**：手伸入画面后被紫色框实时框住，框上方显示 `hand 0.9x`；左右手同时入镜时各有一个框；手快速移动时框跟随。
- **串口打印**：每帧耗时；取消 `print(res)` 注释可看到检测框原始数据。
- **成功现象**：距离摄像头 20~60cm 的手掌稳定被框住，置信度普遍 0.8 以上。
- **失败现象排查**：
  - 模型加载失败 → 检查 `/sdcard/examples/kmodel/hand_det.kmodel`；
  - 检不到手 → 手太小太远、与背景颜色过于接近、光线太暗；把手掌张开、正对摄像头、靠近些；
  - 只框住手指/手腕局部 → 置信度阈值调整，或手没有完整入镜；
  - 贴画面边缘的手不显示 → 被边缘过滤规则跳过，属设计行为。

## 6. 代码使用说明

1. **硬件准备**：断电插好摄像头与 LCD 排线，插入 TF 卡，Type-C 上电。
2. **固件要求**：正点原子 **CanMV K230D 固件**（内置 libs、aicube、nncase_runtime 等），固件与 CanMV IDE 版本匹配。
3. **运行方式**：CanMV IDE 连接开发板 → 打开本目录 `main.py` → 运行。
4. **模型文件需求**（从正点原子资料拷贝到 TF 卡）：
   - `/sdcard/examples/kmodel/hand_det.kmodel` —— 手掌检测模型（本实验唯一模型文件，锚框已写在代码里）。
5. **依赖库**：`libs.PipeLine / libs.AIBase / libs.AI2D`（板载 `/sdcard/app/libs/`）、固件内置 `aicube`、`ulab.numpy`、`image`、`media.sensor`、`nncase_runtime`；不依赖本例程文件夹内其他文件。

## 7. 拓展思考

1. **入门级**：把检测框中心的坐标计算出来（`(x1+x2)/2, (y1+y2)/2`），打印或显示在屏幕上，观察手掌移动时中心点的轨迹——这是所有手势交互的"鼠标指针"。
2. **进阶级**：做一个"挥手开关"——手掌中心从左到右快速划过画面时点亮蓝色 LED（GPIO59），反向划过熄灭；只需在循环里记录连续几帧的中心点 x 坐标并判断位移方向。
3. **挑战级**：以本实验为第一级，串联实验9（手势识别）或实验10（手掌关键点）：先用手掌框裁剪 ROI，再送入第二个模型——体会"检测 + 精细分析"两级架构（参考实验2 的两级写法），这也是实验12/14 综合项目的骨架。

## 8. 常见问题

**Q1：kmodel 加载失败怎么办？**
A：确认 TF 卡 `/sdcard/examples/kmodel/hand_det.kmodel` 存在、文件名大小写一致、来自正点原子配套资料；TF 卡需正确挂载为 `/sdcard`（IDE 文件管理里能看到即正常）。

**Q2：检测不到手或时有时无，怎么调？**
A：手掌检测受外观影响大：① 手与背景对比度要高（深色背景 + 亮手掌效果最好）；② 光照充足，避免逆光；③ 手掌张开比握拳更容易被检出；④ 把 `confidence_threshold` 降到 0.15 观察分数；⑤ 手在画面中至少占 30x30 像素以上，太远检不到。

**Q3：两只手只框出一只？**
A：两手重叠或交叉时 NMS 可能把邻近框当作重复框删掉，试着把 `nms_threshold` 调高到 0.6；另外确认两只手都完整入镜、没有贴画面边缘（贴边的小框会被 `draw_result` 里的过滤规则跳过）。

**Q4：帧率是否正常？**
A：512x512 输入的 YOLO 手掌模型一般十几到二十几 FPS，属正常。打开 `debug_mode=1` 可看各阶段耗时；要提速可把 `rgb888p_size` 保持 640x360 不变的情况下确认预处理输入没有被改大，或适当降低送 AI 的分辨率。

**Q5：为什么检测框比实际手掌大/小一圈？**
A：模型学的是"包含整只手的框"，训练数据的标注习惯决定框会略大于手掌轮廓，属正常；框坐标可直接用于后续 ROI 裁剪，略大反而留有余量。若偏差过大，检查是否改动了 `rgb888p_size` 或 pad 参数导致坐标映射错位。
