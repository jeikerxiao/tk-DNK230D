# 实验13 局部放大器实验

> 张开拇指和中指对准物体，两指之间的画面区域会被实时"隔空放大"并叠加显示在屏幕上——手指张得越大，放大倍率越高，就像拿了一个随心变化的放大镜。难度：中级。

## 1. 例程简介

本例程结合手掌检测（`hand_det.kmodel`）与手掌关键点（`handkp_det.kmodel`）定位**拇指指尖**和**中指指尖**两个关键点，以两指中点为中心裁剪一块正方形区域，按"当前指间距 / 首次入镜指间距"的比例放大后，贴回屏幕对应位置，实现动态局部放大。你将学到：关键点驱动的交互设计、Ai2d 的 crop+resize 图像变换、numpy 数组直接贴图（OSD mask 技巧）、"基准帧"标定思想。难度为**中级**，预期用途：趣味交互演示、辅助观察细小物体、无接触缩放控制的原型。

## 2. 硬件连接与原理

用到的硬件：

- **CNK230DF 核心板（K230D）**：RISC-V 双核 + 6TOPS KPU；
- **MIPI CSI 摄像头**：`Sensor(width=1280, height=960)`；送 AI 分辨率 `rgb888p_size=[1024,768]`；
- **2.4 寸 MIPI LCD**：`display_mode="lcd"`、`display_size=[640,480]`；
- **TF 卡（/sdcard）**：存放两个 kmodel 模型（锚框写在代码中；`anchors_path` 变量虽定义但未实际加载）。

**局部放大原理（通俗版）**：程序记住你**第一次**把手伸进画面时拇指尖与中指尖的距离，作为"基准距离"（放大倍率 1.0，区域边长取距离的 0.8 倍）。之后每一帧：

1. 找到两个指尖点，取中点作为放大区域的中心；
2. 算当前指间距与基准距离的比值 `ori_new_ratio`——手指张开得比基准大，倍率就大于 1；
3. 用 Ai2d 硬件把原图中该区域 crop 出来、resize 成放大后的尺寸；
4. 把放大图贴回屏幕原位置附近，形成"透过放大镜看"的效果。

这就是手机相机"双指缩放"的手势版：距离变化 = 缩放滑块。

> [!WARNING]
> - 摄像头/LCD 的 MIPI 排线必须**断电插拔**，插反通电会烧坏器件；
> - 供电不足会导致推理时重启，请使用足功率 Type-C 电源；
> - AI 推理发热明显，**属正常现象**；
> - 基准距离在**首次手掌入镜**时标定，如果第一帧手势没摆好（比如两指没张开），后续倍率都会以它为参照——手移出画面重新入镜**不会**重新标定（`first_start` 只生效一次），需要重新运行程序才能重新标定。

## 3. 相关前置知识

- **关键点索引**：21 点手部模型中，0 号手腕，拇指为点 1~4（4 号是拇指尖），食指 5~8，中指 9~12（12 号是中指尖）。代码取 `results_show[8], results_show[9]`（点 4，拇指尖坐标 x,y）和 `results_show[24], results_show[25]`（点 12，中指尖坐标）。
- **两点距离**：欧氏距离公式 `sqrt((x1-x2)² + (y1-y2)²)`，代码用 `np.sqrt(pow(...,2)+pow(...,2))` 计算指间距。
- **基准标定（first_start）**：第一次检测到有效的两指坐标（都在画面内）时，把指间距 x0.8 存为基准区域边长 `two_point_mean_w/h`，并置 `first_start = False`。
- **crop + resize**：Ai2d 硬件先裁剪后放大，`imgprocess()` 中输出格式为 RGB_packed（HWC 排布），方便直接赋值给显示数组。
- **OSD 贴图（mask 技巧）**：`self.masks` 是与屏幕同尺寸的 ARGB numpy 数组，包装成 `image.Image`（`alloc=image.ALLOC_REF` 表示引用外部内存不另分配）。贴图 = 把 alpha 通道置 255（不透明）、RGB 通道填入放大图数据：

```python
self.masks[y:y+h, x:x+w, 0] = 255          # alpha
self.masks[y:y+h, x:x+w, 1:4] = space_np_out[0][0:h, 0:w, :]   # RGB
```

- **锚框 / 置信度 / NMS / strides**：手掌检测用，同实验8。
- **kmodel / KPU / 张量 / 推理 / 帧率**：概念同前。

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

### 4.2 两个模型类

- **HandDetApp**：手掌检测，pad（114 灰边）+ resize 到 512x512，`aicube.anchorbasedet_post_process` 后处理——同实验10~12；
- **HandKPClassApp**：手掌关键点，crop（中心外扩 1.26 倍）+ resize 到 256x256；后处理只把归一化坐标换算到**原图坐标**（注意：本实验没有再换算到显示坐标，显示换算在 `run()` 里做）：

```python
def postprocess(self, results):
    results = results[0].reshape(results[0].shape[0]*results[0].shape[1])
    results_show = np.zeros(results.shape, dtype=np.int16)
    results_show[0::2] = (results[0::2] * self.crop_params[3] + self.crop_params[0])
    results_show[1::2] = (results[1::2] * self.crop_params[2] + self.crop_params[1])
    return results_show
```

### 4.3 放大变换：imgprocess()

```python
def imgprocess(self, input_np, x, y, w, h, out_w, out_h):
    self.ai2d.crop(x, y, w, h)
    self.ai2d.resize(nn.interp_method.tf_bilinear, nn.interp_mode.half_pixel)
    self.ai2d.build([1,3,self.rgb888p_size[1],self.rgb888p_size[0]],[1,out_h,out_w,3])
    return self.ai2d.run(input_np).to_numpy()
```

注意这里的 Ai2d 输出类型在构造时设为 `RGB_packed`（`set_ai2d_dtype(..., nn.ai2d_format.NCHW_FMT, nn.ai2d_format.RGB_packed, ...)`），输出形状 `[1, out_h, out_w, 3]`，是可直接贴图的 HWC 格式。

### 4.4 核心逻辑：SpaceResize.run()

```python
det_boxes = self.hand_det.run(input_np)     # 手掌检测
...                                          # 三条"太小/贴边"过滤规则，得到 det_res
if len(det_res) != 0:
    det_box = det_res[0]                     # 只取第一只手
    self.hand_kp.config_preprocess(det_box)
    results_show = self.hand_kp.run(input_np)
    # 拇指尖(点4)与中指尖(点12)坐标
    two_point[0],two_point[1],two_point[2],two_point[3] = results_show[8],results_show[9],results_show[16+8],results_show[16+9]
    if (self.first_start):
        if (两指坐标都在画面内):
            self.two_point_mean_w = 指间距 * 0.8    # 标定基准区域边长
            self.two_point_mean_h = 指间距 * 0.8
            self.first_start = False
    else:
        self.mask_img.clear()
        # 以两指中点为中心、基准边长为大小的裁剪区（max/min 防止越界）
        self.two_point_left_x = int(max(中点x - 基准w/2, 0))
        self.two_point_top_y  = int(max(中点y - 基准h/2, 0))
        self.two_point_crop_w/h = ...         # 越界收缩处理
        # 当前指间距/基准指间距 = 放大倍率
        self.ori_new_ratio = np.sqrt(pow(two_point[0]-two_point[2],2) + pow(two_point[1]-two_point[3],2))*0.8 / self.two_point_mean_w
        # 放大后的像素尺寸（换算到显示坐标系，且上限 600）
        self.new_resize_w = min(int(self.two_point_crop_w * self.ori_new_ratio / self.rgb888p_size[0] * self.display_size[0]), 600)
        self.new_resize_h = min(int(self.two_point_crop_h * self.ori_new_ratio / self.rgb888p_size[1] * self.display_size[1]), 600)
        # 贴图起点（原图坐标 -> 屏幕坐标），并防止超出屏幕右/下边界
        self.rect_frame_x/y = ...
        self.draw_w = min(self.new_resize_w, self.display_size[0]-self.rect_frame_x-1)
        self.draw_h = min(self.new_resize_h, self.display_size[1]-self.rect_frame_y-1)
        # Ai2d 裁剪+放大，然后写入 masks 数组完成贴图
        space_np_out = self.imgprocess(input_np, self.two_point_left_x, self.two_point_top_y,
                                       self.two_point_crop_w, self.two_point_crop_h,
                                       self.new_resize_w, self.new_resize_h)
        self.masks[y:y+h, x:x+w, 0] = 255
        self.masks[y:y+h, x:x+w, 1:4] = space_np_out[0][0:self.draw_h, 0:self.draw_w, :]
return det_res
```

### 4.5 绘制

```python
def draw_result(self, pl, det_res):
    pl.osd_img.clear()
    if len(det_res) == 1:
        pl.osd_img.copy_from(self.mask_img)     # 恰好一只手：显示放大贴图
    else:
        pl.osd_img.draw_string_advanced((self.display_size[0]//2), (self.display_size[1]//2),
                                        32, "请保证一只手入镜!", color=(255,0,0))
```

> [!NOTE]
> 严格要求**画面里只有一只有效的手**：0 只或 2 只以上都会显示红字"请保证一只手入镜!"。

### 4.6 主流程

```python
hand_det_kmodel_path = "/sdcard/examples/kmodel/hand_det.kmodel"
hand_kp_kmodel_path = "/sdcard/examples/kmodel/handkp_det.kmodel"
rgb888p_size = [1024, 768]
hand_det_input_size = [512, 512]
hand_kp_input_size = [256, 256]
confidence_threshold = 0.2
nms_threshold = 0.5
labels = ["hand"]

sensor = Sensor(width=1280, height=960)
pl = PipeLine(rgb888p_size=rgb888p_size, display_size=display_size, display_mode=display_mode)
pl.create(sensor=sensor)
sr = SpaceResize(...)
while True:
    os.exitpoint()
    with ScopedTiming("total",1):
        img = pl.get_frame()
        det_res = sr.run(img)
        sr.draw_result(pl, det_res)
        pl.show_image()
        gc.collect()
```

### 4.7 可调参数一览

| 参数 | 位置 | 影响 |
| --- | --- | --- |
| `*0.8` 系数 | `run()` 标定与倍率计算处 | 放大区域边长与指间距的比例；调大则放大区更大、倍率变化更平缓 |
| `min(..., 600)` | `new_resize_w/h` 计算处 | 放大图的最大边长（像素），调大可放得更大但更耗资源 |
| `confidence_threshold=0.2` / `nms_threshold=0.5` | `__main__` | 手掌检测灵敏度 |
| `rgb888p_size=[1024,768]` | `__main__` | 送 AI 分辨率，决定放大画质上限 |
| `if len(det_res)==1` | `draw_result()` | 可放宽为 `>=1`（取第一只手），多人场景也能用 |

## 5. 运行效果

- **屏幕画面**：一只手入镜后，屏幕上手掌两指之间的区域出现一块**明显放大的画面贴片**，位置跟着手移动；拇指与中指逐渐张开，贴片变大（放大倍率升高，上限 600 像素边长）；两指捏近，贴片缩小。画面中超过一只手或没有手时，屏幕中央显示红字"请保证一只手入镜!"。
- **串口打印**：每帧耗时（ScopedTiming）。
- **成功现象**：把两指对准桌面小字，张开两指即可在屏幕上看到放大的文字，效果类似放大镜。
- **失败现象排查**：
  - 一直提示"请保证一只手入镜!" → 检测不到手（光线、手的完整性）或同时入了两只手；
  - 有贴片但不放大/倍率怪异 → 首次入镜时手势没摆好，基准距离标定偏了；**停止程序重新运行**，第一次入镜就把拇指中指自然张开；
  - 贴片位置与手不重合 → 正常现象，贴片以两指中点为原点向右下展开，放大后会偏移；
  - 画面卡顿 → 放大区域过大时 resize 数据量大，捏小一点指间距。

## 6. 代码使用说明

1. **硬件准备**：断电插好摄像头与 LCD 排线，插入 TF 卡，Type-C 上电。
2. **固件要求**：正点原子 **CanMV K230D 固件**（内置 libs、aicube、nncase_runtime 等），固件与 CanMV IDE 版本匹配。
3. **运行方式**：CanMV IDE 连接开发板 → 打开本目录 `main.py` → 运行 → **先摆好手势（拇指与中指张开）再把手送入画面**完成基准标定 → 移动手掌对准要观察的物体 → 张合两指控制放大倍率。
4. **模型文件需求**（从正点原子资料拷贝到 TF 卡）：
   - `/sdcard/examples/kmodel/hand_det.kmodel` —— 手掌检测模型；
   - `/sdcard/examples/kmodel/handkp_det.kmodel` —— 手掌关键点模型；
   - `anchors_path="/sdcard/examples/utils/prior_data_320.bin"` 在代码中定义但**未加载使用**，非必需。
5. **依赖库**：`libs.PipeLine / libs.AIBase / libs.AI2D`（板载 `/sdcard/app/libs/`）、固件内置 `aicube`、`ulab.numpy`、`image`、`media.sensor`、`nncase_runtime`；不依赖本例程文件夹内其他文件。

## 7. 拓展思考

1. **入门级**：在屏幕角落实时显示当前放大倍率 `ori_new_ratio`（用 `draw_string_advanced` 写文本），直观理解"指间距比例 = 倍率"的关系。
2. **进阶级**：把触发手指换成食指+拇指（点 8 与点 4），并加一个"捏合到位（倍率 < 0.5）时冻结画面"的功能——把当前 `masks` 停止更新即可实现"拍照定格"。
3. **挑战级**：结合 UART 把放大倍率作为控制量输出（比如控制外部电机的转速），或用 KEY0 按键重新标定基准距离（把 `first_start` 置回 True），做成更完善的交互放大镜。

## 8. 常见问题

**Q1：为什么手移出去再伸进来，放大倍率还是怪怪的？**
A：基准距离只在 `first_start=True` 的第一次有效入镜时标定，之后永不更新（这是刻意设计，保证倍率稳定）。手势变化后想重新标定，请停止并重新运行程序。想改成"手离开画面自动重标定"，可在 `len(det_res)==0` 时把 `self.first_start` 置回 `True`。

**Q2：屏幕上始终显示"请保证一只手入镜!"？**
A：两种可能：① 一只有效的手都没检到——检查光线、手是否完整入镜、是否被"太小/贴边"过滤规则拦截；② 检到了两只以上的手（包括背景中误检的"手"）——`draw_result` 要求 `len(det_res)==1`。移开另一只手或调高 `confidence_threshold`。

**Q3：kmodel 加载失败怎么办？**
A：确认 `/sdcard/examples/kmodel/hand_det.kmodel` 与 `handkp_det.kmodel` 已拷贝、文件名大小写一致、来自正点原子配套资料。

**Q4：放大后的图像很模糊？**
A：放大的本质是把小区域像素插值放大，源区域只有 `two_point_crop_w` 见方（约为指间距的 0.8 倍，在原图 1024x768 中），放大倍率越高越糊。改善：① 手离被观察物体近一些，让物体占满两指区域；② 加大 `rgb888p_size`（如 [1280,960]）提高源画质，帧率会降。

**Q5：帧率是否正常？**
A：本实验每帧跑手掌检测 + 关键点 + 一次 Ai2d 裁剪放大，帧率与实验10 相近（十几 FPS 量级）属正常。放大贴片很大时（接近 600x600）绘制开销增加，帧率略降。
