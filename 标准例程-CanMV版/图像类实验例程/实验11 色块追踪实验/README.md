# 实验11 色块追踪实验

> 按下 KEY0 让摄像头"学习"取景框内物体的颜色，之后自动追踪画面中同色物体并框出质心。难度：**中级**

## 1. 例程简介

本例程实现一个交互式色块追踪器：运行后按下开发板 **KEY0**，屏幕中央出现一个 50x50 的学习取景框，把要追踪的物体放进框内，程序用**直方图百分位法自动学习**该物体的 LAB 颜色阈值，之后持续用 `find_blobs` 追踪同色物体，画出外框和质心十字。学习知识点是按键 GPIO 输入、get_histogram 直方图、百分位取阈值、merge/margin 参数、状态机式的分阶段逻辑。难度为**中级**，预期用途是"追踪任意指定颜色的物体"，如追踪小球、追随彩色标志的云台。

## 2. 硬件连接与原理

用到的外设（以 main.py 实际代码为准）：

| 外设 | 代码依据 | 说明 |
| --- | --- | --- |
| MIPI CSI 摄像头 | `Sensor(width=1280, height=960)`，`set_framesize(Sensor.VGA)`，`set_pixformat(Sensor.RGB565)` | 输出 VGA（640x480）RGB565 彩色图像 |
| 2.4 寸 MIPI LCD | `Display.init(Display.ST7701, width=640, height=480, fps=90, to_ide=True)` | ST7701 驱动，同步输出 IDE |
| 按键 KEY0 | `fpioa.set_function(34, FPIOA.GPIO34)`，`key0 = Pin(34, Pin.IN, pull=Pin.PULL_UP, drive=7)` | 通过 FPIOA 把芯片引脚 34 映射为 GPIO34，配置为**上拉输入**；按下时 `key0.value()` 读到 0 |

原理通俗讲解：**手工调 LAB 阈值很麻烦，本例让程序自己学**。步骤是：

1. 按下 KEY0 进入"学习模式"，在画面中央画出 50x50 像素的取景框；
2. 把目标物体放进框内，程序对框内区域（roi=r）做 `get_histogram()` 统计 LAB 直方图；
3. 取直方图的 1% 和 99% **百分位**作为每个通道的低/高边界——即"去掉最暗最亮的 1% 极端像素，剩下的范围就是物体颜色范围"；
4. 连续多帧学习并取平均，得到稳定的阈值；
5. 之后每帧用学到的阈值 find_blobs，实现追踪。

> [!WARNING]
> 新手风险提醒：摄像头/LCD 排线必须**断电插拔**，插反可能烧毁器件；使用 5V/2A 以上稳定电源。KEY0 是板载按键，不要外接电源到引脚 34。

> [!NOTE]
> 关于代码中的一处细节：源程序分支条件写的是 `frame_count < 60 & threshold_flag == 1`，使用了按位与 `&`。由于 Python 中比较运算优先级低于 `&`，该表达式实际按 `frame_count < (60 & threshold_flag) == 1` 求值，学习分支可能永远不成立。学习本例时建议把两处分枝条件改成逻辑与：`(frame_count < 60) and (threshold_flag == 1)`，即可得到下文描述的正常流程。

## 3. 相关前置知识

- **FPIOA 与 GPIO**：K230D 的引脚功能可重映射，`fpioa.set_function(34, FPIOA.GPIO34)` 把物理引脚 34 设为普通 GPIO；`Pin(34, Pin.IN, pull=Pin.PULL_UP)` 配置为输入+内部上拉，按键未按下读到 1，按下接地读到 0。
- **LAB 颜色空间与阈值**：L 亮度、A 红绿轴、B 黄蓝轴；阈值是六个数的区间（见实验8）。
- **ROI（感兴趣区域）**：`r = [(640//2)-(50//2), (480//2)-(50//2), 50, 50]`——在 640x480 画面正中央画一个 50x50 的方框，学习和追踪都围绕它进行（学习阶段只对框内取直方图）。
- **直方图 get_histogram(roi=r)**：统计区域内 LAB 各通道像素值的分布。
- **百分位 get_percentile(0.01/0.99)**：返回累计分布 1%/99% 处的值，即去掉两端极端的 1% 像素后的颜色范围，比"最小/最大值"抗噪。
- **merge=True 与 margin=10**：find_blobs 把相距 10 像素以内的色块**合并**成一个大色块，物体上有小阴影/高光缝隙时仍能框成整体。
- **pixels_threshold=100 / area_threshold=100**：命中像素数和外接框面积的最低门槛，过滤小噪点。
- **帧计数状态机**：frame_count 0~59 显示取景框、60~119 学习阈值、之后进入追踪，用计数值划分阶段。

## 4. 软件代码逻辑说明

### 第一步：按键初始化

```python
from machine import Pin
from machine import FPIOA

fpioa = FPIOA()                      # 实例化FPIOA
fpioa.set_function(34, FPIOA.GPIO34) # 为IO分配相应的硬件功能
key0 = Pin(34, Pin.IN, pull=Pin.PULL_UP, drive=7) # 构造GPIO对象
```

### 第二步：初始化摄像头、显示与状态变量

```python
sensor = Sensor(width=1280, height=960) # 构建摄像头对象
sensor.reset()
sensor.set_framesize(Sensor.VGA)    # 设置帧大小VGA(640x480)，默认通道0
sensor.set_pixformat(Sensor.RGB565) # 设置输出图像格式，默认通道0

Display.init(Display.ST7701, width=640, height=480, fps=90, to_ide=True)
MediaManager.init() # 初始化media资源管理器
sensor.run() # 启动sensor
clock = time.clock() # 构造clock对象

frame_count = 0
threshold_flag = 0  # 捕获颜色LAB值的标记
threshold = [50, 50, 0, 0, 0, 0] # 中间的 L, A, B 值.
r = [(640//2) - (50//2), (480//2) - (50//2), 50, 50] # 50x50 center of QVGA.
```

`r` 是画面中央 50x50 的取景框坐标（注释里的 QVGA 是模板遗留说法，实际按 640x480 计算）。

### 第三步：主循环三阶段逻辑

```python
while True:
    os.exitpoint()
    clock.tick()
    img = sensor.snapshot()

    if key0.value() == 0:      # 按下KEY0，启动学习流程
        frame_count = 0
        threshold_flag = 1
    if frame_count < 60 & threshold_flag == 1:   # 阶段1：显示取景框（0~59帧）
        if frame_count == 0:
            print("Letting auto algorithms run. Don't put anything in front of the camera!")
            ...
        img.draw_rectangle([v for v in r])
        frame_count = frame_count + 1
    elif (frame_count < 120) & threshold_flag == 1:  # 阶段2：学习阈值（60~119帧）
        if frame_count == 60:
            print("Learning thresholds...")
        elif frame_count == 119:
            print("Thresholds learned...")
            threshold_flag = 0 # 解除标记
        hist = img.get_histogram(roi=r)
        lo = hist.get_percentile(0.01) # 取直方图1%分位
        hi = hist.get_percentile(0.99) # 取直方图99%分位
        threshold[0] = (threshold[0] + lo.l_value()) // 2
        ...
        for blob in img.find_blobs([threshold], pixels_threshold=100, area_threshold=100, merge=True, margin=10):
            img.draw_rectangle([v for v in blob.rect()])
            img.draw_cross(blob.cx(), blob.cy())
            img.draw_rectangle([v for v in r])
        frame_count = frame_count + 1
        del hist
    else:                       # 阶段3：正常追踪
        for blob in img.find_blobs([threshold], pixels_threshold=100, area_threshold=100, merge=True, margin=10):
            img.draw_rectangle([v for v in blob.rect()])
            img.draw_cross(blob.cx(), blob.cy())
    Display.show_image(img)
    print(clock.fps())
```

关键函数说明：

- `key0.value() == 0`：按键按下（上拉输入，按下接地为 0），复位帧计数并置学习标记。
- `img.get_histogram(roi=r)`：只统计取景框内像素的 LAB 直方图。
- `hist.get_percentile(0.01)` / `(0.99)`：取 1%/99% 百分位对象，`lo.l_value()/a_value()/b_value()` 读出各通道低端值，hi 读高端值。
- `threshold[k] = (threshold[k] + 新值) // 2`：**滑动平均**，每帧把新学到的边界与旧值平均，60 帧下来阈值收敛且平滑抗抖。
- `img.find_blobs([threshold], pixels_threshold=100, area_threshold=100, merge=True, margin=10)`：用学到的阈值找色块；`merge=True, margin=10` 让相距 10 像素内的碎块合并成整体。
- `blob.rect()` / `blob.cx()` / `blob.cy()`：外接矩形与质心，分别画框、画十字。
- `del hist`：手动释放直方图对象，节省内存。

### 第四步：异常处理与资源释放

`finally` 中依次 `sensor.stop()` → `Display.deinit()` → `MediaManager.deinit()`。

**可调参数**：
- 百分位 0.01/0.99（第 76~77 行）：物体颜色不均匀时可放宽到 0.005/0.995；背景干扰多时收紧到 0.05/0.95。
- 学习帧数 60/120：加长学习窗口（如 60→100、120→200）阈值更稳。
- `margin=10`、两个 threshold=100：按目标大小调整。

## 5. 运行效果

- **屏幕画面**：启动后是普通摄像头画面；按下 KEY0 后画面中央出现白色 50x50 取景框（阶段1），把目标物体放入框内，约 2 秒学习期（阶段2，框内同时实时试追踪），随后进入追踪模式（阶段3）：物体被白色矩形框套住、中心画十字，物体移动框跟随；画面中其他同色物体也会被框。
- **串口打印**：学习流程各阶段的英文提示（"Learning thresholds..." / "Thresholds learned..." / "Tracking colors..."）以及每帧帧率。
- **成功现象**：学习一次后，拿着该物体在镜头前移动，白框稳定跟随。
- **失败现象排查**：
  - 按 KEY0 没反应 → 检查是否按的是 KEY0（GPIO34）；确认代码中 `&` 已按上文 NOTE 改成 `and`，否则学习分支不会触发；
  - 学习后什么都追不到 → 学习时物体没有完全放进 50x50 框内，或框内混入大量背景，重学；
  - 追到一堆杂物 → 场景里有同色物体，或百分位范围太宽，收紧百分位重学。

## 6. 代码使用说明

1. 用 **CanMV IDE** 打开本文件夹 `main.py`（IDE 与板载 **CanMV K230D 固件**版本需匹配）。
2. USB 连接开发板，点击 IDE 连接按钮，再点击运行。
3. 操作顺序：运行 → 按 KEY0 → 等取景框出现 → 把目标物体放入框中并保持 → 等串口打印 "Tracking colors..." → 移动物体观察追踪。
4. 本例程**不需要额外文件**、无 SD 卡依赖；脱机自启动将 main.py 放入 TF 卡 `/sdcard/`（按键学习功能在脱机模式下同样可用）。

> [!NOTE]
> 学习的阈值保存在内存变量里，**断电即丢失**，每次上电都要重新按 KEY0 学习。想固定追踪某颜色，可把学到的 threshold 打印出来抄进代码，改用实验8 的固定阈值写法。

## 7. 拓展思考

1. **打印学到的阈值**：在 "Thresholds learned..." 后加 `print(threshold)`，把自动学习结果记录下来，固化成常量阈值，理解"学习"与"固定阈值"两种工程做法。
2. **云台追踪**：用 `blob.cx()-320`、`blob.cy()-240` 作为偏差量，通过 UART 或 PWM 控制两轴云台（参考正点原子舵机例程）让摄像头始终对准目标。
3. **只追最大的色块**：追踪阶段对 blobs 按 `b.pixels()` 取最大值（参考实验12 的 `max(blobs, key=lambda b: b.pixels())` 写法），忽略同色小干扰物。

## 8. 常见问题

**Q1：按下 KEY0 后没有出现取景框/没有学习提示？**
A：注意源码分支条件用的是按位 `&`（`frame_count < 60 & threshold_flag == 1`），由于运算优先级问题条件恒为假。把两处改成 `(frame_count < 60) and (threshold_flag == 1)` 和 `(frame_count < 120) and (threshold_flag == 1)` 即可恢复正常流程。

**Q2：学习时提示 "Don't put anything in front of the camera!" 是什么意思？**
A：阶段1（0~59 帧）本意是先让自动曝光等算法收敛，此时**不要**把物体挡在镜头前；到阶段2 再把物体放进取景框。实际操作中在取景框出现后就放入物体即可。

**Q3：为什么追踪到的是所有同颜色物体，而不是只追我学习的那一个？**
A：程序学的是"颜色"不是"物体"，find_blobs 会框出画面中所有匹配该颜色的色块。想只追一个，可取面积最大的 blob，或结合上一帧位置做最近邻匹配（进阶）。

**Q4：物体有高光/阴影时框会碎成好几块？**
A：本例已经用 `merge=True, margin=10` 合并相距 10 像素内的碎块；如果还碎，可加大 margin（如 20），或放宽百分位让阈值覆盖高光阴影部分。

**Q5：学习完过一会儿就追不准了？**
A：光照在变化而阈值是学习瞬间的。可定期重新学习（再按 KEY0），或把百分位范围放宽一点增强鲁棒性；工业做法是固定光源。
