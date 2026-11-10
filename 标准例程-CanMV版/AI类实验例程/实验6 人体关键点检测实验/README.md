# 实验6 人体关键点检测实验

> 用 YOLOv8n-pose 模型实时检测人体的 17 个关键点并连成"火柴人"骨骼，是姿态类 AI 应用（跌倒检测、体感游戏）的基础。难度：中级。

## 1. 例程简介

本例程运行 `yolov8n-pose.kmodel`（YOLOv8-nano 姿态模型），一次推理同时输出人体框和 **17 个身体关键点**（鼻子、双眼、双耳、双肩、双肘、双腕、双髋、双膝、双踝，即 COCO 格式），然后按预定义的骨骼连接关系 `SKELETON` 用彩色线段把关键点连成骨架，叠加显示在 LCD 上。你将学到：单阶段姿态估计模型的输出结构、关键点置信度的用法、骨骼拓扑与分色绘制。难度为**中级**，预期用途：运动分析、跌倒检测（实验7 的思想基础）、体感交互。

## 2. 硬件连接与原理

用到的硬件：

- **CNK230DF 核心板（K230D）**：RISC-V 双核 + 6TOPS KPU；
- **MIPI CSI 摄像头**：`Sensor(width=1280, height=960)`；送 AI 分辨率 `rgb888p_size=[640,360]`；
- **2.4 寸 MIPI LCD**：`display_mode="lcd"`、`display_size=[640,480]`；
- **TF 卡（/sdcard）**：存放模型文件（只需一个 kmodel，YOLOv8 是 anchor-free 结构，代码里不需要锚框）。

**姿态估计原理（通俗版）**：YOLOv8-pose 把"找人"和"找关节"合成一步：网络输出里每个候选目标除了框坐标和"有人"分数，还带 17 组 (x, y, 分数)。每个关键点的分数表示"这个关节在这个位置的可信度"——被遮挡的关节（比如手插兜里）分数会很低，程序据此决定画不画。后处理由 `aidemo.person_kp_postprocess` 完成解码、过滤和 NMS。

**骨骼连接**：17 个点本身只是散点，代码用 `SKELETON` 列表规定哪些点两两相连（如 (16,14) 表示右踝-右膝），一共 19 条"肢体"，连起来就是火柴人。

> [!WARNING]
> - 摄像头/LCD 的 MIPI 排线必须**断电插拔**，插反通电会烧坏器件；
> - 请用足功率 Type-C 电源，供电不足会导致推理时重启；
> - AI 推理时核心板发热明显，**属正常现象**。

## 3. 相关前置知识

- **关键点（keypoint）**：人体上固定编号的关节/器官位置点。本模型采用 COCO 17 点格式：1 鼻、2 左眼、3 右眼、4 左耳、5 右耳、6 左肩、7 右肩、8 左肘、9 右肘、10 左腕、11 右腕、12 左髋、13 右髋、14 左膝、15 右膝、16 左踝、17 右踝（代码中索引从 0 开始，即 `kpses[i][0]` 是鼻子）。
- **骨骼拓扑（SKELETON）**：关键点的连接表，决定火柴人的"骨架长相"。
- **关键点置信度**：每个点附带一个分数 `kps_s`，代码里 `if (kps_s > 0)` 才画点；肢体线要求两端分数都 `> 0.0` 才画。
- **anchor-free**：YOLOv8 不再使用预设锚框（对比实验5 的 YOLOv5），直接回归目标框，所以代码里没有 anchors 参数。
- **置信度阈值 / NMS 阈值 / IoU**：目标级别的过滤与去重，`confidence_threshold = 0.2`、`nms_threshold = 0.5`。
- **kmodel / KPU / 张量 / 推理**：`.kmodel` 是 K230 专用模型格式，本模型输入 320x320；KPU 加速推理；张量即多维数组。
- **RGB888P / pad+resize 预处理**：PipeLine 输出平面 RGB 图，Ai2d 硬件补黑边后缩放到模型输入尺寸。
- **帧率（FPS）**：`ScopedTiming("total",1)` 打印每帧耗时。

## 4. 软件代码逻辑说明

### 4.1 框架导入

```python
from libs.PipeLine import PipeLine, ScopedTiming
from libs.AIBase import AIBase
from libs.AI2D import Ai2d
import nncase_runtime as nn
import ulab.numpy as np
import aidemo
```

框架同实验1；后处理使用 `aidemo` 库的专用接口。

### 4.2 自定义类 PersonKeyPointApp 的数据定义

```python
# 骨骼信息
self.SKELETON = [(16, 14),(14, 12),(17, 15),(15, 13),(12, 13),(6, 12),(7, 13),(6, 7),
                 (6, 8),(7, 9),(8, 10),(9, 11),(2, 3),(1, 2),(1, 3),(2, 4),(3, 5),(4, 6),(5, 7)]
# 肢体颜色
self.LIMB_COLORS = [...]   # 19 种 ARGB 颜色，与 SKELETON 一一对应
# 关键点颜色，共17个
self.KPS_COLORS = [...]    # 17 种 ARGB 颜色
```

`SKELETON` 里的编号是 1~17（COCO 编号），代码取点时用 `ske[0]-1` 转成 0 基索引。腿、臂、头部分别用不同颜色，显示效果更直观。

### 4.3 预处理（pad 黑边 + resize）

```python
self.ai2d.pad([0,0,0,0,top,bottom,left,right], 0, [0,0,0])
self.ai2d.resize(nn.interp_method.tf_bilinear, nn.interp_mode.half_pixel)
self.ai2d.build([1,3,ai2d_input_size[1],ai2d_input_size[0]],[1,3,self.model_input_size[1],self.model_input_size[0]])
```

`get_padding_param()` 计算等比缩放后的补边量，图像居中、四周补黑，与 YOLOv8 训练预处理一致。

### 4.4 后处理

```python
def postprocess(self, results):
    results = aidemo.person_kp_postprocess(results[0], [self.rgb888p_size[1], self.rgb888p_size[0]],
        self.model_input_size, self.confidence_threshold, self.nms_threshold)
    return results
```

`aidemo.person_kp_postprocess` 输入模型原始输出张量，返回 `results[0]`（人体框列表）和 `results[1]`（每个人的关键点数组，`kpses[i][k] = [x, y, score]`，坐标已映射回 `rgb888p_size` 坐标系）。

### 4.5 绘制骨骼

```python
def draw_result(self, pl, res):
    if res[0]:
        pl.osd_img.clear()
        kpses = res[1]
        for i in range(len(res[0])):            # 遍历每个人
            for k in range(17+2):               # 遍历17个关键点及骨骼
                if (k < 17):
                    kps_x, kps_y, kps_s = round(kpses[i][k][0]), round(kpses[i][k][1]), kpses[i][k][2]
                    kps_x1 = int(float(kps_x) * self.display_size[0] // self.rgb888p_size[0])
                    kps_y1 = int(float(kps_y) * self.display_size[1] // self.rgb888p_size[1])
                    if (kps_s > 0):
                        pl.osd_img.draw_circle(kps_x1, kps_y1, 5, self.KPS_COLORS[k], 4)
                ske = self.SKELETON[k]
                # 取骨骼两端点坐标，换算到显示分辨率
                pos1_s, pos2_s = kpses[i][(ske[0]-1)][2], kpses[i][(ske[1]-1)][2]
                if (pos1_s > 0.0 and pos2_s > 0.0):
                    pl.osd_img.draw_line(pos1_x_, pos1_y_, pos2_x_, pos2_y_, self.LIMB_COLORS[k], 4)
```

逻辑：先画关键点圆圈（半径 5、线宽 4），再画 19 条肢体线；坐标都要从 `rgb888p_size`（640x360）换算到 `display_size`（640x480）。注意循环写的是 `range(17+2)`，共 19 次——前 17 次同时画点和线，后 2 次只画剩余的线（k>=17 时跳过画点分支）。

### 4.6 主流程

```python
kmodel_path = "/sdcard/examples/kmodel/yolov8n-pose.kmodel"
confidence_threshold = 0.2
nms_threshold = 0.5
rgb888p_size = [640, 360]

sensor = Sensor(width=1280, height=960)
pl = PipeLine(rgb888p_size=rgb888p_size, display_size=display_size, display_mode=display_mode)
pl.create(sensor=sensor)
person_kp = PersonKeyPointApp(kmodel_path, model_input_size=[320,320],
    confidence_threshold=confidence_threshold, nms_threshold=nms_threshold,
    rgb888p_size=rgb888p_size, display_size=display_size, debug_mode=0)
person_kp.config_preprocess()
while True:
    os.exitpoint()
    with ScopedTiming("total",1):
        img = pl.get_frame()
        res = person_kp.run(img)
        person_kp.draw_result(pl, res)
        pl.show_image()
        gc.collect()
```

### 4.7 可调参数一览

| 参数 | 位置 | 影响 |
| --- | --- | --- |
| `confidence_threshold = 0.2` | `__main__` | 人体框过滤阈值，调高减少误检 |
| `nms_threshold = 0.5` | `__main__` | 去重强度 |
| `rgb888p_size=[640,360]` | `__main__` | 送 AI 分辨率；调大可改善远处小目标关键点质量，帧率下降 |
| `model_input_size=[320,320]` | 构造处 | 与 kmodel 绑定，不可改 |
| `SKELETON / LIMB_COLORS / KPS_COLORS` | `__init__` | 骨骼连接与配色，可自定义 |
| 关键点分数判断 `kps_s > 0` | `draw_result()` | 可提高到 0.3 等，过滤低质量点，骨架更干净 |

## 5. 运行效果

- **屏幕画面**：画面中每个人身上覆盖一个彩色"火柴人"骨架——关节处有圆点，四肢/躯干/头部分别用不同颜色线段连接；人活动时骨架实时跟随；被遮挡的关节（分数为 0）不显示。
- **串口打印**：每帧耗时（ScopedTiming）；取消 `print(res)` 注释可看到框和 17x3 的关键点数组。
- **成功现象**：全身入镜时 17 点齐全、骨架结构正确；侧身时部分点消失属正常（被遮挡）。
- **失败现象排查**：
  - 模型加载失败 → 检查 `/sdcard/examples/kmodel/yolov8n-pose.kmodel`；
  - 检不到人 → 光线、距离、`confidence_threshold`；
  - 关键点乱飞 → 人太小（只占几十像素时关键点不准）、剧烈运动导致模糊；靠近摄像头、保证光照再试；
  - 只有框没有骨架 → 确认没有改动 `model_input_size`，模型与代码版本配套。

## 6. 代码使用说明

1. **硬件准备**：断电插好摄像头与 LCD 排线，插入 TF 卡，Type-C 上电。
2. **固件要求**：正点原子 **CanMV K230D 固件**（内置 libs、aidemo、nncase_runtime 等），固件与 CanMV IDE 版本匹配。
3. **运行方式**：CanMV IDE 连接开发板 → 打开本目录 `main.py` → 运行。
4. **模型文件需求**（从正点原子资料拷贝到 TF 卡）：
   - `/sdcard/examples/kmodel/yolov8n-pose.kmodel` —— YOLOv8n 人体姿态模型（本实验唯一需要的模型文件，无需锚框 bin）。
5. **依赖库**：`libs.PipeLine / libs.AIBase / libs.AI2D`（板载 `/sdcard/app/libs/`）、固件内置 `aidemo`、`ulab.numpy`、`image`、`media.sensor`、`nncase_runtime`；不依赖本例程文件夹内其他文件。

## 7. 拓展思考

1. **入门级**：把画关键点的分数门槛从 `> 0` 提到 `> 0.3`，对比骨架"干净程度"；再修改 `LIMB_COLORS` 给不同肢体换色。
2. **进阶级**：利用关键点做**举手检测**——当腕部关键点（编号 10/11）的 y 坐标高于肩部（6/7）时，在屏幕上显示"举手了！"并点亮 LED；类似的还可以做"深蹲计数"（髋-膝-踝夹角变化）。
3. **挑战级**：理解实验7（跌倒检测）的两种思路差异：本实验用几何规则（关键点夹角/位置）也能做跌倒判断——试着用"髋部关键点低于膝部"或"躯干与地面夹角"写一个纯规则版跌倒检测，并与实验7的模型版对比优劣。

## 8. 常见问题

**Q1：kmodel 加载失败怎么办？**
A：确认 `/sdcard/examples/kmodel/yolov8n-pose.kmodel` 已拷贝到 TF 卡且文件名一致；模型需来自正点原子配套资料；TF 卡需格式化为板子可识别的文件系统（出厂卡直接可用）。

**Q2：为什么有的人只画出部分骨架？**
A：正常现象。被遮挡或出画的关键点分数低，代码只在 `kps_s > 0` 时画点、两端分数都有效时画线。半身入镜自然只有上半身骨架。

**Q3：关键点抖动明显，怎么改善？**
A：① 输入分辨率偏低（模型 320x320、送 AI 图 640x360），人离得远时每个关节只有几个像素，抖动难免；靠近些或加大 `rgb888p_size`；② 保证光照充足、快门时间不过长；③ 进阶：对关键点坐标做跨帧滑动平均。

**Q4：帧率是否正常？如何提速？**
A：320x320 输入的 YOLOv8n-pose 一般可达十几到二十几 FPS，属正常。提速可减小 `rgb888p_size`；`debug_mode=1` 可查看各阶段耗时定位瓶颈。

**Q5：`for k in range(17+2)` 为什么是 19 而不是 17？**
A：这个循环把"画 17 个点"和"画 19 条骨骼线"合并进了一个循环：k<17 时既画第 k 个点、又画第 k 条线；k=17、18 时只画最后两条线。是省一层循环的写法，改动时注意别破坏这个对应关系（`SKELETON[k]` 与 `LIMB_COLORS[k]` 按下标一一对应）。
