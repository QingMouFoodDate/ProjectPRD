# 轻眸识刻离线数据与训练执行候选契约

> **状态**：`training.v1` 候选。
> **边界**：TrainPlatform 治理、ModelTrain 执行；InferPlatform 仅移交受控待审记录。M0 允许单机执行与手工登记。
> **依据**：[DATA.md](DATA.md)、[MODEL.md](MODEL.md)、[数据质量规范.md](../standards/数据质量规范.md)、[系统架构.md](../系统架构.md)。

## 一、领域实体与唯一名称

| 实体 | 最小候选字段或引用 | 约束 |
| :--- | :--- | :--- |
| `SampleAsset` | 来源、许可、审核、文件/标注、split/group_key | 不等同在线 Image；未审不入训练 |
| `ManagedFile` | file_id、sha256、size_bytes、asset_ref、created_at | 内容去重不消除逻辑样本各自的许可；受控引用不泄露路径 |
| `DatasetVersion` | dataset_version_id、parent_version、status、manifest_sha256、created_at、frozen_at | 冻结后样本、标注与切分均不可改 |
| `DatasetVersionItem` | dataset_version_id、sample_id、annotation_version、split、group_key | 明确标注修订版本，同源组不跨 split |
| `LabelTask` | label_task_id、sample_ids、annotation_types、assignee_ref、status | 划定标注范围与任务授权 |
| `Annotation` | annotation_id、sample_id、annotation_version、annotator_ref、content_ref | 修订新增不覆盖；包含框、转录与语义 |
| `LabelReview` | review_id、annotation_id、reviewer_ref、reviewed_at、result、reasons | result 为 APPROVED/REJECTED，未通过禁用 |
| `TrainJob` | train_job_id、mode、attempt、dataset_version_id、train_config_hash、code_commit、status | 调度事实由 Train 管理，算法不反向改状态 |
| `EvalRecord` | eval_record_id、model_version、dataset_version_id、split、evaluation_config_hash、metrics | 记录评测口径、分层与环境，不填示例为实测 |
| `ModelBundle` | model_version、bundle_version、manifest_sha256、bundle_sha256、dataset_version_id | 内容与血缘不可改；状态与发布独立追加 |

样本采集、排除特征与标注规则严格依据 [数据质量规范.md](../standards/数据质量规范.md)，本契约不作重复定义。

## 二、纠偏移交、质量审核与冻结

1. InferPlatform 接受合法 Correction 后即时重算 Evidence/Decision，保存待审记录；不覆盖原 Observation；
2. 移交记录引用 correction_id、session_id、image_id、原观察值与受控 asset_ref；
3. TrainPlatform 独立核验来源许可、原图质量、标注与同源分组；
4. 审核通过（APPROVED）获得训练候选资格；通过审核不自动加入数据集或启动训练；
5. 将候选样本纳入新 DatasetVersion，补齐标注、审核、切分并冻结。

DatasetVersion 候选状态为 DRAFT、FROZEN、DEPRECATED。冻结时固化全部成员、文件哈希、标注版本与 split/group_key，置 FROZEN。修改必须派生新版本。

split 分为 train、val、test；同商品、包装批次、采集会话按 `group_key` 切分，严防跨集合泄漏。测试集严禁参与调参。

## 三、任务级输入输出

执行入口由 ModelTrain 实现仓声明（CLI 或受控 worker）：

### 1. 执行请求

| 必需内容 | 候选结构与规则 |
| :--- | :--- |
| 身份 | schema_version=training.v1、train_job_id、attempt（首次为 1） |
| mode | TRAIN / EVALUATE / EXPORT |
| 数据 | dataset_version_id、manifest_sha256、受控 asset_ref（必须 FROZEN） |
| 配置 | train_config_hash、config_ref（校验配置内容一致） |
| 代码 | 完整 code_commit 及工作区状态引用 |
| 环境 | environment_ref、runtime_version、设备与依赖约束 |
| 父模型 | parent_bundle_id（可为空；微调必须校验父包关系） |
| 输出 | 受控 output_ref 与隔离目录，不得与只读冻结输入重叠 |

ModelTrain 严格只读消费数据与标注，禁止重切分或写回冻结目录；校验不一致时拒绝执行。

### 2. 执行结果与事件

结束结果包含退出状态、数据/代码/配置引用、EvalRecord、候选包与哈希；未产出字段为 null。SUCCEEDED 表示执行与产物校验通过，不代表已发布。

失败或中断保留错误类别与阶段记录，输出隔离，不登记为有效 ModelBundle。

## 四、状态、重试与幂等

| 当前 TrainJob 状态 | 允许的后继及条件 |
| :--- | :--- |
| PENDING | RUNNING；校验失败到 FAILED；取消到 CANCELLED |
| RUNNING | 成功到 SUCCEEDED；失败到 FAILED；终止到 CANCELLED |
| FAILED | 符合重试策略且旧进程已隔离到 RETRYING；最终失败保留 |
| RETRYING | 递增 attempt 并准备后到 RUNNING；失败到 FAILED |
| SUCCEEDED / CANCELLED | 终态，不可逆转 |

- 重试复用 train_job_id，递增 `attempt`，以 `(train_job_id, attempt)` 建立幂等追踪；
- 看门狗超时先确认终止旧进程并隔离写入权限，再行重试；
- 启动前检查算力与存储，超出单机容量排队调度。

## 五、评测、登记与发布

ModelTrain 产出实验与评测记录；TrainPlatform 登记并比对。结论须关联数据版本、分层、硬件与指标定义。

ModelTrain 仅交付 candidate 产物；TrainPlatform 治理 register/validate/release；InferPlatform 控制 activate/rollback。register 以不可变 model_version 与制品哈希幂等登记。

## 六、权限、保留与冻结检查

数据读取、标注、审核、调度与发布为独立授权动作；任务仅分配只读数据与隔离输出权限。资产清理前检查冻结版本与血缘引用，不得删除仍被引用的文件。

冻结前核验项：状态迁移合法性、重复回调幂等、看门狗隔离、同源组泄漏检查、来源许可、标注修订历史、冻结只读性、模型同版异包拒绝与发布授权留痕。
