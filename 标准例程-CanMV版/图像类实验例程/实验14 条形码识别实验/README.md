# 实验14 条形码识别实验

> 用 find_barcodes 识别一维条形码（EAN13、CODE128 等 16 种类型），显示内容、类型和质量。难度：**初级**

## 1. 例程简介

本例程调用 `img.find_barcodes()` 识别画面中的一维条形码，用红框标出条码位置，串口打印条码**类型、内容、旋转角、质量值和帧率**，并把内容叠加显示在画面左上角；代码还定义了一个 `barcode_name()` 辅助函数，把 16 种条码类型常量翻译成名字。学习知识点是一维码与二维码的区别、条码类型常量、quality 质量评估、弧度转角度。难度为**初级**，预期用途是商品条码扫描、图书 ISBN 识别、物流标签读取等。

## 2. 硬件连接与原理

用到的外设（以 main.py 实际代码为准）：

| 外设 | 代码依据 | 说明 |
| --- | --- | --- |
| MIPI CSI 摄像头 | `Sensor(width=1280, height=960)`，`set_framesize(Sensor.VGA)`，`set_pixformat(Sensor.RGB565)` | 输出 VGA（640x480）RGB565 图像 |
| 2.4 寸 MIPI LCD | `Display.init(Display.ST7701, width=640, height=480, fps=90, to_ide=True)` | ST7701 驱动，同步输出 IDE |

原理通俗讲解：一维条形码是宽窄不同的黑白竖条序列，信息只沿**水平方向**编码。识别过程是：先在图像中寻找"黑白竖条交替"的区域（利用梯度找条纹特征），确定条码边界和方向后，沿一条扫描线测量每根条的宽度比例，按对应码制（EAN13、CODE128 等）的编码表解码成字符。`code.quality()` 返回解码质量，质量越高说明条纹越清晰、结果越可信。

> [!WARNING]
> 新手风险提醒：摄像头/LCD 排线必须**断电插拔**，插反可能烧毁器件；请使用 5V/2A 以上稳定电源。

## 3. 相关前置知识

- **一维条形码**：只在水平方向编码的条状码，常见码制有商品码 EAN13/EAN8/UPCA/UPCE、图书码 ISBN10/ISBN13、物流码 CODE128/CODE39、二维堆叠码 PDF417 等，本例 `barcode_name()` 列出了 CanMV 支持的全部 16 种类型常量（`image.EAN13`、`image.CODE128`……）。
- **code.type()**：返回条码类型常量，与 `image.XXX` 比较即可判断码制。
- **payload（载荷）**：条码解码出的字符串内容，如商品数字串。
- **quality（质量）**：`code.quality()` 返回整数值，表示本次解码的可靠程度，多帧观察时数值越高越可信；应用中常用"quality > 某值才采纳"做过滤。
- **rotation（旋转）与弧度制**：`code.rotation()` 返回**弧度**，代码用 `(180 * code.rotation()) / math.pi` 换算成角度——这是"弧度转角度"的标准公式（180/π ≈ 57.3）。
- **rect()**：条码外接矩形 (x, y, w, h)。
- **帧率 FPS**：clock 对象统计。

## 4. 软件代码逻辑说明

### 第一步：条码类型翻译函数

```python
# 定义条形码类型
def barcode_name(code):
    if(code.type() == image.EAN2):
        return "EAN2"
    if(code.type() == image.EAN5):
        return "EAN5"
    ...
    if(code.type() == image.CODE128):
        return "CODE128"
```

依次比较 `code.type()` 与 16 个 `image.XXX` 类型常量，返回可读的字符串名字，仅用于打印展示。

### 第二步：初始化摄像头与显示

```python
sensor = Sensor(width=1280, height=960) # 构建摄像头对象
sensor.reset()    # 复位和初始化摄像头
sensor.set_framesize(Sensor.VGA)    # 设置帧大小VGA(640x480)，默认通道0
sensor.set_pixformat(Sensor.RGB565) # 设置输出图像格式，默认通道0

Display.init(Display.ST7701, width=640, height=480, fps=90, to_ide=True)
MediaManager.init()    # 初始化media资源管理器
sensor.run()  # 启动sensor
clock = time.clock() # 构造clock对象
```

### 第三步：主循环——找条码、画框、打印详情

```python
while True:
    os.exitpoint() # 检测IDE中断
    clock.tick()   # 记录开始时间（ms）
    img = sensor.snapshot() # 从通道0捕获一张图

    # 遍历图像中所有条形码
    for code in img.find_barcodes():
        img.draw_rectangle([v for v in code.rect()], color=(255, 0, 0), thickness=4)
        # 打印相关信息
        print_args = (barcode_name(code), code.payload(), (180 * code.rotation()) / math.pi, code.quality(), clock.fps())
        print("Barcode %s, Payload \"%s\", rotation %f (degrees), quality %d, FPS %f" % print_args)
        img.draw_string_advanced(0, 0, 30, code.payload(), color = (255, 255, 255)) # 图像显示条码信息

    # 显示图片
    Display.show_image(img)
    print(clock.fps()) # 打印FPS
```

关键函数说明：

- `img.find_barcodes()`：全图搜索一维条码，返回 barcode 对象列表，可同时找到多个；可加 `roi=(x,y,w,h)` 限定区域提速（本例未用）。
- `code.rect()`：外接矩形，画红色粗框（thickness=4）。
- `print_args` 元组：把类型名、内容、旋转角度、质量、帧率打包，用 `%` 格式化一次性打印——这是把多个信息组织进一条日志的常用写法。
- `(180 * code.rotation()) / math.pi`：弧度转角度，条码倾斜摆放时该值不为 0。
- `img.draw_string_advanced(0, 0, 30, code.payload(), ...)`：白色 30 号字把内容显示在画面左上角。

### 第四步：异常处理与资源释放

`finally` 中依次 `sensor.stop()` → `Display.deinit()` → `MediaManager.deinit()`。

**可调参数**：
- 给 `find_barcodes` 加 `roi`：条码位置固定的流水线场景可大幅提速。
- quality 过滤：可自己加 `if code.quality() > 5:` 只采纳高质量结果（本例未过滤）。
- 显示字号 30 与位置 (0, 0)：内容过长时可改小字号或改画在条码框上方。

## 5. 运行效果

- **屏幕画面**：条码被红色粗框套住，画面左上角白色文字显示条码内容；条码轻微倾斜也能识别。
- **串口打印**：识别到条码时打印一行完整信息，如：
  ```text
  Barcode EAN13, Payload "6901234567890", rotation 0.5 (degrees), quality 8, FPS 25.3
  ```
  另外每帧打印一次帧率。
- **成功现象**：拿一件带商品条码的物品（饮料瓶、书本 ISBN）对着摄像头，20~40cm 距离内稳定出结果。
- **失败现象排查**：
  - 识别不到 → 条码太小/太近虚焦/反光（覆膜包装），调整距离和角度、避免顶光直射；
  - 内容错误或缺位 → quality 很低时解码可能出错，取多帧中出现次数最多的内容；
  - 只识别部分码制 → 打印质量差的条码条纹对比度不足，换激光打印件。

## 6. 代码使用说明

1. 用 **CanMV IDE** 打开本文件夹 `main.py`（IDE 与板载 **CanMV K230D 固件**版本需匹配）。
2. USB 连接开发板，点击 IDE 连接按钮，再点击运行。
3. 准备条码：商品包装条码、书本封底 ISBN，或网上生成 CODE128 条码后打印。
4. LCD 与 IDE 同步显示，串口终端打印类型/内容/角度/质量。
5. 本例程**不需要额外文件**、无 SD 卡依赖；脱机自启动将 main.py 放入 TF 卡 `/sdcard/`。

> [!NOTE]
> 测试小技巧：屏幕显示条码（手机打开条码图片）也可以识别，但反光较强；打印件成功率更高。

## 7. 拓展思考

1. **质量过滤与多次确认**：只采纳 `quality` 大于阈值的解码结果，并要求同一内容连续出现 3 帧才算"扫到"，通过 UART 输出，做一个防误读的扫码枪。
2. **商品计数器**：识别到 EAN13 类商品码时把 payload 存入集合（set），统计"扫过多少种不同商品"，显示在画面上，模拟收银场景。
3. **按类型分流**：用 `code.type()` 区分图书（ISBN13）和商品（EAN13），分别用不同颜色画框，练习类型判断逻辑。

## 8. 常见问题

**Q1：find_barcodes 能识别二维码吗？**
A：不能。一维条码和 QR 码算法不同，二维码请用实验13 的 `find_qrcodes()`；PDF417 虽是"堆叠式"条码，属于 find_barcodes 支持的类型之一。

**Q2：quality 是什么单位？多大算好？**
A：quality 是解码器给出的整数质量评分，无固定单位，越大越可信。实用做法是运行时观察你场景下的典型值，取其一半以上作为采纳门槛。

**Q3：为什么 rotation 打印出来是小数角度而不是 0？**
A：条码很少完全水平放置，rotation 反映检测到的条纹方向，`180/π` 换算后就是倾斜角度；正负号表示倾斜方向。

**Q4：`print("..." % print_args)` 里的 %s、%d、%f 是什么？**
A：Python 字符串格式化占位符：%s 填字符串、%d 填整数、%f 填浮点数，按顺序用 print_args 元组里的值替换。这是把多项信息拼成一条日志的常用手法。

**Q5：条码识别帧率低，怎么提速？**
A：条码搜索是多方向滑窗扫描，计算量大。提速手段：加 roi 只扫条码可能出现的区域；避免画面里同时有多个大面积复杂纹理；必要时降分辨率（但小条码会更难识别，需权衡）。
