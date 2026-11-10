# 实验16 DM码识别实验

> 用 find_datamatrices 识别 Data Matrix（DM）工业二维码，显示行数、列数、内容和旋转角。难度：**初级**

## 1. 例程简介

本例程调用 `img.find_datamatrices()` 识别画面中的 Data Matrix 码（简称 DM 码，工业领域常用的小型矩阵码），用红框标出位置，把解码内容显示在画面左上角，串口打印矩阵行列数、内容、旋转角和帧率。学习知识点是 DM 码的结构特征（L 形实边 + 虚边）、find_datamatrices 的 margin 参数、rows/columns 信息。难度为**初级**，预期用途是工业零件打标识读、电子元件追溯、小尺寸标签扫描——DM 码在很小面积上也能存信息，是制造业的主流选择。

## 2. 硬件连接与原理

用到的外设（以 main.py 实际代码为准）：

| 外设 | 代码依据 | 说明 |
| --- | --- | --- |
| MIPI CSI 摄像头 | `Sensor(width=1280, height=960)`，`set_framesize(Sensor.VGA)`，`set_pixformat(Sensor.RGB565)` | 输出 VGA（640x480）RGB565 图像 |
| 2.4 寸 MIPI LCD | `Display.init(Display.ST7701, width=640, height=480, fps=90, to_ide=True)` | ST7701 驱动，同步输出 IDE |

原理通俗讲解：DM 码和 QR 码同属矩阵码，但长相不同——它**没有 QR 的三个"回"字定位角**，而是靠两条相邻的**实心 L 形边**（定位边）和两条对面的**虚线边**（定时图案）来确定码的位置、方向和格子数。识别流程是：先找黑白交替的"虚线边"特征定位候选区域，再按 L 形边校正方向，逐格采样黑白模块，解码得到内容。由于模块密度高，**要求每个模块在画面中占足够像素**，所以本例用 VGA 分辨率。

> [!WARNING]
> 新手风险提醒：摄像头/LCD 排线必须**断电插拔**，插反可能烧毁器件；请使用 5V/2A 以上稳定电源。

## 3. 相关前置知识

- **Data Matrix 码**：ECC200 标准的矩阵条码，常见 10x10 到 144x144 模块，可存文本/数字；工业零件激光打标常用，耐磨损、小尺寸。
- **L 形定位边与虚线定时边**：DM 码左边和下边是实线（L 形），上边和右边是黑白相间虚线，用于定位与定向——这是与 QR 码最直观的区别。
- **rows() / columns()**：`matrix.rows()`、`matrix.columns()` 返回该 DM 码的行数和列数（模块数），如 12x12、16x24（长方形 DM 码也合法）。
- **payload（载荷）**：解码出的内容字符串，`matrix.payload()` 获取。
- **rotation 与弧度转角度**：`matrix.rotation()` 返回弧度，`(180 * matrix.rotation()) / math.pi` 换算为角度。
- **margin（搜索余量）**：find_datamatrices 可选参数，指在码四周额外搜索的像素余量；本例未显式传入，使用默认值。
- **帧率 FPS**：clock 对象统计（本例帧率信息拼在识别日志里打印，主循环没有单独打印 fps 语句）。

## 4. 软件代码逻辑说明

### 第一步：初始化摄像头与显示

```python
sensor = Sensor(width=1280, height=960) # 构建摄像头对象
sensor.reset() # 复位和初始化摄像头
sensor.set_framesize(Sensor.VGA)    # 设置帧大小VGA(640x480)，默认通道0
sensor.set_pixformat(Sensor.RGB565) # 设置输出图像格式，默认通道0

Display.init(Display.ST7701, width=640, height=480, fps=90, to_ide=True)
MediaManager.init() # 初始化media资源管理器
sensor.run() # 启动sensor
clock = time.clock() # 构造clock对象
```

### 第二步：主循环——找 DM 码、画框、显示与打印

```python
while True:
    os.exitpoint() # 检测IDE中断
    clock.tick()  # 记录开始时间（ms）
    img = sensor.snapshot() # 从通道0捕获一张图

    # 遍历图像中的二维码
    for matrix in img.find_datamatrices():
        # 绘制识别到的 Data Matrix 码的矩形框
        img.draw_rectangle([v for v in matrix.rect()], color=(255, 0, 0))
        img.draw_string_advanced(0, 0, 30, matrix.payload(), color=(255, 255, 255)) # 图像显示DM码信息
        print_args = (matrix.rows(), matrix.columns(), matrix.payload(), (180 * matrix.rotation()) / math.pi, clock.fps())
        print("矩阵 [%d:%d], 内容 \"%s\", 旋转 %f (度), FPS %f" % print_args)

    # 显示图片
    Display.show_image(img)
```

关键函数说明：

- `img.find_datamatrices()`：全图搜索 DM 码，返回 datamatrix 对象列表，可同时识别多个；可选参数：
  - `roi=(x,y,w,h)`：限定搜索区域提速（本例未用）；
  - `margin=20`：码四周的额外搜索余量像素，码贴边或检测不稳定时可调整（本例未用，走默认值）。
- `matrix.rect()`：外接矩形 → `draw_rectangle` 画红框（本例用默认线宽）。
- `matrix.payload()`：内容字符串，白色 30 号字显示在画面左上角 (0, 0)。
- `matrix.rows() / matrix.columns()`：矩阵行列数，与内容一起用 `%` 格式化打印。
- `clock.fps()`：帧率拼进同一条识别日志（注意本例主循环**没有**每帧单独 print fps 的语句，只有识别到码时才打印）。

### 第三步：异常处理与资源释放

`finally` 中依次 `sensor.stop()` → `Display.deinit()` → `MediaManager.deinit()`。

**可调参数**：
- 给 `find_datamatrices` 加 `roi`：工业工位码位置固定，加 roi 可显著提速。
- `margin`：识别时好时坏可尝试加大。
- `set_framesize`：VGA 是识别小码的保障；换 QVGA 帧率高但只适合大尺寸打印的 DM 码。

## 5. 运行效果

- **屏幕画面**：摄像头画面中的 DM 码被红色矩形框套住，画面左上角白色文字显示解码内容；旋转码片仍能识别。
- **串口打印**：识别到码时打印一行中文日志，如：
  ```text
  矩阵 [12:12], 内容 "ALIENTEK", 旋转 1.2 (度), FPS 20.5
  ```
  （行列数、内容、旋转角、帧率一目了然）；未识别到码时无该打印。
- **成功现象**：打印一个 DM 码（在线生成工具选 Data Matrix 格式）放镜头前 15~40cm，红框和内容稳定出现。
- **失败现象排查**：
  - 识别不到 → 打印太小（模块像素不足），放大打印或拉近；喷墨打印发虚，换激光打印；
  - 手机屏幕上的 DM 码识别不稳 → 屏幕摩尔纹干扰，调低亮度、微倾角度，优先用打印件；
  - 内容乱码 → payload 含特殊字符时屏显可能缺字形，以串口打印为准。

## 6. 代码使用说明

1. 用 **CanMV IDE** 打开本文件夹 `main.py`（IDE 与板载 **CanMV K230D 固件**版本需匹配）。
2. USB 连接开发板，点击 IDE 连接按钮，再点击运行。
3. 准备 DM 码：用在线条码生成器选 "Data Matrix" 类型生成（内容如 HELLO、序列号），激光打印成 2cm 见方以上。
4. LCD 与 IDE 同步显示识别结果，串口终端打印详情。
5. 本例程**不需要额外文件**、无 SD 卡依赖；脱机自启动将 main.py 放入 TF 卡 `/sdcard/`。

> [!NOTE]
> DM 码识读对打印质量敏感：模块边界要锐利、黑白对比要强，打印时勿用"省墨"模式，勿缩放导致模块变形。

## 7. 拓展思考

1. **工单扫描器**：识别到的 payload 与预设工单号列表比对，匹配则通过 UART 发送 "OK,编号"、不匹配发送 "NG"，模拟产线核验工位。
2. **多码管理**：画面中放多个不同内容的 DM 码，用 dict 记录每个 `matrix.payload()` 首次出现的时间，统计"今天扫到过哪些码"，练习数据结构应用。
3. **与二维码实验对比**：同一场景分别运行实验13（QR）和本实验（DM），比较两种码在不同打印尺寸下的识别距离，写一个自己的"选型结论"，理解为什么工业上偏爱 DM 码。

## 8. 常见问题

**Q1：DM 码和 QR 码怎么一眼区分？识别函数能混用吗？**
A：QR 码有三个"回"字形定位角，DM 码没有——它是 L 形实边加虚线边。函数不能混用：QR 用 `find_qrcodes()`，DM 码用 `find_datamatrices()`，互相都识别不了对方的码。

**Q2：为什么我的 DM 码打印出来只有指甲盖大就识别不了？**
A：DM 码模块密度高，12x12 的码在 VGA 画面中要求每个模块至少占 2~3 像素。码太小就必须凑很近，而太近又会虚焦。建议打印不小于 1.5cm 见方，工作距离 15~40cm。

**Q3：串口打印的"矩阵 [12:12]"是什么意思？**
A：`matrix.rows()` 和 `matrix.columns()`，即该 DM 码是 12 行 x 12 列的模块矩阵。DM 码也支持长方形规格，如 [8:18]。

**Q4：本例为什么主循环里没有单独 print(clock.fps())？**
A：例程把帧率拼进了识别日志（`print_args` 的最后一项），只有识别到 DM 码的那一帧才会打印 FPS；没识别到码时无输出。想持续观察帧率可自己在 `Display.show_image(img)` 后加一行 `print(clock.fps())`。

**Q5：find_datamatrices 帧率低，如何提速？**
A：DM 码搜索需多方向扫描虚线边特征，计算量大。最有效手段是加 `roi` 限定码出现的区域；其次是保持 VGA 下简化背景（避免大面积复杂纹理）；若码打印得大，可降分辨率到 QVGA 换帧率。
