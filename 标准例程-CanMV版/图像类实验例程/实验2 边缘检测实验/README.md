# 实验2 边缘检测实验

> 用 Canny 边缘检测算法把画面中物体的"轮廓线"提取出来，是理解后续直线/圆形检测的基础。难度：**初级**

## 1. 例程简介

本例程调用 `img.find_edges()` 对摄像头灰度画面做 Canny 边缘检测，屏幕上只显示黑白轮廓线，串口打印实时帧率。学习知识点是"边缘""梯度""双阈值"等图像处理基本概念和 `find_edges` 函数用法。难度为**初级**，预期用途是理解机器视觉如何"看见"物体轮廓，为直线检测（实验3）、圆形检测（实验5）等打基础。

## 2. 硬件连接与原理

用到的外设（以 main.py 实际代码为准）：

| 外设 | 代码依据 | 说明 |
| --- | --- | --- |
| MIPI CSI 摄像头 | `Sensor(width=1280, height=960)`，`set_framesize(Sensor.VGA)`，`set_pixformat(Sensor.GRAYSCALE)` | 输出 VGA（640x480）**灰度**图像 |
| 2.4 寸 MIPI LCD | `Display.init(Display.ST7701, width=640, height=480, fps=60, to_ide=True)` | ST7701 驱动，640x480 缓冲，同步输出到 IDE |

原理通俗讲解：图像中明暗突变的地方就是"边缘"，比如白色桌面和黑色盒子的交界。算法先计算每个像素的**梯度**（相邻像素亮度差多大、朝哪个方向变化），梯度大的点就是候选边缘；Canny 再用高低两个阈值筛选：梯度高于高阈值的一定是边缘，低于低阈值的一定不是，介于两者之间的只有"连到强边缘"才保留。这样得到的轮廓又细又连续。

> [!WARNING]
> 新手风险提醒：摄像头/LCD 排线必须**断电插拔**，插反上电可能烧毁器件；请保证 5V 供电稳定，供电不足会导致画面异常或系统重启。

## 3. 相关前置知识

- **灰度图（GRAYSCALE）**：每个像素只有亮度值 0（黑）~255（白），没有颜色。边缘检测只关心明暗变化，用灰度图计算量小、效果更好，所以本例 `set_pixformat(Sensor.GRAYSCALE)`。
- **边缘**：图像中亮度发生剧烈变化的位置，通常对应物体的轮廓。
- **梯度**：描述亮度变化的快慢和方向。想象爬坡：坡越陡梯度越大。平坦区域梯度接近 0，轮廓处梯度很大。
- **threshold 双阈值**：`threshold=(50, 80)` 中 50 是低阈值、80 是高阈值。调高阈值 → 只留下更强的边缘（线条更少更干净）；调低阈值 → 更多细节被当成边缘（线条更密但噪点多）。
- **Canny 与 SIMPLE 两种检测器**：`image.EDGE_CANNY` 效果好、计算多；`image.EDGE_SIMPLE` 速度快、效果糙（代码中被注释掉可对比）。
- **帧率 FPS**：每秒处理多少帧画面，由 `time.clock()` 对象的 `tick()/fps()` 测出并打印。

## 4. 软件代码逻辑说明

### 第一步：初始化摄像头（灰度输出）

```python
sensor = Sensor(width=1280, height=960) # 构建摄像头对象
sensor.reset() # 复位和初始化摄像头
sensor.set_framesize(Sensor.VGA)    # 设置帧大小VGA(640x480)，默认通道0
sensor.set_pixformat(Sensor.GRAYSCALE) # 设置输出图像格式，默认通道0
```

注意与实验1 不同：像素格式改为 `Sensor.GRAYSCALE` 灰度，这是 `find_edges` 的要求。

### 第二步：初始化显示与计时器

```python
Display.init(Display.ST7701, width=640, height=480, fps=60, to_ide=True)
MediaManager.init() # 初始化media资源管理器
sensor.run() # 启动sensor
clock = time.clock() # 构造clock对象
```

`clock = time.clock()` 构造计时对象，用于统计帧率。

### 第三步：主循环——抓帧 + 边缘检测

```python
while True:
    os.exitpoint() # 检测IDE中断
    clock.tick()  # 记录开始时间（ms）
    img = sensor.snapshot() # 从通道0捕获一张图
    # 使用Canny边缘检测器
    img.find_edges(image.EDGE_CANNY, threshold=(50, 80))
    # 使用简单快速边缘检测
    #dect_img.find_edges(image.EDGE_SIMPLE, threshold=(100, 255))
    # 显示图片
    Display.show_image(img)
    print(clock.fps()) # 打印FPS
```

关键函数说明：

- `clock.tick()`：每轮循环开始时调用，记录起点；`clock.fps()`：计算从上次 tick 到现在的帧率。
- `img.find_edges(image.EDGE_CANNY, threshold=(50, 80))`：**原地**修改 img，把图像变成只有边缘线的黑白图。第一参数选检测器（Canny/SIMPLE），第二参数是 (低阈值, 高阈值)。
- 注释行展示了另一种用法：`image.EDGE_SIMPLE, threshold=(100, 255)`，想对比效果可取消注释并注释掉 Canny 那行（注意注释行里的变量名 `dect_img` 是例程笔误，实际应写成 `img`）。
- `Display.show_image(img)`：分辨率与显示缓冲一致（都是 640x480），所以不需要 x、y 偏移。

### 第四步：异常处理与资源释放

`finally` 块中依次执行 `sensor.stop()`、`Display.deinit()`、`MediaManager.deinit()`，保证 IDE 停止后资源被正确释放。

**可调参数**：
- `threshold=(50, 80)`：核心参数，在 main.py 第 41 行。光线暗/物体对比度低时可降到 (30, 60)；画面噪点多、线条太密时可升到 (80, 120)。
- `EDGE_CANNY` ↔ `EDGE_SIMPLE`：切换检测器比较效果与帧率。
- `set_framesize(Sensor.VGA)`：改成 `Sensor.QVGA` 可显著提高帧率，适合快速验证。

## 5. 运行效果

- **屏幕画面**：LCD 和 IDE 中显示黑底白线的轮廓图——你的手、桌面物品、盒子的边框都会以细白线勾勒出来，平坦区域全黑。
- **串口打印**：终端每帧打印一次当前帧率（如 `52.3`），VGA + Canny 下帧率一般在几十 FPS。
- **成功现象**：移动物体时轮廓线实时跟随，线条细而连续。
- **失败现象排查**：
  - 画面几乎全黑（没有线）→ 阈值太高或场景对比度太低，把 threshold 调小、增强光照；
  - 画面全是密密麻麻白点 → 阈值太低或噪点太多，调大 threshold；
  - 全白一片 → 摄像头过曝，降低环境亮度。

## 6. 代码使用说明

1. 用 **CanMV IDE** 打开本文件夹的 `main.py`（IDE 版本需与板载 **CanMV K230D 固件**匹配，使用正点原子出厂配套版本）。
2. USB 线连接开发板，点击 IDE 左下角连接按钮。
3. 点击运行按钮，画面出现在 LCD 与 IDE 帧缓冲区，串口终端打印帧率。
4. 停止运行点 IDE 停止按钮即可，资源会自动释放。
5. 本例程**不需要额外文件**，无 SD 卡资源依赖；脱机运行可把 main.py 放到 TF 卡 `/sdcard/` 目录。

> [!NOTE]
> 边缘检测是纯软件算法，不需要 KPU/AI 模型文件，下载即跑。

## 7. 拓展思考

1. **阈值扫描实验**：把 threshold 做成变量，每隔几秒自动加 10，观察轮廓线条从"密集"到"稀疏"的变化，找到你桌面的最佳阈值。
2. **边缘 + 绘制**：在 find_edges 之后用 `img.draw_string_advanced` 在画面角落叠加中文"帧率: xx"，做一个带 OSD 的边缘检测器。
3. **为巡线做准备**：对着黑白打印的粗线做边缘检测，观察线条边缘呈"双轨"状的现象，思考实验7 的线性回归为什么直接处理灰度图而不需要边缘检测。

## 8. 常见问题

**Q1：为什么必须用 GRAYSCALE 而不是 RGB565？**
A：`find_edges` 是灰度算法，只处理亮度信息；彩色图需要先转灰度，直接用 `Sensor.GRAYSCALE` 输出可以省去转换、提高速度。

**Q2：为什么边缘检测后原图就"没了"，看不到彩色画面？**
A：`find_edges` 是原地操作，直接把 img 修改成边缘图。想同时保留原图，可以先 `img.copy()` 一份再对副本做检测（注意内存开销）。

**Q3：threshold 两个数字分别代表什么，该怎么调？**
A：(低阈值, 高阈值)。梯度高于高阈值的点直接算边缘，低于低阈值的直接丢弃，中间的看是否与强边缘相连。想线条更少更"干净"就调大，想捕捉更多细节就调小。

**Q4：为什么代码里注释掉的 EDGE_SIMPLE 行写的是 `dect_img`？**
A：这是例程源码中的笔误（应为 `img`），如果你想启用 SIMPLE 模式，请自行把变量名改成 `img.find_edges(image.EDGE_SIMPLE, threshold=(100, 255))`。

**Q5：帧率打印太多刷屏，能关掉吗？**
A：删除或注释 `print(clock.fps())` 即可，帧率只是调试信息，不影响功能。
