# 实验3 人脸姿态检测实验

> 检测每张人脸的朝向（俯仰、偏航、翻滚三个角度），并在屏幕上用红色 3D 立方体线框直观"套"在人脸上显示头部姿态。难度：中级。

## 1. 例程简介

本例程同样是**两级模型串联**：先用 `face_detection_320.kmodel` 检出人脸框，再用 `face_pose.kmodel` 估计该人脸的 3D 旋转姿态，输出一个旋转矩阵并换算成欧拉角（pitch/yaw/roll），最后在 LCD 上以人脸中心为原点画一个随头部转动而转动的红色立方体线框。你将学到：姿态估计原理、旋转矩阵与欧拉角、3D 顶点投影到 2D 画面的画法。难度为**中级**（涉及少量 3D 几何数学），预期用途：驾驶员头部姿态监测、课堂/会议专注度分析、人脸摆正（对齐）等。

## 2. 硬件连接与原理

用到的硬件：

- **CNK230DF 核心板（K230D）**：RISC-V 双核 + 6TOPS KPU，连续运行检测与姿态两个模型；
- **MIPI CSI 摄像头**：`Sensor(width=1280, height=960)`，送 AI 分辨率 `rgb888p_size=[1280,960]`；
- **2.4 寸 MIPI LCD**：`display_mode="lcd"`、`display_size=[640,480]`；
- **TF 卡（/sdcard）**：存放两个 kmodel 与锚框数据文件。

**姿态估计原理（通俗版）**：人脸姿态模型看一张摆正的 120x120 人脸小图，输出一个 3x3 的**旋转矩阵 R**。你可以把 R 想象成"这颗头相对于正对镜头的标准姿势，绕三个轴各转了多少"的数学描述。程序把 R 换算成更直观的**欧拉角**：pitch（点头/抬头）、yaw（左右摇头）、roll（歪头），单位是度。画立方体时，先在人脸上定义 8 个三维顶点（后面 4 个 + 前面 4 个），用 R 旋转它们，再加上人脸中心平移到画面上，就连成一个跟着头转的"立体框"——头往左转，立方体也往左转。

> [!WARNING]
> 新手风险提醒：
> - 摄像头/LCD 的 MIPI 排线必须**断电插拔**，插反通电可能烧坏器件；
> - 请使用足功率 Type-C 电源，供电不足会在双模型推理时导致重启；
> - 连续 AI 推理时核心板发热明显，**属正常现象**，注意别堵住散热面。

## 3. 相关前置知识

- **两级模型 / 级联推理**：第一级检脸给出框，第二级只看框内区域做姿态估计，原理同实验2。
- **仿射变换（affine）**：把原图里的人脸区域"抠出、放大、摆到正中"变成 120x120 模型输入图的几何变换。`get_affine_matrix()` 中缩放因子 `factor = 2.7`，表示取框最大边的 2.7 倍范围（多留背景），缩放比 `scale = edge_size * 2.0 / maximum_edge`。
- **旋转矩阵 R**：3x3 矩阵，描述三维空间中的一个旋转。模型输出张量的前 3x3 部分即 R（代码 `R = data[:3, :3].copy()`）。
- **欧拉角（pitch/yaw/roll）**：把旋转拆成三个直观角度——pitch 俯仰（点头）、yaw 偏航（摇头）、roll 翻滚（歪头）。`rotation_matrix_to_euler_angles()` 用 `arctan2` 从 R 反解出这三个角，并对 `sy < 1e-6` 的万向锁奇异情况做了特殊处理。
- **3D 投影**：把三维点 (x,y,z) 投到二维画面。本例程用简化投影：顶点经 R 旋转后取 x、y 分量（代码中 `sum_y` 用了 `-R[cc][1]` 翻转方向以匹配图像 y 轴向下的习惯），再加人脸中心坐标。
- **置信度阈值 / NMS 阈值 / 锚框 / IoU**：第一级人脸检测用，含义同实验1：`confidence_threshold=0.5`、`nms_threshold=0.2`、锚框文件 `prior_data_320.bin`（4200x4）。
- **kmodel / KPU / 张量 / 推理 / 帧率**：同实验1，`.kmodel` 是 K230 专用模型格式，KPU 是 AI 加速器，`ScopedTiming("total",1)` 打印每帧耗时。

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

与实验1/2 相同的框架，不赘述。

### 4.2 第一级：FaceDetApp（人脸检测）

与实验2的检测类基本一致：`config_preprocess()` 做 pad（补边色 104/117/123）+ resize 到 320x320；`postprocess()` 调 `aidemo.face_det_post_process` 得到人脸框列表。

### 4.3 第二级：FacePoseApp（姿态估计）

预处理——按人脸框计算仿射矩阵：

```python
def get_affine_matrix(self, bbox):
    factor = 2.7                              # 设置缩放因子
    x1, y1, w, h = map(lambda x: int(round(x, 0)), bbox[:4])
    edge_size = self.model_input_size[1]      # 模型输入大小
    trans_distance = edge_size / 2.0
    center_x = x1 + w / 2.0
    center_y = y1 + h / 2.0
    maximum_edge = factor * (h if h > w else w)
    scale = edge_size * 2.0 / maximum_edge
    cx = trans_distance - scale * center_x
    cy = trans_distance - scale * center_y
    affine_matrix = [scale, 0, cx, 0, scale, cy]
    return affine_matrix
```

后处理——从模型输出取旋转矩阵并换算欧拉角：

```python
def postprocess(self, results):
    R, eular = self.get_euler(results[0][0])
    return R, eular

def get_euler(self, data):
    R = data[:3, :3].copy()
    eular = self.rotation_matrix_to_euler_angles(R)
    return R, eular
```

### 4.4 调度类 FacePose

```python
def run(self, input_np):
    det_boxes = self.face_det.run(input_np)        # 人脸检测
    pose_res = []
    for det_box in det_boxes:
        self.face_pose.config_preprocess(det_box)  # 每张脸重新配置仿射
        R, eular = self.face_pose.run(input_np)
        pose_res.append((R, eular))
    return det_boxes, pose_res
```

### 4.5 绘制：3D 立方体投影

`build_projection_matrix(det)` 以人脸框中心为原点定义 8 个三维顶点：后平面 4 个点取框宽高的 0.5 倍、深度 0；前平面 4 个点取 `sqrt(2)` 倍、深度也取 `sqrt(2)*rear_width`，构成一个"前大后小"的立方体。

`draw_result()` 中对每个顶点做旋转 + 平移 + 越界裁剪：

```python
for pp in range(8):
    sum_x, sum_y = 0.0, 0.0
    for cc in range(3):
        sum_x += projections[pp][cc] * R[cc][0]
        sum_y += projections[pp][cc] * (-R[cc][1])
    x = (sum_x + center_x) / self.rgb888p_size[0] * self.display_size[0]
    y = (sum_y + center_y) / self.rgb888p_size[1] * self.display_size[1]
    x = max(0, min(x, self.display_size[0]))
    y = max(0, min(y, self.display_size[1]))
```

前 4 点与后 4 点分别用 `aidemo.polylines(..., True, ...)` 画成闭合四边形，再用 `draw_img.draw_line` 把前后对应的 4 对顶点连起来，得到红色立方体线框，最后 `pl.osd_img.copy_from(draw_img)` 上屏。

### 4.6 主流程与可调参数

```python
face_det_kmodel_path = "/sdcard/examples/kmodel/face_detection_320.kmodel"
face_pose_kmodel_path = "/sdcard/examples/kmodel/face_pose.kmodel"
anchors_path = "/sdcard/examples/utils/prior_data_320.bin"
rgb888p_size = [1280, 960]
face_det_input_size = [320, 320]
face_pose_input_size = [120, 120]
confidence_threshold = 0.5
nms_threshold = 0.2

sensor = Sensor(width=1280, height=960)
pl = PipeLine(rgb888p_size=rgb888p_size, display_size=display_size, display_mode=display_mode)
pl.create(sensor=sensor)
fp = FacePose(...)
while True:
    os.exitpoint()
    with ScopedTiming("total",1):
        img = pl.get_frame()
        det_boxes, pose_res = fp.run(img)
        fp.draw_result(pl, det_boxes, pose_res)
        pl.show_image()
        gc.collect()
```

| 参数 | 位置 | 影响 |
| --- | --- | --- |
| `confidence_threshold=0.5` / `nms_threshold=0.2` | `__main__` | 第一级检脸灵敏度，同实验1 |
| `factor = 2.7` | `FacePoseApp.get_affine_matrix()` | 抠脸范围倍数，太小会切掉部分脸导致姿态不准，太大脸占比过小 |
| `rgb888p_size=[1280,960]` | `__main__` | 送 AI 分辨率，降低可提帧率 |
| `face_pose_input_size=[120,120]` | `__main__` | 必须与 face_pose.kmodel 一致，不可改 |

> [!NOTE]
> 代码算出的欧拉角 `eular` 目前只存进 `pose_res`，并没有画到屏幕上；主循环里的 `print(det_boxes, pose_res)` 被注释了。想看角度数值，取消该行注释即可在串口打印。

## 5. 运行效果

- **屏幕画面**：每张人脸上套一个红色立方体线框。正对镜头时线框近乎正方形；左右摇头时线框跟着透视变形（一侧变窄）；抬头低头、歪头同样实时跟随。
- **串口打印**：`ScopedTiming("total",1)` 周期打印每帧耗时；取消注释 `print(det_boxes, pose_res)` 后可看到每张脸的旋转矩阵与 `[pitch, yaw, roll]` 角度值（度）。
- **成功现象**：缓慢转头时立方体平滑转动，无跳变；大角度侧脸（约 ±60° 内）仍能跟踪。
- **失败现象排查**：
  - 模型加载失败 → 核对 `/sdcard/examples/kmodel/` 下两个 kmodel 与 `/sdcard/examples/utils/prior_data_320.bin`；
  - 无线框 → 第一级没检到脸：光线、距离、阈值（降到 0.3 试试）；
  - 立方体乱转/抖动 → 侧脸角度过大、光照剧烈变化或分辨率不足；正对摄像头、保证光照均匀后再观察；
  - 角度打印全是怪值 → 确认没有改动 `face_pose_input_size`，且模型文件是配套的 `face_pose.kmodel`。

## 6. 代码使用说明

1. **硬件准备**：断电插好摄像头、LCD 排线，插入 TF 卡，Type-C 上电。
2. **固件要求**：正点原子 **CanMV K230D 固件**（内置 libs、aidemo、nncase_runtime 等），固件与 CanMV IDE 版本需匹配。
3. **运行方式**：CanMV IDE 连接开发板 → 打开本目录 `main.py` → 运行。
4. **模型文件需求**（从正点原子资料拷贝到 TF 卡）：
   - `/sdcard/examples/kmodel/face_detection_320.kmodel` —— 人脸检测模型；
   - `/sdcard/examples/kmodel/face_pose.kmodel` —— 人脸姿态模型；
   - `/sdcard/examples/utils/prior_data_320.bin` —— 锚框数据。
5. **依赖库**：`libs.PipeLine / libs.AIBase / libs.AI2D`（板载 `/sdcard/app/libs/`）、固件内置的 `aidemo`、`image`、`ulab.numpy`、`media.sensor`、`nncase_runtime`；不依赖本例程文件夹内其他文件。

## 7. 拓展思考

1. **入门级**：取消 `print(det_boxes, pose_res)` 注释，转头观察 pitch/yaw/roll 三个角度的变化规律，弄清哪个角对应点头、哪个对应摇头、哪个对应歪头。
2. **进阶级**：加"姿态报警"逻辑——当 |yaw| 或 |pitch| 超过设定角度（比如 30°）持续 2 秒时，点亮板载红色 LED（GPIO61）或驱动蜂鸣器，做一个"坐姿/专注度提醒器"。
3. **挑战级**：只保留正脸帧做后续处理——把本实验作为过滤器与实验4（人脸识别）串联：姿态角小于阈值才允许识别/注册，可显著提高识别准确率；或结合 UART 把姿态角实时上报 PC 做数据记录。

## 8. 常见问题

**Q1：kmodel 加载失败怎么办？**
A：检查 TF 卡 `/sdcard/examples/kmodel/` 下是否有 `face_detection_320.kmodel` 和 `face_pose.kmodel`，`/sdcard/examples/utils/` 下是否有 `prior_data_320.bin`；文件名大小写必须与代码一致；模型需来自正点原子配套资料（不同芯片平台的 kmodel 不通用）。

**Q2：立方体线框抖得厉害，正常吗？怎么改善？**
A：轻微抖动正常（模型输出本身有噪声，检测框也在逐帧微颤）。改善方法：① 提高光照、避免逆光；② 适当调高 `confidence_threshold`；③ 软件平滑：自己保存上一帧的 R，与本帧做加权平均（如 `R_new = 0.7*R_now + 0.3*R_last`）后再绘制。

**Q3：大角度侧脸时线框消失或错乱？**
A：属于模型能力边界。侧脸超过约 60° 后，检测模型本身难检到脸，姿态模型输出也不再可靠。应用上可先用欧拉角判断"是否正脸"，非正脸时不显示/不处理。

**Q4：帧率低是否正常？如何提速？**
A：正常。每帧 1 次检测 + N 次姿态推理，且 `rgb888p_size` 高达 1280x960。提速首选把 `rgb888p_size` 降到 [640,480]，检测与仿射都会更快，精度损失有限。

**Q5：为什么绘制时 y 方向要乘 `-R[cc][1]`？**
A：图像坐标系的 y 轴向下为正，而 3D 数学坐标系通常 y 轴向上为正，两者方向相反，所以投影时对 y 分量取负，立方体转向才与人头一致。若你发现立方体上下颠倒，多半是改动过这里。
