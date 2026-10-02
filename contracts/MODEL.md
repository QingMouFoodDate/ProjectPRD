# 《轻眸识刻》算法模型轻量化交付规约

> **规范性质**：算法研发到推理运行时的标准化模型交付硬性契约  
> **适用范围**：`ModelTrain`（算法导出）与 `TrainPlatform`（推理服务加载）

---

## 一、 小目标检测模型交付规约

### 1. 模型格式与输入规格
- **模型文件**：`qingmou_det.onnx`
- **运行框架**：ONNX Runtime 1.15+（`opset 17`）；
- **数据精度**：默认提供 FP32，可选配 INT8 量化版本；
- **输入张量**：`images: [1, 3, 640, 640]`，数据类型 `float32`；
- **预处理归一化**：RGB 通道顺序，像素值除以 255.0 进行 `[0.0, 1.0]` 归一化。

### 2. 输出张量
- **输出张量**：`output0: [1, 5, 8400]`，数据类型 `float32`；
- **通道含义**：
  - `[0..3]`：检测框中心点与宽高归一化坐标 `(cx, cy, w, h)`；
  - `[4]`：打码目标类别置信度分数 `score`；
- **后处理约定**：由推理引擎执行非极大值抑制（NMS），默认置信度过滤阈值 0.25，IoU 阈值 0.45。

---

## 二、 包装日期 OCR 识别模型交付规约

### 1. 模型格式与输入规格
- **模型文件**：`qingmou_rec.onnx`
- **运行框架**：ONNX Runtime 1.15+（`opset 17`）；
- **数据精度**：默认提供 FP32，可选配 INT8 量化版本；
- **输入张量**：`input: [1, 3, 32, 320]`，数据类型 `float32`；
- **预处理约定**：保持高度为 32，沿宽度方向等比例缩放并在右侧进行对称白底 Padding 补齐，标准化为均值 0.5、方差 0.5。

### 2. 输出与字典集映射
- **输出张量**：`output: [1, 80, num_classes]`，数据类型 `float32`；
- **解码机制**：采用 CTC 贪婪解码或束搜索（Beam Search）；
- **受限字符字典表（`dict.txt`）**：
  - 仅包含阿拉伯数字 `0-9`；
  - 常见工业分隔符：`-`, `/`, `.`, `:`, 空格；
  - 少量核心汉字与缩写词：`生`, `产`, `日`, `期`, `保`, `质`, `至`, `效`, `年`, `月`, `EXP`, `MFG`, `LOT`。

---

## 三、 交付产物目录结构与模型卡片

每次算法发布必须以自包含打包形式交付，目录结构如下：

```text
release_bundle_v2/
├── qingmou_det.onnx               # 极小目标日期打码检测 ONNX 模型文件
├── qingmou_rec.onnx               # 微型文本与断笔点阵识别 ONNX 模型文件
├── dict.txt                       # 工业打码数字与字符字典映射表文件
└── metadata.yaml                  # 模型元数据卡片与输入尺寸评测指标声明
```

### 模型元数据卡片示例

```yaml
model_version: "v2.0"
created_at: "2026-10-02"
detection:
  architecture: "YOLO-P2"
  input_profile: [640, 640]
  mAP_0.5: 0.938
  avg_latency_ms: 18.2
recognition:
  architecture: "CRNN-MobileNetV4"
  input_profile: [32, 320]
  char_accuracy: 0.914
  cer: 0.042
  avg_latency_ms: 11.5
environment:
  opset: 17
  cuda_version: "12.1"
  recommended_runtime: "onnxruntime-gpu>=1.15.0"
```
