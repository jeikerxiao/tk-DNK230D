# 实验10 色块计数实验

> 数出画面中有几个指定颜色的色块，把数量显示在屏幕上，是"视觉数数"的入门实验。难度：**初级**

## 1. 例程简介

本例程用 `img.find_blobs()` 按 LAB 阈值找出画面中所有目标颜色（默认黄色）色块，每个色块画白色外框和中心十字，并用 `len(blobs)` 统计数量、以"Num: N"显示在画面左上角。学习知识点是 blob 列表与计数、blob 索引访问（矩形与质心）、屏幕 OSD 文字叠加。难度为**初级**，预期用途是数零件、数小球、数色块等"有几个"类应用，也是后续按数量做决策（如分拣）的基础。

## 2. 硬件连接与原理

用到的外设（以 main.py 实际代码为准）：

| 外设 | 代码依据 | 说明 |
| --- | --- | --- |
| MIPI CSI 摄像头 | `Sensor(width=1280, height=960)`，`set_framesize(Sensor.VGA)`，`set_pixformat(Sensor.RGB565)` | 输出 VGA（640x480）RGB565 **彩色**图像 |
| 2.4 寸 MIPI LCD | `Display.init(Display.ST7701, width=640, height=480, fps=90, to_ide=True)` | ST7701 驱动，同步输出 IDE |

原理通俗讲解：计数的核心就一句话——**find_blobs 返回的列表里有几个元素，画面里就有几个色块**。find_blobs 会把所有命中颜色阈值的连通像素区域分别打包成一个 blob 对象放进列表，Python 内置函数 `len()` 取列表长度即得数量。要让"数量"准确，关键是让每个物体恰好对应一个 blob：物体之间要分开放（粘连会被算成一个），噪点要被 pixels_threshold 过滤掉。

> [!WARNING]
> 新手风险提醒：摄像头/LCD 排线必须**断电插拔**，插反可能烧毁器件；使用 5V/2A 以上稳定电源，供电不足会导致画面异常。

## 3. 相关前置知识

- **LAB 颜色空间与阈值六元组**：(l_lo, l_hi, a_lo, a_hi, b_lo, b_hi)，L 亮度、A 红绿轴、B 黄蓝轴。本例默认黄色阈值 `(18, 72, -13, 31, 18, 83)`——B 通道 18~83（偏黄）是主要特征。
- **blob（色块）**：一片连通的命中像素区域。
- **pixels_threshold=100**：像素数少于 100 的 blob 丢弃，过滤噪点；本例比实验8 的 200 更宽松，小物体也能数到。
- **blob 索引访问**：`blob[0]~blob[3]` 是外接矩形 x、y、w、h；`blob[5]、blob[6]` 是色块**质心坐标 cx、cy**（等效于 `blob.cx()`、`blob.cy()`）。
- **len() 计数**：Python 内置函数，返回列表元素个数。
- **OSD 叠加**：`draw_string_advanced` 把文字画在实时画面上（On-Screen Display），本例显示计数结果。
- **帧率 FPS**：clock 对象统计。

## 4. 软件代码逻辑说明

### 第一步：定义颜色阈值

```python
# 颜色识别阈值 (l_lo, l_hi, a_lo, a_hi, b_lo, b_hi) 即LAB模型
thresholds = [(18, 72, -13, 31, 18, 83)] # 识别物体的颜色阈值，默认黄色色块
```

只有一组阈值，默认针对黄色物体。换成你自己的物体颜色请用 IDE 阈值编辑器重取。

### 第二步：初始化摄像头与显示

```python
sensor = Sensor(width=1280, height=960) # 构建摄像头对象
sensor.reset() # 复位和初始化摄像头
sensor.set_framesize(Sensor.VGA)    # 设置帧大小VGA(640x480)，默认通道0
sensor.set_pixformat(Sensor.RGB565) # 设置输出图像格式，默认通道0

Display.init(Display.ST7701, width=640, height=480, fps=90, to_ide=True)
MediaManager.init() # 初始化media资源管理器
sensor.run()        # 启动sensor
clock = time.clock() # 构造clock对象
```

### 第三步：主循环——找色块、画标记、计数显示

```python
while True:
    os.exitpoint() # 检测IDE中断
    clock.tick()   # 记录开始时间（ms）
    img = sensor.snapshot() # 从通道0捕获一张图
    blobs = img.find_blobs([thresholds[0]], pixels_threshold=100) # 0,1,2分别表示红，绿，蓝色。
    for blob in blobs:
        img.draw_rectangle(blob[0], blob[1], blob[2], blob[3], color=(255, 255, 255),  thickness=4)
        img.draw_cross(blob[5], blob[6], color=(255, 255, 255),  thickness=4)

    # 显示计算信息
    img.draw_string_advanced(0, 0, 30, 'Num: ' + str(len(blobs)), color=(255, 255, 255))
    # 显示图片
    Display.show_image(img)
    print(clock.fps()) # 打印FPS
```

关键代码说明：

- `img.find_blobs([thresholds[0]], pixels_threshold=100)`：查找所有黄色色块，面积小于 100 像素的忽略，返回列表 blobs（没有目标时是空列表）。
- `img.draw_rectangle(blob[0], blob[1], blob[2], blob[3], ...)`：用 blob 前 4 个索引值（x, y, w, h）画白色外框，thickness=4。
- `img.draw_cross(blob[5], blob[6], ...)`：blob[5]、blob[6] 是质心 cx、cy，在色块中心画白色十字标记。
- `'Num: ' + str(len(blobs))`：把色块个数转成字符串拼到 "Num: " 后面，`draw_string_advanced(0, 0, 30, ...)` 以 30 号字显示在画面左上角（该函数支持中文，也可显示中文提示）。
- 注意：**计数就是这一行的 `len(blobs)`**，不需要额外循环累加。

### 第四步：异常处理与资源释放

`finally` 中依次 `sensor.stop()` → `Display.deinit()` → `MediaManager.deinit()`。

**可调参数**：
- `thresholds`（第 26 行）：按实际物体颜色重取。
- `pixels_threshold=100`（第 44 行）：数小物体（远处小球）可调小到 50；背景有同色噪点时调大到 300。
- 显示位置/字号：`draw_string_advanced(0, 0, 30, ...)` 的前三个参数是 x、y、字号。

## 5. 运行效果

- **屏幕画面**：摄像头实时画面中，每个黄色色块被白色矩形框套住、中心有白色十字；画面左上角白色大字显示 `Num: 3` 这样的计数，随物体增减实时变化。
- **串口打印**：每帧打印一次帧率数字。
- **成功现象**：在桌面上分开摆放 3 个黄色小球/积木，屏幕显示 Num: 3；拿走一个变 Num: 2。
- **失败现象排查**：
  - 数量始终为 0 → 阈值与物体颜色不匹配，用 IDE 阈值编辑器重取；
  - 数量比实际多 → 背景有同色杂物、或噪点通过过滤，清理背景、调大 pixels_threshold；
  - 两个物体只数出 1 个 → 两物体靠太近像素连通了，分开摆放（find_blobs 把连通区域算一个 blob）。

## 6. 代码使用说明

1. 用 **CanMV IDE** 打开本文件夹 `main.py`（IDE 与板载 **CanMV K230D 固件**版本需匹配）。
2. USB 连接开发板，点击 IDE 连接按钮，再点击运行。
3. 取阈值：IDE"工具 → 机器视觉 → 阈值编辑器"，选帧缓冲区为图像源，对着你的目标物体取 LAB 阈值，替换第 26 行。
4. LCD 与 IDE 同步显示计数结果，串口终端打印帧率。
5. 本例程**不需要额外文件**、无 SD 卡依赖；脱机自启动将 main.py 放入 TF 卡 `/sdcard/`。

> [!NOTE]
> 数数实验对"物体分开摆放"要求较高：目标之间至少留 1~2 厘米空隙，且与背景颜色差异明显。

## 7. 拓展思考

1. **数量报警**：当 `len(blobs)` 不等于预设值（如 5）时，通过 UART 发送报警帧，或驱动蜂鸣器（参考正点原子蜂鸣器例程）提示"数量不对"，模拟产线质检。
2. **多颜色分别计数**：结合实验9 的循环写法，对红、绿、蓝各数一次，屏幕分三行显示 `R:2 G:1 B:3`，做颜色分类统计器。
3. **稳定计数（消抖）**：连续 10 帧取数量的众数/平均值再显示，避免单帧误检导致数字跳动，体会"多帧滤波"思想。

## 8. 常见问题

**Q1：为什么两个挨着的小球只数出 1 个？**
A：find_blobs 按"连通区域"划分 blob，两球接触或间隙极小时命中像素连成一片，就被当成一个色块。把物体分开放；若必须紧挨，可尝试给 find_blobs 传 `merge=False` 并配合腐蚀类预处理（进阶），或改用形状特征区分。

**Q2：blob[5]、blob[6] 是什么？为什么不用 blob.cx()？**
A：blob 对象支持下标访问：0~3 是外接矩形 x/y/w/h，5/6 是质心 cx/cy。`blob[5]` 与 `blob.cx()` 等价，例程用下标写法只是风格不同。

**Q3：数量在 2 和 3 之间来回跳，不稳定？**
A：某个色块处于 pixels_threshold 边缘（时而被过滤时而保留），或光照闪烁。对策：把该物体挪近一点让像素数远离阈值边界；调整 pixels_threshold；固定光源；应用层做多帧滤波（见拓展3）。

**Q4：为什么本例阈值只有一组，注释却写"0,1,2分别表示红绿蓝"？**
A：注释是从实验8/9 沿用的模板文字，本例 thresholds 列表实际只有 1 组黄色阈值，`thresholds[0]` 取的就是它。你要换颜色直接替换这组六元组即可。

**Q5：能把计数结果显示成中文吗？**
A：可以。`draw_string_advanced` 支持中文矢量字体，例如：`img.draw_string_advanced(0, 0, 30, '数量: ' + str(len(blobs)), color=(255,255,255))`。
