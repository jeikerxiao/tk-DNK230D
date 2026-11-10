# 实验5 圆形检测实验

> 用霍夫圆变换找出画面中的圆形物体（硬币、瓶盖、球），画出圆心和半径。难度：**初级**

## 1. 例程简介

本例程调用 `img.find_circles()` 在摄像头画面中检测圆形，把检出的圆用红色圆圈标出，串口打印每个圆的位置、半径信息和帧率。学习知识点是霍夫圆变换的直觉理解、threshold 与 x/y/r_margin、r_min/r_max/r_step 等参数。难度为**初级**，预期用途是检测硬币、瓶盖、乒乓球、圆孔等圆形目标，是形状识别的入门实验。

## 2. 硬件连接与原理

用到的外设（以 main.py 实际代码为准）：

| 外设 | 代码依据 | 说明 |
| --- | --- | --- |
| MIPI CSI 摄像头 | `Sensor(width=1280, height=960)`，`set_framesize(Sensor.QVGA)`，`set_pixformat(Sensor.RGB565)` | 传感器按 1280x960 构建，通道0 输出 **QVGA（320x240）** RGB565 图像 |
| 2.4 寸 MIPI LCD | `Display.init(Display.ST7701, width=640, height=480, fps=60, to_ide=True)` | ST7701 驱动，640x480 缓冲，小图居中显示，同步输出 IDE |

原理通俗讲解：**霍夫圆变换**也是"投票"。图像先做边缘提取得到轮廓点，每个边缘点宣称"我可能在一个以某处为圆心、某半径的圆上"，给所有可能的 (圆心x, 圆心y, 半径r) 组合投票；得票超过 threshold 的组合就是一个圆。所以圆越"标准"、边缘越清晰，票越集中，检出越准。参数 r_min/r_max 限定半径搜索范围，能大幅减少计算量并排除误检。

> [!WARNING]
> 新手风险提醒：摄像头/LCD 排线必须**断电插拔**，插反上电可能烧毁器件；请使用 5V/2A 以上稳定电源，供电不足会导致初始化失败或画面异常。

## 3. 相关前置知识

- **边缘与梯度**：圆检测的输入是边缘点（亮度突变处），梯度描述亮度变化快慢。find_circles 内部自动完成边缘提取。
- **霍夫投票**：把"哪些点构成一个圆"转化为"哪个 (x, y, r) 得到的支持票最多"。
- **threshold（累加器阈值）**：得票门槛，本例 3500。越大越严格（误检少、漏检多），越小越宽松。与分辨率相关，QVGA 下 3500 左右是常用起点。
- **x_margin / y_margin / r_margin**：两个圆的圆心 x、y 距离和半径差分别小于这些值时视为同一个圆，只保留最强的，防止一个圆被重复检出多次。本例都是 10。
- **r_min / r_max / r_step**：搜索半径的下限（6 像素）、上限（120 像素）和步长（2 像素）。步长越大搜索越快，但半径定位越粗。
- **QVGA 分辨率**：320x240，计算量小，圆检测对分辨率不敏感，小图足够且快。
- **帧率 FPS**：`clock.tick()/fps()` 统计。

## 4. 软件代码逻辑说明

### 第一步：初始化摄像头与显示

```python
sensor = Sensor(width=1280, height=960) # 构建摄像头对象
sensor.reset() # 复位和初始化摄像头
sensor.set_framesize(Sensor.QVGA)    # 设置帧大小QVGA(320x240)，默认通道0
sensor.set_pixformat(Sensor.RGB565)  # 设置输出图像格式，默认通道0

Display.init(Display.ST7701, width=640, height=480, fps=60, to_ide=True)
MediaManager.init() # 初始化media资源管理器
sensor.run() # 启动sensor
clock = time.clock() # 构造clock对象
```

### 第二步：主循环——找圆、画圆

```python
while True:
    os.exitpoint() # 检测IDE中断
    clock.tick()   # 记录开始时间（ms）
    img = sensor.snapshot() # 从通道0捕获一张图
    for c in img.find_circles(threshold=3500, x_margin=10, y_margin=10,
                              r_margin=10, r_min=6, r_max=120, r_step=2):
        # 画红色圆做指示
        img.draw_circle(c.x(), c.y(), c.r(), color=(255, 0, 0), thickness=2)
        print(c) # 打印圆形的信息
    # 显示图片
    Display.show_image(img, x=round((640 - sensor.width()) / 2), y=round((480 - sensor.height()) / 2))
    print(clock.fps()) # 打印FPS
```

关键函数说明：

- `img.find_circles(...)`：返回 circle 对象列表，参数含义见上文前置知识；还可传 `roi=(x,y,w,h)` 限定检测区域提速（本例未用）。
- `c.x() / c.y() / c.r()`：圆心坐标和半径（像素）。
- `c.magnitude()`：该圆的"得票强度"，越大越可信（print(c) 会输出）。
- `img.draw_circle(c.x(), c.y(), c.r(), ...)`：按检测结果原位画红色圆，thickness=2。
- `Display.show_image(img, x, y)`：320x240 图像在 640x480 屏幕上居中。

### 第三步：异常处理与资源释放

`finally` 中依次 `sensor.stop()` → `Display.deinit()` → `MediaManager.deinit()`，保证 IDE 停止运行后摄像头与显示资源被正确释放，避免下次运行报资源占用错误。

> [!NOTE]
> 调试顺序建议：先在 IDE 帧缓冲区里目测目标的像素半径（鼠标放在圆边缘看坐标差），再据此设定 r_min/r_max，最后微调 threshold。先定半径范围、后调阈值，比盲目调阈值高效得多。

**可调参数**：
- `threshold=3500`（第 40 行）：检不到圆先调小（如 2000）；误检多再调大（如 5000）。
- `r_min=6, r_max=120`：目标硬币在画面中半径约 20~40 像素时，可缩小范围到 `r_min=15, r_max=50`，速度和准确率都会提升。
- `r_step=2`：加大到 4 更快，但半径可能偏差 2~4 像素。
- 三个 margin：同一圆被画出多个圈时加大。

## 5. 运行效果

- **屏幕画面**：屏幕中央 320x240 摄像头画面中，每个检出的圆被红色圆圈套住；拿一枚硬币或瓶盖在镜头前移动，红圈会跟随。
- **串口打印**：每检出一个圆打印一次对象信息（x、y、r、magnitude），每帧打印一次帧率。
- **成功现象**：红圈与圆形物体边缘基本重合，圆心稳定。
- **失败现象排查**：
  - 找不到圆 → 物体不够圆/边缘对比度低/threshold 太高，换标准圆形物（硬币）、加强侧光、调小 threshold；
  - 到处都是假圆 → threshold 太低或背景纹理复杂，调大 threshold、缩小 r_min~r_max 范围、简化背景；
  - 帧率过低 → 缩小 r_max、加大 r_step，或加 roi 只检测画面局部。

## 6. 代码使用说明

1. 用 **CanMV IDE** 打开本文件夹 `main.py`（IDE 与板载 **CanMV K230D 固件**版本需匹配）。
2. USB 连接开发板，点击 IDE 连接按钮，再点击运行。
3. LCD 与 IDE 帧缓冲区同步显示结果，串口终端打印圆信息与帧率。
4. 本例程**不需要额外文件**、无 SD 卡资源依赖；脱机自启动将 main.py 放入 TF 卡 `/sdcard/`。

> [!NOTE]
> 推荐测试道具：一元硬币（浅色桌面上）、黑色瓶盖、乒乓球。摄像头与目标距离 15~40cm 效果最佳。

## 7. 拓展思考

1. **硬币计数器**：统计每帧检出的圆数量（`len(find_circles(...))` 或循环计数），在画面上用 `draw_string_advanced` 显示"数量: N"，摆 3 枚硬币验证。
2. **按大小分拣**：用 `c.r()` 判断圆的大小，半径大于 30 画红圈、小于 30 画绿圈，模拟"大小分类"逻辑。
3. **圆心追踪输出**：把 `c.x(), c.y()` 换算成相对画面中心的偏差，格式化为 `CIRCLE,x,y,r` 通过 UART 发送，为云台/小车对准圆形目标做准备。

## 8. 常见问题

**Q1：为什么我的硬币检测不到，threshold 调多小都没用？**
A：先检查 r_min/r_max 是否覆盖了硬币的实际像素半径——把硬币放在常用工作距离，在 IDE 帧缓冲区用鼠标看坐标估算半径；其次确认边缘对比度，硬币放在同色背景上很难检出。

**Q2：一个瓶盖被套了三四个红圈，抖动不停？**
A：加大 `x_margin/y_margin`（如 10→20）和 `r_margin`，让相近的圆合并；同时适当提高 threshold 只保留最强的圆。

**Q3：threshold 为什么是 3500 这么奇怪的数？和实验3 的 1000 有何区别？**
A：都是霍夫累加器票数门槛，数值没有统一标准，与图像分辨率、边缘强度、目标大小有关。圆形投票空间是三维的 (x,y,r)，通常需要比直线更大的票数，实践中在 2000~5000 之间调。

**Q4：find_circles 帧率比 find_lines 低很多，正常吗？**
A：正常，圆搜索空间比直线大一个维度。缩小 r_min~r_max 范围、加大 r_step、使用 roi 都能明显提速。

**Q5：为什么用 QVGA 而不是 VGA？**
A：圆检测计算量随像素数增长很快，QVGA（320x240）在检出常见圆形目标时精度已足够，帧率却高得多；需要检测很小的圆时才考虑更高分辨率。

**Q6：能把圆心坐标和半径保存到变量里做后续判断吗？**
A：可以。find_circles 返回的是列表，先存下来再处理，例如：`circles = img.find_circles(...)`，然后 `if len(circles) > 0: c = circles[0]; x, y, r = c.x(), c.y(), c.r()`，即可用 x、y、r 参与大小比较、串口输出等逻辑。
