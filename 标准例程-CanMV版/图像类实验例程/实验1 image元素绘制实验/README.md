# 实验1 image元素绘制实验

> 在摄像头画面上用代码画出直线、矩形、圆、文字等图形元素，是机器视觉的"Hello World"。难度：**入门**

## 1. 例程简介

本例程演示如何用 CanMV 的 `image` 对象在摄像头实时画面上绘制各种图形元素：线段、矩形、圆形、椭圆、十字线、箭头和字符串（含中文）。学习知识点是 image 对象的绘图 API 和"采集→绘制→显示"的基本流程。难度为**入门**，是所有图像类实验的第一课，预期用途是让你熟悉开发板的摄像头+屏幕基本工作流程，并掌握后续实验中"把算法结果画出来"的技能。

## 2. 硬件连接与原理

用到的外设（以 main.py 实际代码为准）：

| 外设 | 代码依据 | 说明 |
| --- | --- | --- |
| MIPI CSI 摄像头 | `Sensor(width=1280, height=960)`，随后 `set_framesize(Sensor.VGA)` | 传感器按 1280x960 构建，通道0 输出 VGA（640x480）分辨率、RGB565 彩色格式 |
| 2.4 寸 MIPI LCD | `Display.init(Display.ST7701, width=640, height=480, to_ide=True)` | 驱动芯片为 ST7701，显示缓冲区 640x480，同时把画面推给 CanMV IDE |

原理通俗讲解：摄像头每一帧拍出一张 640x480 的图片（img 对象），我们在这张图片上"涂涂画画"（draw_xxx 系列函数只是修改图片内存中的像素值），再把画好的图片送到 LCD 屏幕和 IDE 显示。因为循环速度很快，看上去就像图形"贴"在实时视频上。

> [!WARNING]
> 新手风险提醒：
> - 摄像头 MIPI 排线插反、插歪后上电可能烧坏摄像头或核心板，务必**断电**插拔，金手指方向按丝印对准。
> - LCD 排线同理，必须断电操作。
> - 请使用配套 5V/2A 以上电源供电，供电不足会导致屏幕闪烁、摄像头掉线甚至无法开机。

## 3. 相关前置知识

- **像素**：图像的最小单位，一张 640x480 的图由 640 列 x 480 行共 307200 个像素点组成，坐标 (x, y) 从左上角 (0,0) 开始，向右 x 增大，向下 y 增大。
- **RGB565 颜色格式**：每个像素用 16 位表示，红 5 位、绿 6 位、蓝 5 位。代码中颜色写成 `(R, G, B)` 三元组，如 `(255, 0, 0)` 是纯红、`(0, 255, 0)` 是纯绿、`(0, 0, 255)` 是纯蓝。
- **帧与循环**：`sensor.snapshot()` 每调用一次抓取一"帧"画面；while True 循环不断抓帧、绘制、显示，形成流畅视频。
- **thickness（线宽）**：绘制线条的粗细，单位像素。
- **fill（填充）**：画矩形/圆形时是否在内部填满颜色。
- **IDE 缓冲区输出（to_ide=True）**：让画面同时显示在电脑上的 CanMV IDE 里，方便不接屏幕调试。

## 4. 软件代码逻辑说明

### 第一步：初始化摄像头

```python
sensor = Sensor(width=1280, height=960) # 构建摄像头对象
sensor.reset()                          # 复位和初始化摄像头
sensor.set_framesize(Sensor.VGA)        # 设置帧大小VGA(640x480)，默认通道0
sensor.set_pixformat(Sensor.RGB565)     # 设置输出图像格式，默认通道0
```

`Sensor(width, height)` 构建传感器对象；`set_framesize(Sensor.VGA)` 把输出通道0 的分辨率设为 640x480；`set_pixformat(Sensor.RGB565)` 设为彩色格式（绘图需要颜色，所以不能用灰度）。

### 第二步：初始化显示与媒体资源

```python
Display.init(Display.ST7701, width=640, height=480, to_ide=True)
MediaManager.init() # 初始化media资源管理器
sensor.run()        # 启动sensor
```

`Display.init` 初始化 ST7701 驱动的 LCD，`MediaManager.init()` 分配底层媒体内存（必须先于 `sensor.run()`），`sensor.run()` 让摄像头开始出流。

### 第三步：主循环——抓帧并绘制元素

```python
while True:
    os.exitpoint()          # 检测IDE中断
    img = sensor.snapshot() # 从通道0捕获一张图

    img.draw_line(10, 30, 70, 30, color=(255, 0, 0), thickness=2)
    img.draw_rectangle(10, 70, 60, 60, color=(0, 255, 0), thickness=2, fill=False)
    img.draw_circle(40, 170, 30, color=(0, 0, 255), thickness=2, fill=True)
    img.draw_ellipse(40, 240, 30, 15, 45, color=(255, 0, 0), thickness=2)
    img.draw_cross(40, 310, color=(0, 255, 0), size=30, thickness=2)
    img.draw_arrow(10, 380, 70, 380, color=(0, 0, 255), thickness=2)
    img.draw_string_advanced(150, 20, 60, "正点原子", color=(255, 0, 0))
    img.draw_string(150, 100, "Hello\r\nK230D BOX", color=(255, 0, 0), scale=6)

    Display.show_image(img, x=round((640 - sensor.width()) / 2), y=round((480 - sensor.height()) / 2))
```

关键函数逐个说明：

- `draw_line(x0, y0, x1, y1, color, thickness)`：从 (10,30) 到 (70,30) 画一条红色横线。
- `draw_rectangle(x, y, w, h, ...)`：左上角 (10,70)、宽高各 60 的绿色空心矩形（`fill=False`）。
- `draw_circle(cx, cy, r, ...)`：圆心 (40,170)、半径 30 的蓝色实心圆（`fill=True`）。
- `draw_ellipse(cx, cy, rx, ry, rotation, ...)`：中心 (40,240)、横半径 30、纵半径 15、旋转 45 度的红色椭圆。
- `draw_cross(cx, cy, size=..., ...)`：在 (40,310) 画绿色十字，臂长由 `size=30` 控制。
- `draw_arrow(x0, y0, x1, y1, ...)`：画一条从 (10,380) 指向 (70,380) 的蓝色箭头。
- `draw_string_advanced(x, y, size, str, color)`：绘制矢量字符，**支持中文**，这里 60 是字号，显示"正点原子"。
- `draw_string(x, y, str, color, scale)`：绘制 ASCII 字符，`scale=6` 是放大倍数，`\r\n` 实现换行。
- `Display.show_image(img, x, y)`：把图片显示到 LCD 和 IDE；x、y 用 `(640-宽)/2`、`(480-高)/2` 计算，让画面**居中**显示。

### 第四步：异常处理与资源释放

```python
finally:
    if isinstance(sensor, Sensor):
        sensor.stop()
    Display.deinit()
    os.exitpoint(os.EXITPOINT_ENABLE_SLEEP)
    time.sleep_ms(100)
    MediaManager.deinit()
```

程序停止时（IDE 点停止或异常）依次停摄像头、关显示、释放媒体内存，避免下次运行报错。

**可调参数**：所有 draw_xxx 的坐标、颜色、thickness、fill、scale 都可以随意改，改完重新运行立刻看到效果；`set_framesize` 可换成 `Sensor.QVGA`（320x240）等更小分辨率（帧率更高，但绘制坐标要相应缩小）。

## 5. 运行效果

- **屏幕画面**：LCD 与 IDE 显示摄像头实时画面，左侧从上到下依次是红色横线、绿色空心方框、蓝色实心圆、红色斜椭圆、绿色十字、蓝色箭头；右侧显示红色中文"正点原子"和两行英文 "Hello / K230D BOX"。
- **串口打印**：本例程正常情况下没有周期性打印，仅在异常时打印 `user stop:` 或 `Exception ...`。
- **成功现象**：图形位置固定、颜色正确，摄像头画面在图形下层实时刷新。
- **失败现象排查**：
  - 屏幕黑屏但 IDE 有图像 → 检查 LCD 排线是否插好；
  - IDE 无图像 → 检查 USB 线、固件是否正常启动；
  - 摄像头画面全绿/全黑 → 排线接触不良或插反，断电重插。

## 6. 代码使用说明

1. 用 **CanMV IDE**（版本需与开发板烧录的 **CanMV K230D 固件**匹配，正点原子出厂固件即可）打开本文件夹的 `main.py`。
2. 点击 IDE 左下角"连接"按钮，通过 USB 线连接开发板（板载 USB 转串口/调试口）。
3. 点击绿色"运行"按钮，代码会下载到板子上运行，IDE 右上角帧缓冲区显示画面，LCD 同步显示。
4. 点击红色"停止"按钮结束运行，`finally` 中的代码会自动释放资源。
5. 本例程**不需要任何额外文件**，无需 SD 卡资源（若要把代码脱机自启动，可将 main.py 放入 TF 卡挂载的 `/sdcard/` 目录下）。

> [!NOTE]
> 想脱机运行（不连电脑）：把 main.py 拷贝到 TF 卡的 `/sdcard/` 下，上电即自动执行。

## 7. 拓展思考

1. **改造成动态图形**：在循环里用 `time.ticks_ms()` 让十字线的坐标随时间移动，做一个"游动"的标记。
2. **画出算法结果的雏形**：把 `draw_rectangle` 的坐标改成变量，尝试用两个 for 循环画出 3x3 的网格线（用 `draw_line`），为后续 ROI 划分打基础。
3. **结合按键交互**：查阅正点原子按键例程，按下 KEY0 时切换绘制颜色（红→绿→蓝），体会状态机写法。

## 8. 常见问题

**Q1：为什么 `draw_string` 写中文变成乱码或方块？**
A：`draw_string` 只支持 ASCII 字符，中文必须用 `draw_string_advanced`，它使用矢量字体渲染，第三个参数是字号。

**Q2：为什么画出来的图形显示位置和我算的坐标不一样？**
A：图形是画在 `sensor.snapshot()` 得到的图像坐标系上（本例为 640x480），如果你改了 `set_framesize`（比如改成 QVGA 320x240），坐标 (40, 310) 就可能超出图像范围而不显示。

**Q3：运行报 `Exception` 且屏幕无显示，重新运行还是失败？**
A：多半是上次异常退出没释放资源，按一下开发板复位键（或断电重启）再运行即可。

**Q4：`Display.show_image` 里的 x、y 参数是干什么的？**
A：图像分辨率与 LCD 显示缓冲分辨率不一致时，x、y 指定图像贴到屏幕的偏移位置，本例用居中公式让画面显示在屏幕正中间。
