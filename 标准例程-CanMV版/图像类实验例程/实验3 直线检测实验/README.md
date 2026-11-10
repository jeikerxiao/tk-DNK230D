# 实验3 直线检测实验

> 用霍夫变换在画面中找出"无限长"的直线并画出来，还能按角度筛选。难度：**初级**

## 1. 例程简介

本例程调用 `img.find_lines()`（霍夫变换直线检测）找出画面中的所有直线，用红色线绘制到屏幕上，并按角度范围 0°~179° 过滤，串口打印每条直线的参数和帧率。学习知识点是霍夫变换的直觉理解、theta/rho 直线表示法、threshold 与 margin 参数。难度为**初级**，预期用途是检测桌沿、门框、打印的黑线等长直特征，是工业视觉和巡线小车的常用基础技能。

## 2. 硬件连接与原理

用到的外设（以 main.py 实际代码为准）：

| 外设 | 代码依据 | 说明 |
| --- | --- | --- |
| MIPI CSI 摄像头 | `Sensor(width=640, height=480)`，`set_framesize(Sensor.VGA)`，`set_pixformat(Sensor.RGB565)` | 输出 VGA（640x480）RGB565 彩色图像 |
| 2.4 寸 MIPI LCD | `Display.init(Display.ST7701, width=640, height=480, fps=90, to_ide=True)` | ST7701 驱动，同步输出到 IDE |

原理通俗讲解：**霍夫变换**是一种"投票找直线"的方法。先对图像做边缘检测找出轮廓点，然后让每个边缘点给"所有可能经过它的直线"投一票，得票超过门槛的直线就是真实存在的直线。每条直线用两个数描述：

- **rho（ρ）**：直线到图像原点（左上角）的垂直距离，单位像素；
- **theta（θ）**：直线垂线的角度，单位度。0° 表示竖直线，90° 表示水平线。

`find_lines` 返回的是**无限延伸的直线方程**（画出来贯穿整幅图），这和实验4 的"线段检测"（有起点终点）不同。

> [!WARNING]
> 新手风险提醒：摄像头排线插反上电可能烧坏摄像头/核心板，务必断电插拔；LCD 排线同理；供电请使用 5V/2A 以上的稳定电源。

## 3. 相关前置知识

- **边缘**：亮度剧烈变化的位置，霍夫找线的第一步就是提取边缘点（find_lines 内部自动完成）。
- **霍夫变换投票**：把"很多点是否共线"的问题变成"参数空间里哪个 (ρ, θ) 得票最多"，抗噪声、能连接断开的线。
- **theta（角度）与 rho（距离）**：直线的极坐标表示。`l.theta()` 返回角度，`l.rho()` 返回距离，`l.line()` 返回可直接绘制的线段坐标 [x1,y1,x2,y2]。
- **threshold（累加器阈值）**：得票总数超过它才算一条直线。越大 → 检测越严格，误检越少，但弱直线也漏掉。
- **theta_margin / rho_margin（合并余量）**：角度差小于 theta_margin、距离差小于 rho_margin 的直线视为同一条，只保留最强的，避免一条边被重复检出多条。
- **帧率 FPS**：`clock.tick()/clock.fps()` 统计并打印。

## 4. 软件代码逻辑说明

### 第一步：角度过滤范围定义

```python
min_degree = 0
max_degree = 179
```

只保留角度在 0°~179° 的直线（霍夫角度的完整范围），你可以改成如 `min_degree = 80; max_degree = 100` 只找近似水平的线。

### 第二步：初始化摄像头与显示

```python
sensor = Sensor(width=640, height=480) # 构建摄像头对象
sensor.reset()
sensor.set_framesize(Sensor.VGA)    # 设置帧大小VGA(640x480)，默认通道0
sensor.set_pixformat(Sensor.RGB565) # 设置输出图像格式，默认通道0

Display.init(Display.ST7701, width=640, height=480, fps=90, to_ide=True)
MediaManager.init() # 初始化media资源管理器
sensor.run() # 启动sensor
clock = time.clock() # 构造clock对象
```

### 第三步：主循环——找线、过滤、绘制

```python
while True:
    os.exitpoint() # 检测IDE中断
    clock.tick()   # 记录开始时间（ms）
    img = sensor.snapshot() # 从通道0捕获一张图
    for l in img.find_lines(threshold=1000, theta_margin=25, rho_margin=25):
        if (min_degree <= l.theta()) and (l.theta() <= max_degree):
            img.draw_line([v for v in l.line()], color=(255, 0, 0))
            print(l)
    Display.show_image(img)
    print(clock.fps()) # 打印FPS
```

关键函数说明：

- `img.find_lines(threshold=1000, theta_margin=25, rho_margin=25)`：在整幅图上做霍夫直线检测，返回 line 对象列表。
  - `threshold=1000`：累加器得票阈值，VGA 大图常用 1000 左右；图像小（QVGA）时应相应调小，否则一条线都找不到。
  - `theta_margin=25`：角度相差 25° 以内的算同一条线；
  - `rho_margin=25`：距离相差 25 像素以内的算同一条线。
- `l.theta()`：直线角度，用于 if 过滤。
- `l.line()`：返回 (x1, y1, x2, y2) 直线端点坐标，`[v for v in l.line()]` 把元组转成列表传给 `draw_line`。
- `print(l)`：打印 line 对象信息（含 theta、rho、magnitude）。

### 第四步：异常处理与资源释放

`finally` 中 `sensor.stop()` → `Display.deinit()` → `MediaManager.deinit()`。

**可调参数**：
- `threshold=1000`（第 43 行）：找不到线时调小（如 500），误检太多时调大（如 2000）。
- `min_degree / max_degree`（第 24~25 行）：按需求筛选水平线（80~100）或竖直线（0~10 和 170~179）。
- `theta_margin / rho_margin`：画面中一条边被画出多条线时，加大这两个值合并重复线。

## 5. 运行效果

- **屏幕画面**：摄像头彩色画面上，检测到的直线以红色贯穿线显示（霍夫直线是无限长的，会延伸到图像边界）。对着有直边的场景（书本、桌沿、键盘、打印的黑十字线）效果最明显。
- **串口打印**：每检出一条线打印一次 line 对象参数，随后每帧打印一次帧率，如：
  ```text
  <image.line ...>
  45.2
  ```
- **成功现象**：红色线与物体真实直边贴合，移动物体线跟着动。
- **失败现象排查**：
  - 一条线都没有 → threshold 太大，或场景边缘太弱（光照不足、对比度低），调小 threshold、增强光线；
  - 满屏乱线 → threshold 太小或场景纹理太杂，调大 threshold、简化背景；
  - 帧率很低 → find_lines 在 VGA 上计算量大，可将 `set_framesize` 改为 `Sensor.QVGA`（threshold 同步减小）。

## 6. 代码使用说明

1. 用 **CanMV IDE** 打开本文件夹的 `main.py`（IDE 版本与板载 **CanMV K230D 固件**需匹配）。
2. USB 连接开发板 → 点击 IDE 连接按钮 → 点击运行。
3. 画面显示在 LCD 与 IDE 帧缓冲区，直线参数和帧率打印在串口终端。
4. 本例程**不需要额外文件**、不需要 SD 卡资源；脱机运行将 main.py 放入 TF 卡 `/sdcard/` 即可。

> [!NOTE]
> 建议测试场景：在白纸上用黑色粗笔画一条长直线或一个方框，放在摄像头正下方 20~30cm 处，光照均匀，检出效果最好。

## 7. 拓展思考

1. **只找水平线做"水平仪"**：把 `min_degree/max_degree` 改为 80/100，检测到水平线时点亮蓝色 LED（参考 GPIO 例程，GPIO59），做一个简易水平校准器。
2. **统计线条数量**：在循环里计数，若检出的直线多于 3 条则串口输出"复杂场景"，否则输出"简单场景"，练习用视觉做场景分类。
3. **输出直线角度给串口协议**：把 `l.theta()`、`l.rho()` 格式化成自定义协议（如 `LINE,45.2,120\n`）通过 UART 发送，为小车循线转向做准备。

## 8. 常见问题

**Q1：为什么 find_lines 画出的线贯穿整个屏幕，比实际物体的边长很多？**
A：这是霍夫直线的特性——它描述的是"无限长直线方程"，`l.line()` 返回的是该直线在图像内的截断端点。如果你想要"有头有尾"的真实线段，请用实验4 的 `find_line_segments`。

**Q2：threshold=1000 是怎么定的？我改成 QVGA 后一条线都找不到。**
A：threshold 与图像尺寸、边缘强度相关，像素越多、累加票数越多。分辨率减半后应把 threshold 也调小（例如 QVGA 用 200~500 起步试）。

**Q3：一条边被检测出好几条平行的红线怎么办？**
A：加大 `rho_margin`（如 25→40）让距离相近的线合并；同时适当加大 `threshold` 只保留最强的那条。

**Q4：`print(l)` 打出来的 theta 有时是 0 有时是 179，同一条竖线为什么两个值？**
A：霍夫参数空间中 θ 在 0° 和 180° 附近表示同方向直线，数值抖动属正常现象，筛选角度时注意同时覆盖两端（如竖直线筛 0~10 或 170~179）。

**Q5：帧率只有十几 FPS，正常吗？**
A：正常。VGA 分辨率下霍夫变换计算量大，对帧率敏感的应用建议用 QVGA 或改用 `find_line_segments`。
