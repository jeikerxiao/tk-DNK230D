# 实验18 车牌号识别实验

> 本例程在正点原子 DNK230D（K230D BOX）开发板上运行"车牌检测 + 字符识别"两级模型，先用四边形框出车牌，再把车牌矫正抠图后送入字符识别模型，直接在屏幕上显示出完整车牌号（如"粤A12345"）。难度：高级。

## 1. 例程简介

本实验是实验17（车牌检测）的完整版：在检测出车牌四边形之后，把倾斜的车牌**矫正并抠图**，送入 CRNN 风格的字符识别模型，逐列预测每个位置的字符，再经过类似 CTC 的去重解码得到车牌号字符串，最终在屏幕上同时显示四边形框和识别出的车牌文字。

- **知识点**：两级级联模型（检测→识别）、`aidemo.ocr_rec_preprocess` 四边形矫正抠图、按检测框尺寸动态重配 Ai2d 预处理、字符字典解码、CTC 式"去重复+去空白"后处理、中文字符绘制。
- **难度**：高级（涉及两个 kmodel、动态预处理配置和序列解码逻辑）。
- **用途**：停车场出入口系统、交通违章抓拍、物流车辆登记等车牌识别（LPR）场景的原理演示。

> [!NOTE]
> 本例程使用两个模型：
> - 车牌检测：`/sdcard/examples/kmodel/LPD_640.kmodel`，输入 640×640；
> - 车牌字符识别：`/sdcard/examples/kmodel/licence_reco.kmodel`，输入 220×32。

## 2. 硬件连接与原理

### 2.1 硬件清单

| 硬件 | 说明 |
| --- | --- |
| CNK230DF 核心板 | 嘉楠 K230D 芯片（RISC-V 双核 + 6TOPS 算力 KPU） |
| DNK230D 底板 | 提供 LCD、摄像头、TF 卡槽等接口 |
| 2.4 寸 MIPI LCD | 带电容触摸，640×480，显示检测框与车牌号 |
| MIPI CSI 摄像头 | `Sensor(width=1280, height=960)` 采集图像 |
| TF 卡 | 存放固件、两个模型文件和 `libs` 库 |

本实验只使用摄像头和 LCD。

> [!WARNING]
> - 摄像头/LCD 排线**插反会烧坏接口**，务必断电后按正确方向插接，禁止带电插拔。
> - 本实验每帧要跑两个模型（多车牌时识别模型还会跑多次），KPU 负荷大、芯片发热明显，属正常现象。
> - 请保证供电充足，供电不足会导致重启或识别异常。

### 2.2 工作原理（通俗版）

1. 摄像头采集 1280×960 图像，PipeLine 缩成 640×360 的 AI 图。
2. **第一级（检测）**：Ai2d 直接 resize 成 640×640，KPU 运行 LPD_640.kmodel，`aidemo.licence_det_postprocess` 输出每块车牌的四个角点。
3. **矫正抠图**：`aidemo.ocr_rec_preprocess` 根据四角点做透视矫正，把倾斜的车牌"掰正"并裁剪成标准小图（每块车牌一张），返回给 MicroPython 侧。
4. **第二级（识别）**：对每张车牌小图，按其实际尺寸重新配置 Ai2d（resize 到 220×32），KPU 运行 licence_reco.kmodel。模型输出是"每个横向位置 × 74 个字符类别"的得分矩阵。
5. **解码**：对每个位置取 argmax 得到最可能字符；跳过空白（index 0），并把连续重复的字符合并成一个（CTC 解码思想），查 `dict_rec` 字典拼出车牌号字符串。
6. **绘制**：绿色四边形框 + 框上方橙色车牌号文字一起显示到 LCD。

## 3. 相关前置知识

- **两级级联模型**：先"找到车牌在哪"，再"看清写了什么"。第二级只对第一级抠出的小图推理，输入小、精度高，这是 OCR 类任务的通用套路。
- **透视矫正**：倾斜车牌的四个角点可以确定一个透视变换，把车牌"掰正"成长条形标准图，字符识别模型只认这种摆正的输入。
- **CRNN / 序列识别**：识别模型把车牌图切成许多竖条，对每个竖条位置输出一个字符类别的概率分布，因此输出形状可整理为 `(位置数, 74)`。
- **字符字典 dict_rec**：模型输出的类别索引到字符的映射表，包含省份简称（挂、使、领、澳、港、皖、沪、京、粤……）、数字 0–9、大写字母（无 I、O，避免与 1、0 混淆）以及占位符 `_`、`-`，共 74 项。
- **CTC 式解码**：模型在每个位置都可能重复预测同一字符、也会输出"空白"（index 0）。解码规则：跳过空白、合并相邻重复，剩下的字符按序拼接。
- **置信度阈值 / NMS / IoU**：与实验17 相同，用于检测级筛选与去重（本例 conf=0.2、nms=0.2）。
- **动态预处理配置**：每块车牌抠出的小图尺寸不同，所以识别模型每处理一张图前都要调用一次 `config_preprocess(input_image_size=...)` 重配 Ai2d。
- **推理/张量**：KPU 计算的输入输出都是多维数组（张量）；识别模型输入张量形状为 `[1,3,32,220]`（NCHW）。

## 4. 软件代码逻辑说明

### 4.1 程序整体流程

```
初始化 PipeLine（摄像头 + LCD）
→ 创建 LicenceRec（内部创建检测 App + 识别 App，加载两个 kmodel）
→ 主循环：取帧 → 车牌检测 → ocr_rec_preprocess 矫正抠图
   → 对每张车牌图：重配 Ai2d → 字符识别 → CTC 解码出字符串
→ 绘制四边形框 + 车牌号文字 → 显示 → gc.collect()
→ 退出时两个 App 分别 deinit() + pl.destroy()
```

### 4.2 模型与参数配置（`__main__` 部分）

```python
display_mode="lcd"
display_size=[640,480]
licence_det_kmodel_path="/sdcard/examples/kmodel/LPD_640.kmodel"     # 检测模型
licence_rec_kmodel_path="/sdcard/examples/kmodel/licence_reco.kmodel" # 识别模型
rgb888p_size=[640,360]
licence_det_input_size=[640,640]
licence_rec_input_size=[220,32]
confidence_threshold=0.2
nms_threshold=0.2
sensor = Sensor(width=1280, height=960)
```

### 4.3 LicenceDetectionApp（检测类）

与实验17 完全一致：预处理仅 resize；后处理调用

```python
det_res = aidemo.licence_det_postprocess(results, [self.rgb888p_size[1], self.rgb888p_size[0]],
                                         self.model_input_size, self.confidence_threshold, self.nms_threshold)
```

### 4.4 LicenceRecognitionApp（识别类）

字符字典与后处理解码是本类的核心：

```python
self.dict_rec = ["挂", "使", "领", ..., "Y", "Z", "_", "-"]
self.dict_size = len(self.dict_rec)   # 74

def postprocess(self,results):
    output_data=results[0].reshape((-1,self.dict_size))  # (位置数, 74)
    max_indices = np.argmax(output_data, axis=1)         # 每个位置最可能的类别
    result_str = ""
    for i in range(max_indices.shape[0]):
        index = max_indices[i]
        # 跳过空白(index==0)，合并相邻重复字符
        if index > 0 and (i == 0 or index != max_indices[i - 1]):
            result_str += self.dict_rec[index - 1]
    return result_str
```

> [!NOTE]
> `index - 1` 是因为类别 0 被留作"空白符"，真正字符从字典第 0 项开始对应类别 1。

### 4.5 LicenceRec（调度类）

```python
def run(self,input_np):
    det_boxes=self.licence_det.run(input_np)              # 第一级：检测
    imgs_array_boxes = aidemo.ocr_rec_preprocess(input_np,
        [self.rgb888p_size[1],self.rgb888p_size[0]],det_boxes)  # 矫正抠图
    imgs_array = imgs_array_boxes[0]   # 每张车牌的图像数组列表
    boxes = imgs_array_boxes[1]
    rec_res = []
    for img_array in imgs_array:
        # 按当前车牌图的实际尺寸重配识别模型的预处理
        self.licence_rec.config_preprocess(input_image_size=[img_array.shape[3],img_array.shape[2]])
        licence_str=self.licence_rec.run(img_array)       # 第二级：识别
        rec_res.append(licence_str)
        gc.collect()
    return det_boxes,rec_res
```

### 4.6 绘制结果

```python
def draw_result(self,pl,det_res,rec_res):
    pl.osd_img.clear()
    if det_res:
        point_8 = np.zeros((8),dtype=np.int16)
        for det_index in range(len(det_res)):
            for i in range(4):   # 坐标从 AI 分辨率换算到显示分辨率
                x = det_res[det_index][i*2+0]/self.rgb888p_size[0]*self.display_size[0]
                y = det_res[det_index][i*2+1]/self.rgb888p_size[1]*self.display_size[1]
                ...
            for i in range(4):   # 绿色四边形
                pl.osd_img.draw_line(..., color=(255, 0, 255, 0), thickness=4)
            # 车牌号文字（橙色，字号40，位于四边形左下角上方）
            pl.osd_img.draw_string_advanced(point_8[4], point_8[5]-42, 40,
                                            rec_res[det_index], color=(255,255,153,18))
```

### 4.7 主循环

```python
while True:
    os.exitpoint()
    with ScopedTiming("total",1):
        img=pl.get_frame()
        det_res,rec_res=lr.run(img)          # 检测 + 识别
        lr.draw_result(pl,det_res,rec_res)
        pl.show_image()
        gc.collect()
```

退出时在 `finally` 中分别释放两个模型：`lr.licence_det.deinit()`、`lr.licence_rec.deinit()`，再 `pl.destroy()`。

### 4.8 可调参数一览

| 参数 | 位置 | 作用 |
| --- | --- | --- |
| `confidence_threshold=0.2` | `__main__` | 检测级置信度阈值，误检多则调大 |
| `nms_threshold=0.2` | `__main__` | 检测级去重阈值 |
| `rgb888p_size=[640,360]` | `__main__` | AI 图分辨率，调大利于小车牌但更慢 |
| `licence_rec_input_size=[220,32]` | `__main__` | 识别模型输入尺寸，**必须与 kmodel 匹配，不可随意改** |
| 文字颜色/字号 `(255,255,153,18)`、`40` | `draw_result` | 车牌号显示样式 |
| `THRESH` 类阈值 | — | 识别级无阈值，靠 argmax 直接取最优字符 |

## 5. 运行效果

- **屏幕输出**：LCD 实时显示摄像头画面；对准车牌后，车牌被绿色四边形框住，框的左下角上方显示橙色的识别结果文字，如 `粤A12345`。多块车牌会被逐一框出并分别识别。
- **串口输出**：`print(det_res, rec_res)` 被注释，打开后可看到每帧的角点坐标和识别字符串列表；`ScopedTiming("total",1)` 打印每帧总耗时。
- **成功标志**：清晰正对车牌时，识别结果稳定且与真实车牌号一致。
- **失败现象排查**：
  - 框出来了但文字乱码/缺字 → 车牌太小或倾斜过大导致矫正图模糊，靠近、摆正后重试。
  - 完全无框 → 参照实验17：降低 `confidence_threshold`、增大车牌在画面中的占比。
  - 帧率低 → 正常，双模型级联且识别按车牌数量循环执行；单块车牌场景最快。
  - 报模型路径错误 → 检查 TF 卡上 `LPD_640.kmodel` 和 `licence_reco.kmodel` 是否都已拷贝。

## 6. 代码使用说明

1. **固件准备**：烧录与 CanMV IDE 版本匹配的 **CanMV K230D 固件**（正点原子出厂固件即可）。
2. **拷贝模型文件**：将正点原子资料中的模型拷贝到 TF 卡，本实验需要：
   - `/sdcard/examples/kmodel/LPD_640.kmodel`（车牌检测）
   - `/sdcard/examples/kmodel/licence_reco.kmodel`（车牌字符识别）
3. **依赖库确认**：`libs.PipeLine`、`libs.AIBase`、`libs.AI2D` 位于 `/sdcard/app/libs/`；后处理依赖固件内置 `aidemo`（`licence_det_postprocess`、`ocr_rec_preprocess`）、`nncase_runtime`、`ulab.numpy`，无需第三方库。
4. **运行步骤**：
   - 打开 CanMV IDE → 连接开发板 → 打开本目录 `main.py` → 运行；
   - 将摄像头对准真实车牌或打印/屏幕显示的车牌图片；
   - 观察 LCD 上的绿色四边形框与识别出的车牌号；
   - 停止运行，程序自动释放两个模型与 PipeLine 资源。

> [!NOTE]
> 测试建议：正对车牌、距离 20~50cm、光线均匀，识别率最高。反光、遮挡、大角度倾斜都会显著降低识别率。

## 7. 拓展思考

1. **结果稳定性过滤**：单帧识别偶尔出错字符，尝试在 `LicenceRec.run` 外维护一个历史列表，对连续多帧的识别结果做"多数表决"，只输出稳定出现 N 次以上的车牌号。
2. **白名单闸机模拟**：定义一个允许通行的车牌号列表，识别结果命中列表时把四边形框画成绿色、未命中画成红色，模拟停车场道闸逻辑（可进一步结合 GPIO 控制舵机/蜂鸣器）。
3. **识别耗时分析**：把 `debug_mode` 置 1，观察 `ScopedTiming` 打印的各阶段耗时，分析画面中车牌数量对帧率的影响，思考如何用"检测帧率全速、识别隔帧执行"的策略优化。

## 8. 常见问题

**Q1：识别结果里少了重复的字符，比如"粤BB1233"识别成"粤B123"？**
A：解码时 `index != max_indices[i-1]` 会合并相邻重复字符（CTC 规则），依赖模型输出中的空白符来分隔真正的重复字符。若车牌模糊，模型未在重复字符间输出空白，就会误合并。改善方法：靠近车牌、提高清晰度、增大 `rgb888p_size`。

**Q2：为什么字典里没有字母 I 和 O？**
A：中国车牌编码规则中不使用 I、O（易与数字 1、0 混淆），所以模型字典本身就不包含这两个字母。

**Q3：识别模型为什么每处理一张车牌都要重新 `config_preprocess`？**
A：`aidemo.ocr_rec_preprocess` 抠出的车牌图尺寸随车牌在画面中的大小变化，Ai2d 的输入尺寸必须与实际图像一致，因此每次都要用 `input_image_size=[img_array.shape[3],img_array.shape[2]]` 重配。

**Q4：两个模型可以同时换掉吗？比如用自己训练的车牌模型？**
A：可以，但要满足：检测模型输出格式与 `aidemo.licence_det_postprocess` 约定一致；识别模型输入 220×32、输出类别数与 `dict_rec`（74 项）一致。否则需要同步修改后处理代码。

**Q5：画面上文字显示为方块或乱码？**
A：`draw_string_advanced` 依赖固件内置的中文字库。若省份简称显示异常，请确认使用的是正点原子配套固件，不要精简 TF 卡上的字体资源文件。

**Q6：程序运行一段时间后内存不足报错？**
A：多车牌场景每帧会生成多张抠图数组。代码已在每块车牌识别后调用 `gc.collect()`；若仍出现内存问题，减少画面中同时出现的车牌数量，或适当降低 `rgb888p_size`。
