# 实验10 手掌关键点检测实验

> 先检测手掌，再定位每只手的 21 个关键点（手腕 + 每根手指 4 个关节点），用五种颜色的线条画出"手指骨架"。难度：中级。

## 1. 例程简介

本例程是两级模型：第一级 `hand_det.kmodel` 检出手掌框（同实验8/9），第二级 `handkp_det.kmodel`（256x256 输入）对裁剪出的手掌区域回归 **21 个关键点**的坐标，随后在 LCD 上画出关键点圆点，并把 5 根手指分别用红、品红、黄、绿、蓝 5 种颜色的折线连成"手骨架"。你将学到：关键点回归模型的后处理（归一化坐标 → crop 区域坐标 → 显示坐标的三层换算）、手指骨架拓扑结构。难度为**中级**，预期用途：手指计数、动态手势（实验12）、指哪打哪的"隔空触摸"、手语识别的基础。

## 2. 硬件连接与原理

用到的硬件：

- **CNK230DF 核心板（K230D）**：RISC-V 双核 + 6TOPS KPU；
- **MIPI CSI 摄像头**：`Sensor(width=1280, height=960)`；送 AI 分辨率 `rgb888p_size=[1280,960]`；
- **2.4 寸 MIPI LCD**：`display_mode="lcd"`、`display_size=[640,480]`；
- **TF 卡（/sdcard）**：存放两个 kmodel 模型（锚框写在代码中）。

**手部关键点原理（通俗版）**：关键点模型看一张 256x256 的手掌裁剪图，直接"报出" 21 个点的相对位置（0~1 的比例值）。21 个点的布局是：**第 0 号点是手腕**；随后每根手指 4 个点，按"拇指、食指、中指、无名指、小指"顺序排列（点 1~4 拇指、5~8 食指、9~12 中指、13~16 无名指、17~20 小指）。绘制时把每根手指的 4 个点依次连线，再从手腕连到每根手指的根关节，就得到完整手骨架。

**坐标三层换算**：模型输出的是裁剪小图内的归一化坐标（0~1），后处理要：① 乘 crop 区域的宽高、加 crop 原点 → 还原到摄像头原图坐标；② 再按 `display_size / rgb888p_size` 比例 → 屏幕坐标。这样画出的点才能精确落在屏幕上的手上。

> [!WARNING]
> - 摄像头/LCD 的 MIPI 排线必须**断电插拔**，插反通电会烧坏器件；
> - 供电不足会导致双模型推理时重启，请使用足功率 Type-C 电源；
> - AI 推理发热明显，**属正常现象**；
> - 1280x960 送 AI 分辨率较大，帧率偏低属正常，不要误判为死机。

## 3. 相关前置知识

- **关键点（keypoint）回归**：模型直接输出每个点的坐标，而不是分类分数。21 个点即 42 个输出数值（x、y 交替）。
- **归一化坐标**：输出值在 0~1 之间，表示点在输入图中的相对位置，乘上实际宽高才是像素坐标。
- **ROI / crop**：`ai2d.crop(x,y,w,h)` 硬件裁剪。`get_crop_param()` 以检测框中心向外扩 `1.26 * max(w,h)/2` 得到正方形裁剪区，且用 max/min 限制在画面内。
- **置信度 / 阈值 / IoU / NMS / 锚框 / strides**：第一级检测用：`confidence_threshold=0.2`、`nms_threshold=0.5`、`strides=[8,16,32]`、9 对手掌锚框写在代码中。
- **kmodel / KPU / 张量 / 推理**：两模型均为 `.kmodel` 格式，KPU 加速。
- **pad 补边色 114**：第一级预处理补灰边 [114,114,114]，与模型训练一致。
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

与实验9 的检测类完全同构：pad(114 灰边) + resize 到 512x512，后处理 `aicube.anchorbasedet_post_process(...)`，`labels=["hand"]`。

### 4.3 第二级：HandKPDetApp（关键点）

预处理——按框裁剪：

```python
def config_preprocess(self, det, input_image_size=None):
    ai2d_input_size = input_image_size if input_image_size else self.rgb888p_size
    self.crop_params = self.get_crop_param(det)
    self.ai2d.crop(self.crop_params[0], self.crop_params[1], self.crop_params[2], self.crop_params[3])
    self.ai2d.resize(nn.interp_method.tf_bilinear, nn.interp_mode.half_pixel)
    self.ai2d.build([1,3,ai2d_input_size[1],ai2d_input_size[0]],[1,3,self.model_input_size[1],self.model_input_size[0]])
```

后处理——坐标三层换算（关键代码）：

```python
def postprocess(self, results):
    results = results[0].reshape(results[0].shape[0]*results[0].shape[1])   # 展平成42个数
    results_show = np.zeros(results.shape, dtype=np.int16)
    results_show[0::2] = results[0::2] * self.crop_params[3] + self.crop_params[0]   # x: 归一化->crop区域->原图
    results_show[1::2] = results[1::2] * self.crop_params[2] + self.crop_params[1]   # y: 同理
    results_show[0::2] = results_show[0::2] * (self.display_size[0] / self.rgb888p_size[0])  # 原图->屏幕
    results_show[1::2] = results_show[1::2] * (self.display_size[1] / self.rgb888p_size[1])
    return results_show
```

`results_show[0::2]` 表示所有偶数下标（x 坐标），`[1::2]` 是所有奇数下标（y 坐标），一次切片运算完成 21 个点的换算。返回的数组直接就是**屏幕坐标**，绘制时不需要再换算。

> [!NOTE]
> 细看代码会发现 x 乘的是 `crop_params[3]`（crop 高）、y 乘的是 `crop_params[2]`（crop 宽）——因为裁剪区是正方形（`get_crop_param` 保证 w_kp≈h_kp），宽高几乎相等，这样写不影响效果；若你改动裁剪逻辑使宽高不等，需要留意这里的对应关系。

### 4.4 调度类 HandKeyPointDet

```python
def run(self, input_np):
    det_boxes = self.hand_det.run(input_np)     # 手掌检测
    hand_res = []
    boxes = []
    for det_box in det_boxes:
        x1, y1, x2, y2 = det_box[2], det_box[3], det_box[4], det_box[5]
        # 丢弃不合理的框（太小/贴边，三条规则同实验8/9）
        ...
        self.hand_kp.config_preprocess(det_box)
        results_show = self.hand_kp.run(input_np)
        boxes.append(det_box)
        hand_res.append(results_show)
    return boxes, hand_res
```

### 4.5 绘制：21 点 + 5 指分色骨架

```python
for i in range(len(results_show)/2):        # 画21个关键点小圆圈
    pl.osd_img.draw_circle(results_show[i*2], results_show[i*2+1], 1, color=(255,0,255,0), fill=False)
for i in range(5):                          # 5根手指各一种颜色
    j = i*8
    if i==0: R=255; G=0;   B=0              # 拇指-红
    if i==1: R=255; G=0;   B=255            # 食指-品红
    if i==2: R=255; G=255; B=0              # 中指-黄
    if i==3: R=0;   G=255; B=0              # 无名指-绿
    if i==4: R=0;   G=0;   B=255            # 小指-蓝
    pl.osd_img.draw_line(results_show[0], results_show[1], results_show[j+2], results_show[j+3], color=(255,R,G,B), thickness=3)
    pl.osd_img.draw_line(results_show[j+2], results_show[j+3], results_show[j+4], results_show[j+5], color=(255,R,G,B), thickness=3)
    pl.osd_img.draw_line(results_show[j+4], results_show[j+5], results_show[j+6], results_show[j+7], color=(255,R,G,B), thickness=3)
    pl.osd_img.draw_line(results_show[j+6], results_show[j+7], results_show[j+8], results_show[j+9], color=(255,R,G,B), thickness=3)
```

每根手指画 4 段线：**手腕 → 指根关节 → 第二节 → 第三节 → 指尖**。数组下标换算：点 k 的坐标存放在 `results_show[2k]`、`results_show[2k+1]`，`j = i*8` 即第 i 根手指第一个关节点（点号 i*4+1）的 x 下标。

### 4.6 主流程

```python
hand_det_kmodel_path = "/sdcard/examples/kmodel/hand_det.kmodel"
hand_kp_kmodel_path = "/sdcard/examples/kmodel/handkp_det.kmodel"
anchors_path = "/sdcard/examples/utils/prior_data_320.bin"   # 定义了但未实际加载，锚框写在下面
rgb888p_size = [1280, 960]
hand_det_input_size = [512, 512]
hand_kp_input_size = [256, 256]
confidence_threshold = 0.2
nms_threshold = 0.5
labels = ["hand"]
anchors = [26,27, 53,52, 75,71, 80,99, 106,82, 99,134, 140,113, 161,172, 245,276]

sensor = Sensor(width=1280, height=960)
pl = PipeLine(rgb888p_size=rgb888p_size, display_size=display_size, display_mode=display_mode)
pl.create(sensor=sensor)
hkd = HandKeyPointDet(...)
while True:
    os.exitpoint()
    with ScopedTiming("total",1):
        img = pl.get_frame()
        det_boxes, hand_res = hkd.run(img)
        hkd.draw_result(pl, det_boxes, hand_res)
        pl.show_image()
        gc.collect()
```

### 4.7 可调参数一览

| 参数 | 位置 | 影响 |
| --- | --- | --- |
| `confidence_threshold=0.2` / `nms_threshold=0.5` | `__main__` | 第一级检手灵敏度 |
| `ratio_num = 1.26*length` | `HandKPDetApp.get_crop_param()` | 裁剪余量，需与关键点模型训练一致，不建议改 |
| `rgb888p_size=[1280,960]` | `__main__` | 送 AI 分辨率；降到 [640,480] 提速明显 |
| `hand_kp_input_size=[256,256]` | `__main__` | 与模型绑定，不可改 |
| 手指颜色 R/G/B 赋值 | `draw_result()` | 自定义五指配色 |

## 5. 运行效果

- **屏幕画面**：每只手上有紫框，手内部覆盖 21 个绿色小圆点和 5 种颜色的手指骨架线；弯曲手指时对应颜色的骨架跟着弯；双手入镜各自成骨架。
- **串口打印**：每帧耗时；取消 `print(det_boxes, hand_res)` 注释可看到框数据和 42 个屏幕坐标值。
- **成功现象**：五指分开、手背正对摄像头时，21 点贴合关节位置，骨架线沿手指走向。
- **失败现象排查**：
  - 模型加载失败 → 核对 `hand_det.kmodel` 与 `handkp_det.kmodel` 路径；
  - 关键点整体错位 → 是否改动过 `rgb888p_size` 或 `display_size` 而未同步？后处理已按比例换算，一般只需恢复默认；
  - 点乱飞 → 手太小（远离摄像头）、光线暗、手指并拢互相遮挡；
  - 只画框没有骨架 → 第二级模型未加载成功，检查文件名。

## 6. 代码使用说明

1. **硬件准备**：断电插好摄像头与 LCD 排线，插入 TF 卡，Type-C 上电。
2. **固件要求**：正点原子 **CanMV K230D 固件**（内置 libs、aicube、nncase_runtime 等），固件与 CanMV IDE 版本匹配。
3. **运行方式**：CanMV IDE 连接开发板 → 打开本目录 `main.py` → 运行，手距摄像头 20~50cm、五指自然分开效果最佳。
4. **模型文件需求**（从正点原子资料拷贝到 TF 卡）：
   - `/sdcard/examples/kmodel/hand_det.kmodel` —— 手掌检测模型；
   - `/sdcard/examples/kmodel/handkp_det.kmodel` —— 手掌关键点模型；
   - 代码中定义了 `anchors_path="/sdcard/examples/utils/prior_data_320.bin"` 但**未实际加载**（锚框写在代码里），该文件非必需。
5. **依赖库**：`libs.PipeLine / libs.AIBase / libs.AI2D`（板载 `/sdcard/app/libs/`）、固件内置 `aicube`、`ulab.numpy`、`image`、`media.sensor`、`nncase_runtime`；不依赖本例程文件夹内其他文件。

## 7. 拓展思考

1. **入门级**：**手指计数**——判断每根手指"指尖到手腕的距离"是否明显大于"指根到手腕的距离"（伸直状态），统计伸直手指数量并显示在屏幕上，做出"隔空数字 0~5"。
2. **进阶级**：**隔空触摸**——取食指尖坐标（点 8 的 x,y）作为"虚拟鼠标"，当指尖在某个屏幕区域停留 0.5 秒时触发事件（点亮 LED GPIO59 / 蜂鸣器响），实现无接触按钮。
3. **挑战级**：把本实验的关键点数据按帧缓存，分析连续多帧指尖运动方向，实现"挥手翻页""画圈切换"等**动态手势**（对照实验12 的思路，理解它是如何用关键点序列做时序分类的）。

## 8. 常见问题

**Q1：kmodel 加载失败怎么办？**
A：确认 TF 卡 `/sdcard/examples/kmodel/` 下同时存在 `hand_det.kmodel` 和 `handkp_det.kmodel`，大小写一致，来自正点原子配套资料；其他平台的模型不能混用。

**Q2：关键点抖动厉害，正常吗？**
A：轻微抖动正常（模型逐帧回归，本身有噪声）。手离摄像头越远、抖动越明显。改善：① 手靠近些让手掌占更多像素；② 光照充足；③ 进阶做跨帧平滑（当前点 = 0.6*本帧 + 0.4*上帧）。

**Q3：手指弯曲或并拢时关键点错乱？**
A：属于模型能力边界：手指互相遮挡时回归精度下降。应用时尽量五指分开；做手指计数等逻辑时要容忍个别帧的错误，用多帧投票。

**Q4：帧率低是否正常？**
A：正常。每帧 1 次 512x512 检测 + 每只手一次 256x256 关键点推理，且 `rgb888p_size=[1280,960]`。提速优先把 `rgb888p_size` 降到 [640,480]。

**Q5：为什么绘制坐标不需要再从 rgb888p 换算到 display？**
A：因为 `HandKPDetApp.postprocess` 已经把三层换算全部做完（归一化 → crop 区域 → 原图 → 屏幕），返回的 `results_show` 就是屏幕像素坐标。这与实验2 在绘制时才换算的做法不同，改代码时别重复换算，否则点位会偏。
