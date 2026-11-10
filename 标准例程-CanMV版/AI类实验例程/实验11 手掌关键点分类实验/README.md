# 实验11 手掌关键点分类实验

> 与实验10 相同的两级模型（手掌检测 + 21 关键点），但手势判别不再用分类模型，而是用**几何规则**：计算每根手指的弯曲角度，组合判断出 fist（握拳）、five（五指）、gun（手枪）、love（爱心）、one（数字1）、six（六）、three（三）、thumbUp（点赞）、yeah（剪刀手）共 9 种手势。难度：中级。

## 1. 例程简介

本例程运行 `hand_det.kmodel` 检出手掌、`handkp_det.kmodel` 定位 21 个关键点，然后由**纯代码规则**（`hk_gesture` 函数）根据 5 根手指的弯曲角度判断手势类别，把骨架、检测框和手势名一起画在 LCD 上。与实验9（用分类模型识别 3 种手势）相比，本实验展示了另一条技术路线：**关键点 + 几何逻辑**，可解释性强、不依赖训练数据、规则可自己改。难度为**中级**，预期用途：数字手势交互、手语教学演示、自定义手势规则引擎。

## 2. 硬件连接与原理

用到的硬件：

- **CNK230DF 核心板（K230D）**：RISC-V 双核 + 6TOPS KPU；
- **MIPI CSI 摄像头**：`Sensor(width=1280, height=960)`；送 AI 分辨率 `rgb888p_size=[1280,960]`；
- **2.4 寸 MIPI LCD**：`display_mode="lcd"`、`display_size=[640,480]`；
- **TF 卡（/sdcard）**：存放两个 kmodel 模型（锚框写在代码中）。

**几何判别原理（通俗版）**：21 个关键点里，每根手指有 4 个点（指根、两节指骨间关节、指尖前关节、指尖），加上手腕共 21 点。判断一根手指"伸没伸直"，看两个向量的夹角：一个从"远端关节"指向**手腕**，另一个从**指尖**指向"远端关节"。手指伸直时两段方向一致、夹角小；手指弯起时夹角变大。代码对 5 根手指各算一个角度，然后用一组 if/elif 规则把"哪几根伸、哪几根弯"映射成手势名，例如：

- 5 根全弯（角度都 > 65°）→ `fist`；
- 5 根全伸（角度都 < 49°）→ `five`；
- 拇指+食指伸、其余弯 → `gun`；
- 拇指+小指伸、其余弯 → `six`……

> [!WARNING]
> - 摄像头/LCD 的 MIPI 排线必须**断电插拔**，插反通电会烧坏器件；
> - 供电不足会导致双模型推理时重启，请使用足功率 Type-C 电源；
> - AI 推理发热明显，**属正常现象**。

## 3. 相关前置知识

- **关键点（21 点手部模型）**：0 号手腕；每指 4 点，按拇指→食指→中指→无名指→小指排列。点 k 的坐标存于数组下标 `2k`、`2k+1`。
- **向量与夹角**：向量即"有方向的线段"。两向量夹角用点积公式求：`cosθ = (v1·v2) / (|v1| |v2|)`，代码 `hk_vector_2d_angle()` 就是这个公式，再用 `arccos` 换成角度制。
- **角度阈值**：`thr_angle=65.`（大于它算弯曲）、`thr_angle_s=49.`（小于它算伸直）、`thr_angle_thumb=53.`（拇指专用，拇指活动方式特殊，阈值不同）。49~65 之间是"模糊区"，不满足任何规则时返回 `None`（屏幕显示 None）。
- **65535 哨兵值**：关键点数组是 `int16` 类型，模型输出的无效/溢出坐标可能表现为 65535，代码 `if 65535. not in angle_list` 用它排除异常帧（异常坐标算出的角度不可信）。
- **crop / ROI / 归一化坐标**：第二级裁剪手掌区域（中心外扩 1.26 倍），模型输出 0~1 相对坐标，后处理换算到屏幕坐标——三层换算细节见实验10。
- **锚框 / 置信度 / IoU / NMS / strides**：第一级检测用，同实验8/10。
- **kmodel / KPU / 张量 / 推理 / 帧率**：概念同前几个实验；`ScopedTiming("total",1)` 打印每帧耗时。

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

### 4.2 第一级 HandDetApp 与第二级 HandKPClassApp 的推理部分

与实验10 完全同构（pad 114 灰边 + resize 512x512 检测；crop 1.26 倍 + resize 256x256 关键点），不再重复。**核心区别在第二级的后处理**：

```python
def postprocess(self, results):
    results = results[0].reshape(results[0].shape[0]*results[0].shape[1])
    results_show = np.zeros(results.shape, dtype=np.int16)
    results_show[0::2] = results[0::2] * self.crop_params[3] + self.crop_params[0]
    results_show[1::2] = results[1::2] * self.crop_params[2] + self.crop_params[1]
    gesture = self.hk_gesture(results_show)      # 先在原图坐标系下判手势
    results_show[0::2] = results_show[0::2] * (self.display_size[0] / self.rgb888p_size[0])
    results_show[1::2] = results_show[1::2] * (self.display_size[1] / self.rgb888p_size[1])
    return results_show, gesture                 # 同时返回屏幕坐标和手势名
```

注意顺序：**先判手势、再换算到屏幕坐标**（角度与坐标系缩放无关，先算后算都行，但代码在中间插入调用）。

### 4.3 角度计算

```python
def hk_vector_2d_angle(self, v1, v2):
    v1_x, v1_y, v2_x, v2_y = v1[0], v1[1], v2[0], v2[1]
    v1_norm = np.sqrt(v1_x*v1_x + v1_y*v1_y)
    v2_norm = np.sqrt(v2_x*v2_x + v2_y*v2_y)
    dot_product = v1_x*v2_x + v1_y*v2_y
    cos_angle = dot_product/(v1_norm*v2_norm)
    angle = np.acos(cos_angle)*180/np.pi
    return angle
```

标准二维向量夹角公式。

### 4.4 手势规则 hk_gesture

```python
for i in range(5):
    angle = self.hk_vector_2d_angle(
        [(results[0]-results[i*8+4]), (results[1]-results[i*8+5])],      # 远端关节 -> 手腕
        [(results[i*8+6]-results[i*8+8]), (results[i*8+7]-results[i*8+9])])  # 指尖 -> 远端关节
    angle_list.append(angle)
thr_angle, thr_angle_thumb, thr_angle_s, gesture_str = 65., 53., 49., None
```

9 条 if/elif 规则（角度小=伸直，角度大=弯曲）：

| 手势 | 拇指 | 食指 | 中指 | 无名指 | 小指 |
| --- | --- | --- | --- | --- | --- |
| fist | 弯(>53°) | 弯(>65°) | 弯 | 弯 | 弯 |
| five | 伸(<49°) | 伸 | 伸 | 伸 | 伸 |
| gun | 伸 | 伸 | 弯 | 弯 | 弯 |
| love | 伸 | 伸 | 弯 | 弯 | 伸 |
| one | 弯(>5°) | 伸 | 弯 | 弯 | 弯 |
| six | 伸 | 弯 | 弯 | 弯 | 伸 |
| three | 弯(>53°) | 伸 | 伸 | 伸 | 弯 |
| thumbUp | 伸 | 弯 | 弯 | 弯 | 弯 |
| yeah | 弯(>53°) | 伸 | 伸 | 弯 | 弯 |

不匹配任何规则时 `gesture_str` 保持 `None`。

### 4.5 调度与绘制

`HandKeyPointClass.run()` 与实验10 相同：检测 → 边缘框过滤 → 每只手做关键点+手势判断，返回 `boxes` 和 `gesture_res`（每项为 `(坐标数组, 手势名)`）。

`draw_result()` 画三样东西：紫色检测框、21 个关键点圆圈 + 五指分色骨架（红/品红/黄/绿/蓝，画法与实验10 相同），以及手势名文字：

```python
gesture_str = gesture_res[k][1]
pl.osd_img.draw_string_advanced(x_det, y_det-50, 32, " " + str(gesture_str), color=(255,0,255,0))
```

### 4.6 主流程

```python
hand_det_kmodel_path = "/sdcard/examples/kmodel/hand_det.kmodel"
hand_kp_kmodel_path = "/sdcard/examples/kmodel/handkp_det.kmodel"
anchors_path = "/sdcard/examples/utils/prior_data_320.bin"   # 定义了但未实际加载，锚框写在代码里
rgb888p_size = [1280, 960]
hand_det_input_size = [512, 512]
hand_kp_input_size = [256, 256]
confidence_threshold = 0.2
nms_threshold = 0.5
labels = ["hand"]

sensor = Sensor(width=1280, height=960)
pl = PipeLine(rgb888p_size=rgb888p_size, display_size=display_size, display_mode=display_mode)
pl.create(sensor=sensor)
hkc = HandKeyPointClass(...)
while True:
    os.exitpoint()
    with ScopedTiming("total",1):
        img = pl.get_frame()
        det_boxes, gesture_res = hkc.run(img)
        hkc.draw_result(pl, det_boxes, gesture_res)
        pl.show_image()
        gc.collect()
```

### 4.7 可调参数一览

| 参数 | 位置 | 影响 |
| --- | --- | --- |
| `thr_angle=65. / thr_angle_thumb=53. / thr_angle_s=49.` | `hk_gesture()` | 弯曲/伸直判定线；手势总判 None 可缩小两者差距（如 60/52），误判多则拉大 |
| 9 条 if/elif 规则 | `hk_gesture()` | 可自行增删手势组合，比如加"two"（食指+中指伸） |
| `confidence_threshold=0.2` / `nms_threshold=0.5` | `__main__` | 第一级检手灵敏度 |
| `rgb888p_size=[1280,960]` | `__main__` | 送 AI 分辨率，降低可提帧率 |
| `ratio_num=1.26*length` | `get_crop_param()` | 裁剪余量，与关键点模型训练匹配，不建议改 |

## 5. 运行效果

- **屏幕画面**：手上有紫框、21 个关键点和五色手指骨架；框上方显示当前手势名（fist/five/gun/love/one/six/three/thumbUp/yeah），不满足任何规则时显示 `None`；手势切换时文字实时更新。
- **串口打印**：每帧耗时；取消 `print(det_boxes, gesture_res)` 注释可看到框、坐标数组与手势名。
- **成功现象**：手背正对摄像头、动作标准时，9 种手势都能稳定识别。
- **失败现象排查**：
  - 常显示 None → 手势不标准（手指处于 49°~65° 模糊区）、侧对摄像头导致 2D 角度失真；转成手背正对镜头再试；
  - 模型加载失败 → 核对两个 kmodel 路径；
  - 关键点错位 → 参考实验10 的排查方法；
  - 某个手势总被认成别的 → 该手势的角度组合踩在规则边界上，微调阈值或规则。

## 6. 代码使用说明

1. **硬件准备**：断电插好摄像头与 LCD 排线，插入 TF 卡，Type-C 上电。
2. **固件要求**：正点原子 **CanMV K230D 固件**（内置 libs、aicube、nncase_runtime 等），固件与 CanMV IDE 版本匹配。
3. **运行方式**：CanMV IDE 连接开发板 → 打开本目录 `main.py` → 运行；手掌距摄像头 20~50cm、**手背正对镜头**，逐个比出 9 种手势验证。
4. **模型文件需求**（从正点原子资料拷贝到 TF 卡）：
   - `/sdcard/examples/kmodel/hand_det.kmodel` —— 手掌检测模型；
   - `/sdcard/examples/kmodel/handkp_det.kmodel` —— 手掌关键点模型；
   - `anchors_path` 变量虽定义但**未实际使用**，`prior_data_320.bin` 非必需。
5. **依赖库**：`libs.PipeLine / libs.AIBase / libs.AI2D`（板载 `/sdcard/app/libs/`）、固件内置 `aicube`、`ulab.numpy`、`image`、`media.sensor`、`nncase_runtime`；不依赖本例程文件夹内其他文件。

> [!NOTE]
> 本实验的手势判断**完全在 Python 代码里**（`hk_gesture`），不涉及第三个模型——想加新手势不用训练模型，改 if/elif 就行，这是它比实验9 灵活的地方。

## 7. 拓展思考

1. **入门级**：在 `hk_gesture` 里新增一条规则识别"two"（食指+中指伸直、其余弯曲，即中指版 yeah），体会规则引擎的扩展方式；把 None 情况显示为"unknown"更友好。
2. **进阶级**：做**手势数字密码锁**——要求依次比出 one→six→three 才点亮蓝色 LED（GPIO59）并蜂鸣一声，序列错误则红灯（GPIO61）；用状态机记录已匹配的进度。
3. **挑战级**：对比实验9（分类模型）与本实验（几何规则）对同一手势的识别稳定性：光照变化、手旋转 90°（手心对自己）、远距离小目标三种条件下分别测试，总结两种方案的适用场景；再把结果通过 UART 输出，做手势控制的小车/云台指令源。

## 8. 常见问题

**Q1：明明比了手势，屏幕却一直显示 None？**
A：三个原因最常见：① 手掌没有正对摄像头——2D 角度判别对视角敏感，侧着手时投影角度失真；② 手指伸直/弯曲不到位，角度落在 49°~65° 模糊区；③ 关键点本身检测错了（手太小、光线暗）。先保证手背正对镜头、距离 30cm 左右、五指动作夸张一点。

**Q2：kmodel 加载失败怎么办？**
A：确认 `/sdcard/examples/kmodel/hand_det.kmodel` 与 `handkp_det.kmodel` 都已拷贝且大小写一致；模型来自正点原子配套资料。

**Q3：为什么 fist 和 thumbUp 容易混？**
A：两者只差拇指：fist 要求拇指角 > 53°，thumbUp 要求拇指角 < 49°。竖大拇指时如果拇指不够直、或其他四指没握紧，角度就会跨界。可微调 `thr_angle_thumb`（如 56）或要求其余四指角度更大才判 fist。

**Q4：帧率低是否正常？**
A：正常。每帧 1 次检测 + 每只手 1 次关键点推理，`rgb888p_size=[1280,960]` 数据量大。降到 [640,480] 可明显提速，关键点精度略降但手势判别一般不受影响。

**Q5：`65535.` 是什么？为什么要判断它？**
A：关键点结果存放在 `int16` 数组中，当模型输出异常（如无效坐标溢出）时会出现 65535 这类哨兵值，用它算角度毫无意义。`if 65535. not in angle_list` 把这种异常帧直接跳过（返回 None），防止显示错误手势。
