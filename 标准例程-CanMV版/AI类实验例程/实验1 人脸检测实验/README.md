# 实验1 人脸检测实验

> 用 K230D 的 KPU（AI 加速器）跑一个人脸检测模型，实时框出摄像头画面里的每一张人脸。这是本套 AI 例程的"Hello World"，难度：初级。

## 1. 例程简介

本例程通过摄像头采集实时画面，送入 KPU 运行人脸检测模型 `face_detection_320.kmodel`，把画面中所有人脸用黄色矩形框标出来并显示在 LCD 屏幕上。你将学到 CanMV K230D 上 AI 例程的通用框架：`PipeLine`（图像采集与显示流水线）+ `AIBase`（KPU 推理基类）+ `Ai2d`（硬件预处理）+ 后处理（阈值过滤与 NMS）。难度为**初级**，是所有后续 AI 实验（人脸关键点、人脸识别、人体检测等）的基础模板，适合作为第一个上手的 AI 例程。

## 2. 硬件连接与原理

用到的硬件：

- **CNK230DF 核心板（K230D）**：RISC-V 双核 CPU + 6TOPS 算力的 KPU（神经网络加速单元）；
- **MIPI CSI 摄像头**：代码中通过 `Sensor(width=1280, height=960)` 构建摄像头对象，即传感器输出 1280x960 分辨率的图像；
- **2.4 寸 MIPI LCD**：代码中 `display_mode="lcd"`、`display_size=[640,480]`，检测结果叠加显示在屏幕上；
- **TF 卡（挂载为 /sdcard）**：存放模型文件 `face_detection_320.kmodel` 和锚框数据 `prior_data_320.bin`。

**KPU 推理原理（通俗版）**：摄像头每拍一帧照片，程序先把照片"整理"成神经网络认识的格式（缩放到 320x320、补边），然后交给 KPU。KPU 内部运行的是一个提前训练好的卷积神经网络，它把整张图划分成密密麻麻的小格子，每个格子回答一个问题："我这里有没有可能是人脸？有多大概率？框应该画多大？"程序再把这些回答汇总，扔掉概率低的，合并重复的，最后得到几个确定的人脸框。

> [!WARNING]
> 新手风险提醒：
> - **摄像头 MIPI 排线插反或没插到位就通电，可能烧坏摄像头或核心板**，务必断电操作、确认金手指方向；
> - 所有插拔（摄像头、LCD、TF 卡）都必须**先断电**；
> - 请用足功率的 Type-C 电源供电，供电不足会导致 AI 推理时死机或重启；
> - AI 推理时核心板**发热明显属于正常现象**，KPU 满载工作必然发热，不必惊慌。

## 3. 相关前置知识

- **神经网络**：一种模仿人脑神经元连接方式的数学模型。给它看几十万张"有人脸/没人脸"的图片并不断纠错（训练），它就学会了"看到什么样的像素组合算人脸"。
- **模型 / kmodel 文件**：训练好的神经网络保存成的文件。`.kmodel` 是嘉楠 K230 系列专用的模型格式，由 nncase 工具链把常规模型转换而来，KPU 只能运行这种格式。本例程的模型是 `/sdcard/examples/kmodel/face_detection_320.kmodel`，文件名中的 320 表示模型输入是 320x320 的图。
- **KPU**：K230D 芯片里专门算神经网络的硬件加速器，算力 6TOPS。用它跑模型比用 CPU 快几十倍，且更省电。
- **张量（tensor）**：就是多维数组。一张 320x320 的彩色图在模型里是形状 `[1,3,320,320]` 的张量：1 表示一批一张图，3 表示红绿蓝三个通道。代码中 `self.ai2d.build([1,3,...],[1,3,...])` 里的两组数字就是在描述输入、输出张量的形状。
- **推理（inference）**：把一张图喂给训练好的模型、让它给出答案的过程。对应代码里的 `face_det.run(img)`。
- **置信度（confidence）**：模型对每个候选框给出的"我有多确定这是人脸"的分数，范围 0~1。
- **阈值（threshold）**：过滤线。代码中 `confidence_threshold = 0.5`，即置信度低于 0.5 的候选框直接丢弃。
- **锚框（anchor）**：一组预设的"标准框尺寸"。模型不直接凭空画框，而是在每个格子上按这些标准尺寸微调出实际的框。本例程的锚框从二进制文件加载：`anchors = np.fromfile("/sdcard/examples/utils/prior_data_320.bin", dtype=np.float)`，共 `anchor_len = 4200` 个、每个 `det_dim = 4` 个数（中心 x、中心 y、宽、高）。
- **IoU（交并比）**：两个框重叠程度的度量 = 交集面积 / 并集面积，范围 0~1。IoU 越大说明两个框越"重复"。
- **NMS（非极大值抑制）**：同一个脸往往被多个格子同时框住，产生一堆几乎重叠的框。NMS 的做法是：留下置信度最高的那个框，把与它 IoU 超过 `nms_threshold = 0.2` 的其他框全部删掉。
- **帧率（FPS）**：每秒处理并显示多少帧画面。代码中 `with ScopedTiming("total",1)` 会在串口打印每帧总耗时，可用来估算帧率。

## 4. 软件代码逻辑说明

### 4.1 导入例程框架库

```python
from libs.PipeLine import PipeLine, ScopedTiming
from libs.AIBase import AIBase
from libs.AI2D import Ai2d
import nncase_runtime as nn
import ulab.numpy as np
import aidemo
```

- `libs.PipeLine`：官方例程库（位于开发板 `/sdcard/app/libs/`），封装了"摄像头采集 → VO 显示"的完整通路；`ScopedTiming` 是计时小工具；
- `libs.AIBase`：AI 推理基类，封装了 kmodel 加载、输入输出内存分配、`run()` 执行流程，子类只需实现 `postprocess()`；
- `libs.AI2D`：Ai2d 是 K230D 的**硬件图像预处理加速器**，支持 crop/shift/pad/resize/affine，比用 CPU 做预处理快得多；
- `nncase_runtime`（别名 `nn`）：KPU 运行时接口；`ulab.numpy`（别名 `np`）：MicroPython 版的 numpy；
- `aidemo`：嘉楠提供的 C 加速后处理库，人脸检测后处理就在里面。

### 4.2 自定义人脸检测类 FaceDetectionApp

```python
class FaceDetectionApp(AIBase):
    def __init__(self, kmodel_path, model_input_size, anchors, confidence_threshold=0.5,
                 nms_threshold=0.2, rgb888p_size=[224,224], display_size=[1920,1080], debug_mode=0):
        super().__init__(kmodel_path, model_input_size, rgb888p_size, debug_mode)
        ...
        self.ai2d = Ai2d(debug_mode)
        self.ai2d.set_ai2d_dtype(nn.ai2d_format.NCHW_FMT, nn.ai2d_format.NCHW_FMT, np.uint8, np.uint8)
```

构造函数把模型路径、模型输入尺寸、锚框、两个阈值都存下来，并实例化 `Ai2d`、设置输入输出均为 NCHW 格式的 uint8 数据。

### 4.3 配置预处理（pad + resize）

```python
def config_preprocess(self, input_image_size=None):
    with ScopedTiming("set preprocess config", self.debug_mode > 0):
        ai2d_input_size = input_image_size if input_image_size else self.rgb888p_size
        top, bottom, left, right = self.get_padding_param()
        self.ai2d.pad([0, 0, 0, 0, top, bottom, left, right], 0, [104, 117, 123])
        self.ai2d.resize(nn.interp_method.tf_bilinear, nn.interp_mode.half_pixel)
        self.ai2d.build([1,3,ai2d_input_size[1],ai2d_input_size[0]],[1,3,self.model_input_size[1],self.model_input_size[0]])
```

`get_padding_param()` 先算等比例缩放后剩下的空白，再用 `pad` 在右侧和下侧补上灰蓝色边（RGB 值 104/117/123），最后 `resize` 到模型的 320x320。这样图像不会被拉伸变形，检测精度更高。

### 4.4 后处理（阈值过滤 + NMS）

```python
def postprocess(self, results):
    with ScopedTiming("postprocess", self.debug_mode > 0):
        post_ret = aidemo.face_det_post_process(self.confidence_threshold, self.nms_threshold,
            self.model_input_size[1], self.anchors, self.rgb888p_size, results)
        if len(post_ret) == 0:
            return post_ret
        else:
            return post_ret[0]
```

模型原始输出（`results`）是一大堆候选框数据，交给 C 库 `aidemo.face_det_post_process` 一次性完成置信度过滤、锚框解码、NMS，返回最终的人脸框列表，每个框是 `[x, y, w, h, ...]`（左上角坐标 + 宽高，位于 rgb888p 图像坐标系下）。

### 4.5 绘制结果

```python
pl.osd_img.draw_rectangle(x, y, w, h, color=(255, 255, 0, 255), thickness=2)
```

`draw_result()` 把框坐标从 `rgb888p_size`（640x360）等比换算到 `display_size`（640x480），再画到 `PipeLine` 的 OSD 叠加层上，颜色为黄色、线宽 2。

### 4.6 主流程

```python
kmodel_path = "/sdcard/examples/kmodel/face_detection_320.kmodel"
confidence_threshold = 0.5
nms_threshold = 0.2
anchors_path = "/sdcard/examples/utils/prior_data_320.bin"
anchors = np.fromfile(anchors_path, dtype=np.float)
anchors = anchors.reshape((anchor_len, det_dim))
rgb888p_size = [640, 360]

sensor = Sensor(width=1280, height=960)   # 构建摄像头对象
pl = PipeLine(rgb888p_size=rgb888p_size, display_size=display_size, display_mode=display_mode)
pl.create(sensor=sensor)                  # 创建PipeLine实例
face_det = FaceDetectionApp(kmodel_path, model_input_size=[320, 320], anchors=anchors, ...)
face_det.config_preprocess()              # 配置预处理

while True:
    os.exitpoint()                        # 检查是否有退出信号
    with ScopedTiming("total",1):
        img = pl.get_frame()              # 获取当前帧数据
        res = face_det.run(img)           # 推理当前帧
        face_det.draw_result(pl, res)     # 绘制结果
        pl.show_image()                   # 显示结果
        gc.collect()                      # 垃圾回收
```

循环里每一帧都经历：取图 → 预处理（在 `run` 内由 Ai2d 自动完成）→ KPU 推理 → 后处理 → 画框 → 上屏。`finally` 中调用 `face_det.deinit()` 和 `pl.destroy()` 释放资源。

### 4.7 可调参数一览

| 参数 | 代码位置 | 修改后的影响 |
| --- | --- | --- |
| `confidence_threshold = 0.5` | `__main__` 开头 | 调低（如 0.3）更容易检出侧脸/小脸，但误检增多；调高则只框很确定的正脸 |
| `nms_threshold = 0.2` | `__main__` 开头 | 调低去重更狠，可能漏掉贴近的两张脸；调高则同一张脸可能出现双框 |
| `rgb888p_size = [640, 360]` | `__main__` 开头 | 送给 AI 的图更大则小目标检出更好，但推理更慢 |
| `model_input_size=[320, 320]` | 构造 `FaceDetectionApp` 处 | **不能随意改**，必须与 kmodel 编译时的输入尺寸一致 |
| `display_size=[640,480]` | `__main__` 开头 | LCD 显示分辨率 |

## 5. 运行效果

- **屏幕画面**：LCD 显示摄像头实时画面，每张被检出的人脸上套一个黄色矩形框；人走动时框实时跟随；无人的画面没有框。
- **串口打印**：`ScopedTiming("total",1)` 会周期性打印每帧总耗时（据此可估算 FPS）；把主循环中被注释的 `print(res)` 取消注释，可看到每个框的坐标和置信度数据。
- **成功现象**：程序启动几秒内出画面，正对摄像头的人脸被稳定框住，多人同框时每张脸都有框。
- **失败现象排查**：
  - 报 `OSError` 或模型加载失败 → 检查 `/sdcard/examples/kmodel/face_detection_320.kmodel` 是否存在（资料里的模型文件要拷到 TF 卡）；
  - 有画面但框不出人脸 → 光线太暗、人脸太小太远，或把 `confidence_threshold` 调低到 0.3 试试；
  - 同一张脸多个框 → 调低 `nms_threshold`；
  - 黑屏无显示 → 检查 LCD 排线与 `display_mode` 设置。

## 6. 代码使用说明

1. **硬件准备**：摄像头排线、LCD 排线在**断电状态**下插好，TF 卡插入卡槽，Type-C 供电。
2. **固件要求**：需烧录正点原子提供的 **CanMV K230D 固件**（固件内置 `libs`、`aidemo`、`nncase_runtime`、`aicube` 等模块），固件版本需与你使用的 CanMV IDE 版本匹配（建议使用正点原子资料中配套的 IDE 与固件）。
3. **打开运行**：启动 CanMV IDE → 左下角点击连接图标连接开发板 → `文件-打开` 选择本目录 `main.py` → 点击绿色运行按钮。停止用 IDE 的红色停止按钮。
4. **模型文件需求**（必须存在于 TF 卡对应路径，可从正点原子例程资料中拷贝）：
   - `/sdcard/examples/kmodel/face_detection_320.kmodel` —— 人脸检测模型；
   - `/sdcard/examples/utils/prior_data_320.bin` —— 4200 个锚框数据。
5. **依赖库**：`libs.PipeLine`、`libs.AIBase`、`libs.AI2D` 位于固件的 `/sdcard/app/libs/` 目录（正点原子出厂 TF 卡已带），`aidemo`、`nncase_runtime`、`ulab`、`media` 均为固件内置模块，无需额外安装，也不依赖本例程文件夹内的其他文件。

## 7. 拓展思考

1. **入门级**：把 `confidence_threshold` 依次改为 0.3、0.7，观察检出率和误检的变化；再取消 `print(res)` 注释，读出框的置信度数值。
2. **进阶级**：在检测到人脸时点亮板载蓝色 LED（GPIO59）或让蜂鸣器响一声，做一个"有人靠近报警器"（可参考基础实验例程中 GPIO/蜂鸣器的写法）。
3. **挑战级**：结合 UART 串口实验，把人脸框坐标通过串口发给 PC 或单片机，实现"人脸追踪云台"或简易门禁的雏形；也可以把本例程的人脸检测作为第一步，串联实验4（人脸识别）做完整的刷脸系统。

## 8. 常见问题

**Q1：运行时报错找不到 kmodel 文件怎么办？**
A：说明 TF 卡里没有模型。确认 TF 卡已被识别为 `/sdcard`，并把正点原子资料中的 `face_detection_320.kmodel` 拷贝到 `/sdcard/examples/kmodel/`、`prior_data_320.bin` 拷贝到 `/sdcard/examples/utils/`，路径必须与代码中完全一致（区分大小写）。

**Q2：帧率不高（每秒十几帧甚至更低），正常吗？**
A：正常。KPU 推理 + 后处理 + 显示本身需要时间，320x320 的人脸检测模型在该分辨率配置下十几到二十几 FPS 属合理范围，人眼看上去基本流畅。想提速可以减小 `rgb888p_size`；打开 `debug_mode=1` 还能看到各阶段耗时，定位瓶颈。

**Q3：检测不到人脸，该怎么调？**
A：按顺序排查：① 光线是否充足，脸是否正对摄像头；② 人脸在画面中是否太小（模型输入只有 320x320，脸太小像素不够）；③ 把 `confidence_threshold` 从 0.5 降到 0.3；④ 侧脸、戴口罩、逆光都会显著降低检出率，属于模型能力边界，换个环境再试。

**Q4：为什么框的坐标要乘除换算（`x * display_size[0] // rgb888p_size[0]`）？**
A：因为 AI 是在 640x360 的小图上跑的，而屏幕显示是 640x480，两个坐标系尺寸不同，必须按比例换算，框才会正好套在屏幕上对应的人脸上。改 `rgb888p_size` 或 `display_size` 时不需要动这段换算代码，它会自动适配。

**Q5：运行一段时间后内存不足报错？**
A：主循环里的 `gc.collect()` 就是用来回收内存的，请勿删除。若仍偶发内存问题，可降低 `rgb888p_size` 分辨率或重启程序；AI 例程常驻内存较大，属于 MicroPython 环境的正常约束。
