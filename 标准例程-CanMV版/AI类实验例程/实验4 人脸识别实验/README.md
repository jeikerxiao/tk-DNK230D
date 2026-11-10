# 实验4 人脸识别实验

> 分两步实现"刷脸认人"：先运行 main1.py 把照片里的人脸注册成特征库，再运行 main2.py 用摄像头实时识别画面中的人是谁。难度：中级。

## 1. 例程简介

本实验包含**两个脚本**，必须按顺序使用：

- **main1.py（人脸注册）**：不用摄像头，而是遍历 TF 卡 `/sdcard/examples/utils/db_img/` 目录下的照片，检测每张图中的人脸（要求一图一人），提取 128 维人脸特征向量，以 `图片名.bin` 保存到数据库目录 `/sdcard/examples/utils/db/`；
- **main2.py（人脸识别）**：摄像头实时检测人脸 → 提取特征 → 与数据库逐一比对相似度 → 在 LCD 上的人脸框旁显示 `name: xxx, score:0.xx` 或 `unknown`。

你将学到：人脸特征提取、特征向量归一化与相似度（余弦相似度）计算、人脸对齐（Umeyama 算法 + 仿射变换）、文件形式的特征数据库管理。难度为**中级**，预期用途：门禁、考勤、个性化交互设备。

## 2. 硬件连接与原理

用到的硬件：

- **CNK230DF 核心板（K230D）**：RISC-V 双核 + 6TOPS KPU，运行检测与识别两个模型；
- **MIPI CSI 摄像头**：main2.py 中 `Sensor(width=1280, height=960)`，送 AI 分辨率 `rgb888p_size=[1024,768]`（main1.py 不用摄像头，直接读图片文件）；
- **2.4 寸 MIPI LCD**：main2.py 中 `display_mode="lcd"`、`display_size=[640,480]`；
- **TF 卡（/sdcard）**：存放模型、锚框文件、注册照片目录 `db_img/`、特征数据库目录 `db/`。

**人脸识别原理（通俗版）**：识别模型 `face_recognition.kmodel` 不看"整张脸的图片"，而是把一张**对齐后的 112x112 人脸**压缩成 128 个数字（特征向量）——可以理解为这张脸的"数字指纹"。同一个人的指纹彼此接近，不同人的相距很远。比对时把两个 128 维向量各自归一化（长度变为 1），再做点积，得到 -1~1 的余弦相似度；代码把它换算到 0~1：`v_score = np.dot(feature, db_feature)/2 + 0.5`，超过阈值 0.75 就认为是同一个人。

**为什么要"对齐"**：拍照时脸有偏移、旋转、大小差异，直接送模型误差大。代码用检测模型给出的 5 个五官点（双眼、鼻尖、双嘴角），通过 **Umeyama 算法**算出一个仿射矩阵，把脸变换到"标准 112x112 模板"上——标准五官点坐标写死在 `self.umeyama_args_112` 里。注册与识别用同一套对齐流程，特征才可比。

> [!WARNING]
> 新手风险提醒：
> - 摄像头/LCD MIPI 排线必须**断电插拔**，插反通电可能烧坏器件；
> - 供电不足会导致双模型推理时死机重启，请使用足功率 Type-C 电源；
> - AI 持续推理时核心板发热明显，**属正常现象**；
> - main1.py 会向 TF 卡写文件，**运行中不要拔卡断电**，以免数据库损坏。

## 3. 相关前置知识

- **特征向量**：模型输出的一串数字（本实验 128 个 float），代表一张脸的"身份指纹"。代码中 `feature_num = 128`。
- **归一化**：把向量除以自己的长度（`feature /= np.linalg.norm(feature)`），使它变成单位长度。归一化后两个向量的点积就等于夹角的余弦值。
- **余弦相似度**：两个单位向量的点积，范围 -1~1，越接近 1 方向越一致、两张脸越像。代码换算为 `dot/2 + 0.5` 落到 0~1。
- **识别阈值**：`face_recognition_threshold = 0.75`。相似度低于它判为 `unknown`。调高更严格（不容易认错人，但本人也可能被拒），调低更宽松（反之）。
- **关键点（5 官点）**：检测模型除人脸框外还输出 5 个五官点（`landms`），用于人脸对齐。
- **Umeyama 算法**：由两组对应点求最优"旋转+缩放+平移"变换的经典算法，内部用到 SVD 分解（代码里的 `svd22` 是 2x2 矩阵的 SVD）。
- **仿射变换**：见实验2/3；这里由 Umeyama 结果构造，交给 `ai2d.affine()` 硬件执行。
- **锚框 / IoU / NMS / 置信度**：第一级人脸检测用，`confidence_threshold=0.5`、`nms_threshold=0.2`、锚框来自 `prior_data_320.bin`（4200x4）。
- **kmodel / KPU / 张量**：`.kmodel` 为 K230 专用模型格式；KPU 是 AI 加速器；张量即多维数组。
- **帧率（FPS）**：main2.py 实时循环中 `ScopedTiming("total", 1)` 打印每帧耗时。

## 4. 软件代码逻辑说明

### 4.1 两个脚本共同的类

两个文件都定义了：

- **FaceDetApp(AIBase)**：人脸检测。pad（补边色 104/117/123）+ resize 到 320x320，后处理 `aidemo.face_det_post_process`。注意本实验的 `postprocess` 返回**两个值**：`res[0]` 是人脸框、`res[1]` 是 5 官点：

```python
def postprocess(self, results):
    res = aidemo.face_det_post_process(self.confidence_threshold, self.nms_threshold,
        self.model_input_size[0], self.anchors, self.rgb888p_size, results)
    if len(res) == 0:
        return res, res
    else:
        return res[0], res[1]
```

- **FaceRegistrationApp(AIBase)**：人脸特征提取（注册/识别共用同一个模型 `face_recognition.kmodel`）。核心是 `image_umeyama_112()` + `svd22()` 算对齐矩阵，`config_preprocess(landm)` 用 `ai2d.affine` 把脸对齐到 112x112；`postprocess` 直接返回 128 维特征：`return results[0][0]`。

### 4.2 main1.py：人脸注册流程

```python
face_det_kmodel_path = "/sdcard/examples/kmodel/face_detection_320.kmodel"
face_reg_kmodel_path = "/sdcard/examples/kmodel/face_recognition.kmodel"
anchors_path = "/sdcard/examples/utils/prior_data_320.bin"
database_dir = "/sdcard/examples/utils/db/"        # 人脸信息注册数据存放路径
database_img_dir = "/sdcard/examples/utils/db_img/"  # 需要注册的人脸图片存放位置
face_det_input_size = [320, 320]
face_reg_input_size = [112, 112]
max_register_face = 100      # 数据库最多人脸个数
feature_num = 128            # 人脸识别特征维度
```

主程序遍历图片目录：

```python
img_list = os.listdir(database_img_dir)
for img_file in img_list:
    full_img_file = database_img_dir + img_file
    img = image.Image(full_img_file)      # 本地读取一张图像
    img.compress_for_ide()
    rgb888p_img_ndarry = fr.image2rgb888array(img)   # 转rgb888的chw格式
    fr.run(rgb888p_img_ndarry, img_file)             # 人脸注册
    gc.collect()
```

`FaceRegistration.run()` 的关键逻辑：

```python
det_boxes, landms = self.face_det.run(input_np)
if det_boxes:
    if det_boxes.shape[0] == 1:
        # 若是只检测到一张人脸，则将该人脸注册到数据库
        db_i_name = img_file.split('.')[0]
        for landm in landms:
            self.face_reg.config_preprocess(landm, input_image_size=[...])
            reg_result = self.face_reg.run(input_np)
            with open(self.database_dir + '{}.bin'.format(db_i_name), "wb") as file:
                file.write(reg_result.tobytes())
                print('Success!')
    else:
        print('Only one person in a picture when you sign up')
else:
    print('No person detected')
```

要点：**图片文件名（去掉扩展名）就是这个人的名字**；一图多人或无人都会打印提示并跳过。

### 4.3 main2.py：实时识别流程

`FaceRecognition` 类启动时先加载数据库：

```python
def database_init(self):
    db_file_list = os.listdir(self.database_dir)
    for db_file in db_file_list:
        if not db_file.endswith('.bin'):
            continue
        ...
        feature = np.frombuffer(data, dtype=np.float)
        self.db_data.append(feature)
        name = db_file.split('.')[0]
        self.db_name.append(name)
        self.valid_register_face += 1
```

每帧识别与比对：

```python
def run(self, input_np):
    det_boxes, landms = self.face_det.run(input_np)
    recg_res = []
    for landm in landms:
        self.face_reg.config_preprocess(landm)
        feature = self.face_reg.run(input_np)
        res = self.database_search(feature)
        recg_res.append(res)
    return det_boxes, recg_res

def database_search(self, feature):
    ...
    feature /= np.linalg.norm(feature)
    for i in range(self.valid_register_face):
        db_feature = self.db_data[i]
        db_feature /= np.linalg.norm(db_feature)
        v_score = np.dot(feature, db_feature)/2 + 0.5   # 余弦相似度换算到0~1
        if v_score > v_score_max:
            v_score_max = v_score
            v_id = i
    if v_id == -1 or v_score_max < self.face_recognition_threshold:
        return 'unknown'
    else:
        return 'name: {}, score:{}'.format(self.db_name[v_id], v_score_max)
```

绘制：红框 + 文字：

```python
pl.osd_img.draw_rectangle(x1, y1, w, h, color=(255,0,0,255), thickness=4)
pl.osd_img.draw_string_advanced(x1, y1, 32, recg_text, color=(255, 255, 0, 0))
```

主流程（注意 `rgb888p_size=[1024,768]`）：

```python
sensor = Sensor(width=1280, height=960)
pl = PipeLine(rgb888p_size=rgb888p_size, display_size=display_size, display_mode=display_mode)
pl.create(sensor=sensor)
fr = FaceRecognition(..., face_recognition_threshold=0.75, rgb888p_size=[1024,768], display_size=[640,480])
while True:
    os.exitpoint()
    with ScopedTiming("total", 1):
        img = pl.get_frame()
        det_boxes, recg_res = fr.run(img)
        fr.draw_result(pl, det_boxes, recg_res)
        pl.show_image()
        gc.collect()
```

### 4.4 可调参数一览

| 参数 | 文件/位置 | 影响 |
| --- | --- | --- |
| `face_recognition_threshold = 0.75` | main2.py `__main__` | 认人严格程度；本人常被认成 unknown 就调低（如 0.7），总认错人则调高 |
| `confidence_threshold=0.5` / `nms_threshold=0.2` | 两个文件 | 检脸灵敏度 |
| `database_img_dir` | main1.py | 注册照片目录，换成自己的照片路径 |
| `database_dir` | 两个文件 | 特征库目录，两边必须一致 |
| `max_register_face = 100` | main2.py（main1.py 中亦有同名变量） | 数据库最大容量 |
| `rgb888p_size=[1024,768]` | main2.py | 送 AI 分辨率，降低可提帧率 |
| `face_reg_input_size=[112,112]` | 两个文件 | 与识别模型绑定，不可改 |

## 5. 运行效果

- **main1.py（注册）**：LCD 无实时画面（不走 PipeLine），串口逐张打印图片路径，每张成功注册打印 `Success!`；一人一图打印 `Only one person in a picture when you sign up`；没检到脸打印 `No person detected`。运行完 `db/` 目录下生成与图片同名的 `.bin` 文件。
- **main2.py（识别）**：LCD 实时画面中每张人脸套红框，框上方显示 `name: 张三, score:0.83` 之类的文字；数据库里没有的人显示 `unknown`。
- **成功现象**：注册过的人正对摄像头，score 一般 0.75 以上且稳定；陌生人稳定显示 unknown。
- **失败现象排查**：
  - 所有人都显示 unknown → 是否先运行过 main1.py？`db/` 里有没有 `.bin` 文件？识别阈值是否过高？注册照片与实际光照/角度差异过大？
  - score 忽高忽低 → 光线不稳、脸没对正、距离太近太远；
  - 注册时报错 → `db_img/` 目录是否存在、里面是否是支持的图片格式；
  - 模型/锚框路径错误 → 核对 TF 卡文件。

## 6. 代码使用说明

1. **硬件准备**：断电插好摄像头与 LCD 排线，插入 TF 卡，Type-C 上电。
2. **固件要求**：正点原子 **CanMV K230D 固件**（内置 libs、aidemo、nncase_runtime 等），固件与 CanMV IDE 版本匹配。
3. **操作步骤（顺序不能反）**：
   1. 在 TF 卡建好 `/sdcard/examples/utils/db_img/`，放入待注册人的照片（**一图一人**，建议正脸、光照均匀；文件名即人名，如 `zhangsan.jpg` → 注册后叫 zhangsan）；
   2. IDE 打开并运行 **main1.py**，等待串口打印完所有 `Success!`；
   3. IDE 停止，再打开并运行 **main2.py**，对着摄像头验证识别效果。
4. **模型文件需求**（从正点原子资料拷贝到 TF 卡）：
   - `/sdcard/examples/kmodel/face_detection_320.kmodel` —— 人脸检测模型；
   - `/sdcard/examples/kmodel/face_recognition.kmodel` —— 人脸识别（特征提取）模型；
   - `/sdcard/examples/utils/prior_data_320.bin` —— 锚框数据；
   - `/sdcard/examples/utils/db/`、`/sdcard/examples/utils/db_img/` —— 数据库与注册照片目录（需自行创建/放置照片）。
5. **依赖库**：`libs.PipeLine / libs.AIBase / libs.AI2D`（板载 `/sdcard/app/libs/`）、固件内置 `aidemo`、`image`、`ulab.numpy`、`media.sensor`、`nncase_runtime`；不依赖本例程文件夹内其他文件。

> [!NOTE]
> 想清空数据库重新注册：删除 `db/` 目录下所有 `.bin` 文件即可；main2.py 中也提供了 `database_reset()` 方法清空内存中的库（清空后需重启程序重新 `database_init` 才能再加载）。

## 7. 拓展思考

1. **入门级**：注册 3 个不同的人，观察互相之间的 score 与本人 score 的差距，尝试把 `face_recognition_threshold` 调到 0.65 和 0.85，体会"拒识率/误识率"的权衡。
2. **进阶级**：做一个**人脸门禁**——识别到指定人名时点亮绿色（蓝）LED（GPIO59）并让蜂鸣器短鸣一声，unknown 时点亮红色 LED（GPIO61），逻辑加在 main2.py 的 `draw_result` 之后即可。
3. **挑战级**：用摄像头代替照片注册——仿照 main1.py 增加一个"按键 KEY0 按下即抓取当前帧注册"的流程（参考基础实验的按键例程）；或通过 UART 把识别结果（姓名+分数）实时上报 PC 做考勤记录。

## 8. 常见问题

**Q1：main1.py 报 `OSError` / 列不出图片？**
A：确认 TF 卡上 `/sdcard/examples/utils/db_img/` 目录存在且里面有图片文件；目录名区分大小写；图片建议用 JPG/PNG 常规格式。目录不存在时 `os.listdir` 会直接抛异常。

**Q2：为什么注册的照片必须只有一张脸？**
A：代码逻辑规定：`det_boxes.shape[0] == 1` 才注册，多于一张脸打印 `Only one person in a picture when you sign up` 并跳过。因为特征库以"图片文件名=人名"为键，一图多人无法确定特征属于谁。

**Q3：识别时本人也常显示 unknown 怎么办？**
A：按顺序尝试：① 把 `face_recognition_threshold` 从 0.75 降到 0.7 左右；② 注册照片尽量与使用场景一致（同一光照、正脸、相近距离）；③ 同一个人可注册多张不同角度的照片（多张图取最高分）；④ 提高环境光照。

**Q4：识别帧率低，正常吗？**
A：正常。每帧要跑检测 + 每张脸一次特征提取，`rgb888p_size` 又是 1024x768。人多时更慢。可把 `rgb888p_size` 降到 [640,480]，或限制只识别画面中最大的一张脸来提速。

**Q5：kmodel 加载失败？**
A：核对 `/sdcard/examples/kmodel/face_detection_320.kmodel`、`/sdcard/examples/kmodel/face_recognition.kmodel`、`/sdcard/examples/utils/prior_data_320.bin` 三个文件是否齐全、大小写一致，且来自正点原子配套资料（其他平台的 kmodel 不通用）。

**Q6：注册和识别用的必须同一个 `face_recognition.kmodel` 吗？**
A：是的。不同模型提取的特征空间不同，A 模型注册的特征拿 B 模型的特征去比对没有意义。两个脚本中 `face_reg_kmodel_path` 必须指向同一文件。
