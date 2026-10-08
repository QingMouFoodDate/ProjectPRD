# 轻眸识刻模型交付与生命周期候选契约

> **状态**：`model-bundle.v1` 候选。
> **适用范围**：ModelTrain 导出，TrainPlatform 登记/验证/发布治理，InferPlatform 控制激活/回滚，内部 Runtime 校验与执行。

## 一、职责和产物边界

| 主体 | 设计职责 | 不负责 |
| :--- | :--- | :--- |
| ModelTrain | 训练、评测、消融、导出候选包与实验信息 | 客户端 API、日期 Decision、发布审批或线上激活 |
| TrainPlatform | 模型登记、血缘追溯、交付/评测校验、发布授权 | 在线前向推理或修改 active 指针 |
| InferPlatform | 选择授权版本、activate/rollback、记录实际版本 | 模型训练与替代发布审核 |
| Inference Runtime | 哈希/配置/张量校验、模型加载、检测/OCR、Observation | 会话、Evidence、日期 Decision 或发布授权 |

模型输出张量，Runtime 解码后构建 [observation.v1](DATA.md)；模型与 Runtime 均不输出 marked_status、review_status 或食品安全结论。

## 二、候选包必需文件

交付包统一目录结构：

```text
release_bundle/                   # 不可变候选交付包
├── qingmou_det.onnx               # 日期区域检测模型
├── qingmou_rec.onnx               # 局部 OCR 模型
├── dict.txt                      # 字符与类别索引字典
├── preprocess.yaml               # 图像与张量预处理参数
├── postprocess.yaml              # NMS、阈值与解码参数
├── labels.json                   # 检测类别定义
├── metadata.yaml                 # 模型卡、输入输出 Profile 与兼容性
├── manifest.json                 # 文件清单、大小与 SHA-256
├── evaluation.json               # 评测配置、口径与基准指标
└── smoke/                        # 可复现的运行验证输入输出
    ├── input.json                # 输入资产/张量及其哈希
    └── expected.json             # 期望输出与容差范围
```

manifest.files 列出除自身外的全部包文件及 SHA-256（含配置、字典、标签与 smoke），防止自哈希循环。同 model_version 不得换包或重新压缩并沿用旧哈希。

## 三、版本、模型卡与血缘

| 字段/记录 | 候选约束 |
| :--- | :--- |
| model_version | 不可变模型包标识 |
| bundle_version | 包结构版本，当前候选 model-bundle.v1 |
| manifest_sha256 / bundle_sha256 | 外部登记的真实完整性校验引用 |
| 文件清单 | path、size_bytes、sha256；拒绝绝对路径与父目录穿越 |
| 输入输出 Profile | 名字、dtype、shape/动态维度、坐标单位与轴含义 |
| 预后处理 | 通道、缩放、padding、mean/std、阈值、NMS、字典与解码版本 |
| 兼容性 | runtime、min_runtime_version、opset 与目标平台 Profile |
| 血缘 | dataset_version_id、train_config_hash、code_commit、eval_record_id |
| 评测 | 冻结数据版本、分层划分、硬件环境与指标定义 |

模型内容、配置与血缘不可篡改。生命周期状态由外部治理记录追加，不修改包内文件哈希。

## 四、输入输出 Profile 候选

| 模型 | 候选输入 | 候选输出与待核验项 |
| :--- | :--- | :--- |
| 检测 | ONNX、FP32、RGB `[1,3,640,640]`，像素归一化 | `[1,5,N]` 候选布局，输出框坐标须正确映射回原图物理像素 |
| OCR | ONNX、FP32、RGB `[1,3,32,320]` | `[1,T,num_classes]` 候选布局，字典与类别索引对齐 |
| 运行时 | ONNX Runtime，opset 17 路线 | 最低版本与算子支持由实际加载测试确认 |

检测配置须明确 letterbox/padding、NMS 阈值与原图坐标变换；OCR 超宽切片须在 Profile 中明确缩放/分段策略。

## 五、状态与动作

候选状态迁移：`candidate -> validated -> released -> deprecated`。

| model_status | 候选语义 |
| :--- | :--- |
| candidate | 已登记导出候选，待门禁检查 |
| validated | 规定交付、评测与兼容门禁通过 |
| released | 获授权供生产 Runtime 加载（不等于 active） |
| deprecated | 停止新的部署授权 |

| 动作 | 主体 | 门禁与结果 |
| :--- | :--- | :--- |
| register | TrainPlatform 治理或 M0 手工登记 | 包与血缘完整，版本不可重用；幂等登记 |
| validate | 交付校验程序，Train 登记 | 全文件完整性、真实加载、Profile 与 smoke 校验通过后置 validated |
| release | 授权审核人，Train 记录 | 审核用途与许可，追加发布记录后置 released |
| activate | Infer 控制，Runtime 执行 | 指定 released 版本，隔离加载并 smoke 验证后原子切换 active |
| rollback | Infer 控制，Runtime 执行 | 恢复仍获授权且兼容的已验证旧版，重新核验并留痕 |

## 六、激活、失败与回滚

- 同一请求绑定检测、OCR、字典与配置的完整 model_version，严禁混版；Observation 记录实际版本；
- 候选实例隔离加载并执行 smoke 验证后切换引用；切换异常保持原 active 不变；
- 保留仍有授权的 N-1 版本包与配置；回滚失败同样保持原 active 稳定；
- 模型回滚不静默重算历史 Decision；历史记录与当时观察值保持原状。

## 七、端侧分发与冻结检查

C3 端侧元数据接口由 [API](API.md) 定义，返回 released 包的元数据、Profile 及下载清单。客户端下载后完整校验，失败时保留旧版。端侧仅生成候选观察值，端云协议未就绪前不替代统一定决。
