# 实验17 车牌检测实验

> 本例程在正点原子 DNK230D（K230D BOX）开发板上运行车牌检测模型（LPD_640.kmodel），实时找出摄像头画面中的车牌位置，并用绿色四边形框出车牌的四个角点。难度：中级。

## 1. 例程简介

车牌检测是智能交通、停车场管理的第一步。与普通目标检测输出"正矩形框"不同，车牌在画面中常常是**倾斜的**，因此本例程的模型输出的是车牌的**四个角点**（四边形），代码用四条绿色线段把它们连起来，紧贴车牌边缘。

- **知识点**：四边形角点检测（4-point detection）、坐标从 AI 分辨率到显示分辨率的换算、`aidemo.licence_det_postprocess` C 库后处理、置信度/NMS 阈值。
- **难度**：中级（模型单一，但输出格式与矩形框检测不同，需要理解 8 个坐标值的组织方式）。
- **用途**：作为实验18"车牌号识别"的前级任务；也可单独用于车辆计数、道闸触发等场景。

> [!NOTE]
> 本例程使用的模型为 `/sdcard/examples/kmodel/LPD_640.kmodel`，输入分辨率 640×640。LPD 即 License Plate Detection（车牌检测）。

## 2. 硬件连接与原理

### 2.1 硬件清单

| 硬件 | 说明 |
| --- | --- |
| CNK230DF 核心板 | 嘉楠 K230D 芯片（RISC-V 双核 + 6TOPS 算力 KPU） |
| DNK230D 底板 | 提供 LCD、摄像头、TF 卡槽等接口 |
| 2.4 寸 MIPI LCD | 带电容触摸，640×480，显示检测结果 |
| MIPI CSI 摄像头 | 代码中 `Sensor(width=1280, height=960)` 采集图像 |
| TF 卡 | 存放固件、模型 `LPD_640.kmodel`、`libs` 库 |

本实验只使用摄像头和 LCD，无需其他外设。

> [!WARNING]
> - 摄像头/LCD 排线**插反或斜插会烧坏接口**，务必断电后按正确方向插接。
> - 不要带电插拔任何排线。
> - KPU 持续推理时芯片发热属于正常现象；请使用供电充足的电源，供电不足会引起重启、花屏。

### 2.2 工作原理（通俗版）

1. 摄像头采集 1280×960 图像，PipeLine 硬件将其缩为 640×360 的 RGB888P 图交给 AI。
2. Ai2d 硬件把这张图直接 resize（**没有 pad 补边**）成模型输入 640×640——画面会被纵向拉伸，模型训练时已适应这种输入，因此不影响检测。
3. KPU 运行车牌检测模型，输出每个车牌的四个角点坐标（x1,y1,x2,y2,x3,y3,x4,y4）。
4. `aidemo.licence_det_postprocess` 按置信度阈值（0.2）筛选、按 NMS 阈值（0.2）去重，并把坐标换算回 640×360 的 AI 图坐标系。
5. `draw_result` 再把坐标按 `AI 分辨率 → 显示分辨率` 的比例换算到 640×480 屏幕上，用 `draw_line` 依次连接角点 1→2→3→4→1，画出绿色四边形。

## 3. 相关前置知识

- **四角点检测**：矩形框只需 (x,y,w,h) 4 个数；倾斜车牌需要 4 个角点共 8 个数。代码中一个 `det` 就是 8 个值，`det[i*2+0]` 是第 i 个角点的 x，`det[i*2+1]` 是 y。
- **置信度阈值 confidence_threshold**：模型对"这块区域是车牌"的把握程度，低于 0.2 的候选被丢弃。
- **IoU 与 NMS**：同一车牌可能被多个候选四边形重复覆盖，NMS（非极大值抑制）保留得分最高的一个，删除与其重叠度（IoU）超过 nms_threshold=0.2 的其余候选。本例 NMS 阈值较小，去重更严格。
- **坐标分辨率换算**：模型工作在 640×360 的 AI 图上，屏幕是 640×480，所以绘制前要按比例 `x/rgb888p_w*display_w`、`y/rgb888p_h*display_h` 换算，否则框会错位。
- **kmodel**：嘉楠 nncase 工具链编译出的、可在 K230D KPU 上运行的神经网络模型文件。
- **推理**：把预处理后的图像张量送入 KPU 计算出结果的过程，由基类 `AIBase.run()` 封装完成。
- **无 pad 的预处理**：与实验15/16 不同，本例程 `config_preprocess` 只调用 `resize`，不做灰边填充，图像比例会被改变，属于"训练时怎么喂、推理时就怎么喂"的对应关系。

## 4. 软件代码逻辑说明

### 4.1 程序整体流程

```
系统初始化 → 创建 PipeLine（摄像头 + LCD）→ 创建 LicenceDetectionApp 并加载 kmodel
→ 配置 Ai2d 预处理（仅 resize）→ 主循环：取帧 → KPU 推理 → aidemo 后处理
→ 坐标换算 → 绘制绿色四边形 → 显示 → gc.collect() → 下一帧
→ 退出时 deinit() + pl.destroy()
```

### 4.2 模型与参数配置（`__main__` 部分）

```python
display_mode="lcd"          # 显示模式，默认"lcd"
display_size=[640,480]
kmodel_path="/sdcard/examples/kmodel/LPD_640.kmodel"
confidence_threshold = 0.2  # 置信度阈值
nms_threshold = 0.2         # NMS 阈值
rgb888p_size=[640,360]      # sensor 给到 AI 的图像分辨率
sensor = Sensor(width=1280, height=960)
```

创建实例时指定 `model_input_size=[640,640]`。

### 4.3 LicenceDetectionApp 类

继承自 `AIBase`，需要自己实现 `config_preprocess`、`postprocess` 两个方法：

**预处理（仅 resize）：**

```python
def config_preprocess(self, input_image_size=None):
    with ScopedTiming("set preprocess config", self.debug_mode > 0):
        ai2d_input_size = input_image_size if input_image_size else self.rgb888p_size
        self.ai2d.resize(nn.interp_method.tf_bilinear, nn.interp_mode.half_pixel)
        self.ai2d.build([1,3,ai2d_input_size[1],ai2d_input_size[0]],
                        [1,3,self.model_input_size[1],self.model_input_size[0]])
```

**后处理（调用 aidemo C 库）：**

```python
def postprocess(self, results):
    with ScopedTiming("postprocess", self.debug_mode > 0):
        det_res = aidemo.licence_det_postprocess(results,
            [self.rgb888p_size[1], self.rgb888p_size[0]],  # AI 图尺寸
            self.model_input_size,                          # 模型输入尺寸
            self.confidence_threshold, self.nms_threshold)
        return det_res
```

返回值 `det_res` 是一个列表，每个元素是一块车牌的 8 个角点坐标（AI 图坐标系下）。

### 4.4 绘制结果（坐标换算 + 连线）

```python
def draw_result(self, pl, dets):
    if dets:
        pl.osd_img.clear()
        point_8 = np.zeros((8), dtype=np.int16)
        for det in dets:
            # 将检测框坐标从sensor图像分辨率转换为显示分辨率
            for i in range(4):
                x = det[i * 2 + 0] / self.rgb888p_size[0] * self.display_size[0]
                y = det[i * 2 + 1] / self.rgb888p_size[1] * self.display_size[1]
                point_8[i * 2 + 0] = int(x)
                point_8[i * 2 + 1] = int(y)
            # 依次连接四个角点
            for i in range(4):
                pl.osd_img.draw_line(point_8[i*2], point_8[i*2+1],
                    point_8[(i+1)%4*2], point_8[(i+1)%4*2+1],
                    color=(255, 0, 255, 0), thickness=4)
    else:
        pl.osd_img.clear()
```

`(255, 0, 255, 0)` 是 ARGB 颜色的绿色，线宽 4 像素；`(i+1)%4` 让第 4 个点连回第 1 个点，闭合四边形。

### 4.5 主循环

```python
while True:
    os.exitpoint()
    with ScopedTiming("total",1):
        img=pl.get_frame()               # 获取当前帧
        res=licence_det.run(img)         # KPU 推理 + 后处理
        licence_det.draw_result(pl,res)  # 绘制
        pl.show_image()                  # 显示
        gc.collect()
```

### 4.6 可调参数一览

| 参数 | 位置 | 作用 |
| --- | --- | --- |
| `confidence_threshold=0.2` | `__main__` | 调大减少误检（如把矩形物体误认成车牌），调小提高检出率 |
| `nms_threshold=0.2` | `__main__` | 调小去重更严格 |
| `rgb888p_size=[640,360]` | `__main__` | 送给 AI 的图像大小；调大可提升远处小车牌的检出率，但推理变慢 |
| `color=(255,0,255,0)` | `draw_result` | 框线颜色（ARGB） |
| `thickness=4` | `draw_result` | 框线粗细 |

## 5. 运行效果

- **屏幕输出**：LCD 实时显示摄像头画面；把摄像头对准含车牌的图像（可打印一张车牌照片或用屏幕显示车牌图片），车牌周围出现**绿色四边形框**，紧贴车牌四角；倾斜车牌的框也随之倾斜。
- **串口输出**：`print(res)` 在代码中被注释，打开后可看到每帧的角点坐标列表；`ScopedTiming("total",1)` 会打印每帧耗时。
- **成功标志**：正对车牌时 1 秒内出现稳定绿框，轻微晃动摄像头框能跟随。
- **失败现象排查**：
  - 检不出车牌 → 车牌在画面中太小/太模糊，让车牌占画面 1/5 以上；或降低 `confidence_threshold`。
  - 框位置偏移 → 确认 `rgb888p_size`、`display_size` 与传入类的参数一致，不要只改一处。
  - 报模型文件错误 → 检查 TF 卡 `/sdcard/examples/kmodel/LPD_640.kmodel` 是否存在。
  - 画面拉伸变形属正常（预处理无 pad），不影响检测效果。

## 6. 代码使用说明

1. **固件准备**：烧录与 CanMV IDE 版本匹配的 **CanMV K230D 固件**（正点原子出厂固件即可），版本不匹配会连接失败或运行报错。
2. **拷贝模型文件**：将正点原子资料中的模型拷贝到 TF 卡，本实验需要：
   - `/sdcard/examples/kmodel/LPD_640.kmodel`（车牌检测模型）
3. **依赖库确认**：`libs.PipeLine`、`libs.AIBase`、`libs.AI2D` 位于固件 TF 卡 `/sdcard/app/libs/` 目录；后处理依赖固件内置的 `aidemo`、`nncase_runtime`、`ulab.numpy`，无需安装第三方库。
4. **运行步骤**：
   - 打开 CanMV IDE → 连接开发板；
   - 打开本目录下的 `main.py`，点击运行；
   - 将摄像头对准车牌（实物或打印图片均可），观察 LCD 上的绿色四边形框；
   - 停止运行后程序自动执行 `deinit()` 与 `pl.destroy()` 释放资源。

> [!NOTE]
> 文件名为 `main.py` 时，上电后无需 IDE 连接即可自动运行。

## 7. 拓展思考

1. **车牌区域放大显示**：利用四角点坐标，结合 Ai2d 的 crop 功能把车牌区域抠出来放大显示在屏幕一角，模拟"车牌抓拍"效果。
2. **结合实验18**：本实验是实验18 的前级。思考：为什么识别字符前必须先做四边形检测而不是矩形检测？（提示：倾斜车牌直接送入字符识别模型会因形变导致识别率下降，实验18 中 `aidemo.ocr_rec_preprocess` 会做矫正。）
3. **多角度鲁棒性测试**：打印不同倾斜角度、不同光照下的车牌图片，统计检出率，尝试通过调整 `confidence_threshold` 找到误检与漏检的平衡点。

## 8. 常见问题

**Q1：为什么画出来的是四条线而不是一个矩形框？**
A：车牌检测模型输出的是四个角点（8 个坐标值），因为真实场景中车牌经常是倾斜的，正矩形框无法贴合。代码用 `draw_line` 依次连接角点 1→2→3→4→1 形成四边形。

**Q2：把画面里的长方形盒子误检成车牌怎么办？**
A：适当调大 `confidence_threshold`（如 0.4~0.5）。该模型主要在中国车牌数据集上训练，对蓝底白字/黄底黑字车牌最敏感，建议使用真实车牌图片测试。

**Q3：同一块车牌出现两个重叠的框？**
A：调小 `nms_threshold`（当前已是 0.2，较严格）。若仍有重复，可能是两个候选四边形 IoU 计算值偏低，属正常边界情况。

**Q4：为什么图像看起来纵向被拉伸了？**
A：预处理只有 resize 没有 pad：640×360 被直接拉成 640×640。这是该模型训练时约定的输入方式，不影响检测精度；屏幕显示的原始画面并未变形，变形的只是"喂给模型"的那份数据。

**Q5：远处的小车牌检测不到？**
A：640×360 的 AI 输入下，远处车牌像素太少。可尝试把 `rgb888p_size` 调大（如 [1280,720]），推理耗时相应增加；或者让车牌离摄像头更近。

**Q6：报错 `no attribute 'licence_det_postprocess'`？**
A：固件中的 `aidemo` 库版本过旧，请刷写与本例程配套的正点原子 CanMV K230D 固件。
