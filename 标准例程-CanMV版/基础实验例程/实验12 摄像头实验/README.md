# 实验12 摄像头实验

> 打开 MIPI 摄像头，把实时画面直接送到 LCD 和 IDE 预览，体验"硬件直显"的零拷贝显示链路。难度：初级

## 1. 例程简介

本例程初始化开发板的 **MIPI CSI 摄像头**（1280×960 采集），把通道 0 的输出设为 VGA（640×480）YUV420SP 格式，并通过 `Display.bind_layer` 将摄像头数据流**直接绑定到 LCD 的视频图层**实时显示，同时回传 CanMV IDE 预览。整个主循环只做一件事：`os.exitpoint()` 等待中断。你将学会 **Sensor 摄像头对象、通道（chn）概念、bind 直显机制、MediaManager 资源管理**。难度为**初级**，它是后续拍照、AI 视觉等所有摄像头应用的起点。

## 2. 硬件连接与原理

| 外设 | 接口 | 代码中的体现 |
| --- | --- | --- |
| MIPI CSI 摄像头（板配两路 CSI 接口，默认使用接好的那一路） | MIPI CSI | `sensor = Sensor(width=1280, height=960)` |
| 2.4 寸 MIPI LCD（ST7701） | MIPI DSI | `Display.init(Display.ST7701, ...)` |
| USB 串口 | IDE 预览回传 | `to_ide=True` |

摄像头出厂已连接在底板 CSI 接口上，**无需用户接线**；如需更换摄像头或重插排线，务必断电操作。

### 硬件原理（通俗版）

**摄像头（Sensor）** 通过镜头把光线聚焦到图像传感器芯片上，传感器按"帧"输出数字图像。每帧就是一张完整照片，每秒输出的帧数就是帧率。

CanMV K230D 的摄像头有一个重要特性——**多通道（channel）输出**：同一路摄像头数据可以在芯片内部同时输出多种规格。本例只用了**通道 0**（VGA、YUV420SP）。

更关键的是 **bind（绑定）直显机制**：普通做法是"CPU 取一帧 → 处理 → 画到屏幕"，费 CPU 又慢；而 `Display.bind_layer` 让摄像头数据流在**硬件层面直通显示图层**，CPU 几乎不参与，所以主循环空着也能满帧流畅显示。这就像给摄像头和屏幕之间架了一条"专用管道"。

- **YUV420SP**：一种图像颜色格式，亮度（Y）全分辨率存储、色度（UV）减半存储，节省带宽，是视频链路的常用格式。
- **VGA**：640×480 分辨率的代号，正好匹配 LCD 的分辨率。

### 新手风险提醒

> [!WARNING]
> - MIPI CSI 排线精密脆弱，**必须断电插拔**，金手指对准卡槽平插，锁扣轻压。
> - 摄像头镜头避免指纹和灰尘，清洁时用镜头布，不要用手摸。
> - 排线接反或没插到位会导致 `Sensor` 初始化报错，这是新手最常见的问题。

## 3. 相关前置知识

- **Sensor / 图像传感器**：把光信号转成数字图像的芯片，代码中 `Sensor` 类即其软件封装。
- **帧大小（framesize）**：输出图像的分辨率。`Sensor.VGA` 是固件预定义的 640×480 常量，`Sensor.SXGAM` 是 1280×960。
- **像素格式（pixformat）**：每个像素的编码方式，本例 `Sensor.YUV420SP`；拍照实验中还会见到 `Sensor.RGB565`。
- **通道 chn**：摄像头输出的"分路"，通道 0 常用于预览/显示，通道 1 可配不同规格用于抓拍或 AI（见[实验15 拍照实验](../实验15%20拍照实验/README.md)）。
- **图层（Layer）**：显示输出可以叠多个图层（视频层、OSD 层等），`Display.LAYER_VIDEO1` 是视频图层 1，摄像头画面绑定到它。
- **MediaManager**：统一管理摄像头、显示等媒体内存的模块，先 `init()` 后使用，退出前 `deinit()`。
- **os.exitpoint()**：IDE 中断检查点；本例主循环只有它，因此停止脚本时能立刻干净退出。

## 4. 软件代码逻辑说明

流程：**构建 Sensor → 配置通道 0 → 绑定显示图层 → 初始化显示与媒体 → 启动采集 → 空循环等待中断 → finally 释放资源**。

### 第 1 步：构建并复位摄像头

```python
sensor = Sensor(width=1280, height = 960) #构建摄像头对象
sensor.reset() #复位和初始化摄像头
```

以 1280×960 规格构建 Sensor 对象并复位。这是摄像头传感器本身的采集规格，后面各通道输出的画面都从它"分流"而来。

### 第 2 步：配置通道 0 输出

```python
sensor.set_framesize(Sensor.VGA)      #设置帧大小VGA(640x480)，默认通道0
sensor.set_pixformat(Sensor.YUV420SP) #设置输出图像格式，默认通道0
```

不写 `chn` 参数时默认配置**通道 0**：分辨率 VGA（640×480）、格式 YUV420SP，用于显示预览。

### 第 3 步：绑定到显示视频图层

```python
bind_info = sensor.bind_info()
Display.bind_layer(**bind_info, layer=Display.LAYER_VIDEO1)
```

- `sensor.bind_info()`：取出通道 0 的数据流描述信息（字典）。
- `Display.bind_layer(**bind_info, layer=...)`：`**` 把字典展开成关键字参数，将摄像头数据流绑定到显示器的**视频图层 1**。绑定之后摄像头→屏幕的通路完全由硬件维护。

### 第 4 步：初始化显示、媒体，启动采集

```python
Display.init(Display.ST7701, sensor.width(), sensor.height(), fps=90, to_ide=True)
MediaManager.init() #初始化media资源管理器
sensor.run() #启动sensor
```

- `Display.init`：初始化 ST7701 LCD，显示窗口尺寸取自 `sensor.width()/sensor.height()`（即 1280×960，超出 LCD 物理分辨率的部分由显示子系统缩放/裁剪处理），`fps=90` 设置显示刷新率上限，`to_ide=True` 同步回传 IDE 预览。
- `MediaManager.init()`：分配媒体缓冲区，**必须在 `sensor.run()` 之前**。
- `sensor.run()`：启动摄像头连续采集，画面开始流动。

### 第 5 步：主循环

```python
while True:
    os.exitpoint() #检测IDE中断
```

因为显示是硬件直通的，CPU 无事可做，循环里只保留 IDE 中断检查。**注意顺序：绑定 → Display.init → MediaManager.init → run，缺一步或顺序错都会黑屏/报错。**

### 第 6 步：异常处理与资源释放

```python
except KeyboardInterrupt as e:
    print("user stop: ", e)
except BaseException as e:
    print(f"Exception {e}")
finally:
    if isinstance(sensor, Sensor):
        sensor.stop()
    Display.deinit()
    os.exitpoint(os.EXITPOINT_ENABLE_SLEEP)
    time.sleep_ms(100)
    MediaManager.deinit()
```

`finally` 保证无论正常停止还是异常都会执行：停摄像头（`isinstance` 先确认 sensor 构建成功，避免构建失败时再报错）→ 关显示 → 释放媒体资源。**不释放会导致下次运行报 buffer 错误，需要复位开发板。**

### 关键 API 小结

| API | 作用 |
| --- | --- |
| `Sensor(width, height)` | 构建摄像头对象，指定采集规格 |
| `sensor.reset()` | 复位并初始化摄像头 |
| `sensor.set_framesize(size, chn=0)` | 设置某通道输出分辨率 |
| `sensor.set_pixformat(fmt, chn=0)` | 设置某通道像素格式 |
| `sensor.bind_info()` | 获取通道数据流绑定信息 |
| `Display.bind_layer(**info, layer)` | 摄像头数据流绑定到显示图层（硬件直显） |
| `sensor.run() / sensor.stop()` | 启动/停止采集 |

### 可调参数

- `Sensor(width=1280, height=960)`：采集规格，与所用摄像头的能力相关，一般保持默认。
- `set_framesize(Sensor.VGA)`：可换其他分辨率常量，但应与 LCD 显示能力匹配，改大未必更清晰、反而增加带宽。
- `fps=90`：显示刷新率上限参数，可按需调低。
- `to_ide=True`：不需要 IDE 预览时改为 `False` 可省 USB 带宽。

## 5. 运行效果

- **成功现象**：
  1. LCD 屏幕上出现摄像头的**实时画面**，镜头对着哪里屏幕就显示哪里，移动物体画面流畅跟随；
  2. CanMV IDE 帧缓冲区窗口同步显示画面；
  3. 主循环不占 CPU，点击 IDE 停止后终端打印 `user stop:` 并干净退出。
- **失败现象排查**：
  1. 报错 Sensor 初始化失败 → 摄像头排线没插好或接反，**断电**重新插拔；确认使用的 CSI 接口与固件默认通道一致。
  2. 屏幕黑屏但 IDE 有画面 → LCD 排线问题，断电重插。
  3. 报 buffer/memory 错误 → 上次未正常释放资源，复位开发板重试。
  4. 画面很暗/偏色 → 环境光线问题或镜头脏污，多数摄像头会自动曝光调节，稍等片刻。

## 6. 代码使用说明

1. 确认 MIPI 摄像头已连接到底板 CSI 接口、LCD 已连接（出厂一般已装好）。
2. 用 **CanMV IDE** 打开本目录下的 `main.py`，USB Type-C 连接开发板，点击**连接**、**运行**。
3. 固件要求：**CanMV K230D 固件**（含 sensor/display 媒体驱动），版本与 IDE 匹配。
4. **不需要 TF 卡、不需要模型文件**——本实验只取景显示，不做任何 AI 推理和存储。

> [!NOTE]
> `from media.sensor import *`、`from media.display import *`、`from media.media import *` 三行导入是 CanMV 媒体例程的标配；`Sensor`、`Display`、`MediaManager`、`image` 等都来自这些模块，均属于 CanMV 官方固件原生 API，无需安装第三方库。

## 7. 拓展思考

1. **抓拍一帧**：在主循环里加入按键检测（参考[实验3 按键输入实验](../实验3%20按键输入实验/README.md)），按下 KEY0 时用 `sensor.snapshot()` 抓一帧图像对象，为[实验15 拍照实验](../实验15%20拍照实验/README.md)做铺垫。
2. **画面上叠加 OSD**：另建一张 `image.Image` 画布，用 `draw_string_advanced` 写上时间或提示文字，叠加显示到 OSD 图层，体会"视频层 + 叠加层"的分层显示。
3. **双通道输出**：参照拍照实验，给通道 1 配置不同分辨率/格式（`chn=CAM_CHN_ID_1`），通道 0 继续预览、通道 1 用于抓高清图，理解多通道的用途。

## 8. 常见问题

**Q1：为什么主循环里只有一句 `os.exitpoint()`，画面却一直在动？**
A：因为摄像头到屏幕的数据流通过 `Display.bind_layer` 绑定了，由**硬件自动持续搬运**，不需要 CPU 每帧参与。这正是 bind 直显比"软件逐帧搬运"高效的原因。`os.exitpoint()` 只负责响应 IDE 的停止请求。

**Q2：`Sensor(width=1280, height=960)` 和 `set_framesize(Sensor.VGA)` 不是矛盾吗？**
A：不矛盾，是两层概念。前者是**传感器采集规格**（从摄像头"源头"取多大的画面）；后者是**通道输出规格**（这一路输出被缩放/裁剪成多大）。源头 1280×960，通道 0 输出 640×480，芯片内部自动完成缩放。

**Q3：初始化顺序有什么讲究？**
A：本例的顺序是：构建 Sensor 并配置通道 → `bind_info` 绑定图层 → `Display.init` → `MediaManager.init` → `sensor.run`。绑定必须在 `MediaManager.init` 之前完成，`run` 必须最后；顺序错了会黑屏或抛异常。照抄例程顺序最稳妥。

**Q4：`finally` 里的 `isinstance(sensor, Sensor)` 是干什么的？**
A：防御性写法。如果 `Sensor(...)` 构建本身就失败了，`sensor` 变量可能不是有效的 Sensor 对象，直接调用 `sensor.stop()` 会引发第二个异常。`isinstance` 先确认类型再停止，让清理过程更健壮。

**Q5：板上有两路 CSI 接口，代码里怎么选第二路摄像头？**
A：`Sensor()` 默认使用固件配置的第一路摄像头。要使用另一路需要按固件文档指定传感器/接口参数，且两路同时使用时要注意内存和带宽分配。本实验先用默认单路即可。
