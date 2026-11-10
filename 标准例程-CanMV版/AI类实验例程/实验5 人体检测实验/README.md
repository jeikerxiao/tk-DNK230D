# 实验5 人体检测实验

> 用 YOLOv5n 人体检测模型实时框出画面中的"person"，是学习 anchor-based 目标检测后处理的入门例程。难度：初级。

## 1. 例程简介

本例程通过摄像头采集画面，用 KPU 运行 `person_detect_yolov5n.kmodel`（YOLOv5-nano 结构、单类别 person），把画面中的人体用紫色矩形框标出，并在框上方显示类别名和置信度。你将学到：YOLO 系列模型在 K230D 上的完整部署流程——三尺度特征图输出、`aicube.anchorbasedet_post_process` 通用后处理、以及绘制时的边缘框过滤技巧。难度为**初级**，预期用途：人流统计、监控报警、机器人避人等。

## 2. 硬件连接与原理

用到的硬件：

- **CNK230DF 核心板（K230D）**：RISC-V 双核 + 6TOPS KPU；
- **MIPI CSI 摄像头**：`Sensor(width=1280, height=960)`；送 AI 分辨率 `rgb888p_size=[640,360]`；
- **2.4 寸 MIPI LCD**：`display_mode="lcd"`、`display_size=[640,480]`；
- **TF 卡（/sdcard）**：存放模型文件（本实验只需一个 kmodel，不需要锚框 bin 文件，锚框直接写在代码里）。

**YOLO 检测原理（通俗版）**：YOLO（You Only Look Once）把图片一次性送入网络，网络输出三张"热力图式"的结果（对应原图缩小 8、16、32 倍的三个尺度，即 `strides=[8,16,32]`）。每个尺度上的每个格子都会给出若干候选框：位置偏移 + 宽高 + "有人的概率"。小尺度格子看大目标，大尺度格子看小目标。后处理再把三个尺度的所有候选框解码回原图坐标，按置信度过滤、用 NMS 去重。

**锚框（anchor）**：YOLOv5 预设了 9 种标准框形（3 个尺度各 3 种），代码里直接给出：

```python
anchors = [10, 13, 16, 30, 33, 23, 30, 61, 62, 45, 59, 119, 116, 90, 156, 198, 373, 326]
```

每两个数是一对（宽, 高），候选框以这些标准形为基准微调得到。

> [!WARNING]
> 新手风险提醒：
> - 摄像头 MIPI 排线**插反并通电会烧坏**摄像头或核心板，务必断电插拔；
> - LCD 排线同样断电操作；
> - 供电不足会导致推理时死机重启，请用足功率 Type-C 电源；
> - AI 推理时核心板发热属**正常现象**。

## 3. 相关前置知识

- **神经网络 / 模型 / kmodel**：训练好的网络保存为 K230 专用 `.kmodel` 格式，KPU 直接运行。本模型输入 640x640。
- **KPU / 推理 / 张量**：KPU 是 AI 加速硬件；推理即"喂图出结果"；张量即多维数组，本模型一次输出 3 个张量（三个尺度的特征图）。
- **置信度（confidence）与阈值**：每个候选框附带"这里有人"的概率；`confidence_threshold = 0.2` 过滤低分框（YOLO 通常用较低阈值先粗筛）。
- **IoU（交并比）**：两框重叠度 = 交集/并集。
- **NMS（非极大值抑制）**：同一个人被多个格子框住时，保留分最高的框，删掉与它 IoU 超过 `nms_threshold = 0.6` 的框。
- **strides（步长/降采样倍数）**：`[8,16,32]` 表示三个输出特征图分别是输入图的 1/8、1/16、1/32 大小，后处理时格子坐标要乘回步长才是原图坐标。
- **nms_option**：传给后处理接口的选项，`False` 表示普通 NMS（另一种可选 soft-NMS 类行为，此处不用深究）。
- **帧率（FPS）**：`ScopedTiming("total",1)` 打印每帧耗时，可估算帧率。

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

与实验1相同的框架；区别是后处理库换成了 **`aicube`**——嘉楠提供的通用 AI 后处理库，里面的 `anchorbasedet_post_process` 能处理所有 anchor-based 的 YOLO 类检测模型（实验5/7/8 都用它）。

### 4.2 自定义人体检测类 PersonDetectionApp

```python
class PersonDetectionApp(AIBase):
    def __init__(self, kmodel_path, model_input_size, labels, anchors, confidence_threshold=0.2,
                 nms_threshold=0.5, nms_option=False, strides=[8,16,32], ...):
```

比实验1多了 `labels`（类别名列表，本例只有 `["person"]`）和 `strides` 两个参数。

### 4.3 预处理（pad + resize，补黑边）

```python
def config_preprocess(self, input_image_size=None):
    ai2d_input_size = input_image_size if input_image_size else self.rgb888p_size
    top, bottom, left, right = self.get_padding_param()
    self.ai2d.pad([0,0,0,0,top,bottom,left,right], 0, [0,0,0])
    self.ai2d.resize(nn.interp_method.tf_bilinear, nn.interp_mode.half_pixel)
    self.ai2d.build([1,3,ai2d_input_size[1],ai2d_input_size[0]],[1,3,self.model_input_size[1],self.model_input_size[0]])
```

注意与实验1的区别：这里 pad 的补边色是**黑色 [0,0,0]**（YOLOv5 训练时的习惯），且 `get_padding_param()` 上下左右都补（图像居中）。

### 4.4 后处理：三尺度解码 + NMS

```python
def postprocess(self, results):
    dets = aicube.anchorbasedet_post_process(results[0], results[1], results[2],
        self.model_input_size, self.rgb888p_size, self.strides, len(self.labels),
        self.confidence_threshold, self.nms_threshold, self.anchors, self.nms_option)
    return dets
```

`results[0..2]` 是模型的三个输出张量，接口内部完成：sigmoid 置信度过滤 → 锚框解码 → 坐标映射回 `rgb888p_size` → NMS。返回的每个 `det_box` 结构为 `[类别id, 置信度, x1, y1, x2, y2]`（左上、右下两点式坐标）。

### 4.5 绘制结果（含边缘框过滤）

```python
x1, y1, x2, y2 = det_box[2], det_box[3], det_box[4], det_box[5]
...
if (h < (0.1 * self.display_size[0])):
    continue
if (w < (0.25 * self.display_size[0]) and ((x1 < (0.03 * self.display_size[0])) or (x2 > (0.97 * self.display_size[0])))):
    continue
pl.osd_img.draw_rectangle(x1, y1, int(w), int(h), color=(255, 0, 255, 0), thickness=2)
pl.osd_img.draw_string_advanced(x1, y1-50, 32, " " + self.labels[det_box[0]] + " " + str(round(det_box[1],2)), color=(255,0,255,0))
```

三个 `if...continue` 是**边缘误检过滤**：太矮的框、以及又窄又贴着画面左右边缘的框多半是截了一半的人或误检，直接不画。画框为紫色 `(255,0,255,0)`，框上方写 `person 0.87` 这样的文字。

### 4.6 主流程

```python
kmodel_path = "/sdcard/examples/kmodel/person_detect_yolov5n.kmodel"
confidence_threshold = 0.2
nms_threshold = 0.6
rgb888p_size = [640, 360]
labels = ["person"]
anchors = [10, 13, 16, 30, 33, 23, 30, 61, 62, 45, 59, 119, 116, 90, 156, 198, 373, 326]

sensor = Sensor(width=1280, height=960)
pl = PipeLine(rgb888p_size=rgb888p_size, display_size=display_size, display_mode=display_mode)
pl.create(sensor=sensor)
person_det = PersonDetectionApp(kmodel_path, model_input_size=[640,640], labels=labels,
    anchors=anchors, confidence_threshold=confidence_threshold, nms_threshold=nms_threshold,
    nms_option=False, strides=[8,16,32], rgb888p_size=rgb888p_size, display_size=display_size, debug_mode=0)
person_det.config_preprocess()
while True:
    os.exitpoint()
    with ScopedTiming("total",1):
        img = pl.get_frame()
        res = person_det.run(img)
        person_det.draw_result(pl, res)
        pl.show_image()
        gc.collect()
```

### 4.7 可调参数一览

| 参数 | 位置 | 影响 |
| --- | --- | --- |
| `confidence_threshold = 0.2` | `__main__` | 调高减少误检但可能漏检远处的人；调低更灵敏 |
| `nms_threshold = 0.6` | `__main__` | 调低去重更狠；过高同一人可能双框 |
| `rgb888p_size=[640,360]` | `__main__` | 送 AI 分辨率，加大提升远处小目标检出、降低帧率 |
| `model_input_size=[640,640]` | 构造处 | 与 kmodel 绑定，不可改 |
| 边缘过滤的三个 `if` | `draw_result()` | 可按需放宽/收紧，比如允许画面边缘的半个人 |

## 5. 运行效果

- **屏幕画面**：实时画面中每个人被紫色框套住，框上方显示 `person 0.92`（类别+置信度，保留两位小数）；多人同框各自有框；走到画面边缘被截断的人多数不会出框（被边缘过滤逻辑跳过）。
- **串口打印**：每帧耗时信息（ScopedTiming）；取消 `print(res)` 注释可看到 `[[类别, 置信度, x1, y1, x2, y2], ...]` 原始数据。
- **成功现象**：人站/走/坐都能稳定框住，框跟随流畅。
- **失败现象排查**：
  - 模型加载失败 → 检查 `/sdcard/examples/kmodel/person_detect_yolov5n.kmodel` 是否存在；
  - 检不到人 → 人太小太远（试试靠近或加大 `rgb888p_size`）、光线太暗、或人正好贴在画面边缘被过滤；
  - 框乱跳 → 光线剧烈变化或置信度太低，调到 0.3~0.4 试试。

## 6. 代码使用说明

1. **硬件准备**：断电插好摄像头与 LCD 排线，插入 TF 卡，Type-C 上电。
2. **固件要求**：正点原子 **CanMV K230D 固件**（内置 libs、aicube、nncase_runtime 等模块），固件与 CanMV IDE 版本匹配。
3. **运行方式**：CanMV IDE 连接开发板 → 打开本目录 `main.py` → 点绿色运行按钮。
4. **模型文件需求**（从正点原子资料拷贝到 TF 卡）：
   - `/sdcard/examples/kmodel/person_detect_yolov5n.kmodel` —— YOLOv5n 人体检测模型；
   - 本实验**不需要** prior_data 锚框文件，锚框数组直接写在代码中。
5. **依赖库**：`libs.PipeLine / libs.AIBase / libs.AI2D`（板载 `/sdcard/app/libs/`）、固件内置 `aicube`、`ulab.numpy`、`image`、`media.sensor`、`nncase_runtime`；不依赖本例程文件夹内其他文件。

## 7. 拓展思考

1. **入门级**：统计画面中的人数——`len(res)` 就是检出人数，把它用 `draw_string_advanced` 写到屏幕角落，做一个"实时人数计数器"；再试试调整 `confidence_threshold` 观察计数变化。
2. **进阶级**：加"区域入侵报警"——在屏幕上定义一个矩形区域，当某个人框中心落入该区域时点亮红色 LED（GPIO61）并让蜂鸣器响（参考基础实验例程）。
3. **挑战级**：与实验6（人体关键点）串联，先检人再画骨骼；或通过 UART 把人数/坐标上报 PC，做客流统计系统。

## 8. 常见问题

**Q1：kmodel 加载失败怎么办？**
A：确认 TF 卡中 `/sdcard/examples/kmodel/person_detect_yolov5n.kmodel` 存在、文件名大小写与代码一致，且来自正点原子配套资料。K210 或其他平台的模型不能在本板运行。

**Q2：为什么画面边缘的人经常不被框？**
A：这是 `draw_result()` 里刻意的边缘过滤：太小的框、又窄又贴边的框会被 `continue` 跳过，目的是抑制"半个身子"造成的误检框。想让边缘的人也出框，把三个 `if (...) continue` 注释掉即可（可能出现一些贴边误检框）。

**Q3：检测不到远处的人怎么调？**
A：模型输入 640x640，而送 AI 的图只有 640x360，远处的人可能只占十几个像素。办法：① 加大 `rgb888p_size`（如 [1280,720]，帧率会降）；② 降低 `confidence_threshold` 到 0.15；③ 让摄像头光学上更"望远"（离人近些）。

**Q4：帧率大概多少算正常？**
A：640x640 输入的 YOLOv5n 在该配置下一般十几 FPS，画面基本流畅。打开 `debug_mode=1` 可查看预处理/推理/后处理各阶段耗时；想提速优先减小 `rgb888p_size`。

**Q5：同一个人出现两个框？**
A：把 `nms_threshold` 从 0.6 调低到 0.45 左右，重复框会被更严格地合并。若两框分别来自不同尺度且重叠不多，也可适当调高 `confidence_threshold`。
