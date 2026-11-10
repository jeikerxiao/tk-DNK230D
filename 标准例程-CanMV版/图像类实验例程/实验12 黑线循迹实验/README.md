# 实验12 黑线循迹实验

> 用三条横带 ROI 加权求黑线质心，再换算成偏转角，输出给小车做循迹转向。难度：**中级**

## 1. 例程简介

本例程在灰度画面中划分上、中、下三条横向 ROI，在每条 ROI 里用 `find_blobs`（灰度阈值）找黑线的最大色块质心，按"越靠近画面底部权重越大"做加权平均得到线中心位置，再用反正切函数换算成 -45°~45° 的**偏转角**显示在屏幕上。学习知识点是 ROI 分区采样、加权平均、质心、非线性角度映射，这是经典巡线小车（灰度摄像头方案）的完整视觉算法。难度为**中级**，预期用途是配合小车底盘实现黑线循迹，输出转向角。

## 2. 硬件连接与原理

用到的外设（以 main.py 实际代码为准）：

| 外设 | 代码依据 | 说明 |
| --- | --- | --- |
| MIPI CSI 摄像头 | `Sensor(width=1280, height=960)`，`set_framesize(Sensor.VGA)`，`set_pixformat(Sensor.GRAYSCALE)` | 输出 VGA（640x480）**灰度**图像 |
| 2.4 寸 MIPI LCD | `Display.init(Display.ST7701, width=640, height=480, fps=90, to_ide=True)` | ST7701 驱动，同步输出 IDE |

原理通俗讲解：摄像头朝下看地面黑线时，画面**底部**的线代表"车眼前方近处"，最可靠；**顶部**的线代表远处，能预告弯道但容易受干扰。所以算法把画面分成三条横带（ROI）：

- 底部横带权重 0.7（说话最算数）；
- 中间横带权重 0.3；
- 顶部横带权重 0.1（只做参考）。

每条横带里找出黑线最大色块的质心 x 坐标，加权平均得到"线的综合中心" center_pos；再用 `-atan((center_pos-320)/240)` 把像素偏差变成角度：线在画面正中 → 0°；线偏左 → 正角度（要左转）；线偏右 → 负角度。用反正切是**非线性映射**——偏得越多，角度响应越强，有利于急弯时快速打方向。

> [!WARNING]
> 新手风险提醒：摄像头/LCD 排线必须**断电插拔**，插反可能烧毁器件；实车测试建议用充电宝给开发板供电，注意 USB 线不要缠绕车轮；保证 5V/2A 供电稳定，电压跌落会导致重启。

## 3. 相关前置知识

- **灰度图与灰度阈值**：`GRAYSCALE_THRESHOLD = [(0, 64)]`——灰度值 0~64 的像素视为"黑线"（注释说明：换成 `[(128,255)]` 就能循白线）。灰度图每像素 0~255 亮度。
- **ROI（感兴趣区域）**：(x, y, w, h) 矩形，find_blobs 只在里面找，既提速又排除干扰。本例三条 ROI 都是通栏宽 640、高 80，分别位于 y=0（顶）、y=200（中）、y=400（底）。
- **权重与加权平均**：每条 ROI 附带一个权重（0.7/0.3/0.1），`centroid_sum += cx * 权重`，最后除以权重总和（`weight_sum`，代码里循环累加 = 1.1）得到加权平均质心。
- **质心 cx**：色块所有像素的"重心"横坐标，比外接框中心更能代表线的位置。
- **最大色块筛选**：`max(blobs, key=lambda b: b.pixels())`——ROI 内可能有多个黑色区域（干扰物），只取像素最多的那个当作线。
- **merge=True**：find_blobs 合并相近碎块，黑线上的虚线段/反光仍能连成整体。
- **反正切 atan 与弧度/角度**：`math.atan` 返回弧度，`math.degrees` 转成度；atan 的非线性让偏差大时角度增长快。
- **帧率 FPS**：clock 对象统计。

## 4. 软件代码逻辑说明

### 第一步：定义灰度阈值、ROI 表和权重和

```python
# 跟踪一条黑线。使用[(128,255)]来跟踪白线。
GRAYSCALE_THRESHOLD = [(0, 64)]

# 采样图像VGA 640*480
ROIS = [ # [ROI, weight]
        (0, 400, 640, 80, 0.7), # 可以根据机器人的实际情况调整权重值。
        (0, 200, 640, 80, 0.3),
        (0,   0, 640, 80, 0.1)
       ]

# 计算权重值（weight）的和 (结果不一定为1).
weight_sum = 0
for r in ROIS: weight_sum += r[4] # r[4] 是矩形权重值.
```

每个 ROI 元组是 (x, y, w, h, weight)：`r[0:4]` 是矩形区域，`r[4]` 是权重。代码还给出了 QQVGA 160x120 和 QVGA 320x240 的注释版 ROI 表，改分辨率时配套切换。

### 第二步：初始化摄像头（灰度）与显示

```python
sensor = Sensor(width=1280, height=960) # 构建摄像头对象
sensor.reset() # 复位和初始化摄像头
sensor.set_framesize(Sensor.VGA)    # 设置帧大小VGA(640x480)，默认通道0
sensor.set_pixformat(Sensor.GRAYSCALE) # 设置输出图像格式，默认通道0

Display.init(Display.ST7701, width=640, height=480, fps=90, to_ide=True)
MediaManager.init() # 初始化media资源管理器
sensor.run() # 启动sensor
clock = time.clock() # 构造clock对象
```

### 第三步：主循环——分带找线、加权求中心、算角度

```python
while True:
    os.exitpoint()
    clock.tick()
    img = sensor.snapshot() # 从通道0捕获一张图

    centroid_sum = 0
    for r in ROIS:
        # 在灰度图中寻找黑线
        blobs = img.find_blobs(GRAYSCALE_THRESHOLD, roi=r[0:4], merge=True)
        if blobs:
            # 寻找矩形中最多像素的区域.
            largest_blob = max(blobs, key=lambda b: b.pixels())
            # 对该区域进行标记.
            img.draw_rectangle([v for v in largest_blob.rect()])
            img.draw_cross(largest_blob.cx(), largest_blob.cy())
            centroid_sum += largest_blob.cx() * r[4] # r[4] 是矩形的权重值.

    center_pos = (centroid_sum / weight_sum) # 确定直线的中心。

    # 使用反正切函数计算直线中心偏离角度.
    deflection_angle = -math.atan((center_pos - 320) / 240) # VGA 640*480
    deflection_angle = math.degrees(deflection_angle) # 弧度转角度

    # LCD显示偏移角度
    img.draw_string_advanced(0, 0, 24, str('%.1f' % deflection_angle), color=(255,255,255), thickness=4)

    Display.show_image(img)
    print(clock.fps()) # 打印FPS
```

关键代码说明：

- `img.find_blobs(GRAYSCALE_THRESHOLD, roi=r[0:4], merge=True)`：注意这里第一参数直接传**灰度阈值列表**——find_blobs 对灰度图就是按亮度区间找块；`roi=r[0:4]` 取元组前 4 项作区域；`merge=True` 合并碎块。
- `max(blobs, key=lambda b: b.pixels())`：Python 惯用法，按像素数取最大 blob。
- `img.draw_rectangle / draw_cross`：画出每条横带找到的线段框与质心十字（灰度图上默认白色绘制）。
- `centroid_sum += largest_blob.cx() * r[4]`：质心 x 乘权重累加。
- `center_pos = centroid_sum / weight_sum`：加权平均得线中心（像素坐标，理想居中为 320）。
- `deflection_angle = -math.atan((center_pos - 320) / 240)`：偏差像素除以 240 归一化后取反正切，负号使"线偏左输出正角"；结果约在 -45°~45°。
- 第 102 行注释掉的 `print("Turn Angle: %f" % deflection_angle)` 是**留给小车的接口**：把角度发给底盘控制器即可完成循迹。

### 第四步：异常处理与资源释放

`finally` 中依次 `sensor.stop()` → `Display.deinit()` → `MediaManager.deinit()`。

**可调参数**：
- `GRAYSCALE_THRESHOLD = [(0, 64)]`（第 25 行）：地面偏暗或线不够黑时调整；循白线改 `[(128, 255)]`。
- ROIS 的 y 位置、高度和权重（第 44~48 行）：摄像头安装角度不同，三条横带位置要相应调整；权重决定"信近处还是信远处"。
- `/240`、`-320`（第 96 行）：与分辨率绑定，换 QVGA/QQVGA 时按注释切换对应公式。

## 5. 运行效果

- **屏幕画面**：灰度画面中三条横带内各出现一个白色矩形框和十字标记（框住该带内的黑线段），左上角白色数字实时显示偏转角（如 `-12.3`）；把画面里的黑线左右移动，角度值连续变化。
- **串口打印**：每帧打印一次帧率（取消第 102 行注释则打印转向角）。
- **成功现象**：黑线居中时角度≈0，线偏左角度为正、偏右为负，弯折处角度变化明显；三条横带都能框到线。
- **失败现象排查**：
  - 某条横带没有框 → 该带内没有黑线或线太细，属正常（if blobs 已处理），三条全没有则检查阈值和环境亮度；
  - 角度乱跳 → 地面有深色干扰物（影子、地砖缝）被当成线，清理场地或调小阈值上限；
  - 角度恒为 0 附近但车不循线 → 检查小车端是否接收/使用该角度输出。

## 6. 代码使用说明

1. 用 **CanMV IDE** 打开本文件夹 `main.py`（IDE 与板载 **CanMV K230D 固件**版本需匹配）。
2. USB 连接开发板，点击 IDE 连接按钮，再点击运行。
3. 场地准备：白底（白纸/白地垫）上用黑色电工胶带贴一条宽 1.5~2.5cm 的线，可有缓弯；摄像头距地面 20~40cm 俯视。
4. LCD 与 IDE 同步显示，串口打印帧率。
5. 装车使用时：取消第 102 行注释或自行通过 UART 把 `deflection_angle` 发给底盘；代码无额外文件依赖；脱机自启动将 main.py 放入 TF 卡 `/sdcard/`。

> [!NOTE]
> 与实验7 的 get_regression 相比，本方案输出的是"转向角"而非直线方程，且三条横带能更好应对大曲率弯道和局部遮挡，是更接近实际比赛/产品的巡线写法。

## 7. 拓展思考

1. **接上小车底盘**：把 deflection_angle 通过 UART 发给运动控制板（或本机 PWM），按"角度>0 左转、<0 右转、比例控制"实现真正循迹，理解视觉端与控制端的分工。
2. **丢线检测与恢复**：当三条横带都找不到 blob 时（可加标志位判断），输出特定角度值（如 90）让小车原地转圈找线，或用上一次 center_pos 的方向继续走一段。
3. **权重与 ROI 调优实验**：把底部权重从 0.7 改到 0.9 或 0.5，观察直道稳定性和弯道响应的变化；再尝试增加第 4 条横带，体会采样密度与稳定性的权衡。

## 8. 常见问题

**Q1：为什么灰度图也能用 find_blobs？它不是找颜色的吗？**
A：find_blobs 找的是"落在阈值区间内的连通像素"。彩色图区间是 LAB 六元组，灰度图区间就是亮度两元组 (0, 64)，同一函数两种用法。

**Q2：weight_sum 是 1.1 不是 1，角度会算错吗？**
A：不会。加权平均本来就是"加权和除以权重总和"，代码特意用循环累加 weight_sum 而不是写死 1.0，就是为了权重随便改都自动正确（注释也说明"结果不一定为1"）。

**Q3：摄像头装在小车上晃动，角度抖得厉害怎么办？**
A：常见对策：对 deflection_angle 做滑动平均（保留最近 5 帧取平均）；适当调低帧间响应（控制端用 PD 控制）；固定摄像头支架减少机械震动。视觉端也可以加大 ROI 高度让线块更大更稳。

**Q4：场地是深色地板，黑线对比不出来？**
A：本算法假设"浅色地面+深色线"。深色地板可改用白线并把阈值换成 `[(128, 255)]`（注释已给出白线方案），同时注意环境反光。

**Q5：换 QVGA 分辨率要改哪些地方？**
A：三处联动：`set_framesize(Sensor.QVGA)`；启用注释里 QVGA 版 ROIS 表（y=200/100/0，高 40，宽 320）；角度公式换成注释中的 `-math.atan((center_pos - 160) / 120)`。分辨率、ROI、公式三者必须一致。
