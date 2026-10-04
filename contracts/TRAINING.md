# 轻眸识刻离线数据与训练执行候选契约

> **状态**：`training.v1` 候选，尚无由本文证明的数据/标注平台、调度器或训练入口实现。
> **边界**：TrainPlatform治理、ModelTrain执行；InferPlatform只移交受控待审核记录。M0允许手工登记和单机执行，不依赖完整管理平台。
> **依据**：[DATA](DATA.md)、[MODEL](MODEL.md)、[数据质量](../standards/DATA_QUALITY.md)、[研究](../RESEARCH.md)、[ADR-001](../decisions/ADR-001-REPOSITORY_BOUNDARIES.md)。

## 一、领域实体与唯一名称

| 实体 | 最小候选字段或引用 | 约束 |
| :--- | :--- | :--- |
| SampleAsset | DATA中的来源、许可、审核、文件/标注、split/group_key | 不等同在线Image；未经审核不入训练 |
| ManagedFile | file_id、sha256、size_bytes、asset_ref、created_at | 文件内容去重复用不消除逻辑样本各自的来源许可；受控引用不泄露物理路径 |
| DatasetVersion | dataset_version_id、parent_version、status、manifest_sha256、created_at、frozen_at、change_reason | 冻结后样本、文件、标注和切分均不可改 |
| DatasetVersionItem | dataset_version_id、sample_id、annotation_version、split、group_key | 一个冻结成员使用明确标注修订，同源组不跨split |
| LabelTask | label_task_id、sample_ids、annotation_types、assignee_ref、status、created_at | 标注范围、任务授权和返工可追溯 |
| Annotation | annotation_id、sample_id、annotation_version、annotator_ref、created_at、content_ref | 修订新增，不覆盖；内容包括任务所需框、转录、日期语义与质量 |
| LabelReview | review_id、annotation_id、annotation_version、reviewer_ref、reviewed_at、result、reasons | result为APPROVED/REJECTED，未通过不得使用 |
| TrainJob | train_job_id、mode、attempt、dataset_version_id、train_config_hash、code_commit、environment_ref、parent_bundle_id、status、created_at | 调度事实由Train管理，算法不反向改平台状态 |
| EvalRecord | eval_record_id、model_version、dataset_version_id、split、evaluation_config_hash、code_commit、environment_ref、metrics、created_at | 记录测量口径、分层和运行环境，不填示例为实测 |
| ModelBundle | model_version、bundle_version、manifest_sha256、bundle_sha256、dataset_version_id、train_config_hash、code_commit、runtime_version、eval_record_id、parent_bundle_id | 内容/血缘不可回写；状态及发布事件独立追加，遵循MODEL |

ParentDataset与ParentBundle是血缘关系的说明词，不另建第二套实体；对应字段分别使用parent_version与parent_bundle_id。AnnotationJob、TrainingJob、Evaluation、ModelArtifact不是这些实体的隐式别名。字段表是候选最小协议，不证明已存在数据库或管理API；新增字段必须同步事实源、实现及测试。

## 二、纠偏移交、质量审核与冻结

1. Infer接受合法Correction后立即重算当前Evidence/Decision，并保存待训练审核记录；不等待训练审核，不覆盖Observation。
2. 移交引用correction_id、session_id、image_id、原观察值及新旧证据、受控asset_ref，以及允许训练用途的授权依据。匿名会话凭证不作为训练使用许可。
3. Train独立核验来源/许可、原图质量、标注正确性与同源分组；审核事件含review_id、reviewer_ref、reviewed_at、result、reasons及引用。
4. APPROVED才获得训练候选资格；REJECTED保留原因。通过审核仍不自动加入数据集、不自动启动任务或发布模型。
5. 明确把候选纳入新的DatasetVersion、补标注、审核、切分并冻结。相同图片不同纠偏修订保持各自历史，不能覆盖冻结版本。

DatasetVersion候选状态为DRAFT、FROZEN、DEPRECATED。DRAFT可按审核流程修订；冻结时固化全部成员、原图文件哈希、标注修订及split/group_key，追加审计事件并置FROZEN。内容修改创建新版本，parent_version可为空或引用已存在父版本，血缘不可循环。DEPRECATED不删除历史或自动授权清理引用。

split使用train、val、test；同商品、包装批次、采集会话、连拍与近重复簇按关联组切分。冻结至少检查训练/验证输入非空，发布评测还须满足独立冻结测试集门禁；不能用训练或调参数据冒充独立测试。测试集不用于调参。来源/许可、审阅、标注、文件/清单一致性及组泄漏检查任一失败拒绝冻结；哈希唯一不证明实拍真实性。

## 三、任务级输入输出

不规定尚不存在的模块名或可运行命令。入口由ModelTrain实现仓显式声明，可以是CLI或受控worker；容器是可选环境包装，不强制Kubernetes或远程编排。

### 1. 执行请求

| 必需内容 | 候选结构与规则 |
| :--- | :--- |
| 身份 | schema_version=training.v1、train_job_id、attempt正整数，首次为1 |
| mode | TRAIN / EVALUATE / EXPORT，实际入口须声明支持范围 |
| 数据 | dataset_version_id、manifest_sha256、受控dataset_asset_ref，必须FROZEN并可完整取得 |
| 配置 | train_config_hash、config_ref，校验实际配置内容一致；不能执行客户端注入的任意shell |
| 代码 | 完整code_commit及可重现工作区；未记录本地改动不得冒充该提交产物 |
| 环境 | environment_ref、runtime_version、设备/资源约束；固定可取得环境与依赖清单 |
| 父模型 | parent_bundle_id可为空；续训/微调必须校验父包、checkpoint、配置和数据关系 |
| 输出 | 受控output_ref与每attempt隔离位置，不能与只读冻结输入重叠 |

平台以版本与受控引用组装请求；执行器在可信边界解析文件位置，客户端不获得路径。ModelTrain只读数据和标注，不能直接重切分或写回冻结目录；拒绝清单、配置、代码、环境或父包校验不一致。

### 2. 执行结果与事件

每个事件保留event_id、train_job_id、attempt、worker_ref、event_time、state及必要error/产出引用；同任务事件顺序和回调身份须校验。具体CLI输出/回调传输与授权在实现前冻结，不能据此宣称已经有服务。

结束结果包含明确退出状态、实际数据/代码/配置/环境引用、EvalRecord/实验记录、候选包与manifest/bundle哈希，未产出字段为null且不得当作成功交付。独立评测指标差于目标不等于训练进程故障：SUCCEEDED表示指定执行和产物校验成功，不表示已validated/released或指标达标。

失败或中断结果保留错误类别、发生阶段与关联引用；错误日志不含秘密、完整敏感转录或不必要物理路径。不完整包与失败attempt输出隔离，不登记为有效ModelBundle。

## 四、状态、重试与幂等

| 当前TrainJob状态 | 允许的后继及条件 |
| :--- | :--- |
| PENDING | RUNNING；启动校验失败到FAILED；取消到CANCELLED |
| RUNNING | 成功到SUCCEEDED；失败到FAILED；worker确认停止后到CANCELLED |
| FAILED | 符合重试政策、未超上限且旧worker已终止或隔离才到RETRYING；最终失败保留 |
| RETRYING | 递增attempt并准备新执行后到RUNNING；准备失败到FAILED；取消到CANCELLED |
| SUCCEEDED / CANCELLED | 终态，回调不得恢复执行或回退状态 |

- 重试使用同一train_job_id，仅attempt递增；产出登记以(train_job_id, attempt)幂等，重复当前回调只登记一次。
- watchdog超时先停止旧worker或隔离其计算/输出写权限，再重试。无法确认隔离则停止自动重试并转人工，防止双worker同时写包。
- 迟到的旧attempt事件不能修改当前状态、重置active或登记为当前有效产物；保留审计及隔离输出。
- 启动前检查算力、并发和存储，超出单机容量排队；重试上限、超时及取消策略须版本化并验证。
- checkpoint必须通过哈希与输入相容检查；更换数据、代码、配置或父包必须显式新任务，不伪装成同输入重试。

## 五、评测、登记与发布

ModelTrain产生实际实验与评测记录；Train登记及比较。每个结论关联数据版本/分层、评测脚本、配置、模型、设备、精度、输入、冷/热启动、并发、计时边界及样本失败统计。字符、字段、业务状态、前向与端到端指标分别定义；H1-H8仅引用RESEARCH，不在本协议复制研究表或预设改善。

ModelTrain只交付candidate，Train执行register/validate/release治理，Infer执行activate/rollback；状态、文件、授权与失败切换以MODEL为准。register以不可变model_version及制品哈希幂等：同版同包复用登记，同版异包拒绝。血缘事实一次登记、内容不覆盖，审批和状态追加留痕。

M0可采用审核人与单机工具执行上述门禁并保存记录，而非先实现平台UI/服务。是否具备真实训练、smoke、评测或发布以产物和运行记录为证据，不从README或候选协议推定。

## 六、权限、保留与冻结检查

数据读取、标注、审核、调度、删除和发布是独立授权动作；工作者仅取得当前任务所需的只读数据与隔离输出权限。日志记录必要ID、版本、事件、耗时和脱敏错误。资产删除先检查冻结版本、实验和模型引用，不能清理仍用于血缘或历史复现的文件。

冻结前至少测试：合法/非法状态迁移、重复/迟到回调、watchdog与取消、资源不足、同源组泄漏、来源许可、标注修订、冻结只读、配置/代码/哈希不一致、失败产物隔离、父版本循环、模型同版异包、独立评测与发布授权。当前仅为候选协议，未声称这些实现测试已运行。
