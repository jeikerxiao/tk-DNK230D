# 实验7 快速线性回归（循线）实验

> 用 get_regression 对二值化后的黑线做线性回归，一条直线拟合出线的走向，是巡线小车的核心算法。难度：**中级**

## 1. 例程简介

本例程把摄像头灰度画面按灰度阈值二值化（黑白化），再调用 `img.get_regression()` 对所有白色像素做**线性回归**，拟合出一条最贴合黑线的直线，画在屏幕上并显示线的角度 theta。学习知识点是灰度阈值、二值化、线性回归、magnitude 拟合质量，以及巡线的基本思路。难度为**中级**，预期用途是黑白线巡迹小车（配合实验12）的核心视觉环节——快速、稳定地算出"线朝哪个方向"。

## 2. 硬件连接与原理

用到的外设（以 main.py 实际代码为准）：

| 外设 | 代码依据 | 说明 |
| --- | --- | --- |
| MIPI CSI 摄像头 | `Sensor(width=1280, height=960)`，`set_framesize(Sensor.VGA)`，`set_pixformat(Sensor.GRAYSCALE)` | 输出 VGA（640x480）**灰度**图像 |
| 2.4 寸 MIPI LCD | `Display.init(Display.ST7701, width=640, height=480, fps=90, to_ide=True)` | ST7701 驱动，同步输出 IDE |

原理通俗讲解：巡线的本质是"在视野里找到线，并算出线的方向"。做法分两步：

1. **二值化**：`img.binary([(0, 100)])` 把灰度值落在 0~100（暗，即黑线）的像素变成白色 255，其余变黑色 0——整幅图只剩黑白两色，黑线变成"白色像素团"。
2. **线性回归**：`get_regression` 对这些白色像素做数学上的"最小二乘拟合"，求出一条最能代表这团像素走向的直线。它比霍夫找线快得多，因为不需要投票搜索，直接一步算出斜率和截距，所以叫"快速线性回归"，非常适合巡线这种要求高帧率的场合。

> [!WARNING]
> 新手风险提醒：摄像头/LCD 排线必须**断电插拔**，插反可能烧毁器件；巡线演示常在地面进行，注意 USB 线拉扯，建议使用充电宝供电做脱机测试；保证 5V 供电稳定。

## 3. 相关前置知识

- **灰度图**：每像素 0（黑）~255（白）的亮度值，无颜色信息。
- **灰度阈值 THRESHOLD=(0,100)**：灰度值在 [0,100] 区间内的像素被视为"黑线"。区间要根据现场光照调整——线不够黑或环境太暗时可能要改成 (0, 60)。
- **二值化 binary**：把图像变成纯黑白两色。`BINARY_VISIBLE = True` 时屏幕上显示二值化后的图，你能**亲眼看到算法"看到"了什么**（代价是额外计算，帧率略降）；设为 False 则显示原始灰度图，回归直接对灰度阈值做。
- **线性回归**：统计方法，给一堆散点找一条"总体最贴合"的直线（各点到线的距离平方和最小）。
- **line 对象的方法**：注释中列出 `x1(), y1(), x2(), y2(), length(), theta(), rho(), magnitude()`——theta 是线的角度（度），巡线转向就靠它。
- **magnitude（拟合质量）**：范围 (0, INF]，0 表示像素团是个圆（无方向），值越大说明像素越"成一条线"，回归结果越可信。巡线时可用 magnitude 判断"是否还看得到线"。
- **帧率 FPS**：clock 对象统计（本例 fps 打印被注释掉了）。

## 4. 软件代码逻辑说明

### 第一步：全局参数

```python
THRESHOLD = (0, 100)  # 黑白图像的灰度阈值
BINARY_VISIBLE = True # 使用二值化图像你可以看到什么是线性回归。
                      # 这可能降低帧率
```

### 第二步：初始化摄像头（灰度）与显示

```python
sensor = Sensor(width=1280, height=960) # 构建摄像头对象
sensor.reset() # 复位和初始化摄像头
sensor.set_framesize(Sensor.VGA)    # 设置帧大小VGA(640x480)，默认通道0
sensor.set_pixformat(Sensor.GRAYSCALE) # 设置输出图像格式，默认通道0

Display.init(Display.ST7701, width=640, height=480, fps=90, to_ide=True)
MediaManager.init() # 初始化media资源管理器
sensor.run()        # 启动sensor
clock = time.clock() # 构造clock对象
```

### 第三步：主循环——二值化 + 回归 + 绘制

```python
while True:
    os.exitpoint() # 检测IDE中断
    clock.tick()   # 记录开始时间（ms）
    img = sensor.snapshot() # 从通道0捕获一张图
    img = img.binary([THRESHOLD]) if BINARY_VISIBLE else img
    # 返回一个类似于find_lines()和find_line_segments()方法返回的对象.
    # 有以下方法：x1(), y1(), x2(), y2(), length(),
    # theta() (rotation in degrees), rho(), 和 magnitude().
    line = img.get_regression([(255,255) if BINARY_VISIBLE else THRESHOLD])
    if (line):
        img.draw_line(line.line(), color=127, thickness=5)
        # LCD显示theta角度
        img.draw_string_advanced(0, 0, 24, str('%.1f' % line.theta()), color=(255,255,255), thickness=4)
        print(line) # 打印结果
    Display.show_image(img, x=round((640 - sensor.width()) / 2), y=round((480 - sensor.height()) / 2))
```

关键代码说明：

- `img = img.binary([THRESHOLD]) if BINARY_VISIBLE else img`：Python 三元表达式——BINARY_VISIBLE 为 True 时对图像做二值化（灰度 0~100 的黑线变成白色），否则保留原图。
- `img.get_regression([(255,255) if BINARY_VISIBLE else THRESHOLD])`：对指定灰度区间的像素做线性回归。二值化后画面只有 0 和 255，所以回归区间是 `(255,255)`（只取白色像素，即原来的黑线）；未二值化时直接用 `(0,100)`。返回 line 对象；若区间内没有像素则返回 None，所以要用 `if (line):` 判断。
- `img.draw_line(line.line(), color=127, thickness=5)`：把回归直线画成灰色（127）、宽 5 的线。灰度图上颜色就是一个亮度值，127 是中间灰。
- `img.draw_string_advanced(0, 0, 24, str('%.1f' % line.theta()), ...)`：在画面左上角显示线的角度，保留 1 位小数。
- `print(line)`：把 line 对象详情打印到串口。
- 被注释的第 57 行 `print("FPS %f, mag = %s" ...)` 可打印帧率和 magnitude，取消注释即可观察拟合质量。

### 第四步：异常处理与资源释放

`finally` 中依次 `sensor.stop()` → `Display.deinit()` → `MediaManager.deinit()`。

**可调参数**：
- `THRESHOLD = (0, 100)`（第 24 行）：最核心参数，取决于黑线和地面的实际灰度。在 IDE 帧缓冲区观察二值化效果：白块（原黑线）完整且无杂点为佳。
- `BINARY_VISIBLE`（第 25 行）：True 便于调试观察，False 提高帧率用于实战。
- `get_regression` 还可加 `roi=(x,y,w,h)` 只在画面下半部分回归（巡线小车常用），减少干扰。

## 5. 运行效果

- **屏幕画面**：`BINARY_VISIBLE=True` 时屏幕显示黑白二值图——原来的黑线是白色、背景是黑色，一条灰色粗直线贯穿白线中央，左上角白色数字显示当前角度（如 89.5 表示近水平）。
- **串口打印**：检出直线时每帧打印一次 line 对象信息（取消注释第 57 行还会打印 FPS 和 magnitude）。
- **成功现象**：把打印的粗黑线（或黑色胶带）放在镜头下，灰线始终贴合黑线走向；转动纸张，左上角角度值随之变化。
- **失败现象排查**：
  - line 为 None（什么都不画）→ 二值化后没有白色像素：黑线不够黑或 THRESHOLD 太小，观察二值图调整阈值；
  - 二值图上大片白色 → 环境太暗，背景也被当成"黑"，调小 THRESHOLD 上限（如 (0,60)）或加强照明；
  - 直线方向乱跳 → 画面中有其他深色物体干扰，清理背景或加 roi 限制区域。

## 6. 代码使用说明

1. 用 **CanMV IDE** 打开本文件夹 `main.py`（IDE 与板载 **CanMV K230D 固件**版本需匹配）。
2. USB 连接开发板，点击 IDE 连接按钮，再点击运行。
3. 打印一条宽 1.5~2cm 的黑线（或用黑色电工胶带贴地），摄像头距线面 15~30cm 俯视。
4. LCD 与 IDE 同步显示，串口打印回归结果。
5. 本例程**不需要额外文件**、无 SD 卡资源依赖；脱机自启动将 main.py 放入 TF 卡 `/sdcard/`。

> [!NOTE]
> 调试技巧：先把 `BINARY_VISIBLE` 保持 True，确认二值图中黑线完整清晰，再改 False 提高帧率做实际巡线。

## 7. 拓展思考

1. **巡线转向控制**：用 `line.theta()` 计算偏差——theta≈90 表示车与线平行；小于 90 向左修正、大于 90 向右修正，把修正量通过 UART 发给小车底盘（PWM 差速），完成最简巡线闭环。
2. **magnitude 丢线保护**：取消第 57 行注释观察 magnitude；当 `line.magnitude()` 小于某值或 line 为 None 时让小车原地旋转找线或停车报警。
3. **ROI 优化**：把 `get_regression` 限制在画面下方 1/3 区域（`roi=(0, 320, 640, 160)`），排除远处干扰，同时观察帧率变化，理解巡线摄像头为什么装在车头低位朝下看。

## 8. 常见问题

**Q1：get_regression 和 find_lines 都能找线，巡线该用哪个？**
A：巡线用 get_regression。它一步算出唯一一条拟合线，速度远快于霍夫投票的 find_lines，且天然只输出"最像线"的结果；find_lines 会输出多条候选直线，还需要自己挑选，适合找多条直线的场景。

**Q2：为什么回归的阈值写成 (255,255)？**
A：因为二值化之后图像里只有 0（黑）和 255（白）两种灰度，黑线已被 binary 映射成白色，所以只对 (255,255) 区间的像素回归，就是只对黑线回归。

**Q3：屏幕上一片全白或全黑，看不到线？**
A：THRESHOLD 与实际灰度不匹配。全白说明背景也被归为"暗像素"（环境太暗），全黑说明黑线灰度高于 100（线不够黑或曝光过度）。在 IDE 中临时把 BINARY_VISIBLE 改 False 看原始灰度图，用鼠标查看线和背景的灰度值，再重新设定阈值。

**Q4：theta 显示的角度是相对什么的？**
A：theta 是回归直线在图像坐标系中的角度（度）：约 90 表示水平线，约 0 或 180 表示竖直线（图像 y 轴向下）。注意倾斜拍摄时画面中的线角度会透视变形。

**Q5：为什么本例用灰度摄像头输出而不是彩色？**
A：黑白巡线只需要亮度信息，GRAYSCALE 数据量小、binary/get_regression 处理快；彩色反而会引入颜色转灰度的开销，对巡线没有增益。
