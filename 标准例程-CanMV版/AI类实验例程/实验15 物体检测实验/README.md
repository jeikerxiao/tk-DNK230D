# 实验15 物体检测实验

> 用 YOLOv8n 模型识别 COCO 数据集的 80 类常见物体（人、车、猫狗、杯子、手机……），实时画框标注，是"什么都能认一点"的通用物体检测例程。难度：初级。

## 1. 例程简介

本例程运行 `yolov8n_320.kmodel`（YOLOv8-nano，输入 320x320），可检测 **COCO 数据集 80 个类别**：person、bicycle、car、cat、dog、bottle、cup、cell phone、keyboard 等日常物体，每个类别用不同颜色的框标注并显示类别名和置信度。与实验5（YOLOv5、anchor-based）不同，YOLOv8 是 **anchor-free** 结构，其后处理直接在 Python 代码里实现（含一个纯 Python 版 NMS），你能完整看到"模型输出 → 框坐标 → NMS"的每一步。难度为**初级**（框架与实验5 相同），预期用途：智能监控、物品识别、机器人环境感知，也是理解通用目标检测后处理的最佳教材。

## 2. 硬件连接与原理

用到的硬件：

- **CNK230DF 核心板（K230D）**：RISC-V 双核 + 6TOPS KPU；
- **MIPI CSI 摄像头**：`Sensor(width=1280, height=960)`；送 AI 分辨率 `rgb888p_size=[320,320]`（与模型输入一致，帧率优先）；
- **2.4 寸 MIPI LCD**：`display_mode="lcd"`、`display_size=[640,480]`；
- **TF 卡（/sdcard）**：存放模型文件（无需锚框文件，YOLOv8 无锚框）。

**YOLOv8 检测原理（通俗版）**：模型把 320x320 的图变成一张"候选表"：每行是一个候选框（中心 x、中心 y、宽、高）加上 80 个类别分数。后处理分四步：① 每行取 80 类里的最高分作为该框置信度，低于 0.2 的整行扔掉；② 剩下的框从"模型 320x320 坐标"换算回摄像头图像坐标（乘 `x_factor/y_factor`）；③ 跑 NMS 去掉互相重叠的重复框（阈值 0.2）；④ 最多保留 `max_boxes_num=50` 个框。注意本实验预处理**只做 resize 不做 pad**（320x320 送 320x320 模型，比例一致无需补边）。

> [!WARNING]
> - 摄像头/LCD 的 MIPI 排线必须**断电插拔**，插反通电会烧坏器件；
> - 供电不足会导致推理时重启，请使用足功率 Type-C 电源；
> - AI 推理发热明显，**属正常现象**。

## 3. 相关前置知识

- **COCO 80 类**：微软 COCO 数据集定义的 80 种常见物体类别，代码里的 `labels` 列表按类别 id 0~79 排列（0=person，79=toothbrush）。模型输出的类别 id 就按这个表翻译成名字。
- **anchor-free（无锚框）**：YOLOv8 直接回归框的中心与宽高，不再依赖预设锚框（对比实验5 的 YOLOv5），所以代码里没有 anchors。
- **模型输出转置**：模型原始输出形状是 `[1, 84, N]`（84 = 4 坐标 + 80 类分数，N 为候选数），代码 `result.transpose()` 转成 `[N, 84]`，一行一个候选框，便于遍历。
- **置信度 / 阈值**：`confidence_threshold = 0.2`，每框取 `np.max(scores)` 为置信度、`np.argmax(scores)` 为类别 id。
- **IoU / NMS**：本实验的 NMS 是**纯 Python 实现**（`nms()` 方法），按分数从高到低逐个保留，把与它 IoU 超过 `nms_threshold = 0.2` 的框全部剔除——代码里 `xx1/yy1/xx2/yy2` 求交集、`ovr = inter / (areas[i] + new_areas - inter)` 就是 IoU 公式。
- **max_boxes_num**：一帧最多显示的框数（50），防止密集场景绘制过载。
- **kmodel / KPU / 张量 / 推理 / 帧率**：概念同前；`ScopedTiming("total",1)` 打印每帧耗时。

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

本例后处理不用 aidemo/aicube 的现成接口，全部在 Python 里手写。

### 4.2 自定义类 ObjectDetectionApp 初始化

```python
self.color_four = [(255, 220, 20, 60), (255, 119, 11, 32), ...]   # 20 种预置颜色，按类别id取模选色
# 宽高缩放比例
self.x_factor = float(self.rgb888p_size[0])/self.model_input_size[0]
self.y_factor = float(self.rgb888p_size[1])/self.model_input_size[1]
```

`get_color(x)` 用 `x % 20` 给 80 个类别循环分配 20 种颜色。

### 4.3 预处理（只 resize）

```python
def config_preprocess(self, input_image_size=None):
    ai2d_input_size = input_image_size if input_image_size else self.rgb888p_size
    self.ai2d.resize(nn.interp_method.tf_bilinear, nn.interp_mode.half_pixel)
    self.ai2d.build([1,3,ai2d_input_size[1],ai2d_input_size[0]],[1,3,self.model_input_size[1],self.model_input_size[0]])
```

没有 pad——因为 `rgb888p_size=[320,320]` 与模型输入相同比例。若你把 `rgb888p_size` 改成非正方形，图像会被拉伸，检测精度会受影响（模型训练时是等比输入）。

### 4.4 后处理（手写解码 + 纯 Python NMS）

```python
def postprocess(self, results):
    result = results[0]
    result = result.reshape((result.shape[0] * result.shape[1], result.shape[2]))
    output_data = result.transpose()
    boxes_ori = output_data[:, 0:4]      # 前4列：cx, cy, w, h（模型输入坐标系）
    scores_ori = output_data[:, 4:]      # 后80列：各类别分数
    confs_ori = np.max(scores_ori, axis=-1)
    inds_ori = np.argmax(scores_ori, axis=-1)
    boxes, scores, inds = [], [], []
    for i in range(len(boxes_ori)):
        if confs_ori[i] > confidence_threshold:          # 阈值过滤
            x, y, w, h = boxes_ori[i,0], boxes_ori[i,1], boxes_ori[i,2], boxes_ori[i,3]
            left   = int((x - 0.5*w) * self.x_factor)    # 中心宽高 -> 左上右下，并换算到rgb888p坐标
            top    = int((y - 0.5*h) * self.y_factor)
            right  = int((x + 0.5*w) * self.x_factor)
            bottom = int((y + 0.5*h) * self.y_factor)
            boxes.append([left, top, right, bottom])
    ...
    keep = self.nms(boxes, scores, nms_threshold)        # NMS过程
    dets = np.concatenate((boxes, scores..., inds...), axis=1)   # [x1,y1,x2,y2,score,类别id]
    dets_out = dets_out[:self.max_boxes_num, :]          # 最多max_boxes_num个框
    return dets_out
```

> [!NOTE]
> 这里的 `confidence_threshold`、`nms_threshold` 引用的是**模块级全局变量**（`__main__` 里定义的那两个），而不是 `self.confidence_threshold`。改参数时改 `__main__` 里的全局变量即可生效，这是本文件与其他实验的一个细节差异。

纯 Python NMS 核心：

```python
def nms(self, boxes, scores, thresh):
    areas = (x2 - x1 + 1) * (y2 - y1 + 1)
    order = np.argsort(scores, axis=0)[::-1]     # 按分数从高到低排序
    keep = []
    while order.size > 0:
        i = order[0]
        keep.append(i)                            # 保留当前最高分框
        # 计算其余框与它的IoU（交集/并集）
        xx1 = np.maximum(x1[i], new_x1)
        yy1 = np.maximum(y1[i], new_y1)
        xx2 = np.minimum(x2[i], new_x2)
        yy2 = np.minimum(y2[i], new_y2)
        inter = np.maximum(0.0, xx2-xx1+1) * np.maximum(0.0, yy2-yy1+1)
        ovr = inter / (areas[i] + new_areas - inter)
        # IoU小于阈值的框进入下一轮
        order = np.array([order[k] for k, ind in enumerate(ovr) if ind < thresh], dtype=np.uint8)
    return keep
```

### 4.5 绘制

```python
pl.osd_img.draw_rectangle(x, y, w, h, color=self.get_color(int(det[5])), thickness=4)
pl.osd_img.draw_string_advanced(x, y-50, 32, " " + self.labels[int(det[5])] + " " + str(round(det[4],2)), color=self.get_color(int(det[5])))
```

每类一种颜色（20 色循环），框上方写"类别名 + 置信度"。

### 4.6 主流程

```python
kmodel_path = "/sdcard/examples/kmodel/yolov8n_320.kmodel"
labels = ["person", "bicycle", "car", ..., "toothbrush"]   # COCO 80类
confidence_threshold = 0.2
nms_threshold = 0.2
max_boxes_num = 50
rgb888p_size = [320, 320]

sensor = Sensor(width=1280, height=960)
pl = PipeLine(rgb888p_size=rgb888p_size, display_size=display_size, display_mode=display_mode)
pl.create(sensor=sensor)
ob_det = ObjectDetectionApp(kmodel_path, labels=labels, model_input_size=[320,320],
    max_boxes_num=max_boxes_num, confidence_threshold=confidence_threshold,
    nms_threshold=nms_threshold, rgb888p_size=rgb888p_size, display_size=display_size, debug_mode=0)
ob_det.config_preprocess()
while True:
    os.exitpoint()
    with ScopedTiming("total",1):
        img = pl.get_frame()
        res = ob_det.run(img)
        ob_det.draw_result(pl, res)
        pl.show_image()
        gc.collect()
```

### 4.7 可调参数一览

| 参数 | 位置 | 影响 |
| --- | --- | --- |
| `confidence_threshold = 0.2` | `__main__`（全局变量） | 调低更灵敏但误检多；调高只认很确定的目标 |
| `nms_threshold = 0.2` | `__main__`（全局变量） | 去重强度，密集物体场景可调到 0.4 防漏检 |
| `max_boxes_num = 50` | `__main__` | 单帧最多画多少个框 |
| `rgb888p_size=[320,320]` | `__main__` | 送 AI 分辨率；加大到 [640,640] 小物体检出更好但帧率下降（预处理仍是纯 resize，会保持 1:1） |
| `model_input_size=[320,320]` | 构造处 | 与 kmodel 绑定，不可改 |

## 5. 运行效果

- **屏幕画面**：镜头前的物体被彩色框标出——人（person）、杯子（cup）、瓶子（bottle）、手机（cell phone）、椅子（chair）、键盘（keyboard）等常见物品都能识别；框上方显示类别名和置信度（如 `cup 0.86`），不同类别颜色不同。
- **串口打印**：每帧耗时；取消 `print(res)` 注释可看到 `[x1,y1,x2,y2,score,类别id]` 数组。
- **成功现象**：把手机、水杯、书本等放到镜头前 20~50cm 处，1 秒内出框；320x320 的低分辨率下帧率较高、画面流畅。
- **失败现象排查**：
  - 模型加载失败 → 检查 `/sdcard/examples/kmodel/yolov8n_320.kmodel`；
  - 检不到物体 → 物体太小（320x320 输入对远处小目标不友好，靠近些）、光照暗、类别不在 COCO 80 类中（比如"充电宝"就检不出）；
  - 类别认错 → 小分辨率下相似物体（cup/bottle）易混，靠近拍摄、调高阈值；
  - 框偏少 → `nms_threshold=0.2` 较严，密集场景调到 0.4。

## 6. 代码使用说明

1. **硬件准备**：断电插好摄像头与 LCD 排线，插入 TF 卡，Type-C 上电。
2. **固件要求**：正点原子 **CanMV K230D 固件**（内置 libs、nncase_runtime 等），固件与 CanMV IDE 版本匹配。
3. **运行方式**：CanMV IDE 连接开发板 → 打开本目录 `main.py` → 运行，把 COCO 类别内的物体放到镜头前观察。
4. **模型文件需求**（从正点原子资料拷贝到 TF 卡）：
   - `/sdcard/examples/kmodel/yolov8n_320.kmodel` —— YOLOv8n 80 类物体检测模型（本实验唯一资源文件）。
5. **依赖库**：`libs.PipeLine / libs.AIBase / libs.AI2D`（板载 `/sdcard/app/libs/`）、固件内置 `ulab.numpy`、`image`、`media.sensor`、`nncase_runtime`（本实验后处理为纯 Python，未用到 aidemo/aicube 的检测接口）；不依赖本例程文件夹内其他文件。

## 7. 拓展思考

1. **入门级**：做"只认某类物体"的过滤器——在 `draw_result` 前判断 `int(det[5])` 是否等于目标类别 id（如 67=cell phone），只画它并把其他框忽略；配合蜂鸣器实现"手机靠近报警器"。
2. **进阶级**：统计画面中 person 数量并通过 UART 上报，或用框中心坐标控制云台转向目标；也可以把 `rgb888p_size` 提到 [640,640]，对比小物体检出率与帧率的取舍。
3. **挑战级**：阅读 `nms()` 的纯 Python 实现，把它与实验5 使用的 `aicube.anchorbasedet_post_process`（C 加速）做耗时对比（打开 `debug_mode=1`），体会"后处理为什么建议用 C 库"；再尝试把 NMS 换成"按类别分组再做 NMS"的多类版本。

## 8. 常见问题

**Q1：kmodel 加载失败怎么办？**
A：确认 `/sdcard/examples/kmodel/yolov8n_320.kmodel` 已拷贝到 TF 卡、文件名大小写一致、来自正点原子配套资料。

**Q2：为什么很多物体检不出来？**
A：先确认物体属于 COCO 80 类（看 `labels` 列表），不在表里的（如"路由器""充电宝"）模型天生不认识。在表里仍检不出：① 物体离镜头太远太小——320x320 输入下建议物体占画面 1/8 以上；② 光照不足；③ 把 `confidence_threshold` 降到 0.15 观察分数。

**Q3：改了 self.confidence_threshold 没生效？**
A：注意本文件的 `postprocess` 用的是**全局变量** `confidence_threshold`/`nms_threshold`（没有 `self.` 前缀），要改 `__main__` 顶部那两个全局变量的值才生效。

**Q4：帧率比实验5 高，正常吗？**
A：正常。本实验 `rgb888p_size=[320,320]` 远小于实验5 的 [640,360]，预处理数据量小；且模型输入同为 320/640 级别。数据通量小自然快。想要更准可以加大 `rgb888p_size`，帧率会相应下降。

**Q5：同一个物体偶尔出现两个不同类别的框？**
A：小分辨率下模型对相似类别（cup/bottle/vase 等）区分度下降，属正常现象。靠近拍摄、改善光照、调高 `confidence_threshold` 都可缓解。
