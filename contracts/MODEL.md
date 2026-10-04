# 轻眸识刻模型交付与生命周期候选契约

> **状态**：`model-bundle.v1` 候选；当前没有由本文证明的模型、评测、发布或激活实现。
> **适用范围**：ModelTrain 导出，TrainPlatform 登记/验证/发布治理，InferPlatform 控制激活/回滚，内部 Runtime 校验和执行。
> **冻结门槛**：真实模型导出、独立评测、完整性、加载与输入输出 smoke、兼容和失败切换测试通过，保留审核记录。

## 一、职责和产物边界

| 主体 | 设计职责 | 不负责 |
| :--- | :--- | :--- |
| ModelTrain | 训练、评测、消融、导出候选包与实验信息 | 客户端 API、日期 Decision、发布审批或线上激活 |
| TrainPlatform | 模型登记、血缘、交付/评测校验、发布授权 | 在线前向推理或修改 active |
| InferPlatform | 选择授权版本、activate/rollback、记录实际版本 | 模型训练和替代发布审核 |
| Inference Runtime | 哈希/配置/张量校验、模型加载、检测/OCR、Observation | 会话、Evidence、日期 Decision、训练或发布授权 |

网络本身输出张量，Runtime 解码后构建 [observation.v1](DATA.md)；模型与 Runtime 均不输出 marked_status、review_status 或食品安全结论。M0 可手工完成治理记录，不要求先建完整 TrainPlatform，也不要求容器或分布式调度。

## 二、候选包必需文件

以下为目标交付清单，不代表已经有这些文件；算法权重与样本放受控制品存储，不直接进入普通代码 Git 历史。

```text
release_bundle/                   # 不可变候选交付包，目录名不是 model_version
├── qingmou_det.onnx               # 日期区域检测模型
├── qingmou_rec.onnx               # 局部 OCR 模型
├── dict.txt                      # 字符/token 与类别索引映射
├── preprocess.yaml               # 实际图像与张量预处理参数
├── postprocess.yaml              # NMS、阈值、裁剪与解码参数
├── labels.json                   # 检测类别映射
├── metadata.yaml                 # 模型卡、真实输入输出和兼容性
├── manifest.json                 # 文件清单、大小、SHA-256及包格式版本
├── evaluation.json               # 数据、代码、配置、环境、评测口径与结果
└── smoke/                        # 可重复的结构与运行验证输入
    ├── input.json                # 可取得的输入资产/张量及其哈希
    └── expected.json             # 期望输出、坐标和数值容差
```

输入二进制可作为授权 fixture 一并交付，或使用受控可取得的资产引用；依赖未授权、已过期或无法取得的输入不能算 smoke 通过。FP32 为候选基础交付，FP16/INT8及其他格式是可选产物，须单独记录 Profile、校准数据版本及相对基准评测，不把量化或端侧作为 M0 前置要求。

manifest.files 列出除 manifest.json 自身之外的全部包文件及 SHA-256，包含配置、字典、标签和 smoke；避免自哈希循环。manifest 的整体 SHA-256 和交付制品 bundle_sha256 在外部登记记录中绑定。实际打包形式和制品字节必须固定，同 model_version 不得换包、换配置或重新压缩后静默沿用旧哈希。

## 三、版本、模型卡与血缘

| 字段/记录 | 候选约束 |
| :--- | :--- |
| model_version | 不可变模型包标识；不能填 candidate、released 等状态值 |
| bundle_version | 包结构版本，当前候选 model-bundle.v1；不是算法版本 |
| manifest_sha256 / bundle_sha256 | 外部登记的真实完整性引用，不使用示例哈希替代 |
| 文件清单 | path、size_bytes、sha256；相对包路径须拒绝绝对路径、父目录穿越、重复条目和符号链接逃逸 |
| 输入输出 Profile | 实际导出的名字、dtype、shape/动态维度、坐标单位与轴含义 |
| 预后处理 | 通道、缩放、letterbox/padding、mean/std、逆变换、阈值、NMS、字典、blank及解码器版本 |
| 兼容性 | runtime、min_runtime_version、opset、目标平台与各精度Profile；只列通过真实检查的发布适配范围 |
| 血缘 | dataset_version_id、train_config_hash、code_commit、runtime_version、eval_record_id、parent_bundle_id及实际训练输入 |
| 评测 | 冻结数据版本、划分/分层、脚本版本、硬件、输入尺寸、冷/热启动、并发和指标定义 |

模型内容、配置、数据和代码引用不可回写。生命周期状态及审批记录由外部治理记录追加，不通过修改包内 metadata 来改变已登记版本的哈希。包内可记录导出时状态，但消费者必须核验外部发布授权，不能只信文件中的 model_status。

未测量指标使用 null，并注明原因；正式 validated/released 必须有相应门禁所需的真实评测。README、示例元数据、单图置信度或参数量不能作为准确率和速度成绩。字符准确率、字段完全匹配率、CER、参数量、FLOPs和端到端/纯前向耗时分别定义，研究口径见 [RESEARCH](../RESEARCH.md)。

## 四、输入输出 Profile 候选

以下数值仅保留早期设计路线，最终由真实导出确认，不是冻结生产张量或已发布模型能力。

| 模型 | 候选输入 | 候选输出与待核验项 |
| :--- | :--- | :--- |
| 检测 | ONNX、FP32、RGB `[1,3,640,640]`，像素/255 | `[1,5,N]` 仅为候选布局，名称、类别轴、objectness、坐标及N由实际模型确认；不能把8400固定到所有P2结构 |
| OCR | ONNX、FP32、RGB `[1,3,32,320]` | `[1,T,num_classes]` 仅为候选布局，T、blank与类别索引由导出及字典确认 |
| 运行时 | ONNX Runtime，opset 17路线 | 最低版本和算子支持由真实加载测试确认；不是声称任意1.15+或任意设备都兼容 |

检测须在配置中明确 resize/letterbox、padding、NMS前后布局、框的像素或归一化单位以及回原图变换。0.25检测阈值、0.45 IoU仅为历史候选起点，不视为已校准策略。

OCR 候选保持高度32，宽度等比例缩放、右侧白底补齐到320；超宽切片如何缩放/分段必须由Profile明确，不自动截掉字符。候选标准化为 `(pixel / 255 - mean) / std`，mean/std均取0.5时须明确是标准差而非方差，并按实际模型调整。

字符字典保留数字、分隔符及必要汉字/字母候选。EXP/MFG/LOT 在字符级解码时按单字符索引，若采用多字符token则另立Profile。字典顺序、CTC blank/unknown、num_classes、重复字符合并与字符置信对齐必须经测试；不保证受限字典足以识别所有包装说明。

## 五、状态与动作

| model_status | 候选语义 |
| :--- | :--- |
| candidate | 已登记导出候选，尚待规定检查 |
| validated | 规定交付、评测与兼容门禁已通过，但未必获发布授权 |
| released | 获授权供生产 Runtime 加载，不表示已经 active |
| deprecated | 已结束新的部署授权；能否回滚须依据明确治理政策，不默认许可 |

状态与动作分开，候选状态迁移为 candidate -> validated -> released -> deprecated；门禁失败保留原状态。内容变更创建新版本；发布授权撤销立即阻止新激活并触发明确处置，不用状态字符串冒充版本。

| 动作 | 主体 | 门禁与结果 |
| :--- | :--- | :--- |
| register | TrainPlatform治理或M0手工登记 | 包/manifest/评测/血缘完整，版本不可重用；幂等登记 |
| validate | 交付校验与评测程序，Train登记 | 全文件完整性、真实加载、输入输出、字典、坐标、独立评测及兼容检查；通过置validated |
| release | 授权审核人，Train记录 | 审核门禁与用途/许可，追加发布证据，置released；不修改包内容 |
| activate | Infer控制，Runtime执行 | 指定released且兼容版本，隔离加载、smoke，再原子切换active |
| rollback | Infer控制，Runtime执行 | 恢复仍有授权、兼容且可取得的已验证旧版，重新核验并留痕 |

register、validate、release不是线上请求中的自动动作。Runtime只接受明确选择的已授权版本，不自行追随latest。验证环境可以隔离加载candidate执行smoke，但不能把它当作生产版本对外服务。

## 六、激活、失败与回滚

- 同一请求固定检测/OCR/字典/配置的完整 model_version，禁止文件混版；Observation记实际版本，业务层验证请求与返回引用。
- 加载到隔离候选实例后核验结构与smoke，再切换引用；切换失败保留原active，正在执行请求仍使用已固定版本。
- 无可用旧active且初次激活失败时明确不可用，不生成伪成功Observation。
- 保留仍有授权的N-1包和配置；新发布不自动删除或废弃全部旧版。回滚失败同样保留原active。
- 被撤销授权、安全禁用或不能验证完整性的版本不作为回滚目标。处置决定及申请者、审核者、版本、时间、原因、结果均可审计。
- 模型回滚不静默重算历史Decision；历史观察值、Evidence、规则和原观察时间保持原样。

## 七、端侧分发与冻结检查

C3端侧元数据接口由 [API](API.md) 定义，返回released包的model_version、bundle_version、哈希、Profile/兼容信息及受控完整文件清单。客户端下载后校验，更新失败保留可用旧版；端侧只能生成候选观察值，端云接入协议未定义前不得直接替代统一业务Decision。

冻结前至少具备：真实导出文件、匹配字典/预后处理、授权输入smoke、结构和坐标测试、评测数据/代码/配置/硬件记录、制品及manifest哈希、发布审批、激活失败和回滚演练、观察值实际版本记录。当前文档及Schema校验不等于这些实验或运行验证已完成。
