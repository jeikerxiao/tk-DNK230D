# 实验16 物体分割实验

> 本例程在正点原子 DNK230D（K230D BOX）开发板上运行 YOLOv8-seg 实例分割模型，不仅能框出画面中的物体，还能给每个物体"涂上"一层彩色轮廓掩码，直观展示物体的精确外形。难度：中级。

## 1. 例程简介

本实验是"物体检测实验（实验15）"的进阶版。实验15只能画出一个矩形框，而本实验使用 YOLOv8-seg 实例分割模型，可以逐像素地判断"哪些像素属于哪个物体"，并在屏幕上用半透明的彩色掩码把物体的真实轮廓填充出来，同时标注类别名称和置信度分数。

- **知识点**：实例分割（Instance Segmentation）、掩码（Mask）阈值、YOLOv8-seg 网络结构、ARGB8888 图像格式、numpy 数组与 image 对象的零拷贝引用。
- **难度**：中级（后处理由 `aidemo.segment_postprocess` C 库完成，MicroPython 侧代码量不大，但概念比纯检测多一层）。
- **用途**：机器人避障的可通行区域分析、工业零件轮廓测量、抠图/换背景、AR 特效等需要像素级轮廓的场景。

> [!NOTE]
> 本例程使用的模型为 `/sdcard/examples/kmodel/yolov8n_seg_320.kmodel`，输入分辨率 320×320，支持 COCO 数据集的 80 个类别。

## 2. 硬件连接与原理

### 2.1 硬件清单

| 硬件 | 说明 |
| --- | --- |
| CNK230DF 核心板 | 嘉楠 K230D 芯片（RISC-V 双核 + 6TOPS 算力 KPU） |
| DNK230D 底板 | 提供 LCD 接口、摄像头接口、TF 卡槽等 |
| 2.4 寸 MIPI LCD | 带电容触摸，显示分辨率 640×480，本例程结果显示在它上面 |
| MIPI CSI 摄像头 | 本例程通过 `Sensor(width=1280, height=960)` 采集 1280×960 图像 |
| TF 卡 | 存放固件、模型文件 `yolov8n_seg_320.kmodel` 和 `libs` 库 |

本实验只需要摄像头和 LCD，无需连接其他外设。摄像头通过底板的 MIPI CSI 座子连接，LCD 通过 MIPI DSI 排线连接。

> [!WARNING]
> - 摄像头和 LCD 的排线**方向插反或强行插入会烧坏接口**，请在断电状态下按正确方向插接。
> - 严禁带电插拔排线，插拔前务必先断开电源。
> - AI 推理时 KPU 满负荷工作，芯片发热属于正常现象；若使用电脑 USB 供电，请确保供电充足，供电不足会导致重启或图像异常。

### 2.2 工作原理（通俗版）

1. 摄像头采集一帧 1280×960 的图像，PipeLine 硬件把它缩成 320×320 的 RGB888P 小图交给 AI 程序。
2. AI2D 硬件加速器对小图做 pad（补灰边 [114,114,114]）+ resize，凑成模型要求的 320×320 输入。
3. KPU 运行 YOLOv8-seg 模型。它比检测模型多输出一个"原型掩码分支"：网络先预测若干个"基础形状模板"，再对每个检测到的物体预测一组"混合系数"，把模板加权组合起来，就得到该物体的像素级轮廓。
4. `aidemo.segment_postprocess`（C 语言后处理库）完成置信度筛选、NMS、掩码阈值化（大于 `mask_threshold` 的像素算作物体内部），并把结果直接写进一块与屏幕等大的 ARGB 数组 `self.masks` 中，每个物体一种颜色。
5. MicroPython 侧把这块 numpy 数组"包装"成 `image.Image` 对象（不复制数据），整体拷贝到屏幕 OSD 图层，再在掩码上方绘制类别文字。

## 3. 相关前置知识

- **目标检测 vs 实例分割**：检测只回答"物体在哪个矩形框里"；实例分割进一步回答"框内哪些像素属于这个物体"。"实例"指同类的多个个体会被分别分割（两个人会各有自己的掩码）。
- **掩码（Mask）**：一张和图像同样大小的"打分图"，每个像素的值表示它属于该物体的概率。概率高于阈值的像素被标记为物体内部。
- **mask_threshold（掩码阈值）**：本例程为 0.5，即概率 ≥ 0.5 的像素才算物体的一部分。调大 → 掩码更"瘦"更保守；调小 → 掩码更"胖"、更容易把背景也涂进来。
- **置信度阈值 confidence_threshold**：物体类别得分低于 0.2 的候选框直接丢弃。
- **NMS（非极大值抑制）与 IoU**：同一物体常被预测出多个重叠框，NMS 保留得分最高的框，并删掉与它 IoU（交并比）超过 nms_threshold（0.5）的其余框。
- **ARGB8888 格式**：每像素 4 字节（Alpha 透明度 + R + G + B）。Alpha=0 的像素全透明（显示底下摄像头画面），Alpha≠0 的像素显示掩码颜色，这就是"半透明涂色"的实现基础。
- **np.zeros / alloc=image.ALLOC_REF**：`self.masks` 是一块 (1, 480, 640, 4) 的 numpy 数组；`image.Image(..., alloc=image.ALLOC_REF, data=self.masks)` 让 image 对象直接引用这块内存而不复制，效率高。
- **帧率**：320×320 的 YOLOv8n-seg 在 K230D 上可以较流畅运行；场景物体越多，后处理耗时越长，帧率越低。

## 4. 软件代码逻辑说明

### 4.1 程序整体流程

```
系统初始化 → 创建 PipeLine（摄像头 + LCD）→ 创建 SegmentationApp 并加载 kmodel
→ 配置 Ai2d 预处理 → 主循环：取帧 → KPU 推理 → aidemo 分割后处理（填充 masks）
→ 掩码贴屏 + 绘制类别文字 → 显示 → gc.collect() → 下一帧
→ 退出时 deinit() + pl.destroy()
```

### 4.2 模型与参数配置（`__main__` 部分）

```python
display_mode="lcd"                 # 代码注释：默认"hdmi"，可以选择"hdmi"和"lcd"
if display_mode=="hdmi":
    display_size=[1920,1080]
else:
    display_size=[640,480]
kmodel_path="/sdcard/examples/kmodel/yolov8n_seg_320.kmodel"
labels = ["person", "bicycle", "car", ... , "toothbrush"]  # COCO 80 类
confidence_threshold = 0.2   # 置信度阈值
nms_threshold = 0.5          # NMS 阈值
mask_threshold = 0.5         # 掩码阈值
rgb888p_size=[320,320]       # sensor 给到 AI 的图像分辨率
sensor = Sensor(width=1280, height=960)
```

本例程代码中实际设置 `display_mode="lcd"`，因此使用板载 2.4 寸 LCD（640×480）显示；如果接 HDMI 屏，把它改成 `"hdmi"` 即可自动切换 1920×1080 分辨率。

### 4.3 SegmentationApp 类的初始化

```python
class SegmentationApp(AIBase):
    def __init__(self,kmodel_path,labels,model_input_size,confidence_threshold=0.2,
                 nms_threshold=0.5,mask_threshold=0.5,rgb888p_size=[224,224],
                 display_size=[1920,1080],debug_mode=0):
        ...
        # 分割结果的numpy.array，用于给到aidemo后处理接口
        self.masks=np.zeros((1,self.display_size[1],self.display_size[0],4))
```

关键点：`self.masks` 按显示分辨率预分配一块 ARGB 内存，C 库后处理时直接把每个物体的彩色掩码写进这块内存，MicroPython 侧完全不用逐像素处理。

### 4.4 预处理：pad + resize

```python
top,bottom,left,right=self.get_padding_param()
self.ai2d.pad([0,0,0,0,top,bottom,left,right], 0, [114,114,114])
self.ai2d.resize(nn.interp_method.tf_bilinear, nn.interp_mode.half_pixel)
self.ai2d.build([1,3,ai2d_input_size[1],ai2d_input_size[0]],
                [1,3,self.model_input_size[1],self.model_input_size[0]])
```

`get_padding_param()` 按长边比例缩放后计算上下/左右补边，保持图像不变形；补边颜色为 [114,114,114]（YOLO 系列惯用灰色）。

### 4.5 后处理：aidemo.segment_postprocess

```python
seg_res = aidemo.segment_postprocess(results,
    [self.rgb888p_size[1],self.rgb888p_size[0]],  # AI 输入图尺寸
    self.model_input_size,                         # 模型输入尺寸
    [self.display_size[1],self.display_size[0]],   # 显示尺寸
    self.confidence_threshold, self.nms_threshold, self.mask_threshold,
    self.masks)                                    # 输出掩码写到这里
```

返回值 `seg_res` 是三个列表：`seg_res[0]` 为检测框（x1, y1, w, h，显示坐标系）、`seg_res[1]` 为类别 id、`seg_res[2]` 为置信度分数。

### 4.6 绘制结果

```python
if seg_res[0]:
    pl.osd_img.clear()
    mask_img=image.Image(self.display_size[0], self.display_size[1],
                         image.ARGB8888, alloc=image.ALLOC_REF, data=self.masks)
    pl.osd_img.copy_from(mask_img)
    dets,ids,scores = seg_res[0],seg_res[1],seg_res[2]
    for i, det in enumerate(dets):
        x1, y1, w, h = map(lambda x: int(round(x, 0)), det)
        pl.osd_img.draw_string_advanced(x1,y1-50,32,
            " " + self.labels[int(ids[i])] + " " + str(round(scores[i],2)),
            color=self.get_color(int(ids[i])))
else:
    pl.osd_img.clear()
```

先把整张掩码图拷到 OSD 层，再在每个物体框上方 50 像素处写"类别名 + 分数"，文字颜色由 `get_color(x)`（`color_four` 20 色循环，`x % 20`）按类别取色。

### 4.7 主循环

```python
while True:
    os.exitpoint()
    with ScopedTiming("total",1):
        img=pl.get_frame()          # 获取当前帧
        seg_res=seg.run(img)        # KPU 推理 + 后处理
        seg.draw_result(pl,seg_res) # 绘制
        pl.show_image()             # 显示
        gc.collect()                # 垃圾回收
```

### 4.8 可调参数一览

| 参数 | 位置 | 作用 |
| --- | --- | --- |
| `confidence_threshold=0.2` | `__main__` | 调大减少误检、可能漏检小物体 |
| `nms_threshold=0.5` | `__main__` | 调小去重更狠，同物体多框更少 |
| `mask_threshold=0.5` | `__main__` | 调大掩码更贴合核心区域；调小掩码更外扩 |
| `rgb888p_size=[320,320]` | `__main__` | 送给 AI 的图像大小，影响小物体检出率 |
| `display_mode` | `__main__` | `"lcd"` 或 `"hdmi"` |
| `draw_string_advanced(x1,y1-50,32,...)` | `draw_result` | 文字位置与字号 |

## 5. 运行效果

- **屏幕输出**：LCD 上实时显示摄像头画面，画面中每个被识别的物体（人、杯子、手机等）都被一层与其轮廓一致的彩色半透明掩码覆盖，掩码上方显示如 ` person 0.87` 的类别名和分数，不同类别颜色不同。
- **串口输出**：`print(seg_res)` 在代码中被注释掉了；打开后可在 CanMV IDE 串口终端看到每帧的检测框、类别 id、分数列表。`ScopedTiming("total",1)` 会在串口打印每帧总耗时。
- **成功标志**：把摄像头对准人、桌面物品，1 秒内出现轮廓掩码和文字标注。
- **失败现象排查**：
  - 画面全黑/无显示 → 检查 LCD 排线、`display_mode` 是否为 `"lcd"`。
  - 报 `OSError` 找不到模型 → 确认 TF 卡根目录下存在 `examples/kmodel/yolov8n_seg_320.kmodel`。
  - 有框但掩码很碎/很大 → 调整 `mask_threshold`。
  - 什么都检不出 → 降低 `confidence_threshold`，让物体离摄像头更近、光线更充足。

## 6. 代码使用说明

1. **固件准备**：给开发板烧录与 CanMV IDE 版本匹配的 **CanMV K230D 固件**（正点原子出厂固件即可）。固件与 IDE 版本不匹配会导致连接失败或运行报错。
2. **拷贝模型文件**：将正点原子资料中的模型文件拷贝到 TF 卡，本实验需要：
   - `/sdcard/examples/kmodel/yolov8n_seg_320.kmodel`（分割模型）
3. **依赖库确认**：代码依赖 `libs.PipeLine`、`libs.AIBase`、`libs.AI2D`，它们位于固件 TF 卡的 `/sdcard/app/libs/` 目录下；后处理依赖固件内置的 `aidemo` C 库、`nncase_runtime`、`ulab.numpy`，均无需另行安装。不要删除或改动 `/sdcard/app/libs/` 目录。
4. **运行步骤**：
   - 打开 CanMV IDE → 连接开发板（左下角连接按钮）；
   - 打开本目录下的 `main.py`；
   - 点击绿色运行按钮，将摄像头对准待分割物体，观察 LCD 显示效果；
   - 点击停止按钮结束运行（程序会自动执行 `deinit()` 与 `pl.destroy()` 释放资源）。

> [!NOTE]
> 例程默认以 `main.py` 命名时，上电后无需 IDE 也会自动运行（固件开机自启动机制）。

## 7. 拓展思考

1. **掩码透明度与颜色**：`segment_postprocess` 写入的掩码颜色由 C 库决定，尝试在后处理后用 numpy 修改 `self.masks` 中非零像素的 Alpha 通道值，观察掩码透明度变化。
2. **只保留指定类别**：修改 `draw_result`，仅当 `self.labels[int(ids[i])] == "person"` 时绘制掩码和文字，实现"只分割人"的应用（其他区域需要用 numpy 把 `self.masks` 对应颜色清零）。
3. **面积统计**：利用 `seg_res[0]` 的框面积或 numpy 统计 `self.masks` 中每种颜色的像素个数，估算物体在画面中的占比，尝试做一个"画面拥挤度检测"小应用。

## 8. 常见问题

**Q1：为什么掩码颜色是固定的，和 `color_four` 有什么关系？**
A：`color_four` 只用于类别**文字**的颜色（`get_color(x%20)`）；掩码本身的彩色填充由 `aidemo.segment_postprocess` 在 C 库内部生成，MicroPython 侧拿到时已是成品。

**Q2：运行报错 `module 'aidemo' has no attribute 'segment_postprocess'` 怎么办？**
A：说明固件版本过旧，不含分割后处理接口。请刷写正点原子提供的、与本例程配套的 CanMV K230D 固件。

**Q3：掩码和物体轮廓对不齐、有偏移？**
A：检查 `rgb888p_size` 是否保持 `[320,320]` 且与传入 `SegmentationApp` 的参数一致；pad 参数由 `get_padding_param()` 自动计算，若手动改过预处理（如去掉 pad）会导致坐标错位。

**Q4：帧率明显比实验15（物体检测）低？**
A：正常现象。分割模型多了原型掩码分支，且后处理要把掩码放大到显示分辨率并逐像素阈值化，计算量大于纯检测。可以适当调低 `display_size` 或减少画面中的物体数量。

**Q5：把 `display_mode` 改成 `"hdmi"` 后 LCD 不显示了？**
A：`"hdmi"` 模式下输出到 HDMI 接口（1920×1080），LCD 无显示属于预期行为。使用板载 LCD 请保持 `display_mode="lcd"`。

**Q6：为什么画面边缘的物体掩码缺一块？**
A：320×320 的输入分辨率较低，边缘小物体分割精度有限。可尝试增大 `rgb888p_size`（需模型支持对应输入），或让物体尽量位于画面中央。
