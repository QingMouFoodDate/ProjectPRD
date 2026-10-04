# 轻眸识刻工程协作与规范指南

> **适用范围**：现有八仓的文档、契约与开发协作
> **设计依据**：[ADR-001](../decisions/ADR-001-REPOSITORY_BOUNDARIES.md)
> **状态边界**：协作规范不代表功能已实现、权限已配置或生产契约已冻结

通用协作规则与领域契约分别管理。字段、枚举、模型形状和数据命名属于候选领域契约，变更须评估关联实现和测试；提交、评审、文档排版与真实验证属于协作要求。

## 一、命名与领域分类

### 1. 样本与资产

- 稳定样本 ID 与文件名通过清单关联，不以文件名推定真实来源、样本数量或语义。
- 既有五位数字名如 `00001.jpg` 可作为兼容示例，不要求为连续编号重命名历史资产；新增命名规则在数据/训练契约中评审。
- 原图保留原始格式、分辨率与完整性；不为统一 `.jpg` 重新编码无损审阅原件。训练裁剪与增强单独记录派生关系。
- `manifest.jsonl` 是候选图像清单名；bbox、OCR、语义、来源许可、group_key 与 split 须有明确关联。样例名称不是实际规模。

### 2. API 路由与字段

- 路由使用小写名词、复数、短横线或斜杠层级；如 `POST /api/v1/sessions`、`POST /api/v1/sessions/{session_id}/images`，仅为候选示例，不代表端点可用。
- JSON 字段使用 `snake_case`，如 `session_id`、`image_id`、`mfg_date`、`exp_date`、`shelf_life`、`marked_status`；派生字段必须带来源与规则版本。
- API 接入属于 InferPlatform 模块，不独立设立 Gateway 仓。具体路由、错误码、上传限制和响应形状以评审后的版本化契约为准。

### 3. 枚举按领域约定

不能规定所有领域枚举统一大写。下表保留已有候选值，分类定义大小写；不因列出枚举而宣称某对象已达到该状态。

| 领域 | 形式 | 候选值与约束 |
| :--- | :--- | :--- |
| 标示期限 `marked_status` | 大写下划线 | `WITHIN_MARKED_PERIOD`、`NEAR_EXPIRY`、`PAST_MARKED_PERIOD`、`UNDETERMINED` |
| 证据复核 `review_status` | 大写下划线 | `EVIDENCE_COMPLETE`、`INFORMATION_MISSING`、`EVIDENCE_CONFLICT`、`LOW_CONFIDENCE`，与期限状态正交 |
| 会话状态与业务错误标识 | 大写下划线 | 如 `AWAITING_INPUT`、`IMAGE_INVALID`、`INFERENCE_TIMEOUT`；完整状态机待 API 契约评审 |
| 文本语义类型 | 大写缩写 | `MFG`、`EXP`、`SHELF_LIFE`；未知值与不可读原因待数据契约定义 |
| 打码工艺 | 小写下划线 | `inkjet_dot_matrix`、`inkjet`、`laser_engraving`、`emboss`、`label_thermal`、`print` |
| 包装载体 | 小写下划线 | `plastic_bag`、`can`、`bottle`、`bottle_cap`、`box`、`tetra_pak`、`plastic_tray` |
| 图片角色与客户端 | 小写下划线或小写词 | 如 `macro_date`、`packaging_full`、`barcode`、`web`，按对应契约定义 |
| 模型治理状态 | 小写词 | `candidate`、`validated`、`released`、`deprecated`；治理状态不等于线上激活状态 |
| 需求当前状态 | 首字母大写 | 无可追溯验收链接时只使用 `Planned` 或 `Proposed` |

训练任务、标注审核与线上激活的状态机由各领域契约定义，不复用模型状态表达所有生命周期。审核待审/通过/拒绝须可追溯，但本规范不擅自发明其机器枚举。

### 4. 模型包与治理动作

现有候选资产名为 `qingmou_det.onnx`、`qingmou_rec.onnx`、`dict.txt`、`metadata.yaml`，包名可示例为 `release_bundle_v2`。格式、输入输出形状、opset 和字表仍待模型契约评审；名称与示例数值都不是已发布产物或实测结果。

- ModelTrain 只读冻结数据，执行训练、评测、导出并提供血缘产物；
- TrainPlatform 治理 `register / validate / release`，记录数据版本、任务、模型包摘要、评测与批准依据；
- InferPlatform 控制 `activate / rollback`，内部 runtime 校验兼容性并执行切换；release 不自动激活；
- 业务纠偏接受后立即重算，训练资格独立审核，不自动更新模型。

### 5. 文档命名

顶层核心文档、契约和标准使用大写文件名，如 `README.md`、`DATE_RULES.md`、`TRAINING.md`、`DATA_QUALITY.md`。设计决议使用 `ADR-编号-主题.md`。候选机器契约路径保持其声明名称，不能套用 Markdown 命名规则。

各级 Markdown 标题不含中英文括号。目录树仅展示必要核心文档与模块，每行附简短中文职责说明，不展开实现细节或私有资源。

## 二、分支与中文提交规范

### 1. 分支管理

- `main` 保持可验证：代码仓满足其实际构建/运行/测试要求；ProjectPRD 文档仓满足文档链接、结构和契约一致性校验，不宣称可运行产品。
- 主干保护是否启用以平台实际配置为准；禁止直接强制推送覆盖历史。
- `feat/*`、`fix/*` 从主干建立，按切片评审并提供相关验证后合并；不擅自处理他人的工作区或未解决合并。

### 2. 提交信息

格式为 `<emoji> <type>: <简明中文改动>`，中文动宾短语，不以句号结尾。emoji 必填，使用下表约定；正文文档不使用装饰性 emoji，此处提交规范与示例为必要例外。

| 前缀 | 含义 | 示例 |
| :--- | :--- | :--- |
| `🎉 init` | 项目初始化 | `🎉 init: 初始化仓库工程结构` |
| `✨ feat` | 新特性 | `✨ feat: 新增多图会话接口` |
| `🐞 fix` | 修复缺陷 | `🐞 fix: 修复闰年日期推导偏差` |
| `🎈 perf` | 性能与体验优化 | `🎈 perf: 优化模型推理耗时` |
| `🦄 refactor` | 代码重构 | `🦄 refactor: 调整业务平台模块分层` |
| `📃 docs` | 仅修改文档 | `📃 docs: 记录八仓职责与单图里程碑` |
| `🌈 style` | 仅格式缩进 | `🌈 style: 统一组件代码排版` |
| `🧪 test` | 增加或修改测试 | `🧪 test: 增加会话接口集成测试` |
| `🔧 build` | 依赖相关 | `🔧 build: 更新模型导出依赖` |
| `🐎 ci` | CI/CD 配置 | `🐎 ci: 调整持续集成检查流程` |
| `🐳 chore` | 构建或辅助工具 | `🐳 chore: 更新开发辅助脚本` |
| `↩ revert` | 版本回滚 | `↩ revert: 回滚会话接口变更` |

### 3. 提交前核验

1. 使用只读状态和差异检查确认写集，如 `git --no-pager --no-optional-locks status` 与 `git --no-pager diff`，排除缓存、依赖目录、日志和临时文件。
2. 运行仓库已有且与改动相关的检查，记录实际命令与结果；没有入口、失败、超时或未运行均须如实说明。
3. 大容量原图、视频、训练权重与模型包不得直接进入代码仓历史，使用受控资产交付和版本引用。
4. 权限以本次任务授权为准；仅文档编辑任务不因此获得暂存、提交、合并、重写历史或推送授权。

## 三、文档状态与权限

- 用客观中文说明场景、边界、输入输出、依赖和验收，不写宣传口号，不将拟议方案描述为已支持功能。
- 职责 ADR 的 Accepted 是设计状态，需求 Planned/Proposed 是实现计划，模型治理状态是另一领域，三者不可相互替代。
- 性能/准确率无数据时标为未测量目标；示例模型卡片、接口响应、截图和预测值不是实测或验收记录。
- 不写本地专属私有路径、资产规模、用户信息、秘密或授权凭据；公开链接不能隐含授权下载私人资源。
- 文档内容按可公开边界编写，但仓库可见性、成员访问、分支保护及资产权限须以实际配置为准，不能从仓名或文档推定。
- 公开设计不授予会话、图片、模型或训练资产访问权；线上资源按归属授权，离线治理按用途授权，训练与公开分发许可分别审核。
- 包装状态不等于食品安全；必须保留缺失、冲突、低置信度与人工核对提示。
- 数学使用纯文本：映射用 `->`，乘除用 `*`、`/`，指数用 `^`；不使用数学排版源码。

## 四、契约与真实验证

人工契约按 API、DATA、DATE_RULES、MODEL、TRAINING 分工，现有内容仍属候选。机器契约已经纳入 `contracts/openapi.json`，以及 `contracts/schemas/observation.v1.schema.json`、`contracts/schemas/evidence.v1.schema.json`、`contracts/schemas/decision.v1.schema.json`、`contracts/schemas/correction.v1.schema.json`。机器文件须实际校验语法、引用与人工契约一致性；文件存在或结构校验通过不表示服务实现、生产冻结或模型验证完成。

- 文档仓检查需求 ID、职责与阶段一致性、链接、Markdown 结构、候选缺项及隐私边界；机器文件实际存在后再校验语法、Schema 与人工契约一致性。
- 实现仓分别验证 runtime Observation、Evidence/Decision 规则、业务纠偏立即重算、Train 独立审核、只读冻结输入、分组防泄漏与模型发布/激活隔离。
- 验收记录须关联需求 ID、代码/契约/数据版本、环境、实际命令、结果与链接；只对实际运行通过的检查声称通过。
- M0 优先单机单图闭环；多图、完整纠偏审核与暂停客户端不成为 M0 门禁，也不能因此删除后续 C0 范围。

## 五、迁移与协作

1. 保留八仓原名，旧版职责弃用；MobileClient、WxClient 暂停实现，不强制合并；不新增 Data、Registry、Gateway、Runtime 仓。
2. 按文档 -> 代码 -> 测试 -> CI 迁移，每步保留版本映射和可验证记录，不自动重写提交历史。
3. 契约先评审再实现，内部模块不建立对其他仓实现细节或可变训练资产的直接依赖。
4. 先核验、后执行、再复检；不伪造门禁结果，不简化有意义的功能以掩盖失败，不修改任务授权之外的文件。
