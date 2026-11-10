# 实验10 LCD实验

> 在 2.4 寸 MIPI 屏幕上每秒刷新一帧随机位置、随机颜色、随机大小的 "Hello World"，点亮你的第一块屏。难度：初级

## 1. 例程简介

本例程初始化开发板配套的 **2.4 寸 MIPI LCD（驱动芯片 ST7701，分辨率 640×480）**，在内存中创建一张 640×480 的画布，每秒清空一次并随机画上 10 条不同位置、颜色、大小的 `"Hello World!，你好世界！！！"` 字符串，然后整幅刷新到屏幕（同时回传到 CanMV IDE 预览）。你将学会 CanMV 多媒体框架的三大件：**Image 画布、Display 显示、MediaManager 资源管理**，以及 `os.exitpoint()` 中断机制。难度为**初级**，是所有带屏显示例程的模板。

## 2. 硬件连接与原理

使用板载/配套资源：

| 外设 | 接口 | 代码中的体现 |
| --- | --- | --- |
| 2.4 寸 MIPI LCD（带电容触摸，本实验不用触摸） | MIPI DSI 接口 | `Display.init(Display.ST7701, width=640, height=480, to_ide=True)` |
| USB 串口 | 调试/预览回传 | `to_ide=True` 参数 |

出厂状态下屏幕已连接在底板的 MIPI 屏接口上，**无需用户接线**。若屏幕未显示，请先断电检查屏排线是否插到位。

### 硬件原理（通俗版）

LCD 屏幕本质上是一个 640×480 的"像素点阵"，每个像素可以显示任意颜色。屏幕本身不会"自己动"，需要主控不断地把**一整幅画面数据**通过 MIPI DSI 高速接口送过去，屏幕按数据点亮像素——每秒送多少幅画面就是**帧率**。

CanMV 的显示流程是"**先画后刷**"：

1. 在内存里建一张和屏幕一样大的"画布"（`image.Image` 对象）；
2. 用画布的各种 `draw_xxx` 方法作画；
3. 调用 `Display.show_image()` 把画布一次性"贴"到屏幕上。

`ST7701` 是这块屏的驱动芯片型号，`Display.init(Display.ST7701, ...)` 就是告诉固件"我这块屏用它，按 640×480 初始化"。

### 新手风险提醒

> [!WARNING]
> - MIPI 排线非常脆弱：**必须在断电状态下插拔排线**，带电插拔可能烧毁屏幕或主板接口。
> - 排线金手指要对准卡槽平插，锁扣轻压固定，不要硬掰。
> - 屏幕表面是玻璃，避免硬物划伤和挤压。

## 3. 相关前置知识

- **帧（Frame）**：一幅完整的画面。屏幕上看到的活动画面其实是每秒几十帧静态图快速连续播放的结果。
- **分辨率 640×480**：横向 640 个像素、纵向 480 个像素。画布坐标从左上角 `(0, 0)` 到右下角 `(639, 479)`。
- **RGB888 颜色格式**：每个像素用红、绿、蓝三个分量各 8 位（0~255）表示，可组合出约 1677 万种颜色。`(255, 0, 0)` 是纯红。
- **urandom 随机数**：`urandom.getrandbits(n)` 返回一个 n 位的随机整数，本例用它生成随机坐标、颜色和字号。
- **MediaManager（媒体资源管理器）**：CanMV 固件统一管理显示/摄像头等媒体内存缓冲区的模块，**用媒体功能前必须 `MediaManager.init()`，退出前 `MediaManager.deinit()`**。
- **os.exitpoint()**：CanMV 的"退出检查点"。脚本中周期性调用它，IDE 点击停止时才能及时、干净地中断程序（否则死循环程序可能不好停）。

## 4. 软件代码逻辑说明

流程：**导入模块 → 创建画布 → 初始化显示与媒体 → 主循环绘制刷新 → 异常处理 → 释放资源**。

### 第 1 步：导入模块

```python
import time, os, urandom, sys
from media.display import *
from media.media import *
```

`media.display` 提供 `Display` 类，`media.media` 提供 `MediaManager` 和 `image` 相关支持。

### 第 2 步：创建画布并初始化显示

```python
img = image.Image(640, 480, image.RGB888)
Display.init(Display.ST7701, width=640, height=480, to_ide=True)
MediaManager.init()
```

- `image.Image(640, 480, image.RGB888)`：在内存创建 640×480、RGB888 格式的画布对象。
- `Display.init(Display.ST7701, width=640, height=480, to_ide=True)`：以 ST7701 驱动初始化 LCD；`to_ide=True` 表示把显示内容同时回传给 CanMV IDE 的帧缓冲区预览窗口。
- `MediaManager.init()`：初始化媒体内存池，必须在 `Display.init` 之后、开始显示之前调用。

### 第 3 步：主循环——清屏、随机绘制、刷新

```python
while True:
    img.clear()
    for i in range(10):
        x = (urandom.getrandbits(11) % img.width())
        y = (urandom.getrandbits(11) % img.height())
        r = (urandom.getrandbits(8))
        g = (urandom.getrandbits(8))
        b = (urandom.getrandbits(8))
        size = (urandom.getrandbits(30) % 64) + 32
        # 在不同位置刷新不同大小、颜色的字符串
        img.draw_string_advanced(x, y, size, "Hello World!，你好世界！！！", color=(r, g, b),)

    # 刷新到显示器上
    Display.show_image(img)

    time.sleep(1)
    os.exitpoint()
```

逐行解释：

- `img.clear()`：清空画布（涂黑），否则上一帧的字会残留叠加。
- `for i in range(10)`：每帧画 10 条字符串。
- `x`、`y`：随机数对 `img.width()`（640）、`img.height()`（480）取余，得到画布范围内的随机坐标。
- `r/g/b`：各取 8 位随机数（0~255），组成随机颜色。
- `size`：32~95 之间的随机字号（像素高）。
- `img.draw_string_advanced(x, y, size, 文本, color=(r,g,b))`：在画布上绘制**支持中文的高级字符串**（普通 `draw_string` 不支持中文）。
- `Display.show_image(img)`：把整张画布刷新到 LCD（和 IDE 预览）。
- `time.sleep(1)`：每帧停留 1 秒，即约 1 帧/秒的"幻灯片"节奏。
- `os.exitpoint()`：检查 IDE 是否请求中断。

### 第 4 步：异常处理与资源释放

```python
except KeyboardInterrupt as e:
    print("user stop: ", e)
except BaseException as e:
    print(f"Exception {e}")

# deinit display
Display.deinit()
os.exitpoint(os.EXITPOINT_ENABLE_SLEEP)
time.sleep_ms(100)
# release media buffer
MediaManager.deinit()
```

- IDE 点停止会触发 `KeyboardInterrupt`，其他异常被 `BaseException` 捕获并打印。
- 退出时**按顺序释放**：`Display.deinit()` → 切换退出点模式并小睡 → `MediaManager.deinit()`。资源不释放会导致下次运行报"内存/buffer 不足"类错误。
- 入口处的 `os.exitpoint(os.EXITPOINT_ENABLE)`（在 `__main__` 里）开启中断检查点功能，与循环里的 `os.exitpoint()` 配对使用。

### 关键 API 小结

| API | 作用 |
| --- | --- |
| `image.Image(w, h, image.RGB888)` | 创建内存画布 |
| `Display.init(Display.ST7701, width, height, to_ide)` | 初始化 LCD 显示 |
| `MediaManager.init()/deinit()` | 申请/释放媒体内存资源 |
| `img.clear()` | 清空画布 |
| `img.draw_string_advanced(x, y, size, str, color)` | 绘制支持中文的字符串 |
| `Display.show_image(img)` | 把画布刷新到屏幕 |
| `os.exitpoint()` | IDE 中断检查点 |

### 可调参数

- `range(10)`：每帧字符串条数，改大画面更密。
- `size` 的 `% 64 + 32`：字号范围（32~95）。
- `time.sleep(1)`：刷新间隔，改小动画更快（如 0.1）。
- 文本 `"Hello World!，你好世界！！！"`：任意替换，支持中文。

## 5. 运行效果

- **成功现象**：
  1. LCD 屏幕每秒刷新一次，黑底上出现 10 条位置、颜色、大小都随机的 "Hello World!，你好世界！！！"；
  2. CanMV IDE 右侧帧缓冲区窗口同步显示相同画面（`to_ide=True`）；
  3. 点击 IDE 停止，终端打印 `user stop:` 且程序干净退出。
- **失败现象排查**：
  1. 屏幕黑屏但 IDE 有画面 → 检查屏排线是否插好（断电重插）、排线锁扣是否压紧。
  2. 报错 buffer/memory 相关异常 → 上次程序没有正常释放资源，复位开发板后重试。
  3. 中文显示为方块/乱码 → 确认使用的是 `draw_string_advanced`（需要固件内置字库支持）。
  4. IDE 预览不更新 → 确认 `to_ide=True`，并打开 IDE 的帧缓冲区窗口。

## 6. 代码使用说明

1. 确认 2.4 寸 MIPI 屏已正确连接到底板屏接口（出厂一般已装好）。
2. 用 **CanMV IDE** 打开本目录下的 `main.py`。
3. USB Type-C 连接开发板与电脑，点击 IDE 左下角**连接**，再点击**运行**。
4. 固件要求：**CanMV K230D 固件**，版本与 IDE 匹配（显示与中文字库功能依赖固件支持）。
5. **不需要 TF 卡、不需要模型文件**；画布和绘制全部在内存中完成。

> [!NOTE]
> `Display.init` 与 `MediaManager.init` 的先后顺序、退出时 `deinit` 的成对调用，是 CanMV 所有媒体类例程的固定套路，后续触摸屏、摄像头实验都会复用这套结构，务必掌握。

## 7. 拓展思考

1. **画其他图形**：把 `draw_string_advanced` 换成画布的其他绘图方法（如画圆、画矩形、画线），做一个随机图形动画；参考[实验11 触摸屏实验](../实验11%20触摸屏实验/README.md)中 `draw_circle` 的用法。
2. **滚动字幕**：固定 y 坐标、每帧让 x 减小 10（到负值后回到屏幕右侧），做一条从右向左滚动的字幕，体会"帧与动画"的关系。
3. **自定义时钟**：结合[实验7 RTC实验](../实验7%20RTC实验/README.md)，每秒把 RTC 时间格式化成字符串画在屏幕中央，做一个数字时钟。

## 8. 常见问题

**Q1：为什么屏幕是 640×480，而宣传说 2.4 寸？会不会设置错分辨率？**
A：不会。本开发板配套的 2.4 寸 MIPI LCD 原生分辨率就是 640×480（像素密度很高的小屏），`Display.init` 的 `width=640, height=480` 与画布 `image.Image(640, 480, ...)` 保持一致即可。

**Q2：`draw_string_advanced` 和普通 `draw_string` 有什么区别？**
A：`draw_string_advanced` 使用矢量/点阵字库渲染，**支持中文**、支持任意字号和更平滑的效果；普通 `draw_string` 只能画内置的英文小字符。想显示中文必须用 advanced 版本。

**Q3：程序停止后再运行报内存或 buffer 错误？**
A：多半是上一次异常退出没有走到 `MediaManager.deinit()`。按开发板**复位键**（或重新上电）后再运行即可。这也是例程把释放代码放在循环之外、并用异常捕获包住主循环的原因。

**Q4：`to_ide=True` 有什么用？可以关掉吗？**
A：它让显示内容同时通过 USB 回传到 IDE 的帧缓冲区窗口，方便没接屏幕或在电脑上观察时调试。可以改为 `False` 关闭回传、减轻 USB 带宽占用（如[实验15 拍照实验](../实验15%20拍照实验/README.md)中就是 `to_ide=False`）。

**Q5：为什么画面是"每秒一跳"，不流畅？**
A：代码故意用 `time.sleep(1)` 让每帧停留 1 秒，方便观察随机效果。想要流畅动画，把延时改小（如 `time.sleep_ms(50)`，约 20 帧/秒），或去掉 `img.clear()` 做累积轨迹效果。
