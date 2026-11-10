# 实验4 线段检测实验

> 用 LSD 算法找出画面中一段一段有起点、有终点的"线段"，比霍夫直线更贴近真实物体边缘。难度：**初级**

## 1. 例程简介

本例程调用 `img.find_line_segments()` 检测画面中的线段（LSD，Line Segment Detector 算法），把检出的线段用红色粗线绘制出来，并支持相近线段自动合并。学习知识点是"线段 vs 直线"的区别、merge_distance 与 max_theta_diff 参数。难度为**初级**，预期用途是检测物体的真实棱边（盒子边、书本边、尺子），比实验3 的无限长直线更符合直觉。

## 2. 硬件连接与原理

用到的外设（以 main.py 实际代码为准）：

| 外设 | 代码依据 | 说明 |
| --- | --- | --- |
| MIPI CSI 摄像头 | `Sensor(width=1280, height=960)`，`set_framesize(Sensor.QVGA)`，`set_pixformat(Sensor.RGB565)` | 传感器按 1280x960 构建，通道0 输出 **QVGA（320x240）** RGB565 图像 |
| 2.4 寸 MIPI LCD | `Display.init(Display.ST7701, width=640, height=480, fps=90, to_ide=True)` | ST7701 驱动，640x480 缓冲，同步输出 IDE |

原理通俗讲解：**LSD（线段检测器）**先计算图像中每个像素的梯度（亮度变化），把梯度方向一致的相邻像素聚成"区域"，再用数学方法判断这个区域是不是一条又细又直的带子——是就输出为线段。与霍夫变换找"无限长直线"不同，LSD 直接给出线段的**起点和终点**，画出来就是物体边缘的真实长度。

画面是 QVGA（320x240），屏幕缓冲是 640x480，所以 `Display.show_image` 传入了居中偏移 x、y，把小画面显示在屏幕正中间。

> [!WARNING]
> 新手风险提醒：摄像头/LCD 排线务必**断电插拔**，插反上电可能烧毁器件；请使用稳定的 5V 电源，供电不足可能导致摄像头初始化失败。

## 3. 相关前置知识

- **梯度**：相邻像素亮度差异的大小与方向，LSD 的第一步就是逐像素求梯度。
- **线段 vs 直线**：直线无限长（实验3 霍夫），线段有明确起点终点（本实验 LSD），线段更适合描述"物体的一条边"。
- **merge_distance（合并距离）**：两条线段距离小于该值（像素）时合并成一条，默认 0 不合并。本例设 2。
- **max_theta_diff（最大角度差）**：两条线段角度差小于该值（度）才允许合并。本例设 15，即方向差不多、位置又近的线段被拼成一条长线段。
- **QVGA 分辨率**：320x240，比 VGA（640x480）小四倍，计算量小、帧率高，代价是细节少。
- **居中显示偏移**：`x=round((640 - sensor.width()) / 2)`，用屏幕宽减图像宽除以 2 得到水平偏移，垂直同理。
- **帧率 FPS**：`clock.tick()/fps()` 统计。

## 4. 软件代码逻辑说明

### 第一步：初始化摄像头（QVGA 输出）

```python
sensor = Sensor(width=1280, height=960) # 构建摄像头对象
sensor.reset()  # 复位和初始化摄像头
sensor.set_framesize(Sensor.QVGA)   # 设置帧大小QVGA(320x240)，默认通道0
sensor.set_pixformat(Sensor.RGB565) # 设置输出图像格式，默认通道0
```

注意 `Sensor(...)` 构建参数与 `set_framesize` 是两回事：前者是传感器工作模式，后者决定通道0 实际输出分辨率，本例输出 320x240。

### 第二步：初始化显示与计时

```python
Display.init(Display.ST7701, width=640, height=480, fps=90, to_ide=True)
MediaManager.init() # 初始化media资源管理器
sensor.run() # 启动sensor
clock = time.clock() # 构造clock对象
```

### 第三步：主循环——检测线段、绘制、居中显示

```python
while True:
    os.exitpoint() # 检测IDE中断
    clock.tick()   # 记录开始时间（ms）
    img = sensor.snapshot() # 从通道0捕获一张图

    for l in img.find_line_segments(merge_distance=2, max_theta_diff=15):
        img.draw_line(l.line(), color=(255, 0, 0), thickness=2)
        print(l)

    Display.show_image(img, x=round((640 - sensor.width()) / 2), y=round((480 - sensor.height()) / 2))
    print(clock.fps()) # 打印FPS
```

关键函数说明：

- `img.find_line_segments(merge_distance=2, max_theta_diff=15)`：LSD 线段检测，返回线段对象列表。
  - `merge_distance=2`：相距 2 像素以内、方向相近的线段合并，减少碎线；
  - `max_theta_diff=15`：合并时允许的最大角度差 15°；
  - 还可以传 `roi=(x, y, w, h)` 只在感兴趣区域内检测，提高速度（本例未使用，可自行添加）。
- `l.line()`：返回线段的 (x1, y1, x2, y2)，直接传给 `draw_line` 画出红色、宽 2 的线段。
- `l.theta() / l.rho() / l.length() / l.magnitude()`：线段的角度、距离、长度、强度信息，`print(l)` 会一并输出。
- `Display.show_image(img, x, y)`：因图像 320x240 小于屏幕 640x480，用偏移公式把画面居中。

### 第四步：异常处理与资源释放

`finally` 中依次 `sensor.stop()`、`Display.deinit()`、`MediaManager.deinit()`，释放资源。

**可调参数**：
- `merge_distance=2`（第 41 行）：碎线太多时加大（5~10）；想保留细节短线时减小到 0。
- `max_theta_diff=15`：加大后弯曲边缘可能被错误"拉直"合并，减小则合并更保守。
- `set_framesize(Sensor.QVGA)`：改 `Sensor.VGA` 可提升检测精度，但帧率下降。

## 5. 运行效果

- **屏幕画面**：屏幕中央一块 320x240 的摄像头画面，物体边缘（书本边、盒子棱、手指轮廓）被红色粗线段描出，线段长度与真实边缘一致，不会像霍夫直线那样延伸出画面。
- **串口打印**：每检出一条线段打印一次对象详情，每帧打印一次帧率。QVGA 下帧率明显高于 VGA 的霍夫找线。
- **成功现象**：拿一个方形纸盒对着摄像头，能看到四条红边大致勾出盒子轮廓。
- **失败现象排查**：
  - 检不到线段 → 场景边缘对比度太低（白墙+白纸），换有明显棱边的物体、增强侧光；
  - 线段碎成一截截 → 加大 `merge_distance`；
  - 画面只显示在屏幕左上角 → 检查 `show_image` 的 x、y 偏移是否被改动。

## 6. 代码使用说明

1. 用 **CanMV IDE** 打开本文件夹 `main.py`（IDE 与板载 **CanMV K230D 固件**版本需匹配）。
2. USB 连接开发板，点击 IDE 连接按钮，再点击运行。
3. LCD 与 IDE 帧缓冲区同步显示结果，串口终端打印线段信息与帧率。
4. 本例程**不需要额外文件**、无 SD 卡依赖；脱机自启动将 main.py 放入 TF 卡 `/sdcard/`。

> [!NOTE]
> 由于输出是 QVGA 小图，在 IDE 中观察线段细节时可点击帧缓冲区的放大按钮查看。

## 7. 拓展思考

1. **测量线段长度**：用 `l.length()` 过滤掉短于 50 像素的线段，只绘制"长边"，实现简单的大小筛选。
2. **判断矩形**：统计每帧检出的线段，若同时存在近似水平（theta≈0 或 180）和近似垂直（theta≈90）的线段各两条以上，串口输出"检测到矩形物体"。
3. **加 ROI 提速**：给 `find_line_segments` 传入 `roi=(80, 60, 160, 120)`（QVGA 画面中央区域），观察帧率提升，并用 `img.draw_rectangle` 把 ROI 框画出来。

## 8. 常见问题

**Q1：find_line_segments 和 find_lines 有什么区别，该用哪个？**
A：find_lines（霍夫）输出无限长直线，适合找"方向"（如巡线）；find_line_segments（LSD）输出真实起点终点的线段，适合找"物体的边"。要测量边缘长度、判断形状用后者。

**Q2：为什么 Sensor 构建时写 1280x960，图像却只有 320x240？**
A：`Sensor(width=1280, height=960)` 设定的是传感器的工作模式（最大采集规格），`set_framesize(Sensor.QVGA)` 才决定通道0 实际输出给算法的分辨率，硬件会自动缩放裁剪。

**Q3：线段检测结果抖动很大，同一条边一会儿有一会儿没有？**
A：这是逐帧检测的正常现象。可以：增强光照提高边缘对比度；适当加大 `merge_distance`；或在应用层做"连续 N 帧都检出才算数"的滤波。

**Q4：画面四周有黑边，正常吗？**
A：正常。图像 320x240 居中显示在 640x480 缓冲里，四周是未填充区域，不是屏幕故障。

**Q5：想只在画面某个区域找线段怎么写？**
A：给函数加 roi 参数：`img.find_line_segments(merge_distance=2, max_theta_diff=15, roi=(x, y, w, h))`，坐标基于 320x240 图像。
