# 实验13 二维码识别实验

> 用 find_qrcodes 识别画面中的二维码，把内容和位置框显示出来，是扫码应用的入门实验。难度：**初级**

## 1. 例程简介

本例程调用 `img.find_qrcodes()` 识别摄像头画面中的 QR 二维码，用红色矩形框标出二维码位置，并把解码出的**内容字符串**叠加显示在框上方、同时打印到串口。学习知识点是二维码的结构常识、find_qrcodes 用法、payload 与 rect。难度为**初级**，预期用途是扫码读取网址/文本、扫码触发设备动作（扫到指定内容点灯、开门等）。

## 2. 硬件连接与原理

用到的外设（以 main.py 实际代码为准）：

| 外设 | 代码依据 | 说明 |
| --- | --- | --- |
| MIPI CSI 摄像头 | `Sensor(width=1280, height=960)`，`set_framesize(Sensor.VGA)`，`set_pixformat(Sensor.RGB565)` | 输出 VGA（640x480）RGB565 图像 |
| 2.4 寸 MIPI LCD | `Display.init(Display.ST7701, width=640, height=480, fps=90, to_ide=True)` | ST7701 驱动，同步输出 IDE |

原理通俗讲解：QR 码是黑白方块组成的矩阵码，三个角上有"回"字形**定位图案**，算法先找到这种特征确定码的位置和方向，再按标准采样每个黑白格（模块）还原成比特流，经过纠错解码得到文本内容。摄像头分辨率要足够让每个模块占好几个像素，所以 VGA（640x480）比 QVGA 更容易识别小码。识别是纯软件算法，**不使用 KPU、不需要模型文件**。

> [!WARNING]
> 新手风险提醒：摄像头/LCD 排线必须**断电插拔**，插反可能烧毁器件；请使用 5V/2A 以上稳定电源。

## 3. 相关前置知识

- **QR 二维码**：矩阵式二维码，可存网址、文本、数字等；自带纠错能力，轻微污损也能识别。
- **模块（module）**：二维码里最小的黑白方格，一个模块代表一个比特；画面里一个模块至少要占 2~3 个像素才可能解出。
- **payload（载荷）**：二维码里存的实际内容字符串，`code.payload()` 获取。
- **rect()**：二维码的外接矩形 (x, y, w, h)，`code.rect()` 返回。
- **find_qrcodes**：CanMV 内置二维码检测解码函数，遍历返回 qrcode 对象列表，可同时识别多个码。
- **帧率 FPS**：clock 对象统计；画面中没有二维码时 find_qrcodes 仍会做全图搜索，帧率略低于纯显示。
- **OSD 文字叠加**：`draw_string_advanced` 把解码结果显示在实时画面上。

## 4. 软件代码逻辑说明

### 第一步：初始化摄像头与显示

```python
sensor = Sensor(width=1280, height=960) # 构建摄像头对象
sensor.reset()    # 复位和初始化摄像头
sensor.set_framesize(Sensor.VGA)    # 设置帧大小VGA(640x480)，默认通道0
sensor.set_pixformat(Sensor.RGB565) # 设置输出图像格式，默认通道0

Display.init(Display.ST7701, width=640, height=480, fps=90, to_ide=True)
MediaManager.init() # 初始化media资源管理器
sensor.run() # 启动sensor
clock = time.clock() # 构造clock对象
```

### 第二步：主循环——找码、画框、显示内容

```python
while True:
    os.exitpoint() # 检测IDE中断
    clock.tick()   # 记录开始时间（ms）
    img = sensor.snapshot() # 从通道0捕获一张图

    # 遍历图像中的二维码
    for code in img.find_qrcodes():
        rect = code.rect()
        img.draw_rectangle([v for v in rect], color=(255, 0, 0), thickness=4)
        img.draw_string_advanced(rect[0], rect[1], 32, code.payload())
        print(code)

    # 显示图片
    Display.show_image(img)
    print(clock.fps()) # 打印FPS
```

关键函数说明：

- `img.find_qrcodes()`：全图搜索二维码，返回 qrcode 对象列表；可选参数 `roi=(x,y,w,h)` 限定区域提速（本例未用）。
- `code.rect()`：外接矩形 (x, y, w, h)，`[v for v in rect]` 转成列表传给 `draw_rectangle` 画红色粗框（thickness=4）。
- `code.payload()`：解码出的内容字符串，用 `draw_string_advanced(rect[0], rect[1], 32, ...)` 以 32 号字显示在框的左上角。
- `print(code)`：把 qrcode 对象信息（位置、内容、版本等）打印到串口。
- qrcode 对象还有 `code.version()`（QR 版本号，决定码的大小规格）、`code.rotation()`（旋转弧度）等方法，本例未使用。

### 第三步：异常处理与资源释放

`finally` 中依次 `sensor.stop()` → `Display.deinit()` → `MediaManager.deinit()`，保证 IDE 停止运行后资源被正确释放。

> [!NOTE]
> 内容太长时 `draw_string_advanced` 会画出屏幕边界，可先截断再显示：`img.draw_string_advanced(rect[0], rect[1], 24, code.payload()[:20])`（只显示前 20 个字符）。

**可调参数**：
- 可给 `find_qrcodes` 加 `roi`：只扫画面中央区域（如 `roi=(160, 80, 320, 320)`），提高帧率、减少误扫背景中的其他码。
- `set_framesize`：VGA 保证小码可识别；若只用大码且追求帧率可试 QVGA。
- 显示字号 32（第 45 行）：内容太长时可改小，或截断 payload。

## 5. 运行效果

- **屏幕画面**：摄像头画面中的每个二维码被红色粗框套住，框左上角显示解码内容（如网址 `www.alientek.com` 或任意文本）；手机屏幕上的二维码也能识别。
- **串口打印**：识别到码时每帧打印 qrcode 对象详情；另外每帧打印一次帧率数字。
- **成功现象**：把打印的二维码放在镜头前 15~40cm，红框和内容稳定显示；同一画面多个码可同时识别。
- **失败现象排查**：
  - 识别不到 → 距离太近/太远导致码太小或虚焦，调整到 20~40cm；屏幕反光（手机扫码时）调整角度；
  - 打印的码识别率低 → 打印模糊或纸张反光，换激光打印件、避免覆膜反光；
  - 内容显示成乱码/方块 → payload 含非 ASCII 字符（draw_string_advanced 支持中文，但特殊符号可能缺字形），以串口打印为准。

## 6. 代码使用说明

1. 用 **CanMV IDE** 打开本文件夹 `main.py`（IDE 与板载 **CanMV K230D 固件**版本需匹配）。
2. USB 连接开发板，点击 IDE 连接按钮，再点击运行。
3. 准备一个二维码：任意二维码生成网站生成后打印，或直接用手机打开一张二维码图片。
4. LCD 与 IDE 同步显示识别结果，串口终端打印码信息与帧率。
5. 本例程**不需要额外文件**、无 SD 卡依赖；脱机自启动将 main.py 放入 TF 卡 `/sdcard/`。

> [!NOTE]
> 二维码识别对光照要求不高，但要避免强反光覆盖码面；倾斜 30° 以内一般都能识别。

## 7. 拓展思考

1. **扫码开灯**：判断 `code.payload()` 是否等于预设口令（如 "LED ON"），是则点亮蓝色 LED（参考 GPIO 例程），做一个"扫码开关"。
2. **扫码记录**：识别到新内容（与上次不同）时把 payload 追加写入 `/sdcard/scan_log.txt`（用 open(..., 'a')），做一个简易扫码盘点记录器。
3. **ROI 收银台模式**：只识别画面中央固定区域（加 roi 参数）并把区域框画出来，模拟收银台扫码窗，观察帧率提升。

## 8. 常见问题

**Q1：为什么二维码要放得很近才识别得到？**
A：解码要求每个模块至少占 2~3 个像素。VGA 画面下，25x25 模块的标准码在 30cm 左右大小合适；太远的码模块不足 1 像素，算法无从采样。靠近一点或打印大一点的码即可。

**Q2：手机屏幕上的二维码能扫吗？为什么有时候不行？**
A：能扫，但手机屏幕自发光且反光，摄像头容易过曝。把手机亮度调低、倾斜一点角度避开反光即可稳定识别。

**Q3：find_qrcodes 会不会把条形码、DM 码也识别出来？**
A：不会。find_qrcodes 只识别 QR 码；条形码用实验14 的 find_barcodes，Data Matrix 用实验16 的 find_datamatrices，各函数算法独立。

**Q4：画面里有两个二维码，只显示了一个的内容？**
A：for 循环会处理每个码，但两个 `draw_string_advanced` 都画在各自 rect 左上角，如果两码位置接近文字可能互相覆盖；另外两码内容相同时肉眼看似只有一处。以串口打印为准核对数量。

**Q5：识别时帧率明显下降，正常吗？**
A：正常。二维码搜索需要多尺度滑窗检测，计算量大。画面中没有码时是全图盲搜，最耗时；加 roi 缩小搜索范围是最有效的提速手段。
