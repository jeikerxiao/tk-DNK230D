# 实验7 跌倒检测实验

> 用专门训练的 YOLOv5n 跌倒检测模型，实时判断画面中的人是"跌倒（Fall）"还是"正常（NoFall）"，并用不同颜色的框和标签显示。难度：中级。

## 1. 例程简介

本例程运行 `yolov5n-falldown.kmodel`——一个在跌倒数据集上训练的两类别 YOLOv5n 目标检测模型，类别为 `["Fall", "NoFall"]`。画面中的人被框出后，程序根据类别用不同颜色画框并标注类别名与置信度。你将学到：**多类别目标检测**的完整部署（与实验5 的单类别相比只多了类别处理）、如何用现成模型解决行为识别问题。难度为**中级**（代码本身与实验5 几乎同构，但涉及对模型行为边界的理解），预期用途：独居老人看护、病房/养老院监护、安防报警。

## 2. 硬件连接与原理

用到的硬件：

- **CNK230DF 核心板（K230D）**：RISC-V 双核 + 6TOPS KPU；
- **MIPI CSI 摄像头**：`Sensor(width=1280, height=960)`；送 AI 分辨率 `rgb888p_size=[1280,960]`（比实验5高，小目标姿态细节更清楚）；
- **2.4 寸 MIPI LCD**：`display_mode="lcd"`、`display_size=[640,480]`；
- **TF 卡（/sdcard）**：存放模型文件（锚框直接写在代码里，无需 bin 文件）。

**跌倒检测原理（通俗版）**：把"跌倒"当成一种**外观形态**来识别。模型在大量"人摔倒在地"和"人正常站立/坐/走"的图片上训练后，学会了从人体形状、姿态、与地面的相对关系上区分两类。推理时它对画面里的每个人体给出类别分数，后处理选出最高分的类别。注意它判断的是"这一帧的样子像不像摔倒"，**不分析运动过程**——所以躺沙发上可能被判 Fall，弯腰捡东西可能被判 NoFall，这是单帧模型的固有局限。

> [!WARNING]
> - 摄像头/LCD 的 MIPI 排线必须**断电插拔**，插反通电会烧坏器件；
> - 供电不足会在 1280x960 大分辨率推理时引发重启，请用足功率 Type-C 电源；
> - AI 推理时核心板发热明显，**属正常现象**。

## 3. 相关前置知识

- **多类别目标检测**：模型对每个框输出多个类别的分数，取最高者。类别数由 `len(self.labels)` 传给后处理接口，本例 labels 为 `["Fall", "NoFall"]`，共 2 类。
- **YOLOv5 / anchor-based 检测**：三尺度输出（`strides=[8,16,32]`），每尺度 3 个锚框，代码内给出 9 对锚框尺寸：`[10,13, 16,30, 33,23, 30,61, 62,45, 59,119, 116,90, 156,198, 373,326]`。
- **置信度 / 阈值**：`confidence_threshold = 0.3` 过滤低分框。
- **IoU / NMS**：`nms_threshold = 0.45`，重叠度超过它的低分框被删除。
- **kmodel / KPU / 张量 / 推理**：`.kmodel` 为 K230 专用模型格式（本模型输入 640x640）；KPU 硬件加速；张量即多维数组。
- **aicube 后处理库**：嘉楠通用 anchor-based 检测后处理，接口 `anchorbasedet_post_process`，输出 `[类别id, 置信度, x1, y1, x2, y2]` 列表。
- **pad + resize 预处理**：Ai2d 硬件补黑边、等比缩放到 640x640，避免图像拉伸变形。
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

与实验5 完全相同的框架与后处理库。

### 4.2 自定义类 FallDetectionApp

```python
class FallDetectionApp(AIBase):
    def __init__(self, kmodel_path, model_input_size, labels, anchors, confidence_threshold=0.2,
                 nms_threshold=0.5, nms_option=False, strides=[8,16,32], ...):
        ...
        self.color = [(255,0,0,255), (255,0,255,0), (255,255,0,0), (255,255,0,255)]  # 用于绘制不同类别的颜色
```

比实验5 多了一个 `self.color` 颜色表：类别 0（Fall）用红色 `(255,0,0,255)`，类别 1（NoFall）用品红/绿色 `(255,0,255,0)`，颜色按类别 id 索引。

### 4.3 预处理

```python
self.ai2d.pad([0, 0, 0, 0, top, bottom, left, right], 0, [0,0,0])   # 补黑边
self.ai2d.resize(nn.interp_method.tf_bilinear, nn.interp_mode.half_pixel)
self.ai2d.build([1,3,ai2d_input_size[1],ai2d_input_size[0]],[1,3,self.model_input_size[1],self.model_input_size[0]])
```

等比缩放 + 黑边填充到 640x640，与 YOLOv5 训练预处理一致。

### 4.4 后处理

```python
def postprocess(self, results):
    dets = aicube.anchorbasedet_post_process(results[0], results[1], results[2],
        self.model_input_size, self.rgb888p_size, self.strides, len(self.labels),
        self.confidence_threshold, self.nms_threshold, self.anchors, self.nms_option)
    return dets
```

与实验5 一字不差的通用接口，`len(self.labels)` 自动适配类别数。

### 4.5 绘制（按类别分色）

```python
for det_box in dets:
    x1, y1, x2, y2 = det_box[2], det_box[3], det_box[4], det_box[5]
    w = (x2 - x1) * self.display_size[0] // self.rgb888p_size[0]
    h = (y2 - y1) * self.display_size[1] // self.rgb888p_size[1]
    ...
    pl.osd_img.draw_rectangle(x1, y1, int(w), int(h), color=self.color[det_box[0]], thickness=2)
    pl.osd_img.draw_string_advanced(x1, y1-50, 32, " " + self.labels[det_box[0]] + " " + str(round(det_box[1],2)), color=self.color[det_box[0]])
```

注意：与实验5 不同，这里**没有边缘框过滤**，所有检测结果都绘制；框和文字颜色都随类别变化。

### 4.6 主流程

```python
kmodel_path = "/sdcard/examples/kmodel/yolov5n-falldown.kmodel"
confidence_threshold = 0.3
nms_threshold = 0.45
rgb888p_size = [1280, 960]
labels = ["Fall", "NoFall"]       # 模型输出类别名称
anchors = [10, 13, 16, 30, 33, 23, 30, 61, 62, 45, 59, 119, 116, 90, 156, 198, 373, 326]

sensor = Sensor(width=1280, height=960)
pl = PipeLine(rgb888p_size=rgb888p_size, display_size=display_size, display_mode=display_mode)
pl.create(sensor=sensor)
fall_det = FallDetectionApp(kmodel_path, model_input_size=[640,640], labels=labels, anchors=anchors,
    confidence_threshold=confidence_threshold, nms_threshold=nms_threshold, nms_option=False,
    strides=[8,16,32], rgb888p_size=rgb888p_size, display_size=display_size, debug_mode=0)
fall_det.config_preprocess()
while True:
    os.exitpoint()
    with ScopedTiming("total",1):
        img = pl.get_frame()
        res = fall_det.run(img)
        fall_det.draw_result(pl, res)
        pl.show_image()
        gc.collect()
```

### 4.7 可调参数一览

| 参数 | 位置 | 影响 |
| --- | --- | --- |
| `confidence_threshold = 0.3` | `__main__` | 调低更灵敏（可能把蹲下误判成 Fall），调高更保守 |
| `nms_threshold = 0.45` | `__main__` | 去重强度 |
| `rgb888p_size=[1280,960]` | `__main__` | 送 AI 分辨率；降到 [640,480] 可明显提帧率，远处小目标精度下降 |
| `labels = ["Fall","NoFall"]` | `__main__` | 顺序对应模型训练时的类别 id，**不能调换**，否则颜色和名称张冠李戴 |
| `self.color` 颜色表 | `__init__` | 自定义每个类别的框色 |

## 5. 运行效果

- **屏幕画面**：人正常站立/行走时，框上标 `NoFall 0.9x`（类别1颜色）；模拟摔倒躺到地上后，框变为红色并标 `Fall 0.8x`；框上方文字随类别和分数实时刷新。
- **串口打印**：每帧耗时；取消 `print(res)` 注释可看到 `[[类别id, 置信度, x1,y1,x2,y2], ...]`。
- **成功现象**：躺平/摔倒姿势在 1 秒内被判为 Fall，站起后恢复 NoFall。
- **失败现象排查**：
  - 模型加载失败 → 检查 `/sdcard/examples/kmodel/yolov5n-falldown.kmodel`；
  - 摔倒检不出 → 人体在画面中太小（`rgb888p_size` 是 1280x960 但模型输入 640x640，缩得太小细节丢失），靠近些；置信度阈值降到 0.2；
  - 误报 Fall → 弯腰、蹲下、躺沙发都可能触发，属单帧模型局限，见拓展思考的改进方向；
  - 帧率偏低 → 1280x960 输入较大，属正常，可降低分辨率。

## 6. 代码使用说明

1. **硬件准备**：断电插好摄像头与 LCD 排线，插入 TF 卡，Type-C 上电；建议把摄像头架高一点、俯拍测试区域，更接近实际监护安装场景。
2. **固件要求**：正点原子 **CanMV K230D 固件**（内置 libs、aicube、nncase_runtime 等），固件与 CanMV IDE 版本匹配。
3. **运行方式**：CanMV IDE 连接开发板 → 打开本目录 `main.py` → 运行；测试时注意安全，建议在床垫/垫子上模拟跌倒。
4. **模型文件需求**（从正点原子资料拷贝到 TF 卡）：
   - `/sdcard/examples/kmodel/yolov5n-falldown.kmodel` —— 跌倒检测模型（本实验唯一模型文件，锚框写在代码中）。
5. **依赖库**：`libs.PipeLine / libs.AIBase / libs.AI2D`（板载 `/sdcard/app/libs/`）、固件内置 `aicube`、`ulab.numpy`、`image`、`media.sensor`、`nncase_runtime`；不依赖本例程文件夹内其他文件。

## 7. 拓展思考

1. **入门级**：统计一段时间内 Fall 出现的连续帧数，只有**连续 N 帧（如 10 帧）都是 Fall** 才认定为跌倒事件——在 `while True` 循环里加一个计数器即可，能大幅减少弯腰/下蹲引起的瞬时误报。
2. **进阶级**：跌倒报警联动——确认跌倒后点亮红色 LED（GPIO61）、蜂鸣器长鸣，并通过 UART 向 PC/单片机发送报警帧（如 `b'FALL!'`），构成一个最小看护系统。
3. **挑战级**：与实验6（人体关键点）对比：关键点方案可以用几何规则（髋部低于膝部、躯干与竖直方向夹角大于 60° 等）判断跌倒，可解释性强；模型方案开箱即用但依赖训练数据。尝试两者融合：模型判 Fall 且关键点几何也满足才报警，进一步压低误报率。

## 8. 常见问题

**Q1：kmodel 加载失败怎么办？**
A：确认 `/sdcard/examples/kmodel/yolov5n-falldown.kmodel` 已拷入 TF 卡、文件名大小写一致、来自正点原子配套资料。若整个 `/sdcard/examples` 目录都不存在，请先按资料教程把例程资源包拷贝到卡中。

**Q2：明明摔倒了却显示 NoFall / 检不到？**
A：单帧模型的判断依赖"摔倒的样子"是否典型：① 侧面躺平、四肢伸展最容易被判 Fall；蜷缩、被家具遮挡时容易漏判；② 人体在画面中占比太小，走近一些或调整摄像头角度；③ 把 `confidence_threshold` 降到 0.2 观察分数变化；④ 换光照充足的环境。

**Q3：蹲下、弯腰经常被误判成 Fall，怎么办？**
A：这是模型能力边界，可从应用层缓解：要求 Fall 连续多帧出现才报警（见拓展思考1）；结合实验6 的关键点几何校验；安装时采用俯拍视角、保持人体完整入镜。若应用场景苛刻，需要用自己场景的数据重新训练模型。

**Q4：本实验帧率比实验5 低，正常吗？**
A：正常。本实验 `rgb888p_size=[1280,960]` 远大于实验5 的 [640,360]，预处理数据量更大。对帧率敏感可降到 [640,480]，跌倒目标通常较大，精度损失有限。

**Q5：为什么 labels 顺序不能随便写？**
A：模型输出的是类别 **id**（0 或 1），`labels[det_box[0]]` 按 id 索引取名字，`self.color[det_box[0]]` 按 id 取颜色。id 与名字的对应关系在训练时就固定了（本模型 0=Fall、1=NoFall），调换 labels 顺序会导致名称和颜色全部对调。
