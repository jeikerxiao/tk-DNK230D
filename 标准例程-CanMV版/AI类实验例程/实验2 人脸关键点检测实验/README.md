# 实验2 人脸关键点检测实验

> 在人脸检测的基础上，进一步定位脸上的 106 个关键点，并按眉毛、眼睛、鼻子、嘴唇、脸廓等部位分色描出人脸轮廓。难度：初级。

## 1. 例程简介

本例程是一个**两级模型串联**的经典案例：先用 `face_detection_320.kmodel` 检出人脸框，再用 `face_landmark.kmodel` 在每张脸上定位关键点（五官点位），最后在 LCD 上按不同面部区域用不同颜色绘制轮廓线。你将学到：多模型协同推理、仿射变换（affine）预处理、关键点坐标的逆变换还原，以及 `aidemo` 绘图接口（polylines/contours）的使用。难度为**初级**（代码稍长，但框架与实验1一致），预期用途：美妆试戴、活体检测前置、表情分析等。

## 2. 硬件连接与原理

用到的硬件：

- **CNK230DF 核心板（K230D）**：双核 RISC-V CPU + 6TOPS KPU，本实验要连续跑两个模型，KPU 是流畅的关键；
- **MIPI CSI 摄像头**：`Sensor(width=1280, height=960)`，传感器输出 1280x960；送 AI 的分辨率为 `rgb888p_size=[1280,960]`；
- **2.4 寸 MIPI LCD**：`display_mode="lcd"`、`display_size=[640,480]`；
- **TF 卡（/sdcard）**：存放两个 kmodel 模型和锚框数据文件。

**两级推理原理（通俗版）**：整个流程像"先找到人，再仔细看脸"。第一级（人脸检测）在大图上快速找出所有人脸的矩形框；第二级（关键点）不是把整张图再算一遍，而是用**仿射变换**把每个人脸框区域"抠出来、摆正、放大"到 192x192 的小图，专门喂给关键点模型——这样模型只需专心看脸，既省算力又准。模型输出的是这张小图坐标系下的点，程序再用**逆仿射变换**把点位换算回原图坐标，才能画到屏幕上。

> [!WARNING]
> 新手风险提醒：
> - 摄像头 MIPI 排线**插反带电操作会烧坏硬件**，务必断电插拔、插到位再上电；
> - LCD 排线同理，断电操作；
> - 供电不足会导致双模型推理时重启，请使用规定功率的 Type-C 电源；
> - 两个模型连续推理时 KPU 发热比单模型更明显，**属正常现象**。

## 3. 相关前置知识

- **关键点（landmark）**：用一组固定编号的点描述人脸结构，比如"左眼角""鼻尖""嘴角"。本例程的模型输出 106 个点，代码中的 `dict_kp_seq` 把这些点按 10 个部位分组：左眉、右眉、左眼、右眼、瞳孔、鼻梁、鼻翼、外嘴唇、内嘴唇、脸廓（basin）。
- **两级模型 / 级联推理**：一个模型的输出（人脸框）作为下一个模型的输入范围。好处是第二个模型只需处理脸部小图，速度快、精度高。
- **仿射变换（affine）**：一种"平移 + 缩放（+旋转）"的几何变换，用一个 2x3 矩阵描述。`get_affine_matrix()` 根据人脸框算出这个矩阵，让人脸中心正好落在 192x192 输入图的正中央，缩放系数为 `model_input_size[0] / (max(w,h) * 1.5)`（1.5 表示框外多留一半的余量）。
- **逆仿射变换**：仿射矩阵求逆。`aidemo.invert_affine_transform()` 得到逆矩阵后，把模型输出的小图坐标逐点换算回摄像头大图的坐标。
- **kmodel 文件**：K230 专用模型格式，本实验有两个：检测模型输入 320x320，关键点模型输入 192x192。
- **置信度 / 阈值 / 锚框 / IoU / NMS**：与实验1相同——`confidence_threshold=0.5` 过滤低分框、`nms_threshold=0.2` 去重复框、锚框来自 `prior_data_320.bin`（4200 个 x 4 维）。
- **张量**：多维数组。注意 `ai2d.build([1,3,H,W], ...)` 描述的就是张量形状。
- **RGB888P**：一种平面存储的 RGB 图像格式（R、G、B 三个通道分开存），是 PipeLine 送给 AI 的数据格式。
- **帧率（FPS）**：每秒处理帧数。本实验跑两个模型，帧率会低于实验1，`ScopedTiming("total",1)` 可打印每帧耗时。

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

与实验1相同的例程框架：`PipeLine` 管采集显示，`AIBase` 管 KPU 推理，`Ai2d` 管硬件预处理，`aidemo` 提供 C 加速的后处理与绘图。

### 4.2 第一级：FaceDetApp（人脸检测类）

结构与实验1的 `FaceDetectionApp` 几乎一样：

- `config_preprocess()`：`pad`（补边色 104/117/123）+ `resize` 到 320x320；
- `postprocess()`：调用 `aidemo.face_det_post_process(...)` 完成阈值过滤 + NMS，返回人脸框列表。

### 4.3 第二级：FaceLandMarkApp（关键点类）

```python
def config_preprocess(self, det, input_image_size=None):
    with ScopedTiming("set preprocess config", self.debug_mode > 0):
        ai2d_input_size = input_image_size if input_image_size else self.rgb888p_size
        self.matrix_dst = self.get_affine_matrix(det)
        affine_matrix = [self.matrix_dst[0][0], self.matrix_dst[0][1], self.matrix_dst[0][2],
                         self.matrix_dst[1][0], self.matrix_dst[1][1], self.matrix_dst[1][2]]
        self.ai2d.affine(nn.interp_method.cv2_bilinear, 0, 0, 127, 1, affine_matrix)
        self.ai2d.build([1,3,ai2d_input_size[1],ai2d_input_size[0]],[1,3,self.model_input_size[1],self.model_input_size[0]])
```

每检出一张脸，就用该脸的框 `det` 重新配置一次仿射预处理（**每帧每张脸都要重新 config**，因为脸的位置在变）。`ai2d.affine` 让硬件直接把原图中的人脸区域变换成 192x192 的"摆正人脸图"。

后处理是坐标还原：

```python
def postprocess(self, results):
    pred = results[0]
    half_input_len = self.model_input_size[0] // 2
    pred = pred.flatten()
    for i in range(len(pred)):
        pred[i] += (pred[i] + 1) * half_input_len
    matrix_dst_inv = aidemo.invert_affine_transform(self.matrix_dst)
    ...
    for kp_id in range(half_out_len):
        new_x = old_x * matrix_dst_inv[0] + old_y * matrix_dst_inv[1] + matrix_dst_inv[2]
        new_y = old_x * matrix_dst_inv[3] + old_y * matrix_dst_inv[4] + matrix_dst_inv[5]
```

分三步：① 把模型输出的归一化坐标展开成像素坐标；② 求仿射矩阵的逆；③ 每个点乘逆矩阵，从"192x192 小图坐标"变回"原图坐标"。

### 4.4 调度类 FaceLandMark

```python
def run(self, input_np):
    det_boxes = self.face_det.run(input_np)     # 第一级：检人脸
    landmark_res = []
    for det_box in det_boxes:
        self.face_landmark.config_preprocess(det_box)   # 按框配置仿射
        res = self.face_landmark.run(input_np)          # 第二级：推关键点
        landmark_res.append(res)
    return det_boxes, landmark_res
```

构造函数里还定义了 `dict_kp_seq`（106 点按 10 个部位的分组索引）和 `color_list_for_osd_kp`（每个部位一种 ARGB 颜色）。

### 4.5 绘制：分部位描轮廓

`draw_result()` 新建一张与屏幕同尺寸的 ARGB8888 numpy 画布，遍历每个部位：

```python
if sub_part_index in (9, 6):        # 脸廓和鼻翼用闭合折线
    aidemo.polylines(draw_img_np, face_sub_part_point_set, False, color, 5, 8, 0)
elif sub_part_index == 4:           # 瞳孔直接画小圆圈
    draw_img.draw_circle(x, y, 2, color, 1)
else:                               # 其余部位画轮廓线
    aidemo.contours(draw_img_np, face_sub_part_point_set, -1, color, 2, 8)
```

画完通过 `pl.osd_img.copy_from(draw_img)` 叠加到显示层。

### 4.6 主流程与可调参数

```python
face_det_kmodel_path = "/sdcard/examples/kmodel/face_detection_320.kmodel"
face_landmark_kmodel_path = "/sdcard/examples/kmodel/face_landmark.kmodel"
anchors_path = "/sdcard/examples/utils/prior_data_320.bin"
rgb888p_size = [1280, 960]
face_det_input_size = [320, 320]
face_landmark_input_size = [192, 192]
confidence_threshold = 0.5
nms_threshold = 0.2

sensor = Sensor(width=1280, height=960)
pl = PipeLine(rgb888p_size=rgb888p_size, display_size=display_size, display_mode=display_mode)
pl.create(sensor=sensor)
flm = FaceLandMark(...)
while True:
    os.exitpoint()
    with ScopedTiming("total",1):
        img = pl.get_frame()
        det_boxes, landmark_res = flm.run(img)
        flm.draw_result(pl, det_boxes, landmark_res)
        pl.show_image()
        gc.collect()
```

| 参数 | 位置 | 影响 |
| --- | --- | --- |
| `confidence_threshold=0.5` / `nms_threshold=0.2` | `__main__` | 影响第一级检脸，同实验1 |
| `rgb888p_size=[1280,960]` | `__main__` | 送 AI 的分辨率，越大细节越好但越慢 |
| `face_landmark_input_size=[192,192]` | `__main__` | 关键点模型输入，须与 kmodel 一致，不可改 |
| `scale_ratio` 中的 `* 1.5` | `get_affine_matrix()` | 抠脸时留边余量，改大则脸周围背景更多 |

> [!NOTE]
> `finally` 中要分别反初始化两个模型：`flm.face_det.deinit()`、`flm.face_landmark.deinit()`，再 `pl.destroy()`。自己改代码时别漏掉。

## 5. 运行效果

- **屏幕画面**：摄像头实时画面上，每张人脸被描出彩色轮廓——眉毛绿色、眼睛黄蓝色、瞳孔红点、鼻子橙黄色、嘴唇粉/黄色、脸廓红色，多张脸同时生效；转头时轮廓跟随面部移动。
- **串口打印**：`ScopedTiming("total",1)` 周期性打印每帧耗时；取消 `print(det_boxes, landmark_res)` 注释可看到人脸框和 106 点坐标数组。
- **成功现象**：轮廓线紧贴五官、抖动小；侧脸时点位仍大致贴合。
- **失败现象排查**：
  - 模型加载失败 → 检查两个 kmodel 与 `prior_data_320.bin` 是否已拷入 TF 卡对应路径；
  - 完全无轮廓 → 先看第一级是否检到脸（光线、距离、阈值）；检不到脸就没有关键点；
  - 轮廓整体错位 → 确认 `rgb888p_size` 与 `Sensor` 分辨率比例关系未被改乱，`display_size` 换算在 `draw_result` 中自动完成；
  - 帧率低 → 本实验双模型 + 1280x960 输入，明显低于实验1属正常。

## 6. 代码使用说明

1. **硬件准备**：断电插好摄像头与 LCD 排线，插入 TF 卡，再接 Type-C 电源。
2. **固件要求**：烧录正点原子 **CanMV K230D 固件**（内置 `libs`、`aidemo`、`nncase_runtime` 等模块），固件与 CanMV IDE 版本需匹配（用资料配套版本最稳）。
3. **运行方式**：CanMV IDE 连接开发板 → 打开本目录 `main.py` → 点运行。
4. **模型文件需求**（从正点原子资料拷贝到 TF 卡）：
   - `/sdcard/examples/kmodel/face_detection_320.kmodel` —— 人脸检测模型；
   - `/sdcard/examples/kmodel/face_landmark.kmodel` —— 人脸关键点模型；
   - `/sdcard/examples/utils/prior_data_320.bin` —— 锚框数据。
5. **依赖库**：`libs.PipeLine / libs.AIBase / libs.AI2D`（位于板载 `/sdcard/app/libs/`），`aidemo`、`image`、`ulab`、`media` 为固件内置；不依赖本例程文件夹内其他文件。

## 7. 拓展思考

1. **入门级**：修改 `color_list_for_osd_kp` 给各部位换颜色；或只绘制眼睛和嘴唇两个部位，观察代码中 `dict_kp_seq` 索引与部位的对应关系。
2. **进阶级**：利用左右嘴角、上下嘴唇关键点计算**张嘴程度**（点间距），做一个"打哈欠检测/张嘴计数"；同理可用左右眼关键点做"眨眼计数"。
3. **挑战级**：把瞳孔关键点作为注视方向估计，做一个"眼神跟随"的互动效果；或结合实验3（人脸姿态）过滤侧脸，只在正脸时绘制轮廓。

## 8. 常见问题

**Q1：kmodel 加载失败怎么办？**
A：逐项检查：① TF 卡中 `/sdcard/examples/kmodel/` 下是否同时存在 `face_detection_320.kmodel` 和 `face_landmark.kmodel`；② 文件名大小写是否与代码完全一致；③ 模型是否为 K230D 配套版本（用正点原子资料里的，不要用 K210/K230 其他开发板的模型）。

**Q2：本实验帧率比实验1低很多，正常吗？**
A：正常。本实验每帧要跑 1 次检测 + N 次关键点推理（N 为人脸数），且送 AI 的图像分辨率高达 1280x960。人脸越多越慢。想提速可把 `rgb888p_size` 降到如 [640,480]，精度略降但流畅度明显改善。

**Q3：关键点轮廓抖动、不稳定怎么调？**
A：抖动主要来自检测框逐帧微颤导致仿射变换跟着颤。可以：① 适当调高 `confidence_threshold`（如 0.6）过滤低质量框；② 保证光照充足；③ 进阶做法是自己加"平滑"逻辑——把当前帧关键点与上一帧做加权平均后再绘制。

**Q4：为什么每张脸都要重新调用 `config_preprocess(det_box)`？**
A：因为仿射矩阵是由当前这张脸的框位置算出来的，脸动了矩阵就得重算。这与实验1只在开头 config 一次不同——实验1的 pad/resize 参数固定，而这里的 affine 参数逐脸变化。

**Q5：画面有脸但一个关键点都没画出来？**
A：说明第一级检到了脸、第二级输出异常。先确认 `face_landmark.kmodel` 路径正确；再取消主循环里 `print(det_boxes, landmark_res)` 的注释，看 landmark_res 是否为空或数值异常；最后检查是否改动过 `face_landmark_input_size`（必须保持 [192,192] 与模型匹配）。
