# 实验15 AprilTag码识别实验

> 识别专为机器人设计的 AprilTag 视觉基准标签，输出标签家族、ID 和旋转角。难度：**中级**

## 1. 例程简介

本例程调用 `img.find_apriltags()` 同时支持 6 个 AprilTag 标签家族（TAG16H5、TAG25H7、TAG25H9、TAG36H10、TAG36H11、ARTOOLKIT），检出标签后画红框和绿色质心十字，串口打印标签家族名、ID 编号和旋转角度。学习知识点是 AprilTag 的设计思想、标签家族与 H 值的含义、位掩码组合 families 参数、弧度转角度。难度为**中级**，预期用途是机器人定位、无人机降落引导、AGV 站点识别、多目标编号区分——AprilTag 是机器人领域最常用的视觉基准码。

## 2. 硬件连接与原理

用到的外设（以 main.py 实际代码为准）：

| 外设 | 代码依据 | 说明 |
| --- | --- | --- |
| MIPI CSI 摄像头 | `Sensor(width=1280, height=960)`，`set_framesize(Sensor.QVGA)`，`set_pixformat(Sensor.RGB565)` | 输出 **QVGA（320x240）** RGB565 图像（AprilTag 检测在黑白二值层面进行，QVGA 速度足够） |
| 2.4 寸 MIPI LCD | `Display.init(Display.ST7701, width=640, height=480, fps=90, to_ide=True)` | ST7701 驱动，小图居中显示，同步输出 IDE |

原理通俗讲解：AprilTag 是一种黑白方形标签：外圈黑色方框便于定位，内部 NxN 的黑白格子编码一个 ID。与二维码不同，它**不追求存很多信息，而追求"看得远、认得准、算得快"**——即使模糊、倾斜、光照差，也能可靠解出小小的 ID 号，非常适合做机器人的"路标"。每个"家族"是一套编码方案：TAG16H5 内部是 4x4 格子（16 bit），TAG36H11 是 6x6 格子（36 bit）。格子越少单个码越大、看得越远，但编码空间小、**误识别（假阳性）率高**；H 值是纠错汉明距离，越大越不容易认错。

> [!WARNING]
> 新手风险提醒：摄像头/LCD 排线必须**断电插拔**，插反可能烧毁器件；请使用 5V/2A 以上稳定电源。

## 3. 相关前置知识

- **AprilTag**：密歇根大学 APRIL 实验室设计的视觉基准标签，机器人定位标配；黑白打印即可使用。
- **标签家族（family）**：不同的编码规格，本例支持 6 种（见代码注释）：TAG16H5 是 4x4 码、TAG36H11 是 6x6 码（**默认推荐**）、ARTOOLKIT 兼容另一套 AR 标签体系。
- **H 值（汉明距离）**：家族名里的 H5/H7/H9/H11 表示合法编码间的最小差异位数，H 越大，噪声导致误认成另一个 ID 的概率越低。16H5 假阳性率明显高于 36H11（代码注释有说明）。
- **位掩码 `|=`**：`tag_families |= image.TAG36H11` 用按位或把多个家族常量组合成一个整数，一次检测可同时启用最多 6 个家族；不想检测某家族就注释掉对应行。
- **tag.id()**：标签在家族内的编号（如 TAG36H11 家族有 0~586 号），多个标签靠 ID 区分身份。
- **tag.rotation()**：标签平面内旋转角（弧度），`(180 * tag.rotation()) / math.pi` 转成角度。
- **rect() / cx() / cy()**：外接矩形和质心坐标。
- **帧率 FPS**：clock 对象统计。

## 4. 软件代码逻辑说明

### 第一步：组合要检测的标签家族

```python
# AprilTag代码支持多达6个标签族，可以同时处理。
tag_families = 0
tag_families |= image.TAG16H5 # 注释掉，禁用此家族
tag_families |= image.TAG25H7 # 注释掉，禁用此家族
tag_families |= image.TAG25H9 # 注释掉，禁用此家族
tag_families |= image.TAG36H10 # 注释掉，禁用此家族
tag_families |= image.TAG36H11 # 注释掉，禁用此家族 (默认家族)
tag_families |= image.ARTOOLKIT # 注释掉，禁用此家族
```

6 个家族全部启用。实际项目建议**只留你打印的家族**（一般 TAG36H11），检测更快、假阳性更少。

### 第二步：家族名翻译函数与初始化

```python
def family_name(tag):
    if(tag.family() == image.TAG16H5):
        return "TAG16H5"
    ...
    if(tag.family() == image.ARTOOLKIT):
        return "ARTOOLKIT"

sensor = Sensor(width=1280, height=960) # 构建摄像头对象
sensor.reset() # 复位和初始化摄像头
sensor.set_framesize(Sensor.QVGA)   # 设置帧大小QVGA(320x240)，默认通道0
sensor.set_pixformat(Sensor.RGB565) # 设置输出图像格式，默认通道0

Display.init(Display.ST7701, width=640, height=480, fps=90, to_ide=True)
MediaManager.init() # 初始化media资源管理器
sensor.run() # 启动sensor
clock = time.clock() # 构造clock对象
```

### 第三步：主循环——找标签、画标记、打印信息

```python
while True:
    os.exitpoint() # 检测IDE中断
    clock.tick()   # 记录开始时间（ms）
    img = sensor.snapshot() # 从通道0捕获一张图

    for tag in img.find_apriltags(families=tag_families):
        img.draw_rectangle([v for v in tag.rect()], color=(255, 0, 0), thickness=4)
        img.draw_cross(tag.cx(), tag.cy(), color=(0, 255, 0), thickness=2)
        # 打印AprilTag码信息
        print_args = (family_name(tag), tag.id(), (180 * tag.rotation()) / math.pi)
        print("Tag Family %s, Tag ID %d, rotation %f (degrees)" % print_args)

    # 显示图片
    Display.show_image(img, x=round((640-sensor.width())/2), y=round((480-sensor.height())/2))
    print(clock.fps()) # 打印FPS
```

关键函数说明：

- `img.find_apriltags(families=tag_families)`：按启用的家族搜索标签，返回 tag 对象列表；还可传 `roi=(x,y,w,h)` 限定区域、`margin` 等参数（本例未用）。
- `tag.rect()`：外接矩形 → 红框（thickness=4）。
- `tag.cx() / tag.cy()`：质心 → 绿色十字（thickness=2）。
- `tag.family() / tag.id() / tag.rotation()`：家族、编号、旋转弧度；rotation 乘 `180/math.pi` 转角度。
- tag 对象还有 `tag.x_translation()/y_translation()/z_translation()` 和 `tag.rx()/ry()/rz()` 等**位姿估计**方法（需要给 find_apriltags 传入相机焦距等标定参数才有物理意义，本例未使用，属进阶方向）。
- `Display.show_image(img, x, y)`：QVGA 小图在 640x480 屏幕上居中显示。

### 第四步：异常处理与资源释放

`finally` 中依次 `sensor.stop()` → `Display.deinit()` → `MediaManager.deinit()`。

**可调参数**：
- `tag_families`（第 26~32 行）：只保留实际使用的家族可提速并减少误检。
- `set_framesize`：QVGA 兼顾速度与识别距离；标签很小/很远时可改 VGA（帧率下降）。
- roi：固定检测区域时添加可提速。

## 5. 运行效果

- **屏幕画面**：屏幕中央 320x240 画面中，每个 AprilTag 被红色外框套住、中心有绿色十字；倾斜、旋转标签仍能识别。
- **串口打印**：每识别到一个标签打印一行，如：
  ```text
  Tag Family TAG36H11, Tag ID 3, rotation 12.5 (degrees)
  ```
  另外每帧打印一次帧率。
- **成功现象**：网上下载 TAG36H11 家族图案打印（黑白激光打印，边长 5cm 以上），放镜头前 20~60cm 稳定识别，ID 号与图案编号一致。
- **失败现象排查**：
  - 识别不到 → 打印件太小/太模糊、距离太远（QVGA 下模块像素不足）、家族不匹配（打印的是 36H11 却只启用了其他家族）；
  - 没放标签也偶尔报出 ID → 假阳性，减少启用的家族数（只留 TAG36H11）；
  - 屏显画面小 → QVGA 居中显示属正常，四周是未填充区域。

## 6. 代码使用说明

1. 用 **CanMV IDE** 打开本文件夹 `main.py`（IDE 与板载 **CanMV K230D 固件**版本需匹配）。
2. USB 连接开发板，点击 IDE 连接按钮，再点击运行。
3. 准备标签：搜索 "AprilTag 36H11 下载"获取官方图案包，选几个 ID 打印成 5~10cm 见方的黑白标签（勿缩放变形）。
4. LCD 与 IDE 同步显示，串口终端打印家族/ID/旋转角。
5. 本例程**不需要额外文件**、无 SD 卡依赖；脱机自启动将 main.py 放入 TF 卡 `/sdcard/`。

> [!NOTE]
> 打印时选"实际大小/100%"，不要"适应页面"，标签必须保持正方形且黑框完整，四周最好留白边。

## 7. 拓展思考

1. **多站点定位**：打印 ID 0~3 四张标签贴在桌面四角当"站点"，识别到不同 ID 时通过 UART 发送 `STATION,n`，模拟 AGV 到站报告。
2. **旋转角度控制**：用 `tag.rotation()` 的角度偏差控制一个舵机（参考正点原子舵机例程）转到对应角度，做"标签对准器"。
3. **位姿估计入门**：查阅 CanMV 文档给 `find_apriltags` 传入 `fx/fy/cx/cy` 相机参数，读取 `x_translation()/z_translation()` 估算标签相对摄像头的位置和距离，理解无人机降落引导的原理（进阶）。

## 8. 常见问题

**Q1：AprilTag 和二维码有什么区别，该用哪个？**
A：二维码追求容量（存网址、文本），AprilTag 追求**远距离、抗畸变的可靠识别 + 位姿估计**，只存一个小 ID。要传内容用二维码（实验13），要做定位/编号路标用 AprilTag。

**Q2：为什么注释说 TAG16H5 看得更远但假阳性高？**
A：16H5 内部只有 4x4 格子，同样打印尺寸下格子更大、更远的距离上仍能分辨，所以看得远；但可用编码少、纠错 H 值低，噪声图案碰巧凑成合法编码的概率高，所以容易误报。36H11 反之：更抗误报但同尺寸下看得稍近。

**Q3：六个家族全开和只开一个，速度差多少？**
A：家族越多，每个候选区域要尝试解码的方案越多，帧率越低。只用一种标签时，把其他 5 行 `tag_families |=` 注释掉，能明显提速并减少假阳性。

**Q4：手机屏幕显示 AprilTag 图案能识别吗？**
A：能，但屏幕像素栅格和反光可能干扰解码，打印件成功率更高。若必须用屏幕，调低亮度、正对摄像头。

**Q5：识别到的 rotation 为什么在 ±180 度跳变？**
A：标签旋转接近 180° 边界时数值会正负跳变（如 179° → -179°），这是角度表示的固有现象；应用时用相邻帧差值或做角度归一化处理。
